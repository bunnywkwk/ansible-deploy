# Runbook — install a machine from the repo server

Happy-path commands for the three install approaches, as run on 2026-11-16. Why each one exists:
[d2-post-playbook.md](d2-post-playbook.md), [d3-firstboot.md](d3-firstboot.md). Errors met:
[troubleshooting.md](troubleshooting.md).

| Approach | Kickstart | Result |
| --- | --- | --- |
| D1 — OS only | `kickstart/base.cfg` | Plain RHEL 9.8, no playbook |
| D2 — Playbook in `%post` | `kickstart/d2-post-playbook.cfg` | Experiment only: firewalld fails in the installer |
| D3 — Playbook at first boot | `kickstart/d3-firstboot.cfg` | **The pattern we keep** |

Run everything from the workstation in `~/ansible-deploy`, unless marked "on the VM".

## 0. Ready before any install (once)

- Libvirt network `deploy-offline` (isolated, `192.168.150.1/24`, DHCP `.100–.200`), made in Cockpit.
- `rhel9-webserver` at `192.168.150.10`, httpd serving the RHEL 9.8 DVD at `/rhel9/9.8/`.

```bash
# Folders and the admin's public key on the repo server
scp ~/.ssh/id_rsa.pub frqadmin@192.168.150.10:/tmp/
ssh -t frqadmin@192.168.150.10 'sudo install -d -m 0755 /var/www/html/ks /var/www/html/releases &&
  sudo install -m 0644 /tmp/id_rsa.pub /var/www/html/ks/authorized_keys'

# Check
curl -sI http://192.168.150.10/rhel9/9.8/BaseOS/repodata/repomd.xml | head -1   # HTTP/1.1 200 OK
curl -s http://192.168.150.10/ks/authorized_keys | cut -c1-20                    # ssh-rsa AAAA...
```

## 1. Build the bundle (D2 and D3)

The playbook, the role tags, `ansible.posix` and ansible-core 2.16.20 in one file. Set `R` to `d2` or `d3`.

```bash
R=d3
rm -rf build/$R && mkdir -p build/$R
cp -r ansible.cfg requirements.yml requirements-ansible.txt sysconfig playbooks build/$R/
echo $R > build/$R/RELEASE
/opt/ci/bin/ansible-galaxy role install -r requirements.yml -p build/$R/roles
/opt/ci/bin/ansible-galaxy collection install -r requirements.yml -p build/$R/collections
/opt/ci/bin/pip download -d build/$R/wheels -r requirements-ansible.txt --only-binary=:all: \
  --python-version 3.11 --platform manylinux_2_34_x86_64 --platform manylinux_2_28_x86_64 --platform manylinux2014_x86_64
tar -czf build/deploy-$R.tar.gz -C build/$R .
```

Check: `ls build/$R/roles` shows `motd webserver`; `ls build/$R/wheels` includes `ansible_core-2.16.20`.

## 2. Publish

```bash
KS=d3-firstboot.cfg          # base.cfg (D1), d2-post-playbook.cfg (D2), d3-firstboot.cfg (D3)
scp kickstart/$KS frqadmin@192.168.150.10:/tmp/
ssh -t frqadmin@192.168.150.10 "sudo install -m 0644 /tmp/$KS /var/www/html/ks/"

# D2 and D3 only: the bundle
scp build/deploy-$R.tar.gz frqadmin@192.168.150.10:/tmp/
ssh -t frqadmin@192.168.150.10 "sudo install -d -m 0755 /var/www/html/releases/$R &&
  sudo install -m 0644 /tmp/deploy-$R.tar.gz /var/www/html/releases/$R/"

# Check
curl -sI http://192.168.150.10/ks/$KS | head -1                          # 200
curl -sI http://192.168.150.10/releases/$R/deploy-$R.tar.gz | head -1    # 200 (D2, D3)
```

## 3. Install the VM

```bash
VM=ks-d3
virt-install --connect qemu:///system --name $VM --memory 4096 --vcpus 2 \
  --disk size=20 --boot uefi --network network=deploy-offline --os-variant rhel9.0 \
  --graphics none --location http://192.168.150.10/rhel9/9.8/ \
  --extra-args "inst.ks=http://192.168.150.10/ks/$KS console=ttyS0"
```

- `--memory 4096`: the installer downloads `install.img` (1.25 GB) into RAM; 3 GB is not enough.
- Watch it in the terminal; it ends at `ks-test login:`. `Ctrl+]` leaves the console.

```bash
virsh -c qemu:///system net-dhcp-leases deploy-offline    # the VM's IP: the newest lease
ssh frqadmin@<IP>
```

## 4. Check

**D1 — OS only** (on the VM)

```bash
sudo dnf repolist          # only BaseOS and AppStream (repo server)
getenforce                 # Enforcing
lsblk                      # vg_sys: lv_root, lv_home, lv_tmp, lv_var, lv_vartmp, lv_varlog, lv_audit, lv_swap
curl -sS --max-time 5 https://www.redhat.com   # fails: no internet
```

**D2 — Playbook in `%post`** (on the VM)

```bash
sudo cat /root/ks-post-playbook.log   # motd changed; nginx installed; firewalld task FAILED (expected)
```

**D3 — Playbook at first boot** (on the VM)

```bash
sudo systemctl status deploy-firstboot --no-pager    # ExecStart, ExecStartPost, ExecStopPost: SUCCESS
sudo journalctl -u deploy-firstboot --no-pager | grep -E 'ok=|Finished'   # failed=0
sudo cat /var/lib/deploy/record                      # <time> release=d3 result=success
systemctl is-active nginx; sudo firewall-cmd --list-ports; curl -s localhost/webserver.html | grep Hello
sudo reboot
# after the reboot:
sudo journalctl -u deploy-firstboot -b --no-pager    # "skipped because of an unmet condition check"
sudo cat /var/lib/deploy/record                      # still one line
```

## 5. Clean up

```bash
virsh -c qemu:///system destroy $VM        # only if it is running
virsh -c qemu:///system undefine $VM --nvram --remove-all-storage
```

Never remove `rhel9-webserver`: every install needs it.
