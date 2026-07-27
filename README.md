# gitops-ansible

Dieses Repository enthält Ansible-Playbooks zur Basiskonfiguration von Servern (aktuell: Rolle `base`). Diese Doku beschreibt, wie du Ansible unter Debian installierst und das Playbook ausführst.

## Voraussetzungen

- Ansible wird **direkt auf jeder VM lokal ausgeführt** (die VM ist gleichzeitig Control Node und Managed Node). Es gibt also keinen zentralen Rechner, der sich per SSH auf andere Hosts verbindet.
- Ein Debian-System auf der jeweiligen VM.
- Python 3 (auf Debian standardmäßig vorhanden).
- `sudo`/root-Rechte auf der VM, da das Playbook mit `become: true` läuft.

## 1. Ansible unter Debian installieren

Auf Debian ist es empfehlenswert, Ansible über `apt` zu installieren (Paket `ansible` ist ab Debian 11/Bullseye im Standard-Repo enthalten, ansonsten Backports nutzen):

```bash
sudo apt update
sudo apt install -y ansible git
```

Version prüfen:

```bash
ansible --version
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

Die Datei `inventory.ini` definiert die Zielhosts. Da Ansible auf jeder VM **lokal** ausgeführt wird, bleibt sie immer auf die lokale Ausführung eingestellt – das ist bereits der Standard und muss pro VM nicht geändert werden:

```ini
[all]
localhost ansible_connection=local
```

Dadurch verbindet sich Ansible nicht per SSH, sondern führt alle Tasks direkt auf der VM aus, auf der `ansible-playbook` gestartet wird. Das Repository (bzw. zumindest `inventory.ini`, `ansible.cfg`, `site.yml`, `requirements.yml` und `roles/`) muss also auf jede VM ausgecheckt/kopiert werden, auf der die Konfiguration angewendet werden soll.

> Falls du stattdessen von einem zentralen Rechner aus mehrere VMs per SSH verwalten möchtest, müsstest du hier echte Hostnamen/IPs statt `localhost` eintragen (siehe Abschnitt "Alternative: zentrale Verwaltung mehrerer VMs per SSH" unten) – für den hier beschriebenen Workflow (lokale Ausführung je VM) ist das aber nicht nötig.

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

- Erreichbarkeit lokal testen:

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

## Alternative: zentrale Verwaltung mehrerer VMs per SSH

Falls du den Workflow doch einmal ändern und Ansible von einem zentralen Control Node aus per SSH auf mehreren VMs ausführen möchtest, passt du `inventory.ini` so an:

```ini
[all]
server1.example.com ansible_user=deploy
server2.example.com ansible_user=deploy
```

Falls du dich mit einem anderen Benutzer verbindest oder einen anderen SSH-Key nutzt, kannst du zusätzliche Variablen wie `ansible_ssh_private_key_file` ergänzen. Mit `--limit server1.example.com` sprichst du dann gezielt einzelne Hosts an. Für den Standard-Workflow dieses Projekts (Ausführung direkt auf der jeweiligen VM) ist das aber nicht notwendig.

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

## 7. Node Exporter (Prometheus Metrics)

Die Rolle `roles/node_exporter` installiert den Prometheus Node Exporter auf jedem Zielhost über das Debian-Paket `prometheus-node-exporter` (verfügbar ab Debian 11/Bullseye):

- Installiert das Paket `prometheus-node-exporter` via `apt`.
- Das Paket legt automatisch einen dedizierten System-User an und liefert eine fertige `systemd`-Unit (`prometheus-node-exporter.service`).
- Der Service wird aktiviert und gestartet.
- Der Exporter lauscht standardmäßig auf Port `9100` (Metriken unter `http://<host>:9100/metrics`).

Die Rolle ist bereits in `site.yml` eingebunden und wird bei jedem Playbook-Lauf mit ausgeführt.

> Hinweis: Die über `apt` bereitgestellte Version ist an das jeweilige Debian-Release gebunden (ggf. etwas älter als die neueste GitHub-Release-Version), wird dafür aber automatisch über `apt upgrade` aktuell gehalten.

### Standardwerte anpassen (Node Exporter)

Siehe `roles/node_exporter/defaults/main.yml`:

```yaml
node_exporter_port: 9100
```

## 8. Docker (optional, nicht auf jedem Host)

Die Rolle `roles/docker` installiert Docker (`docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin`) über das offizielle Docker-Apt-Repository.

Im Gegensatz zu `base` und `node_exporter` wird diese Rolle **standardmäßig auf keinem Host ausgeführt**. Da dieses Projekt auf jeder VM lokal ausgeführt wird (`inventory_hostname` ist überall `localhost`), kann die Steuerung nicht über eine Inventory-Gruppe erfolgen – stattdessen wird sie über den **echten Systemhostnamen** (`ansible_hostname`) gesteuert, gepflegt in der Datei `group_vars/all.yml`, die ganz normal über Git versioniert wird:

```yaml
# site.yml
- name: Install Docker (only on hosts listed in docker_hosts)
  hosts: all
  become: true
  roles:
    - role: docker
      when: ansible_hostname in docker_hosts
```

```yaml
# group_vars/all.yml
docker_hosts: []
```

Da `group_vars/all.yml` Teil des Repositories ist, reicht ein einziger Commit/Push, um zentral festzulegen, auf welchen VMs Docker installiert wird – jede VM zieht beim nächsten `git pull` denselben Stand und wendet ihn lokal an.

### Docker auf einer VM aktivieren

Trage den **echten Hostnamen** der gewünschten VM (Ausgabe von `hostname` auf der jeweiligen VM) in `group_vars/all.yml` ein und committe die Änderung:

```yaml
# group_vars/all.yml
docker_hosts:
  - webserver01
  - ansible02
```

Anschließend auf der betreffenden VM:

```bash
git pull
ansible-playbook -i inventory.ini site.yml
```

Nur VMs, deren echter Hostname in `docker_hosts` steht, bekommen die Rolle `docker` zugewiesen; alle anderen VMs überspringen diesen Play (Task wird als `skipped` markiert).

Optional kannst du Benutzer per `docker_users` (Liste) automatisch zur Gruppe `docker` hinzufügen lassen, damit sie Docker ohne `sudo` nutzen können:

```yaml
docker_users:
  - deploy
```

### Docker-Daemon-Konfiguration (`/etc/docker/daemon.json`)

Die Rolle erstellt zusätzlich `/etc/docker/daemon.json` anhand der Variable `docker_daemon_options` (Standard in `roles/docker/defaults/main.yml`):

```yaml
docker_daemon_options:
  live-restore: true
```

Das entspricht:

```bash
nano /etc/docker/daemon.json
```

```json
{
  "live-restore": true
}
```

Bei einer Änderung dieser Datei wird der Docker-Dienst automatisch neu gestartet (Handler `restart docker`). Weitere Optionen kannst du einfach als zusätzliche Keys in `docker_daemon_options` ergänzen, z. B. per `host_vars`/`group_vars` oder direkt beim Aufruf mit `-e`.

## 9. Backup (restic + rclone)

Die Rolle `roles/backup_tools` installiert `restic` und `rclone`, deployt das restic-Repository-Passwort sowie optional eine `rclone.conf` und initialisiert bei Bedarf das restic-Repository. Die Rolle ist bereits in `site.yml` eingebunden.

**Wichtig:** Da Ansible auf jeder VM lokal läuft, wird auf **allen** Servern exakt dieselbe restic-/rclone-Konfiguration verwendet (Passwort, rclone-Remote, `restic_repo_base` usw.) – zentral gepflegt in `group_vars/all/main.yml` bzw. `group_vars/all/vault.yml`. Der **einzige** Unterschied zwischen den Servern ist der restic-Repository-Pfad: Dieser wird automatisch anhand des echten Systemhostnamens gebildet (`{{ restic_repo_base }}/{{ ansible_hostname }}`, siehe `roles/backup_tools/tasks/main.yml`), sodass jeder Server sein eigenes, isoliertes Backup-Repository unter demselben Remote bekommt, ohne dass pro Host etwas manuell konfiguriert werden muss.

### restic-Passwort erstellen

Das restic-Passwort (`restic_password`) muss **stark und zufällig** sein, da es das gesamte Backup-Repository absichert. Am einfachsten erzeugst du es lokal mit `openssl` oder `pwgen`:

```bash
# Variante 1: openssl (liefert z. B. 32 Byte, Base64-kodiert)
openssl rand -base64 32

# Variante 2: pwgen (falls installiert: sudo apt install -y pwgen)
pwgen -s 40 1
```

Kopiere dir die erzeugte Zeichenkette – sie wird im nächsten Schritt in die Vault-Datei eingetragen.

> Wichtig: Bewahre das Passwort zusätzlich an einem sicheren Ort außerhalb des Repositories auf (z. B. Passwort-Manager). Geht es verloren, sind die restic-Backups **nicht wiederherstellbar**.

### Passwort mit Ansible Vault verschlüsselt hinterlegen

Das Repository enthält eine Vorlage unter `group_vars/all/vault.yml.example`. So richtest du die echte, verschlüsselte Datei ein:

```bash
# 1. Vorlage kopieren
cp group_vars/all/vault.yml.example group_vars/all/vault.yml

# 2. Das generierte Passwort eintragen (restic_password: "...")
nano group_vars/all/vault.yml

# 3. Datei mit Ansible Vault verschlüsseln
ansible-vault encrypt group_vars/all/vault.yml
```

`group_vars/all/vault.yml` kann danach bedenkenlos committet werden – der Inhalt ist ohne Vault-Passwort unlesbar. Das Vault-Passwort selbst darf **nicht** ins Repository gelangen (siehe `.gitignore`).

Beim Ausführen des Playbooks muss das Vault-Passwort angegeben werden:

```bash
# interaktiv abfragen
ansible-playbook -i inventory.ini site.yml --ask-vault-pass

# oder aus einer lokalen, nicht versionierten Datei lesen
ansible-playbook -i inventory.ini site.yml --vault-password-file ~/.vault_pass.txt
```

### rclone-Konfiguration (pCloud-Remote)

Der OAuth-Login gegen pCloud lässt sich nicht automatisieren und muss einmalig manuell per `rclone config` durchgeführt werden. Die daraus entstehende `rclone.conf` legst du anschließend (idealerweise ebenfalls vault-verschlüsselt) ins Repo unter `roles/backup_tools/files/rclone.conf`. `rclone_conf_src` zeigt in `group_vars/all/main.yml` bereits standardmäßig auf diesen Pfad, sodass alle Server automatisch dieselbe `rclone.conf` erhalten – eine host-spezifische Anpassung ist nicht nötig.

### Weitere Variablen

Siehe `roles/backup_tools/defaults/main.yml`:

```yaml
restic_password: ""
restic_repo_base: "rclone:pCloud:/Backups"
rclone_remote_name: "pCloud"
rclone_conf_src: ""
rclone_conf_dest: "/root/.config/rclone/rclone.conf"
```

Solange `restic_password` bzw. `rclone_conf_src` leer sind, werden die zugehörigen Tasks (Passwortdatei, rclone-Config-Deployment, Repository-Init) übersprungen – nur die Installation von `restic`/`rclone` per `apt` erfolgt dann.

## Kurzübersicht (Quickstart)

Auf jeder VM, die konfiguriert werden soll:

```bash
# 0. Repository auf die VM holen
git clone <repository-url> gitops-ansible && cd gitops-ansible

# 1. Ansible installieren
sudo apt update && sudo apt install -y ansible

# 2. Collections installieren
ansible-galaxy collection install -r requirements.yml

# 3. Erreichbarkeit prüfen (lokal)
ansible -i inventory.ini all -m ping

# 4. Playbook lokal ausführen
ansible-playbook -i inventory.ini site.yml
```
