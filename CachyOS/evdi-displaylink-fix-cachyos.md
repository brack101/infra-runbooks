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

DKMS baut Module standardmässig mit GCC. CachyOS wechselt gelegentlich zwischen **GCC** und **Clang** als Kernel-Compiler, was zu zwei Problemen führen kann:

1. **Unknown symbol / bad vermagic** – Compiler-Mismatch zwischen Modul und Kernel
2. **BTF-Validierungsfehler** – die Debug-Typeninfo (BPF Type Format) ist inkompatibel mit dem Kernel

## Compiler prüfen

Zuerst immer prüfen mit welchem Compiler der laufende Kernel gebaut wurde:

```bash
cat /proc/version
```

- Zeigt `gcc` → Standard-Build, kein `LLVM=1` nötig
- Zeigt `clang` → `LLVM=1` erforderlich

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

**Bei GCC-Kernel (aktuell):**
```
MAKE[0]="make all INCLUDEDIR=/lib/modules/$kernelver/build/include KVERSION=$kernelver DKMS_BUILD=1 KBUILD_SKIP_BTF=1"
```

**Bei Clang-Kernel:**
```
MAKE[0]="make all INCLUDEDIR=/lib/modules/$kernelver/build/include KVERSION=$kernelver DKMS_BUILD=1 LLVM=1 KBUILD_SKIP_BTF=1"
```

- `LLVM=1` setzt automatisch `CC=clang`, `LD=ld.lld`, `AR=llvm-ar` etc. – nur bei Clang-Kernel
- `KBUILD_SKIP_BTF=1` verhindert BTF-Generierung, die der Kernel ablehnt – immer nötig

### 3. Modul neu bauen und laden

```bash
sudo dkms remove evdi/1.14.16 --all
sudo dkms autoinstall
sudo modprobe drm_ttm_helper
sudo modprobe evdi
sudo systemctl start displaylink
systemctl status displaylink
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
| `Unknown symbol` | Compiler-Mismatch Modul/Kernel | `LLVM=1` bei Clang-Kernel, weglassen bei GCC |
| `BTF: -22` | BTF-Debuginfo inkompatibel | `KBUILD_SKIP_BTF=1` in dkms.conf |
| `bad vermagic` | Header passen nicht zum Kernel | `linux-cachyos-headers` installieren |
| Build-Fehler `unknown argument` | `LLVM=1` bei GCC-Kernel gesetzt | `LLVM=1` entfernen, `cat /proc/version` prüfen |
