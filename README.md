# gitops-ansible

Dieses Repository enthält Ansible-Playbooks zur Basiskonfiguration von Servern (aktuell: Rolle `base`). Diese Doku beschreibt, wie du Ansible unter Debian installierst und das Playbook ausführst.

## Voraussetzungen

- Ein Debian-System (lokal oder remote), auf dem Ansible ausgeführt wird (der "Control Node").
- Python 3 (auf Debian standardmäßig vorhanden).
- SSH-Zugriff auf die Zielhosts (Managed Nodes), falls nicht `localhost` verwendet wird.
- `sudo`/root-Rechte auf den Zielhosts, da die Playbooks mit `become: true` laufen.

## 1. Ansible unter Debian installieren

Auf Debian ist es empfehlenswert, Ansible über `apt` zu installieren (Paket `ansible` ist ab Debian 11/Bullseye im Standard-Repo enthalten, ansonsten Backports nutzen):

```bash
sudo apt update
sudo apt install -y ansible
```

Version prüfen:

```bash
ansible --version
```

Alternativ (aktuellere Version via `pip`):

```bash
sudo apt install -y python3-pip
pip3 install --user ansible
```

## 2. Repository klonen

```bash
git clone <repository-url> gitops-ansible
cd gitops-ansible
```

## 3. Externe Collections installieren

Das Projekt benötigt die in `requirements.yml` gelisteten Collections (`community.general`, `ansible.posix`). Diese vor der ersten Ausführung installieren:

```bash
ansible-galaxy collection install -r requirements.yml
```

## 4. Inventory anpassen

Die Datei `inventory.ini` definiert die Zielhosts. Standardmäßig ist sie auf die lokale Ausführung eingestellt:

```ini
[all]
localhost ansible_connection=local
```

Für die Verwaltung entfernter Debian-Server passt du die Datei entsprechend an, z. B.:

```ini
[all]
server1.example.com ansible_user=deploy
server2.example.com ansible_user=deploy
```

Falls du dich mit einem anderen Benutzer verbindest oder einen anderen SSH-Key nutzt, kannst du zusätzliche Variablen wie `ansible_ssh_private_key_file` ergänzen.

## 5. Playbook ausführen

Das Haupt-Playbook ist `site.yml` und wendet die Rolle `base` auf alle Hosts (`hosts: all`) an, mit `become: true` (also root-Rechte via `sudo`):

```bash
ansible-playbook -i inventory.ini site.yml
```

### Nützliche Optionen

- Trockenlauf ohne Änderungen (Check-Mode):

  ```bash
  ansible-playbook -i inventory.ini site.yml --check
  ```

- Detaillierte Ausgabe (Diff der Änderungen anzeigen):

  ```bash
  ansible-playbook -i inventory.ini site.yml --check --diff
  ```

- Nach `sudo`-Passwort fragen (falls kein passwortloses `sudo` konfiguriert ist):

  ```bash
  ansible-playbook -i inventory.ini site.yml --ask-become-pass
  ```

- Nur bestimmte Hosts ansprechen:

  ```bash
  ansible-playbook -i inventory.ini site.yml --limit server1.example.com
  ```

- Erreichbarkeit der Hosts vorab testen:

  ```bash
  ansible -i inventory.ini all -m ping
  ```

## Hinweis zur Python-Interpreter-Warnung

Beim Ausführen von `ansible-playbook` kann folgende Warnung erscheinen:

```
[WARNING]: Host 'localhost' is using the discovered Python interpreter at '/usr/bin/python3.13', but future
installation of another Python interpreter could cause a different interpreter to be discovered.
```

Das ist keine Fehlermeldung — der Task (z. B. `Gathering Facts`) läuft trotzdem erfolgreich durch (`ok: [localhost]`). Ansible weist nur darauf hin, dass der Python-Interpreter automatisch ermittelt wurde und sich das Ergebnis bei einer späteren zusätzlichen Python-Installation ändern könnte.

Um die Warnung dauerhaft zu unterdrücken bzw. den Interpreter fest zu definieren, liegt im Projekt eine `ansible.cfg` bei:

```ini
[defaults]
inventory = inventory.ini
interpreter_python = auto_silent
```

Alternativ kannst du den Interpreter explizit setzen, z. B. in `inventory.ini`:

```ini
[all]
localhost ansible_connection=local ansible_python_interpreter=/usr/bin/python3
```

## 6. Was die Rolle `base` konfiguriert

Die Rolle `roles/base` (siehe `roles/base/tasks/main.yml`) führt auf jedem Zielhost folgende Schritte aus:

- Installiert die Pakete `unattended-upgrades` und `btop`.
- Setzt die Zeitzone gemäß der Variable `server_timezone` (Standard: `Europe/Berlin`).
- Konfiguriert automatische Sicherheitsupdates (`unattended-upgrades`), sofern `unattended_upgrades_enabled: true` gesetzt ist.

### Standardwerte anpassen

Die Standardwerte findest du in `roles/base/defaults/main.yml`:

```yaml
server_timezone: "Europe/Berlin"
unattended_upgrades_enabled: true
unattended_upgrades_reboot: false
unattended_upgrades_reboot_time: "04:00"
```

Diese Variablen kannst du überschreiben, z. B. direkt beim Aufruf:

```bash
ansible-playbook -i inventory.ini site.yml -e "server_timezone=Europe/Vienna"
```

oder dauerhaft in `inventory.ini` als Host-/Group-Variablen bzw. in einer eigenen `group_vars`/`host_vars`-Struktur.

## Kurzübersicht (Quickstart)

```bash
# 1. Ansible installieren
sudo apt update && sudo apt install -y ansible

# 2. Collections installieren
ansible-galaxy collection install -r requirements.yml

# 3. Erreichbarkeit prüfen
ansible -i inventory.ini all -m ping

# 4. Playbook ausführen
ansible-playbook -i inventory.ini site.yml
```
