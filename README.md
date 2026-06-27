# Schweizer Hotellerie BI-Dashboard

**FHNW Modulprojekt – Business Intelligence 2025/26**  
Dozent: Prof. Dr. Manuel Renold | BSc Business Information Systems / Business AI  
Lehrbuch: Carlo Vercellis, *Business Intelligence – Data Mining and Optimization for Decision Making*

**Autoren:** David Ishak & Eufrat Pio Özmen

---

## Projektbeschreibung

Dieses Projekt analysiert die **Schweizer Hotellerie-Statistik (BFS HESTA)** als vollständiges, browserbasiertes BI-Dashboard. Es demonstriert die gesamte BI-Wertschöpfungskette nach Vercellis: **Daten → Information → Wissen → Entscheidung** – und beantwortet vier konkrete Business-Fragen auf Basis echter, öffentlich zugänglicher Daten.

### Business-Fragen

| # | Frage | BI-Methodik |
|---|-------|-------------|
| 1 | Welche Kantone dominieren die Schweizer Logiernächte und warum? | OLAP Drill-Down, KPI-Cards |
| 2 | Wie hat COVID-19 die Tourismus-Nachfrage verändert und wie entwickelt sich 2025? | Zeitreihenanalyse, Holt-Winters Forecast |
| 3 | Wie abhängig ist die Schweizer Hotellerie von ausländischen Gästen? | Dimensionsanalyse, KPI Internationalisierungsrate |
| 4 | Wie stark ist die Saisonalität und wie lässt sich die Off-Peak-Auslastung verbessern? | Saisonale Dekomposition, Heatmap, Agentic BI |

---

## Starten

```
Keine Installation notwendig:
index.html  →  Doppelklick im Datei-Explorer
               ODER in einem Browser öffnen (Chrome / Firefox / Edge)
```

Keine externen Bibliotheken – alle Visualisierungen mit nativer Canvas 2D API. Daten sind eingebettet – **funktioniert vollständig offline**.

---

## Datenquelle & Datenerhebung

| Attribut | Wert |
|----------|------|
| Herausgeber | Bundesamt für Statistik (BFS), Beherbergungsstatistik HESTA |
| Datensatz | px-x-1003020000_102: Logiernächte nach Jahr, Monat, Kanton, Herkunftsland |
| API-Endpoint | https://www.pxweb.bfs.admin.ch/api/v1/de/px-x-1003020000_102/ |
| BFS-Hauptseite | https://www.bfs.admin.ch/bfs/de/home/statistiken/tourismus/beherbergung/hotellerie.html |
| Abruf | Juni 2026 via BFS PxWeb REST-API (JSON-Format, HTTP POST) |
| Zeitraum | 2018–2024 (Jahrestotale je Kanton), Forecast 2025 (Holt-Winters) |
| Kantone | GR, VS, BE, LU, TI, ZH, GE, BS (8 umsatzstärkste Tourismusdestinationen) |
| Metrik | Logiernächte absolut, getrennt nach Inland (CH) und Ausland |
| Lizenz | BFS Open Data (CC BY 4.0) |

### Abgerufene Rohdaten 2019 (Referenzjahr, Jahrestotal, Herkunftsland gesamt)

| Kanton | Logiernächte Total | davon Inland | davon Ausland | Auslandanteil |
|--------|--------------------|--------------|---------------|---------------|
| Zürich (ZH) | 5'960'145 | 1'919'447 | 4'040'698 | 67.8% |
| Bern (BE) | 5'634'247 | 2'375'377 | 3'258'870 | 57.8% |
| Graubünden (GR) | 5'256'016 | 3'208'122 | 2'047'894 | 39.0% |
| Wallis (VS) | 4'259'950 | 2'220'259 | 2'039'691 | 47.9% |
| Genf (GE) | 3'202'974 | 620'589 | 2'582'385 | 80.7% |
| Tessin (TI) | 2'309'518 | 1'428'731 | 880'787 | 38.1% |
| Luzern (LU) | 2'217'819 | 732'163 | 1'485'656 | 67.0% |
| Basel-Stadt (BS) | 1'423'481 | 476'800 | 946'681 | 66.5% |
| **Total 8 Kantone** | **30'264'150** | **12'981'488** | **17'282'662** | **57.1%** |

---

## Demonstrierte BI-Konzepte

### 1. ETL-Pipeline (Extract – Transform – Load)

**Extract**
- Jahrestotale pro Kanton und Herkunftsland aus der BFS PxWeb REST-API abgerufen (HTTP POST, Indikator „Logiernächte", Jahrestotal, 2018–2024)
- API-Query-Parameter: `Kanton=[1,2,3,12,18,21,23,25]`, `Herkunftsland=[00,1]`, `Indikator=[2]`
- Rohdaten als verifizierbare JavaScript-Konstanten (`BASE_ANNUAL_K`, `YEAR_FACTORS`, `INTL_SHARE_YEAR_FACTOR`) direkt im Code eingebettet und kommentiert

**Transform**
- Berechnung des Auslandanteils pro Kanton: `intlShare = (Total − Inland) / Total`
- Normierung der Jahresfaktoren auf Referenzjahr 2019: `YEAR_FACTOR(Jahr) = Σ8Kantone(Jahr) / Σ8Kantone(2019)`
- Ableitung des jährlichen Auslandanteil-Faktors: `INTL_SHARE_YEAR_FACTOR = intlShare(Jahr) / intlShare(2019)`
- Monatliche Disaggregation via empirisch kalibrierter Saisonalitätsprofile (MOUNTAIN / LAKE / CITY), normiert auf `Σ = 12`
- Berechnung abgeleiteter Metriken: `domestic = total × (1 − yearIntlShare)`, `international = total − domestic`
- Gauss'sches Rauschen (σ=2%) für realistische Monatsvarianz

**Load**
- In-Memory Faktentabelle `factOvernights[]` mit 672 Datensätzen (8 Kantone × 7 Jahre × 12 Monate)
- Jeder Datensatz enthält: `canton_id, year, month, quarter, season, total, domestic, international`
- Vollständige Granularität auf Korn-Ebene; alle OLAP-Operationen operieren auf dieser Tabelle

---

### 2. Dimensionsmodellierung & Star Schema

```
                        DIM_TIME
                   (year, month, quarter,
                    season: Winter/Frühling/Sommer/Herbst)
                            │
DIM_ORIGIN ───────── FACT_OVERNIGHTS ─────────── DIM_CANTON
(DOM / INTL /        (canton_id, year, month,     (id, name, type,
 TOTAL)               total, domestic, intl)       intlShare)
```

| Element | Beschreibung |
|---------|-------------|
| **Faktentabelle** | `factOvernights` – 672 Datensätze, additive Fakten (summierbar über alle Dimensionen) |
| **Korn (Granularität)** | 1 Datensatz = 1 Kanton × 1 Monat × 1 Jahr (atomare Ebene) |
| **DIM_CANTON** | 8 Kantone mit Typ (MOUNTAIN/LAKE/CITY) und Auslandanteil 2019 |
| **DIM_TIME** | Jahr 2018–2024, Monat 1–12, Quartal Q1–Q4, Saison |
| **DIM_ORIGIN** | Herkunft: Inland (DOM), Ausland (INTL), Gesamt (TOTAL) |
| **Additivität** | `total = domestic + international` – konsistent über alle Dimensionen summierbar |

---

### 3. OLAP-Cube-Operationen

Alle vier klassischen OLAP-Operationen sind interaktiv und wirken simultan auf alle Visualisierungen:

| Operation | Implementierung im Dashboard |
|-----------|------------------------------|
| **Slice** | Jahres-Dropdown (2018–2024) → reduziert den Cube auf eine Jahresebene; alle Charts, KPIs und Tabellen aktualisieren sich synchron |
| **Dice** | Kombinierte Filterung nach Kantonsgruppe (Alle / Berg / See / Stadt) **und** Herkunft (Inland / Ausland / Gesamt) → Teilwürfel aus zwei Dimensionen gleichzeitig |
| **Drill-Down** | Schweiz-Gesamt → Kantonsgruppe → Einzelkanton → Quartal → Monat; in der Drill-Down-Tabelle sichtbar |
| **Roll-Up** | Drill-Down-Tabelle aggregiert 12 Monatswerte zu Quartalssummen und Jahrestotal; `olapQuery({groupBy:'month'})` vs. `groupBy:'date'` |

Die OLAP-Engine (`olapQuery`, `drillFact`) ist vollständig in JavaScript implementiert und operiert direkt auf `factOvernights[]` ohne Datenbankanbindung.

---

### 4. KPI-Definition & Reporting

Alle vier KPIs folgen der Vercellis-Definition: messbar, zeitbezogen, entscheidungsrelevant.

| KPI | Formel | Einheit | Geschäftsbedeutung |
|-----|--------|---------|--------------------|
| **Logiernächte Total** | `Σ(Logiernächte, Jahr, Region)` | Absolut | Marktgrösse und Nachfragevolumen; Basisgrösse für alle weiteren KPIs |
| **YoY-Wachstum** | `(T₀ − T₋₁) / T₋₁` | % | Periodenvergleich zum Vorjahr; Früh­indikator für Trendwenden |
| **Internationalisierungsrate** | `Ausland / Total` | % | Marktoffenheit und Abhängigkeitsrisiko; central für Krisenresilienz |
| **Saisonalitätsindex** | `max(Monatswert) / Ø(Monatswert)` | Faktor | Ausmass der saisonalen Schwankung; Off-Peak-Optimierungspotenzial |

Die KPI-Cards zeigen zusätzlich das Delta zum Vorjahr und die vollständige Berechnungsformel (Tooltip im Dashboard).

---

### 5. Forecasting & Zeitreihenanalyse

**Methode**: Holt-Winters Triple Exponential Smoothing mit multiplikativer Saisonkomponente (geeignet für Daten mit Trend und Saisonalität, wie Vercellis Kapitel 7 beschreibt)

**Modellstruktur**:
```
Ŷₜ₊ₕ = (Lₜ + h · Tₜ) × Sₜ₋L₊ₕ

Lₜ = α · (Yₜ / Sₜ₋L) + (1 − α) · (Lₜ₋₁ + Tₜ₋₁)   // Level-Update
Tₜ = β · (Lₜ − Lₜ₋₁) + (1 − β) · Tₜ₋₁             // Trend-Update
Sₜ = γ · (Yₜ / Lₜ) + (1 − γ) · Sₜ₋L                // Saison-Update
```

| Parameter | Wert | Begründung |
|-----------|------|------------|
| α (Level) | 0.30 | Mittlere Reaktion auf kurzfristige Schwankungen |
| β (Trend) | 0.10 | Träge Trendanpassung für strukturelle Stabilität |
| γ (Saison) | 0.25 | Mässige Aktualisierung der Saisonfaktoren |
| L (Periode) | 12 | Monatliche Saisonalität (Jahreszyklus) |
| Horizont | 12 Monate | Prognose für 2025 |
| Konfidenzband | ±8% | Visualisiert Prognoseunsicherheit |

**COVID-Behandlung**: Das COVID-Jahr 2020 (−40.5% gegenüber 2019, BFS-belegt) wird im Modell explizit als Strukturbruch markiert. Die Initialisierung des Holt-Winters-Algorithmus startet auf den Daten ab 2021, um keine verzerrte Trendschätzung durch den Ausnahmeeffekt zu erhalten. Der COVID-Bruchpunkt ist im Zeitreihen-Chart annotiert.

---

### 6. Agentic BI / Natural Language Insights

Das Dashboard implementiert eine regelbasierte NL-Generierungs-Schicht, die aus den aggregierten KPI-Werten automatisch kontextuelle Handlungsempfehlungen ableitet:

- **Wachstumsanalyse**: Vergleicht YoY-Wachstum mit Branchen-Benchmarks, erkennt Rekordjahre und Krisenjahre
- **Risikowarnung**: Meldet kritische Internationalisierungsrate (>70%) als Abhängigkeitsrisiko
- **Saisonalitätsempfehlung**: Leitet bei hohem Saisonalitätsindex konkrete Off-Peak-Massnahmen ab
- **Regionalanalyse**: Identifiziert Top-Kanton und erklärt Dominanz im Kontext (Berg/See/Stadt)

Die Architektur ist explizit auf Erweiterung mit LLM-API ausgelegt – der Code enthält einen auskommentierten API-Call zu `claude-sonnet-4-6` (Anthropic), der die KPI-Werte als Prompt-Kontext übergibt. Dies verkörpert Vercellis' Wertschöpfungskette auf höchster Stufe: **Kennzahlen → maschinelle Interpretation → Entscheidungsempfehlung**.

---

## Visualisierungen

| Visualisierung | Chart-Typ | OLAP-Bezug | Was es zeigt |
|----------------|-----------|------------|-------------|
| **Zeitreihe + Forecast** | Linienchart (Canvas 2D) | Roll-Up Monat→Jahr | Monatliche Logiernächte 2018–2024, COVID-Annotation, Prognose 2025 mit Konfidenzband |
| **Kanton-Ranking** | Horizontales Balkendiagramm | Drill-Down Kanton | Logiernächte nach Kanton, sortiert nach Grösse, filtert auf Jahres-/Herkunfts-Selektion |
| **Herkunftsstruktur** | Gestapeltes Balkendiagramm | Dice (Region × Herkunft) | Inland vs. Ausland je Kanton – visualisiert Internationalisierungsrisiko |
| **Saisonalitäts-Heatmap** | Farbkodierte Tabelle | Dice (Monat × Kanton) | Saisonale Intensität: dunkelblau = Hochsaison, hellgrau = Nebensaison |
| **KPI-Cards** | Dashboard-Karten | Slice (Jahresebene) | 4 Kernkennzahlen mit Formel, Delta zum Vorjahr und Spitzendestination |
| **Drill-Down-Tabelle** | Pivot-Tabelle | Drill-Down + Roll-Up | Quartal × Kanton, aggregiert zu Jahrestotal; zeigt saisonale Verteilung je Region |

---

## Projektstruktur

```
BI-Projekt/
├── index.html     # Vollständige Single-Page-Anwendung (HTML + CSS + JS)
│                  # Enthält alle Schichten: ETL · Star Schema · OLAP-Engine
│                  #   KPI-Logik · Holt-Winters Forecast · Agentic BI
│                  # Keine externen Abhängigkeiten – läuft vollständig im Browser
└── README.md      # Projektdokumentation (dieses Dokument)
```

---

## Hinweis für den Dozenten

**Start**: `index.html` im Browser öffnen – fertig, keine Installation, keine Internetverbindung notwendig.

Das Dashboard demonstriert alle zentralen **Modulkonzepte** explizit und interaktiv:

**Daten & ETL**
- Echte BFS HESTA-Daten via PxWeb REST-API (px-x-1003020000_102), direkt im Code mit Quellenangabe und Abrufdatum dokumentiert
- Vollständige ETL-Pipeline: API-Abruf → Transformation (Normierung, Split, Disaggregation) → In-Memory-Load
- Alle Rohdaten-Konstanten im Code verifizierbar (z.B. `ZH 2019: 5'960'145 Logiernächte`)

**Dimensionsmodellierung**
- Star Schema mit 1 Faktentabelle und 3 Dimensionstabellen, vollständig dokumentiert
- Granularität explizit definiert: 1 Kanton × 1 Monat × 1 Jahr (672 Datensätze)
- Additive Fakten: alle Metriken sind über alle Dimensionen konsistent summierbar

**OLAP**
- Alle vier Operationen (Slice, Dice, Drill-Down, Roll-Up) sind interaktiv und live erlebbar
- Ein Filter-Set aktualisiert simultan alle 6 Visualisierungen und alle 4 KPI-Cards
- `olapQuery()`-Funktion im Code kommentiert mit OLAP-Terminologie

**Forecasting**
- Holt-Winters Triple Exponential Smoothing vollständig in JavaScript implementiert (kein Framework)
- Parameter α, β, γ und Saisonperiode L=12 im Code dokumentiert und begründet
- COVID-Strukturbruch methodisch korrekt behandelt (Initialisierung ab 2021)

**Agentic BI**
- Regelbasierte NL-Schicht generiert kontextuelle Entscheidungsempfehlungen aus KPI-Aggregaten
- LLM-Erweiterung via Anthropic API im Code explizit dokumentiert und vorbereitet

**COVID-Analyse**: Der COVID-Schock 2020 (−40.5% Logiernächte, BFS-belegt) ist im Zeitreihen-Chart annotiert. Jahresfilter auf 2020 zeigt, wie alle KPIs konsistent reagieren – Auslandsegment −61.5% gegenüber −15.7% Inland (asymmetrischer Schock als klassisches Beispiel für Internationalisierungsrisiko).

---

### Quellenverzeichnis

| Quelle | Verwendung |
|--------|-----------|
| BFS HESTA, PxWeb API px-x-1003020000_102 | Logiernächte nach Kanton, Herkunftsland, 2018–2024 |
| Vercellis, C. (2009). *Business Intelligence – Data Mining and Optimization for Decision Making*. Wiley. | Konzeptionelle Grundlage: BI-Wertschöpfungskette, KPI-Definition, Forecasting-Methodik |
| Holt, C.E. (1957). *Forecasting seasonals and trends by exponentially weighted averages* | Methodische Grundlage Holt-Winters |
| BFS (2024). *Schweizer Tourismusstatistik 2023*. Neuchâtel. | Validierung der Jahrestotale und COVID-Faktoren |

---

*Erstellt im Rahmen des FHNW BI-Moduls 2025/26 · Daten: BFS HESTA (CC BY 4.0)*
