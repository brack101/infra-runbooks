# Security Monitoring & Nützliche Befehle

## Security Monitoring

```bash
# Nginx – blockierte Requests, Rate Limiting
sudo tail -f /var/log/nginx/error.log

# Nginx – alle Zugriffe
sudo tail -f /var/log/nginx/access.log

# Fail2ban – gebannte IPs
sudo fail2ban-client status guacamole
sudo tail -f /var/log/fail2ban.log

# Guacamole – fehlgeschlagene Logins
docker compose logs -f guacamole | grep -i "failed\|error\|warn"

# UFW – blockierte Verbindungen live
sudo tail -f /var/log/ufw.log

# Nur blockierte Verbindungen
sudo grep "BLOCK" /var/log/ufw.log

# Top IPs die blockiert wurden
sudo grep "BLOCK" /var/log/ufw.log | grep -oP 'SRC=\K[\d.]+' | sort | uniq -c | sort -rn | head -20

# Blockierte IPs mit Zielport
sudo grep "BLOCK" /var/log/ufw.log | grep -oP 'SRC=\K[\d.]+ .* DPT=\K[\d]+' | sort | uniq -c | sort -rn | head -20

# Nur heute
sudo grep "BLOCK" /var/log/ufw.log | grep "$(date '+%b %e')" | grep -oP 'SRC=\K[\d.]+' | sort | uniq -c | sort -rn
```

Konkret nach Angriffen suchen:

```bash
# Top IPs nach Anfragen (potenzielle Scanner)
sudo awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Alle gebannten IPs
sudo fail2ban-client status guacamole | grep "Banned IP"

# Rate Limiting Treffer
sudo grep "limiting requests" /var/log/nginx/error.log | tail -20
```

## Nützliche Befehle

```bash
# Container Status
cd ~/guacamole
docker compose ps
docker compose logs -f guacamole
docker compose logs -f guacd

# Container neu starten
docker compose restart guacamole

# Zertifikat prüfen
sudo certbot certificates

# Fail2ban Status
sudo fail2ban-client status guacamole

# UFW Status
sudo ufw status verbose

# Nginx Konfiguration testen
sudo nginx -t

# Tailscale Status
tailscale status
tailscale ping 100.x.x.x

# Top IPs nach Anzahl Requests
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Nur heute
grep "$(date '+%d/%b/%Y')" /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -20

# Nur 4xx/5xx Fehler pro IP
awk '$9 ~ /^[45]/' /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -20

# Nur 444 (CONNECT-Versuche)
awk '$9 == "444"' /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -rn

# CONNECT-Versuche mit IP, Zeitstempel und Ziel
awk '$6 == "\"CONNECT"' /var/log/nginx/access.log | awk '{print $1, $4, $7}' | sort

# Nur IPs bei CONNECT zählen
awk '$6 == "\"CONNECT"' /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -rn
```
