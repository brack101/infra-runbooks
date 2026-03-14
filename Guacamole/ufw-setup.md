# UFW Setup Skript

Skript herunterladen und ausführen:

```bash
chmod 700 ufw-setup.sh
sudo bash ufw-setup.sh
```

`ufw-setup.sh`:

```bash
#!/bin/bash
# UFW Setup für Guacamole VPS
# Kann mehrmals ausgeführt werden – idempotent
# Ausführen als: sudo bash ufw-setup.sh

set -e

# ── Konfiguration ──────────────────────────────────────
SSH_IP=""        # Deine IP für SSH-Zugang, z.B. "85.1.2.3"
                 # Leer lassen = SSH von überall (nicht empfohlen)
# ───────────────────────────────────────────────────────

echo "=== UFW Setup für Guacamole VPS ==="

# Bestehende Regeln zurücksetzen
ufw --force reset

# Defaults
ufw default deny incoming
ufw default allow outgoing

# SSH
if [ -n "$SSH_IP" ]; then
    echo "[+] SSH erlaubt von $SSH_IP"
    ufw allow from "$SSH_IP" to any port 22 proto tcp comment "SSH"
else
    echo "[!] WARNUNG: SSH von überall erlaubt – SSH_IP setzen empfohlen"
    ufw allow 22/tcp comment "SSH"
fi

# Port 80 – dauerhaft gesperrt, Certbot öffnet kurz bei Erneuerung
ufw deny 80/tcp comment "HTTP – nur für Certbot via pre/post Hook"

# Port 443 – Guacamole HTTPS
ufw allow 443/tcp comment "Guacamole HTTPS"

# UFW aktivieren
ufw --force enable

echo ""
echo "=== Certbot Pre/Post Hooks einrichten ==="

# Pre-Hook – Port 80 öffnen
cat > /etc/letsencrypt/renewal-hooks/pre/open-80.sh << 'HOOK'
#!/bin/bash
ufw allow 80/tcp
HOOK
chmod +x /etc/letsencrypt/renewal-hooks/pre/open-80.sh
echo "[+] Pre-Hook erstellt: /etc/letsencrypt/renewal-hooks/pre/open-80.sh"

# Post-Hook – Port 80 schliessen + Nginx reload
cat > /etc/letsencrypt/renewal-hooks/post/close-80.sh << 'HOOK'
#!/bin/bash
ufw deny 80/tcp
systemctl reload nginx
HOOK
chmod +x /etc/letsencrypt/renewal-hooks/post/close-80.sh
echo "[+] Post-Hook erstellt: /etc/letsencrypt/renewal-hooks/post/close-80.sh"

echo ""
echo "=== Fertig ==="
ufw status verbose
```

> ⚠️ `ufw --force reset` setzt alle Regeln zurück – auf produktiven Servern mit Bedacht ausführen.
