# Review of the working build

What was actually built, what it proves, what is broken, and what to carry into the rebuild.

Five reviewers went through the repository, the live database, and the deployed site. A sixth
pass attacked their strongest claims and overturned several of them. Where that happened, the
corrected position is what appears below, and the overturned version is named so you can see
what changed.

Every code claim carries a `file:line`. Every data claim names the query or the file it came
from. Where a number is uncertain, it says so.

---

## 1. The verdict in one page

**Yes, this is a good foundation to learn from. No, it is not a foundation to build on.**

The developer built a real six-stage pipeline that runs on a schedule, accounts for its own
spending to the cent, records why it rejects things, and survives being killed mid-run by a
serverless timeout. She also built something our documents never asked for and never
specified: a blind experimental harness for testing whether our product thesis is true. That
harness is the single most valuable thing in the repository, and it is worth more than the
pipeline it was built alongside.

**What works.** The X post rail on the live site — real media, real handles, real metrics, real
deep links, with a well-built queue so a page reload does not flood the screen with "new"
posts that are hours old. The token detail page — DexScreener pool embed, Jupiter quotes, live
trades and holders and safety, all fed by real API routes. The 57,032-row view-count time
series, which cannot be bought and cannot be backfilled. The cost accounting, which is
measured from real API usage blocks rather than estimated. The near-miss table, which
remembers the 8,101 posts the system rejected — the only negative training corpus you will
ever get for free.

**What is broken.** The site shows one row out of 1,289 narratives, for reasons explained in
section 3. The clustering does not cluster: 91% of narratives contain exactly one post by one
author, because what the code calls "embeddings" is word-overlap counting. The coin matcher
identifies coins by symbol string alone, with no time check, and has already linked a "gym
day" post to $PUMP ($1.84B, 375 days old) and "stefan back on the grass" to $GRASS ($225M, 633
days old) — while a working Jupiter buy path sits one click away. The Create Coin button
fabricates a Solana address in the browser using a random number generator and tells the user
their coin is live. Six user-facing surfaces leak internal scoring, one of which names our
model vendor and our daily spend in a banner.

**The single most important thing to know.** *The evidence in this repository does not test our
core hypothesis, in either direction.* The backtest that appears to refute three of our feed
features and confirm three others does neither — it was run on the wrong population, with
features read off a surface contaminated by the outcome, at a sample size where the answer was
always going to be a coin flip. The live meme scores that appear to prove ingest is surfacing
nothing coinable were produced by a prompt the developer has already replaced; the current
prompt in the working tree would score several of those rejected posts as passes. Almost every
confident conclusion available from this build dissolves when you check what produced the
number.

That is not a failure. It is what a first build is for. But it means the rebuild should inherit
the *instruments* — the outcome labeller, the blind protocol, the snapshot series, the cost
meter — and inherit almost none of the *conclusions*.

---

## 2. What the backtest proves

### The headline, and why it is wrong

The merged reveal report (`backtest/backtest-merged-reveal-report.md`) shows 866 rows, 433
winners and 433 losers, with seven hand-labelled features. Read at face value it says:

| Feature | Winner − loser | Reads as |
|---|---|---|
| ticker_in_replies | **+25.7 pts** | strong signal |
| crypto_noticed | **+21.3 pts** | strong signal |
| others_copying_it | **+16.9 pts** | strong signal |
| funny_not_serious | +15.2 pts | mild signal |
| works_as_photo | −1.9 pts | nothing |
| has_character | −3.4 pts | nothing |
| one_word_name | −8.0 pts | negative |

The three at the bottom are the three our feed design rests on. `has_character` and
`one_word_name` are not abstractions — they are near-verbatim the live scoring gate
(`worker/adapters/anthropic/meme-score.js:22`, "a named or nameable entity — an animal,
creature, character, object, or person"; and `:57`, "If you cannot name the coinable subject in
one word that would work as a ticker, score below 0.3").

**Do not act on this table.** Here is what the adversarial pass found.

### Finding 1: nothing in the table is statistically significant

Fisher exact tests, two-sided, recomputed independently from
`backtest/backtest-merged-enriched.csv` using the same parser as
`worker/backtest/reveal-merged.js:33-40,78-88`. Both reviewers reproduced these to three
decimals:

| Feature | Winners | Losers | p | 95% CI on the difference |
|---|---|---|---|---|
| crypto_noticed | 27/36 | 22/41 | 0.061 | −0.2 to +40.0 |
| ticker_in_replies | 9/17 | 6/22 | 0.184 | −4.6 to +50.9 |
| others_copying_it | 21/36 | 17/41 | 0.173 | −5.3 to +36.8 |
| funny_not_serious | 16/36 | 12/41 | 0.235 | −6.1 to +35.0 |
| one_word_name | 11/17 | 16/22 | 0.730 | −35.4 to +19.5 |
| has_character | 26/36 | 31/41 | 0.798 | −22.7 to +15.7 |
| works_as_photo | 29/36 | 33/40 | 1.000 | −19.8 to +15.5 |

Every confidence interval spans zero, and every one of them also spans +10 points. The
labelled sample is 77 rows out of 866 — 8.9% — because labels were only filled where a human
manual search actually found an origin post. Power to detect the observed `crypto_noticed`
effect at these sample sizes is roughly 45%. Power to detect a generic 10-point effect is
14–19%.

That cuts both ways, and the symmetry is the point. The negative results on our three feed
features do not refute them. The positive results on the other three do not support them
either. The study is uninformative about all seven.

### Finding 2: two of the four biggest effects are half-sample, because a spreadsheet ate them

`one_word_name` and `ticker_in_replies` have zero labels from wave 2 — only 39 rows each, all
from wave 3. The cause is visible in the file. `backtest/filled-backtest2-blind-sheet.csv:1`
came back from Google Sheets with a mangled header:

```
row_id,ticker,me,description,...,funny_t_serious (Y/N),others_copying (Y/N),crypto_ticed (Y/N),tes
```

A case-sensitive strip of the literal string "no" ran over the entire file. `name` → `me`,
`notes` → `tes`, `not_serious` → `t_serious`, `noticed` → `ticed`. It hit data cells too
("Toly The Claynosaur" → "Claysaur"). The `one_word` and `ticker_in_replies` columns were
deleted outright for all 436 wave-2 rows. `worker/backtest/merge-blind-datasets.js:38-58` adds
aliases that recover the mangled survivors but cannot recover the two deleted columns.

So the biggest headline number in the study, +25.7 points, rests on 39 rows from a single wave
with no replication possible.

### Finding 3: the three "winning" features are read off the outcome

`crypto_noticed`, `ticker_in_replies` and `others_copying_it` were judged by a human opening
the origin post on X and reading its replies — months after the coin launched. A coin that 10x'd
gets shilled in its origin post's replies for months afterward. "Crypto noticed it" and
"someone proposed a ticker in the replies" are near-mechanical consequences of the outcome.

The developer already flagged this class of problem herself. Line 2 of
`backtest/backtest-merged-enriched.csv` reads: *CONTAMINATED: views_now, replies_now,
reposts_now are post-launch — not early signals*. And `replies_now` is the only variable in the
entire dataset that reaches significance — winners median 807 replies versus losers 186,
Mann-Whitney p=0.034, AUC 0.66. It is the same surface the human read the three judgements
from.

The three features that appear to work are the most likely artefacts in the study, not the most
likely findings.

### Finding 4: the population is wrong, so no sample size would fix it

Every row in this study is a coin that already graduated to PumpSwap
(`worker/backtest/build-case-control-blind.js:51-53`,
`worker/backtest/lib/graduated-discovery.js:50-56`, and
`worker/backtest/purge-non-graduated.js:23-28` deletes anything that did not). Graduations run
about 107/day against roughly 30,000 mints/day — 0.36%.

The outcome is post-graduation price: winner = peak ≥10x **and** all-time-high ≥$250k; loser =
peak <1.5x (`build-case-control-blind.js:55-66`).

Our gate decides something *upstream* of all of that. The prompt itself says so at
`worker/adapters/anthropic/meme-score.js:17-19`: it scores "whether people would actually
deploy a memecoin from this content… You are NOT predicting whether a coin would succeed,
moon". Inside a sample where the gate's own target has already happened — the coin was deployed
*and* graduated — content features that decide whether a post gets coined at all are
range-restricted to near zero by construction.

This is the decisive point, and it is worth being blunt about: *citing low statistical power
implies more rows would settle the question. They would not.* The null is a selection artefact.
Labelling another 2,000 graduated coins produces the same nothing.

### Finding 5: blindness was procedural, and leaky

The blind/answer file split is real and well done (`build-case-control-blind.js:32-38` writes
the blind headers without the `group` column; outcomes go to a separate answers file with a
do-not-open banner on line 1). But the sheet handed to the labeller carried the pump.fun URL
and the raw mint address (`worker/backtest/export-blind3-sheet.js:14`). One click shows the
coin's full price history, and a ≥10x/$250k winner is unmistakable next to a <1.5x loser.

I found no positive evidence of peeking — origins were actually found slightly more often among
losers (42 vs 37), the opposite of what motivated peeking would produce. But the design cannot
prove its own blindness, and it did not have to be that way.

### Finding 6: the results do not replicate across waves

Recomputed per wave: `works_as_photo` was −15.8 points in wave 2 and +10.9 in wave 3.
`has_character` was 0.0 then −8.0. `others_copying_it` was +5.3 then +28.3. Only
`crypto_noticed` held steady. And wave 1's timing result is the exact reverse of the merged
one: `backtest/backtest-reveal-report.md` reports winners at 19.4h post-to-launch versus losers
at 5.7h, while `backtest/backtest-merged-reveal-report.md` reports 0.4h versus 2.3h.

### What actually survives

Two things.

**The outcome label.** `worker/backtest/dune-peak-multiples.sql` is the best single artefact in
the repository. It takes the graduation price from the first PumpSwap/Raydium trade at or after
migration with `amount_usd >= 10` (lines 78-82), sidestepping bonding-curve price artefacts,
and counts trades within 90% of max price as a wash-trade guard. The classes separate cleanly:
winners median 18x peak and $902k ATH, losers 1.19x and $42k, with only 5 of 433 losers
clearing the winner ATH bar. I also cross-checked every merged row's group assignment against
the Dune checkpoints: 830 of 841 resolvable rows carry the correct group, zero duplicate mints.
The plumbing is sound. The problems are all upstream in what was measured.

**The timing direction.** Restricting `hours_post_to_launch` to a sane [0,72]h range — 16 of 62
rows fall outside it, including 10 where the "origin" post was published *after* the coin
existed — winners' origin posts precede the mint by a median 0.70h versus 5.48h for losers.
Still n=19/27, still not significant. But it is the only direction in the study that points the
same way as our independent measurement in `docs/analysis/research/07-virality-and-detection.md`
and that could actually be captured before the mint.

### Does the core hypothesis survive?

**It is not tested. Say that, and do not soften it in either direction.**

The honest statement: a real Stage 0 was built and run, and it was run against the wrong
population with post-hoc features on 8.9% of its own sample. Our feed design is neither
vindicated nor refuted. The features that look strong are the ones most likely to be measuring
the outcome. The features that look dead are the ones a graduated-coin sample is structurally
incapable of testing.

**What would settle it, and it is half-built already.** `worker/backtest/lib/post-search.js`
with `launchSearchWindow()` (`worker/lib/x-official-adapter.js:330-342`) pins `end_time` to the
mint timestamp so nothing after the mint is visible.
`worker/backtest/lib/features.js:60-120` already computes views-at-T, velocity, acceleration
and ticker-proposal counts from snapshots. It was never used for waves 2 and 3 because X's
recent-search endpoint only reaches back 7 days (`x-official-adapter.js:328`) and the waves used
30- and 60-day windows — which is exactly why she fell back to manual labelling.

So run it forward. Sample coins as they mint, freeze the pre-mint state, wait 30 days, apply
`dune-peak-multiples.sql`. Blindness becomes structural because the outcome does not exist yet.
Contamination becomes impossible. `ticker_in_replies` becomes a legitimately pre-mint feature.
And the denominator sits where the product's decision actually is. Cost is not the constraint —
reads are $0.005 against a $10 guard, and 62 of 79 enrichments came free from the syndication
endpoint. Calendar time is the constraint, which is an argument for starting it this week.

---

## 3. Why the site shows one row

### The gate condition

The display gate is one pure function, `deriveGateReason` at
`worker/cluster/lib/cluster-engine.js:291-339`. It checks in fixed order: too old → below views
→ low meme score → no velocity, and returns the first failing reason as a string.

Across all 1,289 narratives, the reasons break down as:

```
too_old      1,225
low_meme        38
below_views     15
(null)           7
no_velocity      3
eligible         1
```

Query: `narratives?select=gate_reason` with `count=exact`, live Supabase project
`layazmzgbrusnspjhiep`.

**The 1,288 number is mostly a graveyard, not a calibration problem.** 1,225 of them aged out.
Age is checked first (`cluster-engine.js:307-313`) and short-circuits everything below it.
`MAX_NARRATIVE_AGE_MIN` defaults to 720 (12 hours).

### The actual cause: ingest is dead and everything else kept running

```
last_run_ingest    2026-08-05T19:10:05Z
last_run_snapshot  2026-08-06T02:18:23Z
last_run_score     2026-08-06T02:18:31Z
last_run_cluster   2026-08-06T02:18:46Z
```

Query: `worker_pipeline_state?select=last_run_*`. Zero X API calls have been made since
19:12 UTC — there is no `source='x'` row in `worker_usage` for 2026-08-06 at all.

So for seven hours the cluster stage kept re-evaluating a frozen set of posts, watching them
age past 12 hours, and marking them `too_old`. The score stage made zero calls because
`loadCandidatePosts` only looks back 3 hours (`worker/score/score.js:40`). The cluster stage has
no such guard and kept spending — today's entire Anthropic bill is 147 title calls re-titling
stale narratives that will never surface.

The failure is silent by construction. `api/cron/_lib/run-stage.js:56` calls `recordLastRun`
only inside the `try` block; the catch at `:60-67` returns a 500 and writes nothing to the
database. A stage that dies looks exactly like a stage that ran and found nothing.

### The narrower question: of the rows that are NOT too old, what is stopping them?

Of the 39 narratives touched by the last cluster run: 29 `low_meme`, 6 `too_old`, 3
`below_views`, 1 eligible. The 29 rejected scores are 0.10, 0.12, 0.15 and 0.25 against a
threshold of 0.70 (`worker/score/lib/meme-gate.js:4`). Their titles are "Vinicius Jr Contract
Talks Positive", "Stefon Diggs Signs With Commanders", "Newcastle Appoints Matthias Jaissle".

**One reviewer concluded from this that the gate is correct and ingest is genuinely surfacing
nothing coinable. The adversarial pass overturned that, and the correction matters.**

Every meme score on the live board was written by a superseded judge. The newest score in
`post_meme_scores` is timestamped 2026-08-05T19:12Z. The commit that introduced the current
`COINABILITY_CORE` prompt (54ee0a3) landed 2026-08-06T02:10Z. 191 of the 228 scores from Aug 5
contain the phrase "organic viral moment" — vocabulary from the previous prompt, which the
current one never uses and explicitly disclaims at `meme-score.js:19`.

And the current prompt would score those posts differently. `meme-score.js:26-29` states: "A
named, recognisable PERSON with a fanbase is a coinable subject — K-pop idols, athletes,
streamers… Score these PASS (0.7+)" and "Do NOT downgrade because the post describes something
they did." The live board gave VINICIUS 0.15 for "sports transfer news, not an organic viral
moment", AESPA 0.10 for "official behind-the-scenes promo", RUTSCHMAN 0.25 for "named athlete
in a sports trade news story". Those are exactly the cases the current prompt names as passes.

Corroborating the regime break: on 23–25 July, 8–12% of scored posts cleared 0.7. On 4–5
August, 3 of 438 did — 0.7%. A tenfold collapse in pass rate that tracks a prompt change, not a
change in what ingest pulls.

### The fix, in order

1. **Find out why the ingest cron dies while its siblings survive, and make the failure loud.**
   `run-stage.js` must persist run outcome including errors, and something must alarm on
   `last_run_ingest` age. This is the actual outage.
2. **Redeploy the current prompt and rescore.** Roughly 40 Haiku calls settles whether the gate
   is over-tight before anyone touches thresholds. Do this before any ingest redesign is
   scoped.
3. **Only then look at ingest.** The ingest query genuinely contains no topic or coinability
   filter — `worker/ingest/lib/ingest-query.js:11` is
   `min_faves:${floor} lang:en -filter:replies -filter:retweets ${timeOps}`, i.e. "the most-liked
   English tweets in the last 30 minutes". That is verified and it holds for all three X lanes.
   But note the correction: the claim that this "is why sports news dominates" was never
   measured — no topical composition of the ingest stream exists anywhere in the repo. The real
   consequence is one stage later. Nothing narrows topic before the paid LLM call
   (`score.js:67-82` filters only on platform, id and freshness; ranking is pure views velocity
   at `:53-57`), so the ten paid slots per cycle (`SCORE_TOP_N`, `score.js:41`) are allocated by
   engagement alone. That is recall loss and wasted spend at the bottleneck, which is a sharper
   problem than sports on the board.

---

## 4. What works and should be kept

### Where she solved something our documents did not

**The blind/reveal experimental harness.** Our documents specified a testing strategy for code
and said nothing about how to evaluate a predictive claim without fooling ourselves. She
invented the missing discipline: outcome labels written to a physically separate CSV with a
do-not-open warning on line 1, features hand-labelled, the join happening only in a separate
reveal step, a seeded shuffle so every wave reproduces from seed 42, and wave 3 excluding wave
2's mints by mint address so cross-wave replication could be checked at all. The report even
self-labels "Descriptive only — not statistically significant" and flags cells at n<5. The
protocol has a real hole (the sheet linked to the live coin page) and the results turned out to
be uninformative — but the instrument is right, and it is the reason we know the results are
unstable.

**The anti-self-clustering rule.** `worker/cluster/lib/cluster-engine.js:98` — a post never
merges into a narrative on keyword overlap or embedding similarity if that narrative already
contains a post from the same author. Only a shared cashtag can do it. This stops one prolific
account manufacturing an apparent narrative. Our documents gestured at "organic score" and
never said how. Keep the rule regardless of what the clustering becomes.

**The near-miss table.** `worker/ingest/lib/near-miss.js` parks sub-threshold posts with their
platform ID and re-checks them *by ID* — a cheap batch lookup, not a re-search — for 12 hours,
promoting them if they cross the threshold. 8,101 rows live, with the complete raw X payload.
Our design had no memory of what it rejected. This is both the only mechanism that can catch a
post on the way up, and the negative-label training corpus any future classifier needs.

**Schema constants that throw.** `lib/db-schema.js:328-361` — `t()`, `c()`, `cs()` throw at call
time on an unknown table or column rather than letting Postgres reject the write. This is our
"row types are generated, never hand-written" rule achieved in plain JavaScript with no build
step. (It has drifted; see section 5.)

**Per-feature READMEs with a "what breaks here" section.** `site/features/{feed,token,wallet}/README.md`
— nine lines each, symptom-first so a bug report routes itself, plus which tables it reads and
which APIs it calls. Make this mandatory per module.

**TikTok scored by video frame, not caption.** `meme-score.js:60-70` — "the coinable subject
usually lives in the VIDEO — judge from what you SEE in the image, not hashtag spam in the
caption", with `:113-140` fetching the thumbnail and base64-encoding it because the TikTok CDN
blocks Anthropic's URL fetcher. Our documents treated posts as text. She found empirically that
they are not, and worked around a vendor limitation to fix it.

**The self-authored defect register.** `docs/schema.md:337-353` lists schema defects with IDs,
evidence and file:line. She found her own bugs and wrote them down. Three of them (M2, M4, M11)
were independently confirmed here. Carry the register forward as the rebuild's punch list.

### Code and data worth porting

| Item | Why |
|---|---|
| `worker/backtest/dune-peak-multiples.sql` | The outcome labeller. Clean class separation, wash-trade guard, correct decoded table. Lift wholesale. |
| `post_snapshots` (57,032 rows) | 2,410 of 2,411 posts have ≥1 snapshot, 1,821 have ≥2, median 9, max 135. Zero non-monotonic series across 994 checked. Cannot be backfilled — migrate the rows, not just the code. |
| `ingest_near_miss` (8,101 rows) | Full raw API payloads on rejected posts. Same argument. |
| `worker/ingest/lib/near-miss.js` | Park-and-recheck-by-ID. |
| `deriveGateReason`, `cluster-engine.js:291-339` | Pure function returning a named reason string, persisted to the row. This is how I could tell in one query that ingest death, not scoring, empties the site. |
| `api/cron/_lib/run-stage.js` + `worker/lib/time-guard.js` | Resumable checkpointing so a 60-second serverless function can stop mid-pass and resume. Non-obvious work; orthogonal to every detection decision. |
| `worker/adapters/anthropic/pricing.js` + `budget.js` | Correct Haiku 4.5 rates, per-call-type breakdown, measured averages replacing estimates, per-call spend gate. |
| `worker/lib/cycle-log.js` | Every cycle logs its counts *and* the thresholds it used. Small discipline, outsized payoff. |
| `worker/score/lib/nameability.js` | `applyNameabilityCap` is deterministic post-processing that fires on 18% of rows. Encodes the product rule in code, not prose. Model-independent. |
| `worker/score/lib/tt-score-gate.js` | Requires recency AND a views floor AND one of three signals before spending a vision call. Result: 0 calls, $0.00, in eight days. This is the shape the naming path is missing. |
| `worker/cluster/lib/replication.js` | Measures replication by live entity search rather than by cluster membership — correctly decoupled from whether the clusterer worked. Currently starved of budget (5 searches/cycle across 29+ narratives). |
| `site/features/feed/feed.js:346-475` | The table reconciler: `Map<id, element>`, JSON display-state key to skip untouched rows, per-cell patching with directional tick, order reconciliation without teardown, scroll preservation. Exactly the "values update live, positions do not" machinery the design specifies. Port it. |
| `site/features/feed/feed.js:873-991` | The announce queue: dedupe set, 1,800ms drain gap, first-seen-vs-page-open gate. Without this a reload floods the rail with fake "new" cards. |
| `site/features/token/tokenpage.js` (all 169 lines) | Clean IIFE, one export surface, no globals. `pairAddressOf` at `:30-36` refuses to embed the mint URL because it resolves to the wrong pair — a real bug someone hit and fixed correctly. The one file written the way the rebuild should be written. |
| `site/styles/index.css` | Custom properties, tabular numerals, `.up`/`.down` semantics, grid templates with explicit minimum tracks so columns drop rather than squash. |
| `worker/adapters/anthropic/` as a directory boundary | All three LLM call sites behind one directory. Nothing else in the codebase talks HTTP to a model. This is what makes swapping the judge cheap. |

---

## 5. What is broken, ordered by what it costs to leave

### 5.1 The Create Coin flow fabricates a contract address and tells the user their coin is live

`site/index.html:672-708`. `deployFiled()` calls `genCA()` — a linear-congruential
pseudo-random generator at `index.html:291` — to produce a Solana-looking address, pushes a
synthetic token into the in-memory array, and renders "$TICK is live" with a **Copy CA** button.
There is no create or mint endpoint anywhere in `api/`. The modal also states invented
economics: 3.0% creator fee, "Insidor takes 1%", "~1.02 SOL to launch"
(`index.html:643-648`).

A user who clicks Create Coin is handed a fabricated address and told to copy it. Delete the
flow entirely. A disabled button is honest; this is not.

### 5.2 The coin matcher can put a Buy button on the wrong token

This is the highest-consequence defect in the system, because the buy path is real.

**How matching works.** `lib/token-lookup.js:138-155` searches DexScreener by symbol string.
`pickBestPair` (`:57-67`) filters to an exact symbol match and picks the highest-liquidity pair.
`worker/cluster/lib/enrich-tickers.js:9-19` sets `first_deployed=true` whenever a pair is found
with liquidity > 0. **There is no check that the coin was created after the post.** There is no
confidence score. There is no liquidity or age floor.

**What it has already produced.** 328 of 1,025 ticker rows are marked `first_deployed=true`.
The largest matches by market cap are all wrong:

- "gym day" → **$PUMP**, $1.84B market cap, 375 days old
- "enhypen such a special night" → ENHY → **Jupiter**, $624M, 710 days old
- "stefan back on the grass" → **$GRASS**, $225M, 633 days old
- "breaking 85% of supply" → **$XRP**

Systematically: of the 323 tickers that resolved with liquidity > 0, **189 (59%) share zero
content words** between the DexScreener coin name and the narrative's title, blurb and post
text. `$FRED` from "Freddie Woodman signs Liverpool contract" resolved to "Fred the Raccoon".
`$GOAT` from a Messi/Ronaldo banter tweet resolved to Goatseus Maximus at $13.2M. `$RATE` from a
White House ratepayer pledge resolved to "MicroStrategy xStock" at $40M.

**Why it can reach a user.** `site/features/feed/feed.js:138-155` renders a live market-cap cell
for any token with a mint, liquidity and market cap; `api/swap.js` is a real Jupiter swap signed
by a real Privy wallet.

**Why the schema cannot express doubt.** `worker/schema.sql:44-46` — `canonical` and
`first_deployed` are `boolean not null default false`. There is no verdict, no confidence, no
source column. A boolean cannot abstain. An LLM-invented ticker and a real observed cashtag are
indistinguishable in the row.

**Fix:** match confidence must be a first-class column; the coin must postdate the post; and the
UI must render no button when unsure. Identity should come from the mint resolved at the launch
event, never from symbol equality.

### 5.3 Provenance is inverted — the hallucinated ticker outranks the real one

`worker/cluster/lib/cluster-engine.js:240-274` puts real cashtags scraped from post text (line
244) and LLM-suggested tickers (line 255) into **one map with no field separating them**. Real
cashtags are created with `memeScore: 0` (line 251). LLM inventions inherit the post's meme
score (line 265). Line 271 sorts descending by meme score and line 272 marks the top entry
`canonical: true`.

An invented ticker therefore *always* beats a real cashtag for the canonical slot.

Line 266 then writes the post author's handle into `endorsed_by`. 937 of 1,025 ticker rows carry
an `endorsed_by` handle, but 868 come from narratives whose posts contain no cashtag at all —
the account never mentioned any ticker. Live values include `@FCBarcelona`,
`@BleacherReport`, `@NHLBlackhawks`.

Attributing a coin endorsement to a real named account that never made one is a defamation and
market-manipulation exposure, not a data-quality nit. Adding a provenance column alone would not
fix this, because the ranking would still surface the hallucination first. The rebuild needs two
separate concepts: an **observed cashtag** (evidence, has a mint) and a **proposed name**
(hypothesis, has no mint), never in the same list and never sharing a canonical flag.

### 5.4 Internal scoring leaks to user-facing surfaces

Ordered worst first.

**(1) A shipped banner names our vendor, our internal score, and our spend.**
`site/index.html:241-244` plus `site/live.js:1646-1664`, wired to Supabase Realtime on
`worker_pipeline_state` at `live.js:1719-1728`:

> "Meme scoring paused — Anthropic budget exhausted. New viral posts will not appear until
> scoring resumes."
>
> "Daily scoring budget reached ($4.10 / $6.00). Feed will not refresh with new scored posts
> until UTC midnight or budget is raised."

This violates the rule three ways at once. If the feed is stale, say the feed is stale.

**(2) The narrative modal's second paragraph is a metric dump.** `narrTiming()` at
`feed.js:219-236`, rendered at `:736`: "221K combined views · 477K gained in 24h · 1.3K
views/min · no coin deployed yet." The other branches are worse — "Coin already up ~8x since
the trend broke — exit-liquidity risk", "mcap still low relative to narrative heat", "window
narrowing", and a one-word verdict (Early / Mid / Likely late) which is precisely the judgement
we are meant to make silently. It also has a grammar bug: "1 posts tracked across X."

**(3) A views-per-minute sub-line** in the feed row (`feed.js:558`) — our rate estimate computed
off our own snapshot cadence, not a fact about the world.

**(4) A Gain 24h column whose colour flips on a hardcoded internal threshold** —
`feed.js:549`, `gain>=150000?'up':'down'`. A judgement rendered as a colour.

**(5) "live · 1 narratives"** (`live.js:863`) exposes that the pipeline produced one eligible row.
**"1 clusters"** (`index.html:66`) uses our internal word for what the user calls a story.
**"Google trend / Google Trends · search interest"** (`feed.js:242-246`, `index.html:72`) names
our data vendor.

**(6) The whole scoring table is publicly readable.** `worker/schema-score.sql:23-26` enables
RLS on `post_meme_scores` and grants anon SELECT with `using (true)`, covering the `reason`
column (the judge's plain-English rationale) and the `raw` column (the full Claude API response
with token counts and message ID). The anon key is published at `site/config.js:8` and served at
`/api/public-config`. Two reviewers independently pulled the judge's verbatim sentences using
the live site's own credentials — e.g. *"Official team account posting motivational content;
promotional/brand messaging, not organic viral moment."*

Additionally `gate_reason`, `organic_score`, `accel`, `engagement_velocity`, `lifecycle` and
`meme_score` all cross the wire to the browser (`live.js:397-407`, `:785-800`). An unused
`.gate-pill` CSS class survives at `index.css:824`, so a gate reason was on screen at some
point.

"Our reasoning never leaves the building" is not satisfied by declining to render it. Enforce
it in the SELECT list and the RLS policy.

### 5.5 The clustering does not cluster

982 of 1,079 populated narratives (91%) contain exactly one post by exactly one author. 1,150 of
2,411 posts are not assigned to any narrative at all. Every open narrative shows
`author_velocity = 0`.

The cause is `worker/cluster/lib/embeddings.js:5-15`: `embedText` is bag-of-words term-frequency
with L2 normalisation and a 60-word stoplist. It is a lexical overlap score wearing the name
"embedding". So the third matching pass ("embedding cosine", threshold 0.42) is the same signal
as the second pass ("keyword overlap", threshold 0.28) with different weighting. Two posts about
the same event in different words never merge — "Fans Beg Vini Jr To Stay" and "Real Madrid
raised their offer to Vini Jr" sit in separate narratives.

The product's whole premise is cross-post narrative detection, and it is not happening. Real
sentence embeddings are cheap and local; this is the highest-leverage single swap in the
rebuild.

**Related:** cashtag matching returns on the *first* narrative sharing *any* cashtag
(`cluster-engine.js:84-98`), before keyword or embedding is considered. One crypto narrative
accreted 39 unrelated posts and 88 ticker rows — SOL, BTC, ETH, NVDA, TSLA, SPY, QQQ, DOGE —
because they all said `$SOL`. Common tickers need a stoplist.

### 5.6 The wrong series is plotted, and the right one is thrown away

The feed's line graph plots `search_series` — Google Trends via SerpAPI. That column is non-null
on 32 of 1,293 narratives, and **null on the one row the site displays**, so the cell renders
empty. Verified in a browser: `document.querySelector('.nm-pulse').innerHTML === ""`.

Meanwhile the view-count series the design actually asks for exists at scale: 57,032 snapshots
covering 2,410 of 2,411 posts, median 9 points per post. The displayed narrative's post has 10
snapshots rising 50,445 → 220,568 views.

**Correction to an earlier claim.** One reviewer called this "a one-function fix" because
`live.js:392` fetches snapshots nested in the narratives query and `live.js:512-545` discards
all but the newest. The discard is real and is the choke point. But the render path does not use
that query — `live.js:394-396` defines `NARR_SELECT_LIGHT` ("Fast boot — skip nested
snapshots"), `live.js:1885` defaults to it, and the boot call at `live.js:2027` passes
`lightFetch: true` explicitly. Confirmed at runtime: `'post_snapshots' in NARRATIVES[0].posts[0]`
is `false`. The snapshots do arrive on every page load, but through a *different* query
(`fetchPipelineNarrativeRows`, `live.js:1024`) landing in a *different* array
(`window.NARRATIVES_ALL`).

Real scope: about six functions across three files — retain the series in `mapPostRow`, flip off
`lightFetch` or merge from `NARRATIVES_ALL`, stop aliasing `search_series` at `feed.js:196`,
relabel the column, re-tune `narrViralGain24h` and `narrLifecycle` (both calibrated to a 0–100
Trends index and would misbehave on raw view counts), and give `narrSparkSVG` a time-based
x-axis. That last one matters: the displayed post's 10 snapshots are three tight bursts of 2–4
minutes separated by two ~50-minute gaps, and an index-spaced sparkline would silently flatten
those gaps and fabricate the curve's shape.

Still worth doing. Just budget a small multi-file change, not a one-liner.

### 5.7 The designed row was not built; the rejected row was

`docs/analysis/visual/sections/02-feed.html` specifies six rows of seven elements, with two
lines of plain English as the largest, brightest text and the title deliberately smaller. What
shipped is a sortable-column trading terminal — Narrative / Google trend / Views / Gain 24h /
Top mcap / Age (`site/index.html:70-77`) — which is the layout the same document shows under the
heading *"Rejected — our workings on the user's screen."*

The plain-English sentence exists in the schema as **one** sentence capped at 160 characters
(`worker/adapters/anthropic/narrative-title.js:32,83`) and renders at 11.5px muted, nowrap,
ellipsised, beneath a 14px title (`site/styles/index.css:850-851`). The design mock sets the
sentence at 15.5px `--ink` with a two-line min-height and the title at 12.5px `--muted`. Both
the size order and the colour order are inverted.

Two corrections worth recording. First, a two-line block *does* ship — `.nm-about-1` /
`.nm-about-2` at `index.css:898-899`, rendered in the narrative modal one click off the row. The
element exists; it was demoted. Second, "one sentence" describes the prompt, not the data: of
1,000 live blurbs, 265 end in an ellipsis because they are raw top-post text truncated at 118
characters by `worker/lib/text-utils.js:190-195`, 312 have no terminal punctuation, and 50 start
lowercase. Roughly three quarters exceed the ~55 characters the column can show, so most rows
ellipsise the element the design calls the point of the entire screen.

Also absent entirely: the narrative page. No per-story URL, no shareable link, no thesis layer,
no per-post evidence list, no coin table with First / Biggest / Unverified labels. Both
defensibility arguments in the walkthrough live on that page.

### 5.8 Migrations were written and never applied

`worker/schema-token-lookup.sql` adds `mint_ca`, `dex_url`, `pump_url`, `lookup_at` to
`narrative_tickers`. All four return Postgres error 42703 "column does not exist" against the
live project. There is no `worker/apply-schema-token-lookup.js` (11 other migrations have one)
and no npm script. `worker/schema-posted-at.sql` is also unapplied — `posted_at_ts` returns
42703 — even though its applier *does* exist.

**Two corrections here, in opposite directions.**

Against the severity: the code is written so nobody ever sees the failure.
`worker/cluster/lib/enrich-tickers.js:38-44` catches any error matching `/column/i` and retries
without the four fields, so the mint is fetched from DexScreener and silently discarded on every
enrichment. And the frontend has a recovery path — `site/live.js:747` falls back to
`lookupTickerRemote(sym)` against `/api/token-lookup`, repopulating mint and URLs client-side.
So "every coin link is dead" is too strong as a mechanism claim. What was *observed* on the live
page is that zero of 1,025 ticker rows had a mint, so the narrative → coin → Buy path has never
been exercised end to end against real data. I cannot fully reconcile the observation with the
fallback path; treat that as an open question rather than a settled defect.

For the severity: the residual cost is that mint identity is re-derived by *ambiguous symbol
search* on every page load rather than pinned once — which is section 5.2's defect, not a
separate one.

Also: `docs/schema.md:100` asserts `mint_ca` is present on the live DB. It is not. The cause is
in the generator — `worker/generate-schema-doc.js:108-115` seeds each table from the `.sql`
files and then unions the live probe on top, stamping *everything* `(live)`. So the doc's own
legend ("(live) = confirmed present by row probe" vs "(sql) = not confirmed") is a distinction it
never actually produces, and **every SQL-only column in `docs/schema.md` is mislabelled**. That
also propagates into `lib/db-schema.js:156-174` via
`worker/generate-schema-constants.js:28-31`, so `c('narrative_tickers','mint_ca')` resolves
instead of throwing — the guard rail defeats itself for exactly this class of drift.

One live trap: `docs/schema.md:391` recommends syncing `posted_at_ts` on every `posted_at`
write. Acting on that would add the field to the ingest payload, and
`worker/ingest/lib/ingest-upsert.js:69` throws on upsert error — ingest would hard-fail rather
than degrade.

### 5.9 Enrichment is destroyed every cycle

`worker/cluster/lib/cluster-persist.js:156-182` deletes every ticker row for a narrative and
re-inserts bare rows with `mcap`, `liquidity`, `vol24h`, `holders` and `age_min` hardcoded to 0.
`cluster.js:310` calls it, then `:312` re-enriches — but `enrich-tickers.js:10` returns null on a
lookup miss, leaving the row zeroed. A single DexScreener miss permanently erases previously
stored on-chain data. She logged this herself as M4 and did not fix it.

### 5.10 Smaller, but worth carrying as requirements

- **Stock-photo fabrication is still live.** Seven `loremflickr` / `picsum` images render on the
  current page (`index.html:302-308`, `feed.js:211-214`). Fabricating media on a precision
  instrument is a lie told in pixels. Replace with a seeded gradient and a "no media" chip.
- **The Tokens tab is unfiltered DexScreener noise** with a *fabricated* 24-hour sparkline —
  `sparkSVG` at `index.html:427-433` synthesises a 24-point curve from a hash of the contract
  address. The live registry contains SOL twice, at $1.53B and $1.63B.
- **~400 lines of live simulation code** keyed on permanently empty arrays, including
  `setInterval(pushStream, 2600)` firing every 2.6 seconds forever on every tab
  (`feed.js:1133`), and a canvas candlestick chart drawing a random walk into an element the
  token page no longer renders.
- **The meme gate is enforced in the browser.** `feed.js:1112-1118` filters the cached feed
  against `window.MEME_MIN_X`, thresholds also published onto `window` at `live.js:386-389`.
  Score, threshold and decision are all editable from the console.
- **Privy loads React, ReactDOM and two Privy bundles as four runtime CDN imports**
  (`site/features/wallet/privy.js:27-41`), and the two Privy bundles resolve to different module
  instances — 14 `useWallets was called outside the PrivyProvider` warnings per page load, so
  the Solana hook returns empty and `Auth()` never resolves an address.
- **`meme_min` is computed, returned and logged but never persisted** — there is no such column
  on `narratives`. If you persist a verdict you must persist the threshold it was judged
  against, or past decisions cannot be audited after config changes.
- **214 narratives have views but zero posts**, 35 of which still carry a ticker row. A narrative
  with no posts is unfalsifiable.
- **X and TikTok view counts share one column and one threshold.** X-only narratives have median
  145,500 views; TikTok-only 14,100,000 — a 97× scale difference against a single 200,000 bar.
- **There are no tests.** No test directory, no test script. `parseMemeScoreResponse`,
  `applyNameabilityCap`, `deriveGateReason` and the budget arithmetic are pure functions with
  clear contracts and should be pinned before anything is swapped underneath them.

---

## 6. The LLM question, settled with real numbers

### Where it is called

Exactly three sites, all Anthropic Haiku 4.5 (`claude-haiku-4-5-20251001`), all raw `fetch` to
`/v1/messages`, all behind `worker/adapters/anthropic/`. Nothing else in the codebase talks
HTTP to a model.

1. **`scoreXPost`** — `meme-score.js:273`. Text-only, max_tokens 256, temperature 0.2. Runs on
   the top 10 posts by views velocity every 3 minutes.
2. **`scoreTikTokPost`** — `meme-score.js:333`. Vision call with a base64 thumbnail.
3. **`generateNarrativeCopy`** — `narrative-title.js:136`. Title and blurb, max_tokens 200.

### What it actually costs

From `worker_usage` where `source='anthropic'`, re-queried live:

| Date | Cost | Calls |
|---|---|---|
| 2026-07-24 | $1.7286 | 1,746 |
| 2026-07-25 | $1.8085 | 3,285 |
| 2026-07-26 | $0.3883 | 765 |
| 2026-07-27 / 28 | $0.00 | 0 |
| 2026-08-04 | $1.9998 | 1,619 |
| 2026-08-05 | $3.4459 | 2,893 |
| 2026-08-06 | $0.0867 | 163 (partial) |
| **Total** | **$9.4578** | |

**An earlier draft of this review said "$9.45 over 8 days, $1.18/day, ~$35/month, every dollar
measured from real API usage blocks." Three parts of that are wrong.**

- *"From `cost_breakdown`"* — `cost_breakdown` sums to $8.6288. The other $0.8290 (8.8%) is on
  2026-07-24, before that column existed (`worker/schema-anthropic.sql:3-4`), and has no
  usage-block provenance. The dollars that *are* attributed reconcile exactly to
  input × $1/MTok + output × $5/MTok, which is correct Haiku 4.5 pricing
  (`worker/adapters/anthropic/pricing.js:8-9`, `:29-40`).
- *"8 days"* — 8 rows spanning 14 calendar days, with a 6-day outage gap, two zero days, and one
  partial day that was still incrementing between queries. Depending on denominator: $1.18/day
  (8 rows), $1.58/day (6 active days), or $0.68/day (14-day span).
- *"~$35/month"* — this reads a **budget cap**, not demand, and the cap moved. 2026-08-04 landed
  at $1.999806, pinned to four decimals against what was then a $2 ceiling. The cap was raised;
  the next day spent $3.4459. Cost per score call rose 2.7× over the window as the prompt grew
  (362 input tokens/call on 07-24 → 1,444 on 08-05).

**The honest numbers.** Last full day: $3.45 ≈ **$103/month**. Current configured ceiling:
`worker_pipeline_state.anthropic_budget_usd = 5` ≈ **$152/month**. Uncapped at current cron
cadence: ~$10.13/day ≈ **$300/month**.

Note also that X costs more than the judge: $2.77 and $4.19 on the same two days, and $11.17 on
2026-07-24. And `score_tt` has never fired once — 0 calls, $0.00, every day in the table — so
half the judge's cost surface is entirely unmeasured and still sits at a $0.012/call estimate
(`budget.js:18`), and `MEME_MIN_TT=0.75` is uncalibrated.

**Our research note needs four corrections.** `docs/analysis/research/05-judge-and-trading.md`
says $2/day and "$61/month at any volume, ever." (a) The production cap is 5, not the code
default of 2. (b) It cites `worker/lib/anthropic-budget.js`, which does not exist — the module is
`worker/adapters/anthropic/budget.js`. (c) It says `assertStartupBudget()` refuses to boot if the
projection exceeds the cap; that function is called only from `worker/index.js:101`, the
long-running worker, whose last heartbeat was 2026-07-24 while crons ran through 2026-08-06.
It is dead in production. (d) The coverage table understates: at 10,000 posts/day the doc claims
63% of posts go unjudged; against a $5 cap it is about 9%. At the next plausible growth step the
rationing the doc argues from largely disappears.

**Conclusion on cost: it is not the reason to replace the judge.** But the cap is an env var with
no value pinned anywhere in the repo, and it is one redeploy from being anything. If a hard
ceiling is wanted as a guarantee, pin it in code or in Anthropic-side billing limits.

### The real arguments for replacing it

**1. The output is effectively ternary.** Across 1,000 production scores: 0.15 appears 604 times,
0.85 sixty times, 0.05 fifty-eight, 0.25 fifty-one. Nothing above 0.85. Across 1,282 scored
narratives, only 101 (7.9%) fall in [0.3, 0.7). We are paying generative-model prices for a
yes/no label with a decorative decimal, and getting no usable probability to threshold.

**2. 82% of spend is one static prompt, retransmitted.** `COINABILITY_CORE` is ~1,100 tokens
(`meme-score.js:17-57`) resent uncached on every one of 1,653 daily calls. Prompt caching does
not help — Haiku 4.5's minimum cacheable prefix is 4,096 tokens, and every response shows
`cache_read_input_tokens: 0`. A fixed decision rule re-transmitted per item is exactly what a
trained classifier replaces. (If you stay on an LLM, batching 10 posts per call cuts score spend
roughly 8× for free.)

**3. Naming runs before the gate that rejects the narrative.**
`cluster-engine.js:403` generates the Claude title; `:432-448` computes `display_eligible`. 45
lines apart, in the wrong order. On 2026-08-05 naming was 1,240 of 2,893 calls (43%) and $0.62
of $3.45 (18%), against 103 narratives created that day — about 12 regenerations each.
`narrative-copy.js:14` regenerates whenever `membersAdded > 0`, so attaching one post re-titles
the whole narrative. Today's entire bill is 147 title calls on stale clusters, zero score calls.
Moving naming behind the gate cuts that line roughly 40×. `fallbackCopy()` already exists at
`narrative-title.js:96`.

**4. A spend under-accounting bug.** `narrative-title.js:163-169` calls `recordCall` only inside
`if (parsed)`. Any title rejected by the parser (wrong word count, contains `@ $ #`) was billed
but never recorded against the budget. `meme-score.js` gets this right at `:280` and `:356`.

### What replacing it requires

**The interface is mostly clean.** `memeScore(post, sb)` → `{meme_score, reason,
suggested_ticker, suggested_name, model, raw, scoring_mode}` is the only contract `score.js`
depends on, and the two rules doing real product work — the nameability cap and the meme gate —
are already deterministic code outside the LLM.

**Three things to untangle first**, roughly a day's work:
1. Lift the budget check out of the adapter (`meme-score.js:7-12`, `narrative-title.js:5-12`)
   into the caller, so a local model needs no budget stub.
2. Replace `scoring_mode` string-literal branching (`'budget_skip'`, `'tt_budget_skip'` at
   `score.js:192,198`) with a typed `{ok, reason}` result.
3. Pass the model identifier through the result rather than importing `MODEL` from the adapter
   to fill a DB column default (`score.js:13,95`).

**The acceptance test exists, but its label is broken.** `worker/score/rescore-validate.js` plus
`docs/rescore-threshold-analysis.md` rescore production posts offline — never writing to
production tables — and check whether a Solana token matching the suggested subject launched
after the post. Reported result: 72.9% coined above 0.70 versus 1.9% below, a 71-point gap on
629 narratives, with the rate flat 71–74% from 0.70 to 0.85 (so the threshold is not delicate —
do not spend time tuning it).

**Do not carry that 71-point number forward as established.** The label is circular in two ways.
The search terms come from the model's own `suggested_ticker`
(`worker/score/lib/coin-launch-check.js:30-48`), and when there are no terms line 124-127
returns `coined: false` without querying anything — while `nameability.js:43-49` caps the score
at 0.3 on the *same* missing-ticker condition. Measured on the production score table: 673 of
1,597 below-gate rows (42%) have neither ticker nor name and are uncoinable by construction; 0
of 146 at-or-above-gate rows lack a ticker. The control arm cannot produce cases and the
treatment arm always can. Separately, `hitMatchesTerm` (`:57-65`) accepts a substring hit on the
*token name* for any term ≥4 characters, `:150` has no upper time bound (a coin minted weeks
later counts), and there is no liquidity floor. Against ~30,000 mints/day a symbol collision is
close to free for any famous word. The report's own false-positive list is the tell — YOONGI,
RM, MESSI, LAUFEY.

The harness is the right instrument and should be kept. Point it at a mint-level label with a
bounded window and a null arm before treating any lift as real.

**One thing to give credit for.** The `works_as_photo` test is already gone from the prompt. She
ran the backtest, saw the number was negative, and replaced it with SUBJECT vs STORY — "is there
a THING someone could put on a coin, or is it just a topic?" That is better product judgement,
not worse: nameability is a property of the content, "works as a photo" was a property of our
aesthetic taste. It also has a deterministic implementation in `nameability.js`, which the photo
test never could.

**One thing the prompt gets backwards.** The prompt tests the features the backtest scored
negative (`has_character`, `one_word_name`) and ignores the ones it scored positive
(`ticker_in_replies`, `crypto_noticed`, `others_copying_it`) — even though all three are already
computed elsewhere in the repo (`ticker-proposals.js`, `replication.js`, `tt-score-gate.js`) and
never fed to the classifier. Given section 2, treat this as a *hypothesis to test in the forward
study*, not an instruction to rewire the prompt. Those three features are the contaminated ones.

---

## 7. What the build proves about our documents

The documents are not the client here. Where reality contradicted them, reality wins.

**1. The "34-minute gap" in walkthrough section 01 does not exist in a running system.**
The document stakes the company on Insidor seeing a story at 09:13 and a coin existing at 09:31.
Measured on the live X ingest lane: median detection lag from post time to first seen is **7.7
minutes**, p90 15.2 (n=673 non-near-miss posts; 7.6/15.6 over the last 24 hours). Only 0.7% of
posts are seen within 3.8 minutes.

*Two corrections to the version of this finding that first came back.* The originally reported
9.2 min / p90 83.4 figure was inflated — 327 of those 1,000 rows carry
`filter_label='near_miss'`, which is a *deliberate deferred promotion* out of the holding table
(`near-miss.js:95`), not a detection miss, and the window straddled a 9-day worker outage. And
all these figures are right-censored by `RECENCY_MIN=30`, `MIN_INGEST_VIEWS=30_000` and the
`min_faves` floor, so they understate true time-to-detect.

More importantly: **this is not news, and our own research already settled it.**
`docs/analysis/research/07-virality-and-detection.md:37-44` states the pre-mint window is about
two minutes wide and contested by bots firing in milliseconds, and explicitly moves the clock —
"the winnable window is not 'before the mint', it is 'after the mint and before the run'."
Against a post-to-peak median of roughly six days, a 7.7-minute detection lag spends 0.09% of
the budget. Walkthrough 01 is stale relative to research 07. Close the gap in the document, not
in the pipeline.

**2. Our thresholds converted a pre-mint product into a post-mint one, silently.** A post needs
30,000 views to be ingested (`ingest.js:60`) and a narrative 200,000 combined views to display
(`cluster.js:38`). Those floors were chosen to control vendor spend. Nothing except a
mega-account post reaches 200K views inside any pre-mint window. Entry thresholds must come from
the mint-lag distribution, not from the budget.

**3. Our cost claim about the judge was wrong by 2.5×, and cited a file that does not exist.**
See section 6.

**4. Our architecture document was right about the thing it warned us about.**
`docs/analysis/ARCHITECTURE.md:3499-3505` predicted exactly this failure mode for Vercel Cron:
"an invocation that dies at the ceiling looks identical to one that finished with nothing to
do… no first-class *ran and produced nothing*." That is precisely what happened at
`run-stage.js:56-67`, and it is why the ingest outage went unnoticed for seven hours. Keep that
requirement.

**5. Her architecture document is right where it disagrees with ours on the present, and stale
on its own file inventory.** She is right that `display_eligible` should be a persisted column
rather than a query-time computation — our design under-specifies this and hers made the funnel
auditable. She is right that Vercel Cron plus Supabase was enough to prove a pipeline exists;
ours never ran. But `docs/ARCHITECTURE.md` claims to list every file and names paths that no
longer exist (`worker/ingest.js`, `worker/lib/cluster-engine.js`, `worker/lib/anthropic-budget.js`),
and documents no `worker/adapters/` or `worker/backtest/` tree at all. Generate the inventory or
drop the claim.

**6. Both documents share one error, and the live data shows what it costs.** Neither treats coin
matching as a hard problem with a confidence attached. Ours describes the machinery — image
embeddings, matcher version — without a rule for what to do when it is unsure. Hers has no
matching at all beyond a symbol string. The result is section 5.2.

**7. Our design documents call `search_series` "a fabricated sine wave"
(`ARCHITECTURE.md:1377`, `DESIGN.md:484`). That is stale.** `worker/trends/trends.js` writes real
SerpAPI output and there is no `Math.sin` in the live path. The series is real; it is just
nearly always empty, and near-all-zeros where it is populated. Do not carry the "fabricated"
framing forward — it will make someone dismiss a real column for the wrong reason.

**8. Our documents specified how to test code and never specified how to test a claim.** She
invented the blind harness to fill that hole. Whatever else the rebuild inherits, inherit that
gap being closed.

---

## 8. The rebuild inheritance

### Carry across

**Data first, because it cannot be recreated.**
- `post_snapshots` — 57,032 rows, the view-rate curve, no vendor sells this retroactively.
- `ingest_near_miss` — 8,101 rejected posts with full raw payloads. The negative class.
- The three backtest reveal reports and every wave CSV. Keeping the earlier, wronger waves on
  disk is what made the sign-flip instability visible. Do not delete them.

**Methods and instruments.**
- `worker/backtest/dune-peak-multiples.sql` — the outcome labeller, verbatim.
- The blind/answer file split and the seeded-shuffle reproducibility, upgraded so the answer is
  structurally unavailable rather than merely warned about.
- The offline eval harness shape: rescore into a cache, never write production tables,
  regenerate a markdown report. Keep the harness; replace the ground-truth function.
- `worker/lib/latency.js` — the metric that refuted our premise. Put its successor on a
  dashboard.

**Code and patterns.** Everything in the table in section 4, plus: the pure-function gate
returning named reasons; the resumable stage checkpointing; the per-call-type cost meter; the
cycle log that records thresholds alongside counts; the anti-self-clustering rule; the
park-and-recheck loop; the per-feature README format; `applyNameabilityCap`; the cheap
deterministic pre-filter in front of an expensive vision call.

**Prompt text, as text.** `COINABILITY_CORE` and the SUBJECT-vs-STORY framing survive the model
swap as labelling instructions, independent of any code.

### Leave behind

Vercel Cron as the execution model — no singleton, no backpressure, no productive-run signal,
and a failure path that writes nothing. The undirected catch-all X query. Bag-of-words posing as
embeddings. Symbol-string coin matching. The sortable-column feed. The Google Trends sparkline.
The user-facing budget banner. The simulated Create Coin flow. The fabricated stock photos and
the fabricated token sparkline. Three module systems in one page (`document.write`, ESM, plain
globals). Twenty-one `apply-schema-*.js` scripts with no ordering, no version table and no
forward-only discipline. Anon SELECT on the scoring table.

And leave behind every *conclusion* drawn from the backtest, in both directions.

### The biggest risk not to repeat

**Building a measurement you cannot trust, and then acting on it.**

It happened four separate times in this build, in the same shape each time:

- The backtest measured graduated coins and reported on coinability.
- The validation harness derived its "coined" label from the model's own output, then used it to
  validate that model.
- The coin matcher treated symbol equality as identity and marked the result `first_deployed`.
- The schema doc generator unioned intent with reality and stamped the whole thing "(live)".

Each one produced a number that looked like evidence. Each one, checked, was measuring itself.
And in three of the four cases the developer had *already written down* that something was
wrong — the CONTAMINATED header, the M2/M4 defect register, the "descriptive only" caveat — and
the number got quoted anyway.

The rebuild's rule should be that every measurement declares its population, its label source,
and its window, in the artefact itself, next to the number — and that a number with a caveat
attached is not permitted to appear in a summary without the caveat.

The second risk, concretely: **a wrong Buy button costs a user money.** Section 5.2 is the one
defect where the failure mode is not an empty screen or a wasted dollar. Match confidence must
be a column, the coin must postdate the post, and when we are unsure the correct render is no
button at all.
