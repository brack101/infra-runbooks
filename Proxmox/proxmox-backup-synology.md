# Proxmox-Backups auf Synology NAS (NFS)

VMs, LXC-Container und die Proxmox-Konfiguration werden automatisch auf ein Synology NAS gesichert. Die Synology stellt eine NFS-Freigabe bereit, die Proxmox als Backup-Storage einbindet. Geplante Backup-Jobs sichern alle Maschinen regelmäßig; ein separater Cronjob sichert die Proxmox-Konfiguration unter `/etc/pve`.

---

## Schritt 1 – Synology: NFS aktivieren

DSM → **Systemsteuerung** → **Dateidienste** → Reiter **NFS** → **NFS-Dienst aktivieren** → **Übernehmen**.

---

## Schritt 2 – Synology: Freigegebenen Ordner anlegen

DSM → **Systemsteuerung** → **Freigegebener Ordner** → **Erstellen**.

| Feld | Wert |
|------|------|
| Name | `proxmox-backup` |
| Speicherort | `volume1` (oder gewünschtes Volume) |
| Papierkorb aktivieren | ✗ (spart Platz) |

**Weiter** → **Weiter** → **Fertig**.

---

## Schritt 3 – Synology: NFS-Berechtigung konfigurieren

DSM → **Systemsteuerung** → **Freigegebener Ordner** → `proxmox-backup` markieren → **Bearbeiten** → Reiter **NFS-Berechtigungen** → **Erstellen**.

| Feld | Wert |
|------|------|
| Hostname oder IP | `<proxmox-ip>` (z. B. `192.168.1.10`) |
| Berechtigung | `Lesen/Schreiben` |
| Squash | `Kein Zuordnen` (no\_root\_squash) |
| Sicherheit | `sys` |
| Asynchron | ✓ |
| Verbindungen von nicht-privilegierten Ports erlauben | ✓ |

> ⚠️ `no_root_squash` ist erforderlich, damit Proxmox als `root` in die Freigabe schreiben kann. Schränke die Freigabe auf die konkrete Proxmox-IP ein, um den Zugriff zu begrenzen.

**OK** → **Speichern**.

Den Exportpfad notieren – er ist in der NFS-Regelliste unter **Mountpfad** sichtbar, z. B. `/volume1/proxmox-backup`.

---

## Schritt 4 – Proxmox: NFS-Storage hinzufügen

Proxmox Web-UI → **Datacenter** → **Storage** → **Add** → **NFS**.

| Feld | Wert |
|------|------|
| ID | `synology-backup` |
| Server | `<nas-ip>` |
| Export | `/volume1/proxmox-backup` |
| Content | `VZDump backup file` |
| Max Backups | `3` (oder gewünschte Anzahl) |
| Enable | ✓ |

**Add**.

> ℹ️ Der Wert bei **Max Backups** gilt pro VM/Container. Bei 10 Maschinen und Max Backups = 3 werden maximal 30 Backup-Dateien vorgehalten.

---

## Schritt 5 – Proxmox: Backup-Job einrichten

Proxmox Web-UI → **Datacenter** → **Backup** → **Add**.

| Feld | Wert |
|------|------|
| Storage | `synology-backup` |
| Schedule | `sun 02:00` (oder gewünschter Zeitplan) |
| Selection mode | `All` (alle VMs/Container) |
| Mode | `Snapshot` |
| Compression | `ZSTD` |
| Enabled | ✓ |

**Create**.

> ℹ️ **Snapshot-Modus** sichert laufende Maschinen ohne Downtime. Für VMs ohne QEMU Guest Agent kann der Modus auf `Suspend` oder `Stop` geändert werden.

---

## Schritt 6 – Proxmox-Konfiguration sichern (/etc/pve)

`/etc/pve` ist ein FUSE-Dateisystem und wird von vzdump **nicht** gesichert. Ein Cronjob auf dem Proxmox-Host kopiert die Konfiguration täglich auf die Synology.

### NFS-Share auf dem Proxmox-Host mounten

```bash
mkdir -p /mnt/synology-backup
```

`/etc/fstab` um folgenden Eintrag ergänzen:

```
<nas-ip>:/volume1/proxmox-backup  /mnt/synology-backup  nfs  defaults,_netdev  0  0
```

Mount aktivieren:

```bash
mount /mnt/synology-backup
```

Prüfen:

```bash
df -h /mnt/synology-backup
```

### Cronjob anlegen

```bash
crontab -e
```

Folgenden Eintrag hinzufügen:

```
0 3 * * * tar czf /mnt/synology-backup/pve-config-$(date +\%F).tar.gz /etc/pve 2>/dev/null && find /mnt/synology-backup -name 'pve-config-*.tar.gz' -mtime +30 -delete
```

> ℹ️ Der Cronjob läuft täglich um 03:00 Uhr. Archive älter als 30 Tage werden automatisch gelöscht.

---

## Schritt 7 – Verifikation

### Backup-Job manuell ausführen

Proxmox Web-UI → **Datacenter** → **Backup** → Job auswählen → **Run Now**.

Den Job-Log unter **Tasks** im unteren Bereich beobachten – am Ende erscheint `TASK OK`.

### Backup-Dateien prüfen

Synology DSM → **File Station** → `proxmox-backup` → Unterordner `dump/` → `.vma.zst`-Dateien der gesicherten VMs sind vorhanden.

Alternativ auf dem Proxmox-Host:

```bash
ls -lh /mnt/synology-backup/dump/
```

### Restore testen

Proxmox Web-UI → **Datacenter** → **Storage** → `synology-backup` → **Content** → Backup auswählen → **Restore**.

> ⚠️ Restore-Tests regelmäßig durchführen – ein Backup ist erst dann zuverlässig, wenn der Restore erfolgreich war.

### Proxmox-Konfigurationsbackup prüfen

```bash
ls -lh /mnt/synology-backup/pve-config-*.tar.gz
tar tzf /mnt/synology-backup/pve-config-$(date +%F).tar.gz | head
```
