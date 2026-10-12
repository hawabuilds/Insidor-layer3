# Coin universe: the history import (Part 3.2) — plan only

Nothing in this document has been run. There is no `DUNE_API_KEY` yet, and no history
will be fetched until the founders say so, stage by stage.

The queries are in [`tools/dune/`](../tools/dune). Each is paged by date, so a run can stop
and resume, and each page is a known size.

## Order

| stage | what | query | then |
|---|---|---|---|
| 0 | count every stage, before anything is fetched | `0_counts.sql` | the sizes below stop being guesses |
| 1 | every **graduated** coin since launch | `1_graduated.sql` | metadata read by content id through our gateway, as in 3.1; status `not_checked_yet` |
| 2 | coins that did **not** graduate but reached the market-cap bar, biggest first | `2_above_bar.sql` | as stage 1 |
| 3 | every other coin, **name only** (searchable, `not_checked_yet`) | `3_name_only.sql` | only if asked |

Checking where each coin came from (3.3 and 3.4) is a separate step after the import, and it
is where the real money is. See "After the import" below.

## Size and cost per stage — estimates until stage 0 runs

**Inputs (public figures, not ours):**
- About **11.9 million** launches by June 2026, and about 42,000 a day since then, so roughly
  **15 million** by now.
- Graduation rate **0.8–1.4%**.

| stage | rows (estimate) | Dune result size | our database | metadata reads | Dune cost |
|---|---|---|---|---|---|
| 0 counts | 1 | tiny | — | — | the cheapest query here |
| 1 graduated | **120,000–200,000** | ~50–80 MB over ~20 pages | ~150–250 MB (asset + metadata + indexes) | 120–200k gateway reads: **free**; ~2–4 days at the proposed cap of 50,000/day | small: one join of two decoded tables, ~20 executions |
| 2 above $25k peak | **unknown: guess 300,000–750,000** | ~10–25 MB | ~300–750 MB | free, same gateway; 6–15 days at the cap | **the expensive one**: it scans every trade on the curve (billions of rows), paged by month (~30 executions) |
| 3 name only | **~15,000,000** | ~2.3 GB | **~4–6 GB** | none | large export |

**Dune's dollar cost.** Dune bills each execution in credits, by engine size, plus credits for
exporting the result. I'll turn these into dollars against your plan's rates after stage 0,
which costs a few credits.
- My expectation: stages 0 and 1 fit in a small monthly allowance.
- Stage 2 probably needs a paid plan, or a narrower date range.

**Storage.**
- Stages 1 and 2 fit in a small Postgres.
- **Stage 3 does not fit Supabase's free tier** (500 MB), and needs the Pro plan's 8 GB.
- My recommendation is to skip stage 3 until search over unchecked coins is actually wanted.

## Proposed policy numbers (not yet in `policy.ts`)

| field | proposed | why |
|---|---|---|
| `universe.history.marketCapBarUsd` | 25,000 | a third of the way to graduation; below it, almost nothing has a story |
| `universe.history.metadataPerDay` | 50,000 | a public gateway's comfortable rate; history never competes with live coins |
| `universe.history.pageDays` | 7 (stage 1), 30 (stage 2) | each Dune page a known, bounded size |

## How the import runs (when it does)

- **Batches and resume.** One Dune page at a time. The last finished page is stored, so a
  stopped run resumes there.
- **Metadata reads.** Content id only, through our gateway, with retries and the daily cap.
  This is the same code as live (3.1), so a historical coin and a new one are read identically.
- **Cost log.** Each page records the credits Dune charged and the rows it returned.

## After the import: checking where each coin came from (3.3 and 3.4)

This is not part of 3.2, but it is the number that matters. For stage 1's ~150,000 graduated coins:

| step | rough cost |
|---|---|
| creator posts read by id (~25% link a post) | ~40,000 × $0.00015 ≈ **$6** |
| judge on what is left, batched by 10 | ~20,000 posts ≈ **$14** |
| coin pictures embedded (only coins one of our posts names) | up to 150,000 ≈ **$26** at most |
| **X search back in time**, one per coin, 20 results (3.4.2) | 150,000 × 20 × $0.00015 ≈ **$450** |

The search back is where the money is. Before it runs, it should be limited, for example to
coins whose creator link did not settle them, or to the biggest coins first, and priced again.

## Sources for the inputs

- [Pump.fun 42,000 token launches in 24 hours: the 2% graduation rate (Solana Compass)](https://solanacompass.com/news/pumpfun-launched-42000-tokens-in-one-day-fewer-than-2-will-ever-reach-a-dex)
- [Pump.fun graduation rate: only 0.84% of tokens succeed, April 2026 (pumpfundata.com)](https://pumpfundata.com/blog/pumpfun-graduation-rate-analysis)
- [Longing attention markets — Oregon Blockchain Group's $PUMP report](https://medium.com/oregon-blockchain-group/longing-attention-markets-oregon-blockchain-groups-pump-report-8fcbf736baee)
- [Pump Fun Graduated Tokens (Dune)](https://dune.com/queries/3979196)
