# Guacamole Remote Access – Komplettanleitung

## Architektur

```
Internet
  ↓ Port 443 (Port 80 nur für Let's Encrypt Challenge)
┌─────────────────────────────────────────────────────┐
│  Hetzner Cloud VPS (Ubuntu 24.04, CX23)             │
│  ┌─────────────────────────────────────────────┐    │
│  │ Nginx (TLS 1.2/1.3 · Security Headers       │    │
│  │        · Rate Limiting)                     │    │
│  │   ↓ localhost:8080                          │    │
│  │ Guacamole + TOTP-Extension (MFA)            │    │
│  │   ↓ Tailscale 100.x.x.x                     │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
  ↓ WireGuard-Tunnel (ausgehend, kein offener Port zuhause)
┌─────────────────────────────────────────────────────┐
│  Heimnetzwerk                                       │
│  ┌─────────────────────────────────────────────┐    │
│  │ Homeserver                                  │    │
│  │   └── Tailscale-Node                        │    │
│  └─────────────────────────────────────────────┘    │
│                                                     │
│  ├── Linux-Host (SSH)                               │
│  ├── Windows-PC (RDP)                               │
│  └── weitere Geräte (VNC)                           │
└─────────────────────────────────────────────────────┘
```

- Guacamole ist **nie direkt** erreichbar – immer über Nginx
- Kein offener Port im Heimnetz – Tailscale baut den Tunnel aktiv zum VPS auf
- MFA über eingebaute TOTP-Extension (kein externes Auth-System nötig)

---

## Schritt 0 – Hetzner VPS einrichten

### VPS erstellen

1. [hetzner.com/cloud](https://hetzner.com/cloud) → Neues Projekt → Server erstellen
2. **Image**: Ubuntu 24.04
3. **Typ**: CX23 (2 vCPU, 4 GB RAM)
4. **Region**: Falkenstein (DE) oder Helsinki (FI)
5. **SSH-Key** hinterlegen (kein Passwort-Login)
6. Server erstellen → IPv4-Adresse notieren

### Hetzner Cloud Firewall

Im Hetzner Panel unter **Firewalls → Inbound Rules**:

| Protokoll | Port | Quelle | Zweck |
|-----------|------|--------|-------|
| TCP | 22 | Deine IP | SSH |
| TCP | 80 | Any | Let's Encrypt Challenge (wird später durch UFW gesperrt – Certbot pre/post Hook öffnet ihn temporär bei Erneuerung) |
| TCP | 443 | Bekannte IPs | Guacamole HTTPS |

> ℹ️ Port 443 sollte auf bekannte IPs eingeschränkt werden (Zuhause, Büro, Mobile). Tailscale DERP-Relay läuft über die Tailscale-eigenen Server – der VPS braucht dafür keinen offenen Inbound-Port 443.

> ℹ️ Tailscale braucht keinen eigenen Inbound-Port in der Hetzner Firewall. Der VPS initiiert die UDP-Verbindung ausgehend (immer erlaubt) – Tailscale macht hole punching und baut einen direkten WireGuard-Tunnel auf. Solange kein direkter Tunnel möglich ist, fällt Tailscale auf DERP-Relay-Server über HTTPS zurück:
> ```
> via DERP(<region>) in 46ms        ← Relay bis hole punching klappt
> via <vps-ip>:41641 in 15ms  ← direkter Tunnel nach hole punching
> ```
> ⚠️ Port 80 muss offen sein – Let's Encrypt braucht ihn für die Challenge.

### Ersten Login absichern

```bash
# Als root einloggen
ssh root@<vps-ip>

# System aktualisieren
apt update && apt upgrade -y

# Neuen User anlegen
adduser guac
usermod -aG sudo guac

# SSH-Key für neuen User kopieren
rsync --archive --chown=guac:guac ~/.ssh /home/guac

# SSH Passwort-Login deaktivieren
sed -i 's/^#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sed -i 's/^PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart ssh

# Ab jetzt als guac arbeiten
su - guac
```

### Docker installieren

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker guac
newgrp docker   # Gruppe sofort aktivieren ohne neu einloggen
```

---

## Schritt 1 – Tailscale

### Auf dem VPS

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

### Zuhause (Proxmox VM/LXC – immer-an-Gerät)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-routes=192.168.1.0/24 --accept-dns=false
```

Im Tailscale Admin-Panel: Route Approval für `192.168.1.0/24` aktivieren.

### Verbindung testen

```bash
tailscale status
tailscale ping 100.x.x.x
```

---

## Schritt 2 – Verzeichnisstruktur

```bash
cd ~
mkdir -p guacamole/guacamole-extensions
cd guacamole
```

Vollständige Struktur:

```
/home/guac/guacamole/
├── docker-compose.yml
├── .env
├── initdb.sql
└── guacamole-extensions/
    └── guacamole-auth-totp-1.5.5.jar
```

### TOTP-Extension herunterladen

```bash
curl -L https://downloads.apache.org/guacamole/1.5.5/binary/guacamole-auth-totp-1.5.5.tar.gz \
  | tar -xz --strip-components=1 -C guacamole-extensions \
  guacamole-auth-totp-1.5.5/guacamole-auth-totp-1.5.5.jar
```

---

## Schritt 3 – Umgebungsvariablen

```bash
echo "POSTGRES_PASSWORD=$(openssl rand -hex 32)" > .env
chmod 600 .env        # nur Owner darf lesen/schreiben
chmod 750 ~/guacamole # Verzeichnis nicht für andere lesbar
```

Nach der DB-Initialisierung `initdb.sql` löschen – wird nicht mehr gebraucht:

```bash
rm ~/guacamole/initdb.sql
```

---

## Schritt 4 – Docker Compose

`docker-compose.yml`:

```yaml
services:

  postgres:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      POSTGRES_DB: guacamole_db
      POSTGRES_USER: guacamole
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - guac-internal
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U guacamole"]
      interval: 10s
      timeout: 5s
      retries: 5

  guacd:
    image: guacamole/guacd:1.5.5
    restart: unless-stopped
    networks:
      - guac-internal
      - guac-external    # benötigt für Tailscale-Zugriff auf Heimgeräte

  guacamole:
    image: guacamole/guacamole:1.5.5
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
      guacd:
        condition: service_started
    environment:
      GUACD_HOSTNAME: guacd
      POSTGRESQL_HOSTNAME: postgres
      POSTGRESQL_DATABASE: guacamole_db
      POSTGRESQL_USER: guacamole
      POSTGRESQL_PASSWORD: ${POSTGRES_PASSWORD}
      TOTP_ENABLED: "true"
      TOTP_ISSUER: "Guacamole"
      TOTP_DIGITS: "6"
      TOTP_PERIOD: "30"
    ports:
      - "127.0.0.1:8080:8080"    # nur localhost, nie direkt exponieren
    volumes:
      - ./guacamole-extensions:/etc/guacamole/extensions:ro
    networks:
      - guac-internal
      - guac-external

networks:
  guac-internal:
    internal: true    # postgres + guacd: kein Internet-Zugriff
  guac-external:
    internal: false   # guacamole + guacd: Port-Mapping + Tailscale

volumes:
  pgdata:
```

> ⚠️ Beide `guacd` und `guacamole` brauchen `guac-external`:
> - `guacamole` braucht es für das Port-Mapping zu localhost:8080
> - `guacd` braucht es um Tailscale-IPs der Heimgeräte zu erreichen

---

## Schritt 5 – Datenbank initialisieren

```bash
# SQL-Schema generieren
docker run --rm guacamole/guacamole:1.5.5 \
  /opt/guacamole/bin/initdb.sh --postgresql > initdb.sql

# Postgres starten
docker compose up -d postgres
sleep 10

# Schema einspielen
docker exec -i $(docker compose ps -q postgres) \
  psql -U guacamole -d guacamole_db < initdb.sql

# Alle Container starten
docker compose up -d

# Status prüfen – PORTS muss "127.0.0.1:8080->8080/tcp" zeigen
docker compose ps
```

---

## Schritt 6 – Let's Encrypt Zertifikat

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d guac.deine-domain.ch --email deine@email.ch --agree-tos
```

### Pre/Post Hook – Port 80 nur während Challenge öffnen

Port 80 in UFW generell gesperrt – Certbot öffnet ihn nur für die Dauer der Challenge:

```bash
sudo bash -c 'cat > /etc/letsencrypt/renewal-hooks/pre/open-80.sh << EOF
#!/bin/bash
ufw allow 80/tcp
EOF'

sudo bash -c 'cat > /etc/letsencrypt/renewal-hooks/post/close-80.sh << EOF
#!/bin/bash
ufw deny 80/tcp
systemctl reload nginx
EOF'

sudo chmod +x /etc/letsencrypt/renewal-hooks/pre/open-80.sh
sudo chmod +x /etc/letsencrypt/renewal-hooks/post/close-80.sh
```

Ablauf:
```
Certbot Timer läuft
  ↓ pre-hook:  ufw allow 80/tcp   ← Port kurz öffnen
  ↓ Challenge läuft
  ↓ Zertifikat erneuert
  ↓ post-hook: ufw deny 80/tcp    ← Port sofort wieder schliessen
               systemctl reload nginx
```

### Erneuerung testen

```bash
sudo certbot renew --dry-run
```

### Wie die Challenge funktioniert

Port 80 bleibt offen, Nginx leitet alles auf 443 um – ausser den Challenge-Pfad:

```
Normaler Request:   Browser → Port 80 → Nginx → 301 → Port 443
Challenge Request:  Let's Encrypt → Port 80 → /.well-known/acme-challenge/token → ✅
```

---

## Schritt 7 – Nginx Konfiguration

`/etc/nginx/nginx.conf` – Rate Limiting Zonen im `http`-Block ergänzen:

```nginx
http {
    server_tokens off;    # Nginx-Version nicht preisgeben
    limit_req_zone  $binary_remote_addr zone=guac:10m rate=60r/m;
    limit_conn_zone $binary_remote_addr zone=guac_conn:10m;
    ...
}
```

`/etc/nginx/sites-available/guacamole`:

```nginx
server {
    listen 443 ssl;
    server_name guac.deine-domain.ch;

    # HTTP CONNECT blockieren (Proxy-Tunnel-Versuche)
    if ($request_method = CONNECT) {
        return 444;
    }

    ssl_certificate     /etc/letsencrypt/live/guac.deine-domain.ch/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/guac.deine-domain.ch/privkey.pem;

    # TLS Hardening
    ssl_protocols             TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_timeout       1d;
    ssl_session_cache         shared:SSL:10m;

    # Security Headers
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    add_header X-Frame-Options            SAMEORIGIN always;
    add_header X-Content-Type-Options     nosniff always;
    add_header Referrer-Policy            "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy         "geolocation=(), microphone=(), camera=()" always;

    # Rate Limiting
    limit_req  zone=guac burst=50 nodelay;
    limit_conn guac_conn 10;

    location / {
        proxy_pass         http://127.0.0.1:8080/guacamole/;
        proxy_buffering    off;
        proxy_http_version 1.1;

        # WebSocket – kritisch für stabile Verbindung
        # Connection muss "upgrade" sein (nicht $http_upgrade)
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "upgrade";

        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_cookie_path  /guacamole/ /;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}

server {
    listen 80;
    server_name guac.deine-domain.ch;

    # HTTP CONNECT blockieren
    if ($request_method = CONNECT) {
        return 444;
    }

    # Let's Encrypt Challenge
    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    # Alles andere → HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/guacamole /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default   # Default-Site entfernen
sudo nginx -t && sudo systemctl reload nginx
```

---

## Schritt 8 – UFW Firewall

UFW-Regeln und Certbot Pre/Post-Hooks können zusammen mit dem Setup-Skript gesetzt werden: [ufw-setup.md](ufw-setup.md).

Manuell:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

# SSH – auf eigene IP einschränken
sudo ufw allow from <deine-ip> to any port 22 proto tcp

# Web
# Port 80 wird nur durch Certbot pre/post Hook geöffnet – nicht dauerhaft
sudo ufw deny 80/tcp      # dauerhaft gesperrt, Certbot öffnet kurz bei Erneuerung
sudo ufw allow 443/tcp    # Guacamole HTTPS

sudo ufw enable
sudo ufw status verbose
```

---

## Schritt 9 – Fail2ban

```bash
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
```

`/etc/fail2ban/jail.d/guacamole.conf`:

```ini
[guacamole]
enabled   = true
port      = 80,443
filter    = guacamole[logging=webapp]
logpath   = /var/lib/docker/containers/*/*.log
maxretry  = 5
findtime  = 300
bantime   = 3600
```

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status guacamole
```

> Die `filter.d/guacamole.conf` ist bereits vorhanden – nicht anfassen.

---

## Schritt 10 – Guacamole Ersteinrichtung

### Admin-User in der DB anlegen

Der Standard-`guacadmin` hat Permission-Probleme beim Passwort-Ändern. Neuen Admin direkt in der DB anlegen.

**Schritt 1 – Hash und Salt für eigenes Passwort generieren:**

```bash
PASS="<dein-passwort>"
SALT=$(openssl rand -hex 32 | tr 'a-f' 'A-F')
HASH=$(printf "%s%s" "$PASS" "$SALT" | sha256sum | awk '{print toupper($1)}')
echo "SALT: $SALT"
echo "HASH: $HASH"
```

Die ausgegebenen Werte `SALT` und `HASH` im nächsten Schritt einsetzen.

**Schritt 2 – User in der DB anlegen:**

```bash
cd ~/guacamole
docker exec -it $(docker compose ps -q postgres) psql -U guacamole -d guacamole_db
```

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

INSERT INTO guacamole_entity (name, type) VALUES ('admin', 'USER');

INSERT INTO guacamole_user (entity_id, password_hash, password_salt, password_date)
SELECT entity_id,
  decode('<HASH>', 'hex'),
  decode('<SALT>', 'hex'),
  CURRENT_TIMESTAMP
FROM guacamole_entity WHERE name = 'admin';

INSERT INTO guacamole_system_permission (entity_id, permission)
SELECT entity_id, unnest(ARRAY['ADMINISTER','CREATE_USER','CREATE_CONNECTION','CREATE_CONNECTION_GROUP','CREATE_SHARING_PROFILE','CREATE_USER_GROUP']::guacamole_system_permission_type[])
FROM guacamole_entity WHERE name = 'admin';

INSERT INTO guacamole_user_permission (entity_id, affected_user_id, permission)
SELECT e.entity_id, u.user_id, unnest(ARRAY['READ','UPDATE','DELETE','ADMINISTER']::guacamole_object_permission_type[])
FROM guacamole_entity e, guacamole_user u WHERE e.name = 'admin';

\q
```

### Einloggen

1. `https://guac.deine-domain.ch` öffnen
2. Login: `admin` / `<dein-passwort>`
3. TOTP-QR-Code scannen (Google Authenticator / Aegis)
4. `guacadmin` deaktivieren: **Settings → Users → guacadmin → Disable**

---

## Schritt 11 – SSH-Key für Guacamole

guacd baut die SSH-Verbindung vom VPS auf – der private Key muss dort vorhanden sein:

```bash
# Neues Keypair auf dem VPS generieren
ssh-keygen -t ed25519 -f ~/.ssh/guacamole_key -N ""

# Public Key anzeigen → auf Heimgerät hinterlegen
cat ~/.ssh/guacamole_key.pub
```

Auf dem Heimgerät:

```bash
echo "ssh-ed25519 AAAA..." >> ~/.ssh/authorized_keys
```

---

## Schritt 12 – Verbindungen anlegen

**Settings → Connections → New Connection**

Tailscale-IPs prüfen:
```bash
tailscale status
```

**SSH (Linux):**

```
Name:        <heimgerät>
Protocol:    SSH
Hostname:    100.x.x.x        ← Tailscale-IP
Port:        22
Username:    guac
Private key: (Inhalt von ~/.ssh/guacamole_key einfügen)
```

**RDP (Windows):**

```
Name:                      windows-pc
Protocol:                  RDP
Hostname:                  100.x.x.x
Port:                      3389
Username:                  guac
Password:                  Windows-Passwort
Security:                  NLA
Ignore server certificate: ✓
```

**VNC:**

```
Name:     vnc-gerät
Protocol: VNC
Hostname: 100.x.x.x
Port:     5900
Password: VNC-Passwort
```

> ⚠️ Immer Tailscale-IPs (`100.x.x.x`) verwenden – nie direkte LAN-IPs (`192.168.x.x`). Nur so ist sichergestellt dass der Traffic immer über den verschlüsselten WireGuard-Tunnel läuft.

---

## Sicherheits-Checkliste

| Massnahme | Womit |
|-----------|-------|
| MFA erzwungen | Guacamole TOTP-Extension |
| Kein direkter App-Zugriff | `127.0.0.1:8080` Binding |
| TLS 1.2/1.3 only | Nginx SSL-Config |
| HSTS, X-Frame, nosniff | Security Headers |
| Brute-Force-Schutz | Fail2ban + Rate Limiting |
| Container-Isolation | Docker Network `internal: true` |
| Secrets extern | `.env` Datei (chmod 600) |
| Tunnel verschlüsselt | Tailscale WireGuard |
| Keine offenen Ports zuhause | Tailscale Reverse-Tunnel |
| `guacadmin` deaktiviert | Manuelle Ersteinrichtung |
| Automatische Zertifikatserneuerung | Certbot Timer + Deploy-Hook |
| SSH nur mit Key | Passwort-Login deaktiviert |
| Hetzner Firewall | Zusätzliche Schutzschicht vor UFW |

---

## Option: Vouch-Proxy + Google OAuth (MFA auf Nginx-Ebene)

Siehe [vouch-proxy.md](vouch-proxy.md).

---

## Updates

### Container aktualisieren (gleiche Version)

```bash
cd ~/guacamole
docker compose pull
docker compose up -d
docker image prune -f
```

### Auf neue Guacamole-Version updaten

1. Release Notes prüfen: [guacamole.apache.org/releases](https://guacamole.apache.org/releases)
2. Version in `docker-compose.yml` anpassen:

```bash
nano docker-compose.yml
# guacamole/guacamole:1.5.5 → guacamole/guacamole:1.6.0
# guacamole/guacd:1.5.5     → guacamole/guacd:1.6.0
```

3. Neu starten:

```bash
docker compose pull
docker compose up -d
docker image prune -f
```

> ⚠️ Bei Major-Version-Sprüngen immer Release Notes lesen – manchmal ändert sich das DB-Schema und braucht eine Migration.

---

## Security Monitoring & Nützliche Befehle

Siehe [monitoring.md](monitoring.md).

---

## UFW Setup Skript

Siehe [ufw-setup.md](ufw-setup.md).
