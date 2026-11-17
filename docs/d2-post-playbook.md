# D2 — Playbook in `%post`: findings

**Question:** can the release playbook run inside the installer (`%post`), so the machine is configured before its
first boot? **Answer: no, not reliably.** Files and packages work; anything that needs a running system fails.
We keep `%post` for fetching and enabling only, and run the playbook at first boot (D3).

Kickstart: `kickstart/d2-post-playbook.cfg` (kept on `staging` as the record of this option). Bundle:
`releases/d2/deploy-d2.tar.gz` (motd `v0.1.0`, webserver `v0.1.1`, `ansible.posix` 2.2.2, ansible-core 2.16.20
wheels). Three installs of `ks-d2`, 2026-11-16; log `/root/ks-post-playbook.log`.

## What worked and what failed

| Step | In `%post` | Why |
| --- | --- | --- |
| Download and unpack the bundle (`curl`, `tar`) | Works | The installer's network is up |
| `python3.11 -m venv` + `pip install --no-index` from the bundled wheels | Works | Plain files; proves ansible-core 2.16.20 installs with no internet |
| `ansible-playbook` start | **Failed** until `export LC_ALL=C.UTF-8` | The installer sets `LC_ALL=C`; see [troubleshooting](troubleshooting.md) |
| motd: write `/etc/motd` | Works (`changed`) | A file |
| webserver: `dnf install nginx` | Works (`changed`) | dnf in the chroot uses the repo server |
| webserver: templates (server block, page) | Works (`changed`) | Files |
| webserver: firewalld port (`permanent: true`, `immediate: true`) | **Failed** | `firewall is not currently running, unable to perform immediate actions without a running firewall daemon` |
| webserver: nginx enabled and started | Not reached | The play stopped at the firewalld failure |

```text
TASK [webserver : Open the port in firewalld] **********************************
fatal: [localhost]: FAILED! => {"changed": false, "msg": "firewall is not currently running, unable to perform immediate actions without a running firewall daemon"}

PLAY RECAP *********************************************************************
localhost                  : ok=4    changed=3    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
```

## Why we do not pursue it

- **The install still "succeeds".** `%post` failures do not stop Anaconda (only `%post --erroronfail` does). The machine
  boots half configured: nginx installed, page written, port closed, nginx not enabled. Nothing says so except a log in
  `/root`.
- **The installer is not the system CI tested.** No systemd, no firewalld, its own locale. CI ran the role on a running
  RHEL 9; running it here tests something else.
- **Making it work means changing roles for the installer**, e.g. `immediate: false` when firewalld is not running, and
  services that are only enabled, not started. Every role would need that, and handlers that restart a service would
  still fail.
- **First boot has none of these limits** and is the same path as a cloned image (D6).

## Notes

- Not tested: starting a service in the chroot. The play stopped before that task. `systemctl start` in a chroot is
  ignored by systemd, so it may even report success without starting anything.
- `[DEPRECATION WARNING]: Using a mapping for action is deprecated` appears at each `include_role`. It comes from
  ansible-core itself (most likely the argument-spec validation task it adds for roles with `meta/argument_specs.yml`),
  not from our play; harmless on 2.16.
- To check at work: if their kickstart runs the playbook in `%post`, how do they handle services and firewalld?
