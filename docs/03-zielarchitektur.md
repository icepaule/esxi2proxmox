# 03 — Zielarchitektur

## Übersicht — Logische Sicht

```mermaid
flowchart TB
    Internet[(Internet)]
    Cable[Kabelmodem / CPE]
    FB[FritzBox<br/>192.168.178.1]

    Internet --> Cable
    Cable --> FB

    subgraph "L3-Edge"
        OPN[OPNsense VM<br/>WAN: 192.168.178.x<br/>LAN: 10.10.0.2]
    end

    subgraph "L2-Backbone (MikroTik CRS305, unangetastet)"
        CRS["CRS305 Flat Bridge<br/>10G SFP+ × 4<br/>Jumbo MTU 10218"]
    end

    subgraph "Hypervisor: UM790 Pro / Proxmox VE"
        ProxHost["Proxmox-Host<br/>2× I226 2,5G + 1× QNAP 10G via USB4"]
        OPN
        BDCv[BDC2025 — AD-DC<br/>8 GB / 200 GB]
        CAPE[CAPEv2 — Nested-Virt<br/>16 GB / 500 GB]
    end

    subgraph "K3s-Cluster (3 Nodes, lt. Memory)"
        K3s["splunk · ecoDMS · OS-Watch<br/>MWcrawler · Caldera · CheckMK<br/>Elastic (Kibana-OSINT neu)"]
    end

    subgraph "Storage"
        SyNAS["Synology<br/>NFS 13,9 TB<br/>Archive + PVC + Backups"]
    end

    subgraph "Sonstige Hosts (unangetastet)"
        NUC[NUC-HA<br/>Home Assistant Supervised]
        Cisco[Cisco WiFi-AZ]
        UCK[UniFi UCK+]
    end

    FB --> OPN
    OPN --> CRS
    ProxHost --> CRS
    CRS --> NUC
    CRS --> SyNAS
    CRS --> Cisco
    CRS --> UCK
    K3s -.PVC.-> SyNAS
    ProxHost -.NFS.-> SyNAS

    style OPN fill:#1a5a1a,color:#fff
    style ProxHost fill:#1a5a1a,color:#fff
    style BDCv fill:#1a5a1a,color:#fff
    style CAPE fill:#1a5a1a,color:#fff
    style K3s fill:#1a3a5a,color:#fff
    style SyNAS fill:#1a3a5a,color:#fff
    style CRS fill:#3a3a1a,color:#fff
```

## Physische Verkabelung am UM790 Pro

```mermaid
flowchart LR
    subgraph UM790[UM790 Pro Rückseite]
        eth0[2,5 GbE<br/>Intel I226<br/>WAN]
        eth1[2,5 GbE<br/>Intel I226<br/>LAN-Trunk]
        usb4[USB4 Port 1<br/>40 Gbps]
        hdmi[2× HDMI 2.1<br/>nicht genutzt]
    end

    FB[FritzBox-Port]
    SWAZ["sw-az CSS326<br/>VLAN-Trunk"]
    QNAP[QNAP QNA-T310G1S<br/>SFP+ 10G Adapter]
    CRS[CRS305 SFP+ Port]

    eth0 -- "RJ45 1G Kabel" --> FB
    eth1 -- "RJ45 2,5G Kabel" --> SWAZ
    usb4 -- "TB3-Kabel 0,7 m" --> QNAP
    QNAP -- "DAC-Kabel 1 m" --> CRS

    style eth0 fill:#5a2a1a,color:#fff
    style eth1 fill:#2a5a1a,color:#fff
    style usb4 fill:#1a3a5a,color:#fff
```

## OPNsense — Virtualisierungsdesign

```mermaid
flowchart TB
    subgraph Proxmox-Host
        direction TB
        PCI1[Intel I226 #1<br/>WAN-NIC]
        PCI2[Intel I226 #2<br/>LAN-NIC]
        USB4N[QNAP 10G<br/>USB4-NIC]
    end

    subgraph "OPNsense VM (8 GB RAM)"
        WAN[em0 — WAN<br/>DHCP von FritzBox]
        LAN[em1 — LAN-Trunk<br/>VLANs 11/12/13/666]
        STO[em2 — Storage-VLAN 333<br/>10 GbE]
    end

    PCI1 == "PCI Passthrough" ==> WAN
    PCI2 == "PCI Passthrough" ==> LAN
    USB4N == "Bridge (vmbr-stor)" ==> STO

    style WAN fill:#5a2a1a,color:#fff
    style LAN fill:#2a5a1a,color:#fff
    style STO fill:#1a3a5a,color:#fff
```

**Designentscheidungen**:

| Aspekt | Entscheidung | Begründung |
|---|---|---|
| WAN-NIC | PCI-Passthrough Intel I226 | Maximale Trennung Host ↔ Gast, sicherer |
| LAN-NIC | PCI-Passthrough Intel I226 | OPNsense bekommt rohen VLAN-Trunk |
| 10G NIC | über Proxmox-Bridge (nicht Passthrough) | USB4-Passthrough mit FreeBSD nicht zuverlässig |
| RAM | 8 GB (statt 16 GB der alten UTM) | Real-RAM-Bedarf liegt bei ~1,5 GB |
| Disk | 32 GB virtio | OPNsense braucht keine 100 GB |
| CPU | 4 vCPU, NUMA off | Reserve für IDS/IPS |

## Storage-Strategie

```mermaid
flowchart TB
    subgraph UM790[UM790 Pro]
        NVMe1[NVMe 1 — 1 TB<br/>WD Black SN770]
        NVMe2[NVMe 2 — 1 TB<br/>WD Black SN770]
    end

    subgraph ZFS["ZFS Mirror (rpool/data)"]
        Proxmox[Proxmox-OS<br/>~40 GB]
        OPNvm[OPNsense VM<br/>32 GB]
        BDCvm[BDC2025 VM<br/>200 GB]
        CAPEvm[CAPEv2 VM<br/>500 GB]
        Reserve[Reserve<br/>~200 GB]
    end

    NVMe1 -- mirror --> ZFS
    NVMe2 -- mirror --> ZFS

    subgraph SynologyNFS["Synology NFS (über 10G SFP+ Trunk)"]
        Backup[Proxmox Backup Server-Ziel<br/>VM-Snapshots]
        Archiv[Kibana-OSINT-Archiv<br/>8 TB read-only]
        K3sPV[K3s NFS-PVCs<br/>ecoDMS, OS-Watch, MWcrawler]
    end

    ProxBackup[/Proxmox Backup-Job/] --> Backup

    style NVMe1 fill:#1a5a1a,color:#fff
    style NVMe2 fill:#1a5a1a,color:#fff
    style ZFS fill:#2a5a2a,color:#fff
    style SynologyNFS fill:#1a3a5a,color:#fff
```

**Storage-Tiers**:

| Tier | Was | Worauf |
|---|---|---|
| **Hot** (NVMe local) | OS, Boot-Disks, häufig genutzte VMs | ZFS-Mirror auf UM790 |
| **Warm** (10G NFS) | VM-Disks die kein NVMe brauchen, Snapshots | Synology Volume2 |
| **Cold** (NFS read-only) | Archive, alte OSINT-Daten | Synology Archiv-Share |

## Netz-Integration ins bestehende Setup

Das bestehende **L2-Backbone (MikroTik CRS305)** bleibt **unverändert**:

- Bridge bleibt flat (`vlan-filtering=no`)
- L2 MTU 10218 (Jumbo Frames für Storage-VLAN 333)
- Bestehende Verkabelung der anderen Switches/Hosts unangetastet

**Neuer Anschluss CRS305**:

| CRS305 Port | Vorher | Nachher |
|---|---|---|
| sfp1 | → sw-keller P25 | unverändert |
| sfp2 | unused | **→ UM790 (QNAP 10G SFP+, VLAN 333 für Storage)** |
| sfp3 | → Cisco WiFi-WZ | unverändert |
| sfp4 | → sw-az P24 | unverändert |

Der **UM790-LAN-Port (I226 2,5 G)** geht in den bestehenden Trunk-Port
am sw-az / sw-keller — VLAN-Tagging übernimmt OPNsense.

## VLAN-Schema (unverändert)

Das bestehende VLAN-Schema bleibt 1:1 erhalten — nur das Default-Gateway
pro VLAN wandert von der Sophos zur OPNsense:

| VLAN | Zweck | Subnet | Gateway (neu) |
|---|---|---|---|
| 1 | Mgmt | 10.10.0.0/24 | 10.10.0.2 (OPNsense) |
| 11 | Mgmt extended | 10.10.10.0/24 | 10.10.0.2 |
| 12 | IoT | 10.10.12.0/24 | bleibt UniFi USG für DHCP |
| 13 | Bad-Zone | 10.10.13.0/24 | 10.10.13.1 (OPNsense) |
| 333 | Storage | 10.10.33.0/24 | OPNsense (nur Routing, kein DHCP) |
| 666 | Internet/FB | 192.168.178.0/24 | FritzBox |

## VPN-Endpunkt zu Hetzner

Bestehender IPsec/SSL-Tunnel zur Hetzner-Sophos wird **als WireGuard auf OPNsense** neu aufgebaut:

```mermaid
flowchart LR
    Heim[Heimnetz<br/>OPNsense WG-Peer]
    FB[FritzBox<br/>Port-Forward UDP/51820]
    Internet[(Internet)]
    HetznerFW[Hetzner Sophos<br/>IPsec-Endpoint]
    HetznerLAN[Hetzner VLAN<br/>Server-Cloud]

    Heim --> FB
    FB --> Internet
    Internet --> HetznerFW
    HetznerFW --> HetznerLAN

    style Heim fill:#1a5a1a,color:#fff
    style HetznerFW fill:#3a3a1a,color:#fff
```

**Optionen**:

| Variante | Vorteil | Nachteil |
|---|---|---|
| WireGuard end-to-end (beide Seiten WG) | schneller, einfacher, modern | erfordert Konfig auch auf Hetzner-Seite |
| IPsec wie bisher | Sophos-Seite unverändert | mehr Konfig-Aufwand auf OPNsense |
| Hybrid (WG für neue Routen, IPsec übergangsweise) | sanfter Cutover | doppelte Verwaltung |

→ **Entscheidung**: WireGuard end-to-end, Hetzner-Sophos wird parallel
umgestellt (separates Projekt).

## Hardware-Investitionen ins bestehende Setup integrieren

| Bestehende Komponente | Wird verändert? | Wie integriert? |
|---|---|---|
| HP DL380 Gen9 | ❌ wird abgeschaltet | nach erfolgreichem Cutover dauerhaft aus |
| Sophos UTM SG | ❌ wird durch OPNsense ersetzt | Config-Export als Referenz |
| MikroTik CRS305 | ✅ bleibt unverändert | nur ein zusätzlicher SFP+-Port belegt |
| sw-az / sw-keller (CSS326) | ✅ bleibt unverändert | Trunk-Port am sw-az nimmt UM790 |
| Synology | ✅ bleibt unverändert | zusätzliche NFS-Shares für Proxmox + K3s-PVCs |
| NUC-HA | ✅ bleibt unverändert | übernimmt K3s-Master-Rolle wie bisher |
| K3s-Cluster (3 Nodes) | ✅ erweitert | 8 neue Workloads via Helm-Charts |
| Cisco WiFi-AZ / WZ | ✅ bleibt unverändert | nur Default-Gateway-Mac wird auf OPN umgestellt |
| FritzBox | ✅ bleibt unverändert | Port-Forward für WG-Endpunkt |
| UniFi UCK+ | ✅ bleibt unverändert | weiter DHCP für VLAN 12 (Bad:IoT) |

→ **Keine Eingriffe ins bestehende L2-Backbone**, **kein Re-Patching**
der anderen Server. Migration ist additiv: UM790 wird parallel
hochgezogen, alte Sophos bleibt bis zum Cutover live, dann IP-Tausch
und Sophos wird heruntergefahren.

## Weiter

→ **[04-migrationspfad.md](04-migrationspfad.md)** — Phasen-Plan mit
VM-für-VM Migration.
