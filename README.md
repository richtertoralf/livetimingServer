# livetimingServer
simpler Nachbau des FIS Livetiming Servers

## nginx mit Python und BaseX
```
sudo apt update && sudo apt upgrade
# nginx Server installieren
sudo apt install nginx
# da die Daten vom Winlaufen Zeitnehmer PC im XML Format geliefert werden, habe ich mich für eine spezifische XML NoSQL Datenbank entschieden.
# zur Installation von BaseX und Java direkt auf der Homepage nachsehen:
# https://adoptium.net/installation/linux/   oder einfach:
sudo apt install basex
```
### Endpoint anlegen
In Winlaufen funktioniert derzeit nur die Ziel-Angabe: <Domain:Port>
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
### Backendabhängigkeiten für Python installieren und User 'livetiming' anlegen
```
# überprüfen, ob Python vorhanden ist
python3 --version
# und gegenfalls nachinstallieren
sudo apt install python3
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
mkdir ~/basexdb
```
### BaseX konfigurieren
#### als systemd Dienst einrichten
```
# ermitteln, wo der basexserver liegt:
whereis basexserver
# Verzeichnis für Logdateien anlegen
mkdir -p /home/livetiming/basexdb/log

```
```
sudo bash -c 'cat > /etc/systemd/system/basexserver.service' << EOF
[Unit]
Description=BaseX XML Database
After=network.target

[Service]
ExecStart=/usr/bin/basexserver
User=livetiming
WorkingDirectory=/home/livetiming/basexdb
Restart=always
StandardOutput=append:/home/livetiming/basexdb/log/basexserver.log
StandardError=append:/home/livetiming/basexdb/log/basexserver.log

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

            # XML-Daten in die BaseX-Datenbank einfügen
            session.add("livetiming_db", xml_data)

            print("XML-Daten in die Datenbank eingetragen.")
            return 'XML-Daten empfangen und in BaseX gespeichert.'
    except Exception as e:
        print(f"Fehler beim Speichern der XML-Daten: {e}")
        return f'Fehler beim Speichern der XML-Daten: {e}'

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
