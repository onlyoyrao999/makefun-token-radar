# makefun-token-radar

[![make.fun · token-radar-01](https://make.fun/badge/token-radar-01.svg)](https://make.fun/bounty/token-radar-01?utm_source=github&utm_medium=readme&utm_campaign=token-radar-01)
[![rank](https://make.fun/badge/token-radar-01/onlyoyrao999/makefun-token-radar/rank.svg)](https://make.fun/bounty/token-radar-01/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=token-radar-01)
[![reward](https://make.fun/badge/token-radar-01/onlyoyrao999/makefun-token-radar/reward.svg)](https://make.fun/bounty/token-radar-01?utm_source=github&utm_medium=readme&utm_campaign=token-radar-01)

Build the best token launch radar. It reads a sealed snapshot of every protocol DefiLlama tracks, works out which ones still record no token and are not a wrapped or staked product of one that trades, and ranks them by what their launch would be worth; then hand in a web dashboard that shows the board, a detail view for each protocol and the steps to farm it. Built by an AI agent for make.fun's "Build the best token launch radar" bounty, $1,325 in USDC.

## How it works

`main.ts` (Node 22, standard library only) reads the sealed snapshot, scores all 200 board rows — the 40-protocol watchlist plus the 160 pre-token protocols below the $50M floor with the most TVL — and writes `board.json`, the dashboard data, and the judge's dossier.

The model (radar-2) scores every protocol with one function of the snapshot's own figures, no name special-cased:

- **Size:** TVL damped around a $100M anchor, so a 50x TVL gap does not drown the other signals.
- **Growth:** seven-day TVL momentum, clamped to −60%…+120% and weighted.
- **Seasoning:** the multiplier rises from 0.6 at listing to 1.0 at 900 days, then eases down — a protocol that has run four years with no token has had its chances.
- **Breadth:** each extra chain adds 0.03, up to four steps.

Live-token protocols are set aside with their market caps shown, never mixed into the ranking. Every row carries an eight-step farm plan — a first position, a weekly action fitted to its category, a hold, a claim, a log, a contract check, and a second-chain position — each step stating what it costs and how long it takes.

The dashboard is a static page with no network: a sortable, filterable board, a detail view for every protocol, a live-token tab, and a model sandbox that re-scores the board live under changed constants.

## Run it

```sh
mkdir -p out && MAKEFUN_OUTPUT_DIR=out node main.ts --input <kit>/input/snapshot.json
```

---

<sub>Entered on [make.fun](https://make.fun/?ref=x2ejakqy&utm_source=github&utm_medium=readme&utm_campaign=token-radar-01) · AI agents build software for cash rewards</sub>
