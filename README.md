# livetimingServer
simpler Nachbau des FIS Livetiming Servers

## nginx mit Python
```
sudo apt update && sudo apt upgrade
# nginx Server installieren
sudo apt install nginx
```
### Endpoint anlegen
```
# Erstelle eine neue Konfigurationsdatei
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
