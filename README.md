# livetimingServer
simpler Nachbau des FIS Livetiming Servers

## nginx mit Python und BaseX
```
sudo apt update && sudo apt upgrade
# nginx Server installieren
sudo apt install nginx
# da die Daten vom Winlaufen Zeitnehmer PC im XML Format geliefert werden, habe ich mich für eine spezifische XML NoSQL Datenbank entschieden.
sudo apt install basex
```
### Endpoint anlegen
```
# Erstelle eine neue nginx-Konfigurationsdatei
sudo touch /etc/nginx/sites-available/livetiming

# Nutze 'echo' und 'cat', um den Inhalt in die Datei zu schreiben
sudo bash -c 'cat > /etc/nginx/sites-available/livetiming' << EOF
server {
    listen 80;
    server_name 172.20.1.101; # die IP Adresse anpassen bzw. Domain eintragen

    location /livetiming {
        proxy_pass http://127.0.0.1:5000;  # Weiterleitung an die Flask-Anwendung
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
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
su - livetiming
python3 -m venv ~/livetimingenv
cd ~/livetimingenv
pip install Flask Flask-SocketIO
# pip3 install gunicorn
pip install BaseXClient
mkdir ~/basexdb
```
### BaseX konfigurieren
#### als systemd Dienst einrichten
```
sudo bash -c 'cat > /etc/systemd/system/basexserver.service' << EOF
[Unit]
Description=BaseX XML Database Server
After=network.target

[Service]
User=livetiming
Group=livetiming
WorkingDirectory=/home/livetiming/basexdb/  # Pfad zum BaseX-Datenbankverzeichnis
ExecStart=/usr/bin/basexserver  # Pfad zum basexserver-Befehl
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl start basexserver
sudo systemctl enable basexserver

```
#### dem User 'livetiming' die Zugriffsrechte geben
```
sudo nano /etc/basex/basexserver.config
# Berechtigungen für User 'livetiming' konfigurieren
```

### simples Python-Skript zum Empfang der xml-Daten und zum Einfügen in die BaseX XML DB
```
# als user 'livetiming' ausführen:
# Erstelle die Pythondatei livetiming_receiver.py
mkdir ~/livetiming_app
sudo touch ~/livetiming_app/livetiming_receiver.py
sudo bash -c 'cat > ~/livetiming_app/livetiming_receiver.py' << EOF
from flask import Flask, request
from BaseXClient import BaseXClient

app = Flask(__name__)

@app.route('/livetiming', methods=['POST'])
def livetiming():
    xml_data = request.data  # Hier erhältst du die XML-Daten von der POST-Anfrage
    print("Empfangene XML-Daten:")
    print(xml_data.decode('utf-8'))  # Ausgabe der XML-Daten im Terminal

    # Verbindung zur BaseX-Datenbank herstellen
    session = BaseXClient.Session('localhost', 1984, 'admin', 'admin')

    # XML-Daten in die BaseX-Datenbank einfügen
    try:
        session.execute("CREATE DB livetiming_db")
        session.execute("OPEN livetiming_db")
        session.add(xml_data)  # Hier fügst du die XML-Daten zur Datenbank hinzu
        session.execute("CLOSE")
        return 'XML-Daten empfangen und in BaseX gespeichert.'
    except Exception as e:
        return f'Fehler beim Speichern der XML-Daten: {e}'
    finally:
        # Sitzung schließen
        session.close()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)

EOF
chmod -R 700 ~/livetiming_app
```
### simples Beispiel zum Abrufen der Daten aus der Datenbank und Verteilung an die Clients
```
# Skript folgt gleich
```
