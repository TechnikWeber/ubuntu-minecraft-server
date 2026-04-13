# Minecraft Bedrock Server Komplettanleitung (Ubuntu 24.04)

## Setup: Zwei Server parallel (Lobby + Survival)

---

# 1. Server Einrichtung und Installation (ALLES)

## 1.1 System aktualisieren
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install unzip curl wget screen ufw htop nano -y
```

## 1.2 Firewall konfigurieren (WICHTIG)
```bash
sudo ufw allow 22/tcp
sudo ufw allow 19132/udp
sudo ufw allow 19133/udp
sudo ufw allow 19134/udp
sudo ufw allow 19135/udp
sudo ufw enable
sudo ufw reload
sudo ufw status
```

## 1.3 Ersten Minecraft Server installieren (Lobby)
```bash
mkdir ~/bedrock
cd ~/bedrock
wget https://minecraft.azureedge.net/bedrockdedicatedserver/bin-linux/bedrock-server-1.26.14.1.zip
unzip bedrock-server-1.26.14.1.zip
chmod +x bedrock_server
```

## 1.4 server.properties bearbeiten
```bash
nano ~/bedrock/server.properties
```
Empfohlen:
```properties
server-name=Lobby
level-name=LobbyWorld
gamemode=creative
difficulty=peaceful
allow-cheats=true
server-port=19132
server-portv6=19133
max-players=20
```

## 1.5 Zweiten Server erstellen (Survival)
```bash
cd ~
cp -r bedrock bedrock-survival
nano ~/bedrock-survival/server.properties
```
Empfohlen:
```properties
server-name=Survival
level-name=SurvivalWorld
gamemode=survival
difficulty=normal
allow-cheats=true
server-port=19134
server-portv6=19135
max-players=20
```

## 1.6 Autostart Service Server 1
```bash
sudo nano /etc/systemd/system/bedrock.service
```
```ini
[Unit]
Description=Minecraft Bedrock Lobby
After=network.target

[Service]
User=philipp
WorkingDirectory=/home/philipp/bedrock
ExecStart=/bin/bash -c 'LD_LIBRARY_PATH=. ./bedrock_server'
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## 1.7 Autostart Service Server 2
```bash
sudo nano /etc/systemd/system/bedrock-survival.service
```
```ini
[Unit]
Description=Minecraft Bedrock Survival
After=network.target

[Service]
User=philipp
WorkingDirectory=/home/philipp/bedrock-survival
ExecStart=/bin/bash -c 'LD_LIBRARY_PATH=. ./bedrock_server'
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## 1.8 Services aktivieren
```bash
sudo systemctl daemon-reload
sudo systemctl enable bedrock
sudo systemctl enable bedrock-survival
sudo systemctl start bedrock
sudo systemctl start bedrock-survival
```

## 1.9 Verwaltung Services
```bash
sudo systemctl start bedrock
sudo systemctl stop bedrock
sudo systemctl restart bedrock
sudo systemctl status bedrock

sudo systemctl start bedrock-survival
sudo systemctl stop bedrock-survival
sudo systemctl restart bedrock-survival
sudo systemctl status bedrock-survival
```

## 1.10 Logs live ansehen
```bash
journalctl -u bedrock -f
journalctl -u bedrock-survival -f
```

---

# 2. Verwaltung Minecraft Server Befehle (ALLE)

## 2.1 Adminrechte vergeben (OP)
Server direkt starten:
```bash
sudo systemctl stop bedrock
cd ~/bedrock
LD_LIBRARY_PATH=. ./bedrock_server
```
Dann in der Konsole:
```text
op MaxPower3309
```

## 2.2 Wichtige Ingame Commands
```text
/gamemode creative
/gamemode survival
/gamemode adventure
/give
/tp
/fill
/setblock
/clone
/time set day
/weather clear
/setworldspawn
/defaultgamemode adventure
/gamerule showcoordinates true
/gamerule doDaylightCycle false
/gamerule doWeatherCycle false
/gamerule keepInventory true
```

## 2.3 Lobby Schutz
```text
/defaultgamemode adventure
```
Spieler können kaum abbauen.

## 2.4 Neue Welt erzeugen
```bash
sudo systemctl stop bedrock-survival
nano ~/bedrock-survival/server.properties
```
Ändern:
```properties
level-name=NeueWelt
```
Dann starten:
```bash
sudo systemctl start bedrock-survival
```

---

# 3. Sonstiges wichtiges

## 3.1 Server IP anzeigen
```bash
ip a
```

## 3.2 Ports prüfen
```bash
ss -tulpn | grep 1913
```

## 3.3 RAM prüfen
```bash
free -h
```

## 3.4 CPU / Prozesse prüfen
```bash
htop
```

## 3.5 Speicherplatz prüfen
```bash
df -h
```

## 3.6 Automatisches Abschalten 22:00 Uhr
```bash
sudo timedatectl set-timezone Europe/Berlin
sudo crontab -e
```
Eintragen:
```cron
00 22 * * * /bin/systemctl stop bedrock && /bin/systemctl stop bedrock-survival && /sbin/shutdown now
```

## 3.7 Portweiterleitung FritzBox
Extern freigeben:
- UDP 19132 -> 19132
- UDP 19134 -> 19134

## 3.8 Updates installieren
```bash
sudo systemctl stop bedrock
sudo systemctl stop bedrock-survival
cp -r ~/bedrock ~/bedrock_backup
cp -r ~/bedrock-survival ~/bedrock_survival_backup
```
Neue Dateien entpacken und nur Serverdateien ersetzen.

## 3.9 Texteditor
```bash
nano datei.txt
```
Speichern: CTRL+O
Beenden: CTRL+X

---

# Empfohlene Struktur
- 19132 Lobby Server
- 19134 Survival Server
- Cheats aktiv
- OP Rechte nur Admins
- Tägliche Backups empfohlen

---

# Fertig
Dein Setup ist damit vollständig dokumentiert.

