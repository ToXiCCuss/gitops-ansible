# server-config – Basis-GitOps-Grundgerüst

Ansible-Rolle "base" für Updates, Firewall (UFW), SSH-Hardening, Fail2ban
und automatische Sicherheitsupdates. Läuft per `ansible-pull` selbstständig
auf jeder VM (Pull-Modell, kein zentraler Ansible-Host nötig).

## Struktur

```
server-config/
├── inventory.ini              # Server-Liste (nur zur Doku, ansible-pull nutzt lokalen Host)
├── site.yml                   # Haupt-Playbook
├── requirements.yml           # externe Ansible-Collections
├── roles/base/
│   ├── defaults/main.yml      # anpassbare Variablen (Ports, Firewall-Regeln, ...)
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
- Firewall-Ports in `roles/base/defaults/main.yml` unter
  `firewall_allowed_tcp_ports` / `_udp_ports` ergänzen (z. B. FiveM-Port,
  Arcane-Port, DB-Port falls extern erreichbar).
- Bei Bedarf pro Host/Gruppe eigene Werte über `group_vars/<gruppe>.yml`
  überschreiben (z. B. andere Ports je nach VM).

## Änderungen ausrollen

Einfach committen und pushen – jede VM zieht sich die neue Konfiguration
innerhalb von max. 30 Minuten selbst (Timer-Intervall in
`systemd/ansible-pull.timer` anpassbar).
