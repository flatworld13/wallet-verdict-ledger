# wallet-verdict-ledger · the public audit trail

This repo is the **immutable record** of every prediction VERDICT has made
about a Polymarket wallet, plus every change to the methodology that produced
those predictions.

The engine code lives at https://github.com/flatworld13/wallet-verdict.
This repo is **only the audit trail.**

## Three things you can verify in 30 seconds

### 1. A prediction was made before the market resolved

Open `data/audit_attestations.json`, find any `chain_id`, click its
`github_commit_url`. GitHub shows the commit timestamp on its servers — VERDICT
doesn't control it. If `tier == "forward_verified"`, that timestamp is
**before** the Polymarket market's resolution timestamp. Zero peek-risk.

### 2. The methodology changed on a specific date

Open `methodology_versions.md` (or `data/attestation_map.json`). Each version
links to a real GitHub commit. That commit's timestamp is when the rules
in effect on that date were published. We can't ad-hoc-revise without bumping
the version number.

### 3. A wallet's score on a specific date

Browse this repo's history (`git log`) at the relevant date. Each commit is a
snapshot. Read `data/wallets_combined.snapshot.json` at that commit · every
score / Brier / drift / layer assignment is verifiable.

## File index

| Path | What it is | Update cadence |
|---|---|---:|
| `data/audit_ledger.json` | every prediction chain · 425+ rows | daily |
| `data/audit_attestations.json` | per-chain GitHub commit reference · 3 tiers | daily |
| `data/attestation_map.json` | methodology version → GitHub commit | per version bump |
| `data/wallets_combined.snapshot.json` | full per-wallet records · 241 wallets · daily snapshot | daily |
| `data/vpi_nav.json` | **VERDICT Polymarket Index** · today's NAV + composition | daily |
| `data/vpi_nav_history.json` | **VPI** · append-only daily NAV history | daily |
| `data/orderbook_per_trade_snapshots.json` | **Tier 2 transaction history** · 50,000+ per-trade orderbook fills with bid / ask / midpoint / depth at execution moment · keyed by (wallet, condition_id, trade_ts) | per data-engineer PR |
| `data/wallet_microstructure_book.json` | live-snapshotter aggregate · maker/taker rates · slippage rollups | daily |
| `data/venue_rating.json` | Polymarket-the-venue rating (when populated) | quarterly |
| `methodology_versions.md` | human-readable correction history | per version bump |

## Transaction-history reproducibility (M1 + M2 + M3)

Every per-trade orderbook fill in `data/orderbook_per_trade_snapshots.json` is
keyed by `(wallet, condition_id, trade_ts)` — anyone can:

1. Pull the same Goldsky `OrderFilledEvent` stream we used (subgraph endpoint
   recorded inside the file's `subgraph_endpoint` field)
2. Compute `best_bid` / `best_ask` / `midpoint` / `depth_within_1pct_usd` /
   `edge_over_midpoint` at the same `trade_ts`
3. Get the same numbers we publish

If our numbers differ from a reproduction run, the difference is a methodology
bug we'll publish a correction for under `methodology_versions.md`.

## What's NOT here (deliberately)

- The engine code (lives at https://github.com/flatworld13/wallet-verdict)
- Methodology design rationale (also at the engine repo)
- Roadmap, customer briefs, internal analytics

This repo is **append-only.** Branch protection on `main` disables force-push
(operator setup · see github.com/flatworld13/wallet-verdict-ledger/settings/branches).

## How VERDICT keeps this honest

- Every commit message includes the engine repo's commit SHA for cross-reference.
- The daily 02:00 UTC GitHub Actions cron in the engine repo writes here too.
- The act of pushing IS the M1 witness · GitHub's server timestamps the push.

## Reproducibility check

Anyone can re-run our methodology against the public Polymarket Subgraph + this
repo's frozen `wallets_combined.snapshot.json` and reproduce our published
scores. That's M4 (reproducibility-from-clone). The engine repo has the script.

---

_For questions: open an issue at https://github.com/flatworld13/wallet-verdict/issues_
