---
date: '2025-03-14T12:44:09+01:00'
draft: false
title: 'Proxmox SDN Installation und Konfiguration'
cascade:
  type: docs
---

### Installation Proxmox VE auf einem dedizierten Server (Hetzner, 1 IP, SDN)

#### 1. Basisinstallation Debian 12 (Bookworm)

- Server ins Rescue-System booten
- `installimage` ausführen, Debian 12 (Bookworm) wählen
- RAID-Level, Hostnamen, Partitionierung einstellen
- Installation starten, danach Neustart durchführen

#### 2. APT-Quellen und Proxmox installieren

- Proxmox GPG-Key und Repository hinzufügen:

```bash
curl -o /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg http://download.proxmox.com/debian/proxmox-release-bookworm.gpg
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-install-repo.list
```

- Enterprise-Repository deaktivieren:

```bash
echo '# deb https://enterprise.proxmox.com/debian/pve bookworm InRelease' > /etc/apt/sources.list.d/pve-enterprise.list
```

- System aktualisieren und Proxmox installieren:

```bash
apt update
apt full-upgrade
apt install proxmox-ve
reboot
```

#### 3. Netzwerk-Konfiguration für SDN vorbereiten

- `/etc/network/interfaces` konfigurieren (Beispiel, Interface anpassen!):

```bash
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback
iface lo inet6 loopback

auto enp0s31f6
iface enp0s31f6 inet manual

auto vmbr0
iface vmbr0 inet static
    address <ServerIP>/26
    gateway <Gateway>
    bridge_ports enp0s31f6
    bridge_stp off
    bridge_fd 0
    up route add -net <ServerIP> netmask 255.255.255.192 gw <Gateway> dev enp0s31f6

iface vmbr0 inet6 static
    address <ServerIP>/64
    gateway <Gateway>
```

- Wichtig: Interface (`enp0s31f6`) muss dem Server entsprechen (z.B. `eno1`).
- `<ServerIP>` und `<Gateway>` durch die tatsächlichen IP-Adressen ersetzen.

#### 4. DHCP für SDN einrichten

```bash
apt -y install dnsmasq
systemctl disable --now dnsmasq
```

#### 5. Proxmox SDN in GUI konfigurieren

- GUI öffnen: `https://<ServerIP>:8006` (echte Server-IP einsetzen!)
- SDN aktivieren:
  - `Rechenzentrum > SDN > Zonen` → Simple Zone hinzufügen
    - Erweiterte Optionen aktivieren, DHCP aktivieren
  - `Rechenzentrum > SDN > VNets`
    - `vnet0` erstellen und Zone auswählen
    - Subnetz hinzufügen:
      - Subnetz: `10.0.0.0/24`
      - Gateway: `10.0.0.1`
      - SNAT: aktivieren
      - DHCP-Bereich: `10.0.0.100` - `10.0.0.200`
- Änderungen anwenden (`Rechenzentrum > SDN > Zonen` → Anwenden)

#### 6. Firewall konfigurieren

- Unter `Rechenzentrum > Firewall` Regeln erstellen:

| Regel | Richtung | Aktion | Protokoll | Port | Interface | Makro |
|-------|----------|--------|-----------|------|-----------|-------|
| SSH Zugang | IN | ACCEPT | - | - | - | SSH |
| Web UI Zugang | IN | ACCEPT | TCP | 8006 | - | - |
| DHCP | IN | ACCEPT | - | - | vnet0 | DHCPfwd |
| DNS | IN | ACCEPT | - | - | vnet0 | DNS |

- Firewall aktivieren:

```bash
pve-firewall start
```

#### 7. Container erstellen

- Bei Container-Erstellung Netzwerk auf `vnet0` setzen
- DHCP für automatische IP-Vergabe auswählen

