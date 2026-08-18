désactiver la verificatino de mail
realm setting > login > Email as username    et   Login with email    off

# Installation
```bash
cd /tmp
wget https://github.com/oauth2-proxy/oauth2-proxy/releases/download/v7.15.3/oauth2-proxy-v7.15.3.linux-amd64.tar.gz
tar xvzf oauth2-proxy-v7.15.3.linux-amd64.tar.gz
sudo mv oauth2-proxy-v7.15.3.linux-amd64/oauth2-proxy /usr/local/bin/
sudo chmod +x /usr/local/bin/oauth2-proxy
oauth2-proxy --version
```
Créer un utilisateur système dédié
```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin oauth2-proxy
sudo mkdir -p /etc/oauth2-proxy
sudo chown root:oauth2-proxy /etc/oauth2-proxy
sudo chmod 750 /etc/oauth2-proxy
```
3. Générer le cookie secret
```bash
openssl rand -base64 32 | tr -- '+/' '-_'
```

sudo nano /etc/oauth2-proxy/oauth2-proxy.cfg



sudo nano /etc/systemd/system/oauth2-proxy.service
```bash
[Unit]
Description=OAuth2 Proxy - kpi-motoculture
After=network.target

[Service]
Type=simple
User=oauth2-proxy
Group=oauth2-proxy
ExecStart=/usr/local/bin/oauth2-proxy --config=/etc/oauth2-proxy/oauth2-proxy.cfg
Restart=on-failure
RestartSec=5

# Durcissement
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
6. Activer et démarrer
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now oauth2-proxy
sudo systemctl status oauth2-proxy
```
