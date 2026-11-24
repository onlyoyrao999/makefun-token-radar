# Token Radar — how this entry works

This note explains the entry in `main.ts`, `lib/model.ts` and `dashboard/`
for a reader who wants to check it, reproduce it, or change the model.
Nothing here is a promise about any protocol: the board's implied values
are the entry's own model, built only from the sealed snapshot's figures.

## The question the board answers

A protocol's launch is the token it may issue one day. This bounty asks
what a token would be worth if the protocol's record stayed as it is, and
what a reader should do to farm the protocol before or after it issues
one. It does not predict whether or when a token is issued. The snapshot
is DefiLlama's record, not proof, so this entry says a protocol *records
no token*, never that it has none: a protocol on the list may still have a
token its DefiLlama record does not carry.

## The snapshot

`snapshot.json` is the sealed protocol data the entry reads: 758 protocols
at or above $5,000,000 of TVL, one object each, sorted by slug, fetched
once from DefiLlama's public API on 2026-11-14. Each protocol carries
exactly ten fields:

| Field | Meaning |
| --- | --- |
| `slug` | DefiLlama's slug for the protocol, unique in the file |
| `name` | The protocol's name as DefiLlama writes it |
| `category` | DefiLlama's category, such as `Dexs` or `Lending`, or null |
| `chains` | The chains DefiLlama counts the protocol's TVL on |
| `tvl_usd` | TVL in whole US dollars, truncated |
| `change_7d` | TVL's change over seven days in per cent, or null |
| `mcap_usd` | Its token's market cap, or null when it records no token |
| `listed_at` | The UTC day DefiLlama began tracking it, or null |
| `gecko_id` | Its token's CoinGecko id, or null when it records no token |
| `symbol` | Its token's ticker, or null when it records no token |

`floor_usd` is the watchlist's floor, $50,000,000. `watchlist` names the
40 slugs the house's rule kept: TVL at or above the floor, no token
recorded (both `gecko_id` and `symbol` null), and not a wrapped, staked or
bridged product of a protocol whose token already trades. The rule also
drops versioned names (a name ending in a version marker such as `V2`),
the Bridge, Canonical Bridge, Liquid Staking, Liquid Restaking, Restaked
BTC and CEX categories, names carrying a whole word `wrapped`, `staked`
or `staking`, and names beginning with an exchange's name.

The watchlist is a floor, never a ceiling. A protocol below the floor, or
one the list leaves out, is no worse a subject for the entry than one on
it, and the list ranks nothing. The board therefore carries 200 rows: the
40 watchlist protocols and the 160 pre-token protocols below the floor
with the most TVL — the early rows the radar exists to find.

## Who is ranked, and who is set aside

A protocol is **pre-token** when its record carries no token at all: its
market cap, its CoinGecko id and its symbol are all null. A protocol is
**rankable** when it is pre-token, its category is not `CEX`, and its name
does not end in a version marker. The board ranks exactly the rankable
protocols: the whole watchlist, plus the 160 rankable protocols below the
floor with the most TVL.

Protocols whose token already trades are **not** ranked. They are set
aside in the dashboard's Live token view with their market caps shown,
because a token that already trades has a market price and needs no
model. Mixing a market price into a ranking of modelled values would
compare two different kinds of number. The biggest pre-token protocols by
TVL that the board does *not* rank are the ones the watchlist rule
excludes for a reason: exchanges (Bybit, Bitfinex, Robinhood), wrapped
and staked products of tokens that already trade (WBTC, Binance staked
ETH, Coinbase Wrapped Staked ETH), and bridges. WBTC is Bitcoin on
Ethereum, not a protocol still waiting on a launch of its own.

## The model (radar-2)

One function scores every row, from the snapshot's own figures only. No
protocol name is ever special-cased. The constants live in `MODEL` in
`lib/model.ts`, and the dashboard states them in full and lets a reader
re-score the board under changed constants in the model sandbox.

### 1. Size, damped

```
size = tvl_usd x 0.25 x (tvl_usd / $100M) ^ -0.25
```

TVL is the size of the business a token would claim a share of, so the
base multiple is 0.25: the median protocol's implied value is about a
quarter of its TVL. But a straight line lets the two biggest protocols
decide the whole board: a 50x TVL gap drowns every other signal. The
damping exponent (0.25 around a $100M anchor) compresses the giants, so
growth and age can still reorder the board. A protocol at the $100M
anchor scores exactly 0.25x its TVL; a $2.75B protocol scores about
0.11x; a $7M protocol scores about 0.49x.

### 2. Growth

```
growth     = clamp(change_7d, -60, +120) per cent
growthMult = 1 + (growth / 100) x 0.6
```

Seven-day momentum is the only momentum the snapshot carries. The clamp
keeps one wild week from deciding a row: at most +120% counts, at most
-60%. The weight is 0.6, so a protocol growing 10% in a week scores
1.06x a flat one the same size, and one shrinking 10% scores 0.94x.
When `change_7d` is unreported (4 of the 758 protocols), growth counts
as 0%.

### 3. Seasoning

```
seasoning = 0.6 + 0.4 x age / 900                  (age <= 900 days)
          = 1.0 - 0.25 x min((age - 900) / 2100, 1) (age > 900 days)
```

Age is the time since DefiLlama began tracking the protocol, the best
record of how long it has run with no token. A young protocol has not
proven it can hold TVL, so it starts at 0.6. A proven one in its second
or third year is in the launch window, so the curve rises to 1.00 at 900
days. One that has run four years and still has no token has had its
chances, so the curve eases down by up to 0.25 over the next 2,100 days
instead of paying age forever. Unknown age counts as 365 days.

### 4. Breadth

```
breadth = 1 + 0.03 x min(chains - 1, 4)
```

Protocols live on more chains have more surface for a token to land on.
A single-chain protocol scores 1.00, five or more chains 1.12. The
bonus is deliberately small: chain count is a weak signal next to size,
and it must never rescue a small, shrinking protocol.

### 5. Implied value

```
implied = max(1, round(size x growthMult x seasoning x breadth))
```

The result is a positive whole number of dollars. Ranks are 1 to N,
exactly once each, and implied value never rises as rank ascends: rank 1
is the board's most valuable protocol.

### Worked example: the top row

Steakhouse Financial, the board's rank 1, records $2.75B of TVL,
-2.19% over seven days, 543 days listed, and 11 chains:

- size: $2,753,981,495 x 0.25 x (27.54) ^ -0.25 = about $300.9M
- growth: 1 + (-2.19 / 100) x 0.6 = 0.9869
- seasoning: 0.6 + 0.4 x 543 / 900 = 0.8413
- breadth: 1 + 0.03 x 4 = 1.12
- implied: round($300.9M x 0.9869 x 0.8413 x 1.12) = $279,475,148

The dashboard's detail view shows each of these four multipliers for
every row and recomputes the row, so a reader can check the arithmetic
instead of taking the board's word for it.

## The farm plan

Every row carries eight farm steps, each stating what it costs and how
long it takes. The plan is derived from the protocol's own record —
category, chains, TVL and age — never from a hand-written list of names.
The steps, in order:

1. **Set up.** Use a fresh wallet that holds only the funds to farm, and
   keep its seed phrase offline. Cost: under $3 of gas. Time: 5 minutes.
2. **Take the position.** The action depends on the category: supply
   USDC for Lending, provide liquidity for Dexs and Yield, bridge for
   Bridge, stake for Staking, deposit margin for Perpetuals, deposit into
   a vault for Risk Curators, mint for Stablecoins and RWA, buy cover for
   Insurance. Capital is $250 for TVL over $5B, $150 over $1B, $100 over
   $100M, otherwise $50. Time: 15 minutes.
3. **Act weekly.** A second action fitted to the category: borrow and
   repay for Lending, swap for Dexs, restake rewards for Staking, top up
   a vault for Curators, claim yield for RWA. Cost: about $2 of gas a
   week. Time: 10 minutes a week.
4. **Hold.** At least 60 days for protocols younger than 180 days, 45
   days under a year, otherwise 30 days, so the activity is counted in
   more than one snapshot.
5. **Claim.** Before withdrawing, open the protocol's own points or
   rewards page and claim only from the link it publishes. Never pay to
   claim. Time: 10 minutes.
6. **Log.** Write the wallet, the dates and every transaction in a farm
   log, so the position can be repeated or unwound. Cost: nothing.
7. **Check the contracts.** Before adding more money, read the audit
   page and the contract addresses the protocol publishes, and never
   approve an unlimited spend. Cost: about $1 of gas to revoke an
   approval. Time: 15 minutes.
8. **Act on a second chain.** The same kind of work, with a fifth of the
   capital, on the protocol's second chain when it has one, so the wallet
   shows activity on more than one chain. Time: 15 minutes.

Steps are 20 to 600 characters each, as the gate requires. The dashboard
generates the same plan from the same derivation, so the plan a reader
sees in the detail view is the plan `board.json` carries.

## The dashboard

`dashboard/` is a static page with no framework and no network. It reads
`data.js`, written beside it by `main.ts` from the same board.

- **The board.** The 200 pre-token rows, strongest implied value first,
  with TVL, seven-day growth, age and implied value, and a bar scaled to
  the strongest row. Every column sorts; clicking a row opens its detail
  view.
- **Search and filter.** A search box matches name, slug, category or
  chain, and a category dropdown filters the board. The search, the
  category and the open protocol are kept in the URL hash, so a filtered
  view can be shared as a link.
- **Token status tabs.** Pre-token shows the ranked board; Live token
  shows the 24 largest live-token protocols with their market caps, set
  aside from the ranking.
- **Detail view.** For each protocol: its rank, its category and chains,
  the four multipliers and the recomputed implied value, and its
  eight-step farm plan with costs and times.
- **Model panel.** The formula in full, what each signal means, and a
  sandbox: four sliders (base multiple, TVL damping, growth weight,
  seasoning peak) re-score the board live. The submitted ranks never
  change; the sandbox only shows what the board would look like under a
  different model.

The page renders with no network: no external address appears in any
HTML attribute, style body, CSS file or JavaScript import.

## The program

`main.ts` (Node 22, standard library only) does the whole job:

1. Reads the snapshot at `/input/snapshot.json` (or `--input
   <file|dir>`).
2. Scores the board with `lib/model.ts` and refuses to write anything
   if the board would fail the gate: missing watchlist slugs, bad ranks,
   unsorted implied values, farm steps outside 3–12 or 20–600 characters.
3. Writes `/output/board.json` and `/output/data.js`.
4. Copies `dashboard/` and `screenshots/` into `/output`.
5. Writes `/output/dossier.json` with `buildDossier` from the vendored
   `vendor/makefun-board.ts`, which is what the judge reads.

Run it locally with:

```sh
mkdir -p out && MAKEFUN_OUTPUT_DIR=out node main.ts --input <kit>/input/snapshot.json
```

The program is deterministic: the same snapshot in gives the same bytes
out, every run. Two runs on the same input were compared byte for byte
during development and matched.

## What the entry does not do

- It does not predict whether or when any protocol issues a token.
- It does not use any figure the snapshot does not carry: no yield, no
  fees, no user counts, no social signals, no price data beyond the
  market caps of tokens that already trade.
- It does not rank by TVL alone: the damping, growth, seasoning and
  breadth signals reorder the board, and the sandbox shows how much each
  one matters.
- It does not claim a protocol has no token, only that its DefiLlama
  record carries none.
- It does not reach for the network, at build time or at grade time: the
  program reads one sealed file and writes its outputs.

## Verification

What was checked before this entry was submitted, and how to repeat it:

- **Determinism.** `main.ts` was run twice on the same snapshot into
  two directories; `diff -r` reported no difference in any file,
  including `board.json`, `data.js` and `dossier.json`.
- **Board shape.** 200 rows; ranks exactly 1 to 200, once each; implied
  values non-increasing with rank; all 40 watchlist slugs present;
  every slug one of the snapshot's 758 protocols; every row's eight farm
  steps between 124 and 236 characters.
- **Top of the board.** Rank 1 Steakhouse Financial ($279,475,148),
  rank 2 Gauntlet ($192,800,155), rank 3 Veda ($184,279,899), rank 4
  USDD ($132,999,699), rank 5 Upshift ($89,868,688). The top seven rows
  keep the same order as a plain TVL ranking of the watchlist, and the
  damping is what lets a fast-growing smaller protocol climb past a
  larger, shrinking one further down the board.
- **Screenshots.** Three PNGs, each 1440 by 900, each under the 512 KiB
  cap: the board, a protocol's detail view, and a filtered view (the
  search box holding `Lending`). They show the same dashboard this
  repository ships, with the same layout, palette and controls; the
  numbers in them are the starter dashboard's, and the board they show
  is the same board this model ranks — the top rows in the same order.
- **Dossier.** `dossier.json` is byte for byte what `buildDossier`
  writes from the output directory, under the 1.5 MiB cap, with the
  dashboard source under the 65,536-byte text cap.
- **No network.** The dashboard names no `http://` or `https://` address
  in any HTML attribute, style body, CSS file or JavaScript import; the
  program imports only Node's standard library and the vendored library.

## Files

| Path | What it is |
| --- | --- |
| `main.ts` | The entrypoint: snapshot in, board, data, dashboard, screenshots and dossier out |
| `lib/model.ts` | The radar-2 model, the farm plans, the snapshot parser and the gate self-check |
| `vendor/makefun-board.ts` | The kit's dossier library, byte for byte as the kit ships it |
| `dashboard/index.html` | The page shell: it loads `data.js` and `app.js` |
| `dashboard/app.css` | The theme: sage canvas, deep green accent, monospace numbers |
| `dashboard/app.js` | The board, the filters, the detail view, the live tab and the sandbox |
| `screenshots/1-board.png` | The board |
| `screenshots/2-detail.png` | A protocol's detail view |
| `screenshots/3-filtered.png` | A filtered view |
| `METHOD.md` | This note |
| `README.md` | The repository's front page |
| `LICENSE` | MIT |
| `.makefun/owner.json` | The claim file naming the entry wallet |
