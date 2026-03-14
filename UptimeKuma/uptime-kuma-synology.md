# Uptime Kuma auf Synology NAS installieren

Uptime Kuma ist ein schnelles, einfach zu bedienendes Monitoring-Tool zur Überwachung der Verfügbarkeit und Performance von Websites, Diensten und APIs. Es unterstützt verschiedene Protokolle – darunter HTTP/HTTPS, TCP, DNS und ICMP (Ping) – und sendet bei Ausfällen sofortige Benachrichtigungen über Plattformen wie Discord, Slack und E-Mail.

---

## Voraussetzungen

- Synology NAS mit DSM 7.x
- **Container Manager** installiert (Paket-Zentrum → Container Manager)

---

## Schritt 1 – Image herunterladen

Container Manager → **Registry** → Suchfeld: `louislam/uptime-kuma` → **Herunterladen** → Tag `1` auswählen → **Übernehmen**.

Unter **Image** warten bis der Download abgeschlossen ist.

---

## Schritt 2 – Container erstellen

Container Manager → **Container** → **Erstellen**.

### Allgemeine Einstellungen

| Feld | Wert |
|------|------|
| Image | `louislam/uptime-kuma:1` |
| Containername | `uptime-kuma` |
| Automatischen Neustart aktivieren | ✓ |

**Weiter**.

### Port-Einstellungen

| Lokaler Port | Container-Port | Protokoll |
|-------------|---------------|-----------|
| `8086` (gewünschter Port, auf dem Uptime Kuma erreichbar sein soll) | `3001` | TCP |

**Weiter**.

### Volume-Einstellungen

> ⚠️ Den Ordner vorher in der File Station anlegen: `docker` → **Erstellen** → `uptimekuma`. Container Manager erstellt fehlende Verzeichnisse nicht automatisch.

| Lokaler Pfad | Container-Pfad |
|-------------|----------------|
| `/volume1/docker/uptimekuma` | `/app/data` |

**Weiter**.

---

## Schritt 3 – Erster Login

Im Browser öffnen:

```
http://<nas-ip>:8086
```

Benutzername und Passwort für den Admin-Account festlegen → **Create**.

---

## Schritt 4 – Ersten Monitor anlegen

1. **Add New Monitor** klicken
2. Monitor-Typ wählen (HTTP, TCP Port, Ping, …)
3. URL oder Hostname eintragen
4. Intervall und Benachrichtigungen konfigurieren
5. **Save**

---

## Updates

Container Manager → **Projekt** → `uptime-kuma` → **Aktion** → **Image aktualisieren** → Container wird automatisch neu gestartet.

---

## Sicherheitshinweise

- Port `8086` ist bis zur Einrichtung des ersten Admin-Users ungeschützt – sofort nach dem ersten Start absichern
- Zugriff über HTTPS absichern: DSM → **Anmeldeportal** → **Reverse-Proxy** → Regel für Port 8086 mit eigenem Zertifikat anlegen
- Port `8086` in der Synology Firewall auf interne IPs einschränken: Systemsteuerung → **Firewall**
