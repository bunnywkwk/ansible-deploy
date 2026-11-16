# D3 — How a new machine configures itself

The pattern we keep: the kickstart installs the OS and prepares a one-shot systemd unit; the first real boot runs the
playbook once and leaves a record. Proven 2026-11-16 ([evidence](evidence.md#d3--first-boot-service-2026-11-16)).

```text
rhel9-webserver (192.168.150.10)          new machine
──────────────────────────────            ───────────────────────────────────────────────────────────
ks/d3-firstboot.cfg  ──────────────────►  INSTALLER
rhel9/9.8/  (RPMs)   ──────────────────►    base.cfg part: OS, CIS LVM, repos, admin key, sudo
releases/d3/deploy-d3.tar.gz ──────────►    %post: unpack bundle → /opt/deploy
                                                   venv with ansible-core 2.16.20 from the bundled wheels
                                                   write deploy-firstboot.service, systemctl enable
                                                   (nothing runs yet)
                                          reboot
                                          FIRST BOOT (real OS: systemd, firewalld, services up)
                                            deploy-firstboot.service → ansible-playbook main_playbook.yml
                                            success → /var/lib/deploy/done + a line in /var/lib/deploy/record
                                          LATER BOOTS
                                            done exists → unit skipped
```

## Why each part is the way it is

| Choice | Why |
| --- | --- |
| Playbook at first boot, not in `%post` | In the installer there is no systemd and no firewalld: D2's firewalld task failed and the install still "succeeded" ([D2 findings](d2-post-playbook.md)). First boot is the same running RHEL 9 that CI tested. |
| Unit written in the kickstart `%post` | How Foreman/Satellite and Red Hat's own kickstarts do it. `systemctl enable` only creates a link, so it works in the installer. |
| No script: the unit does it all | systemd already has what we need: `Environment=` (locale), `ExecStartPost=` (runs only after success), `ExecStopPost=` (always runs, gets `$SERVICE_RESULT`), `StateDirectory=` (creates `/var/lib/deploy`). Less code than a wrapper script. |
| `ConditionPathExists=!/var/lib/deploy/done` | Runs once. `done` is written only on success, so a failed run is retried at the next boot. |
| `LC_ALL=C.UTF-8` | ansible-core refuses to start without a UTF-8 locale; installers and services do not set one. `LC_ALL`, because it overrides `LANG` ([troubleshooting](troubleshooting.md)). |
| ansible-core 2.16.20 from the bundle | RHEL 9.8's DVD only has 2.14.18, which CI never tests. The bundle carries the CI pin as wheels; `python3.11` comes from the DVD. No internet needed. |
| Venv made in `%post` | Plain files, proven in D2. The first boot then only runs the playbook. |
| Everything from `rhel9-webserver` | The machine is air-gapped: its only server is the repo server, like the FDS at work. |

## How to check a machine

```bash
sudo systemctl status deploy-firstboot --no-pager     # Process lines: ExecStart, ExecStartPost, ExecStopPost
sudo journalctl -u deploy-firstboot --no-pager        # the whole Ansible output
sudo cat /var/lib/deploy/record                       # one line per run: time, release, result
sudo cat /root/ks-post-deploy.log                     # what %post did during the install
```

If the run failed: the unit shows `failed`, the record says `result=exit-code`, `done` is missing. Fix the cause, then
reboot or `sudo systemctl start deploy-firstboot` to retry.

## Known limits

- The failure path ran only in a workstation test of the record line, not on a VM.
- No checksum on the bundle yet (D4).
- `release=d3` is a hand-written `RELEASE` file; D4 makes it the git tag.
- A cloned image (D6) has no `%post`: its preparation must do the same three steps (unpack, venv, unit).
