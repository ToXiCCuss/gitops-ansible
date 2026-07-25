# server-config – Basic GitOps Framework

Ansible role "base" for updates, SSH hardening (including banners & timeouts), 
kernel hardening (sysctl), and automatic security updates.
Additionally applies CIS Benchmark Level 1 hardening for Debian 13 via the
[ansible-lockdown/DEBIAN13-CIS](https://github.com/ansible-lockdown/DEBIAN13-CIS) role.
Runs independently on each VM via `ansible-pull`.

## Hardening Features

- **SSH**: Session-timeouts, login banner.
- **Kernel**: Protection against IP spoofing, disabling ICMP redirects, TCP syncookies.
- **File System**: Shared Memory (`/run/shm`) protection (noexec, nosuid).
- **Updates**: Fully automatic security updates via `unattended-upgrades`.
- **CIS Benchmark**: Level 1 remediation for Debian 13 via the `DEBIAN13-CIS` role
  (ansible-lockdown), configured in `site.yml` (`debian13cis_level: 1`).

## Structure

```
server-config/
├── inventory.ini              # Server list (for documentation, ansible-pull uses local host)
├── site.yml                   # Main playbook
├── requirements.yml           # External Ansible collections & roles (incl. DEBIAN13-CIS)
├── roles/base/
│   ├── defaults/main.yml      # Customizable variables (ports, ...)
│   ├── tasks/main.yml         # Core configuration
│   ├── templates/             # sshd & unattended-upgrades configs
│   └── handlers/main.yml
└── systemd/
    ├── ansible-pull.service
    └── ansible-pull.timer
```

## Initial Setup per VM

```bash
# 1. Update the Repo URL in systemd/ansible-pull.service (YOUR_ORG/server-config)

# 2. Install Ansible
apt update && apt install -y ansible git

# 2b. Install external roles/collections (incl. DEBIAN13-CIS from Ansible Lockdown)
# By default this installs the role into ~/.ansible/roles/DEBIAN13-CIS,
# which is part of Ansible's default role search path.
ansible-galaxy install -r requirements.yml

# 3. Install systemd units
cp systemd/ansible-pull.service /etc/systemd/system/
cp systemd/ansible-pull.timer /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now ansible-pull.timer

# 4. Trigger first run manually (optional, otherwise it runs after OnBootSec)
systemctl start ansible-pull.service
journalctl -u ansible-pull.service -f
```

## Important Before First Rollout

- **Ensure SSH access is configured**, as the hardening overrides might change settings.
- **CIS hardening (`DEBIAN13-CIS`)** can be disruptive (e.g. SSH/root login restrictions,
  password policies). Review the role's `defaults/main.yml` and test in a non-production
  environment before rolling out broadly.
- **`ansible-galaxy install -r requirements.yml` must be run as the same user that executes
  `ansible-pull`** (typically `root`, since `ansible-pull.service` runs as root). `ansible-pull`
  itself does not install role/collection dependencies, so this step has to be run manually once
  per VM (step 2b) and repeated whenever `requirements.yml` changes.

## Rolling Out Changes

Simply commit and push – each VM will pull the new configuration
within a maximum of 30 minutes (timer interval customizable in
`systemd/ansible-pull.timer`).
