# TradingView listing — ICT PD Array Matrix

Publication copy for the TradingView listing. English.

---

## Title

**ICT PD Array Matrix**

Short title (chart): `PD Arrays`

## Short description (card)

Every ICT Premium/Discount array in one overlay — order blocks, FVG/IFVG, breakers, mitigation & rejection blocks, liquidity, equal highs/lows, dealing range with OTE, reference levels, key opens, SMT, displacement and BOS/MSS — each individually toggleable, on a shared lifecycle that fades mitigated arrays and keeps the chart clean.

## Full description

**One indicator for the entire ICT PD Array Matrix.**

If you trade ICT / Smart Money Concepts you normally end up stacking four or five separate scripts — one for order blocks, one for fair value gaps, one for liquidity, one for market structure — each with its own colours, its own idea of what "mitigated" means, and none of them cleaning up after themselves. After a few hundred bars the chart is unreadable.

**ICT PD Array Matrix** puts every PD array ICT teaches into a single overlay, driven by one shared engine so they look and behave consistently, with built-in decluttering so the chart stays readable on any history length.

### 18 individually toggleable modules

**Order-flow blocks**
- ① Order Block (with a propulsion marker)
- ② Breaker Block
- ③ Mitigation Block
- ④ Rejection Block

**Imbalances**
- ⑤ Fair Value Gap + Consequent Encroachment (BISI / SIBI)
- ⑥ Inversion FVG (IFVG)
- ⑦ Liquidity Void
- ⑧ Volume Imbalance
- ⑨ Opening Gaps — NDOG / NWOG
- ⑭ Balanced Price Range (BPR)

**Liquidity & structure**
- ⑩ Buyside / Sellside Liquidity (old highs & lows, with sweep marking)
- ⑪ Equal Highs / Equal Lows
- ⑯ Displacement — used internally as a quality gate for OB / FVG / MSS, and optionally drawn
- ⑱ Market Structure — BOS / MSS / CHoCH, with the taken swing marked as liquidity

**Dealing range & levels**
- ⑫ Dealing Range — Premium / Discount / Equilibrium + OTE band + custom fib levels (source: auto swing, previous period, or session)
- ⑬ Reference Levels — PDH / PDL, PWH / PWL, PMH / PML
- ⑰ Key Levels & Opens — True Day Open (00:00 NY), 08:30 open, Asia / London / New York session highs & lows

**Confirmation**
- ⑮ SMT Divergence against one or two correlated symbols, with a connector line

Every module has its own enable toggle, bull/bear colours, "max shown" limit and mitigated-handling mode (Fade / Remove / Keep).

### The shared engine — why it stays clean

- **Lifecycle.** Every array moves through *formed → tapped → mitigated*. A filled FVG, a traded-through order block, a swept high — they fade or disappear automatically, according to the rule you set per module. Only arrays that are still in play stay prominent.
- **Decluttering.** A per-module cap on how many arrays are drawn at once (strongest / most recent first), a global object budget, and automatic merging of overlapping same-type zones. Levels far from price keep the line but drop the label, so the right margin never fills with tags.
- **Non-repainting where it can be.** Pivot-based modules confirm after *Pivot Right Bars*; gap and displacement modules confirm on bar close. The confirmation delay is by design, not a bug.
- **Info panel.** Where price sits in the dealing range (premium / discount + % from equilibrium), the nearest un-mitigated array above and below price, the last liquidity sweep, the last structure shift, and the last SMT divergence.
- **Alerts.** Per module (*formed / tapped / mitigated*), plus *liquidity swept*, *market structure shift*, *SMT divergence*, and *price entered the discount OTE zone*. Static alertcondition alerts plus an optional JSON webhook payload.

### How an ICT / SMC trader uses it

1. **Read structure first.** Leave Market Structure and Liquidity on to see where price is being drawn (BSL / SSL) and whether the last break was continuation (BOS) or a change of character (MSS / CHoCH).
2. **Frame the range.** Dealing Range shows premium vs discount and the OTE band. Only look for entries on the correct side of equilibrium.
3. **Pick the PD array.** Inside discount / OTE, look for a fresh Order Block, FVG + CE, Breaker or BPR that lines up with a Reference Level or a session high / low.
4. **Confirm.** SMT Divergence against a correlated pair, and a Displacement leg out of the array.
5. **Manage.** Arrays fade as they get mitigated, so the chart always shows what is still relevant.

### Settings tips

- Fewer modules, or a lower *Max shown*, gives a calmer chart. *Compact Mode* thins borders and drops the CE lines.
- *Lookback Bars* controls how far back detection runs — lower it for performance on long histories.
- Feeder modules keep working when a dependent module is enabled: Breaker still forms with Order Block hidden, and IFVG / BPR still form with Fair Value Gap hidden, so you can test one module at a time.
- For SMT, set *Correlated symbol 1* to a genuinely correlated instrument for your chart (for example AUDUSD or USDCAD when trading AUDCAD).
- Works on any symbol and timeframe. NDOG / NWOG and session levels are hidden automatically on Daily and higher.

### Notes

Original work. All detection, the shared lifecycle, the decluttering system and the info panel are written from scratch in Pine Script v6 — this is not a mash-up of open-source scripts.

This is an analysis and visualization tool. It does not produce buy / sell signals and it does not guarantee any outcome. Test any method built on it thoroughly (in-sample, out-of-sample, forward) before trading it.

## Tags

`ICT` `SMC` `Smart Money Concepts` `Order Block` `Fair Value Gap` `Breaker Block` `Liquidity` `Market Structure` `Premium Discount` `OTE` `SMT Divergence` `PD Array`

## Category

Trend Analysis (or "Chart patterns")
