# server-config – Basis-GitOps-Grundgerüst

Ansible-Rolle "base" für Updates, SSH-Hardening (inkl. Banner & Timeouts), 
Kernel-Hardening (sysctl) und automatische Sicherheitsupdates.
Läuft per `ansible-pull` selbstständig auf jeder VM.

## Hardening Features

- **SSH**: Key-only Login, kein Root-Login, Session-Timeouts, Login-Banner.
- **Kernel**: Schutz vor IP-Spoofing, Deaktivierung von ICMP-Redirects, TCP Syncookies.
- **Dateisystem**: Shared Memory (`/run/shm`) Schutz (noexec, nosuid).
- **Updates**: Vollautomatische Sicherheitsupdates via `unattended-upgrades`.

## Struktur

```
server-config/
├── inventory.ini              # Server-Liste (nur zur Doku, ansible-pull nutzt lokalen Host)
├── site.yml                   # Haupt-Playbook
├── requirements.yml           # externe Ansible-Collections
├── roles/base/
│   ├── defaults/main.yml      # anpassbare Variablen (Ports, ...)
│   ├── tasks/main.yml         # eigentliche Konfiguration
│   ├── templates/             # sshd- & unattended-upgrades-Configs
│   └── handlers/main.yml
└── systemd/
    ├── ansible-pull.service
    └── ansible-pull.timer
```

## Einmaliges Setup pro VM

```bash
# 1. Repo-URL in systemd/ansible-pull.service anpassen (DEIN_ORG/server-config)

# 2. Ansible installieren
apt update && apt install -y ansible git

# 3. systemd-Units installieren
cp systemd/ansible-pull.service /etc/systemd/system/
cp systemd/ansible-pull.timer /etc/systemd/system/
systemctl daemon-reload
systemctl enable --now ansible-pull.timer

# 4. Ersten Lauf manuell anstoßen (optional, sonst läuft er nach OnBootSec)
systemctl start ansible-pull.service
journalctl -u ansible-pull.service -f
```

## Wichtig vor dem ersten Rollout

- **Vorher SSH-Key-Login testen**, bevor `ssh_password_authentication: "no"`
  greift – sonst sperrst du dich aus!

## Änderungen ausrollen

Einfach committen und pushen – jede VM zieht sich die neue Konfiguration
innerhalb von max. 30 Minuten selbst (Timer-Intervall in
`systemd/ansible-pull.timer` anpassbar).
