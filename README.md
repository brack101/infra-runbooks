# infra-runbooks

Runbooks und Setup-Anleitungen für selbst gehostete Dienste: Netzwerk, Hardening, Remote-Zugriff und Infrastruktur-Automatisierung.

## Guides

| Guide | Beschreibung |
|-------|-------------|
| [Guacamole Remote Access](Guacamole/guacamole.md) | Apache Guacamole auf Hetzner VPS mit Nginx, Tailscale, TOTP-MFA, Fail2ban und Let's Encrypt |
| [Tailscale Exit Node](Tailscale/tailscale-exitnode.md) | Arch-Linux-Maschine als Tailscale Exit Node einrichten – inkl. IP Forwarding, Firewall-Konfiguration und Hardening |
| [Uptime Kuma (Synology)](UptimeKuma/uptime-kuma-synology.md) | Uptime Kuma auf einem Synology NAS via Docker Compose installieren und betreiben |
| [VNC Headless Desktop](VNC-Headless-Desktop/vnx-headless-desktop.md) | Headless-Desktop auf Debian-VM (Proxmox) mit TigerVNC und XFCE, zugänglich via Guacamole über Tailscale |
| [Proxmox-Backup (Synology)](Proxmox/proxmox-backup-synology.md) | VMs, LXC-Container und Proxmox-Konfiguration automatisch auf Synology NAS sichern via NFS |
| [evdi / DisplayLink Fix (CachyOS)](CachyOS/evdi-displaylink-fix-cachyos.md) | DisplayLink nach Kernel-Update auf CachyOS (Clang-Kernel) reparieren |
