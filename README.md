# 🎮 Minecraft Bedrock Server

**Einrichtungsanleitung für Proxmox LXC auf Ubuntu 24.04 LTS**

Version 1.26.14.1 | Stand: April 2026

---

## Voraussetzungen

- Proxmox mit einem **Ubuntu 24.04 LTS**-LXC-Template (Noble Numbat)
- LXC RAM: mindestens 1 GB (empfohlen 2 GB)
- LXC Disk: mindestens 2 GB (empfohlen 4 GB – das Auto-Update legt kurzzeitig eine vollständige Kopie des Server-Ordners als Rollback an)
- „Start at boot" im LXC aktiviert (für automatischen Neustart nach Stromausfall)

> **Info:** `features: nesting=1` ist bei aktuellen Proxmox-Templates meist bereits voreingestellt.

> **Warum Ubuntu 24.04 LTS?** Es ist die von Mojang offiziell unterstützte Plattform (Bedrock verlangt Ubuntu 22.04 LTS oder neuer), ausgereift und bekommt regulär bis April 2029 Sicherheitsupdates – mit dem für Privatnutzung kostenlosen Ubuntu Pro sogar deutlich länger. Damit entfällt das Bibliotheks-Risiko, das es auf nicht unterstützten Distributionen geben kann.

---

## Schritt 0 – Uhrzeit & Zeitzone prüfen

Bevor der Server eingerichtet wird, sollte die Systemzeit des LXC überprüft werden. Eine falsche Zeitzone kann dazu führen, dass Backups zu falschen Zeiten laufen oder Log-Einträge falsche Zeitstempel haben.

### Aktuelle Zeit & Zeitzone anzeigen

```bash
timedatectl
```

Beispielausgabe:

```
               Local time: Fri 2026-04-17 08:00:00 CEST
           Universal time: Fri 2026-04-17 06:00:00 UTC
                Time zone: Europe/Berlin (CEST, +0200)
System clock synchronized: yes
              NTP service: active
```

### Zeitzone ändern (z.B. auf Deutschland)

```bash
timedatectl set-timezone Europe/Berlin
```

Andere verfügbare Zeitzonen auflisten:

```bash
timedatectl list-timezones | grep Europe
```

### NTP-Synchronisation prüfen

Falls `System clock synchronized: no` erscheint:

```bash
apt install -y systemd-timesyncd
timedatectl set-ntp true
timedatectl
```

> **Info:** Im LXC übernimmt Proxmox die Hardwareuhr vom Host. Es reicht meistens die Zeitzone korrekt zu setzen.

---

## Schritt 1 – System aktualisieren & Abhängigkeiten installieren

Als root im LXC ausführen:

```bash
apt update && apt upgrade -y
apt install -y curl unzip libcurl4t64 libssl3t64 screen ufw jq
```

> **Info zu den Paketnamen:** Ubuntu 24.04 hat einige Bibliotheken im Zuge der 64-bit-time_t-Umstellung umbenannt – daher `libcurl4t64` und `libssl3t64` (nicht `libcurl4`/`libssl3`). Falls `apt` ein Paket „nicht gefunden" meldet (z.B. auf einer neueren Ubuntu-Version, wo das `t64`-Suffix wieder wegfällt), einfach die Variante ohne `t64` nehmen: `apt install -y libcurl4 libssl3`.

> **Info:** `jq` wird für das Auto-Update benötigt (zum Auslesen der Download-URL aus der Mojang-API).

---

## Schritt 2 – Benutzer anlegen

Aus Sicherheitsgründen wird der Server unter einem eigenen Benutzer betrieben:

```bash
useradd -m -r -s /bin/bash minecraft
su - minecraft
```

---

## Schritt 3 – Server herunterladen & entpacken

Den Bedrock-Server herunterladen. Der Minecraft-Server blockiert Downloads ohne Browser-User-Agent, daher muss dieser mitgegeben werden:

```bash
mkdir -p ~/bedrock && cd ~/bedrock

curl -Lo bedrock-server.zip \
  -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" \
  https://www.minecraft.net/bedrockdedicatedserver/bin-linux/bedrock-server-1.26.14.1.zip

unzip bedrock-server.zip
rm bedrock-server.zip
chmod +x bedrock_server

# Installierte Version festhalten (wird vom Auto-Update-Script aus Schritt 11 verwendet)
echo "1.26.14.1" > ~/bedrock/.installed_version
```

> **Info:** Falls der Download fehlschlägt: Datei auf dem PC herunterladen und per scp übertragen: `scp bedrock-server.zip minecraft@<LXC-IP>:~/bedrock/`

---

## Schritt 4 – Erster Teststart

Vor der systemd-Einrichtung testen ob der Server startet:

```bash
LD_LIBRARY_PATH=. ./bedrock_server
```

Wenn `Server started.` erscheint läuft alles korrekt. Mit `Ctrl+C` beenden.

> **Info:** Den manuellen Testprozess vollständig beenden bevor systemd gestartet wird – sonst ist Port 19132 bereits belegt!

---

## Schritt 4b – (Optional) Bestehende Welt von einem alten Server übertragen

Wenn du von einem bestehenden Bedrock-Server (z.B. dem alten Debian-LXC) umziehst, kannst du Welt **und** Konfiguration einfach mitnehmen. Da ein Backup ohnehin alle wichtigen Teile enthält (Welt, `server.properties`, `permissions.json`, `allowlist.json`), ist ein Backup-Tarball das perfekte Transportmittel.

**Auf dem ALTEN Server** – ein frisches Backup erstellen (oder das letzte nächtliche nehmen):

```bash
/usr/local/bin/minecraft-backup.sh
ls -lh /home/minecraft/backups/
```

**Den Tarball auf den NEUEN Server kopieren** (vom alten Server aus, IP des neuen LXC anpassen):

```bash
scp /home/minecraft/backups/minecraft-backup-2026-04-17_03-00.tar.gz \
  minecraft@<NEUE-LXC-IP>:/home/minecraft/
```

**Auf dem NEUEN Server** – als root einspielen:

```bash
# Falls der Dienst aus Schritt 6 schon läuft, vorher stoppen:
systemctl stop minecraft-bedrock 2>/dev/null

# Tarball entpacken – überschreibt Welt + Konfiguration im bedrock-Ordner
tar -xzf /home/minecraft/minecraft-backup-2026-04-17_03-00.tar.gz \
  -C /home/minecraft/bedrock

# Eigentümer korrigieren (Dateien gehören jetzt minecraft)
chown -R minecraft:minecraft /home/minecraft/bedrock
rm /home/minecraft/minecraft-backup-*.tar.gz
```

> **Wichtig – Weltname:** In der `server.properties` muss `level-name=` **exakt** auf den Ordnernamen unter `worlds/` zeigen, sonst legt der Server eine leere neue Welt an statt deine zu laden. Prüfen mit `ls /home/minecraft/bedrock/worlds/` und den Namen in Schritt 5 entsprechend setzen.

> **Vorsicht – Weltformat:** Beim ersten Start kann eine ältere Welt automatisch auf das neue Format aktualisiert werden. Das ist ein Einbahnstraßen-Vorgang. Bewahre den ursprünglichen Backup-Tarball also auf, bis du sicher bist, dass alles läuft.

> **Hinweis:** Da die Welt von der Xbox-Identität (XUID) der Spieler abhängt und nicht vom Server, bleiben Inventare, Bauten und Fortschritt erhalten – die Spieler sehen ihre Welt genau wie vorher.

---

## Schritt 5 – Konfiguration anpassen

Den minecraft-Benutzer verlassen (`exit`) und die server.properties bearbeiten:

```bash
nano /home/minecraft/bedrock/server.properties
```

Folgende Einstellungen anpassen:

| Einstellung | Beschreibung |
|---|---|
| `server-name=Weber Creative` | Name des Servers im Spiel |
| `gamemode=adventure` | **Standard-Modus für neue Spieler** – im Adventure-Modus kann nichts abgebaut/platziert werden |
| `difficulty=peaceful` | peaceful / easy / normal / hard – peaceful = keine Gegner, kein Hunger |
| `allow-cheats=true` | Nötig, damit Operatoren im Spiel `/gamemode` & Co. nutzen können |
| `force-gamemode=false` | **Wichtig!** Spieler behalten ihren individuell zugewiesenen Modus über Reconnects hinweg |
| `max-players=5` | Maximale Spieleranzahl |
| `online-mode=true` | Xbox-Authentifizierung (empfohlen!) |
| `level-name=Bedrock level` | Weltname (= Ordnername unter worlds/) – bei einer übertragenen Welt entsprechend anpassen! |

Mit `Ctrl+X` → `Y` → `Enter` speichern.

### So funktioniert das „kontrollierte Creative"-Prinzip

Damit niemand versehentlich (oder absichtlich) die Welt kaputt macht, starten **alle** Spieler im **Adventure-Modus** (kein Bauen/Abbauen). Wer bauen darf, wird einzeln in den **Creative-Modus** freigeschaltet:

- **Über die Server-Konsole** (siehe Wartungsbereich, ohne `/`):
  ```
  gamemode creative DeinXboxGamertag
  ```
- **Oder im Spiel als Operator** (mit `/`):
  ```
  /gamemode creative SpielerName
  ```

Weil `force-gamemode=false` gesetzt ist, **bleibt** ein freigeschalteter Spieler auch nach dem nächsten Login im Creative-Modus. Stünde der Wert auf `true`, würde jeder beim Verbinden wieder auf den Standard-Modus (Adventure) zurückgesetzt – das wollen wir hier ausdrücklich nicht.

> **Tipp:** Wer einem Spieler die Baurechte wieder entziehen will, setzt ihn einfach zurück: `gamemode adventure SpielerName`.

---

## Schritt 6 – systemd-Service einrichten

Als root eine systemd-Service-Datei erstellen:

```bash
nano /etc/systemd/system/minecraft-bedrock.service
```

Inhalt der Datei:

```ini
[Unit]
Description=Minecraft Bedrock Server
After=network.target

[Service]
User=minecraft
WorkingDirectory=/home/minecraft/bedrock
ExecStart=/bin/bash -c 'LD_LIBRARY_PATH=. ./bedrock_server'
ExecStop=/bin/kill -SIGINT $MAINPID
Restart=on-failure
RestartSec=10
StandardInput=null

[Install]
WantedBy=multi-user.target
```

Service aktivieren und starten:

```bash
systemctl daemon-reload
systemctl enable minecraft-bedrock
systemctl start minecraft-bedrock
systemctl status minecraft-bedrock
```

Bei `Server started.` in den Logs läuft alles korrekt.

> **Info:** Falls `Port may be in use` erscheint: `pkill -f bedrock_server` ausführen und dann erneut starten.

---

## Schritt 7 – Firewall einrichten (ufw)

Wichtig wenn der Server aus dem Internet erreichbar sein soll:

```bash
# Erst SSH sichern damit du nicht ausgesperrt wirst!
ufw allow 22/tcp

# Minecraft Port freigeben
ufw allow 19132/udp

# Firewall aktivieren
ufw enable

ufw status
```

---

## Schritt 8 – Port-Forwarding am Router

Damit der Server aus dem Internet erreichbar ist, muss am Router eine Weiterleitung eingerichtet werden:

| Einstellung | Wert |
|---|---|
| Protokoll | UDP |
| Externer Port | 19132 |
| Interner Port | 19132 |
| Ziel-IP | IP des LXC (ermitteln mit: `ip a \| grep inet`) |

> **Damit die Weiterleitung nicht ins Leere zeigt:** Der LXC braucht eine **feste IP** – entweder im LXC direkt statisch konfiguriert oder per DHCP-Reservierung im Router. Sonst kann sich die IP ändern und das Port-Forwarding bricht. Mehr dazu in Schritt 13.

> **Hinweis bei mehreren Servern:** Ports sind immer an eine IP gebunden. Wenn jeder Server in einem eigenen LXC mit eigener IP läuft, können alle auf Port 19132 lauschen – es gibt keine Kollision. Am Router muss dann für jeden Server ein eigener externer Port eingerichtet werden, der auf die jeweilige LXC-IP zeigt.

---

## Schritt 9 – Automatisches Backup

Ein tägliches Backup-Script erstellen, das 7 Tage aufbewahrt. **Gesichert werden alle wirklich wichtigen Teile** – nicht nur die Welt, sondern auch die Konfiguration, Operator-Rechte und die Allowlist. Damit lässt sich ein Server aus einem einzigen Backup vollständig wiederherstellen (und genau dieses Tarball eignet sich auch zum Umzug, siehe Schritt 4b).

| Im Backup enthalten | Warum |
|---|---|
| `worlds/` | Die eigentliche(n) Welt(en) |
| `server.properties` | Gesamte Server-Konfiguration |
| `permissions.json` | Operator-Rechte (OP) |
| `allowlist.json` | Allowlist/Whitelist (falls verwendet) |

> **Hinweis:** `valid_known_packs.json` und die mitgelieferten `behavior_packs`/`resource_packs` werden beim Server-Update ohnehin neu geschrieben und müssen nicht gesichert werden. Hast du **eigene** Add-ons/Packs installiert, ergänze deren Ordner in der `ITEMS`-Liste unten.

```bash
nano /usr/local/bin/minecraft-backup.sh
```

Inhalt:

```bash
#!/bin/bash
set -u

BEDROCK_DIR="/home/minecraft/bedrock"
BACKUP_DIR="/home/minecraft/backups"
NOTIFY="/usr/local/bin/minecraft-notify.sh"
MAINT_FLAG="/run/minecraft-bedrock.maintenance"
SERVICE="minecraft-bedrock"
KEEP_DAYS=7

DATE=$(date +%Y-%m-%d_%H-%M)
BACKUP_FILE="$BACKUP_DIR/minecraft-backup-$DATE.tar.gz"
mkdir -p "$BACKUP_DIR"

# Wartungs-Flag nur setzen, wenn nicht schon eines existiert (z.B. vom Update-Script).
# Verhindert, dass der Monitor während des kurzen Backup-Stopps Fehlalarm auslöst.
OWN_FLAG=false
if [ ! -e "$MAINT_FLAG" ]; then touch "$MAINT_FLAG"; OWN_FLAG=true; fi
cleanup(){ [ "$OWN_FLAG" = true ] && rm -f "$MAINT_FLAG"; }
trap cleanup EXIT

# Nur tatsächlich vorhandene Teile sichern
ITEMS=()
for f in worlds server.properties permissions.json allowlist.json; do
  [ -e "$BEDROCK_DIR/$f" ] && ITEMS+=("$f")
done

WAS_ACTIVE=false
if systemctl is-active --quiet "$SERVICE"; then
  WAS_ACTIVE=true
  systemctl stop "$SERVICE"
fi

tar -czf "$BACKUP_FILE" -C "$BEDROCK_DIR" "${ITEMS[@]}"
STATUS=$?

[ "$WAS_ACTIVE" = true ] && systemctl start "$SERVICE"

# Alte Backups aufräumen
find "$BACKUP_DIR" -name "minecraft-backup-*.tar.gz" -mtime +$KEEP_DAYS -delete

if [ $STATUS -eq 0 ]; then
  echo "$(date '+%F %T') Backup erstellt: $BACKUP_FILE ($(du -h "$BACKUP_FILE" | cut -f1))"
else
  echo "$(date '+%F %T') Backup FEHLGESCHLAGEN"
  [ -x "$NOTIFY" ] && "$NOTIFY" "Minecraft: Backup fehlgeschlagen" "high" "warning" \
    "Das Backup ($DATE) ist fehlgeschlagen. Bitte prüfen: /var/log/minecraft-backup.log"
fi
exit $STATUS
```

Script ausführbar machen:

```bash
chmod +x /usr/local/bin/minecraft-backup.sh
```

> **Info:** Routine-Backups melden sich **nicht** über ntfy (sonst gäbe es täglich eine Push-Nachricht). Es kommt nur dann eine Meldung, wenn ein Backup **fehlschlägt**. Der Cronjob für dieses Script wird gesammelt in Schritt 11 eingerichtet.

Script manuell testen:

```bash
/usr/local/bin/minecraft-backup.sh
ls -lh /home/minecraft/backups/
```

---

## Schritt 10 – Benachrichtigungen über ntfy einrichten

Alle Scripts (Update, Monitoring, Backup) melden Erfolg bzw. Fehler über einen kleinen Helfer an **ntfy.sh**. So gibt es eine zentrale Stelle, an der die ntfy-Adresse hinterlegt ist.

```bash
nano /usr/local/bin/minecraft-notify.sh
```

Inhalt:

```bash
#!/bin/bash
# Aufruf: minecraft-notify.sh "Titel" "priorität" "tags" "Nachricht"
# priorität: min | low | default | high | urgent | max
# tags:      kommagetrennte ntfy-Emoji-Codes, z.B. white_check_mark,warning
set -u

NTFY_URL="https://ntfy.sh/online-weber-minecraft-bedrock"

TITLE="${1:-Minecraft}"
PRIO="${2:-default}"
TAGS="${3:-}"
MSG="${4:-}"

curl -fsS --max-time 15 \
  -H "Title: $TITLE" \
  -H "Priority: $PRIO" \
  -H "Tags: $TAGS" \
  -d "$MSG" \
  "$NTFY_URL" >/dev/null 2>&1
```

Ausführbar machen und testen:

```bash
chmod +x /usr/local/bin/minecraft-notify.sh
/usr/local/bin/minecraft-notify.sh "Minecraft: Test" "default" "white_check_mark" "Benachrichtigungen funktionieren ✅"
```

Auf dem Handy die **ntfy-App** (iOS/Android) installieren und das Thema `online-weber-minecraft-bedrock` abonnieren – dann kommen Meldungen als Push-Nachricht.

> **⚠️ Sicherheitshinweis:** Ein öffentliches ntfy.sh-Thema kann von **jedem** gelesen und beschrieben werden, der den Namen kennt. Der Name `online-weber-minecraft-bedrock` ist relativ leicht zu erraten. Für etwas mehr Privatsphäre siehe Schritt 13 (zufälliger Themenname oder Access-Token).

---

## Schritt 11 – Automatisches Update mit Rollback

Dieses Script läuft per Cron und kümmert sich um **alle relevanten Teile**:

1. **Ubuntu** aktualisieren (`apt update && apt upgrade`)
2. **Bedrock-Server** auf die neueste Version aktualisieren (Version wird über die offizielle Mojang-API ermittelt)
3. Vor dem Server-Update wird **automatisch ein Backup** gemacht und zusätzlich ein **vollständiger Rollback-Snapshot** des Server-Ordners angelegt
4. Startet die neue Version **nicht**, wird automatisch die **alte Version zurückgespielt**
5. Bei **Erfolg** und bei **Fehler** kommt eine **ntfy-Nachricht**

```bash
nano /usr/local/bin/minecraft-update.sh
```

Inhalt:

```bash
#!/bin/bash
set -u

### Konfiguration ###
BEDROCK_DIR="/home/minecraft/bedrock"
ROLLBACK_DIR="/home/minecraft/bedrock.rollback"
VERSION_FILE="$BEDROCK_DIR/.installed_version"
BACKUP_SCRIPT="/usr/local/bin/minecraft-backup.sh"
NOTIFY="/usr/local/bin/minecraft-notify.sh"
MAINT_FLAG="/run/minecraft-bedrock.maintenance"
SERVICE="minecraft-bedrock"
API_URL="https://net-secondary.web.minecraft-services.net/api/v1.0/download/links"
UA="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
START_TIMEOUT=120   # Sekunden, die auf "Server started." gewartet wird

log(){ echo "$(date '+%F %T') [update] $*"; }

# Wartungs-Flag setzen, damit der Monitor während des Updates keinen Alarm schlägt
touch "$MAINT_FLAG"
cleanup(){ rm -f "$MAINT_FLAG"; }
trap cleanup EXIT

APT_FAILED=false

### 1) Ubuntu aktualisieren ###
log "APT-Update..."
if ! DEBIAN_FRONTEND=noninteractive apt-get update -y >/tmp/mc-apt.log 2>&1 \
   || ! DEBIAN_FRONTEND=noninteractive apt-get -y upgrade >>/tmp/mc-apt.log 2>&1; then
  APT_FAILED=true
  log "APT-Upgrade FEHLGESCHLAGEN"
  "$NOTIFY" "Minecraft: Ubuntu-Update fehlgeschlagen" "high" "warning" \
    "Das System-Update (apt) ist fehlgeschlagen. Details auf dem Server: /tmp/mc-apt.log"
fi
DEBIAN_FRONTEND=noninteractive apt-get -y autoremove >>/tmp/mc-apt.log 2>&1

### 2) Neueste Bedrock-Version über die Mojang-API ermitteln ###
log "Frage neueste Bedrock-Version ab..."
LINKS_JSON=$(curl -fsS --max-time 30 -A "$UA" "$API_URL" 2>/dev/null)
DOWNLOAD_URL=$(printf '%s' "$LINKS_JSON" \
  | jq -r '.result.links[] | select(.downloadType=="serverBedrockLinux") | .downloadUrl' 2>/dev/null)

if [ -z "${DOWNLOAD_URL:-}" ] || [ "$DOWNLOAD_URL" = "null" ]; then
  log "Download-URL konnte nicht ermittelt werden."
  "$NOTIFY" "Minecraft: Update-Check fehlgeschlagen" "high" "warning" \
    "Die neueste Version konnte nicht abgefragt werden (API/Netzwerk). Server läuft unverändert weiter."
  exit 1
fi

LATEST_VERSION=$(printf '%s' "$DOWNLOAD_URL" | sed -E 's#.*/bedrock-server-([0-9.]+)\.zip#\1#')
INSTALLED_VERSION=$(cat "$VERSION_FILE" 2>/dev/null || echo "unbekannt")
log "Installiert: $INSTALLED_VERSION  /  Verfügbar: $LATEST_VERSION"

# Schon aktuell?
if [ "$INSTALLED_VERSION" = "$LATEST_VERSION" ]; then
  log "Bereits aktuell – kein Server-Update nötig."
  exit 0
fi
# Sicherstellen, dass die "neueste" wirklich neuer ist (kein versehentliches Downgrade)
if [ "$INSTALLED_VERSION" != "unbekannt" ]; then
  HIGHEST=$(printf '%s\n%s\n' "$INSTALLED_VERSION" "$LATEST_VERSION" | sort -V | tail -1)
  if [ "$HIGHEST" = "$INSTALLED_VERSION" ]; then
    log "Installierte Version ist nicht älter – kein Update."
    exit 0
  fi
fi

### 3) Update mit Rollback ###
log "Update von $INSTALLED_VERSION auf $LATEST_VERSION."

# 3a) Backup (Welten + Konfiguration)
if ! "$BACKUP_SCRIPT"; then
  "$NOTIFY" "Minecraft: Update abgebrochen" "high" "warning" \
    "Backup vor dem Update fehlgeschlagen – Update wurde NICHT durchgeführt."
  exit 1
fi

# 3b) Vollständiger Rollback-Snapshot (Binaries + Welten + Konfig)
log "Erstelle Rollback-Snapshot..."
rm -rf "$ROLLBACK_DIR"
cp -a "$BEDROCK_DIR" "$ROLLBACK_DIR"

# Funktion: kompletten Rollback durchführen
rollback(){
  log "Rollback wird durchgeführt..."
  systemctl stop "$SERVICE" 2>/dev/null
  rm -rf "$BEDROCK_DIR"
  mv "$ROLLBACK_DIR" "$BEDROCK_DIR"
  systemctl start "$SERVICE"
  sleep 10
}

# 3c) Server stoppen
systemctl stop "$SERVICE"

# 3d) Download + ZIP-Integrität prüfen
log "Lade neue Version herunter..."
TMP_ZIP="/home/minecraft/bedrock-server-new.zip"
if ! curl -fsSL --max-time 300 -A "$UA" -o "$TMP_ZIP" "$DOWNLOAD_URL" \
   || ! unzip -tq "$TMP_ZIP" >/dev/null 2>&1; then
  log "Download/ZIP defekt."
  rm -f "$TMP_ZIP"
  rollback
  "$NOTIFY" "Minecraft: Update fehlgeschlagen 🚨" "urgent" "rotating_light" \
    "Download von $LATEST_VERSION fehlgeschlagen. Alte Version $INSTALLED_VERSION wurde wiederhergestellt und läuft."
  exit 1
fi

# Konfig sichern, entpacken, Konfig zurückspielen (überschreibt mitgelieferte Defaults)
for f in server.properties permissions.json allowlist.json; do
  [ -e "$BEDROCK_DIR/$f" ] && cp -a "$BEDROCK_DIR/$f" "/tmp/mc-$f.keep"
done

cd "$BEDROCK_DIR" || { rollback; exit 1; }
unzip -o "$TMP_ZIP" >/dev/null
rm -f "$TMP_ZIP"
chmod +x bedrock_server

for f in server.properties permissions.json allowlist.json; do
  [ -e "/tmp/mc-$f.keep" ] && cp -a "/tmp/mc-$f.keep" "$BEDROCK_DIR/$f" && rm -f "/tmp/mc-$f.keep"
done
chown -R minecraft:minecraft "$BEDROCK_DIR"

# 3e) Starten & verifizieren
START_EPOCH=$(date +%s)
systemctl start "$SERVICE"
log "Warte auf 'Server started.' (max ${START_TIMEOUT}s)..."

OK=false
for i in $(seq 1 "$START_TIMEOUT"); do
  if journalctl -u "$SERVICE" --since "@$START_EPOCH" 2>/dev/null | grep -q "Server started."; then
    OK=true; break
  fi
  # Wenn der Dienst gar nicht mehr läuft, früher abbrechen
  if ! systemctl is-active --quiet "$SERVICE" && [ "$i" -gt 15 ]; then break; fi
  sleep 1
done

if [ "$OK" = true ]; then
  echo "$LATEST_VERSION" > "$VERSION_FILE"
  chown minecraft:minecraft "$VERSION_FILE"
  rm -rf "$ROLLBACK_DIR"
  log "Update erfolgreich."
  EXTRA=""; [ "$APT_FAILED" = true ] && EXTRA=" (Hinweis: apt-Upgrade hatte zuvor einen Fehler.)"
  "$NOTIFY" "Minecraft: Update erfolgreich ✅" "default" "white_check_mark,package" \
    "Bedrock wurde von $INSTALLED_VERSION auf $LATEST_VERSION aktualisiert und läuft.$EXTRA"
  exit 0
else
  log "Neue Version startet nicht."
  rollback
  if systemctl is-active --quiet "$SERVICE"; then
    "$NOTIFY" "Minecraft: Update fehlgeschlagen 🚨" "urgent" "rotating_light" \
      "Version $LATEST_VERSION startete nicht. Alte Version $INSTALLED_VERSION wurde wiederhergestellt und läuft wieder."
  else
    "$NOTIFY" "Minecraft: Update fehlgeschlagen – manueller Eingriff nötig ‼️" "max" "rotating_light,sos" \
      "Version $LATEST_VERSION startete nicht UND der Rollback auf $INSTALLED_VERSION läuft ebenfalls nicht. Bitte sofort prüfen: journalctl -u $SERVICE -n 80"
  fi
  exit 1
fi
```

Ausführbar machen:

```bash
chmod +x /usr/local/bin/minecraft-update.sh
```

> **Info:** Reine Routine-Checks (es gibt kein neues Update) lösen **keine** Nachricht aus. Eine Meldung kommt nur bei erfolgreichem Update, bei einem Fehler oder wenn ein Update gar nicht erst geprüft werden konnte.

> **Hinweis zu Ubuntus Auto-Updates:** Ubuntu Server bringt `unattended-upgrades` für Sicherheits-Patches mit. Das ist harmlos, überschneidet sich aber mit dem `apt upgrade` dieses Scripts. Falls du im Update-Log mal „could not get lock"-Fehler siehst, kannst du `unattended-upgrades` deaktivieren (`systemctl disable --now unattended-upgrades`) – dieses Script übernimmt die System-Updates dann komplett.

---

## Schritt 12 – Server-Überwachung (Monitoring)

Ein kleines Watchdog-Script prüft regelmäßig, ob der Server läuft. Falls nicht, versucht es **einen Neustart** und meldet den Status. **Es gibt keine geplanten nächtlichen Neustarts** – das Script greift nur ein, wenn der Server tatsächlich down ist.

Damit es keine Push-Flut gibt:
- Während Backup/Update (Wartungs-Flag gesetzt) wird **nicht** alarmiert.
- Über eine Status-Datei wird nur bei **Zustandswechsel** gemeldet (down → online, online → down).
- Wiederholte automatische Neustarts werden auf max. eine Meldung pro 30 Minuten begrenzt.

```bash
nano /usr/local/bin/minecraft-monitor.sh
```

Inhalt:

```bash
#!/bin/bash
set -u

SERVICE="minecraft-bedrock"
NOTIFY="/usr/local/bin/minecraft-notify.sh"
MAINT_FLAG="/run/minecraft-bedrock.maintenance"
STATE_DIR="/var/lib/minecraft"
STATE_FILE="$STATE_DIR/monitor.state"
RESTART_MARK="$STATE_DIR/last_restart_notice"
mkdir -p "$STATE_DIR"

# Während Wartung (Backup/Update) nicht alarmieren – Flag nur gültig, wenn < 60 min alt
if [ -e "$MAINT_FLAG" ]; then
  if [ -n "$(find "$MAINT_FLAG" -mmin -60 2>/dev/null)" ]; then
    exit 0
  else
    rm -f "$MAINT_FLAG"   # verwaistes Flag aufräumen
  fi
fi

PREV=$(cat "$STATE_FILE" 2>/dev/null || echo "up")

if systemctl is-active --quiet "$SERVICE"; then
  if [ "$PREV" = "down" ]; then
    "$NOTIFY" "Minecraft: Server wieder online ✅" "default" "white_check_mark" \
      "Der Server läuft wieder."
  fi
  echo "up" > "$STATE_FILE"
  exit 0
fi

# Server ist NICHT aktiv -> Reparaturversuch
systemctl reset-failed "$SERVICE" 2>/dev/null
systemctl start "$SERVICE" 2>/dev/null
sleep 15

if systemctl is-active --quiet "$SERVICE"; then
  # Neustart geklappt – höchstens 1 Meldung pro 30 min (gegen "Flattern")
  if [ -z "$(find "$RESTART_MARK" -mmin -30 2>/dev/null)" ]; then
    "$NOTIFY" "Minecraft: Server war down – neu gestartet ⚠️" "high" "warning" \
      "Der Server war nicht aktiv und wurde automatisch neu gestartet. Er läuft jetzt wieder."
    touch "$RESTART_MARK"
  fi
  echo "up" > "$STATE_FILE"
else
  # Neustart fehlgeschlagen – nur beim Wechsel nach "down" melden
  if [ "$PREV" != "down" ]; then
    "$NOTIFY" "Minecraft: Server DOWN 🚨" "urgent" "rotating_light" \
      "Der Server ist nicht aktiv und der automatische Neustart ist fehlgeschlagen. Bitte prüfen: journalctl -u $SERVICE -n 50"
  fi
  echo "down" > "$STATE_FILE"
fi
```

Ausführbar machen:

```bash
chmod +x /usr/local/bin/minecraft-monitor.sh
```

### Cronjobs einrichten

Alle Zeitsteuerungen gesammelt im root-Crontab eintragen:

```bash
crontab -e
```

Folgende Zeilen hinzufügen:

```
# Server-Überwachung alle 5 Minuten
*/5 * * * * /usr/local/bin/minecraft-monitor.sh >> /var/log/minecraft-monitor.log 2>&1

# Tägliches Backup um 03:00 Uhr
0 3 * * * /usr/local/bin/minecraft-backup.sh >> /var/log/minecraft-backup.log 2>&1

# Tägliches Update (Ubuntu + Bedrock) um 04:30 Uhr
30 4 * * * /usr/local/bin/minecraft-update.sh >> /var/log/minecraft-update.log 2>&1
```

> **Info:** Das Update läuft **nach** dem Backup. Findet das Update eine neue Version, erstellt es ohnehin direkt davor noch ein eigenes Backup – doppelt hält besser.

---

## Schritt 13 – Weitere Empfehlungen & Ideen

Optionale Verbesserungen, die sich für deinen Aufbau anbieten:

**1. Feste IP für den LXC (wichtig fürs Port-Forwarding).**
Bekommt der Container per DHCP eine neue IP, zeigt die Router-Weiterleitung ins Leere. Entweder im LXC eine statische IP konfigurieren (Proxmox → Container → Network → static) oder im Router eine DHCP-Reservierung auf die MAC-Adresse des LXC anlegen.

**2. Dynamisches DNS (DDNS) für Freunde von außerhalb.**
Viele Privatanschlüsse bekommen regelmäßig eine neue öffentliche IP. Wer den Server per fester IP eingetragen hat, kommt dann nicht mehr rein. Lösung: ein DDNS-Name (z.B. DuckDNS, oder die DynDNS-Funktion im Router – bei einer FRITZ!Box „MyFRITZ!"). Im Minecraft-Client dann statt der IP den Hostnamen + Port 19132 eintragen.

**3. Erreichbarkeit prüfen – Stichwort CGNAT / DS-Lite.**
Manche Anschlüsse (in Deutschland v.a. einige Kabel-/Glasfaser-Tarife) haben **keine eigene öffentliche IPv4** (DS-Lite/CGNAT). Dann funktioniert IPv4-Port-Forwarding grundsätzlich nicht. Test: die öffentliche IP (z.B. über eine „Wie ist meine IP"-Seite) mit der WAN-IP im Router vergleichen – stimmen sie nicht überein oder liegt die WAN-IP im Bereich `100.64.x.x`, hast du CGNAT. Lösungen: beim Provider eine echte öffentliche IPv4 anfragen (oft kostenlos), über IPv6 verbinden (wenn beide Seiten IPv6 haben), oder ein Overlay-Netzwerk wie Tailscale/ZeroTier nutzen.

**4. Proxmox-Snapshot/Backup als oberste Sicherheitsleine.**
Der Script-Rollback fängt einen kaputten Bedrock-Download ab, aber kein verkorkstes System. Vor größeren Eingriffen vom Proxmox-Host:
```bash
pct snapshot <CTID> vor-update
# Im Notfall zurück:  pct rollback <CTID> vor-update
```
Optional regelmäßig: einen wöchentlichen `vzdump` des Containers über Datacenter → Backup einrichten.

**5. ntfy absichern.** Ein zufälliger Themenname (z.B. `online-weber-mc-a8f3k2`) ist kaum zu erraten. Wer es ernster meint, nutzt ntfy mit Access-Token (`-H "Authorization: Bearer tk_..."`) oder hostet ntfy selbst. Der Themenname muss dann in `minecraft-notify.sh` angepasst werden.

**6. Off-Site-Backup.** Aktuell liegen die Backups auf demselben LXC – stirbt der Container, sind sie weg. Per `rclone`/`rsync` nachts auf ein NAS oder in die Cloud kopieren:
```bash
# Beispielzeile für den Crontab (nach dem Backup):
15 3 * * * rsync -a /home/minecraft/backups/ nas:/minecraft-backups/
```

**7. Allowlist (Whitelist) aktivieren.** Wenn der Server aus dem Internet erreichbar ist, hält eine Allowlist Fremde fern. In `server.properties` `allow-list=true` setzen und in `allowlist.json` die erlaubten Spieler eintragen.

**8. Updates nicht „brandneu" einspielen.** Ganz frische Versionen haben gelegentlich Bugs. Wer mag, lässt das Update z.B. nur sonntags laufen (`30 4 * * 0`) oder wartet ein paar Tage – seltener heißt auch seltener Risiko.

**9. Bibliotheken einfrieren (Notnagel).** Sollte ein Ubuntu-Update einmal die Bibliotheken brechen (auf der unterstützten Plattform sehr selten), lassen sich die kritischen Pakete einfrieren:
```bash
apt-mark hold libssl3t64 libcurl4t64
# Wieder freigeben:  apt-mark unhold libssl3t64 libcurl4t64
```

**10. Log-Rotation.** Damit die Log-Dateien nicht endlos wachsen, eine logrotate-Regel anlegen:
```bash
cat > /etc/logrotate.d/minecraft <<'EOF'
/var/log/minecraft-*.log {
    weekly
    rotate 8
    compress
    missingok
    notifempty
}
EOF
```

**11. RAM-Schonung im kleinen LXC.** Bei nur 1–2 GB RAM in `server.properties` `view-distance` (z.B. 8) und `tick-distance` (z.B. 4) reduzieren – das senkt CPU- und Speicherlast spürbar.

**12. Konsolen-Clients beachten.** Auf Xbox/PlayStation/Switch lässt sich ein eigener Server-IP nicht ohne Weiteres eintragen (das geht offiziell nur über DNS-Umleitungs-Tricks). Mit Handy, Tablet und Windows klappt „Server hinzufügen" dagegen problemlos. Spielt die Familie auf Konsole, vorher einplanen.

---

## Schritt 14 – Was die Automatik NICHT abdeckt (manuell)

Die Scripts halten Ubuntu-Pakete und den Bedrock-Server aktuell. **Manuell** bleiben nur diese Dinge – die meisten betreffen dich höchstens alle paar Jahre:

| Aufgabe | Wie oft | Was tun |
|---|---|---|
| **Ubuntu-LTS-Wechsel** (z.B. 24.04 → 26.04) | alle ~2 Jahre | Das Script macht nur `apt upgrade` **innerhalb** von 24.04. Ein Versionssprung läuft über `do-release-upgrade` und sollte bewusst und mit Proxmox-Snapshot gemacht werden (am besten erst nach dem ersten Point-Release der neuen LTS). |
| **Proxmox-Host-Updates** | gelegentlich | Updates des Proxmox-Hosts selbst laufen auf dem Host, **nicht** im LXC. |
| **Kernel** | – | In einem LXC gibt es keinen eigenen Kernel (der kommt vom Proxmox-Host). Du musst also im Container **keine** Kernel-Updates machen. |
| **Neue `server.properties`-Optionen** | sehr selten | Das Update behält absichtlich deine alte Konfiguration. Bringt eine große Bedrock-Version neue Einstellungen mit, tauchen die in deiner Datei nicht automatisch auf. Nach einem großen Sprung lohnt ein Blick: neue Default-Datei aus dem ZIP mit deiner vergleichen. |
| **Ubuntu Pro / ESM** | ab ~2029 | Erst relevant, wenn der reguläre 24.04-Support endet. Für Privatnutzung kostenlos aktivierbar (`pro attach`), verlängert die Sicherheitsupdates um Jahre. |
| **ntfy-App, Router-Weiterleitung, DDNS** | einmalig | Einmal einrichten (Schritt 8, 10, 13), danach Ruhe. |

> **Faustregel:** Im Alltag musst du nichts tun. Aufmerksam werden solltest du nur, wenn eine **ntfy-Fehlermeldung** kommt – dann steht in der Nachricht, was zu prüfen ist.

---

## ✅ Fertig – Zusammenfassung

- ✅ Läuft auf Ubuntu 24.04 LTS (von Mojang offiziell unterstützt)
- ✅ Server läuft als systemd-Service
- ✅ Startet automatisch beim Booten (LXC + systemd)
- ✅ Startet automatisch nach Stromausfällen
- ✅ Firewall mit ufw konfiguriert
- ✅ Port-Forwarding für Internetzugang eingerichtet
- ✅ Standard-Modus Adventure + peaceful, Creative nur für freigeschaltete Spieler
- ✅ Tägliches Backup um 03:00 Uhr (Welt **+ Konfig + OP + Allowlist**), 7 Tage Aufbewahrung
- ✅ Tägliches Auto-Update (Ubuntu + Bedrock) um 04:30 Uhr **mit Rollback bei Fehler**
- ✅ Überwachung alle 5 Minuten mit automatischem Neustart
- ✅ ntfy-Benachrichtigungen bei Update-Erfolg, Server-Ausfall und Fehlern
- ✅ Logs unter `/var/log/minecraft-*.log`

### Mit dem Server verbinden

Minecraft Bedrock: **Multiplayer → Server hinzufügen → IP des LXC eingeben, Port 19132**

LXC-IP ermitteln:

```bash
ip a | grep inet
```

---

## 🔧 Server-Wartung

### Server starten, stoppen & neu starten

| Befehl | Beschreibung |
|---|---|
| `systemctl start minecraft-bedrock` | Server starten |
| `systemctl stop minecraft-bedrock` | Server stoppen |
| `systemctl restart minecraft-bedrock` | Server neu starten |
| `systemctl status minecraft-bedrock` | Status anzeigen |
| `journalctl -u minecraft-bedrock -n 50` | Letzte 50 Log-Einträge |

### Automatik manuell anstoßen / prüfen

| Befehl | Beschreibung |
|---|---|
| `/usr/local/bin/minecraft-update.sh` | Update sofort prüfen/durchführen |
| `/usr/local/bin/minecraft-monitor.sh` | Überwachung sofort ausführen |
| `/usr/local/bin/minecraft-backup.sh` | Backup sofort erstellen |
| `cat /home/minecraft/bedrock/.installed_version` | Aktuell installierte Version |
| `tail -f /var/log/minecraft-update.log` | Update-Log live mitlesen |

### Spieler in den Creative-Modus freischalten

Über die Server-Konsole (siehe unten, **ohne** `/`):

```
gamemode creative DeinXboxGamertag
```

Zurück in den Adventure-Modus:

```
gamemode adventure DeinXboxGamertag
```

### Backup manuell ausführen

```bash
/usr/local/bin/minecraft-backup.sh
```

Vorhandene Backups anzeigen:

```bash
ls -lh /home/minecraft/backups/
```

Backup-Log einsehen:

```bash
cat /var/log/minecraft-backup.log
```

### Backup einspielen

```bash
# Server stoppen
systemctl stop minecraft-bedrock

# Aktuelle Welt sichern (empfohlen!)
cp -r /home/minecraft/bedrock/worlds /home/minecraft/worlds_vor_wiederherstellung

# Backup einspielen (Dateiname anpassen!) – enthält Welt + Konfig + OP + Allowlist
tar -xzf /home/minecraft/backups/minecraft-backup-2026-04-17_03-00.tar.gz \
  -C /home/minecraft/bedrock

# Eigentümer wieder korrekt setzen
chown -R minecraft:minecraft /home/minecraft/bedrock

# Server wieder starten
systemctl start minecraft-bedrock
```

> **Vorsicht:** Der Ordner `worlds/` (sowie die enthaltenen Konfigurationsdateien) werden dabei überschrieben. Deshalb vorher immer die aktuelle Welt sichern!

### Operator-Rechte (OP) vergeben

**Empfohlene Methode – Server im Terminal starten (interaktive Konsole):**

Der Server hat eine direkte Eingabezeile wenn er manuell gestartet wird:

```bash
# 1. systemd-Service stoppen
systemctl stop minecraft-bedrock

# 2. Als minecraft-User wechseln
su - minecraft
cd ~/bedrock

# 3. Server manuell starten
LD_LIBRARY_PATH=. ./bedrock_server
```

Warten bis `Server started.` erscheint, dann direkt eintippen (ohne Slash!):

```
op DeinXboxGamertag
```

Server sauber beenden und Service wieder starten:

```
stop
exit
```

```bash
# Zurück als root:
systemctl start minecraft-bedrock
```

> **Info:** Befehle in der Server-Konsole werden **OHNE** `/` eingegeben – also `op Name` statt `/op Name`.

**Alternative – Über die Datei permissions.json:**

```bash
nano /home/minecraft/bedrock/permissions.json
```

Inhalt (XUID = Xbox-User-ID des Spielers):

```json
[
  {
    "permission": "operator",
    "xuid": "1234567890123456"
  }
]
```

Die XUID findest du in der Datei `valid_known_packs.json` nachdem der Spieler einmal verbunden war, oder online unter xboxgamertag.com.

Nach Änderung den Server neu starten:

```bash
systemctl restart minecraft-bedrock
```

### Neue Welt erstellen (alte Welt behalten)

Einfach einen neuen Weltnamen in der `server.properties` setzen. Die alte Welt bleibt im `worlds/`-Ordner erhalten:

```bash
nano /home/minecraft/bedrock/server.properties
```

Weltnamen ändern, z.B.:

```
level-name=Neue Welt
```

```bash
systemctl restart minecraft-bedrock
```

Der Server erstellt automatisch eine neue Welt unter `worlds/Neue Welt/`. Die alte Welt liegt weiterhin unter `worlds/Bedrock level/` und kann jederzeit durch Zurücksetzen des Namens reaktiviert werden.

> **Tipp:** Sinnvolle Namen wählen, z.B. `Welt 2026-04`, damit man später weiß welche Welt es war.

### Neue Welt „auswürfeln", bis sie passt

Wenn du beim Start mehrere Zufallswelten durchprobieren willst, bis dir eine gefällt, wiederholst du einfach „löschen → neu generieren → anschauen". Der Server erzeugt beim Start automatisch eine neue Welt, sofern keine vorhanden ist.

**Vorbereitung (einmalig):** Sicherstellen, dass der Seed leer ist, damit jedes Mal eine *zufällige* Karte entsteht:

```bash
nano /home/minecraft/bedrock/server.properties
```

```
level-seed=
```

(Speichern mit `Ctrl+X` → `Y` → `Enter`.)

**Wichtig – damit der Monitor nicht dazwischenfunkt:** Solange du Welten würfelst, ist der Server immer wieder gestoppt. Der Watchdog aus Schritt 12 würde ihn dabei selbstständig neu starten (und damit ungewollt eine Welt erzeugen) sowie eine ntfy-Meldung schicken. Setz deshalb vorher das **Wartungs-Flag** – genau dafür ist es da:

```bash
touch /run/minecraft-bedrock.maintenance
```

**Würfel-Schleife** – so oft wiederholen, wie du magst:

```bash
# 1. Server stoppen
systemctl stop minecraft-bedrock

# 2. Aktuelle Welt löschen (Weltname = level-name, anpassen falls abweichend!)
rm -rf /home/minecraft/bedrock/worlds/"Bedrock level"

# 3. Server starten – neue Zufallswelt wird generiert
systemctl start minecraft-bedrock
```

Dann im Spiel beitreten und umsehen (am schnellsten geht das, wenn du dich kurz selbst auf Creative setzt und fliegst: `/gamemode creative DeinGamertag`). Gefällt sie nicht → Schritte 1–3 wiederholen. Gefällt sie → **fertig, einfach so lassen.**

**Zum Schluss das Wartungs-Flag wieder entfernen**, damit die Überwachung wieder aktiv ist:

```bash
rm -f /run/minecraft-bedrock.maintenance
```

> **Tipp:** Das Flag gilt nur 60 Minuten. Dauert deine Such-Session länger, einfach zwischendurch erneut `touch …` ausführen.

> **Bestimmten Seed testen:** Wenn du eine Karte von einer Seed-Seite ausprobieren willst, statt zu würfeln, trag die Zahl unter `level-seed=` ein (statt leer zu lassen) und generiere wie oben neu.

#### Kollidiert das mit dem Backup?

**Mit der Backup-Datei selbst: nein.** Das Backup ist nur ein Schnappschuss des aktuellen Standes – es zwingt dir nichts auf. Du kannst beliebig würfeln, ohne dass ein Backup das stört.

Worauf du nur achten solltest:

- **Der Watchdog** (Monitor) ist der eigentliche „Gegenspieler" beim Würfeln, nicht das Backup – deshalb oben das Wartungs-Flag. Damit ist die Kollision gelöst.
- **Verworfene Welten landen im Backup**, wenn zufällig um 03:00 (Backup) oder 04:30 (Update) eine gerade-noch-vorhandene Testwelt gesichert wird. Das ist harmlos: Es belegt nur kurz etwas Platz und wird durch die 7-Tage-Aufbewahrung von allein wieder gelöscht.
- **Sobald dir eine Welt gefällt**, mach am besten **sofort ein Backup**, statt bis 03:00 zu warten – dann ist die „Keeper"-Welt direkt gesichert:
  ```bash
  /usr/local/bin/minecraft-backup.sh
  ```

> **Vorsicht:** Das Löschen eines Weltordners ist unwiderruflich. Während des Würfelns ist das gewollt (du wirfst ja bewusst weg). Aber sobald eine Welt ein „Behalter" sein könnte, vorher ein Backup ziehen.

### Welt-Einstellungen: immer Tag, kein Wetter usw. (Gamerules)

Dinge wie „immer Tag" stellst du über **Gamerules** und Befehle ein. Diese Einstellungen werden **in der Welt gespeichert** (nicht in der `server.properties`), bleiben also über Neustarts und Updates erhalten – und sind automatisch im Backup enthalten, weil `worlds/` mitgesichert wird. Du musst sie pro Welt nur **einmal** setzen.

Am einfachsten gibst du die Befehle **im Spiel als Operator** mit `/` ein (funktioniert, weil `allow-cheats=true` gesetzt ist). Alternativ in der Server-Konsole ohne `/` (siehe Methode beim OP-Vergeben weiter oben).

**Immer Tag:**

```
/gamerule dodaylightcycle false
/time set day
```

`dodaylightcycle false` friert den Tag-/Nacht-Wechsel ein, `time set day` setzt die Zeit auf Morgen. Für helle Mittagssonne stattdessen `/time set noon`.

**Weitere nützliche Einstellungen für einen entspannten (Bau-)Server:**

| Befehl | Wirkung |
|---|---|
| `/gamerule doweathercycle false` + `/weather clear` | Immer klares Wetter, kein Regen/Gewitter |
| `/gamerule domobspawning false` | Es spawnen gar keine Tiere/Monster mehr |
| `/gamerule mobgriefing false` | Mobs verändern keine Blöcke (z.B. Endermen) |
| `/gamerule pvp false` | Spieler können sich gegenseitig nicht verletzen |
| `/gamerule falldamage false` | Kein Fallschaden |
| `/gamerule firedamage false` | Kein Feuerschaden |
| `/gamerule dofiretick false` | Feuer breitet sich nicht aus |
| `/gamerule tntexplodes false` | TNT explodiert nicht |
| `/gamerule doinsomnia false` | Keine Phantome bei langem Nicht-Schlafen |
| `/gamerule showcoordinates true` | Koordinaten werden oben links angezeigt |
| `/gamerule keepinventory true` | Items bleiben beim Tod erhalten |

> **Hinweis:** `difficulty=peaceful` (aus Schritt 5) verhindert ohnehin schon feindliche Monster und Hunger. Die Gamerules oben sind on top für noch mehr Ruhe – nimm nur das, was du wirklich willst.

> **Aktuelle Werte prüfen:** Einen einzelnen Gamerule kannst du dir ohne Wert-Angabe ausgeben lassen, z.B. `/gamerule dodaylightcycle`.

### Server-Update (manuell)

Im Normalbetrieb übernimmt das **Auto-Update aus Schritt 11** diese Schritte automatisch. Falls du trotzdem einmal von Hand updaten willst, genügt:

```bash
/usr/local/bin/minecraft-update.sh
```

Das Script prüft die neueste Version, macht ein Backup, aktualisiert und rollt bei einem Fehlstart automatisch zurück. Beim Update werden nur die Server-Binaries ersetzt – Welten, Einstellungen und Berechtigungen bleiben erhalten. In den Logs erscheint danach die neue Version:

```bash
journalctl -u minecraft-bedrock -n 20
```

> **Info:** Die Welten unter `worlds/` werden beim Update niemals überschrieben.
