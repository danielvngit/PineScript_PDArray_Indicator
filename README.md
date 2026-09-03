# ICT PD Array Suite

Eén indicator met **alle PD arrays die ICT onderscheidt** — 18 afzonderlijk schakelbare modules — met een gedeelde levenscyclus (formed → tapped → mitigated → opgeruimd), een begrensd object-budget zodat de chart leesbaar blijft, consistente kleuren/labels en een compact info-paneel. Zie [`PRD.md`](PRD.md) voor de volledige architectuur en requirements.

Uiteindelijk **drie implementaties** met identieke logica:

| Bestand | Platform | Status |
|---|---|---|
| [`PDArray.pine`](PDArray.pine) | TradingView (Pine Script v6) | Bron-implementatie — in aanbouw (epics E1–E7) |
| [`PDArray.mq5`](PDArray.mq5) | MetaTrader 5 (MQL5) | Aparte epic E8 — nog niet gestart |
| [`PDArray.cs`](PDArray.cs) | cTrader (cAlgo C#) | Aparte epic E9 — nog niet gestart |

## De 18 modules

| # | Module | Groep |
|---|---|---|
| 1 | Order Block (+ propulsion / reclaimed) | Order-flow blocks |
| 2 | Breaker Block | Order-flow blocks |
| 3 | Mitigation Block | Order-flow blocks |
| 4 | Rejection Block | Order-flow blocks |
| 5 | Fair Value Gap + Consequent Encroachment (BISI/SIBI) | Imbalance |
| 6 | Inversion FVG (IFVG) | Imbalance |
| 7 | Liquidity Void | Imbalance |
| 8 | Volume Imbalance | Imbalance |
| 9 | Opening Gaps (NDOG / NWOG) | Imbalance |
| 10 | Buyside / Sellside Liquidity (+ sweeps, trendline liq.) | Liquidity & structure |
| 11 | Equal Highs / Equal Lows | Liquidity & structure |
| 12 | Dealing Range — Premium / Discount / Equilibrium + OTE/Fib | Dealing range & levels |
| 13 | Reference Levels — PDH/PDL, PWH/PWL, PMH/PML | Dealing range & levels |
| 14 | Balanced Price Range (BPR) | Imbalance |
| 15 | SMT Divergence (correlated pair) | SMT |
| 16 | Displacement | Liquidity & structure |
| 17 | Key Levels & Opens (Midnight/True Day Open, 08:30, sessies) | Dealing range & levels |
| 18 | Market Structure — BOS / MSS / CHoCH | Liquidity & structure |

Elke module heeft: `Enable`, eigen bull/bear-kleur, `Max Shown`, `Mitigated Handling` (Fade / Remove / Keep), en module-specifieke drempels — allemaal met tooltip.

## Gedeelde eigenschappen

- **Levenscyclus**: gemitigeerde arrays (gevulde FVG's, doorhandelde OB's, geveegde highs) vervagen of verdwijnen automatisch.
- **Decluttering**: per module een instelbaar maximum aantal zichtbare arrays; globaal object-budget; merge van overlappende same-type arrays.
- **Non-repainting waar vermijdbaar**: pivot-modules bevestigen na `Pivot Right Bars`; gap-/displacement-modules op bar-close.
- **Info-paneel**: positie in de dealing range, dichtstbijzijnde onmitigated array boven/onder de prijs, laatste sweep, laatste structuur-event.
- **Alerts**: per module (formed / tapped / mitigated) + structuur / sweep / SMT / OTE-entry; statische `alertcondition` + optionele JSON-webhook-payload.

## Gebruik (Pine / TradingView)

1. Kopieer de inhoud van `PDArray.pine` in de Pine Editor van TradingView en klik **Add to chart** (overlay op de hoofdgrafiek — geen apart paneel).
2. Zet ongewenste modules uit via hun `① … ⑱ Enable`-toggle. `Enable All Modules` bovenaan is de noodrem.
3. Stel per module `Max shown` en `Mitigated handling` (Fade / Remove / Keep) af.
4. `Global / Theme`: `Compact Mode` voor een rustiger chart, `Color Theme` (Neutral / Vivid / Mono), `Lookback Bars` voor performance.
5. Webhook-automatisering: zet `Enable JSON Webhook Alerts` aan en maak een TradingView-alert op **"Any alert() function call"**. Payload-schema: zie [`PRD.md` §4.20](PRD.md).
6. Losse alerts (zonder webhook): maak een alert op een van de zeven `alertcondition`-namen (PD Array — formed / tapped / mitigated, Liquidity swept, Market structure shift, SMT divergence, Discount + OTE entry).

### Standaard-preset

**Alle 18 modules staan standaard aan.** Om het overzichtelijk te houden zijn de per-module `Max shown`, de veroudering en de `Lookback` (400) bewust laag gezet, en worden level-labels ver van de prijs verborgen (de lijn blijft). Rustiger maken: `Compact Mode` aan, `Lookback` lager, of modules uitzetten.

Let op: ⑮ SMT vergelijkt standaard met `FX:AUDUSD` — zet dit op een correct gecorreleerd symbool voor jouw instrument (of zet de module uit) anders krijg je zinloze SMT-markeringen.

### Aanbevolen instellingen (AUDCAD H4 / H1)

| Instelling | H4 | H1 |
|---|---|---|
| Pivot Left / Right Bars | 5 / 5 | 6 / 6 |
| ATR Length | 14 | 14 |
| Lookback Bars | 600 | 800 |
| FVG min gap (x ATR) | 0.15 | 0.20 |
| Displacement (x ATR) | 1.5 | 1.3 |
| SMT symbol 1 / 2 | `FX:AUDUSD` / `FX:USDCAD` | idem |

Meer modules aan of hogere `Max shown` maakt de chart drukker; `Compact Mode` en een lagere `Lookback` maken hem rustiger.

### Voorbeeld-presets

- **Alles aan** — verkennen; zet `Compact Mode` aan en verlaag `Max shown` per module naar 3–4.
- **Alleen imbalance** — modules ⑤ ⑥ ⑦ ⑧ ⑨ ⑭ aan, rest uit. FVG + CE + BPR voor entdifferent-model trading.
- **Structuur & liquidity** — modules ⑩ ⑪ ⑯ ⑱ aan. BOS/MSS + buyside/sellside + displacement.
- **Dealing range + OTE** — modules ⑫ ⑬ ⑰ aan. Premium/discount context met OTE-band en referentielevels.

## Gebruik (MQL5 / cTrader)

De MQL5- en cTrader-ports volgen als aparte epics (E8 / E9). Zie [`PRD.md` §6](PRD.md) voor de geplande platformverschillen.

## Ontwikkelstatus

Zie [`PRD.md` §11](PRD.md#11-agile--epics--backlog) voor de epic-/issue-indeling en [§9](PRD.md#9-verificatiestatus) voor de verificatiestatus per platform.

## Disclaimer

Bedoeld voor educatieve en analytische doeleinden. Geen enkele indicator garandeert winstgevendheid; test altijd grondig voordat je een strategie live inzet.
