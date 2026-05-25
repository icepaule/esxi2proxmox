# ESXi → Proxmox Migration

> Ausmusterung eines HP ProLiant DL380 Gen9 ESXi-Hosts zugunsten eines
> stromsparenden Mini-PC mit Proxmox VE — bei gleichzeitiger Modernisierung
> der gesamten Hausnetz-Architektur und 80+ % Stromersparnis.

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-planning-orange.svg)](#)
[![Hardware](https://img.shields.io/badge/target-Minisforum%20UM790%20Pro-green.svg)](#)
[![Hypervisor](https://img.shields.io/badge/hypervisor-Proxmox%20VE-red.svg)](https://www.proxmox.com/)

---

## Kernzahlen

| Kennzahl | Vorher (DL380 Gen9) | Nachher (UM790 Pro) | Veränderung |
|---|---|---|---|
| Idle-Stromverbrauch | ~200 W | ~15 W | **−92,5 %** |
| Jahresstromkosten (30 ct/kWh) | ~525 € | ~39 € | **−486 €/Jahr** |
| Geräuschemission | 45–55 dB(A) | < 25 dB(A) | flüsterleise |
| Rack-Höheneinheiten | 2 U | 0 U (Mini-PC) | ~25 cm Platz frei |
| CPU-Effizienz (PassMark/W) | ~50 | ~720 | **14× besser** |
| Investition (32 GB DDR5 Crucial-Retail, **empfohlen**) | — | **~907 €** ⭐ | Amortisation **~21 Monate** |
| Investition (64 GB DDR5 Crucial-Retail, optional später) | — | ~1.327 € | Amortisation ~31 Monate |

> ⚠️ Lessons Learned aus eBay-Schnäppchen-Fallen siehe **[docs/07-lessons-learned.md](docs/07-lessons-learned.md)**

> 📊 Vollständige Wirtschaftlichkeitsanalyse mit Live-Daten aus
> Home Assistant: **[docs/05-einsparungen.md](docs/05-einsparungen.md)**

## Was hier dokumentiert ist

1. **[Herleitung & Ausgangslage](docs/01-herleitung.md)** — Warum
   überhaupt? Bestandsanalyse des bestehenden Setups, Schmerzpunkte,
   Trigger-Event (abgelaufene Sophos-UTM-Lizenz).
2. **[Hardware-Auswahl](docs/02-hardware-auswahl.md)** — Vergleichsmatrix
   aller Optionen von N100-Sparlösung bis Ryzen-9-Workstation,
   Preisrecherche, Forum-Korrelation, klare Empfehlung.
3. **[Zielarchitektur](docs/03-zielarchitektur.md)** — UM790 Pro mit
   Proxmox VE + OPNsense-VM, K3s-Integration, Storage-Architektur,
   Netz-Topologie inkl. 10 GbE-SFP+ Trunk.
4. **[Migrationspfad](docs/04-migrationspfad.md)** — VM-für-VM Migrations-
   matrix der 12 ESXi-Workloads, Phasenplan mit Risikostaffelung,
   Rollback-Strategie.
5. **[Einsparungen & PV-Synergie](docs/05-einsparungen.md)** — Strom-
   kosten-Berechnung mit echten 30-Tage-Daten aus HA, PV-Eigenverbrauchs-
   quote, ROI, CO₂-Bilanz.
6. **[Integration ins bestehende Setup](docs/06-integration.md)** —
   wie sich die neue Hardware in LAN/Server/Dienste/Applikationen
   einfügt ohne den produktiven Betrieb zu stören.

## Quick-Architektur

```mermaid
flowchart LR
    subgraph "VORHER (DL380 Gen9, ~200 W)"
        ESXi[ESXi 7.0.3<br/>12 VMs gleichzeitig]
        Sophos[Sophos UTM<br/>16 GB RAM]
        ESXi --> Sophos
    end
    subgraph "NACHHER (UM790 Pro, ~15 W)"
        Proxmox[Proxmox VE]
        OPNsense[OPNsense<br/>8 GB RAM]
        ProxVMs[Win-DC + CAPEv2]
        K3s[K3s-Cluster<br/>8 Workloads]
        Synology[Synology<br/>Archiv & PVC]
        Proxmox --> OPNsense
        Proxmox --> ProxVMs
        Proxmox -.NFS.-> Synology
        K3s -.PVC.-> Synology
    end
    VORHER -- Migration --> NACHHER

    style ESXi fill:#5a1a1a,color:#fff
    style Sophos fill:#5a1a1a,color:#fff
    style Proxmox fill:#1a5a1a,color:#fff
    style OPNsense fill:#1a5a1a,color:#fff
    style ProxVMs fill:#1a5a1a,color:#fff
    style K3s fill:#1a5a1a,color:#fff
    style Synology fill:#1a3a5a,color:#fff
```

## Status

- [x] Analyse Ausgangslage (Stromverbrauch, VM-Inventar)
- [x] Hardware-Recherche (Klasse A/B/C, Schnäppchen-Suche)
- [x] Migrationsmatrix mit strategischen Entscheidungen
- [ ] Hardware-Bestellung
- [ ] Proxmox-Installation
- [ ] OPNsense-Aufbau parallel zur Sophos
- [ ] Cutover OPNsense
- [ ] Phasenweise VM-Migration
- [ ] DL380-Abschaltung & Stromzähler-Verifikation

## Lizenz

MIT — siehe [LICENSE](LICENSE).

Dieses Repository dient als persönliche Migrationsdokumentation und kann
als Vorlage für ähnliche Homelab-Konsolidierungen dienen.
