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

# Configuration srv Oauth2
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
## keycloak
créer client > import

```json
{
  "clientId": "kpi-motoculture-web",
  "name": "KPI Motoculture Web",
  "rootUrl": "https://kpi-motoculture.domaine.fr",
  "baseUrl": "/",
  "enabled": true,
  "clientAuthenticatorType": "client-secret",
  "secret": "CHANGE_MOI_GENERE_UN_SECRET",
  "redirectUris": [
    "https://kpi-motoculture.domaine.fr/oauth2/callback"
  ],
  "webOrigins": [
    "https://kpi-motoculture.domaine.fr"
  ],
  "standardFlowEnabled": true,
  "implicitFlowEnabled": false,
  "directAccessGrantsEnabled": false,
  "serviceAccountsEnabled": false,
  "publicClient": false,
  "frontchannelLogout": true,
  "protocol": "openid-connect",
  "attributes": {
    "post.logout.redirect.uris": "https://kpi-motoculture.domaine.fr/*",
    "backchannel.logout.session.required": "true"
  },
  "fullScopeAllowed": true,
  "defaultClientScopes": [
    "web-origins",
    "profile",
    "roles",
    "email"
  ]
}

```
désactiver la verification de mail
realm setting > login > Email as username    et   Login with email    off

## nginx
```bash

server {
    listen 80;
    server_name monitoring.domaine.fr;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name monitoring.domaine.fr;
    ssl_certificate /etc/nginx/ssl/domaine.fr.cer;
    ssl_certificate_key /etc/nginx/ssl/domaine.fr.key;

    # Sous-requête d'auth, allégée
    location = /oauth2/auth {
        proxy_pass       http://127.0.0.1:4182;
        proxy_set_header Host             $host;
        proxy_set_header X-Real-IP        $remote_addr;
        proxy_set_header X-Scheme         $scheme;
        proxy_set_header Content-Length   "";
        proxy_pass_request_body           off;
    }

    # Routes oauth2-proxy : sign_in, callback, sign_out...
    location /oauth2/ {
        proxy_pass       http://127.0.0.1:4182;
        proxy_set_header Host                    $host;
        proxy_set_header X-Real-IP               $remote_addr;
        proxy_set_header X-Scheme                $scheme;
        proxy_set_header X-Auth-Request-Redirect $request_uri;
    }

    # Appli monitoring, protégée
    location / {
        auth_request /oauth2/auth;
        error_page 401 = /oauth2/sign_in;

        auth_request_set $user  $upstream_http_x_auth_request_user;
        auth_request_set $email $upstream_http_x_auth_request_email;
        proxy_set_header X-User  $user;
        proxy_set_header X-Email $email;

        proxy_pass http://localhost:8520;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```
```bash
sudo systemctl restart oauth2-proxy@monitoring
sudo systemctl reload nginx

```
