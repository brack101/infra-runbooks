# VNC Headless Desktop – Debian + TigerVNC + Guacamole

## Übersicht

Headless-Desktop auf einer Debian-VM (Proxmox), zugänglich via Guacamole über Tailscale. Kein physischer Monitor, kein Display Manager erforderlich.

**Stack:**
- Debian 13 (Trixie) – KVM-VM auf Proxmox
- TigerVNC – virtuelles Display (`:1`, Port `5901`)
- XFCE – leichtgewichtiger Desktop
- Firefox ESR – als echtes Deb (kein Snap)
- Tailscale – Netzwerkzugang (kein Port-Forwarding)
- Guacamole – Browser-basierter VNC-Client

---

## VM-Setup (Proxmox)

```bash
# ISO herunterladen (Proxmox UI: local → ISO Images → Download from URL)
https://debian.ethz.ch/debian-cd/12.11.0/amd64/iso-cd/debian-12.11.0-amd64-netinst.iso
```

**VM-Ressourcen:** 2 vCPU, 4 GB RAM, 20 GB Disk  
**Installation:** Minimal – nur *SSH server* und *standard system utilities* ankreuzen, kein Desktop.

---

## Ersteinrichtung

### sudo einrichten

```bash
su -
apt install -y sudo
usermod -aG sudo <user>
# Neu einloggen damit Gruppe aktiv wird
```

### Auf Debian 13 (Trixie) upgraden

```bash
sudo sed -i 's/bookworm/trixie/g' /etc/apt/sources.list
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

---

## Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
# Auth-Link im Browser öffnen und Node autorisieren
tailscale ip -4  # IP für Guacamole-Connection notieren
```

---

## VNC-Setup

### Pakete installieren

```bash
sudo apt install -y xfce4 xfce4-goodies tigervnc-standalone-server dbus-x11 firefox-esr
```

> **Hinweis:** Firefox ESR ist auf Debian ein echtes Deb-Paket – kein Snap.

### VNC-Passwort setzen

```bash
vncpasswd
```

### xstartup konfigurieren

```bash
mkdir -p ~/.vnc
cat > ~/.vnc/xstartup << 'EOF'
#!/bin/bash
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS
export XDG_SESSION_TYPE=x11
exec dbus-launch --exit-with-session startxfce4
EOF
chmod +x ~/.vnc/xstartup
```

> **Wichtig:** `dbus-launch` isoliert die Session – ohne dies kollidiert XFCE mit anderen laufenden Sessions.

### Systemd User-Service (Autostart)

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/vncserver@.service << 'EOF'
[Unit]
Description=TigerVNC Server Display %i
After=network.target

[Service]
Type=forking
ExecStartPre=-/usr/bin/vncserver -kill :%i
ExecStop=/usr/bin/vncserver -kill :%i
ExecStart=/usr/bin/vncserver :%i -geometry 1920x1080 -depth 24 -localhost no

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now vncserver@1

# Wichtig: Service läuft auch ohne aktive Login-Session
loginctl enable-linger <user>
```

### Status prüfen

```bash
vncserver -list
systemctl --user status vncserver@1
```

---

## Guacamole Connection

Im Guacamole Admin-UI (**Settings → Connections → New Connection**):

| Parameter | Wert |
|-----------|------|
| Protocol | VNC |
| Hostname | `<tailscale-ip>` |
| Port | `5901` |
| Password | VNC-Passwort aus `vncpasswd` |

Kein SSH-Tunnel, kein Port-Forwarding – Tailscale übernimmt die Verbindung direkt.

---

## Troubleshooting

### Schwarzer Bildschirm

xstartup prüfen – `dbus-launch` muss vorhanden sein:
```bash
cat ~/.vnc/xstartup
vncserver -kill :1 && vncserver :1 -geometry 1920x1080 -depth 24 -localhost no
cat ~/.vnc/<hostname>:1.log
```

### Port nicht erreichbar

```bash
# Auf der VNC-VM:
ss -tlnp | grep 5901

# Vom Guacamole-VPS:
nc -zv <tailscale-ip> 5901
```

### Service startet nicht

```bash
journalctl --user -xeu vncserver@1 --no-pager
# Stale Lock-Files bereinigen:
rm -f /tmp/.X1-lock /tmp/.X11-unix/X1
```

### Display-Konflikt (anderes Display belegt)

Display `:2` verwenden (Port `5902`):
```bash
systemctl --user disable vncserver@1
systemctl --user enable --now vncserver@2
```
Guacamole-Connection entsprechend auf Port `5902` anpassen.

---

## Bekannte Probleme

| Problem | Ursache | Lösung |
|---------|---------|--------|
| Schwarzer Bildschirm | XFCE kollidiert mit bestehender Session | `dbus-launch` in xstartup verwenden |
| `server already running` | Stale Lock-Files | `/tmp/.X1-lock` und `/tmp/.X11-unix/X1` löschen |
| Firefox startet nicht | Snap-Version inkompatibel mit VNC | `firefox-esr` als Deb installieren (Debian), kein Snap |
| Tailscale in LXC | Kein TUN-Device | KVM-VM verwenden statt LXC-Container |
| SSH `Too many authentication failures` | 1Password bietet zu viele Keys an | `ssh -o IdentitiesOnly=yes -i <key>` oder `ssh -o PubkeyAuthentication=no` |