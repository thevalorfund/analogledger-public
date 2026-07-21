# Analog Ledger - Public Data

This repository holds the free, public data behind [Analog Ledger](https://analogledger.com/), a quantitative equity research project. Every file here is written by an automated weekly pipeline - nobody edits these files by hand, and every commit is timestamped by GitHub, so you can independently verify when each piece of data was actually published.

If you just want to browse the research, the website is a much easier way to do it. This repo exists so anyone can check the website's numbers against the raw source.

## Why some data looks "locked"

Analog Ledger tracks how its weekly stock candidates perform over a 12-week window before counting the result. To prove the candidates aren't being adjusted after the fact, each week's candidate list is **encrypted and committed immediately** when the scan runs - before anyone knows how it'll turn out. The decryption key is published here automatically once the 12-week window closes.

That means for any week younger than 12 weeks, you'll see scrambled text instead of tickers. That's expected - it's the whole point. Once the week matures, the key shows up in the same file and the real list becomes readable.

Subscribers see each week's candidates immediately, with no wait - they get access through the website itself, backed by a separate [private repository](https://github.com/thevalorfund/analogledger-data) that never redacts anything. That repo isn't browsable on its own; it's only reachable by subscribing at [analogledger.com/subscribe](https://analogledger.com/subscribe). This encryption scheme exists so that free readers aren't just asked to take it on faith that subscribers are seeing the same untouched data - the ciphertext committed here on day one is mathematically provable to unlock into exactly what subscribers already saw, once the key is published. Nobody, including us, can quietly change a candidate list after the fact and have it go unnoticed.

## If we get something wrong

Nothing in this repo is edited after the fact - including to fix a real mistake. If an error is ever found in already-published data, the original file stays exactly as committed. We publish a separate, clearly-labeled correction alongside it instead, with a plain explanation of what was wrong - never a quiet edit to the original.

The entire point of this archive is that nothing can be rewritten after the fact once it's public. That has to be true even when the mistake is ours.

## What's in this repo

### `index.json`
A summary of everything else - useful as a starting point.

| Field | Meaning |
|---|---|
| `available_weeks` | Every scan date that's been published |
| `latest_week` | The most recent scan date |
| `available_outcomes` | Which weeks have finished their 12-week tracking window |
| `current_model_version` | The ranking model currently in use (see `model_history.json`) |
| `candidate_counts` | How many candidates were ranked each week |
| `avg_peak_return_pct` | Average best-case return across all matured candidates |
| `hit_rate_100_pct` | % of matured candidates that at least doubled at some point |
| `total_candidates_tracked` | Total candidates ever tracked |

### `model_history.json`
The version history of the ranking model itself - when each version was promoted and how it scored in backtesting (`lift_p5`/`p10`/`p20` = how much better the top 5/10/20% ranked candidates did versus a random sample).

### `weeks/YYYY-MM-DD.json`
One file per weekly scan. This is the actual list of candidates the model ranked that week.

**A matured week** (12+ weeks old) looks like this:

```json
{
  "scan_date": "2024-01-01",
  "model_version": "v1",
  "published_at": "2026-07-07T16:31:28Z",
  "candidate_count": 21,
  "candidates": [
    { "rank": 1, "ticker": "TAOP", "close": 38.70, "profile_tier": "A", "profile_historical_hit_rate_pct": 43.5 }
  ],
  "candidates_ciphertext": "tDMNy4RboFXb3p4ryYv5FD8p...",
  "candidates_nonce": "jhiSf6QAL/TOchv6",
  "reveal_key": "6yZrRNcrhoDgBM7nkvwGauD5isyV1DK9RQ4PL5+LbpM="
}
```

- `candidates` - the ranked list, one entry per candidate (`rank`, `ticker`, `close` price at scan time, `profile_tier`, and the pattern's historical hit rate).
- `candidates_ciphertext` / `candidates_nonce` - the original encrypted version of that same list, committed the day the scan ran. Left in place permanently as the proof record.
- `reveal_key` - the AES-256-GCM key that unlocks the ciphertext. Only appears once the week matures.

**A week still inside its 12-week window** looks the same, minus the plaintext:

```json
{
  "scan_date": "2026-06-29",
  "model_version": "v8",
  "published_at": "2026-07-07T16:31:30Z",
  "candidate_count": 109,
  "candidates": null,
  "candidates_ciphertext": "6t5M1jFFvPAN8syxkNQfa6BF...",
  "candidates_nonce": "pEH/iNHM5qjmMpGD"
}
```

`candidates` is `null` and there's no `reveal_key` yet - the tickers are genuinely hidden. What's committed on this date can't be changed afterward without it being obvious: the ciphertext you see today is the same ciphertext that'll still be here in 12 weeks, and the key that unlocks it will decrypt to exactly this file's `candidate_count` of candidates, no more, no less.

**To verify a matured week yourself:** the `candidates_nonce` and `reveal_key` are both base64-encoded. Decrypt `candidates_ciphertext` with AES-256-GCM using that key and nonce, and the result should match the `candidates` array exactly, byte for byte.

### `outcomes/YYYY-MM-DD.json`
Written once, automatically, exactly 12 weeks after each scan - how every candidate from that week actually performed.

```json
{
  "scan_date": "2024-01-01",
  "matured_at": "2026-07-02T20:07:03Z",
  "weeks_elapsed": 130.4,
  "results": [
    { "rank": 1, "ticker": "TAOP", "peak_return_pct": 53.5, "hit_100": false }
  ]
}
```

`peak_return_pct` is the highest gain the ticker reached at any point during the 12-week window. `hit_100` is true if it at least doubled.

### `portfolio_simulation.json`
A backtested simulation of what would've happened if someone actually traded every weekly candidate list, run separately for each model version ("era"). Not a real track record - a hypothetical, spelled out in the file's own `disclaimer` field.

```json
{
  "configuration": {
    "n_per_week": 3, "dollars_per_position": 500,
    "stop_loss_pct": 50, "take_profit_pct": 150,
    "scale_out_pct": 1.0, "starting_capital": 15000
  },
  "eras": [
    {
      "model_version": "v8",
      "weeks": 26,
      "date_range": ["2026-01-05", "2026-06-29"],
      "final_equity": 28000.0,
      "return_pct": 86.7,
      "win_rate_pct": 61.1,
      "equity_curve": [15000, "..."],
      "weekly_detail": [
        {
          "date": "2026-01-05", "value": 15000, "cash": 13500, "deployed": 1000,
          "candidate_era": true,
          "transactions": [
            { "type": "open", "ticker": "BIVI", "exit_date": null },
            { "type": "close", "ticker": "RVSN", "entry_date": "2024-01-15", "exit_type": "+150% target", "return_pct": 150.0, "dollar_pnl": 750 }
          ]
        }
      ]
    }
  ]
}
```

`equity_curve` and portfolio value/cash/deployed are always shown live, week by week - those numbers alone can't identify any specific ticker. But `weekly_detail[].transactions` for a week still inside its 12-week window is redacted the same way `weeks/*.json` is:

```json
{ "date": "2026-06-29", "value": 28000, "cash": 7000, "deployed": 0, "candidate_era": true, "transactions": [], "locked": true }
```

## The full picture

This public repo and its private counterpart are what actually render [the website](https://analogledger.com/). Subscribers see the same data early, before the 12-week reveal, through a separate private repo - everyone else, including this repo's readers, sees it exactly when the lock lifts and not a moment sooner.
