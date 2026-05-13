# VERDICT methodology · correction history

Auto-generated from `engine/server.py` `methodology_versions` list at sync time.
Each version has a real GitHub commit URL · timestamp witnessed by GitHub's server.

_Last synced: 2026-05-13 09:40 UTC_

## Why we publish a correction history

Most rating products publish their CURRENT weights but never their
correction history. Mnemom · TipRanks · PA Beacon · HolyPoly · Polycopy ·
do not publish *"we used to score this differently and here's why we changed it."*

VERDICT does. Every methodology change ships with a versioned diff + evidence trail.
S&P paid $58M+ for not having this in 2008-era credit ratings. We ship it free.

---

## v2.3 · LAYER_0_ALPHA tier · 9th axis · Murphy-Winkler decomposition · execution quality

- **Date**: 2026-05-08
- **Shipped**: yes
- **GitHub-witnessed at commit**: [`90a3935c`](https://github.com/flatworld13/wallet-verdict/commit/90a3935c334b01fd90f9808457258af3a8d78c55)

### Short
Adds a NEW top tier (LAYER_0_ALPHA) gated on edge_over_midpoint · adds 9th decision-tree axis · adds Murphy-Winkler (REL/RES/UNC) decomposition as published per-wallet output · adds execution_quality block from orderbook microstructure

### Why this version
After v2.2 stabilized at 8 decision-tree criteria, three independent gaps remained: (1) Brier alone conflates predictive skill with execution skill — a wallet that buys at 0.30 when fair midpoint was 0.45 is more skilled than the same wallet buying at 0.30 when fair was 0.30. v2.2 couldn't distinguish them. (2) Single-number Brier is the A1 anti-pattern (closed-formula composite) that competitors get killed for · academic baseline requires the Murphy 1973 + Winkler-Murphy 1968 decomposition into REL (calibration) − RES (resolution) + UNC (uncertainty). (3) The 8 v2.2 criteria all measure WHAT the wallet predicted · none measure HOW they got there (maker vs taker · slippage · order book imbalance at entry · MM rebate-farmer bot patterns). v2.3 closes all three gaps WITHOUT removing or weakening any v2.2 criterion · the 8 are still the floor · v2.3 ADDS strict layers on top. LAYER_0_ALPHA is the new top tier · only wallets that pass all 8 v2.2 criteria AND show positive average edge over midpoint (with ≥30% snapshot coverage) qualify. Today: 0 wallets · the dormant tier auto-grows as the 02:00 UTC orderbook snapshotter accumulates coverage.

### Diff sketch
```
+ LAYER_0_ALPHA tier added as new top of cohort · gated on:
+   execution_quality.coverage_pct >= 0.30
+   execution_quality.mean_edge_over_midpoint > 0
+   execution_quality.edge_consistency_pct >= 0.55
+ 9th customer-filter axis: min_edge_over_midpoint (range -10pp to +10pp · default 0)
+ Murphy-Winkler decomposition published per-wallet: REL · RES · UNC · BS_check · base_rate
+ Execution quality block published per-wallet: maker_pct · taker_pct · realized_slippage_cents · OBI · is_likely_mm_rebate_farmer · coverage_pct
+ THRESHOLDS_V22 dict extended with min_edge_over_midpoint · min_edge_consistency_pct · min_orderbook_coverage_pct (currently 0.0 · 0.55 · 0.30)
~ /api/wallets/filter counts dict adds LAYER_0_ALPHA: 0 slot
~ /wallets status strip adds ★ ALPHA pill (gold gradient · top of cohort)
~ /wallet/<addr> renders new "Execution quality · orderbook microstructure" section + "Murphy-Winkler decomposition" section
= Decision tree v2.2 8 core criteria PRESERVED · this is additive · existing LAYER_0_STRICT etc still classify identically when execution_quality is absent
= Backward-compat verified: classifier(rec) == classifier(rec, THRESHOLDS_V22)
```

### Evidence trail
- **spec MD** · `_handoff/2026-05-08_orderbook_TIER1_RUN_NOW.md` · Tier 1 brief I wrote · data engineer delivered live-snapshotter framework in commit 9e58595
- **engine code** · `_ingest/compute_murphy_winkler.py` · Murphy 1973 + Winkler-Murphy 1968 decomposition · 171/239 wallets computed
- **engine code** · `_ingest/integrate_orderbook.py` · merges wallet_microstructure_book.json into wallets_expanded.json per-wallet
- **engine code** · `_tools/order_book_snapshotter.py + _tools/join_book_to_trades.py` · data engineer · live CLOB snapshotter · 60s polling · Maker/taker + OBI + slippage attribution
- **classifier** · `engine/server.py _classify_skill_layer · LAYER_0_ALPHA branch` · gated on execution_quality presence · backward-compat preserved
- **data** · `data/wallet_microstructure_book.json` · 239 wallets · 0 attributed today · framework deployed · coverage grows nightly

### Measured impact
241 wallets · LAYER_0_ALPHA = 0 today (dormant · gates on coverage_pct ≥ 30%) · will populate to ~0-3 wallets after 1-2 weeks of nightly snapshotter operation. Other layer counts UNCHANGED from v2.2 (additive, not modifying). Same 8 criteria still apply at the LAYER_0_STRICT / LAYER_1_RELAXED / LAYER_2_LOOSE / LAYER_3_ALL gates. M3 vocabulary now extends to "edge over midpoint" + "OBI" + "maker/taker" · institutional language lock-in. M5 wedge strengthened: published correction history now spans v1 → v1.1 → v2 → v2.1 → v2.2 → v2.3 with full diff per row + evidence trail.

---

## v2.2 · Decision tree v2.2 · MM/arb-bot exclusion (timing-based)

- **Date**: 2026-05-08
- **Shipped**: yes
- **GitHub-witnessed at commit**: [`0003ca10`](https://github.com/flatworld13/wallet-verdict/commit/0003ca104fb14761385fe5622bf011464b57fdf5)

### Short
2 new filter criteria (two_sided_share · pct_sub_1min) added to decision tree · catches market-makers and arb-bots that pass extreme_share filter

### Why this version
Hold-out validation on 76 wallets revealed counter-intuitive ρ patterns: Layer 3 wallets had BETTER hold-out Brier (0.019) than Layer 1 (0.165) and POSITIVE hold-out PnL (+0.06 vs -0.12 for Layers 1/2). Manual examination of the 3 LAYER_1 wallets revealed 2 of 3 had two_sided_share = 78-88% — they were market-makers / arb-bots trading both sides on the same market, NOT genuinely diverse traders. The original decision tree's extreme_share filter missed these structural-edge plays. Added two timing-based filters: two_sided_share <= 0.50 (strict) / 0.65 (relaxed) · pct_sub_1min <= 0.50. Wallets with timing data missing default to PASS (don't reject for missing data). Layer 3 'mm_arb_pattern' reason added for wallets with two_sided > 65%.

### Diff sketch
```
- decision tree v2.0 had 6 criteria · missed MM/arb-bots with low extreme_share
+ decision tree v2.2 has 8 criteria · 2 new timing-based filters
+ two_sided_share <= 0.50 (Layer 0 strict) · <= 0.65 (Layer 1 relaxed)
+ pct_sub_1min <= 0.50 (filters ultra-fast trade timing typical of bots)
+ Layer 3 reason_code "mm_arb_pattern" for wallets with two_sided > 65%
+ Hold-out validation: Spearman ρ = -0.54 (Brier · n=76) · ρ = -0.14 (PnL · n=76)
+ Discrimination check: Layer 3 PnL = +0.06 vs Layers 1/2 PnL = -0.12 · Δ = -0.18
```

### Evidence trail
- **analytics MD** · `_validation/2026-05-07_phase0_skill_check_results.md` · original skill-check on 96 wallets · 1-of-10 strict fail finding
- **analytics MD** · `_validation/2026-05-08_observed_rho_holdout_validation.md` · this version · hold-out ρ + manual LAYER_1 examination · MM/arb discovery
- **data** · `data/wallets_expanded.json (timing_features)` · two_sided_share + pct_sub_1min · 96/96 wallets · enrichment from data engine

### Measured impact
96 wallets re-classified · expected Layer 1 to drop from 3 → 1 (only fengdubiying retained · 2 MM/arb-bots reclassified to Layer 3 with reason_code=mm_arb_pattern)

---

## v2.1 · Continuous magnitude scoring (Lever 2)

- **Date**: 2026-05-07
- **Shipped**: yes
- **GitHub-witnessed at commit**: [`af73a108`](https://github.com/flatworld13/wallet-verdict/commit/af73a108c7f2072826ebac251662c81fa55b2109)

### Short
Replaced binary engine_was_right with continuous right_magnitude · A→F PnL/$ spread = +0.833

### Why this version
Operator audit raised: '55% accuracy isn't enough to justify a recommendation.' True for binary right/wrong (3pp COPY-vs-AVOID spread is structural noise floor on backfill). But the same audit data shows a much sharper continuous signal: A-grade chains average -0.024 PnL/$ (close to break-even), F-grade chains average +0.810 PnL/$ (saved). That 0.833 spread is the real signal binary scoring was hiding. Added `right_magnitude` per chain row (clipped to [-1,+5] for COPY · [-5,+1] for AVOID) and a headline strip on `/audit` showing the spread. We do NOT remove the binary `engine_was_right` field · backward compatibility preserved.

### Diff sketch
```
- engine_was_right: bool       (loses sign + magnitude)
+ engine_was_right: bool       (kept · backward compatible)
+ right_magnitude: float       (PnL/$ relative to baseline · clipped per side)
+ /audit headline now publishes A→F magnitude spread (+0.833) alongside binary accuracy
+ marketing copy: never claim a single accuracy %; always cite the spread
```

### Evidence trail
- **commit** · `bbf423a · Lever 2 · continuous right_magnitude` · A→F discrimination spread = +0.833 PnL/$ · A=-0.024, F=+0.810
- **analytics MD** · `2026-05-07_score_research_industry_benchmark.md` · §5 binary-accuracy is structurally weak · §6 critique of single-Brier
- **analytics MD** · `2026-05-07_v2.1_methodology_lever2_receipt.md` · M5 correction receipt · published-at timestamp this row

### Measured impact
404 chains · A=-0.024 PnL/$ · B=+0.234 · C=+0.394 · D=+0.578 · F=+0.810 · spread A→F = +0.833.

---

## v2 · 4-layer weakest-link composite

- **Date**: 2026-05-07
- **Shipped**: yes
- **GitHub-witnessed at commit**: [`89e17723`](https://github.com/flatworld13/wallet-verdict/commit/89e1772396a2f614af1e02b5687bfa9246151077)

### Short
Replaced 6-signal weighted average with min(L1, L2, L3, L4) per M3 chain rule

### Why this version
Convexly Mar 2026 study (n=10K wallets) found r=+0.148 between calibration-only score and profit. Le 2026 (arxiv 2602.19520, n=292M trades) confirmed 87.3% of calibration variance is along horizon × domain × scale. Single-Brier was structurally insufficient. 4-layer weakest-link composite gives 4 independent discrimination axes AND forces honesty (a wallet can't hide a weak axis under a heavy weight).

### Diff sketch
```
- 35% Brier · 20% OOS · 15% domain · 15% portfolio · 10% drawdown · 5% integrity
- weighted average · masks weakness
+ L1 Outcome (50% ROI/$, 50% win rate · capped at 75 if n<5)
+ L2 Calibration (volume-weighted Brier across qualifying domains, n≥30 max-domain for full)
+ L3 Domain (best-domain with overgeneralization cap if only 1 qualifies)
+ L4 Portfolio (30% diversification · 25% drift · 25% copyability · 20% tail-risk)
+ Composite = min(L1, L2, L3, L4) — M3 weakest-link rule
```

### Evidence trail
- **analytics MD** · `2026-05-07_score_research_industry_benchmark.md` · industry benchmark · academic citations · S01-S06 strategic rows
- **product spec** · `../product/verdict_trader_skill_scoring_framework.md` · original 4-layer framework spec
- **commit** · `4675a6c · S01 · 4-layer skill score` · Code lands on every wallet master record

### Measured impact
97 wallets: 5 B · 2 C · 35 D · 54 F (post-rule). Mean L1=42, L2=71, L3=72, L4=75, composite=37.

---

## v1.1 · A-grade audit-ledger fix · H12

- **Date**: 2026-05-07
- **Shipped**: yes
- **GitHub-witnessed at commit**: [`89e17723`](https://github.com/flatworld13/wallet-verdict/commit/89e1772396a2f614af1e02b5687bfa9246151077)

### Short
Side-aware entry-score rule + n≥30 sample floor · headline 63%→79%

### Why this version
Pre-fix entry_score rewarded `avg_buy_price ≤ 0.50` regardless of side. Combined with selection-biased ultra-low domain Brier (lots of correct No bets on cheap longshots), Yes-side longshots at 0.001 inherited A grade and lost 100%. 86 of 400 backfilled chains were A-graded · only 9/86 (10%) were right. All 77 wrong A-rows were Yes-side longshots. Bug surfaced in the deepened n=400 dataset (was n=7 noise in v1).

### Diff sketch
```
- entry_score = 90 if avg <= 0.50 else 70 if avg <= 0.85 else …
+ side-aware: yes-side <0.20 → 25 (F) · yes-side <0.40 → 50 (D)
+                     favorite-side ≤0.50 → 90 (A eligible)
+ A grade requires cb_n >= 30 in domain (sample-size floor)
```

### Evidence trail
- **analytics MD** · `2026-05-06_dataset_deepening_overnight.md` · paste-ready fix sketch · H12 evidence
- **commit** · `7658988 · H12 fix` · Headline 63% → 79% (+16pp). 77 broken Yes-longshots now F-graded.

### Measured impact
404 chains: A (86 · 10% right) → A (17 · 35% right) + F (63 · 96% right). Headline 63 → 79.

---

## v1 · v1 weighted formula (deprecated)

- **Date**: 2026-05-06
- **Shipped**: superseded
- **GitHub-witnessed at commit**: [`89e17723`](https://github.com/flatworld13/wallet-verdict/commit/89e1772396a2f614af1e02b5687bfa9246151077)

### Short
Original Part-11 6-signal weighted average · superseded by v2

### Why this version
First-pass methodology imported from `verdict_strategic_evolution_framework_may4.md` Part 11. Used a 6-component weighted average. Worked as a v1 baseline but masked weakest-axis risk (a high-Brier wallet with awful portfolio still scored well). Convexly + Le 2026 findings invalidated single-Brier dominance. Replaced by v2 weakest-link composite.

### Diff sketch
```
+ 35% Brier · 20% OOS sharpness · 15% domain · 15% portfolio · 10% drawdown · 5% integrity
  weighted average · all components contribute proportionally
```

### Evidence trail
- **product spec** · `../strategy/verdict_strategic_evolution_framework_may4.md` · Part 11 · the original v1 formula spec

### Measured impact
No measurement available · v1 was wired before audit ledger had n>=400.

---
