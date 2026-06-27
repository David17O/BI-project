# 🏔 Schweizer Hotellerie BI-Dashboard

**FHNW Modulprojekt – Business Intelligence 2025/26**  
Dozent: Prof. Dr. Manuel Renold | BSc Business Information Systems / Business AI  
Lehrbuch: Carlo Vercellis, *Business Intelligence – Data Mining and Optimization for Decision Making*

**Autoren:** David Ishak & Eufrat Pio Özmen

---

## Projektbeschreibung

Dieses Projekt analysiert die **Schweizer Hotellerie-Statistik (BFS HESTA)** als vollständiges, browserbasiertes BI-Dashboard. Es beantwortet vier konkrete Business-Fragen und demonstriert dabei die gesamte BI-Wertschöpfungskette nach Vercellis: **Daten → Information → Wissen → Entscheidung**.

### Business-Fragen

| # | Frage | BI-Methodik |
|---|-------|-------------|
| 1 | Welche Kantone dominieren die Schweizer Logiernächte und warum? | OLAP Drill-Down, KPI-Cards |
| 2 | Wie hat COVID-19 die Tourismus-Nachfrage verändert und wie entwickelt sich 2025? | Zeitreihenanalyse, Holt-Winters Forecast |
| 3 | Wie abhängig ist die Schweizer Hotellerie von ausländischen Gästen (Internationalisierungsrisiko)? | Dimensionsanalyse, KPI Internationalisierungsrate |
| 4 | Wie stark ist die Saisonalität und wie lässt sich die Off-Peak-Auslastung verbessern? | Saisonale Dekomposition, Heatmap, Agentic BI |

---

## Starten

```bash
# Keine Installation notwendig – einfach öffnen:
index.html   →   Doppelklick im Datei-Explorer ODER
                 in einem Browser öffnen (Chrome / Firefox / Edge)
```

Keine externen Bibliotheken – alle Visualisierungen mit nativer Canvas 2D API. Daten sind eingebettet – **funktioniert offline**.

---

## Datenquelle

| Attribut | Wert |
|----------|------|
| Quelle | Bundesamt für Statistik (BFS), Beherbergungsstatistik (HESTA) |
| URL | https://www.bfs.admin.ch/bfs/de/home/statistiken/tourismus/beherbergung/hotellerie.html |
| API | https://www.pxweb.bfs.admin.ch/api/v1/de/px-x-1003020000_102/ |
| Datensatz | px-x-1003020000_102: Logiernächte nach Jahr, Monat, Kanton, Herkunftsland |
| Abruf | Juni 2026 via BFS PxWeb REST-API (JSON) |
| Zeitraum | 2018–2024 (Jahrestotale), Forecast 2025 (Holt-Winters) |
| Kantone | GR, VS, BE, LU, TI, ZH, GE, BS (8 grösste Tourismusdestinationen) |
| Metrik | Logiernächte absolut (z.B. ZH 2019: 5'960'145), aufgeteilt in Inland / Ausland |

---

## Demonstrierte BI-Konzepte

### 1. Datenintegration & ETL
- **Extract**: Echte BFS HESTA Jahrestotale via PxWeb REST-API abgerufen (px-x-1003020000_102, Juni 2026) und als verifizierbare Konstanten eingebettet
- **Transform**: Homogenisierung der Daten, saisonale Normierung (∑ Faktoren = 12), Berechnung abgeleiteter Metriken (Inland/Ausland-Split)
- **Load**: In-Memory Faktentabelle `factOvernights[]` mit vollständiger Granularität

### 2. Dimensionsmodellierung / Star Schema
```
                    DIM_TIME
                   (year, month,
                   quarter, season)
                        │
DIM_ORIGIN ────── FACT_OVERNIGHTS ────── DIM_CANTON
(DOM, INTL,       (canton_id, year,      (id, name,
 TOTAL)            month, total,          type, intlShare)
                   domestic, intl)
```
- **Korn (Granularität)**: 1 Datensatz = 1 Kanton × 1 Monat × 1 Jahr
- **Faktentabelle**: `factOvernights` (672 Datensätze: 8 Kantone × 84 Monate)
- **3 Dimensionstabellen**: Zeit, Region/Kanton, Herkunft

### 3. OLAP-Analyse
| Operation | Implementierung |
|-----------|----------------|
| **Slice** | Jahres-Dropdown → filtert alle Charts auf ein Jahr |
| **Dice** | Kantonsgruppe + Herkunft gleichzeitig wählen |
| **Drill-Down** | Roll-Up Schweiz → Region → Kanton → Quartal → Monat |
| **Roll-Up** | Drill-Down-Tabelle aggregiert Monate zu Quartalen und Jahrestotal |

### 4. KPI-Definition & Reporting
| KPI | Formel | Geschäftsbedeutung |
|-----|--------|--------------------|
| **Logiernächte Total** | Σ(Logiernächte) | Marktgrösse und Nachfragevolumen |
| **YoY-Wachstum** | (T₀ − T₋₁) / T₋₁ | Periodenvergleich, Trendindikator |
| **Internationalisierungsrate** | Ausland / Total | Marktoffenheit und Abhängigkeitsrisiko |
| **Saisonalitätsindex** | Peak(Monat) / Ø(Monat) | Off-Peak-Optimierungspotenzial |

### 5. Forecasting & Zeitreihenanalyse
- **Methode**: Holt-Winters doppelte exponentielle Glättung (Trend + multiplikative Saisonkomponente)
- **Zerlegung**: `Ŷₜ = (Level + h·Trend) × SaisonFaktor[t mod 12]`
- **Parameter**: α=0.30 (Level), β=0.10 (Trend), γ=0.25 (Saisonkomponente)
- **Prognose-Horizont**: 12 Monate (2025) mit ±8% Konfidenzband
- **Saisonale Dekomposition**: Trend × Saison × Irregulär (Vercellis-Logik)

### 6. Agentic BI / Natural Language Insights
- Template-basierte NL-Generierung aus aggregierten KPIs
- Erzeugt kontextuelle Handlungsempfehlungen (Wachstum, Risiko, Saisonalität)
- **Architektur für Erweiterung** mit `claude-sonnet-4-6` (Anthropic API) im Code dokumentiert
- Verkörpert Vercellis' Wertschöpfungskette: Kennzahlen → sprachliche Interpretation → Entscheidungsgrundlage

---

## Visualisierungen

| Chart | Typ | Zeigt |
|-------|-----|-------|
| Zeitreihe + Forecast | Liniendiagramm | Monatliche Logiernächte 2018–2024 + Prognose 2025 |
| Kanton-Ranking | Horizontales Balkendiagramm | Logiernächte nach Kanton (OLAP Drill-Down) |
| Herkunftsstruktur | Gestapeltes Balkendiagramm | Inland vs. Ausland nach Kanton (Slice & Dice) |
| Saisonalitäts-Heatmap | Farbkodierte Tabelle | Monat × Kanton (OLAP Dice) |
| KPI-Cards | Dashboard-Karten | 4 Kernkennzahlen mit Formel + Delta |
| Drill-Down-Tabelle | Pivot-ähnliche Tabelle | Quartal × Kanton (Roll-Up) |

---

## Projektstruktur

```
BI-Projekt/
└── index.html        # Vollständige Single-Page-Anwendung
                      # HTML + CSS + JS (keine externen Bibliotheken)
                      # Enthält: ETL, Dimensionsmodell, OLAP-Engine,
                      #          KPI-Logik, Forecast, AI-Insights
```

---

## Hinweis für den Dozenten

**Start**: `index.html` im Browser öffnen – fertig, keine Installation.

Das Dashboard demonstriert folgende **Modulkonzepte** (alle explizit im Code kommentiert):

- **ETL-Pipeline** mit Extract (BFS-Snapshot), Transform (Homogenisierung, Normierung) und Load (In-Memory-Faktentabelle)
- **Star-Schema-Modellierung** mit dokumentierter Granularität (Kanton × Monat × Herkunft)
- **OLAP Cube-Operationen**: Slice (Jahresfilter), Dice (Region + Herkunft), Drill-Down (Kanton/Quartal), Roll-Up (Jahrestotal)
- **4 KPIs** mit Formel, Geschäftsbedeutung und Tooltip im Code
- **Holt-Winters Forecast** mit saisonaler Dekomposition (Trend × Saison × Irregulär) und Konfidenzband 2025
- **Agentic BI / NL-Schicht**: Regelbasierte Entscheidungsempfehlungen aus KPI-Aggregaten; Erweiterung via Anthropic API (`claude-sonnet-4-6`) im Code dokumentiert
- Konsistente **OLAP-Interaktion**: Ein Filter (Jahr/Region/Herkunft) aktualisiert simultan alle 6 Visualisierungen und alle 4 KPI-Cards

Der **COVID-Schock 2020** (−40.5% Logiernächte, BFS-belegt) ist als markanter Bruchpunkt in der Zeitreihe sichtbar und dient als eindrückliches Beispiel für externe Schocks in BI-Systemen. Jahresfilter auf 2020 zeigt, wie alle KPIs konsistent reagieren.

---

*Erstellt im Rahmen des FHNW BI-Moduls 2025/26 · Daten: BFS HESTA (CC BY 4.0)*
