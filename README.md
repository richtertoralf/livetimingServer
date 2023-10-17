# livetimingServer
simpler Nachbau des FIS Livetiming Servers

## nginx mit Python und MongoDB
```
sudo apt update && sudo apt upgrade
# nginx Server installieren
sudo apt install nginx
# sudo apt install mongodb # funktioniert nicht unter Ubuntu 22.04, deshalb:
wget https://repo.mongodb.org/apt/ubuntu/dists/jammy/mongodb-org/7.0/multiverse/binary-amd64/mongodb-org-server_7.0.2_amd64.deb
# diesen Link erhältst du auf der MongoDB Projektseite. Ich habe die MongoDB Community Version für Ubuntu 22.04 gewählt.
sudo dpkg -i mongodb-org-server_7.0.2_amd64.deb
sudo apt install -f
sudo systemctl start mongod
sudo systemctl enable mongod
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
### Backendabhängigkeiten für Python installieren
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
pip3 install gunicorn
```
### simples Python-Skript zum Empfang der xml-Daten
```
# als user 'livetiming' ausführen:
# Erstelle die Pythondatei livetiming_receiver.py
mkdir ~/livetiming_app
sudo touch ~/livetiming_app/livetiming_receiver.py
sudo bash -c 'cat > ~/livetiming_app/livetiming_receiver.py' << EOF
from flask import Flask, request
from pymongo import MongoClient

app = Flask(__name__)
client = MongoClient('mongodb://localhost:27017/')  # Verbindung zur MongoDB herstellen
db = client['livetiming_db']  # Datenbank auswählen
collection = db['livetiming_data']  # Collection erstellen oder auswählen

@app.route('/livetiming', methods=['POST'])
def livetiming():
    xml_data = request.data  # Hier erhältst du die XML-Daten von der POST-Anfrage
    print("Empfangene XML-Daten:")
    print(xml_data.decode('utf-8'))  # Ausgabe der XML-Daten im Terminal
    Speichere die XML-Daten in MongoDB
    collection.insert_one({'xml_data': xml_data.decode('utf-8')})
    return 'XML-Daten empfangen und in MongoDB gespeichert.'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
chmod -R 700 ~/livetiming_app
```
