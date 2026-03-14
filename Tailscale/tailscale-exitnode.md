# Tailscale Exit Node einrichten (Arch Linux)

Ein Exit Node leitet den gesamten Internet-Traffic von anderen Tailscale-Geräten über diesen Host.

---

## Schritt 1 – Tailscale installieren

```bash
sudo pacman -S tailscale
sudo systemctl enable --now tailscaled
sudo tailscale up
```

Den angezeigten Login-Link im Browser öffnen und die Verbindung genehmigen.

---

## Schritt 2 – Exit Node aktivieren

```bash
sudo tailscale up --advertise-exit-node
```

Bereits laufendes `tailscale up` kann jederzeit mit neuen Flags erneut ausgeführt werden – es überschreibt nur die geänderten Einstellungen.

Mit zusätzlichem Subnet Routing:

```bash
sudo tailscale up --advertise-exit-node --advertise-routes=192.168.1.0/24
```

---

## Schritt 3 – IP Forwarding aktivieren

```bash
sudo sysctl -w net.ipv4.ip_forward=1
sudo sysctl -w net.ipv6.conf.all.forwarding=1
```

Persistent machen – Datei neu anlegen:

```bash
sudo nano /etc/sysctl.d/99-tailscale.conf
```

Inhalt:

```
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

Sofort anwenden ohne Neustart:

```bash
sudo sysctl --system
```

Überprüfen:

```bash
sysctl net.ipv4.ip_forward
sysctl net.ipv6.conf.all.forwarding
# Erwartete Ausgabe: = 1
```

---

## Schritt 4 – Firewall konfigurieren

Falls eine Firewall aktiv ist, muss Forwarding über `tailscale0` erlaubt werden.

**nftables** (Standard auf Arch):

Regeln in `/etc/nftables.conf` im `forward`-Chain ergänzen:

```
chain forward {
    type filter hook forward priority 0; policy drop;
    iifname "tailscale0" accept
    oifname "tailscale0" accept
}
```

```bash
sudo systemctl enable --now nftables
```

**iptables:**

```bash
sudo iptables -A FORWARD -i tailscale0 -j ACCEPT
sudo iptables -A FORWARD -o tailscale0 -j ACCEPT

# Persistent speichern
sudo iptables-save | sudo tee /etc/iptables/iptables.rules
sudo systemctl enable --now iptables
```

**ufw:**

```bash
sudo pacman -S ufw
sudo ufw route allow in on tailscale0
sudo ufw route allow out on tailscale0
sudo ufw enable
```

---

## Schritt 5 – Exit Node im Admin-Panel genehmigen

[login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines) öffnen → Maschine auswählen → **Edit route settings** → Exit Node (und ggf. Subnet Routes) aktivieren.

---

## Schritt 6 – Exit Node auf einem Client verwenden

**CLI:**

```bash
sudo tailscale up --exit-node=<hostname-oder-tailscale-ip>
```

**GUI:** Tray-Menü → *Use exit node…* → gewünschten Host auswählen.

### LAN-Zugriff beibehalten

Damit der lokale Netzwerkzugriff erhalten bleibt, während der Traffic über den Exit Node läuft:

```bash
sudo tailscale up --exit-node=<hostname-oder-tailscale-ip> --exit-node-allow-lan-access=true
```

---

## Schritt 7 – Verifizieren

Vom Client-Gerät:

```bash
curl https://ipinfo.io
```

Die angezeigte IP muss die öffentliche IP des Exit Nodes sein.