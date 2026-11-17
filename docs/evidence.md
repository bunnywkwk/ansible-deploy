# Deploy evidence

Verified results per batch: what was run and what it printed. Errors met on the way are in
[troubleshooting.md](troubleshooting.md).

## D0 — Repo server (2026-11-16)

`rhel9-webserver` (RHEL 9.8, set up by hand), `192.168.150.10` on `deploy-offline`; the DVD is mounted read-only at
`/var/www/html/rhel9/9.8/`.

```text
$ curl -sI http://192.168.150.10/rhel9/9.8/BaseOS/repodata/repomd.xml | head -1
HTTP/1.1 200 OK
$ curl -sI http://192.168.150.10/rhel9/9.8/AppStream/repodata/repomd.xml | head -1
HTTP/1.1 200 OK
$ curl -sI http://192.168.150.10/releases/ | head -1
HTTP/1.1 200 OK
```

Not seen: the server's own `dnf repolist` (reported by me, not pasted).

## D1 — Offline install (2026-11-16)

`ks-test` installed by `kickstart/base.cfg`, no clicks, booted from the repo server:

```bash
virt-install --connect qemu:///system --name ks-test --memory 4096 --vcpus 2 \
  --disk size=20 --boot uefi --network network=deploy-offline --os-variant rhel9.0 \
  --graphics none --location http://192.168.150.10/rhel9/9.8/ \
  --extra-args "inst.ks=http://192.168.150.10/ks/base.cfg console=ttyS0"
```

Ended at `ks-test login:`; DHCP gave `192.168.150.106`. `frqadmin` logs in with the key fetched in `%post`, and
`sudo` asks no password.

```text
$ sudo dnf repolist
repo id                                                    repo name
AppStream                                                  RHEL 9.8 AppStream (repo server)
BaseOS                                                     RHEL 9.8 BaseOS (repo server)
$ getenforce
Enforcing
$ lsblk
vda                  252:0    0   20G  0 disk
├─vda1               252:1    0  600M  0 part /boot/efi
├─vda2               252:2    0    1G  0 part /boot
└─vda3               252:3    0 18.4G  0 part
  ├─vg_sys-lv_root   253:0    0    8G  0 lvm  /
  ├─vg_sys-lv_swap   253:1    0    1G  0 lvm  [SWAP]
  ├─vg_sys-lv_audit  253:2    0    1G  0 lvm  /var/log/audit
  ├─vg_sys-lv_varlog 253:3    0    1G  0 lvm  /var/log
  ├─vg_sys-lv_vartmp 253:4    0    1G  0 lvm  /var/tmp
  ├─vg_sys-lv_var    253:5    0    3G  0 lvm  /var
  ├─vg_sys-lv_tmp    253:6    0    1G  0 lvm  /tmp
  └─vg_sys-lv_home   253:7    0    1G  0 lvm  /home
$ sudo firewall-cmd --list-services
cockpit dhcpv6-client ssh
$ curl -sS --max-time 5 https://www.redhat.com; echo "curl exit=$?"
curl: (6) Could not resolve host: www.redhat.com
curl exit=6
$ sudo cat /root/ks-post.log
(empty: %post ran without errors)
```

Notes:

- `dnf` prints "This system is not registered…": the subscription-manager plugin; harmless, the system is not
  registered on purpose.
- `cockpit` and `dhcpv6-client` are RHEL's default `public` zone services; the kickstart only adds `ssh`.

## D2 — Playbook in `%post` (2026-11-16)

Findings and the failing output: [d2-post-playbook.md](d2-post-playbook.md).

## D3 — First-boot service (2026-11-16)

Final version: `kickstart/d3-firstboot.cfg`, the unit runs `ansible-playbook` itself (no script). Fresh `ks-d3`.

**First boot**

```text
$ sudo systemctl status deploy-firstboot --no-pager
○ deploy-firstboot.service - Deploy: run the release playbook once
     Loaded: loaded (/etc/systemd/system/deploy-firstboot.service; enabled; preset: disabled)
     Active: inactive (dead) since Mon 2026-11-16 15:32:01 UTC; 54s ago
    Process: 946 ExecStart=/opt/deploy/venv/bin/ansible-playbook playbooks/main_playbook.yml (code=exited, status=0/SUCCESS)
    Process: 9553 ExecStartPost=/usr/bin/touch /var/lib/deploy/done (code=exited, status=0/SUCCESS)
    Process: 9566 ExecStopPost=/bin/sh -c echo "$$(date -Is) release=$$(cat RELEASE) result=$$SERVICE_RESULT" >> /var/lib/deploy/record (code=exited, status=0/SUCCESS)
$ sudo journalctl -u deploy-firstboot --no-pager | grep -E 'RECAP|ok=|Finished'
Nov 16 15:32:01 ks-test ansible-playbook[946]: PLAY RECAP *********************************************************************
Nov 16 15:32:01 ks-test ansible-playbook[946]: localhost                  : ok=9    changed=7    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
Nov 16 15:32:01 ks-test systemd[1]: Finished Deploy: run the release playbook once.
$ sudo cat /var/lib/deploy/record; ls -l /var/lib/deploy/done
2026-11-16T15:32:01+00:00 release=d3 result=success
-rw-r--r--. 1 root root 0 Nov 16 15:32 /var/lib/deploy/done
$ systemctl is-active nginx; sudo firewall-cmd --list-ports; curl -s localhost/webserver.html | grep Hello; cat /etc/motd
active
80/tcp
  <p>Hello from the lab</p>
Authorized users only
$ sudo tail -5 /root/ks-post-deploy.log
Successfully installed MarkupSafe-3.0.4 PyYAML-6.0.3 ansible-core-2.16.20 cffi-2.1.1 cryptography-50.0.2 jinja2-3.1.6 packaging-26.3 pycparser-3.12 resolvelib-1.0.1
+ cat
+ restorecon -R /opt/deploy /etc/systemd/system/deploy-firstboot.service
+ systemctl enable deploy-firstboot.service
Created symlink /etc/systemd/system/multi-user.target.wants/deploy-firstboot.service -> /etc/systemd/system/deploy-firstboot.service.
```

**After a reboot: not run again, still one record line**

```text
$ sudo journalctl -u deploy-firstboot -b --no-pager
Nov 16 15:33:50 ks-test systemd[1]: Deploy: run the release playbook once was skipped because of an unmet condition check (ConditionPathExists=!/var/lib/deploy/done).
$ sudo cat /var/lib/deploy/record
2026-11-16T15:32:01+00:00 release=d3 result=success
```

- The firewalld and nginx tasks that failed or were not reached in `%post` (D2) pass at first boot.
- `/etc/motd` says `Authorized users only`, the default of the `v0.1.0` tag (the role's `main` has since changed it):
  the released tag was deployed, not the latest code.
- Failure path, not run on a VM: the same `ExecStopPost=` in a throwaway user unit on the workstation records both
  results:

  ```text
  2026-11-16T23:23:29+08:00 release=d3 result=success
  2026-11-16T23:23:30+08:00 release=d3 result=exit-code
  ```

- An earlier version (the unit ran a `run.sh` from the bundle) passed the same checks at 15:11 the same day; the script
  was dropped because systemd's own settings do the same.
