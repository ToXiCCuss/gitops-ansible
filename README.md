# server-config – Basic GitOps Framework

Ansible role "base" for updates, SSH hardening (including banners & timeouts), 
kernel hardening (sysctl), and automatic security updates.
Runs independently on each VM via `ansible-pull`.

## Hardening Features

- **SSH**: Key-only login, no root login, session timeouts, login banner.
- **Kernel**: Protection against IP spoofing, disabling ICMP redirects, TCP syncookies.
- **File System**: Shared Memory (`/run/shm`) protection (noexec, nosuid).
- **Updates**: Fully automatic security updates via `unattended-upgrades`.

## Structure

```
server-config/
├── inventory.ini              # Server list (for documentation, ansible-pull uses local host)
├── site.yml                   # Main playbook
├── requirements.yml           # External Ansible collections
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

- **Test SSH Key login beforehand** before `ssh_password_authentication: "no"`
  takes effect – otherwise you will be locked out!

## Rolling Out Changes

Simply commit and push – each VM will pull the new configuration
within a maximum of 30 minutes (timer interval customizable in
`systemd/ansible-pull.timer`).
