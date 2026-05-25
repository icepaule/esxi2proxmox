# 06 — Integration ins bestehende Setup

## Architektur-Vergleich VORHER ↔ NACHHER

```mermaid
flowchart TB
    subgraph V["VORHER (~200 W Server-Sockel)"]
        direction LR
        VF[FritzBox]
        VS["Sophos UTM<br/>(VM auf DL380)"]
        VESX["HP DL380 Gen9<br/>ESXi 7.0.3<br/>12 VMs"]
        VCRS[CRS305 L2-Backbone]
        VK3s[K3s-Cluster 3 Nodes]
        VSyn[Synology 14 TB]
        VNUC[NUC-HA]

        VF --> VS
        VS --> VCRS
        VCRS --> VESX
        VCRS --> VK3s
        VCRS --> VSyn
        VCRS --> VNUC
        VESX -.NFS.-> VSyn
    end

    subgraph N["NACHHER (~25 W Mini-PC)"]
        direction LR
        NF[FritzBox]
        NUM[UM790 Pro<br/>Proxmox VE]
        NOPN[OPNsense VM]
        NBDC[BDC2025<br/>AD-DC]
        NCAPE[CAPEv2<br/>Nested-Virt]
        NCRS[CRS305 L2-Backbone<br/>unverändert]
        NK3s[K3s-Cluster 3 Nodes<br/>+8 neue Workloads]
        NSyn[Synology 14 TB<br/>+ Archiv]
        NNUC[NUC-HA<br/>unverändert]

        NF --> NUM
        NUM --> NOPN
        NUM --> NBDC
        NUM --> NCAPE
        NOPN --> NCRS
        NUM == "10G SFP+ via QNAP" ==> NCRS
        NCRS --> NK3s
        NCRS --> NSyn
        NCRS --> NNUC
        NK3s -.PVC.-> NSyn
    end

    V == Migration ==> N

    style VESX fill:#5a1a1a,color:#fff
    style VS fill:#a02020,color:#fff
    style NUM fill:#1a5a1a,color:#fff
    style NOPN fill:#1a5a1a,color:#fff
    style NBDC fill:#1a5a1a,color:#fff
    style NCAPE fill:#1a5a1a,color:#fff
```

## Komponenten-Übersicht: Was bleibt, was kommt, was geht

### ✅ Unverändert (Stabilitätsanker)

```mermaid
flowchart LR
    A[FritzBox<br/>192.168.178.1]
    B[CRS305 Backbone<br/>192.168.178.9<br/>Flat L2-Bridge]
    C[sw-az CSS326<br/>VLAN-Trunk]
    D[sw-keller CSS326<br/>VLAN-Trunk]
    E[Cisco WiFi-AZ<br/>VLAN 666 Distribution]
    F[UniFi UCK+ + USG<br/>DHCP für VLAN 12]
    G[NUC-HA<br/>Home Assistant + K3s-Master]
    H[Synology<br/>NFS + ABB-Backups]
    I[Tibber Pulse<br/>Strommessung]
    J[AdGuard Pi<br/>DNS-Filter]
    K[MAX7219 LED-Matrizen<br/>IoT-Anzeigen]

    style A fill:#1a3a5a,color:#fff
    style B fill:#1a3a5a,color:#fff
    style C fill:#1a3a5a,color:#fff
    style D fill:#1a3a5a,color:#fff
    style E fill:#1a3a5a,color:#fff
    style F fill:#1a3a5a,color:#fff
    style G fill:#1a3a5a,color:#fff
    style H fill:#1a3a5a,color:#fff
    style I fill:#1a3a5a,color:#fff
    style J fill:#1a3a5a,color:#fff
    style K fill:#1a3a5a,color:#fff
```

Diese Komponenten werden **nicht angefasst**. Das ist die größte Stärke
der Migration: das produktive Hausnetz bleibt während der gesamten
Migration **online und funktional**.

### 🆕 Neu hinzugefügt

```mermaid
flowchart LR
    A[Minisforum UM790 Pro<br/>Proxmox VE 8.x]
    B[OPNsense VM<br/>ersetzt Sophos UTM]
    C[QNAP QNA-T310G1S<br/>10G SFP+ Adapter]
    D[Proxmox Backup Server<br/>VM oder native]
    E[8 neue K3s-Workloads<br/>via Helm]
    F[Synology Archiv-Volume<br/>Kibana-OSINT-Daten]

    style A fill:#1a5a1a,color:#fff
    style B fill:#1a5a1a,color:#fff
    style C fill:#1a5a1a,color:#fff
    style D fill:#1a5a1a,color:#fff
    style E fill:#1a5a1a,color:#fff
    style F fill:#1a5a1a,color:#fff
```

### ❌ Entfernt / abgeschaltet

```mermaid
flowchart LR
    A[HP DL380 Gen9<br/>~200 W, 2 U, laut]
    B[Sophos UTM SG VM<br/>Lizenz abgelaufen]
    C[tembo-VM<br/>Restore-Leiche]
    D[MailCOW-VM<br/>seit Monaten aus]

    style A fill:#3a3a3a,color:#fff
    style B fill:#3a3a3a,color:#fff
    style C fill:#3a3a3a,color:#fff
    style D fill:#3a3a3a,color:#fff
```

## Integration in die Netz-Layer

### Layer 2 — Switching (CRS305 Backbone)

Der CRS305 als **Flat L2-Bridge** mit Jumbo Frames (MTU 10218) bleibt
exakt wie er ist. Nur **ein zusätzlicher SFP+ Port** wird belegt:

| CRS305 Port | Vorher | Nachher | Speed |
|---|---|---|---|
| sfp1 | sw-keller P25 | sw-keller P25 | 10G SFP+ |
| **sfp2** | unused | **UM790 (QNAP SFP+ Adapter)** | 10G SFP+ |
| sfp3 | Cisco WiFi-WZ | Cisco WiFi-WZ | 10G SFP+ |
| sfp4 | sw-az P24 | sw-az P24 | 10G SFP+ |
| ether1 | unused | unused | 1G RJ45 |

Damit hat der UM790 einen **dedizierten 10G-Pfad** zum Storage-Backbone
und zur Synology — perfekt für VM-Migration und NFS-Storage-Performance.

### Layer 3 — Routing & VLAN-Gateways

Das bestehende VLAN-Schema bleibt **bit-identisch** erhalten — nur die
**Gateway-MAC** wechselt vom Sophos zur OPNsense:

```mermaid
flowchart TB
    OPN["OPNsense<br/>(neuer Router)"]

    V1[VLAN 1: Mgmt<br/>10.10.0.0/24]
    V11[VLAN 11: Mgmt-Extended<br/>10.10.10.0/24]
    V12["VLAN 12: IoT<br/>10.10.12.0/24<br/>(DHCP von UniFi USG)"]
    V13[VLAN 13: Bad-Zone<br/>10.10.13.0/24]
    V333[VLAN 333: Storage<br/>10.10.33.0/24]
    V666[VLAN 666: Internet<br/>192.168.178.0/24]

    OPN -- ".2" --> V1
    OPN -- ".100/24" --> V11
    OPN -. "Routing, kein DHCP" .-> V12
    OPN -- ".1" --> V13
    OPN -- "Trunk" --> V333
    OPN -- "WAN-Side" --> V666

    style OPN fill:#1a5a1a,color:#fff
    style V12 fill:#3a3a1a,color:#fff
```

Beim **Cutover** wird **nur die Gateway-IP 10.10.0.2** vom Sophos zur
OPNsense umgezogen. Alle anderen Hosts behalten ihre statischen
Konfigurationen — sie merken den Wechsel nicht.

### Layer 7 — Dienste & Applikationen

Wie integrieren sich die Services?

```mermaid
flowchart TB
    subgraph "Endnutzer-Services"
        HA[Home Assistant]
        AdG[AdGuard DNS]
        Cisco[Cisco WiFi]
    end

    subgraph "Migration-Targets"
        OPN[OPNsense]
        Prox[Proxmox UM790]
        K3s[K3s 8 Workloads]
    end

    subgraph "Storage & Backup"
        SyNAS[Synology NFS]
        PBS[Proxmox Backup Server]
        ABB[Active Backup for Business<br/>auf Synology]
    end

    HA -- "DHCP/DNS lookup" --> OPN
    HA -- "Statistics" --> AdG
    AdG -- "Upstream DNS" --> OPN
    Cisco -- "Routing" --> OPN

    Prox -- "NFS" --> SyNAS
    Prox -- "Daily Backup" --> PBS
    PBS -- "Backup-Repo" --> SyNAS
    K3s -- "PVC" --> SyNAS
    SyNAS -- "VM-Schnappschuss" --> ABB
```

| Service | Heutiges Verhalten | Nach Migration |
|---|---|---|
| Home Assistant | DHCP/DNS von Sophos | DHCP/DNS von OPNsense (transparent) |
| AdGuard Home | Upstream DNS = Sophos | Upstream DNS = OPNsense |
| Cisco/UniFi WiFi | Routing via Sophos | Routing via OPNsense |
| Hetzner-VPN | IPsec via Sophos | WireGuard via OPNsense (Hetzner-Seite parallel umstellen) |
| K3s-Workloads | NUC-HA Master + Worker | unverändert + 8 neue Workloads via Helm |
| Synology NFS | bestehende Exports | + Proxmox-Datastore + K3s-PVCs |
| Tibber Pulse | direkt → HA | unverändert |
| MAX7219 IoT | MQTT zu NUC-HA | unverändert |

## Integration des K3s-Clusters

Der bestehende K3s-Cluster (3 Nodes lt. Memory) übernimmt die
**leichten, containerisierbaren Workloads** vom DL380:

```mermaid
flowchart TB
    subgraph K3s["K3s-Cluster (3 Nodes)"]
        N1[Node ki01]
        N2[Node kibana-osint]
        N3[Node Synology]
    end

    subgraph Bisher["Bisherige Workloads"]
        Argo[ArgoCD GitOps]
        Mon[Monitoring]
        Misc["+ 9 Docker-Compose Stacks<br/>(migriert lt. Memory)"]
    end

    subgraph Neu["NEUE Workloads (von DL380)"]
        Cald[Caldera<br/>Helm-Chart MITRE]
        CMK[CheckMK<br/>Helm-Chart]
        ESPL[splunk<br/>splunk-enterprise]
        ECO[ecoDMS<br/>Container]
        ELA[Elasticsearch + Kibana<br/>klein/frisch]
        OSW[OS-Watch<br/>Custom Container]
        MWC[MWcrawler<br/>Custom Container]
    end

    K3s --> Bisher
    K3s --> Neu

    style K3s fill:#1a3a5a,color:#fff
    style Bisher fill:#2a4a6a,color:#fff
    style Neu fill:#1a5a1a,color:#fff
```

**Ressourcen-Verteilung pro Node** (geschätzt nach Migration):

```mermaid
xychart-beta
    title "K3s-Node RAM-Auslastung nach Migration (geschätzt)"
    x-axis ["ki01", "kibana-osint", "Synology"]
    y-axis "GB" 0 --> 32
    bar [12, 14, 8]
```

Empfehlung: vor der Migration **Resource-Capacity pro K3s-Node prüfen**:

```bash
kubectl describe node | grep -E "Allocatable|Allocated resources|memory"
```

Wenn die K3s-Nodes RAM-knapp sind: zuerst **dort RAM aufrüsten**, bevor
die Workloads dorthin verschoben werden.

## Integration der Storage-Schicht

```mermaid
flowchart TB
    subgraph UM790_Storage["UM790 Pro NVMe (ZFS-Mirror, 2× 1 TB)"]
        Proxmox[Proxmox-OS<br/>~40 GB]
        OPNvm[OPNsense<br/>32 GB]
        BDCvm[BDC2025 AD-DC<br/>200 GB]
        CAPEvm[CAPEv2 Sandbox<br/>500 GB]
    end

    subgraph CRS305["CRS305 10G SFP+ Trunk"]
        TrunkVLAN333["VLAN 333<br/>10.10.33.0/24<br/>Jumbo MTU 9000"]
    end

    subgraph Synology_Storage["Synology (NFS via 10G)"]
        SyVol2["/volume2/proxmox-store<br/>VM-Files für nicht-kritische VMs"]
        SyArchive["/volume2/archive/kibana_osint<br/>8 TB read-only Archiv"]
        SyK3s["/volume2/k3s-pvc<br/>PVCs für ecoDMS, OS-Watch, MWcrawler"]
        SyABB["/volume1/@ActiveBackup<br/>Bestehende ABB-Backups"]
        SyPBS["/volume2/pbs-repo<br/>Proxmox Backup Server Repo"]
    end

    UM790_Storage --> TrunkVLAN333
    TrunkVLAN333 --> Synology_Storage

    style UM790_Storage fill:#1a5a1a,color:#fff
    style CRS305 fill:#3a3a1a,color:#fff
    style Synology_Storage fill:#1a3a5a,color:#fff
```

## Hetzner-VPN-Integration

Der bisherige **IPsec-Tunnel** zur Hetzner-Sophos wird auf **WireGuard**
umgestellt:

```mermaid
sequenceDiagram
    participant H as Heim-OPNsense
    participant FB as FritzBox (NAT)
    participant Net as Internet
    participant HZ as Hetzner-Sophos
    participant HZV as Hetzner-VPC

    Note over H,HZV: Tunnel-Aufbau (initial)
    H->>FB: WireGuard Handshake (UDP 51820)
    FB->>Net: Port-Forward 51820 → OPNsense
    Net->>HZ: Hetzner-IP empfängt
    HZ-->>H: WireGuard-Response
    Note over H,HZ: Encrypted Tunnel etabliert

    Note over H,HZV: Datenverkehr
    H->>HZ: Verschlüsselter Traffic
    HZ->>HZV: ins Hetzner-VPC
    HZV-->>HZ: Response
    HZ-->>H: zurück durch Tunnel
```

**Vorteile WireGuard gegenüber IPsec**:
- Schnellere Performance auf AMD-Zen4 mit ChaCha20-Poly1305
- Einfachere Konfiguration (kein IKEv2-Phase-1/2 Theater)
- Modernere Crypto, kleiner Footprint
- Roaming-fähig (Re-Keying nahtlos)

## Sicherheits-Integration

```mermaid
flowchart LR
    subgraph DefInDepth[Defense in Depth]
        L1[Internet]
        L2[FritzBox<br/>NAT + Stateful FW]
        L3[OPNsense<br/>L3 Firewall + IDS]
        L4[VLAN-Segmentierung]
        L5[Endgerät-Sicherheit]
    end

    L1 --> L2 --> L3 --> L4 --> L5

    style L2 fill:#1a3a5a,color:#fff
    style L3 fill:#1a5a1a,color:#fff
    style L4 fill:#1a5a1a,color:#fff
```

| Sophos-Feature | OPNsense-Equivalent |
|---|---|
| Firewall (Stateful) | OPNsense Filter (PF-basiert) |
| Web-Filter | Web Proxy + Squid + SquidGuard |
| IPS | Suricata Plugin (LISTS Emerging Threats / ETOpen) |
| Anti-Spam | extern (MailCOW falls aktiv) oder Hetzner-Filter |
| WAF | externer Reverse-Proxy (Caddy / nginx-proxy-manager) |
| VPN IPsec/SSL | WireGuard (schneller) + OpenVPN (Roadwarrior) |
| HA / Cluster | OPNsense CARP (optional später) |

## Backup-Strategie

```mermaid
flowchart TB
    subgraph Quellen["Backup-Quellen"]
        ProxVMs[Proxmox-VMs]
        K3sV[K3s-Volumes]
        Syn[Synology direkt]
    end

    subgraph Targets["Backup-Targets"]
        PBS["Proxmox Backup Server<br/>(VM oder Bare)"]
        ABB["Active Backup for Business<br/>(auf Synology)"]
        Cold["Cold-Storage<br/>USB / extern"]
    end

    ProxVMs -- "Daily incremental" --> PBS
    K3sV -- "velero / restic" --> Syn
    Syn -- "ABB Snapshots" --> ABB
    PBS -- "Wöchentlicher Sync" --> Cold

    style PBS fill:#1a5a1a,color:#fff
    style ABB fill:#1a3a5a,color:#fff
    style Cold fill:#3a3a3a,color:#fff
```

## Tagesablauf nach erfolgreicher Migration

| Aktion | Bisher | Nachher |
|---|---|---|
| Morgens HA-Dashboard prüfen | unverändert | unverändert |
| Firewall-Log einsehen | Sophos Web-Admin | OPNsense Live-View |
| Neue VM erstellen | ESXi vSphere Client | Proxmox Web-UI (browserbasiert!) |
| K8s-Workload deployen | ArgoCD | unverändert |
| Stromverbrauch tracken | Tibber Pulse / HA | Tibber Pulse / HA (jetzt 200 W weniger) |
| Backup-Status | manuell auf Synology | Proxmox-Backup-Dashboard |

## Lessons Learned aus der Architektur

1. **Trennung Compute vs. Storage**: Storage bleibt zentral auf der
   Synology — Compute wird hochskaliert oder reduziert je nach Bedarf.
   Das macht den UM790 austauschbar/erweiterbar.
2. **L2-Backbone unangetastet**: Die CRS305 als Flat-Bridge ist über
   Jahre stabil — kein Grund das anzufassen. Migrationen erfolgen
   **additiv**.
3. **K3s als "Container-Sammelbecken"**: Workloads die nicht zwingend
   VMs sein müssen, gehören auf den K3s. Spart RAM/Disk und gewinnt
   Deklarativität.
4. **Refurbished-Hardware vom Hersteller**: Preis-Leistung wie
   AliExpress, aber mit EU-Versand, Steuer-Klarheit und Garantie. Der
   minisforumpc.eu-Refurb-Shop ist der MVP-Tipp dieses Projekts.

## Weiter

Zurück zu **[README.md](../README.md)** — Übersicht & Status.
