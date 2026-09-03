# TradingView listing — ICT PD Array Suite

Concept publicatietekst voor de TradingView-listing. Definitieve versie afstemmen ná de eerste live compile/test.

---

## Titel

ICT PD Array Suite — 18 toggleable Premium/Discount arrays

## Korte omschrijving (±120 tekens)

Alle ICT PD arrays in één overlay: order blocks, FVG/IFVG, breakers, liquidity, dealing range, SMT en market structure — elk los aan/uit.

## Volledige omschrijving

Deze indicator brengt **alle Premium/Discount arrays die ICT onderscheidt** samen in één overlay, met een gedeelde levenscyclus en een begrensd object-budget zodat de chart leesbaar blijft — ook na maanden historie.

**18 afzonderlijk schakelbare modules**

1. Order Block (+ propulsion marker)
2. Breaker Block
3. Mitigation Block
4. Rejection Block
5. Fair Value Gap + Consequent Encroachment (BISI / SIBI)
6. Inversion FVG (IFVG)
7. Liquidity Void
8. Volume Imbalance
9. Opening Gaps — NDOG / NWOG
10. Buyside / Sellside Liquidity (+ sweep-markering)
11. Equal Highs / Equal Lows
12. Dealing Range — Premium / Discount / Equilibrium + OTE + fib-levels
13. Reference Levels — PDH/PDL, PWH/PWL, PMH/PML
14. Balanced Price Range (BPR)
15. SMT Divergence (t.o.v. 1–2 gecorreleerde symbolen)
16. Displacement
17. Key Levels & Opens — True Day Open (00:00 NY), 08:30, sessie-highs/lows
18. Market Structure — BOS / MSS / CHoCH

**Gedeelde eigenschappen**

- **Levenscyclus** — elke array doorloopt formed → tapped → mitigated. Gemitigeerde arrays (gevulde FVG's, doorhandelde order blocks, geveegde highs) vervagen of verdwijnen automatisch, per module instelbaar (Fade / Remove / Keep).
- **Decluttering** — per module een maximum aantal zichtbare arrays; overlappende gelijksoortige arrays smelten samen; globaal object-budget.
- **Non-repainting waar vermijdbaar** — pivot-gebaseerde modules bevestigen na `Pivot Right Bars`; gap-/displacement-modules op bar-close.
- **Info-paneel** — positie in de dealing range, dichtstbijzijnde onmitigated array boven/onder de prijs, laatste liquidity sweep, laatste structuur-event, laatste SMT.
- **Alerts** — formed / tapped / mitigated per module, plus liquidity swept, market structure shift, SMT divergence en discount+OTE entry. Statische `alertcondition` + optionele JSON-webhook-payload.
- **Thema's** — Neutral / Vivid / Mono, plus een Compact Mode voor een rustige chart.

**Gebruik**

Voeg toe als overlay op de hoofdgrafiek. Zet ongewenste modules uit via hun `Enable`-toggle. Voor webhooks: `Enable JSON Webhook Alerts` aan en een alert op "Any alert() function call".

## Categorie / tags

`ICT` `SMC` `Smart Money` `Order Block` `Fair Value Gap` `Liquidity` `Market Structure` `SMT` `Premium Discount` `OTE`

## Disclaimer

Bedoeld voor educatieve en analytische doeleinden. Geen enkele indicator garandeert winstgevendheid; test altijd grondig (in-sample / out-of-sample, walk-forward, Monte Carlo) voordat je een strategie hierop live inzet.
