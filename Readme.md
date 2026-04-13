# Minecraft Bedrock Server – Complete README (Ubuntu 24.04)

Setup für 2 Server (Creative + Survival) parallel auf Linux inkl. Autostart, Backup, Updates & Verwaltung.

---

# 1. SERVER SETUP (Installation + 2 Server)

## 1.1 System vorbereiten
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install unzip wget curl ufw nano htop -y
```

---

## 1.2 Firewall (WICHTIG)
```bash
sudo ufw allow 22/tcp
sudo ufw allow 19132/udp
sudo ufw allow 19133/udp
sudo ufw allow 19134/udp
sudo ufw allow 19135/udp
sudo ufw enable
sudo ufw status
```

---

## 1.3 Creative Server (Server 1) installieren
```bash
mkdir ~/bedrock-creative
cd ~/bedrock-creative
wget https://minecraft.azureedge.net/bedrockdedicatedserver/bin-linux/bedrock-server-1.26.14.1.zip
unzip *.zip
chmod +x bedrock_server
```

server.properties:
```properties
server-name=Creative
level-name=CreativeWorld
gamemode=creative
difficulty=peaceful
allow-cheats=true
server-port=19132
server-portv6=19133
```

---

## 1.4 Survival Server (Server 2) erstellen
```bash
cp -r ~/bedrock-creative ~/bedrock-survival
```

server.properties ändern:
```properties
server-name=Survival
level-name=SurvivalWorld
gamemode=survival
server-port=19134
server-portv6=19135
allow-cheats=true
```

---

## 1.5 systemd Autostart

Creative Server:
```bash
sudo nano /etc/systemd/system/bedrock-creative.service
```

Survival Server:
```bash
sudo nano /etc/systemd/system/bedrock-survival.service
```

Creative Service Inhalt:
```ini
[Unit]
Description=Minecraft Bedrock Creative Server
After=network.target

[Service]
User=philipp
WorkingDirectory=/home/YOURNAME/bedrock-creative
ExecStart=/bin/bash -c 'LD_LIBRARY_PATH=. ./bedrock_server'
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Survival Service bleibt gleich strukturiert.

---

## 1.6 Services aktivieren
```bash
sudo systemctl daemon-reload
sudo systemctl enable bedrock-creative
sudo systemctl enable bedrock-survival
sudo systemctl start bedrock-creative
sudo systemctl start bedrock-survival
```

---

## 1.7 Verwaltung
```bash
sudo systemctl start bedrock-creative
sudo systemctl stop bedrock-creative
sudo systemctl restart bedrock-creative
sudo systemctl status bedrock-creative

sudo systemctl start bedrock-survival
sudo systemctl stop bedrock-survival
sudo systemctl restart bedrock-survival
sudo systemctl status bedrock-survival
```

Logs:
```bash
journalctl -u bedrock-creative -f
journalctl -u bedrock-survival -f
```

---

# 2. MINECRAFT VERWALTUNG

## 2.1 OP Rechte (Spieler muss verbunden sein!)
```text
op YOURGAMETAG
```

---

## 2.2 Wichtige Commands
```text
/gamemode creative
/gamemode survival
/tp
/give
/fill
/clone
/time set day
/weather clear
/setworldspawn
/gamerule showcoordinates true
```

---

## 2.3 Cheats aktivieren
```properties
allow-cheats=true
```

---

## 2.4 Welt wechseln
```bash
nano server.properties
level-name=NEUE_WELT
systemctl restart bedrock-creative
```

---

## 2.5 OP Hinweis
- Microsoft Name verwenden

---

# 3. SYSTEM & ADMIN TOOLS

## 3.1 IP anzeigen
```bash
ip a
```

## 3.2 Ports prüfen
```bash
ss -tulpn | grep 1913
```

## 3.3 RAM
```bash
free -h
```

## 3.4 CPU
```bash
htop
```

## 3.5 Speicher
```bash
df -h
```

---

## 3.6 Cron Shutdown 22:00
```bash
sudo timedatectl set-timezone Europe/Berlin
sudo crontab -e
```
```cron
00 22 * * * /bin/systemctl stop bedrock-creative && /bin/systemctl stop bedrock-survival && /sbin/shutdown now
```

---

## 3.7 Portfreigabe FRITZ!Box
- UDP 19132 → Creative
- UDP 19134 → Survival

---

## 3.8 Updates
Server stoppen, Backup machen, neue Version einspielen.

---

## 3.9 Texteditor
```bash
nano datei.txt
CTRL+O speichern
CTRL+X schließen
```

---

## 3.10 AUTOMATISCHES BACKUP
```bash
nano ~/backup.sh
```
```bash
#!/bin/bash
DATE=$(date +%F-%H%M)
systemctl stop bedrock-creative
systemctl stop bedrock-survival
tar -czf /home/YOURNAME/backup-$DATE.tar.gz /home/philipp/bedrock-creative /home/philipp/bedrock-survival
systemctl start bedrock-creative
systemctl start bedrock-survival
find /home/YOURNAME -name "backup-*.tar.gz" -mtime +7 -delete
```
```bash
chmod +x ~/backup.sh
crontab -e
```
```cron
0 3 * * * /home/YOURNAME/backup.sh
```

---

## 3.11 AUTO-UPDATE SCRIPT
```bash
nano ~/update.sh
```
```bash
#!/bin/bash
systemctl stop bedrock-creative
systemctl stop bedrock-survival
cp -r ~/bedrock-creative ~/backup_preupdate_$(date +%F)
cd ~
wget https://minecraft.net/latest-bedrock.zip -O update.zip
unzip -o update.zip -d update_tmp
cp -r update_tmp/* ~/bedrock-creative/
cp -r update_tmp/* ~/bedrock-survival/
chmod +x ~/bedrock-creative/bedrock_server
chmod +x ~/bedrock-survival/bedrock_server
rm -rf update_tmp update.zip
systemctl start bedrock-creative
systemctl start bedrock-survival
```
```bash
chmod +x ~/update.sh
```

---

## 3.12 Webpanel (Netdata)
```bash
sudo apt install netdata -y
sudo systemctl enable netdata
sudo systemctl start netdata
sudo ufw allow 19999/tcp
```
```text
http://SERVER-IP:19999
```

---

## 3.13 Live Spieler Anzeige
```bash
pip3 install mcstatus
mcstatus 127.0.0.1 status
```

