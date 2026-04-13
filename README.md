# ubuntu-minecraft-server
Hot to create and maintain two parallel servers for minecraft bedrock under ubuntu server

Minecraft Bedrock Server Komplettanleitung
(Ubuntu 24.04)
Setup: Zwei Server parallel (Lobby + Survival)
1. Server Einrichtung und Installation (ALLES)
1.1 System aktualisieren
sudo apt update && sudo apt upgrade -y
sudo apt install unzip curl wget screen ufw htop nano -y
1.2 Firewall konfigurieren (WICHTIG)
sudo ufw allow 22/tcp
sudo ufw allow 19132/udp
sudo ufw allow 19133/udp
sudo ufw allow 19134/udp
sudo ufw allow 19135/udp
sudo ufw enable
sudo ufw reload
sudo ufw status
1.3 Ersten Minecraft Server installieren (Lobby)
mkdir ~/bedrock
cd ~/bedrock
wget https://minecraft.azureedge.net/bedrockdedicatedserver/bin-linux/
bedrock-server-1.26.14.1.zip
unzip bedrock-server-1.26.14.1.zip
chmod +x bedrock_server
1.4 server.properties bearbeiten
nano ~/bedrock/server.properties
Empfohlen:
1
server-name=Lobby
level-name=LobbyWorld
gamemode=creative
difficulty=peaceful
allow-cheats=true
server-port=19132
server-portv6=19133
max-players=20
1.5 Zweiten Server erstellen (Survival)
cd ~
cp -r bedrock bedrock-survival
nano ~/bedrock-survival/server.properties
Empfohlen:
server-name=Survival
level-name=SurvivalWorld
gamemode=survival
difficulty=normal
allow-cheats=true
server-port=19134
server-portv6=19135
max-players=20
1.6 Autostart Service Server 1
sudo nano /etc/systemd/system/bedrock.service
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
2
1.7 Autostart Service Server 2
sudo nano /etc/systemd/system/bedrock-survival.service
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
1.8 Services aktivieren
sudo systemctl daemon-reload
sudo systemctl enable bedrock
sudo systemctl enable bedrock-survival
sudo systemctl start bedrock
sudo systemctl start bedrock-survival
1.9 Verwaltung Services
sudo systemctl start bedrock
sudo systemctl stop bedrock
sudo systemctl restart bedrock
sudo systemctl status bedrock
sudo systemctl start bedrock-survival
sudo systemctl stop bedrock-survival
sudo systemctl restart bedrock-survival
sudo systemctl status bedrock-survival
1.10 Logs live ansehen
journalctl -u bedrock -f
journalctl -u bedrock-survival -f
3
2. Verwaltung Minecraft Server Befehle (ALLE)
2.1 Adminrechte vergeben (OP)
Server direkt starten:
sudo systemctl stop bedrock
cd ~/bedrock
LD_LIBRARY_PATH=. ./bedrock_server
Dann in der Konsole:
op MaxPower3309
2.2 Wichtige Ingame Commands
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
2.3 Lobby Schutz
/defaultgamemode adventure
Spieler können kaum abbauen.
2.4 Neue Welt erzeugen
4
sudo systemctl stop bedrock-survival
nano ~/bedrock-survival/server.properties
Ändern:
level-name=NeueWelt
Dann starten:
sudo systemctl start bedrock-survival
3. Sonstiges wichtiges
3.1 Server IP anzeigen
ip a
3.2 Ports prüfen
ss -tulpn | grep 1913
3.3 RAM prüfen
free -h
3.4 CPU / Prozesse prüfen
htop
3.5 Speicherplatz prüfen
df -h
3.6 Automatisches Abschalten 22:00 Uhr
5
sudo timedatectl set-timezone Europe/Berlin
sudo crontab -e
Eintragen:
00 22 * * * /bin/systemctl stop bedrock && /bin/systemctl stop bedrocksurvival && /sbin/shutdown now
3.7 Portweiterleitung FritzBox
Extern freigeben:
UDP 19132 -> 19132
UDP 19134 -> 19134
3.8 Updates installieren
sudo systemctl stop bedrock
sudo systemctl stop bedrock-survival
cp -r ~/bedrock ~/bedrock_backup
cp -r ~/bedrock-survival ~/bedrock_survival_backup
Neue Dateien entpacken und nur Serverdateien ersetzen.
3.9 Texteditor
nano datei.txt
Speichern: CTRL+O Beenden: CTRL+X
Empfohlene Struktur
19132 Lobby Server
19134 Survival Server
Cheats aktiv
OP Rechte nur Admins
Tägliche Backups empfohlen
Fertig
Dein Setup ist damit vollständig dokumentiert.
•
•
•
•
•
•
•
6
