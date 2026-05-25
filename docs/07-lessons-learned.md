# 07 — Lessons Learned: eBay-RAM-Schnäppchen

## Vorgeschichte

Im Mai 2026 sind während der Hardware-Recherche für dieses Projekt zwei
typische eBay-Fallstricke aufgetaucht — beide lehrreich für künftige
Käufe.

## Fall 1 — Samsung "OEM 64 GB Kit 199 €" (Fake)

**Inserat**: Samsung M425R4GA3BB0-CWM 2× 32 GB DDR5-5600 SODIMM,
angeblich mit 10 Jahren Garantie, 1 Monat Widerrufsrecht, DHL-Versand
aus DE, 99 % positive Verkäufer-Bewertung.

**Marktrealität**: Beim Kontakt mit dem Verkäufer wurde der Preis auf
650 € korrigiert — die 199 € war ein **Lockangebot**.

**Warnsignal**, das ich übersehen habe:
- 199 € sind **~66 % unter Marktpreis** (Crucial-Retail ~600 €)
- Bei DDR5-Preisniveau 2026 ist **alles unter 50 % vom Marktpreis**
  praktisch immer ein Fake, Defekt oder Scam

## Fall 2 — Crucial "32 GB DDR5 für 188 €" (defekt)

**Inserat-Kleingedrucktes**:
> "Product condition: RAM memory DAMAGED - for repair or for parts.
> Memtest errors. Original or replacement packaging."

**Was passiert wäre beim Kauf**:
1. POST funktioniert evtl. trotzdem
2. Proxmox startet
3. ZFS schreibt korrupte Daten wegen RAM-Bit-Flips
4. Silent Data Corruption → VMs werden zerstört
5. Wochenlang unbemerkt, bis nichts mehr bootet

## Sicherheits-Checkliste für eBay-RAM-Käufe

```mermaid
flowchart TD
    A[eBay-Angebot ansehen] --> B{Preis vs Markt}
    B -- "< 50%" --> X[🔴 STOP - meist Fake/Defekt]
    B -- "50-70%" --> C{Zustand-Feld}
    B -- "70-90%" --> C
    B -- "≥ 90%" --> C
    C -- "DAMAGED / for parts / untested" --> X
    C -- "Gebraucht / Refurbished / Neu" --> D{MPN auf Hersteller-Site}
    D -- "matched & non-ECC verified" --> E{Verkäufer-Bewertungen}
    D -- "MPN unauffindbar" --> X
    E -- "< 98% / wenig Verkäufe" --> X
    E -- "≥ 99% / viele Verkäufe" --> F{PayPal angeboten?}
    F -- "nein nur Vorkasse" --> X
    F -- "ja PayPal verfügbar" --> G[✅ Sicherer Kauf möglich]

    style X fill:#a02020,color:#fff
    style G fill:#1a5a1a,color:#fff
```

### Verdacht-Indikatoren in Reihenfolge der Wichtigkeit

| Indikator | Aktion |
|---|---|
| 🔴 Zustand-Feld enthält "damaged", "for parts", "untested", "as is", "fault" | **Nicht kaufen** |
| 🔴 Preis < 50 % vom Marktpreis | **Sehr skeptisch** |
| 🔴 Versand aus China, 14-30 Tage Lieferzeit | Vermutlich Fake/Re-Mark |
| 🟡 Stock-Foto statt Verkäufer-Foto | Bei Massenartikeln OK, bei Schnäppchen nicht |
| 🟡 MPN auf Hersteller-Website nicht auffindbar | MPN verifizieren! |
| 🟡 Verkäufer "nur Vorkasse" | PayPal verlangen |
| 🟢 Marktpreis ± 10 %, originale Bilder, 99 % Bewertung, PayPal | OK |

## Realistischer Marktpreis 2026 (Stand Mai)

| Modul | Preis-Range DE |
|---|---|
| Crucial CT2K16G56C46S5 (32 GB Kit DDR5-5600) | 150–200 € |
| Crucial CT2K32G56C46S5 (64 GB Kit DDR5-5600) | 550–650 € |
| Kingston FURY Impact 32 GB Kit | 160–200 € |
| Kingston FURY Impact 64 GB Kit | 580–680 € |
| Samsung OEM single 32 GB Module | 100–140 € (legitim bei seriösen Refurb-Händlern) |

## Goldene Regel

> Bei Hardware, die 24/7 mein Hausnetz und meine Daten managen soll,
> ist die **Garantie wichtiger als die letzten 100 € Ersparnis**.
> Lieber bei Mindfactory/Amazon DE zum Marktpreis kaufen — mit
> 2-Jahres-Gewährleistung, Rückgaberecht und nachvollziehbarer
> Lieferkette.

## Was sich für dieses Projekt ändert

Bestellplan jetzt **konservativ**:

| Position | Was | Preis | Quelle |
|---|---|---|---|
| RAM | Crucial 32 GB Kit (CT2K16G56C46S5) | ~180 € | Mindfactory/Amazon DE |
| RAM-Upgrade später | Crucial 64 GB Kit (CT2K32G56C46S5) | wenn Preise sinken (Ende 2026 erwartet) | Geizhals-Alarm |

**Gesamtinvestition**: ~907 € (Variante B, 32 GB)
**Amortisation**: ~21 Monate

Die ursprünglich erhoffte 64-GB-Vollausstattung kommt
**später per Upgrade**, wenn der DDR5-Markt sich entspannt hat.

## Fall 3 — Wiederholter Preis-Schätzfehler beim DDR5-RAM

**Was passiert ist** (Mai 2026, drei Iterationen):

| Iteration | Behauptung | Realität | Quelle der Realität |
|---|---|---|---|
| 1 | 64 GB DDR5 Kit ~150 € | ~600 € (Tom's Hardware: AI-Driven Shortage) | User-Korrektur |
| 2 | Samsung 64 GB "OEM Schnäppchen" 199 € | Lockangebot, real 650 € | User-Korrektur |
| 3 | 32 GB Crucial Kit ~180 € | **279,95 €** (Geizhals verifiziert) | User-Korrektur + Geizhals |

**Wurzelursache**: Ich habe Preise aus meinem Trainings-/Vorwissen geschätzt statt aktuelle Marktpreise zu verifizieren. Der DDR5-Markt hat sich Oktober 2025 → Mai 2026 **verdreifacht** (DRAM-Shortage durch HBM/AI-Wafer-Konkurrenz). Mein Vorwissen war systematisch veraltet.

**Selbstverpflichtung für die Zukunft**:

> Bei JEDER Preisangabe in diesem Repo gilt:
> - Entweder **verifizierter Live-Preis** mit Quelle und Datum
> - Oder **explizit als Schätzung gekennzeichnet** mit Hinweis "vor Bestellung live prüfen"
> - Niemals beides vermischen oder einen Schätzpreis als verifiziert darstellen

**Marktrealität DDR5 SODIMM Mai 2026** (Geizhals-verifiziert):

| Modul | Günstigster Preis | Stand |
|---|---|---|
| Crucial CT2K16G56C46S5 (32 GB Kit 2×16 GB DDR5-5600) | **279,95 €** | 25.05.2026 |
| Crucial CT2K32G56C46S5 (64 GB Kit 2×32 GB DDR5-5600) | ~600+ € (zu verifizieren) | 25.05.2026 |
| Single 32 GB DDR5-5600 SO-DIMM gebraucht | ~200-250 € | eBay-Stichproben |

**Praktische Konsequenz** für dieses Projekt:
- BOM-Preise sind jetzt klar als "verifiziert" oder "Schätzung" gekennzeichnet
- Investition Variante B (32 GB): ~1.007 € statt ursprünglich ~877 € (+15 %)
- Amortisation: ~24 Monate (statt ursprünglich ~21)
- Variante A (64 GB) wirtschaftlich nicht attraktiv bei aktuellem DDR5-Preisniveau — Upgrade abwarten bis Markt sich entspannt
