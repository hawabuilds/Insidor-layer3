# History: every pump.fun narrative that ever existed — plan only

This replaces the coin-first 3.2–3.6 plan. Nothing here has been run. There is no
`DUNE_API_KEY` yet, and no history is fetched until the founders say so, stage by stage.

The goal is a database of **narratives**, not of coins. A coin matters to history only as
evidence that a narrative existed and when. So the import pulls only the coins that were traded,
groups them into narratives first, and then does the expensive work — finding where a narrative
started — once per narrative instead of once per coin.

The queries are in [`tools/dune/`](../tools/dune). Each one is paged by date, so a run can stop
and resume, and each page has a known size.

## The six steps

### 1. Pull past coins from Dune: traded coins only

| stage | what | query | rows (estimate) |
|---|---|---|---|
| 0 | count stage 1 before fetching it | `0_counts.sql` | 1 |
| 1 | every coin that **bonded**, since launch | `1_graduated.sql` | **140,000–245,000** |
| 2 | coins that did not bond but were **really traded**: at least 50 SOL through the curve from at least 50 wallets | `2_traded.sql` | **350,000–700,000** (the big unknown) |

- **We store only** the name, ticker, metadata link, launch time and the picture, read from the
  metadata through our own gateway, the same code as live (3.1). Creator wallets, volumes and
  trader counts come back from Dune but are not kept.
- **Zero-trade coins are skipped.** The old stage 3, every coin by name only (~15M rows, 4–6 GB),
  is gone.
- **Why both bars for stage 2.** One wallet pumping and dumping clears the volume bar alone, and
  fifty wallets buying a cent each clear the trader bar alone; neither is a market. Holder counts
  would be a better bar, but they need a scan of every token balance. Distinct traders gets the
  same idea from the table the query already reads.
- **Stage 2 is the expensive query.** It scans every trade on the curve, so it is paged one month
  of launches at a time. Its first page (one month) is the measurement: it gives the real ratio
  of traded to launched, which replaces the guess above before the other pages run.
- **Coins pulled: about 0.5–0.95 million.**
  - Inputs: about 11.9M launches by June 2026, plus about 42,000 a day since, which makes about
    17.5M by now.
  - Bonding rate: 0.8–1.4%.
  - Traded but never bonded: guessed at 2–4% of launches.

### 2. Group the coins into historical narratives

This uses GROUP's own evidence, the same signals the live system uses:
- **name and ticker variants**: free, pure logic;
- **picture match**: Voyage embeddings of each coin's picture, with near-misses checked by the
  same image-pair judge as live;
- **shared linked posts**: two coins whose metadata links the same post are the same narrative.
  This is free, because it is read from the metadata we already fetched.

Each group is stored as a historical narrative (a `public.story` with a new origin kind,
`history`) with all its coins. It is searchable, but it is never on the live board.

**Expected narratives: about 150,000–475,000** (30–50% of coins). Most are a single coin. Only
about **25,000–60,000 have two or more coins**, and those are the ones people search for.

### 3. Find the origin once per narrative

Biggest narratives go first: by number of coins, then by peak market cap, inside a daily cap.

1. **Creator links first (free reads).** The 3.3 checks for a promo post, an account link and an
   influencer link, applied to every coin's metadata link in the narrative. The earliest proven
   post wins.
2. **Then one X search back** from just before the narrative's first coin. Picture and name checks
   run on what comes back: a name pre-filter (free), picture embeddings of the results that have
   media, and the judge on the few that survive, in batches of 10.
3. **A post later than the first coin** (past the 10-minute launch allowance) is labelled
   "viral after launch". It is kept and shown, and never counted as the origin.

### 4. Store a timeline per narrative

Each narrative stores the **earliest post found**, the **first coin**, the **biggest coin** and the
**peak**. A gap is stored as a typed reason, never as a blank:

- **origin not found**: searched, and nothing passed the checks;
- **possibly deleted**: a creator linked a post that no longer resolves;
- **TikTok not searchable yet**: the evidence points at TikTok, which has no search we can afford.

This is the "first post, first coin, peak" line the Search tab already draws for a narrative
that is not live (section C).

### 5. Make historical narratives searchable

The Search tab's narrative section (`services/read/src/search.ts`) already searches
`public.story_view` by title and coin ticker. Historical narratives join it by being projected
there with `history` provenance.
- The search filter widens from `observed` to `observed` + `history`.
- The live board and every surface that asserts liveness stay `observed` only.

### 6. Re-estimate everything

| | estimate | what settles it |
|---|---|---|
| **Coins pulled** | 0.5–0.95M (bonded 140–245k, traded 350–700k) | stage 0, then stage 2's first month |
| **Narratives** | 150–475k, of which 25–60k have 2+ coins | grouping stage 1 alone, before stage 2 |
| **Metadata reads** | 0.5–0.95M, free through our gateway; 10–19 days at 50,000 a day | — |
| **Grouping cost** | **$100–200** one-off: ~1,450–1,800 tokens a picture at $0.12 per million ≈ $0.0002 a coin. Plus **$10–30** for the image-pair judge on near-misses | the first 10,000 pictures, measured |
| **Origin search cost** | ≈ **$0.0067 a narrative**: one X search of 20 results ($0.003), pictures of ~8 results ($0.0016), the judge on ~3 survivors ($0.0021). Creator links add ~$11 in total | the first 1,000 narratives, measured |
| — the top 50,000 narratives | **≈ $340**, 10 days at the proposed cap | |
| — the top 100,000 | **≈ $670** | |
| — every narrative (~300,000) | **≈ $2,000**. Not recommended: the long tail is single coins nobody searches for | |
| **Database size** | **1.9–3.0 GB in total** — needs Supabase Pro (8 GB), not the free tier's 500 MB | |
| — coins | 0.7–1.4 GB: 1.47 KB a coin, measured on the real schema (asset row 0.88 KB + metadata 0.59 KB) | |
| — narratives and timelines | 0.17–0.5 GB: ~1.1 KB each, with the origin post where one is found | |
| — picture vectors | 0.1–0.2 GB if one vector is kept per multi-coin narrative. Keeping one per coin is 2–4 GB plus its index, which is why it isn't | |
| — the live universe | ~0.9 GB: 14 days of every coin, then the keep rule (item D) | |

**Dune's own cost** is in credits per execution, plus credits for exporting results.
- Stage 0 and stage 1 (about 20 pages, one join of decoded tables) should fit a small allowance.
- Stage 2 scans every curve trade, about 30 monthly pages on a large engine, and probably needs a
  paid month.
- I'll turn credits into dollars against your plan after stage 0 and stage 2's first page, which
  together cost a few credits.

## Proposed policy numbers (not yet in `policy.ts`)

| field | proposed | why |
|---|---|---|
| `universe.history.volumeBarSol` | 50 | about $7–9k through the curve: a market, not a test |
| `universe.history.traderBar` | 50 | fifty wallets: not one person and their friends |
| `universe.history.pageMonths` | 1 | each Dune page a known, bounded size |
| `universe.history.metadataPerDay` | 50,000 | a public gateway's comfortable rate; history never competes with live coins |
| `universe.history.originsPerDay` | 5,000 | ≈ $34 a day at most; the top 50,000 narratives in ten days |
| `universe.history.searchResults` | 20 | one page of results from just before the first coin |

## How it runs, when it does

- **One Dune page at a time.** The last finished page is stored, so a stopped run resumes there.
  Each page records the credits it cost and the rows it returned.
- **Grouping runs after stage 1, before stage 2 is fetched.** The narrative count from bonded
  coins alone is the cheapest real number we can get, and it decides whether stage 2 is worth its
  scan.
- **Origins run biggest first, under the daily cap.** Each search is priced before it runs, like
  every paid stage (`SPEND=live` and `--yes`, run by you).

## Sources for the inputs

- [Pump.fun 42,000 token launches in 24 hours: the 2% graduation rate (Solana Compass)](https://solanacompass.com/news/pumpfun-launched-42000-tokens-in-one-day-fewer-than-2-will-ever-reach-a-dex)
- [Pump.fun graduation rate: only 0.84% of tokens succeed, April 2026 (pumpfundata.com)](https://pumpfundata.com/blog/pumpfun-graduation-rate-analysis)
- [Longing attention markets — Oregon Blockchain Group's $PUMP report](https://medium.com/oregon-blockchain-group/longing-attention-markets-oregon-blockchain-groups-pump-report-8fcbf736baee)
- [Pump Fun Graduated Tokens (Dune)](https://dune.com/queries/3979196)
