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
