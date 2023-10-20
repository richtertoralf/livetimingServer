# livetimingServer
simpler Nachbau eines Livetiming Servers
auf Ubuntu 22.04 OS, mit Nginx, BaseX DB, Python, Websocket, JavaScript, CSS, HTML

## Installation
```
sudo apt update && sudo apt upgrade
# nginx Server installieren
sudo apt install nginx
##################
# da die Daten vom Winlaufen Zeitnehmer PC im XML Format geliefert werden,
# habe ich mich für eine spezifische XML NoSQL Datenbank entschieden.
# sudo apt install basex # das funktioniert nicht gut!
# Achtung, die auf diese Art und Weise installierte BaseX DB ist nur die Core-Version und nicht aktuell (Stand 10/2023)
```
## Installation der BaseX Datenbank
```
# Deshalb, zur Installation von BaseX und Java direkt auf der Homepage nachsehen.
# BaseX empfiehlt die Java-Umgebung von adoptium.net.
# https://adoptium.net/installation/linux/
echo "deb [arch=amd64] https://some.repository.url focal main" | sudo tee /etc/apt/sources.list.d/adoptium.list > /dev/null
sudo apt install -y wget apt-transport-https
sudo mkdir -p /etc/apt/keyrings
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | sudo tee /etc/apt/keyrings/adoptium.asc
sudo echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | sudo tee /etc/apt/sources.list.d/adoptium.list
deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb jammy main
sudo apt update
apt install temurin-17-jdk
###################
# https://basex.org/
wget https://files.basex.org/releases/10.7/BaseX107.zip
sudo unzip BaseX107.zip -d /opt
sudo chmod -R 755 /opt/basex
sudo /opt/basex/bin/basexhttp -c PASSWORD
# Damit startest du BaseX inklusive der Weboberfläche und wirst aber vorher im Terminal aufgefordert, ein Startpasswort für den User: admin festzulegen.
# Jetzt erreichst du die Weboberfläche via "<die-Ip-deines-Servers>:8080" oder "localhost:8080".
```
>Jetzt kannst du die BaseX DB erkunden und z.B. eine Datenbank anlegen.

## Nginx Endpoint anlegen
In Winlaufen funktioniert derzeit nur die Ziel-Angabe: <Domain:Port>,
deswegen verwende ich keinen Pfad, wie z.B. "<Server-IP>/livetiming".
Als Ziel in Winlaufen (Setup/FIS) wird dann eingegeben: "<meine-Server-IP>:5000",
zum Testen dieses "livetiming-Servers".
### via Port 5000
```
# Erstelle eine neue nginx-Konfigurationsdatei
sudo touch /etc/nginx/sites-available/livetiming
# Inhalt in Datei schreiben
sudo bash -c 'cat > /etc/nginx/sites-available/livetiming' << 'EOF'
server {
    listen 0.0.0.0:5000;
    server_name _;

    location / {
        proxy_pass http://127.0.0.1:5001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
# symbolischen Link erstellen
sudo ln -s /etc/nginx/sites-available/livetiming /etc/nginx/sites-enabled/
```
## Pakete für Python installieren und Linux-User 'livetiming' anlegen
```
# überprüfen, ob Python3 vorhanden ist
python3 --version
# und gegenfalls nachinstallieren
# sudo apt install python3
# überprüfen ob pip installiert ist
pip3 --version
# und gegebenfalls nachinstallieren
sudo apt install python3-pip python3-venv
```
```
# virtuelle Umgebung und User für Python-App anlegen und Pakete mit pip installieren
sudo adduser livetiming
sudo usermod -aG sudo livetiming
su - livetiming
python3 -m venv ~/livetimingenv
cd ~/livetimingenv
pip install Flask Flask-SocketIO
```
## BaseX konfigurieren
### als systemd Dienst einrichten
```
# ermitteln, wo der basexserver liegt:
whereis basexserver
```
```
sudo bash -c 'cat > /etc/systemd/system/basexserver.service' << EOF
[Unit]
Description=BaseX XML Database
After=network.target

[Service]
ExecStart=/opt/basex/bin/basexhttp
Restart=always
[Install]
WantedBy=multi-user.target
EOF

# und jetzt den Dienst starten und richtig einschalten:
sudo systemctl start basexserver
sudo systemctl enable basexserver

```
#### dem User 'livetiming' die Zugriffsrechte für die DB geben
```
Nach dem Start des Dienstes, wird auch die Konfigurationsdatei angelegt:
livetiming@server101:/$ find / -type f -name ".basex" 2>/dev/null
/home/livetiming/basex/.basex
# Hier könnte jetzt u.a. ein User mit Password für den Zugriff eingetragen werden.
sudo nano /home/livetiming/basex/.basex
```

### simples Python-Skript zum Empfang der xml-Daten und zum Einfügen in die BaseX XML DB
```
# als user 'livetiming' ausführen:
# Erstelle die Pythondatei livetiming_receiver.py
mkdir ~/livetiming_app
touch ~/livetiming_app/livetiming_receiver.py
# Dateiinhalt eintragen:
bash -c 'cat > ~/livetiming_app/livetiming_receiver.py' << 'EOF'
from flask import Flask, request
from BaseXClient import BaseXClient

app = Flask(__name__)

@app.route('/', methods=['POST'])
def livetiming():
    xml_data = request.data  # Hier erhältst du die XML-Daten von der POST-Anfrage
    print("Empfangene XML-Daten:")
    print(xml_data.decode('utf-8'))  # Ausgabe der XML-Daten im Terminal

    # Verbindung zur BaseX-Datenbank herstellen
    try:
        with BaseXClient.Session('localhost', 1984, 'livetiming', 'linux') as session:
            # Neue Datenbank erstellen, wenn nicht vorhanden
            session.execute("CREATE DB if not exists livetiming_db")

            # Datenbank öffnen
            session.execute("OPEN livetiming_db")

            # XML-Daten in die BaseX-Datenbank einfügen
            session.add("livetiming_db", xml_data)

            print("XML-Daten in die Datenbank eingetragen.")
            return 'XML-Daten empfangen und in BaseX gespeichert.'

    except Exception as e:
        error_message = f"Fehler beim Speichern der XML-Daten: {e}"
        print(error_message)
        return error_message

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001, debug=True)

EOF
# Berechtigungen setzen
chmod -R 700 ~/livetiming_app
```
### simples Beispiel zum Abrufen der Daten aus der Datenbank und Verteilung an die Clients
```
# Skript folgt gleich
```
