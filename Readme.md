# Installation Oauth2.0 sur serveur
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
# pour un seul   *.DOMAINE.fr
Générer le cookie secret
```bash
openssl rand -base64 32 | tr -- '+/' '-_'
```

```bash
sudo nano /etc/oauth2-proxy/oauth2-proxy.cfg
```
```bash
provider = "keycloak-oidc"
oidc_issuer_url = "https://login.domaine.fr/realms/kpi-test"

client_id = "domaine-software"
client_secret = "LE_SECRET_DU_CLIENT"

# Pas de redirect_url fixe : auto-déduit du Host de la requête entrante

cookie_secret = "LE_SECRET_GENERE"
cookie_domains = [".domaine.fr"]
whitelist_domains = [".domaine.fr"]
cookie_secure = true
cookie_httponly = true
cookie_expire = "168h"
cookie_refresh = "60m"
cookie_samesite = "lax"

email_domains = ["*"]
upstreams = ["static://202"]
http_address = "127.0.0.1:4180"

scope = "openid profile email"

# Passage des identités vers l'app en headers
pass_access_token = true
pass_authorization_header = true
set_xauthrequest = true

standard_logging = true
request_logging = true
auth_logging = true
```
```bash
sudo chown root:oauth2-proxy /etc/oauth2-proxy/oauth2-proxy.cfg
sudo chmod 640 /etc/oauth2-proxy/oauth2-proxy.cfg
```
Service systemd unique
```bash
sudo nano /etc/systemd/system/oauth2-proxy.service
```

```bash
[Unit]
Description=OAuth2 Proxy - shared
After=network.target

[Service]
Type=simple
User=oauth2-proxy
Group=oauth2-proxy
ExecStart=/usr/local/bin/oauth2-proxy --config=/etc/oauth2-proxy/oauth2-proxy.cfg
Restart=on-failure
RestartSec=5

NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now oauth2-proxy
sudo systemctl status oauth2-proxy --no-pager
curl -I http://127.0.0.1:4180/oauth2/auth   # doit renvoyer 401
```

nginx — snippet réutilisable
```bash
sudo nano /etc/nginx/snippets/oauth2-proxy.conf
```
```bash
location = /oauth2/auth {
    proxy_pass       http://127.0.0.1:4180;
    proxy_set_header Host             $host;
    proxy_set_header X-Real-IP        $remote_addr;
    proxy_set_header X-Scheme         $scheme;
    proxy_set_header Content-Length   "";
    proxy_pass_request_body           off;
}

location /oauth2/ {
    proxy_pass       http://127.0.0.1:4180;
    proxy_set_header Host                    $host;
    proxy_set_header X-Real-IP               $remote_addr;
    proxy_set_header X-Scheme                $scheme;
    proxy_set_header X-Auth-Request-Redirect $request_uri;
}
```

Template par site

```bash
server {
    listen 80;
    server_name NOM-DU-SITE.domaine.fr;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name NOM-DU-SITE.domaine.fr;
    ssl_certificate /etc/nginx/ssl/domaine.fr.cer;
    ssl_certificate_key /etc/nginx/ssl/domaine.fr.key;

    include /etc/nginx/snippets/oauth2-proxy.conf;

    location / {
        auth_request /oauth2/auth;
        error_page 401 = /oauth2/sign_in;

        auth_request_set $user  $upstream_http_x_auth_request_user;
        auth_request_set $email $upstream_http_x_auth_request_email;
        proxy_set_header X-User  $user;
        proxy_set_header X-Email $email;

        # ADAPTER selon le site :
        # proxy_pass http://localhost:PORT;   ← backend applicatif
        # ou : root /var/www/html/...; try_files $uri /index.html;  ← statique
    }
}
```



# plusieurs domaine.fr 
```bash
sudo nano /etc/oauth2-proxy/kpi-motoculture.cfg
```
```bash

```
Activer l'instance
```bash

```
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
  "clientId": "sites-multi",
  "protocol": "openid-connect",
  "publicClient": false,
  "standardFlowEnabled": true,
  "directAccessGrantsEnabled": false,
  "serviceAccountsEnabled": false,
  "redirectUris": [
    "https://kpi-motoculture.domaine.fr/oauth2/callback",
    "https://monitoring.domaine.fr/oauth2/callback",
    "https://kpi-jardinage.domaine.fr/oauth2/callback"
  ],
  "webOrigins": [
    "https://kpi-motoculture.domaine.fr",
    "https://monitoring.domaine.fr",
    "https://kpi-jardinage.domaine.fr"
  ],
  "attributes": {
    "post.logout.redirect.uris": "https://*.domaine.fr/*",
    "backchannel.logout.session.required": "true"
  },
  "defaultClientScopes": ["web-origins", "profile", "roles", "email"]
}

```
désactiver la verification de mail
``` realm setting > login > Email as username    et   Login with email    off```

authorization role, droits application
```client > setting > Capability config - Authorization -> ON```

ROLE
```client > role > create role > monitoring_Read ```

GROUP entreprises
```groups > create > entreprise1 ```
```groups > create > entreprise1 > monitoring_read```

USER
créer les droits d'applications
```client > Authorization > Scope > Read / Write / Admin ...```
```client > Authorization > Ressouces > monitoring-Read > Read```
```client > Authorization > Policy > Role > Read policy > monitoring-read-policy > Assign role > client role > Read```
```client > Authorization > Permission > scope based > read authorisation > read policy```

USERS assign group role application
```groups > entreprise1 > monitoring-read > role mapping > client  role > monitoring_Read```

