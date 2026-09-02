# PRD — ICT PD Array Suite

**Auteur**: d.nuland
**Licentie**: MPL 2.0
**Status**: v1 in ontwikkeling (Pine bron-implementatie eerst; MQL5- en cTrader-ports als aparte epics)

De indicator wordt uiteindelijk in **drie implementaties** geleverd, met exact dezelfde detectie-/lifecycle-logica (§4) en alleen platform-gedwongen verschillen in rendering/alerts (§6):

| Bestand | Platform | Status |
|---|---|---|
| [`PDArray.pine`](PDArray.pine) | TradingView (Pine Script v6) | Bron-implementatie, wordt als eerste volledig gebouwd |
| [`PDArray.mq5`](PDArray.mq5) | MetaTrader 5 (MQL5) | 1-op-1 poort — aparte epic (E8), nog niet gestart |
| [`PDArray.cs`](PDArray.cs) | cTrader (cAlgo C#) | 1-op-1 poort — aparte epic (E9), nog niet gestart |

---

## 1. Probleemstelling

ICT/SMC-traders werken met een grote, samenhangende set **PD arrays** (Premium/Discount arrays): order blocks, fair value gaps, breakers, mitigation blocks, rejection blocks, liquidity voids, volume imbalances, opening gaps, buyside/sellside liquidity, equal highs/lows, en het dealing-range-raamwerk (premium/discount/equilibrium/OTE). In de praktijk betekent dit:

- **Versnippering**: voor elk concept een aparte indicator (of handmatig tekenen). Geen enkele weergave toont het geheel, en de verschillende scripts hanteren onderling afwijkende definities, kleuren en "opruim"-gedrag.
- **Chart-vervuiling**: PD-array-indicators tekenen elke gevonden zone en verwijderen niets. Na een paar honderd bars staat de grafiek vol met tientallen overlappende boxes zonder dat duidelijk is welke nog onmitigated (relevant) zijn.
- **Geen levenscyclus**: een order block die volledig doorhandeld is, een FVG die gevuld is, of een swing high die geveegd is, blijft even prominent staan als een verse, ongetouchte array.

**Doel van dit product**: één indicator die **alle PD arrays die ICT onderscheidt** detecteert, waarbij **elke array-soort afzonderlijk aan/uit** kan, met een **gedeelde levenscyclus** (formed → tapped → mitigated → opgeruimd), een **begrensd object-budget** (nooit meer dan N zichtbaar), consistente kleuren/labels en een compact info-paneel — zodat de chart op elke historielengte leesbaar en comfortabel blijft.

**Primaire toepassing bij de gebruiker**: visuele/discretionaire analyse op AUDCAD (Darwinex), H4/H1 — als aanvullend instrument naast de SQX-strategieontwikkeling en de bestaande losse indicatoren (`Wavetrend_Indicator`, `SupportResistance_Indicator`, `FVG_OrderBlocks_Indicator`, `Market_Structure_MTF`). Deze suite is een **zelfstandige superset**: FVG/Order Block/Breaker worden hier opnieuw en zelfstandig geïmplementeerd; het losse `FVG_OrderBlocks_Indicator`-project blijft parallel bestaan.

## 2. Doelgroep

- ICT/SMC-traders die nu 3–6 losse indicatoren stapelen en één samenhangende, opgeruimde weergave willen.
- Discretionaire traders op FX/CFD zonder betrouwbare volumedata (zoals AUDCAD@Darwinex), naast crypto/aandelen mét volume.
- Gebruikers van TradingView (Pine), MetaTrader 5 (MQL5) en cTrader (cAlgo C#).

## 3. Doelen & niet-doelen

**Doelen**
- Eén configureerbare indicator (geen strategie/backtester) die **op de hoofdgrafiek** tekent (`overlay = true` / MT5 chart-window / cAlgo chart-indicator).
- **18 afzonderlijk schakelbare PD-array-modules** (§4.1–§4.18), elk met eigen aan/uit, eigen kleur, eigen "max getoond" en eigen mitigatie-gedrag.
- Eén **gedeelde PD-array core** (§4.0): generieke representatie, levenscyclus, renderer, decluttering, info-paneel en alert-dispatcher — zodat alle 18 modules er identiek uitzien en zich identiek gedragen.
- Bewuste **decluttering**: per module een instelbaar maximum aantal gelijktijdig getoonde arrays, automatische veroudering, auto-fade/auto-remove van gemitigeerde arrays, merge van overlappende same-type arrays, en een globaal object-budget.
- **Non-repainting waar vermeld kan worden**: pivot-gebaseerde modules bevestigen `pivotRightBars` bars na het swingpunt; gap-/displacement-gebaseerde modules op `barstate.isconfirmed`. De confirmatie-vertraging is een geaccepteerde ontwerp-eigenschap (zelfde patroon als `Wavetrend_Indicator` §4.7 en `SupportResistance_Indicator` §5).
- Platformpariteit tussen Pine, MQL5 en cTrader (§6), met gedocumenteerde afwijkingen waar het platform dat dwingt.

**Niet-doelen**
- Geen geautomatiseerde order-executie.
- Geen backtesting/strategy-tester (`strategy()`/cBot).
- Geen MTF-overlay in v1 (elke module detecteert op de chart-timeframe). MTF is een latere epic (E11).
- Geen volledige confluence-/scoring-engine in v1 (Unicorn = Breaker∩FVG, OTE+OB-stacking, geweegde PD-array-score). BPR (§4.14) is de enige concrete confluence-array in v1; bredere scoring is een latere epic (E12).
- Geen volume-profile als detectiemethode — de meeste FX/CFD-symbolen (incl. AUDCAD@Darwinex) hebben geen betrouwbare `volume`. Volume is hooguit een optionele secundaire sterkte-/kwaliteitsfactor, met zichtbare fallback (zelfde constraint als `Wavetrend_Indicator` §4.2 / `SupportResistance_Indicator` §3).
- Geen garantie op winstgevendheid; puur signaal-/analysetooling.

## 4. Functionele requirements

### 4.0 Gedeelde PD-array core (§ Shared Core) — Epic E1

- **Generieke representatie**: elke gedetecteerde array is één record met minimaal: `moduleId`, `typeLabel` (bijv. `"OB+"`, `"FVG-"`, `"BPR"`, `"BSL"`), `dir` (+1 bullish / -1 bearish / 0 neutraal), `top`, `bottom`, `ce` (midpoint / consequent encroachment), `originBar`, `originTime`, `status` (`formed` | `tapped` | `mitigated`), `lastTouchBar`, `strength`, en de teken-handles (box/lines/label).
  - **Pine v6**: geïmplementeerd als een `type PDArray` + `array<PDArray>` registry. Dit wijkt bewust af van de parallelle-arrays-aanpak in `SupportResistance_Indicator` (§6 aldaar) — met 18 modules en één gedeelde registry is een UDT beter leesbaar en minder foutgevoelig. Ports mappen dit naar een `struct` (MQL5) resp. `class`/`record` (cTrader) — zie §6.
- **Levenscyclus** (elke bevestigde bar, voor elke array):
  - `formed` → `tapped` zodra de prijs de zone voor het eerst raakt (`high >= bottom and low <= top`).
  - `tapped` → `mitigated` volgens per-module configureerbare regel: default is "een bevestigde close voorbij de verre grens" (volledige doorhandeling). Voor niveau-arrays (liquidity, referentielevels, equal H/L) is `mitigated` = "geveegd" (`swept`): prijs handelt door het niveau.
  - `mitigated` → opgeruimd: per module instelbaar `Mitigated Handling` = `Fade` (transparanter tonen gedurende `Fade Bars`, dan verwijderen), `Remove` (direct verwijderen), of `Keep` (blijven tonen met afwijkende stijl).
  - **Veroudering**: een `formed`/`tapped` array zonder enige interactie binnen `Array Max Age (bars)` (per module, default 500) wordt verwijderd.
- **Renderer** (gedeeld): zonale arrays → box tussen `top`/`bottom` vanaf `originBar` (+ kleine rechter-marge, optioneel `Extend Right`); niveau-arrays → `line`(s). Optionele **CE/50%-midline** per array-type (niet alleen FVG). Label rechts met `typeLabel` (+ optioneel touch-count). Transparantie schaalt met `strength`/versheid. `mitigated` arrays krijgen een afwijkende stijl (stippelrand / extra transparant).
- **Decluttering** (gedeeld, §4.21): per module top-`Max Shown` op `strength`; globaal object-budget; merge/dedupe van overlappende same-module arrays van dezelfde `dir`.
- **Alert-dispatcher** (gedeeld, §4.20): centrale functie `f_alert(event, array)` die zowel de statische `alertcondition`-vlaggen zet als (optioneel) een JSON-payload via `alert()` verstuurt.
- **Info-paneel** (gedeeld, §4.19).
- **Displacement-gate**: modules 1 (OB), 2 (Breaker), 5 (FVG) en 18 (BOS/MSS) kunnen optioneel eisen dat module 16 (Displacement) op het relevante moment een verplaatsing detecteerde (`Require Displacement` per module, default aan voor OB/FVG).

### 4.1 Order Block (§ Order Block) — Epic E3
- **Bullish OB**: de laatste down-close candle (of down-body cluster) vóór een bullish verplaatsing die de directe swing-structuur breekt. **Bearish OB**: spiegelbeeld.
- Zone = candle `high`–`low` (default) of candle-body (`Use Body Only`, instelbaar).
- **CE** = 50% van de zone.
- Sub-vlaggen (geen aparte module): **Propulsion** (nieuwe OB die binnen/tegen een bestaande OB van dezelfde `dir` vormt) en **Reclaimed** (prijs keert terug en sluit weer door de OB heen zonder hem te mitigeren) — getoond via labelsuffix (`OB+ prop`, `OB+ rcl`).
- **Mitigated**: bevestigde close volledig voorbij de verre zonegrens.

### 4.2 Breaker Block (§ Breaker) — Epic E3
- Een OB (§4.1) die eerst gerespecteerd wordt, dan **gevioleerd** (prijs sluit er doorheen), en waarna prijs **terugkeert** en van de andere kant reageert → de gefaalde OB wordt een breaker van tegengestelde `dir`.
- Vereist het intern volgen van de OB-toestand: `respected → violated → retested`. Zone = oorspronkelijke OB-zone; `dir` klapt om.
- Aparte kleur/label (`BRK+` / `BRK-`).

### 4.3 Mitigation Block (§ Mitigation Block) — Epic E3
- De laatste tegengestelde candle van een gefaalde swing waar prijs naar terugkeert **zonder** dat er een volledige structuurbreuk plaatsvond (onderscheid met breaker: breaker = mét BOS, mitigation block = zonder).
- Zone = die candle; label `MB+` / `MB-`.

### 4.4 Rejection Block (§ Rejection Block) — Epic E3
- Candle met een uitgesproken wick (rejection) op een swingpunt: wick-lengte ≥ `Rejection Wick Ratio` × candle-range en ≥ `Min Wick (x ATR)`.
- Zone = het wick-gebied (van `max(open,close)` tot `high` voor een bearish rejection; spiegel voor bullish). Label `RB+` / `RB-`.

### 4.5 Fair Value Gap + Consequent Encroachment (§ FVG) — Epic E2
- **Bullish FVG (BISI)**: `low[0] > high[2]` — de zone tussen `high[2]` en `low[0]`. **Bearish FVG (SIBI)**: `high[0] < low[2]`.
- Filters: `Min Gap Size (x ATR)` (default 0.10), optioneel `Require Displacement` (§4.0).
- **CE** = 50% van de gap (aparte lijn, apart toggle).
- **Mitigated**: default = prijs bereikt CE (`50% fill`); optioneel `Full fill` (verre grens) via `FVG Fill Rule`.

### 4.6 Inversion FVG / IFVG (§ IFVG) — Epic E2
- Een FVG (§4.5) die **volledig doorhandeld** wordt en waarbij een bevestigde close voorbij de verre grens sluit → de FVG wordt een IFVG van tegengestelde `dir` (support wordt resistance en omgekeerd).
- Vereist het volgen van FVG-toestand. Aparte kleur/label (`IFVG+` / `IFVG-`).

### 4.7 Liquidity Void (§ Liquidity Void) — Epic E2
- Een snelle, grotendeels één-richting beweging over `Void Min Bars` (default 3) opeenvolgende candles met totale range ≥ `Void Min Range (x ATR)` (default 2.5) en minimale candle-overlap (elke candle opent ≈ waar de vorige sloot).
- Zone = van begin tot eind van de void. **Mitigated** = prijs retraceert ≥ `Void Fill %` (default 50%) terug door de void.

### 4.8 Volume Imbalance (§ Volume Imbalance) — Epic E2
- Gap **tussen candle-bodies** terwijl de wicks nog overlappen: `open[0] > close[1]` (bullish) met `low[0] <= high[1]` (wicks raken) → kleine zone tussen `close[1]` en `open[0]`. Spiegel voor bearish.
- Klein zonaal object; **mitigated** = prijs sluit de body-gap.

### 4.9 Opening Gaps — NDOG / NWOG (§ Opening Gaps) — Epic E2
- **NDOG** (New Day Opening Gap): verschil tussen de laatste close van de vorige handelsdag en de eerste open van de nieuwe handelsdag. **NWOG** (New Week Opening Gap): laatste close vrijdag ↔ eerste open zondag/maandag.
- Gedetecteerd op de chart-timeframe door een nieuwe dag/week-grens te herkennen (`ta.change(time("D"))` / `time("W")`) en de vorige close met de nieuwe open te vergelijken.
- Zone tussen de twee prijzen + **CE-lijn**. Bewaar de laatste `Max NDOG` / `Max NWOG` (default 3 elk). **Mitigated** = prijs sluit de gap volledig.

### 4.10 Buyside / Sellside Liquidity (§ Liquidity) — Epic E4
- **BSL** = onbevestigd-gebroken swing highs (old highs); **SSL** = swing lows (old lows). Elk als horizontale lijn vanaf het swingpunt, `Extend Right`, label `BSL` / `SSL`.
- **Sweep-detectie**: zodra prijs door het niveau handelt → status `mitigated` (`swept`), optioneel kort een `SWEPT`-markering, dan opruimen volgens `Mitigated Handling`.
- **Trendline liquidity** (sub-feature, v1 vereenvoudigd): 3 oplopende swing lows / aflopende swing highs → één diagonale lijn. Bij twijfel over robuustheid schuift dit naar §8 / een latere refinement-issue.

### 4.11 Equal Highs / Equal Lows (§ Equal H/L) — Epic E4
- 2+ swing highs binnen `Equal Tolerance (x ATR)` (default 0.10) van elkaar → **EQH** (liquiditeitspool); spiegel voor **EQL**.
- Weergave: dunne lijn/box die de gelijke punten verbindt + label `EQH` / `EQL` met count.
- **Mitigated** = pool geveegd (prijs door het niveau).

### 4.12 Dealing Range — Premium/Discount/Equilibrium + OTE/Fib (§ Dealing Range) — Epic E5
- **Range-bron** (`Dealing Range Source`):
  - `Auto swing` (default): meest recente gekwalificeerde swing high ↔ swing low (pivot-lookback instelbaar). Herberekent bij een nieuwe gekwalificeerde swing.
  - `Previous period`: high/low van de vorige dag / week / maand (`Dealing Range Period`).
  - `Session`: high/low van een gekozen sessie (Asia / London / New York, zie §4.17-tijdzone).
- Tekent: optionele range-box, **EQ (50%)**-lijn, **Premium**-zone (50–100%), **Discount**-zone (0–50%), **OTE-band** (default 0.62–0.79 retracement van het been), plus configureerbare fib-levels (`0.705` sweet spot, `-0.5`, `1.0`, extensies `-0.27`/`-1.0` — via een komma-gescheiden input).
- Premium/discount-labeling volgt de richting van het laatste been (impuls omhoog → retracement in discount = koopzone).

### 4.13 Reference Levels — PDH/PDL, PWH/PWL, PMH/PML (§ Reference Levels) — Epic E5
- Vorige-periode high/low voor **Dag** (PDH/PDL), **Week** (PWH/PWL) en **Maand** (PMH/PML), elk afzonderlijk schakelbaar.
- **Pine**: `request.security(syminfo.tickerid, "D"|"W"|"M", [high[1], low[1]], lookahead = barmerge.lookahead_off)` — de `[1]`-offset garandeert de laatst volledig gesloten periode (non-repaint).
- Lijnen met `Extend Right` + labels. Bij doorhandelen → `SWEPT`-markering.

### 4.14 Balanced Price Range / BPR (§ BPR) — Epic E2
- Wanneer een **bullish FVG** en een **bearish FVG** (§4.5) elkaar in prijs **overlappen** en binnen `BPR Max Bars Apart` (default 10) van elkaar gevormd zijn → de overlap-regio is een BPR.
- Afgeleid uit de door module 5 getrackte FVG's (geen eigen 3-candle-scan). Aparte kleur/label `BPR`. **Mitigated** = prijs sluit voorbij de verre BPR-grens.

### 4.15 SMT Divergence (§ SMT) — Epic E6
- Vergelijkt de chart-symbol met 1 (v1) of optioneel 2 **gecorreleerde symbolen** (`SMT Symbol 1/2`, bijv. voor AUDCAD: `FX:AUDUSD` en `FX:USDCAD`, of goud/olie).
- Op uitgelijnde swingpunten: chart maakt een **higher high** terwijl het gecorreleerde symbool een **lower high** maakt (of omgekeerd voor lows) → **SMT-divergentie**. Bearish SMT op highs, bullish SMT op lows.
- Weergave: label op het chart-swingpunt (`SMT bear` / `SMT bull`) + optionele lijn tussen de twee betrokken swings.
- **Pine**: `request.security` voor de gecorreleerde `high`/`low`-series; swing-uitlijning op basis van gedeelde pivot-bars.
- v1-beperking: geen automatische correlatie-detectie; de gebruiker geeft de symbolen op.

### 4.16 Displacement (§ Displacement) — Epic E4
- Detecteert een energieke verplaatsing: één bar of een run van ≤ `Displacement Max Bars` (default 3) bars met totale gerichte range ≥ `Displacement (x ATR)` (default 1.5) en gemiddelde body/range-ratio ≥ `Min Body Ratio` (default 0.5).
- **Optioneel zichtbaar**: de displacement-leg markeren (pijl / dunne box).
- **Kern-functie**: exposeert `bullDisplacement` / `bearDisplacement` (booleans) die door modules 1, 2, 5 en 18 als kwaliteitsgate gebruikt worden (`Require Displacement`).

### 4.17 Key Levels & Opens (§ Key Levels) — Epic E5
- **True Day Open** = 00:00 America/New_York (`Midnight Open`); **08:30 NY Open** (news open); optioneel **True Week/Month Open**.
- **Sessie-highs/lows**: Asia, London, New York — begin/eind instelbaar, standaard ICT-vensters. Elke sessie: high/low-lijnen voor de lopende + laatste `Session History` (default 1) sessies.
- Tijdzone-input (`Timezone`, default `America/New_York`) gedeeld met §4.12 (`Session`-bron).
- Lijnen + labels; opens met `Extend Right`.

### 4.18 Market Structure — BOS / MSS / CHoCH (§ Market Structure) — Epic E4
- Volgt bevestigde swing highs/lows (pivots) en een interne trend-toestand.
- **BOS** (Break of Structure): bevestigde close voorbij de laatste beschermde swing **in de richting van de bestaande trend** → continuatie. Label `BOS`.
- **MSS / CHoCH**: bevestigde close voorbij de laatste beschermde swing **tegen** de bestaande korte-termijn-trend (optioneel `Require Displacement`) → mogelijke reversal. Label `MSS`.
- Elk gebroken swingpunt: horizontale lijn van het swingpunt tot de breek-bar + `×`-markering "liquidity taken" + label.
- Config: `Structure Basis` = `Close` of `Wick`; v1 werkt op één niveau (swing structure), interne (lower-timeframe) structuur is een latere refinement.

### 4.19 Info-paneel (§ Info Panel) — Epic E7
- Optioneel, positie instelbaar (4 hoeken). Toont:
  - Positie in de dealing range (premium / discount + % afstand tot EQ).
  - Dichtstbijzijnde onmitigated bullish PD array **onder** de prijs (type + afstand).
  - Dichtstbijzijnde onmitigated bearish PD array **boven** de prijs (type + afstand).
  - Laatste liquidity sweep (`BSL @ 0.6841, 3 bars geleden`).
  - Laatste structuur-event (`MSS bull @ 0.6790, 7 bars geleden`).
  - Aantal actieve modules / totaal getekende objecten.
- Puur informatief; geen invloed op detectie, lifecycle of alerts.

### 4.20 Alerts (§ Alerts) — Epic E7
- Per module (waar zinvol): `Array Formed`, `Array Tapped`, `Array Mitigated`, plus module-specifiek: `Liquidity Swept` (§4.10/§4.13), `Structure Shift` (§4.18), `SMT Divergence` (§4.15), `Price entered Discount+OTE` (§4.12).
- **Statisch**: `alertcondition()` per event-familie (altijd beschikbaar, ook zonder dynamische alerts).
- **Dynamisch** (optioneel, `Enable JSON Webhook Alerts`): `alert()` met payload
  `{ticker, timeframe, event_type, module, array_type, direction, top, bottom, ce, price, time}` — zelfde schema-aanpak als `Wavetrend_Indicator` §4.12 en `SupportResistance_Indicator` §4.8.
- Debounce: `Tapped`-alerts per array één keer; reset pas als prijs > `Alert Reset (x ATR)` van de array verwijderd is.

### 4.21 Decluttering / object-budget (§ Declutter) — Epic E1/E7
- Per module: `Max Shown` (default per module 3–8). Van alle intern getrackte arrays worden alleen de sterkste/verste getekend; de rest blijft getrackt en kan terugkeren.
- Globaal: harde interne cap van **500 getrackte arrays** (niet-configureerbaar, performance) en Pine-object-limieten (`max_boxes_count`/`max_lines_count`/`max_labels_count` op 500).
- Merge/dedupe: nieuwe array van dezelfde module + `dir` die > `Merge Overlap %` (default 80%) overlapt met een bestaande → versmelt i.p.v. duplicaat.
- Globale `Lookback Bars`-input begrenst hoe ver terug detectie draait (performance).
- `Compact Mode`: dunnere randen, geen CE-lijnen, alleen labels bij `strength` boven drempel.

### 4.22 Inputs / UX (§ Inputs) — Epic E1/E7
- **Master switch** bovenaan (`Enable All Modules` als noodrem).
- Eén `group` per module, in vaste volgorde, elk met: `Enable`, `Bull Color`, `Bear Color`, `Max Shown`, `Mitigated Handling`, `Extend Right` (waar van toepassing), plus module-specifieke drempels — allemaal met `tooltip`.
- Globale groepen: `Global / Theme` (kleuren-preset, compact mode, transparantie-schaal, lookback), `Info Panel`, `Alerts`.
- Standaard-kleuren: bullish = teal-familie, bearish = rood-familie, neutraal (dealing range / referentielevels) = grijs/blauw — met per-module afwijkingen zodat soorten onderscheidbaar blijven.

## 5. Niet-functionele requirements

- **Non-repainting waar vermijdbaar**: pivot-modules bevestigen na `pivotRightBars`; gap-/displacement-modules op `barstate.isconfirmed`. De vertraging is ontwerp, geen bug (zelfde afweging als `Wavetrend_Indicator` §4.7).
- **Performance**: interne cap 500 getrackte arrays; getekende objecten begrensd door de per-module `Max Shown` + Pine-limieten; `Lookback Bars` begrenst het detectie-venster.
- **Robuustheid tegen ontbrekende volumedata**: modules die volume gebruiken (optionele sterkte-weging) vallen zichtbaar terug op ongewogen gedrag (info-paneel toont dit), zelfde patroon als de zusterprojecten.
- **Configureerbaarheid**: alle drempels/lengtes/kleuren zijn inputs, gegroepeerd per module met tooltips.
- **Compatibiliteit**: Pine Script v6; MQL5 (MT5); cAlgo API (cTrader).
- **Platformpariteit**: identieke detectie-/lifecycle-logica (§4.0–§4.18); verschillen beperkt tot platform-gedwongen rendering/alerts (§6).
- **Geen invloed op de bestaande SQX-workflow**: losstaand analyse-instrument.

## 6. Platformverschillen

| Aspect | Pine | MQL5 | cTrader (cAlgo) |
|---|---|---|---|
| Overlay | `indicator(overlay = true)` | `#property indicator_chart_window` | `[Indicator(IsOverlay = true, ...)]` |
| PD-array-representatie | `type PDArray` + `array<PDArray>` | `struct PDArray` + dynamische array | `class PDArray` + `List<PDArray>` |
| Pivot-detectie | `ta.pivothigh` / `ta.pivotlow` | Handmatige window-scan op gesloten bars | Handmatige window-scan op gesloten bars |
| Zone-object | `box.new()` | `OBJ_RECTANGLE` | `Chart.DrawRectangle()` |
| Lijn-object | `line.new()` | `OBJ_TREND` / `OBJ_HLINE` | `Chart.DrawTrendLine()` / `DrawHorizontalLine()` |
| Label | `label.new()` | `OBJ_TEXT` / `OBJ_LABEL` | `Chart.DrawText()` |
| Info-paneel | `table.*` | Losse `OBJ_LABEL`-objecten | `Chart.DrawStaticText()` |
| Andere-timeframe data (§4.9/§4.13/§4.15/§4.17) | `request.security()` (+ `[1]`-offset voor confirmed) | `CopyHigh`/`CopyLow`/`CopyClose`/`CopyTime` op de doel-timeframe | `MarketData.GetBars(tf)` |
| Gecorreleerd symbool (§4.15) | `request.security(otherSym, tf, ...)` | `SymbolSelect` + `CopyHigh`/`CopyLow` | `MarketData.GetSymbol(...).GetBars(...)` |
| Volume | `volume` (broker-afhankelijk) | `tick_volume` (real volume meestal 0 op FX/CFD) | `Bars.TickVolumes` |
| Alerts | `alertcondition()` + optionele JSON `alert()` | `Alert()` + `SendNotification()` | `Print()` + chart-markering (push/e-mail buiten scope v1) |
| Herberekening | Volledig script vanaf bar 0 | `OnCalculate` met `prev_calculated` (incrementeel) | `Calculate(index)` per bar |

## 7. User stories (samengevat)

1. *Als ICT-trader* wil ik alle PD arrays in één indicator, elk afzonderlijk aan/uit, zodat ik niet 5 losse scripts hoef te stapelen.
2. *Als trader* wil ik dat gemitigeerde arrays (gevulde FVG's, doorhandelde OB's, geveegde highs) automatisch vervagen of verdwijnen, zodat alleen relevante arrays overblijven.
3. *Als trader* wil ik nooit meer dan een instelbaar aantal arrays per soort tegelijk zien, zodat de chart leesbaar blijft op elke historielengte.
4. *Als trader* wil ik in één oogopslag zien of ik in premium of discount zit en wat de dichtstbijzijnde onmitigated array boven/onder de prijs is.
5. *Als trader* wil ik onderscheid tussen BOS (continuatie) en MSS/CHoCH (reversal), met het gebroken swingpunt gemarkeerd als genomen liquidity.
6. *Als AUDCAD-trader* wil ik SMT-divergentie t.o.v. AUDUSD/USDCAD kunnen zien zonder een apart script.
7. *Als trader met alerts* wil ik per array-soort gewaarschuwd worden bij vorming/tap/mitigatie, zonder spam zolang de prijs in de buurt blijft.
8. *Als MetaTrader- of cTrader-gebruiker* wil ik dezelfde indicator, functioneel gelijk aan de Pine-versie.

## 8. Bekende beperkingen / bewuste scope-keuzes

- **Geen MTF-overlay in v1** (elke module draait op de chart-timeframe). Latere epic E11.
- **Geen brede confluence-scoring in v1** — alleen BPR (§4.14) als concrete confluence-array. Latere epic E12.
- **Trendline liquidity** (§4.10) is v1-vereenvoudigd (rechte lijn door 3 punten); geen dynamische her-fit bij nieuwe raakpunten.
- **SMT** (§4.15): geen automatische correlatie-detectie; gebruiker geeft symbolen op. Max 2 gecorreleerde symbolen in v1.
- **Market Structure** (§4.18): v1 werkt op één (swing-)niveau; geen aparte internal/LTF-structuurlaag.
- **Mitigation vs Breaker** (§4.2/§4.3): het onderscheid (wel/geen BOS) hangt af van module 18's structuur-lezing; bij module 18 uit valt de indicator terug op een heuristiek (violated + retest zonder nieuwe swing = mitigation block).
- **Volume-weging** is altijd secundair (optionele sterkte-/kwaliteitsfactor), nooit detectie-basis; zichtbare fallback zonder volumedata.
- **Opening gaps / key opens / sessies** worden op de chart-timeframe afgeleid; op zeer hoge timeframes (≥ D1) zijn NDOG/sessie-levels per definitie niet zinvol en worden ze verborgen met een statusmelding.
- **Repaint tijdens vorming**: de laatst-gevormde pivot-array verschijnt pas na `pivotRightBars` bars; dit is bewust.

## 9. Verificatiestatus

- **Pine**: geschreven en gecontroleerd via handmatige code-review van de Pine v6-taalregels; nog niet gecompileerd/getest op een live TradingView-chart (geen TradingView-compiler in deze omgeving). Hoogste-risico-onderdelen bij een eerste live test: §4.15 (SMT `request.security` op een ander symbool), §4.12/§4.17 (tijdzone- en sessie-logica), §4.2/§4.6 (toestand-machines OB→breaker en FVG→IFVG).
- **MQL5 / cTrader**: nog niet gestart (epics E8/E9).
- Alle versies: array-plaatsing/lifecycle moet visueel vergeleken worden tussen platformen op hetzelfde instrument/timeframe voordat er alerts op draaien.

## 10. Distributie

- **Pine**: kopiëren naar de TradingView Pine Editor, toevoegen aan de chart als overlay op de hoofdgrafiek.
- **MQL5**: naar `MQL5\Indicators\`, compileren (F7) of MT5 herstarten.
- **cTrader**: naar `Sources\Indicators\`, bouwen in cTrader Automate.

## 11. Agile — Epics & backlog

Milestones op GitHub = epics; issues = stories. Elke module-issue heeft acceptatiecriteria in de vorm "detectie klopt op referentie-voorbeeld / lifecycle correct / rendering + toggle + kleur + max-shown werkt / geen repaint na bevestiging".

| Epic (milestone) | Doel | Issues (stories) |
|---|---|---|
| **E1 — Scaffold & Shared Core** | Repo, PRD, README, licentie; indicator-skelet; `PDArray` UDT + registry; lifecycle-manager; gedeelde renderer; decluttering + object-budget; input-framework + master switch; alert-dispatcher-stub; info-paneel-stub | S1.1 repo + docs scaffold · S1.2 indicator-skelet + input-framework · S1.3 `PDArray` UDT + registry + `f_register`/dedupe · S1.4 lifecycle-manager (formed/tapped/mitigated/age) · S1.5 gedeelde renderer (box/line/label/CE) · S1.6 decluttering + globaal budget · S1.7 alert-dispatcher + JSON-schema · S1.8 info-paneel-stub |
| **E2 — Imbalance arrays** | Modules 5, 6, 7, 8, 9, 14 | S2.1 FVG + CE (§4.5) · S2.2 IFVG (§4.6) · S2.3 Liquidity Void (§4.7) · S2.4 Volume Imbalance (§4.8) · S2.5 Opening Gaps NDOG/NWOG (§4.9) · S2.6 BPR (§4.14) |
| **E3 — Order-flow blocks** | Modules 1, 2, 3, 4 | S3.1 Order Block + propulsion/reclaimed (§4.1) · S3.2 Breaker (§4.2) · S3.3 Mitigation Block (§4.3) · S3.4 Rejection Block (§4.4) |
| **E4 — Liquidity & Structure** | Modules 10, 11, 16, 18 | S4.1 Buyside/Sellside Liquidity + sweeps (§4.10) · S4.2 Trendline liquidity (§4.10, vereenvoudigd) · S4.3 Equal Highs/Lows (§4.11) · S4.4 Displacement (§4.16) · S4.5 BOS/MSS/CHoCH (§4.18) |
| **E5 — Dealing Range & Levels** | Modules 12, 13, 17 | S5.1 Dealing Range + Premium/Discount/EQ (§4.12) · S5.2 OTE + configureerbare fib-levels (§4.12) · S5.3 Reference Levels PDH/PDL/PWH/PWL/PMH/PML (§4.13) · S5.4 Key Levels & Opens + sessies (§4.17) |
| **E6 — SMT Divergence** | Module 15 | S6.1 SMT met 1 gecorreleerd symbool (§4.15) · S6.2 optioneel 2e symbool + lijn-weergave |
| **E7 — UX polish & alerts** | Afronden info-paneel, alerts, theming | S7.1 info-paneel volledig (§4.19) · S7.2 alerts per module + debounce (§4.20) · S7.3 compact mode + theme-preset (§4.22) · S7.4 performance-caps + lookback (§4.21) · S7.5 tooltips-review alle inputs |
| **E8 — MQL5 port** | 1-op-1 poort | S8.1 skelet + shared core · S8.2–S8.x modules per epic-groep · S8.y compile `0 errors/0 warnings` |
| **E9 — cTrader port** | 1-op-1 poort | S9.1 skelet + shared core · S9.2–S9.x modules · S9.y `dotnet build` tegen `cAlgo.API` |
| **E10 — Verificatie & docs** | Review + gebruikersdoc | S10.1 Pine v6 review-checklist · S10.2 README-gebruik per platform · S10.3 TRADINGVIEW_LISTING.md · S10.4 screenshots/voorbeeld-setups |
| **E11 — MTF-overlay** (later) | Arrays van hogere TF tonen | backlog |
| **E12 — Confluence-scoring** (later) | Unicorn, OTE+OB-stacking, geweegde score | backlog |

**Sessie-scope (nu)**: E1 t/m E7 volledig in Pine (`PDArray.pine`), plus E10.1/E10.2. E8/E9/E11/E12 blijven open.

## 12. Succescriteria

- Alle 18 modules afzonderlijk aan/uit; uit = geen enkel object van die module.
- Nooit meer dan de per-module `Max Shown` arrays van een soort tegelijk zichtbaar, ongeacht historielengte.
- Gemitigeerde arrays verdwijnen/vervagen volgens hun `Mitigated Handling`.
- Geen onverwachte repaint van bevestigde arrays bij het teruglopen van historie.
- Pine-versie volgt de v6-taalregels correct (handmatige review, §9).
- MQL5 compileert `0 errors, 0 warnings`; cTrader bouwt `0 errors, 0 warnings` (E8/E9).

## 13. Toekomstige overwegingen (out of scope v1)

- MTF-overlay per module (E11).
- Confluence-/scoring-engine: Unicorn (Breaker∩FVG), OTE+OB-stacking, geweegde PD-array-score, "A+ setup"-highlight (E12).
- Automatische correlatie-detectie voor SMT.
- Internal/LTF-structuurlaag naast swing-structuur (§4.18).
- Quarterly Theory / Q1–Q4 opening gaps.
- cTrader push/e-mail-alerts via de `Notifications`-API.

## 14. Disclaimer

Bedoeld voor educatieve en analytische doeleinden. Geen enkele indicator garandeert winstgevendheid; grondig testen (in-sample/out-of-sample, walk-forward, Monte Carlo) blijft nodig voordat een strategie hierop live wordt ingezet.
