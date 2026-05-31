# Pressure Firelog — Public Hash Ledger

This repository is an **append-only** public ledger of SHA-256 hashes for trading signals fired by [Pressure](https://pressure-app.com) algorithms.

Currently logging:

| Algo | Asset | Timeframe |
|------|-------|-----------|
| VENT | BTCUSD | 4H |
| FLOW | ETHUSD | 4H |
| SURGE | SOLUSD | 4H |

## How it works

1. Pressure's signal engine — a TypeScript application running on a dedicated server in Frankfurt — polls `api.binance.com` every ~10 seconds for newly-closed 4-hour bars (closes at 00, 04, 08, 12, 16, 20 UTC)
2. As soon as a bar closes, the engine evaluates VENT, FLOW, and SURGE against it. If a strategy fires an entry signal, its details (direction, price, timestamp, cryptographic nonce) are SHA-256 hashed and sealed within ~10 seconds of bar close
3. The hash is committed to this repo **before the trade closes** — proving the signal existed at that time
4. When the trade exits, the original data is revealed on [pressure-app.com/firelog](https://pressure-app.com/firelog)
5. Anyone can recompute the hash from the revealed data and verify it matches

## Repository layout

```
hashes/                          ← signal hashes, one log file per day
  2026/05/11.log
  2026/05/12.log
  ...
manifests/                       ← source-code fingerprints, one file per algo
  vent-v1.json
  flow-v1.json
  surge-v1.json
retired/                         ← source code of retired algorithm versions
  ...
README.md
```

### Hash log format

Log files contain two kinds of lines.

**Entry lines** — fired when a strategy opens a new position. The signal payload is hashed (with a server-side nonce) and committed BEFORE the trade closes:

    <sha256-hash>  <signal-id>  <algo-id>  <asset>  <commit-timestamp>

**Exit lines** — fired when a position closes. The exit details are public (visible on /firelog), so no hash is needed — the GitHub commit timestamp itself is what makes the exit values immutable:

    EXIT  <signal-id>  <algo-id>  <asset>  exit_price=<X>  exit_reason=<Y>  realized_r=<R>  <commit-timestamp>

Each entry and exit lands in the daily log file keyed by its own timestamp (entries by entry day, exits by exit day — they may be in different files for trades spanning a day boundary). The `algo-id` field identifies which algorithm produced the signal (e.g. `vent-v1`, `flow-v1`, `surge-v1`).

### Source manifest format

Each `manifests/<algo-id>.json` records the SHA-256 fingerprint of the **TypeScript engine** — the code that actually fires live signals — plus a history of every version that has ever run in production:

```json
{
  "algo_id": "vent-v1",
  "schema_version": "1",
  "sources": {
    "typescript_files": [
      "engine/src/types.ts",
      "engine/src/strategies/flow.ts",
      "engine/src/indicators/sma.ts",
      "engine/src/indicators/ema.ts",
      "engine/src/indicators/atr.ts",
      "engine/src/indicators/donchian.ts",
      "engine/src/indicators/percentile.ts"
    ],
    "typescript_canonical_format": "Files concatenated in listed order with '\\n--FILE--\\n<path>\\n--CONTENT--\\n<body>' delimiter, then SHA-256.",
    "typescript_role": "Production code on VPS. This is what fires live signals.",
    "pine_specification_file": "fire/pressure/algos/vent/strategy.pine",
    "pine_specification_role": "Canonical strategy specification. Used for walk-forward backtest in TradingView. NOT live production code."
  },
  "current": {
    "effective_from": "2026-05-20T05:56:21.969Z",
    "ts_sha256": "39efc5b88d2985af...",
    "ts_byte_length": 28253,
    "main_repo_commit": "..."
  },
  "history": [
    {
      "effective_from": "2026-05-19T06:54:11.961Z",
      "effective_until": "2026-05-20T05:56:21.969Z",
      "pine_sha256": "...",
      "ts_sha256": "..."
    }
  ]
}
```

**Two roles, one production fingerprint.** The same engine code runs all three algorithms with per-algo parameters (in `types.ts`), so `ts_sha256` is identical across the three manifests for any given engine version. The Pine Script source for each algorithm lives in the main repo and is preserved as the canonical strategy specification — it's the code used for walk-forward backtesting and is the reference description of what the strategy does. **Pine does not fire live signals** (the TypeScript engine does).

## Source code access

The source code is proprietary. Verification of the algorithm itself works through two channels:

**Public hash chain (anyone, anytime):** Read the public hash chain in this repo. Verify that signal hashes were committed BEFORE their outcomes resolved. The GitHub commit timestamp is independent (attested by GitHub, not by Pressure). This proves signals weren't backdated or curve-fitted to known outcomes — the primary fairness claim.

**Open release at retirement:** When an algorithm version is retired, the full source of that version is released publicly to this repo at `retired/<algo-id>/`. Anyone can then re-hash the source and confirm it matches the historical `ts_sha256` in `manifests/<algo-id>.json`. Past signals from that version become fully open to re-verification.

## Verification

### Verifying a signal hash

Visit [pressure-app.com/firelog](https://pressure-app.com/firelog), switch to **Live**, and click the link icon next to any hash to see its commit in this repo. The GitHub-attested commit timestamp is the independent proof that the signal was published before its outcome was known.

### Verifying an algorithm wasn't tampered with

Every signal carries an `algo_id` and a `ts_iso` (entry timestamp). The manifest commits cryptographically to the engine code that fired the signal:

1. Open the matching `manifests/<algo-id>.json` in this repo
2. Find the entry whose `effective_from ≤ ts_iso < effective_until` — that's the engine version active when your signal fired (use `current` if `effective_until` is null)
3. The `ts_sha256` is an immutable, GitHub-attested record of the exact TypeScript engine code that produced the signal

The hash *itself* is the commitment — because this repo is append-only and branch-protected, the fingerprint cannot be changed retroactively. When the algorithm version is eventually retired (see "Open release at retirement" above), the source is released publicly and the hash can be re-verified directly. Until then, the append-only manifest is the cryptographic record of what the production code actually was at that point in time.

If we ever change a strategy's logic or its parameters, the manifest gets a new entry committed to this repo with a new `effective_from` timestamp. Because this repo is **append-only and branch-protected** (no force pushes, no deletions, no admin override), we cannot retroactively backdate the change. Any code modification leaves a permanent, time-stamped trail.

## Why prices are hidden until close (commit-reveal)

Live signals on /firelog show only the SHA-256 hash and the GitHub commit timestamp — not the entry price. This is by design. The hash commits Pressure cryptographically to a specific entry payload (with a server-side nonce, so it cannot be brute-forced from public price data) **before** the trade outcome is known. The full payload — including entry price, direction, and stop — is revealed when the trade closes (or in real-time to paying subscribers via Discord/webhook).

This pattern is "commit-reveal": commit publicly to a hashed value, reveal the value later. It preserves the provably-fair guarantee (anyone can verify, after close, that the revealed payload hashes to the published hash) while preventing free-riders from copying live signals without a subscription. Paying subscribers get prices in real-time; everyone gets verifiability after the fact.

## Trust model

- This repo is **append-only and GitHub-enforced** — branch protection on `main` blocks force pushes and deletions for everyone including the repo owner ([see ruleset](https://github.com/pressureapp/pressure-firelog/rules))
- GitHub's commit timestamps are attested by GitHub's servers, not by Pressure
- The hash includes a server-side cryptographic nonce, so the commitment cannot be brute-forced from public price data
- The TypeScript engine's source code is fingerprinted via SHA-256 (see `manifests/`); fingerprint changes are themselves append-only commits, making mid-flight engine edits impossible to hide
- The signal engine runs on a fixed, monitored server with verified network access to `api.binance.com` (the same data source TradingView uses for `BINANCE:` symbols)
