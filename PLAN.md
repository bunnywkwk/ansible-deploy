# Ansible Deploy — Plan

Nov 14, 2026 · @Aeron · changed Nov 16: repo server is a new VM `rhel9-webserver`, not `rhel-kvm-server` (D0)

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

  rhel9-webserver (new VM, RHEL 9.8)       http://192.168.150.10/
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
| Repo server | New VM `rhel9-webserver` (RHEL 9.8, installed from the DVD), only on `deploy-offline`; httpd from its own DVD | Changed Nov 16: `rhel-kvm-server` will be removed. A server used only for this is air-gapped like the machines it serves |
| Lab network | New libvirt network `deploy-offline` (no forward), `192.168.150.0/24`, DHCP `.100–.200`; repo server `.10` | Proves the install needs no internet |
| Boot media | Installer kernel and initrd from the repo server (`virt-install --location http://192.168.150.10/rhel9/9.8/`) + `inst.ks=http://…` | Changed Nov 16: the 9.8 boot ISO is gone from `~/ISO`; booting from the repo server needs no ISO and is what PXE does |
| Bare metal in the lab | Blank KVM VM boots the ISO + kickstart | Same path as bare metal; only the boot media differs |
| VM deployment | Golden image clone + the same first-boot unit (D6) | Second way, reusing the same bundle |
| Release | Tag `v<VERSION>` on this repo; Jenkins builds `deploy-<version>.tar.gz` + checksum, publishes it to the repo server | One small versioned file per release |
| Role versions | `requirements.yml` pins released role tags | Only CI-tested code is deployed |
| Ansible on the machine | `ansible-core` 2.16.20 (the CI pin) as wheels inside the bundle, in a `python3.11` venv; `python3.11` from AppStream; collections inside the bundle | Changed Nov 16: RHEL 9.8 only ships 2.14.18, which CI never tests; the target must run what CI tested. Nothing from the internet |
| Jenkins | Folder `cd/`, separate from `role-ci/` | CI and CD stay apart |
| Approval | Jenkins `input` step; the approver is me | Production waits for a click |

## Progress tracker

| Batch | Goal | Ready when | Status |
| --- | --- | --- | --- |
| D0 — Repo server | `rhel9-webserver` serves the RHEL 9.8 repos on `deploy-offline` | From the workstation, `curl http://192.168.150.10/rhel9/9.8/BaseOS/repodata/repomd.xml` works | Done (2026-11-16): `curl` 200 from the workstation; server-side `dnf repolist` reported by me, not pasted; `ks/` and `releases/` were missing, made at the start of D1 |
| D1 — Offline install | A blank VM installs RHEL 9 by kickstart from the repo server, no clicks, no internet | VM boots to login; `dnf repolist` shows only the repo server | Done (2026-11-16), [evidence](docs/evidence.md#d1--offline-install-2026-11-16) |
| D2 — Playbook in `%post` | See what a playbook can and cannot do in the installer | `%post` log and a list of what failed and why | Done (2026-11-16): files and packages work, firewalld fails; not pursued, [findings](docs/d2-post-playbook.md) |
| D3 — First-boot service | The machine configures itself on first boot, once | Unit succeeds, `/var/lib/deploy/done` exists, a reboot does not rerun it | Done (2026-11-16), [how it works](docs/d3-firstboot.md), [evidence](docs/evidence.md#d3--first-boot-service-2026-11-16) |
| D4 — Release bundle | One versioned file holds everything the install needs | Jenkins builds `deploy-<version>.tar.gz` from a tag and publishes it; D3 works from it | Not started |
| D5 — CD job | Staging → approval → production from one release | `cd/deploy` installs staging, waits for approval, installs production; deploy record per run | Not started |
| D6 — VM deployment | Same bundle on a golden image clone | Clone boots, first-boot unit runs the same release | Not started |
| D7 — Upgrade and rollback | Move a running machine v1 → v2 → v1 without reinstall | Both runs succeed; the record shows each version | Not started |
| D8 — Bare metal notes | What changes for USB, PXE, BMC virtual media, Proxmox | One page: only the boot step changes | Not started |

## Before D0

- [ ] Download `rhel-9.8-x86_64-dvd.iso` to `~/ISO` (installs the repo server and fills it).
- [ ] Create the GitHub repo `ansible-deploy` and push this folder.

## D0 — Repo server

**Goal:** `rhel9-webserver` serves the RHEL 9.8 DVD content, kickstarts and releases over HTTP on an isolated network.

- [ ] Create libvirt network `deploy-offline` in Cockpit (forward mode Isolated, `192.168.150.1` on the workstation, DHCP `.100–.200`).
- [ ] Install `rhel9-webserver` by hand from the 9.8 DVD ISO: one NIC on `deploy-offline`, static `192.168.150.10`,
      minimal install, an admin user with my SSH key and sudo. The DVD stays attached as a CD-ROM. Not registered.
- [ ] Set up by hand, no playbook (changed Nov 16, at my choice): the DVD is mounted read-only at
      `/var/www/html/rhel9/9.8/` from `/etc/fstab`, not copied (saves about 11 GB). Because the ISO is read-only,
      `restorecon` cannot relabel it; the mount sets the label instead: `context="system_u:object_r:httpd_sys_content_t:s0"`.
- [ ] The server's own dnf uses that mount (`file://`); httpd installed, started and enabled; http open in firewalld;
      SELinux enforcing.
- [ ] Folders `ks/` and `releases/` under `/var/www/html/`.

**Done when:** from the workstation, `curl -sI http://192.168.150.10/rhel9/9.8/BaseOS/repodata/repomd.xml` gives
`200`; on the server, `dnf repolist` shows only its own copy.

## D1 — Offline install

**Goal:** `virt-install` boots a blank VM on `deploy-offline` from the repo server with my kickstart and ends with a
working RHEL 9.

- [x] `kickstart/base.cfg`, based on `~/ansible-cis/golden-images/kickstart/ks-rhel9.cfg` (UEFI, CIS LVM layout in
      `vg_sys`, 20 GB disk): `url` (BaseOS) + `repo` (AppStream) from the repo server, `@^minimal-environment`,
      `ansible-core`, root locked, admin `frqadmin` (locked password, sudo without one), SELinux enforcing, firewall on.
- [x] `%post` writes `/etc/yum.repos.d/` files pointing to the repo server (the installed system has none otherwise)
      and fetches the admin's key from `http://192.168.150.10/ks/authorized_keys`. Changed Nov 16: no key in git;
      the repo is public and a key change needs no commit.
- [x] `virt-install --memory 4096 --boot uefi --location http://192.168.150.10/rhel9/9.8/ --extra-args "inst.ks=http://192.168.150.10/ks/base.cfg"`.

**Done when:** the VM boots to a login with no clicks; `dnf repolist` shows only the repo server;
`getenforce` is `Enforcing`; `curl https://www.redhat.com` fails (no internet).

## D2 — Playbook in `%post`

**Goal:** see the installer's limits with my own eyes.

- [x] Project files in the `sysconfig` layout: `ansible.cfg`, `requirements.yml` (motd `v0.1.0`, webserver `v0.1.1`,
      `ansible.posix` 2.2.2), `requirements-ansible.txt`, `sysconfig/inventory.yml` (localhost, local connection),
      `playbooks/main_playbook.yml` (one play).
- [x] Bundle built by hand on the workstation (`build/`, not in git): roles, collections, ansible-core wheels;
      published to `releases/d2/deploy-d2.tar.gz`.
- [x] `kickstart/d2-post-playbook.cfg` = `base.cfg` + one `%post`: fetch the bundle, make the venv, run the playbook
      in the chroot, once per role. The webserver role brings the service and firewalld tasks. Kept as the record of this option (Nov 16, my choice; changed Nov 17: merged to `main` too, the docs link to it). `kickstart/deploy.cfg` is the one in use.

**Done when:** `/root/ks-post-playbook.log` shows what worked and what failed, with the reason, in `docs/`.

**Result:** [docs/d2-post-playbook.md](docs/d2-post-playbook.md).

## D3 — First-boot service

**Goal:** the pattern to keep: `%post` downloads the bundle to `/opt/deploy` and enables `deploy-firstboot.service`;
the first boot runs the playbook once.

- [x] `deploy-firstboot.service`, written by the kickstart `%post` (changed Nov 16: was a file in the bundle;
      Foreman/Satellite and Red Hat's kickstarts write the unit in `%post`): one-shot, after `network-online.target`,
      skipped when `/var/lib/deploy/done` exists; a failed run is retried at the next boot.
- [x] The unit does everything itself, no script (changed Nov 16, at my request: a first version used `run.sh` and
      passed; systemd's own settings do the same with less code): `Environment=LC_ALL=C.UTF-8` (lesson from D2),
      `ExecStart=` the playbook, `ExecStartPost=` writes `done` (only on success), `ExecStopPost=` appends time,
      release and `$SERVICE_RESULT` to `/var/lib/deploy/record`.
- [x] `kickstart/d3-firstboot.cfg` = `base.cfg` + one `%post`: fetch the bundle (`releases/d3/deploy-d3.tar.gz`, with a
      `RELEASE` file), make the venv from its wheels, write and enable the unit. A cloned image (D6) has no `%post`,
      so it does the same steps in its own preparation.

**Done when:** motd and webserver are configured after the first boot (nginx running, port open); the unit log is in
the journal; the record shows `result=success`; a second reboot does not run it.

**Result:** [docs/d3-firstboot.md](docs/d3-firstboot.md).

## D4 — Release bundle

**Goal:** a release is one file, built by Jenkins from a tag of this repo and published to the repo server.

A deploy release is the playbook project's version, not a role's: `v1.0.0` of this repo fixes the role tags
(`requirements.yml`), the play, the site values (`sysconfig/`) and the ansible-core version together. Any change to
one of them is a new release.

- [ ] `VERSION` (`1.0.0`), `Jenkinsfile`, `.yamllint` and `.ansible-lint` (copied from the role repos).
- [ ] Branch or PR: Version, YAML lint, Ansible lint (`production` profile; also checks the play's syntax).
- [ ] Merge to `main`, like the role CI: only a PR merge commit; build the bundle (roles, collections, ansible-core
      wheels, `RELEASE` = the version), `deploy-<VERSION>.tar.gz.sha256`, publish to `releases/<VERSION>/` (upload to a
      temporary folder, then rename; an existing folder fails the build), then tag `v<VERSION>`. An existing tag skips.
- [ ] `kickstart/deploy.cfg` (replaces `d3-firstboot.cfg` as the kept kickstart): the release is named on the boot
      line, `deploy.release=<VERSION>`, so D5 can send different releases to staging and production; `%post` checks
      the checksum; `%post --erroronfail` stops the install on any error (lesson from D2).
- [ ] Setup (me): `ansible-deploy` in the `jenkins-laptop` token (Contents write) + smee webhook; GitHub rulesets on
      `main` and `v*` tags as in the role repos; Jenkins folder `cd/` + multibranch job `cd/ansible-deploy`; user
      `publish` on `rhel9-webserver` owning `releases/` (key only, no sudo); its key in Jenkins as `repo-server-publish`.
- [ ] Later: payload grows to `ansible-role-jenkins` + `ansible-role-registry`.

**Done when:** merging a PR with `VERSION` `1.0.0` tags `v1.0.0` and publishes `releases/1.0.0/` (bundle and
checksum); a second build of the same `VERSION` skips; `virt-install … deploy.release=1.0.0` installs a VM whose record
says `release=1.0.0 result=success`.

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
- I think the playbooks are stored on the FDS too (Nov 16): confirm where, how they get there (by hand, CI) and how a
  version is chosen.
- What is a release for them: a git tag, a bundle, a full ISO?
- How do they get updates into the web server (newer DVD, Satellite, `reposync`)?
- How do they know an install finished and worked?
- Where do site-specific values (IPs, hostnames, Wazuh settings) come from at install time?

## How we work

- One batch at a time; propose the steps and files first, and wait for an OK.
- I run `virsh`/`virt-install`, changes to `rhel9-webserver`, Jenkins and lab tests; Claude gives exact commands and
  reviews the output.
- No secrets in the repo. Laptop lab notes stay separate from the air-gapped Proxmox environment.
- Known limit: the DVD content is frozen at 9.8, with no security updates (fine for the lab).

## Repo layout (planned)

```text
ansible-deploy/
├── PLAN.md
├── kickstart/base.cfg        # D1
├── ansible.cfg               # roles_path = roles, inventory = sysconfig/inventory.yml
├── requirements.yml          # pinned role tags
├── sysconfig/
│   ├── inventory.yml         # the target groups; complexity goes here, not in the play
│   └── group_vars/           # site values per group
├── playbooks/main_playbook.yml  # one play per target group, roles by include_role + tags
├── VERSION                   # release version (D4)
├── Jenkinsfile               # lint, bundle and publish (D4), CD job (D5)
└── docs/                     # what each batch found
```
