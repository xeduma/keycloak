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
sudo usermod -aG www-data oauth2-proxy
```
3. Générer le cookie secret
```bash
openssl rand -base64 32 | tr -- '+/' '-_'
```
4. Service systemd
```bash
sudo nano /etc/systemd/system/oauth2-proxy@.service
```
```bash
[Unit]
Description=OAuth2 Proxy - %i
After=network.target

[Service]
Type=simple
User=oauth2-proxy
Group=oauth2-proxy
ExecStart=/usr/local/bin/oauth2-proxy --config=/etc/oauth2-proxy/%i.cfg
Restart=on-failure
RestartSec=5

NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

# Configuration 
## configuration du realm
```bash
sudo nano /etc/oauth2-proxy/kpi-motoculture.cfg
```
```bash

provider = "keycloak-oidc"
oidc_issuer_url = "https://login.domaine.fr/realms/kpi-test"

client_id = "kpi-motoculture"

#copie colle le client > credential > client secret
client_secret = "CmGtN4j38ZjALcDinBeDqSnTVm77qVNG"

redirect_url = "https://kpi-motoculture.domaine.fr/oauth2/callback"

# copie le token generer avec :
# openssl rand -base64 32 | tr -- '+/' '-_'
cookie_secret = "H65QZbomFfeCiK-gSMa3aFnovvIM2H1_nCkC-23f9uw="
cookie_secure = true
cookie_expire = "168h"
cookie_samesite = "lax"

email_domains = ["*"]

#serveur web en reverse proxy
#upstreams = ["http://127.0.0.1:8080/"]

#serveur web classic /var/www/html
upstreams = ["static://202"]


#srv oauth2
http_address = "127.0.0.1:4180"

scope = "openid profile email"
standard_logging = true
request_logging = true
auth_logging = true
```
5. Activer chaque instance
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now oauth2-proxy@kpi-motoculture
sudo systemctl status oauth2-proxy@kpi-motoculture
journalctl -u oauth2-proxy@kpi-motoculture -f
```

# config web
désactiver la verification de mail
realm setting > login > Email as username    et   Login with email    off


