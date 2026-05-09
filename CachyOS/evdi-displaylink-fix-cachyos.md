# evdi / DisplayLink Fix auf CachyOS (Clang-Kernel)

## Problem

Nach einem System-Update (`pacman -Syyu`) startet `displaylink.service` nicht mehr:

```
modprobe: FATAL: Module evdi not found
```

oder:

```
modprobe: ERROR: could not insert 'evdi': Invalid argument
modprobe: ERROR: could not insert 'evdi': Unknown symbol in module
```

## Ursachen

Auf CachyOS wird der Kernel mit **Clang/LLD** kompiliert. DKMS baut Module standardmässig mit GCC, was zu zwei Problemen führt:

1. **Unknown symbol** – Clang-spezifische Symbole werden nicht gefunden
2. **BTF-Validierungsfehler** – die Debug-Typeninfo (BPF Type Format) ist inkompatibel mit dem Kernel

## Fix

### 1. Kernel-Header installieren

```bash
sudo pacman -S linux-cachyos-headers
```

### 2. DKMS-Konfiguration anpassen

```bash
sudo nano /usr/src/evdi-1.14.16/dkms.conf
```

Die `MAKE[0]`-Zeile ersetzen:

```
MAKE[0]="make all INCLUDEDIR=/lib/modules/$kernelver/build/include KVERSION=$kernelver DKMS_BUILD=1 CC=clang LD=ld.lld EXTRA_CFLAGS='-g0'"
```

### 3. Modul neu bauen

```bash
sudo dkms remove evdi/1.14.16 --all
sudo dkms install evdi/1.14.16 -k $(uname -r)
```

### 4. BTF-Sektion entfernen

Clang bettet BTF-Debuginfo ein, die der Kernel ablehnt. Manuell entfernen:

```bash
sudo zstd -d /lib/modules/$(uname -r)/updates/dkms/evdi.ko.zst
sudo objcopy --remove-section=.BTF /lib/modules/$(uname -r)/updates/dkms/evdi.ko
sudo zstd --rm /lib/modules/$(uname -r)/updates/dkms/evdi.ko \
    -o /lib/modules/$(uname -r)/updates/dkms/evdi.ko.zst
```

### 5. Modul laden und Service starten

```bash
sudo modprobe drm_ttm_helper
sudo modprobe evdi
sudo systemctl start displaylink
systemctl status displaylink
```

---

## Dauerhafter Fix (nach Kernel-Updates)

Damit Schritte 3 und 4 nach jedem DKMS-Rebuild automatisch ausgeführt werden, ein Post-Install-Script erstellen:

```bash
sudo nano /usr/src/evdi-1.14.16/dkms-post-install.sh
```

Inhalt:

```bash
#!/bin/bash
set -e
KO="/lib/modules/${kernelver}/updates/dkms/evdi.ko"
zstd -d "${KO}.zst"
objcopy --remove-section=.BTF "${KO}"
zstd --rm "${KO}" -o "${KO}.zst"
```

```bash
sudo chmod +x /usr/src/evdi-1.14.16/dkms-post-install.sh
```

In `dkms.conf` hinzufügen (nach der `MAKE[0]`-Zeile):

```
POST_INSTALL="dkms-post-install.sh"
```

---

## Automatisches Laden beim Systemstart

```bash
echo "drm_ttm_helper" | sudo tee /etc/modules-load.d/drm_ttm_helper.conf
```

---

## Diagnose-Befehle

```bash
# DKMS-Status
dkms status

# Modul-Info
modinfo evdi | grep -E 'version|vermagic|filename|depends'

# Fehlende Symbole prüfen
zstd -d -c /lib/modules/$(uname -r)/updates/dkms/evdi.ko.zst > /tmp/evdi.ko
bash -c 'nm /tmp/evdi.ko | grep " U " | awk "{print \$2}" | while read sym; do grep -qw "$sym" /proc/kallsyms || echo "MISSING: $sym"; done'

# Kernel-Meldungen
sudo dmesg | grep -i evdi | tail -10
```

---

## Hintergrund

| Problem | Ursache | Fix |
|---|---|---|
| `Unknown symbol` | evdi mit GCC gebaut, Kernel mit Clang | `CC=clang LD=ld.lld` in dkms.conf |
| `BTF: -22` | Clang bettet BTF ein, Kernel lehnt es ab | `.BTF`-Sektion mit objcopy entfernen |
| `bad vermagic` | Header passen nicht zum Kernel | `linux-cachyos-headers` installieren |
