# Deploy troubleshooting

Each entry: the exact error, its cause, the fix, and how to avoid it.

## Network install stops: "No space left on device", "Failed to find a root filesystem"

**Error** (D1, first `virt-install` of `ks-test`, 2026-11-16)

```text
WARNING  Overriding memory to 3072 MiB needed for rhel9.0 network install.
 83 1194M   83  996M    0     0   576M      0  0:00:02  0:00:01  0:00:01  575M
dracut-initqueue[1647]: curl: (23) Failure writing output to destination
dracut: FATAL: Failed to find a root filesystem in /tmp/curl_fetch_url1/install.img.
dracut-initqueue[1685]: /lib/dracut-lib.sh: line 454: echo: write error: No space left on device
reboot: System halted
```

**Cause:** with `--location http://…`, the initrd downloads the installer `images/install.img` (1,252,827,136 bytes on
9.8) into a RAM disk that gets about half the VM's RAM (`size=1338588k` at 3 GB). The download stops at 83%.
Booting from a DVD ISO does not hit this: the installer reads `install.img` from the CD.

**Fix:** remove the halted VM and install with 4 GB.

```bash
virsh -c qemu:///system destroy ks-test
virsh -c qemu:///system undefine ks-test --nvram --remove-all-storage
virt-install ... --memory 4096 ...
```

**Prevent:** network installs (here, PXE, Proxmox) need at least 4 GB of RAM during the install; the installed system
can have less.

## Repo server not reachable at 192.168.150.10

**Error** (D0, from the workstation, 2026-11-16)

```text
connect to 192.168.150.10 port 80 from 192.168.150.1 port 34428 failed: No route to host
```

**Cause:** `rhel9-webserver` took a DHCP lease (`192.168.150.197`) instead of the static `.10`.

**Fix:** static address on the server's connection, from its console:
`nmcli con mod "<name>" ipv4.method manual ipv4.addresses 192.168.150.10/24 && nmcli con up "<name>"`.

**Prevent:** check `virsh net-dhcp-leases deploy-offline`; the repo server must never appear there.

## Publishing the kickstart: "cannot create regular file '/var/www/html/ks/base.cfg'"

**Error** (D1, on `rhel9-webserver`, 2026-11-16)

```text
install: cannot create regular file '/var/www/html/ks/base.cfg': No such file or directory
```

**Cause:** D0's `ks/` and `releases/` folders were never created.

**Fix:** `sudo install -d -m 0755 /var/www/html/ks /var/www/html/releases`, then publish again.

**Prevent:** D0's checks now include `curl -sI http://192.168.150.10/releases/` (200).

## Playbook in `%post`: "Ansible requires the locale encoding to be UTF-8; Detected None."

**Error** (D2, `/root/ks-post-playbook.log` on `ks-d2`, 2026-11-16)

```text
+ /opt/deploy/venv/bin/ansible-playbook playbooks/main_playbook.yml --tags motd
ERROR: Ansible requires the locale encoding to be UTF-8; Detected None.
```

**Cause:** ansible-core (2.14 and later) refuses to start unless Python reports a UTF-8 locale. The installer runs
`%post` with `LC_ALL=C`, which overrides everything else.

**First fix tried, did not work:** `export LANG=C.UTF-8`; the second install logged `+ LANG=C.UTF-8` and the same error.
`LANG` is the lowest-priority locale variable: `LC_ALL` overrides it. Reproduced with Python on the workstation:

```text
$ env -i LANG=C.UTF-8 python3 -c 'import locale; locale.setlocale(locale.LC_ALL, ""); print(locale.getlocale())'
('C', 'UTF-8')
$ env -i LANG=C.UTF-8 LC_ALL=C python3 -c 'import locale; locale.setlocale(locale.LC_ALL, ""); print(locale.getlocale())'
(None, None)
```

Confirmed on the third install, by logging the variables at the top of `%post`:

```text
+ env
+ grep -E '^(LANG|LC_)'
LANG=en_US.UTF-8
LC_ALL=C
LC_NUMERIC=C
```

`LANG` was already UTF-8 (from the kickstart's `lang`); the installer's `LC_ALL=C` overrode it.

**Fix:** `export LC_ALL=C.UTF-8` at the top of the `%post` that runs Ansible (`C.UTF-8` is built into glibc, so it
exists even on a minimal install). Republish the kickstart and install the VM again.

**Prevent:** any script that runs Ansible outside a login shell (kickstart `%post`, systemd unit, cron) sets
`LC_ALL=C.UTF-8` itself, not `LANG`.
