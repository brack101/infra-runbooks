# Option: Vouch-Proxy + Google OAuth (MFA auf Nginx-Ebene)

Statt MFA in Guacamole kann die Authentifizierung auf Nginx-Ebene mit Vouch-Proxy und Google OAuth erfolgen. Damit erreicht kein Request Guacamole/Tomcat ohne gültige Google-Session – kleinere Angriffsfläche.

```
Browser → Nginx (auth_request) → Vouch-Proxy → Google OAuth + MFA
                ↓ nach Login
          Guacamole (nur Username/Passwort)
```

> ⚠️ Google 2-Step Verification muss im Google Account aktiviert sein – das ist die eigentliche MFA.

## Schritt 1 – Google OAuth Credentials erstellen

1. [console.cloud.google.com](https://console.cloud.google.com) → Neues Projekt
2. **APIs & Services → Credentials → Create Credentials → OAuth 2.0 Client ID**
3. Application type: **Web application**
4. Authorized redirect URIs: `https://guac.deine-domain.ch/auth`
5. **Client ID** und **Client Secret** notieren

## Schritt 2 – Vouch-Proxy Container hinzufügen

`~/guacamole/docker-compose.yml` ergänzen:

```yaml
  vouch:
    image: quay.io/vouch/vouch-proxy:latest
    restart: unless-stopped
    ports:
      - "127.0.0.1:9090:9090"
    volumes:
      - ./vouch:/config
    networks:
      - guac-external
```

## Schritt 3 – Vouch Konfiguration

```bash
mkdir -p ~/guacamole/vouch
vim ~/guacamole/vouch/config.yml
```

```yaml
vouch:
  logLevel: info
  domains:
    - deine-domain.ch
  allowAllUsers: false
  whiteList:
    - deine@deine-domain.ch     # nur diese Email darf rein
  cookie:
    secure: true
    domain: deine-domain.ch
    maxAge: 480            # 8 Stunden Session

oauth:
  provider: google
  client_id: DEINE_CLIENT_ID.apps.googleusercontent.com
  client_secret: DEIN_CLIENT_SECRET
  callback_url: https://guac.deine-domain.ch/auth
  scopes:
    - openid
    - email
    - profile
```

## Schritt 4 – Nginx anpassen

`/etc/nginx/sites-available/guacamole` – im `server 443` Block:

```nginx
    # Vouch-Proxy Endpunkte
    location = /validate {
        internal;
        proxy_pass http://127.0.0.1:9090/validate;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
    }

    location = /auth {
        proxy_pass http://127.0.0.1:9090/auth;
        proxy_set_header Host $http_host;
    }

    location / {
        # Jeder Request wird zuerst gegen Vouch geprüft
        auth_request /validate;
        error_page 401 = @error401;

        proxy_pass         http://127.0.0.1:8080/guacamole/;
        # ... restliche proxy_set_header bleiben gleich ...
    }

    # Bei 401 → Redirect zu Google Login
    location @error401 {
        return 302 https://guac.deine-domain.ch/auth?url=https://$http_host$request_uri;
    }
```

## Schritt 5 – Starten

```bash
cd ~/guacamole
docker compose up -d vouch
docker compose logs -f vouch
sudo nginx -t && sudo systemctl reload nginx
```

## Schritt 6 – TOTP in Guacamole deaktivieren

Da Auth jetzt über Google läuft:

```bash
cd ~/guacamole/guacamole-extensions
mv guacamole-auth-totp-1.5.5.jar guacamole-auth-totp-1.5.5.jar.disabled
docker compose restart guacamole
```
