# Ansible Deploy — Plan

Nov 14, 2026 · @Aeron · changed Nov 15: repo server on `rhel-kvm-server` (D0)

## Summary

Practice production-style CD in the lab before doing it at work: a machine is installed by **kickstart**, then
**configures itself** by running a playbook it carries (pull), with **no internet** (air-gapped). Packages, kickstarts
and releases come from one **repo web server**, as at work. The same install works on bare metal and on a VM; in the
lab, KVM VMs stand in for both.

- **Input:** released role tags from the role CI (`~/ansible-pipeline`). CD never re-runs role tests.
- **Output:** a working machine, installed from a versioned release, with a record of what was installed.
- **At work:** big roles (Wazuh, KVM), kickstarts that call the playbook, a web server with the DVD content,
  air-gapped Proxmox.
- **In the lab:** my own roles stand in: `ansible-role-motd` first (small), then `ansible-role-jenkins` +
  `ansible-role-registry` from `~/cicd-automation` (a real multi-role deployment, like Wazuh).

## How a release reaches a machine

```text
libvirt network deploy-offline (no internet; the workstation can reach it)

  rhel-kvm-server (existing VM, RHEL 10)   http://192.168.150.10/
  ├── rhel9/9.8/BaseOS/  AppStream/        RHEL 9.8 DVD content
  ├── ks/                                  kickstart files
  └── releases/<version>/                  deploy bundles (Jenkins puts them here, D4)
        ▲ HTTP
        │
  new VM ── boots the boot ISO, inst.ks=http://192.168.150.10/ks/<name>.cfg
            kickstart installs RHEL 9 from the repo server
            %post: .repo files → repo server, fetch the bundle, enable the first-boot unit
            first boot: ansible-playbook -c local, writes "done" and a deploy record
```

## `%post` vs first boot

Both run on the machine (pull). The difference is when.

| | `%post` (inside the installer) | First-boot service (after the first reboot) |
| --- | --- | --- |
| Runs in | The installer, chrooted into the new disk | The real installed system |
| systemd | Not running: `enable` works, `start` fails | Running: services and handlers work |
| Kernel, facts | The installer's | The real ones |
| Same as the CI test | No | Yes: a playbook on a running RHEL 9 |
| A failure | Install still "succeeds" unless `--erroronfail` | Machine boots, unit failed, journal says why |
| Logs | `%post --log=/root/ks-post.log` | `journalctl -u deploy-firstboot` |

Plan: try the playbook in `%post` once (D2) to see what breaks, then use `%post` only to fetch and enable, and run the
playbook at first boot (D3). To check at work which one their kickstarts use.

## Decisions

| Topic | Lab choice | Why |
| --- | --- | --- |
| Model | Pull: the machine runs its own playbook | Air-gapped; nothing logs in from outside |
| Packages | RHEL 9.8 DVD content on the repo server, over HTTP; no registration | Like work: one web server, no Red Hat CDN |
| Repo server | Existing `rhel-kvm-server`, httpd added; no new VM | Reuse; a web server only serves files, so RHEL 10 can host RHEL 9 content |
| Lab network | New libvirt network `deploy-offline` (no forward), `192.168.150.0/24`, DHCP `.100–.200`; repo server `.10` | Proves the install needs no internet |
| Boot media | RHEL 9.8 **boot** ISO (already in `~/ISO`) + `inst.ks=http://…` | Packages come from the repo server; no ISO rebuild per release |
| Bare metal in the lab | Blank KVM VM boots the ISO + kickstart | Same path as bare metal; only the boot media differs |
| VM deployment | Golden image clone + the same first-boot unit (D6) | Second way, reusing the same bundle |
| Release | Tag `v<VERSION>` on this repo; Jenkins builds `deploy-<version>.tar.gz` + checksum, publishes it to the repo server | One small versioned file per release |
| Role versions | `requirements.yml` pins released role tags | Only CI-tested code is deployed |
| Ansible on the machine | `ansible-core` from AppStream; collections inside the bundle | Nothing downloaded from the internet |
| Jenkins | Folder `cd/`, separate from `role-ci/` | CI and CD stay apart |
| Approval | Jenkins `input` step; the approver is me | Production waits for a click |

## Progress tracker

| Batch | Goal | Ready when | Status |
| --- | --- | --- | --- |
| D0 — Repo server | `rhel-kvm-server` serves the RHEL 9.8 repos on `deploy-offline` | From the workstation, `curl http://192.168.150.10/rhel9/9.8/BaseOS/repodata/repomd.xml` works | Not started |
| D1 — Offline install | A blank VM installs RHEL 9 by kickstart from the repo server, no clicks, no internet | VM boots to login; `dnf repolist` shows only the repo server | Not started |
| D2 — Playbook in `%post` | See what a playbook can and cannot do in the installer | `%post` log and a list of what failed and why | Not started |
| D3 — First-boot service | The machine configures itself on first boot, once | Unit succeeds, `/var/lib/deploy/done` exists, a reboot does not rerun it | Not started |
| D4 — Release bundle | One versioned file holds everything the install needs | Jenkins builds `deploy-<version>.tar.gz` from a tag and publishes it; D3 works from it | Not started |
| D5 — CD job | Staging → approval → production from one release | `cd/deploy` installs staging, waits for approval, installs production; deploy record per run | Not started |
| D6 — VM deployment | Same bundle on a golden image clone | Clone boots, first-boot unit runs the same release | Not started |
| D7 — Upgrade and rollback | Move a running machine v1 → v2 → v1 without reinstall | Both runs succeed; the record shows each version | Not started |
| D8 — Bare metal notes | What changes for USB, PXE, BMC virtual media, Proxmox | One page: only the boot step changes | Not started |

## Before D0

- [ ] Download `rhel-9.8-x86_64-dvd.iso` to `~/ISO` (needed once, to fill the repo server).
- [ ] Create the GitHub repo `ansible-deploy` and push this folder.

## D0 — Repo server

**Goal:** `rhel-kvm-server` serves the RHEL 9.8 DVD content, kickstarts and releases over HTTP on an isolated network.

- [ ] Create libvirt network `deploy-offline` (no forward, `192.168.150.1` on the workstation, DHCP `.100–.200`).
- [ ] Add a NIC on `deploy-offline` to `rhel-kvm-server`, static `192.168.150.10`. Its other networks stay as they are.
- [ ] Copy the 9.8 DVD ISO to the server, loop-mount it, copy its content to `/var/www/html/rhel9/9.8/`.
- [ ] httpd installed (from the server's own repos or its attached 10.2 DVD), started, port 80 open in firewalld on
      the `deploy-offline` interface; SELinux enforcing (content under `/var/www/html` keeps the right label).
- [ ] Folders `ks/` and `releases/`. Done by a playbook in this repo, `repo-server.yml`.

**Done when:** from the workstation, `curl -sI http://192.168.150.10/rhel9/9.8/BaseOS/repodata/repomd.xml` gives
`200`; the server's existing KVM work is untouched.

## D1 — Offline install

**Goal:** `virt-install` boots a blank VM on `deploy-offline` from the boot ISO with my kickstart and ends with a
working RHEL 9.

- [ ] `kickstart/base.cfg`: `url` (BaseOS) + `repo` (AppStream) from the repo server, `text`, `@^minimal-environment`,
      `ansible-core`, root locked, one admin user with an SSH key, SELinux enforcing, firewall on, `reboot`.
- [ ] `%post` writes `/etc/yum.repos.d/` files pointing to the repo server (the installed system has none otherwise).
- [ ] `virt-install --location <boot ISO> --extra-args "inst.ks=http://192.168.150.10/ks/base.cfg"`.

**Done when:** the VM boots to a login with no clicks; `dnf repolist` shows only the repo server;
`getenforce` is `Enforcing`; `curl https://www.redhat.com` fails (no internet).

## D2 — Playbook in `%post`

**Goal:** see the installer's limits with my own eyes.

- [ ] Small bundle (motd role + `site.yml`) on the repo server; `%post` downloads it and runs
      `ansible-playbook -c local` in the chroot.
- [ ] Also try a task that starts a service and one that uses firewalld.

**Done when:** `/root/ks-post.log` shows what worked and what failed, with the reason, in `docs/`.

## D3 — First-boot service

**Goal:** the pattern to keep: `%post` downloads the bundle to `/opt/deploy` and enables `deploy-firstboot.service`;
the first boot runs the playbook once.

- [ ] Unit: one-shot, after `network-online.target`, skipped when `/var/lib/deploy/done` exists.
- [ ] `run.sh`: `ansible-playbook -c local -i localhost, site.yml`, then writes `done` and a deploy record
      (version, time, result).

**Done when:** motd is configured after the first boot; the unit log is in the journal; a second reboot does not run it.

## D4 — Release bundle

**Goal:** a release is one file, built by Jenkins from a tag of this repo and published to the repo server.

- [ ] `requirements.yml` pins role tags; Jenkins installs them and the collections into the bundle.
- [ ] Tag rule like the role CI: tag `v<VERSION>` from a merged PR only; tags cannot move.
- [ ] Publish to `releases/<version>/` with its checksum; the kickstart checks the checksum before use.
- [ ] Payload grows to `ansible-role-jenkins` + `ansible-role-registry`.

**Done when:** an install from the published bundle works on `deploy-offline`, and the record shows the version.

## D5 — CD job

**Goal:** one Jenkins job, `cd/deploy`, parameter `RELEASE`: install a staging VM, check it, wait for approval,
install the production VM.

**Done when:** a good release reaches production after approval; a broken release stops at staging; each run leaves a
deploy record.

## D6 — VM deployment

**Goal:** the same bundle on a golden image clone (no kickstart): repos and bundle from the repo server, the same unit,
boot.

**Done when:** the clone ends in the same state as the D5 machine.

## D7 — Upgrade and rollback

**Goal:** on a running machine, fetch a new bundle from the repo server and run it; then the old one again.

**Done when:** v1 → v2 → v1 all succeed and the records show it. Notes on what a role must do for a clean rollback.

## D8 — Bare metal notes

**Goal:** one page on what changes off the lab: PXE (DHCP + TFTP, files from the same repo server), USB stick
(`mkksiso` puts the kickstart on the ISO), BMC virtual media (Redfish), Proxmox (upload the boot ISO). The kickstart,
repo server and bundle stay the same.

## To check at work

- Does their kickstart run the playbook in `%post`, at first boot, or another way?
- What is a release for them: a git tag, a bundle, a full ISO?
- How do they get updates into the web server (newer DVD, Satellite, `reposync`)?
- How do they know an install finished and worked?
- Where do site-specific values (IPs, hostnames, Wazuh settings) come from at install time?

## How we work

- One batch at a time; propose the steps and files first, and wait for an OK.
- I run `virsh`/`virt-install`, changes to `rhel-kvm-server`, Jenkins and lab tests; Claude gives exact commands and
  reviews the output.
- No secrets in the repo. Laptop lab notes stay separate from the air-gapped Proxmox environment.
- Known limit: the DVD content is frozen at 9.8, with no security updates (fine for the lab).

## Repo layout (planned)

```text
ansible-deploy/
├── PLAN.md
├── repo-server.yml           # D0: httpd + DVD content on rhel-kvm-server
├── kickstart/base.cfg        # D1
├── site.yml                  # the deployment's play
├── requirements.yml          # pinned role tags
├── group_vars/               # site values
├── firstboot/                # deploy-firstboot.service, run.sh (D3)
├── Jenkinsfile               # bundle (D4), CD job (D5)
└── docs/                     # what each batch found
```
