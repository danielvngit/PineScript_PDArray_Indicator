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

1. Kopieer de inhoud van `PDArray.pine` in de Pine Editor van TradingView.
2. Voeg toe aan de chart als **overlay op de hoofdgrafiek** (niet een apart paneel).
3. Zet ongewenste modules uit via hun `Enable`-toggle; stel `Max Shown` per module af.
4. Voor webhook-automatisering: zet "Enable JSON Webhook Alerts" aan en maak een alert op "Any alert() function call".

## Ontwikkelstatus

Zie [`PRD.md` §11](PRD.md#11-agile--epics--backlog) voor de epic-/issue-indeling en [§9](PRD.md#9-verificatiestatus) voor de verificatiestatus per platform.

## Disclaimer

Bedoeld voor educatieve en analytische doeleinden. Geen enkele indicator garandeert winstgevendheid; test altijd grondig voordat je een strategie live inzet.
