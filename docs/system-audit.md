# Insidor system audit

How everything works right now, as deployed on 2026-10-13. Written from the live database, the services' logs and the code at the deployed commit — not from plans. Measured between 13:19 and 13:40 UTC unless a time is given. No secrets, keys or passwords appear here; where a setting is secret, only its name is given.

Where something differs from what was asked for, it says so. Where something was not checked, it says "not checked yet".

## In one page

- **Posts in:** about 560–715 X posts an hour, and TikTok videos from one search run about every hour. Discovery runs nonstop except for a gap from 00:08 to 02:04 UTC today (discovery was paused by the old pacing and the database pool was full).
- **Judged:** about 140–290 posts an hour, by Claude Haiku 4.5, with no learned examples in use. Only posts that already clear the For You reach bar are judged.
- **Shown:** For You had 13 posts and Popular 6 at 13:28 UTC, all from X. No narrative is on the board.
- **Judge quality:** on today's test sets the live judge passes 55–58% of gold coin posts and 15% of live non-memes. The targets are 65% and 5%. Neither re-tune today could be applied.
- **Coins:** no feed card shows a confident coin or a Buy button right now. On the gold set the matcher picks the right coin 7 times out of 71, links a wrong coin 1.7% of the time, and misses the rest.
- **Trading:** switched on for the owner's wallet only. Never tested with real money.
- **Money:** $35.75 a day across all lines, under the approved $38. Spent today by 13:24 UTC: $21.16. Lines warn and never stop; only each book's daily total stops spending.

## Fixed during this audit (13:30–13:50 UTC)

The audit found three faults that were fixed and deployed while it was being written. The sections below describe the system as it was found; this list says what has since changed.

1. **Narratives had stopped forming at 23:04 UTC on 2026-10-12.** ADMIT, DETECT and GROUP read the 2,000 oldest posts of the last two days and skipped those already handled, so once two days held more than that, they never reached a new post. They now read newest first (commit `be56a8d`, deployed 13:36 UTC). **Verified:** 200 posts joined narratives in the five minutes after it went live.
2. **TikTok videos were never judged.** They were stored, then credited to the new watch lane, but the scorer read only the 1,000 newest unscored posts, and X adds about 700 an hour, so a video stored hours earlier never made the page. The watch lane now has its own queue (commit `69aeb06`). That alone did not get them judged — see 4.
3. **A TikTok search run held up the X search.** A most-liked, past-day search takes about four minutes, and the X lanes waited behind it; one run at 13:20 UTC went past the five-minute wait and its results were dropped while the vendor still billed it. TikTok asks now run beside the X lanes, and a run is waited on for nine minutes (commit `96dfcf7`).
4. **The judge was never shown a TikTok picture.** Its check on picture addresses refused any address with a query string — a rule written for the API's own base address. Every TikTok cover is a signed address with a query string, so every TikTok video came back "no picture to look at" and was never sent to the judge. Picture addresses now need only be well-formed https (commit `f5d9ed4`, deployed 13:48 UTC). **Not yet verified:** at 13:50 UTC no TikTok video had been judged. The new most-liked search went live at 13:16; its 13:20 run timed out (before the longer wait was deployed) and no TikTok video newer than 13:07 is stored yet.

## 1. The live pipeline, from a post to For You

### Services and cadences

**Runner** (`services/runner/src/main.ts`) is one process with nine loops. The `rank` loop is off. The `[s]` after each cadence is the start offset.

| Loop | Every | What it does | Paid supplier |
|---|---|---|---|
| **discover** | 2 min [45s] | Runs the X catch-all search lanes (`all` and `media` every pass, `slow` every 15 min) and asks TikTok its watch-list word when that is due. Stores posts and authors. The search result itself is saved as the post's first view reading. After the pass it runs three steps in order: **score-posts**, **embed-images**, **backfill**. | X via twitterapi.io (`search`); TikTok via Apify actor `clockworks~tiktok-scraper` (`discover`) |
| ↳ score-posts | after each pass | Judges each catch-all post alone, once, but only after its views clear the For You bar (free check first). A post the judge passes is then asked a second question about what kind of post it is (`HELD_KINDS`). | Anthropic Haiku 4.5 (`judge`), $1 / $5 per million input / output tokens |
| ↳ embed-images | after each pass | Embeds the pictures of posts that passed, or failed unsurely, or sit in a promoted story. 40 per pass (`group.imagesPerPass`). | Voyage `multimodal-3.1024` (`embed`) |
| ↳ backfill | after each pass | Fetches earlier posts for a promoted story: 2 stories a pass, 100 a day, 20 results each. | X (`search`) |
| **universe** | 1 min [50s] | Reads each new launchpad coin's metadata (free) and checks where the coin came from. Looks up coins by the name of each feed post or narrative: 60 names a pass, 100 a minute, 20,000 a day. | DexScreener and GeckoTerminal (free, not in the ledger); X `lookup` and the judge for linked posts |
| **missed** | 30 s [55s] | Works through the owner's pasted "missed posts". | Learning book |
| **admit** | 1 min | Scores arrivals against a percentile bar (`admit.quantile` 0.88, floor 0.7). **Stalled; no effect** (findings 1 and 6). | none |
| **track** | 30 s [3s] | Re-reads view counts for on-screen posts, judged catch-all posts, and young fast unjudged posts. Up to 250 reads a pass. Tier grid `[4, 9, 14, 21, 30, 42, 58, 78]` minutes, slowed by age (`ageTierMinutes`). Stops after 24h (`maxTrackedHours`). TikTok is re-read as one batch at most every 180 min. | X `lookup` ($0.00015 a post); Apify `observe` ($0.151 a batch of 50; never used yet) |
| **detect** | 1 min [6s] | Burst detection. 0 outputs. | none |
| **group** | 2 min [9s] | Puts each post into a narrative or starts a new one. **Stalled** (finding 1). | none (reads stored vectors) |
| **qualify** | 2 min [12s] | Judges promoted stories; looks only at stories created in the last 12h. | Anthropic |
| **resolve** | 1 min [15s] | Links a story to a coin. | none |

**Project service** (`tools/railway-start.mjs`) runs these jobs one at a time, checking for due jobs every 5 seconds:

| Job | Every | What it does |
|---|---|---|
| `project` (`tools/db.mjs project`) | 1 min | Rebuilds the board, For You and Popular, and writes `public.rising_row` and `public.feed_rejection`. Its `/health` returns 503 if the board has not been rebuilt for 5 minutes. |
| `logos` | 2 min | Copies on-screen coin logos into our storage. |
| `market`, `pairs`, `sources` | 5 min | Market reads, image pairs, and the source health indicator. |
| `label` | hourly | Labels. |
| `retune` | checks every 10 min | Re-tunes the judge at 08:00 and 20:00 London time, starting 20 minutes early (`learning.startAheadMs`). Runs detached. |
| `recall` | daily | Recall check. Runs detached. |

### A post's path to For You

1. **Found.** The X `all` lane asks for posts from the last 10 minutes with at least N likes; N is the floor in section 2. At 13:20 N was 125. TikTok is asked for the 20 most-liked videos of the past day for one watch-list word. The post is stored with its first view reading and credited to a lane in `internal.item_found_by`.
2. **Free reach check.** `score-posts` takes up to 1,000 uncredited-by-judge catch-all posts (200 × 5) that have at least one reading of 2,000 views or more. It keeps only those that clear For You's bar (`clearsReachBar`), up to 200 a pass.
3. **Re-reads.** A young unjudged post gets re-read only while it averages at least 500 views a minute (`rising.watchMinReachPerMinute`) and is under 30 minutes old. That is how a post reaches the early bars (20k views within 15 minutes, or 2,000 a minute within 30 minutes).
4. **Judged.** The judge answers in batches of 10. If a post reuses a picture or reposts a post the judge already answered, it gets a copy of that answer for free; this happened only twice in 24h. A "yes" is then asked the kind question. AD, SPORT, FANDOM, SHOWBIZ or POLITICS holds the post back.
5. **Projected.** Within 1 minute the projector puts every passing post that also clears a feed's bar onto For You and/or Popular, at most 60 rows each.

### What it costs (`internal.spend`)

**Last 24h** (really 22:02 on 12 Oct to 13:24 on 13 Oct), live book by line:

| Supplier : endpoint | Calls | Units | USD |
|---|---|---|---|
| anthropic : judge | 1,288 | 1,229 | 5.29 |
| twitterapi.io : search | 1,606 | 24,433 posts | 3.67 |
| apify : discover | 32 | 32 runs | 1.06 |
| voyage : embed | 3,799 | 6.54M tokens | 0.79 |
| twitterapi.io : lookup (re-reads) | 2,417 | 5,199 posts | 0.78 |
| anthropic : same-subject | 32 | 32 | 0.06 |
| **live total** | | | **11.64** |
| job book (`job:`) | | | 7.22 |
| learning book (`learn:`) | | | 4.30 |

**Four 6-hour windows** (measured at 13:30 UTC):

| Window (UTC) | Live | Jobs | Learning | Total |
|---|---|---|---|---|
| 07:30–13:30, 13 Oct | 4.03 | — | 2.30 | 6.33 |
| 01:30–07:30 | 2.88 | 3.93 | 2.46 | 9.27 |
| 19:30–01:30 (live from 22:02) | 4.85 | 3.30 | — | 8.15 |
| 13:30–19:30, 12 Oct | — | — | — | 0 (not running yet) |

Spend since midnight UTC was $21.75. At 13:19 the runner logged `spentTodayUsd=20.77` against its live cap of $21.50. At the current pace the live book runs at about **$16 a day**.

**Spending limits** (`policy.budget`):

- Live day: `dailyUsd` $21.50.
- Supplier lines: posts (X) $10, judge $7.50, short video $2 plus actors $1 (Apify's line is $3 together), embeddings $1.
- Jobs: $3.75. The jobs judge has spent $6.57 against its $2.75 line; the 95% warning is firing.
- Learning: $10.50.
- Spending stops at 97% of a line (`softStopFraction`). Warnings go out at 80% and 95%.
- X search is planned at $6 a day at target and $13.82 in the worst case, inside its $10 line. These figures come from the log line `catch-all search, priced` (`dayAtTargetUsd`, `dayWorstCaseUsd`, `dailyCapUsd=10`).

---
## 2. Discovery

### X catch-all query (`adapters/platform/x/src/discover.ts`)

Each lane sends one advanced search with no keyword:

```
min_faves:<floor> [filter:media] (lang:en OR lang:zxx) -filter:retweets since_time:<now−window> until_time:<now−1s>
```

- `queryType` is Latest, so results come newest first.
- 20 posts a page (`feed.pageLimit`), up to 3 pages (`feed.maxPagesPerLane`). A further page is fetched only while each page comes back full and the budget allows it.
- Replies and quotes are included; plain reposts are not.
- `zxx` means posts with no language, such as a bare picture.
- Each post costs $0.00015.
- No keyword term is configured: X is asked 2 questions a pass (the two lanes) plus `slow` when due.

### Lanes

| Lane | Runs | Window | Floor stored at 13:20 (`internal.feed_floor`) | Floor sent | Target per pass | Found, 24h | Median age when found (3h) | Oldest found (3h) |
|---|---|---|---|---|---|---|---|---|
| `all` | every pass (2 min) | 10 min | 752 | **125** | 33.3 | 5,857 | 7.6 min | 10.7 min |
| `media` | every pass | 10 min, media only | 1,161 | **194** | 11.1 | 632 | 6.0 min | 10.8 min |
| `slow` | every 15 min | 6 h | 400 | **625** | 83.3 (can only get 60) | 379 | 37.7 min | 53.5 min |
| `watch` (TikTok) | about every 57 min | past 24 h | — | — | 20 videos | 28 credited (of 236 TikToks stored) | about 15 h | 17.7 h |

**How the floor works** (`core/src/sources/feed-floor.ts`):

- **Floor sent.** The stored floor is scaled by window ÷ 60 minutes, then clamped to between 30 and 50,000. For example, 752 × 10/60 = 125. The slow lane is sent at least 5 × the `all` lane's sent floor (5 × 125 = 625).
- **Floor adjustment.** After every pass each lane's stored floor is adjusted toward its target:
  - Over target: raised 20%.
  - Under 60% of target: lowered 20%.
  - Empty: lowered 40%.
  - Held while the other lane is empty.
  - The stored floor never goes below 100.
- **Targets.** Targets come from the posts line: $10 × 0.6 (`budgetShare`) ÷ $0.00015 = 40,000 posts a day. The slow lane gets 20% of these and the rest is split 75/25 between `all` and `media`.
- **Floors rose today.** At 08:21 the stored floors were 296 / 432 / 500 (sent: 49 / 72 / 500). At 12:50 they were 434 / 730 / 500. At 13:20 they were 752 / 1,161 / 400.
- **Results, 08:20–12:51.** `all` averaged 28.0 posts a pass (max 60). `media` averaged 10.9 (max 42). Floor moves: 60 raised, 45 lowered, 156 held.

**Slow lane gap (finding 5).** The lane always returns exactly 60 posts and never reaches 83.3. Because 60 is above 60% of the target, the floor is "held" forever. The 60 newest posts with 625+ likes fill the cap before the search goes back an hour.

### TikTok watch lane

- **How it is asked.** TikTok has no catch-all search, so each ask uses one word from `policy.feed.hashtags`, in rotation: meme, memes, funny, fyp, viral, cat, dog, animals, pet, cute, trend, crypto, memecoin, solana.
- **Search settings.** Since about 13:07 the ask uses the scraper's **video search** (`searchSection: '/video'`) with `videoSearchSorting: MOST_LIKED` and `videoSearchDateFilter: PAST_24_HOURS`, and asks for 20 results (`feed.hashtagResults`; `DISCOVER_RESULTS_CAP` = 20).
- **Price.** A run is quoted at $0.001 + 20 × ($0.003 + $0.0005) = **$0.071**. Before this change it was $0.031 for 10 unsorted results.
- **How often.** `watchEveryMs` = price ÷ (line × `budgetShare`) × 1 day = 0.071 ÷ ($3 × 0.6) × 24h ≈ **57 minutes**, so about 25 asks a day, about $1.80. Before the change it was about 25 minutes; the ledger shows runs 23 to 27 minutes apart.
- **TikTok spend.** Adding one 3-hourly re-read batch ($0.151 × 8 = $1.21) would bring TikTok to about $3.01 a day, right at its $3 line.
- **Age limit.** Only videos posted in the last 24h (`feed.watchMaxAgeMs` = 86,400,000 ms) are credited to the lane (`services/runner/src/work/arrivals.ts`). Older ones are stored but never judged.

### What is thrown away before scoring, and why

| Filter | Where | Why |
|---|---|---|
| Below the lane's like floor | sent to X (`min_faves`) | Controls cost. It does not judge quality. |
| Not English or no-language | `lang:en OR lang:zxx` | Policy `feed.langs`. |
| Plain reposts | `-filter:retweets` | The owner wanted replies and quotes kept. |
| Not credited to a lane | `markFoundBy`. Keyword-term posts and TikToks older than 24h are not credited. | Only posts credited to a lane can be scored or shown. |
| Below For You's reach bar | `score-posts.ts`: free check before the paid judge. The database pre-filter is 2,000 views; then the exact bar applies, ×12.5 for TikTok. | Not worth judging a post that cannot be shown. It waits for its next reading. |
| Same answer already given | reuse in `score-posts.ts` | No paying twice for a repost, an empty quote, or the same picture. |
| Track gate | `track-gate.ts` | No paid re-read for a post whose newest judgement fails even with the best possible picture match. Unjudged catch-all posts are always kept. |

---
## 3. The judge

### 3.1 What it is, in one paragraph

Each post that the catch-all search finds, and that has enough views to reach For You, is sent **once** to an AI model (Anthropic **Claude Haiku 4.5**, `claude-haiku-4-5`) along with its words and its picture. The model is asked one question: *could people make a memecoin about this post?* If the answer is yes, a **second question** goes to the same model: *what kind of post is this?* Ads, sport, fandom, showbiz and politics are held back. Fixed rules in code (`core/src/qualify/post-score.ts`) then decide whether the post passes. The model never sees views, likes or shares.

### 3.2 Model, settings, and which code does what

| Item | Value now | Where |
|---|---|---|
| Model | `claude-haiku-4-5`. Set by the env var **`JUDGE_MODEL`**, which has no default, so the judge stays switched off unless it is set. Every live judgement in the DB is filed under `judge:anthropic/claude-haiku-4-5@…`, which confirms the model | `adapters/judge/anthropic/src/credentials.ts`, `services/runner/src/wiring.ts` `buildJudge` |
| Key | `ANTHROPIC_API_KEY` (name only, value never printed). Both variables must be set, or neither | `services/runner/src/config.ts` |
| Price book | Haiku 4.5: $1 per million input tokens, $5 per million output, $1.25 cache write, $0.10 cache read | `adapters/judge/anthropic/src/prices.ts` |
| Posts per call (live) | **10**. One request carries up to 10 posts as a JSON list, with their pictures appended (`policy.qualify.batchSize = 10`) | `wiring.ts` `maxBatch: policy.qualify.batchSize` |
| Answer budget | 800 output tokens per call (`policy.judge.postBatchMaxOutputTokens`) | policy.ts |
| Text sent | the post's text, trimmed to 400 characters (`policy.judge.maxCharsPerText`) | `score-posts.ts` `subjectFor` |
| Pictures | **The "small" size.** For pictures hosted on `pbs.twimg.com` the judge requests `name=small`, which fits inside 680×680 and is about 25% fewer tokens. Pictures from any other host are sent at their original size. If the small size fails to load, the original is used | `wiring.ts` `pictureSize: 'small'`; `adapters/judge/anthropic/src/pictures.ts` |
| What it "looked at" | Set by our code, never by the model: `videoFrame` when the post has a video (its preview still is sent), `picture` when a still picture was fetched and shown, `text` otherwise. A video with no frame we can show is never sent and comes back "unavailable" | `adapters/judge/anthropic/src/to-judgement.ts` |
| Calls per pass | up to 200 posts per discovery pass (`policy.rising.postsPerPass`), newest first. A post is only queued once its views clear a For You bar (`clearsReachBar`). TikTok posts face bars 12.5× higher | `services/runner/src/work/score-posts.ts` |
| Reuse | A plain repost, a quote that adds no words and no picture, or a post whose first picture is the same file as one already judged gets a copy of the earlier answer at $0 (`copied_from`). Last 24h: only 2 copies | `score-posts.ts` `reusableFor` |
| Prompt caching | The fixed instructions carry a cache mark. Haiku 4.5 only caches prompts of at least 4,096 tokens, and the post prompt is about 1,600, so **live calls currently get no cache discount**. The code comment says the same | `client.ts`, `prices.ts` |

**What it outputs**, per post (structured JSON, `ANSWER_SCHEMA` in `adapters/judge/anthropic/src/client.ts`):

- `coinable`: yes or no
- `confidence`: a number from 0 to 1, clamped to that range by our code
- `proposedName`: the name or ticker, or null
- `alternateNames`: a list of other names. **The schema requires this field, but the post prompt never asks for alternates or explains them.** The model fills it anyway, usually with 1–2 names (see §7).
- `subjectSpans`: a list of short strings, also required by the schema and never explained in the post prompt
- `looked_at` and `cost_usd`: added by our code. `looked_at` is text, picture or videoFrame. `cost_usd` is the call's cost divided evenly across the posts in it.

Everything is stored in `internal.post_judgement`. The newest row for a post is the one that counts.

### 3.3 How the prompt is assembled (exactly what the model receives)

`adapters/judge/anthropic/src/prompts.ts` loads the `.txt` files once at start-up and hashes them. `src/index.ts` (`QUESTION` / `promptFor`) picks the set. `src/client.ts` (`buildRequestBody`) builds the request:

1. **System prompt**: `post-system.txt`, word for word.
2. **User message, block 1** (carries the cache mark): `post-coinability.txt`, word for word. A learned-examples block would be appended here after a blank line. Right now none is (§3.6).
3. **User message, block 2**: the literal line `Stories to judge, as JSON. Answer once for each storyId.` followed by a JSON list. Each entry is `{storyId: "post-<id>", posts: [<text>], distinctAuthors: 1, distinctSources: 1, frame: "shown below" | "none"}`.
4. For each post with a picture: the line `Picture for storyId <id> (a still picture, or a frame from its video):` followed by the image.
5. The output is forced to the JSON schema above.

Prompt hashes recorded on every judgement:
- Post question: `post-coinability@ef54be97`. This is the hash of the two files on disk now, and every live post judgement since 2026-10-13 01:43 carries it.
- Kind question: `post-kind@4f623ab8`. In use since 06:22.

#### Post question: system prompt (`adapters/judge/anthropic/prompts/post-system.txt`), verbatim

```text
You judge ONE social media post, and its picture when one is shown, for one question:
could people make a memecoin about it? That is: is there one clear thing in this post —
usually in its picture — that a coin could be named after and pictured with?

This is a naming task, not advice: you are not predicting whether anything will be
popular or valuable. A post about a market, a company or a token is judged like any other
post — by whether one nameable thing is at its centre. Answer only from the post and its
picture. Do not use what you know from elsewhere, and do not guess what happened next.

Return the structured output. Do not add commentary outside it.
```

#### Post question: instructions (`adapters/judge/anthropic/prompts/post-coinability.txt`), verbatim

```text
Each item below is one post (one storyId, one text). Answer for each.

WHAT A COINED POST LOOKS LIKE
Of 77 posts people actually made coins about, labelled by hand:
  - the picture carries it (82%): the image alone could be the coin's picture
  - there is a character (74%): an animal, a creature, a person turned into a character,
    a mascot, an object with a face or a story
  - it has a one-word name (69%), or a name that reads as one (MOODENG, CHILLGUY)
  - most were NOT jokes (only 36% were funny): serious news and striking real things
    were coined as often as jokes
So the question is not "is this funny" and not "is this a meme yet". It is: is there one
nameable thing here that people could rally a coin around?

YES — one nameable thing, in the words or the picture:
  - an animal or creature with a look, a name or a story (a pygmy hippo, a cat with a job)
  - a character: drawn, from a game or a show, or a person turned into one
  - an object or an absurd real thing (the most expensive cow ever sold, a fish given a
    roller coaster, a cat with its own casino)
  - a phrase or a line people repeat, with or without a picture
  - a product everyone is posting about by name (a new AI model, a gadget)
  - a company's moment (a company joining an index, a launch people repeat)
  - one person's single viral moment (a prize-winner's arrest, a chart a leader held up,
    a line someone said that everyone quotes, a bet someone made in public)
  - a crypto or AI story with a name: a rug pull, an airdrop, a token or model launch, a
    market moment, a product coming to crypto, a crypto figure's line or picture the
    timeline repeats, a joke or pun about a coin or its holders
  - a rallying cry around one person or thing ("JUSTICE FOR ...", "FREE ...", "HELP ..."),
    including a fundraiser or a call to help one named person
  - a picture that is itself the thing. WHEN THE WORDS SAY ALMOST NOTHING — a link, one or
    two words, "good morning", "Always trust" — THE PICTURE DECIDES: if it shows one clear
    figure, creature, character, face or object, answer yes and name what it shows. Only a
    picture of nothing in particular (a crowd, a screenshot of text, scenery) is a no
News IS a subject when one of the things above is at its centre. Name the thing, not the
headline. The test between a person's viral moment and general political news: is the post
about ONE named person or thing and what they did or what happened to them (a viral moment
— yes), or about a policy, an ideology, a party or an election (general — no)?

NO — nothing to coin:
  - sport: a score, a result, a highlight or clip, a transfer, a player's stats or news
    about a player or a team
  - fan accounts: an idol's photo or story, a fancam, a comeback, an album, tour or airport
    update, a streaming or voting goal
  - general political news with nothing in it to name: a policy, an ideology, a party, an
    election, a debate about society
  - a post with nothing specific in it: a mood, a generic photo, a list, an ad
Sport, fandom and politics turn into a YES only when there is a meme subject inside them:
an animal on the pitch, a fan's sign, an object thrown on stage, one line or picture that
people repeat.
Judge what the post shows, never who posted it.

THE NAME
Give a ticker-style name of at most three words, as the post or its picture would have
people say it: the cow is CARINA, the fish on a roller coaster is FISHCOASTER, the hippo
is MOODENG. Not a description of the post. If you cannot name it in three words, answer no.

HOW SURE ARE YOU?
A number between 0 and 1, given only this post. High when one thing clearly stands out;
low when the post could be about several things.

Do not consider how many people saw the post. Numbers of views, likes or shares are not
shown to you, deliberately: attention is measured elsewhere.
```

#### Second question, "kind": system prompt (`adapters/judge/anthropic/prompts/post-veto-system.txt`), verbatim

```text
You sort ONE social media post, and its picture when one is shown, into a kind. You are not
judging whether it is good, funny or popular. Answer only from the post and its picture.

Return the structured output. Do not add commentary outside it.
```

#### Second question, "kind": instructions (`adapters/judge/anthropic/prompts/post-veto.txt`), verbatim

```text
Each item below is one post (one storyId, one text). For each, decide whether it is one of
these kinds. Judge what the post shows, never who posted it.

  AD       someone promoting THEIR OWN thing to their own audience: a game or app's own ad
           (download now, an event or banner, a new or free character, patch or update notes,
           a wallpaper, a game character's birthday); a brand's own product, merch or show
           promotion; a sponsored code, raffle or giveaway; a post shilling a token ("$TICKER is
           the next...", "been looking into $X", hype for a coin someone is selling); a
           project's routine status update to its followers; a stream schedule or
           "going live" post; a creator's own art, comic, doodle, selfie or cosplay.
           NOT AD: news ABOUT a company, a product, an animal, a zoo or a person, told as news
           ("X has named...", "X will be added to...", "X just solved..."): that is OTHER
  SPORT    a score or result, a game highlight or clip, a player's stats or news, a team or
           league account celebrating or promoting its own game, wrestling storylines
  FANDOM   fans posting about an idol, a game, a show or a franchise: an idol's photo or
           story, a fancam, a comeback, a tour or airport update, a streaming or voting goal,
           an idol's birthday; fan art and fan comics, game tips and builds, leaks and
           datamines, theories, character discussion
  SHOWBIZ  film, TV, music and celebrity news: casting, release dates, ratings, award
           submissions, anniversaries, a creator's own milestone
  POLITICS a politician's statement, a policy, an election, a party, an ideology
  OTHER    none of the above

Answer with the fields of the structured output like this:
  coinable      true when the kind is OTHER; false for every other kind
  proposedName  the kind, exactly one of: AD, SPORT, FANDOM, SHOWBIZ, POLITICS, OTHER
  confidence    how sure you are of the kind, 0 to 1

ALWAYS OTHER, whoever posts it: an animal or a creature with a story; a record or a striking
real thing (the most expensive cow ever sold); a rallying cry around one person ("JUSTICE
FOR ..."); one person's viral moment or a bet or stunt (a bounty tattoo); something that
HAPPENED in crypto or AI (a rug pull, an airdrop, a model doing something remarkable, a
company joining an index); a crypto figure's line or picture the timeline repeats; a slogan
or a meme picture, a joke or a line people repeat — even from a brand,
team or league account. If the post itself is that moment rather than the promotion, the
result or the celebrity news, its kind is OTHER.
```

`HELD_KINDS` (`services/runner/src/work/score-posts.ts`): `AD`, `SPORT`, `FANDOM`, `SHOWBIZ`, `POLITICS`. The decision is made on the **kind named** in `proposedName`, upper-cased and trimmed, and not on the yes/no field next to it.

### 3.4 Pass or fail: the exact rules

**Step A: is the post asked at all?** It must have been found by a catch-all lane, be observed (not seeded), have been posted within the last 24h (`rising.windowMs`), and clear a For You views bar (`clearsReachBar`): 40k views, **or** 20k within 15 minutes of posting, **or** an average of 2,000 a minute at some reading in its first 30 minutes. TikTok's bars are 12.5× these. Until then it waits unjudged. A post fetched by a story's backfill is judged without the views bar.

**Step B: the first question** (`postScoreReason` in `core/src/qualify/post-score.ts`), in this order:

1. No answer yet. This covers a dry run, a budget pause, a vendor outage, and a video with no frame. The post **waits**: it is not shown and not counted as a fail. Reason `not_scored`.
2. `coinable = no`. The post fails with `not_memeable`, **unless the picture boost applies**. The boost needs a "no" given with confidence **≤ 0.30** (`rising.boostMaxNoConfidence`) **and** a picture whose cosine similarity to some picture of a really-coined post is **≥ 0.44** (`rising.boostMinSimilarity`). In that case the post **passes**.
3. `coinable = yes` but the proposed name is empty or **more than 3 words** (`rising.maxNameWords = 3`). Words are counted as runs of letters or digits, so "J-HOPE" is 2 words and "CHILLGUY" is 1. Fails with `name_test`.
4. `coinable = yes` with confidence **below the bar** (`qualify.minJudgeConfidence`): 0.70 when it looked at text, 0.70 for a picture, 0.75 for a video frame. Fails with `judge_unsure`.
5. Otherwise the post **passes**.

**Step C: the kind question** (live runner only). Posts that passed step B **in that pass** are sent the kind question, in batches of 10. If the reply names AD, SPORT, FANDOM, SHOWBIZ or POLITICS, that kind answer (coinable=false, name = the kind, confidence = how sure it is of the kind) is **saved as the post's newest judgement**. The post then fails step B as `not_memeable`. An OTHER answer saves nothing. If the kind question errors or is paused for budget, the first answer stands.

**Step D: the owner's call wins** (`services/project/src/owner-calls.ts`, read on every projection). If the owner tapped "Not a meme", the post fails with `owner_not_meme`. If the owner tapped "Is a meme", it passes the per-post rule even with no judgement at all.

**Step E: each feed's views bar at projection** (`services/project/src/rising.ts`). For You uses the bars above. Popular needs 250k (3.1M for TikTok). A post whose views were never read appears on neither feed. Reasons `under_reach_bar` and `reach_unread`.

**Every reason a post is off the feed** (`store/src/repo/feed-rejections.ts`): `not_scored`, `not_memeable`, `judge_unsure`, `name_test`, `under_reach_bar`, `reach_unread`, `owner_not_meme`. A narrative can also fail with `too_few_passing_posts`. A post held by the kind question is recorded as plain `not_memeable`. **No rejection row says "held as an ad or sport"**; you can only see it from the stored judgement.

**How the picture boost works** (`services/project/src/rising.ts` `loadCandidates`). The projector compares the post's picture embedding (`internal.image_embedding`) with every row of **`internal.coined_picture`** in the same embedding space and takes the closest match. `coined_picture` currently holds **29 pictures**, filled once by `tools/coined-pictures.ts` from the gold set's origin posts. The similarity number is used only inside the query and never reaches a screen. The bars were tuned on 2026-10-13 on the same gold pictures: gold went from 27 to 29 of 41 and no recorded non-meme got through. The policy comment warns to expect less on new posts.

Edge cases worth knowing:
- **Live, boosted posts are never asked the kind question.** Step C only checks posts that passed *without* the picture, so a boosted ad or sport post can reach the feed. The re-tune's test (`tools/lib/pass-rates.ts`) *does* ask boosted posts the kind question, so the test is slightly stricter than live.
- A held-kind answer is also stored as a "no" with a confidence. If the model was **unsure of the kind** (≤ 0.30) and the picture looks like a coined post, the boost would let the post through.

### 3.5 Live numbers, last 24h (2026-10-12 13:24 → 2026-10-13 13:24 UTC)

From `internal.post_judgement` and `internal.spend`:

| Measure | Value |
|---|---|
| Posts judged with today's post prompt (`post-coinability@ef54be97`, since 01:43) | **2,366** (2 of them copies at $0). Looked at: picture 1,001 · text 753 · video frame 612 |
| Said coinable | **660 = 27.9%** |
| Passed the first-question rules (coinable, confident enough, name of 1–3 words) | **613 = 25.9%**. Failed: confidence too low 45, name test 2, "no" 1,706 (72.1%; 540 of those were unsure "no"s at ≤ 0.30) |
| Kind question (live since about 06:25) | 502 posts passed the first question after 06:25. **74 were held = about 15%** of them (3.1% of all judgements): **AD 33, SPORT 23, SHOWBIZ 10, FANDOM 8, POLITICS 0** |
| Two stray OTHER rows | 2 kind answers labelled `OTHER` were saved at 06:23–06:24, before the rule "only held kinds are saved" was live. Because they are stored as coinable=false, those 2 posts are hidden as `not_memeable` by accident |
| Picture boost, roughly | Of 577 posts whose newest answer is an unsure "no", 190 have a picture embedding and **41 are ≥ 0.44 similar** to a coined picture, so they pass the per-post rule. They still need the views bar to be shown |
| Older answers in the window | Before 01:43 the feed used the **narrative** prompt for single posts: `coinability@780b3231`, 878 posts, 23:23–00:07, 11.2% coinable, $0.80. Separately, a one-off job re-judged 226 posts (`coinability@c6dd982d`, booked as `job:judge`) |
| On the feed right now | 13 distinct posts in `public.rising_row` |

**Cost** (vendor `anthropic`, endpoint exactly `judge`; `job:` and `learn:` lines excluded): **$5.29 over 1,288 calls in 24h**. That breaks down as:
- post question (today's prompt): $3.48, about $0.0015 per post
- held-kind answers: $0.19. OTHER kind answers are not saved, so their cost only appears in the spend total.
- the overnight single-post judging with the narrative prompt: $0.80
- the remaining ~$0.8: narrative (QUALIFY) judging and the unsaved kind answers

Since 07:00 the hourly rate has been $0.21–0.31, which is about **$5–7 a day**. For comparison over the same 24h: `job:judge` $6.56 and `learn:judge` $3.98.

### 3.6 Is a learned-examples block in use live? **No.**

`internal.judge_examples_version` has **one row (version 1), and it is `applied = false`**. The reason recorded is "gold 55/118 is under 65%; live non-memes 13/89 is over 5%". The live runner (`services/runner/src/learned-examples.ts`, wired in `services/runner/src/wiring.ts`) loads the newest **applied** row at boot and checks again every 5 minutes. No row has ever been applied, so **the live judge runs on its written rules alone**. The DB agrees: every live post judgement carries the bare-rules hash `ef54be97`, and an applied block would change that hash.

Stale comment: `adapters/judge/anthropic/src/index.ts` says the examples are "re-tuned hourly". The schedule is actually twice a day (§10).

### 3.7 Where the live judge differs from how it is tested

- **Batching:** the live feed sends **10 posts per call** (`maxBatch = qualify.batchSize = 10`). The re-tune and `tools/feed-pass-rates.ts` send **1 post per call** (`maxBatch: 1` in `tools/lib/pass-rates.ts`). Its header says "one post a call, as the live scorer asks", which is **not true** of the live scorer. Answers can differ when a post is judged alongside nine others.
- Boosted posts and the kind question: see §3.4.

---
## 4. The feed rules

### Reach bars (`core/src/qualify/reach-bar.ts`, `policy.rising`)

| | X post | TikTok (×12.5, `shortVideoReachScale`) |
|---|---|---|
| **For You** | 40,000 views (`fypMinReach`), **or** 20,000 views read within 15 minutes of posting (`fypEarlyReach`), **or** an average of 2,000 views a minute at any reading within 30 minutes of posting | 500,000; or 250,000 within 15 minutes; or 25,000 a minute within 30 minutes |
| **Popular** | 250,000 views | 3,125,000 |

- **Growth is measured from post time**, never from when we first saw the post. A post that cleared a growth path stays cleared.
- **An unread count does not pass.**
- **The growth paths cannot help TikTok in practice.** Videos are found when they are about 15 hours old, so only the total-view bar applies.

### Per-post score (`core/src/qualify/post-score.ts`)

Checks run in this order:

1. **No judgement:** the post waits (`not_scored`).
2. **Judge says "no":** the post fails (`not_memeable`). The exception is an unsure no (confidence ≤ 0.3) whose picture is at least 0.44 similar to a coined post's picture. That post passes (the picture boost).
3. **Bad name:** a "yes" whose name has 0 words or more than 3 fails (`name_test`).
4. **Confidence too low:** a "yes" below `qualify.minJudgeConfidence` fails (`judge_unsure`). The bar is 0.70 for text and pictures and 0.75 for video frames.

The owner's "Not a meme" call overrides a pass, and "Is a meme" overrides a fail (`owner-calls.ts`). The reach bar still applies either way.

**Second question (`HELD_KINDS`).** A post the first question passed is asked its kind. AD, SPORT, FANDOM, SHOWBIZ or POLITICS files a "no". In the last 24h it held back AD 35, SPORT 23, SHOWBIZ 10, FANDOM 8 and POLITICS 0. Judged posts in 24h: 3,054 distinct, 640 of them coinable. This includes job re-scores.

### Sort order (`services/project/src/rising.ts`)

- **For You:** newest post time first. Ties are broken by id.
- **Popular:** most views first. Posts with unread views go last. Ties go to the newest.
- **Both feeds:** at most 60 rows each (`rising.maxRows`), from a 1-day window (`rising.windowMs`).
- **The client never re-sorts,** except in the mobile Popular tab, which separates the "uncounted" rows.

**Live at 13:27 UTC:**

| Feed | Rows | Source | Age of posts |
|---|---|---|---|
| For You | 13 | all X | youngest 0.5h, median 10.8h, oldest 15.9h |
| Popular | 6 | all X | youngest 1.2h, median 14.0h, oldest 15.9h |

### Why posts were hidden (`public.feed_rejection`)

These are counted only when a post that **was showing** left the feed:

| Reason | Last 24h |
|---|---|
| `not_memeable` | 71 |
| `under_reach_bar` | 51 |
| `owner_not_meme` | 38 |
| `reach_unread` | 9 |
| narrative `too_few_passing_posts` | 2 |

**Owner calls** in `internal.owner_feedback`, last 24h:

- Post "not": 73. The latest was at 13:09 UTC. Only 38 of these appear above, because a call on a post that was never showing records no rejection.
- Post "meme": 2.
- coin_match "none": 1.
- coin_match "right": 1.

A narrative stays on the board only while at least 2 of its posts pass (`rising.narrativeMinPassingPosts`, `services/project/src/narrative-rules.ts`). Only the passing posts are listed. The picture boost does not count here.

### Watchlist

The Watchlist tab is **narratives the user starred, stored only on that device**: `insidor.mobile.watchlist.v1` on mobile, `insidor.watchlist.v1` on web. There is no server list and no account sync (`mobile/src/shared/watchlist.ts`, `app/src/features/watchlist/watchlist-store.ts`). It shows starred rows in the board's order, and counts starred narratives that are not on the board. On web, a mint alert fires when a starred narrative's coin link settles. With no narrative on the board right now (finding 2), every Watchlist shows its stars as off-board. Posts cannot be starred.

---
## 5. Narratives

### How posts are grouped (`core/src/group/stage.ts`, runner `groupWork`)

**Which posts GROUP looks at.** Each pass takes posts that are not yet in a story; ADMIT does not gate this. It compares them, oldest first, against every open story from the last 12 hours, up to 200 stories. Real posts are only compared with real stories.

**Match score.** The score is a weighted sum of signals:

| Signal | Weight |
|---|---|
| lineage (quote or reply chain) | 0.60 |
| image carrier | 0.55 |
| exact carrier | 0.40 |
| text carrier | 0.35 |
| representation (picture vector or judge name) | 0.35 |
| time proximity | 0.10 |
| source agreement | 0.05 |

A post joins a story when its score clears 0.30 (`matchBar`) and beats the runner-up by at least 0.05 (`adjudicationBand`). Picture vectors count as near-identical at 0.95, likely at 0.82 and borderline at 0.72. A borderline picture with a matching judge name is sent to the judge's same-subject check, up to 200 a day.

**Outcomes:**

| Outcome | Codes | What happens |
|---|---|---|
| Join | M0 carrier, M1 lineage, M2 semantic | The post becomes a member of the story. |
| Drop | M3 below bar, M5 generic carrier | The post **starts its own new story** (`owesASeed`). |
| Abstain | M4 ambiguous | No seed. |
| Abstain | M6 no candidate | The post starts its own new story. |
| Hold | M9 member cap (200) | No change. |

**Live, last 24h:** M3 787, M0 10, M1 6, M6 1. No M2 picture join has ever been recorded. 775 of 788 stories have exactly one post.

### Promotion and states (`core/src/group/promote.ts`, `membership.ts`, `contracts/src/story.ts`)

**States:** `candidate` → `promoted`. `merged` and `closed` exist in the type, but nothing moves a story into them in the live path.

**Promotion** happens on the GROUP write once a story has at least 3 members, at least 2 distinct authors and at least 1 source. Nothing else is checked: no age, score or engagement.

**After promotion:**

1. **QUALIFY** judges each promoted story created in the last 12h. A pass writes the story's name and picture.
2. **Board check.** The board then also requires 2 passing posts.
3. **RESOLVE** links a coin.

**Live at 13:28 UTC (`public.story`, all observed, none merged):**

| State | Count |
|---|---|
| candidate | 786 |
| promoted | 2 (`story_seed-x-2109772152312680726`, `…2109777709488619965`; promoted 22:58 and 23:00 UTC on 12 Oct; 3 members each) |
| merged / closed | 0 |

**QUALIFY, last 24h:**

- 18,803 `S0_not_applicable`: candidates, which QUALIFY skips.
- 356 `Q4_unnameable`: the same 2 stories, re-decided every 2 minutes.
- 9 `Q11_judge_unavailable`.

Its last non-empty pass was at 11:04 UTC, when both stories passed the 12-hour window. **The narrative layer is effectively frozen until GROUP's pull is fixed (finding 1).**
## 6. Coin matching: which coin, if any, belongs to a feed post

### 6.1 Where coin data comes from

| Source | What it gives | Where |
|---|---|---|
| **Mint stream (chainwatch)** | Every new launchpad coin as it is minted, written to `public.asset` with origin `live_stream`. Runs as its own Railway service, so a runner deploy does not leave a gap in what it saw. | `services/chainwatch` (variable `MINT_STREAM_URL`) |
| **DexScreener search** | Free name search, used for the post's coin lookup. Also gives market standings, which decide which coins are kept. | `adapters/market/dexscreener`, wired in `services/runner/src/wiring.ts:632` |
| **GeckoTerminal / CoinGecko on-chain API** | The second name search, plus the price candles behind the "older coin" signal. On Railway the runner has `COINGECKO_API_KEY` and `COINGECKO_API_PLAN` set, so it uses the owner's paid CoinGecko plan (500 calls a minute, shared with the read service and the projector). Without a key it would fall back to GeckoTerminal's free API. | `adapters/market/geckoterminal`, `services/runner/src/config.ts:498` |
| **Launch metadata** | Each coin's metadata document: picture, its X/website/Telegram links and description. Read through the launchpad's own gateway, with Pinata as fallback. ipfs.io was dropped because it answered every burst with 429 errors. Live: 7,360 read, 5,164 not content-addressed, 395 unavailable. | `internal.coin_metadata` |
| **Chain (Helius RPC)** | For a coin the search found but the stream never saw: its metadata link and true launch time. The runner has `HELIUS_API_KEY`. | `coin-lookup.ts` `chain` |

Live universe: 12,558 coins from the mint stream and 398 added by search (`backfill`).

### 6.2 The lookup: searching a post's name (`services/runner/src/work/coin-lookup.ts`)

The judge names the subject of each For You or Popular post. The runner then searches that name on both sources. Narratives' names are searched too. Limits from `policy.universe.lookup`:

| Setting | Value | Meaning |
|---|---|---|
| `termsPerPass` | 60 | names searched per one-minute pass |
| `perMinute` | 100 | rolling-minute cap, a fifth of the paid plan's 500 a minute |
| `perDay` | 20,000 | per UTC day |
| `keepMs` | 6 h | a name is not searched again within 6 hours |
| `withinMs` | 7 days | only coins launched in the last 7 days become candidates |
| `hitsPerTerm` | 10 | candidates kept per name, newest first. Beyond that the card says "10+" (`more`) |
| `retries` | 3 | a search that failed on both sources is retried, 4 tries in total |
| `giveUpMs` | 10 min | if no search has finished 10 minutes after the post entered the feed, the card shows "Couldn't check" |

The rule: **a search result never links anything by itself.** The search only adds coins to the universe as candidates. If both sources fail, the failure is recorded as a failure, never as "nothing found". In the last 24 hours, 159 post searches finished and 54 of them returned coins. No pending lookup currently has a failed attempt.

### 6.3 The evidence rule (`core/src/match/post-coin.ts`, applied in `services/project/src/post-match.ts`)

Every coin the search returned is weighed on four kinds of fact, strongest first:

1. **Links this post**: the coin's own metadata (X link, website, chat link, description) contains the post's id or its picture's id as a whole token. Ids shorter than 8 characters are ignored.
2. **Same picture**: the coin's picture is near-identical to the post's, or borderline and the judge said "same subject". Uses the group-image bars and the Voyage image vectors.
3. **Launched after**: the coin launched after the post (10 minutes' grace, `origin.launchAllowanceMs`) and within 48 hours of it (`universe.matchWindowMs`).
4. **Name**: the subject's names compared with the coin's name and ticker.
   - `exact`: equal once spaces, punctuation and a leading "the" are removed.
   - `variant`: a looser spelling, needing at least 4 letters. This means the same skeleton with doubled letters squashed and vowels dropped (Peanut and PNUT), or one name inside the other.

**The app shows a confident match ("matched", with Buy) only in these three cases:**

- exactly one coin **links this post**; or
- exactly one coin has the **same picture and launched after** the post (inside the window); or
- the coin **launched after** the post, inside the window, with an **exact name**, and it is the **only coin of that name the search found at all**, whenever launched, and its picture is not known to differ.

Every other case is **unsure**. That includes two coins linking the post, two with the same picture, any `variant`-only name match, and an exact name when other coins share the name.

- **Name variants never make a card confident.** They only move a coin up the "See N coins" list.
- **A coin launched before the post (beyond 10 minutes' grace) can never be the post's coin.** It is ranked last and can never be confident.
- **Unknown is not no.** Unread metadata, an uncompared picture or an unknown launch time counts neither for nor against a coin.
- The card shows only the facts behind a match: "Links this post", "Same picture", "Launched 12m after". The similarity score and the rule's internal reason stay on the server.

Some of these settings are not yet in `contracts/src/policy.ts`. The values `MIN_LINK_ID_LENGTH = 8`, `MIN_VARIANT_LENGTH = 4` and the Create name limits are constants in `services/project/src/post-match.ts`, and that file's header says they are "to move into policy". They equal the gold eval's `MATCH_BARS`, so the rule that was measured is the one deployed. `post-coin.ts` and `post-match.ts` have not changed since the measurement commit `a72648c`.

### 6.4 "An older coin is trading on this": a separate signal, never a Buy

Coins the search returns that launched **before** the post are written to `internal.older_coin` (by `services/runner/src/work/older-coins.ts`). Next to them the runner stores whether each coin's trading jumped after the post. It never puts them in the post's coin slot.

- First, the coin's name or ticker must match the subject's (`nameMatch`, exact or variant).
- Then the runner reads GeckoTerminal candles in 15-minute buckets, from the day before the post to 6 hours after.
- The coin counts as **running** if its busiest hour after the post traded at least **5×** its usual hour (the average over the day before), or its price **doubled**. Either way, that hour must also have traded at least **$1,000**.
- It is re-checked every 15 minutes while the post is on a feed, 20 coins a pass.
- These bars are constants in that file, not in policy.

`services/project/src/older-coins.ts` puts the running coins on the card as `runningOlder`, at most 10 per post. They have no Buy button, only a link to the coin's page.

Live state: 90 older coins measured and not running, **2 running**, 56 rejected because the name differs, and 29 waiting. Four feed cards currently carry an older-coin line: 2 on For You and 2 on Popular.

### 6.5 Card states, and what is live right now

The feed card logic is in `services/project/src/rising.ts` `postCoins`; the app's wording is in `mobile/src/shared/coin-status.ts`. If the post belongs to a promoted narrative, the narrative's own link decides, and that is the only source of `one` and `several`.

| State | When | Card says | Button |
|---|---|---|---|
| `one` | the post's narrative has one coin | "$TICKER, cap" | Buy |
| `matched` | the evidence rule settled one coin | same line, plus why ("Links this post"…) | Buy |
| `unsure` | searched, coins exist, none settled | "N coins use this name. We can't tell which one came from here." | See N coins, Create |
| `none` | searched, no coins | "No coin yet" | Create coin |
| `pending` | not searched yet | a dash | none |
| `unchecked` | no search finished within 10 min | "Couldn't check" | none |

**Live count from `public.rising_row`, 2026-10-13 13:24 UTC.** For You had 13 rows and Popular had 6.

| Feed | one | matched | unsure (See N coins) | none | pending | unchecked |
|---|---|---|---|---|---|---|
| For You | 0 | **0** | 8 (counts 1–10; four of them "N+") | 3 | 2 | 0 |
| Popular | 0 | **0** | 4 (counts 1–8; one "1+") | 1 | 1 | 0 |

**No card on either feed currently shows a confident coin or a Buy.** Every post with coins is "See N coins". Three For You and two Popular cards show "8 coins", so several of these are probably the same subject. This matches the measured miss rate below.

### 6.6 Gold numbers: right, wrong, missed

**Source:** `eval/src/gold/matching-report.md`. It is rendered by `eval/src/gold/run-matching.ts` from `eval/src/gold/match-evidence.json`, which was gathered by `tools/gold-match-fetch.ts`. Committed in `a72648c` on **2026-10-13 07:08 +01:00** (06:08 UTC). The commit message reads: "the evidence rule finds 7 of 71 origin coins with 6 wrong links in 353 posts (1.7%)". There is no later measurement in the repo.

The test covered 353 of 354 posts with a finished search: 71 origin posts (posts a coin was made from) and 282 whose right answer is "no coin" (7 that went viral after their coin, 275 unrelated).

| Evidence rule (deployed), all posts | Number |
|---|---|
| Origin posts matched to the **right** coin | **7 of 71 (9.9%)** |
| Origin posts matched to a **wrong** coin | 1 of 71 (1.4%) |
| Origin posts **missed** (unsure or none) | **63 of 71 (88.7%)** |
| No-coin posts wrongly given a coin | 5 of 282 (1.8%) |
| **Wrong links, all posts** | **6 of 353 (1.7%)** |

By reason, the 13 confident answers were 6 "links this post", 0 "same picture and timing" and 7 "timing and the only exact name".

For comparison, the looser "one coin with the exact name" rule found 14 of 71 but made 51 wrong links (14.4%). The right coin was among the search results for only 25 of the 71 origin posts, so search coverage limits recall. The held-out half alone (22 origin posts) got 1 right, 0 wrong and 21 missed.

**Caveats the report itself states:**

- The subject's name was taken to be the coin's own name, which is the best case for finding the right coin. Live, the judge's name will often differ, so real recall is likely lower.
- Only 24 picture comparisons and 1 judge answer were available.
- Metadata was read for only 31 of 68 coins.

The "same picture" route has therefore not been shown to work on the gold set.

**Compared with what the owner asked for:** wrong links are low (1.7%), but the rule misses about 9 in 10 origin coins, and on the live feeds right now it settles none.

---
## 7. Name suggestions and Create

### 7.1 Where the suggested names come from

**No separate AI call.** The names are the **same per-post judge answer from §3**: `proposedName` first, then `alternateNames`.

1. On each projection run (every minute), for every post currently on For You or Popular, `services/project/src/post-match.ts` takes the post's **newest** judgement (`loadNameIdeas`, then `postNameIdeas`).
2. It drops any name that is, or contains as a word, one of **~50 hard-coded brand names** (iphone, nike, pokemon, tiktok, …; `BRANDS` in the same file).
3. It turns each remaining name into a ticker with `core/src/create/name-ideas.ts` `nameIdeas`:
   - **one-word names go first**
   - a ticker is 3–10 capital letters or digits
   - a phrase that is too long uses its first word if that word is 3–10 characters, otherwise its initials, otherwise the first 10 characters cut off
   - two names that give the same ticker count as one
   - **at most 5** ideas
4. The result is written to **`public.post_name_idea`**, and the whole table is replaced every run. Only posts on a feed get ideas.

`GET /post/:id/names` (read service, `services/read/src/search.ts` `postNames` and `services/read/src/name-ideas.ts`) returns those ideas. It answers 404 when the post has none or is not on a feed. Each idea carries its ticker status, and the list is sorted **Free first, then Couldn't check, then Taken**. Within each group the projector's order is kept. The reply is held in memory for 1 minute, and each client is limited to 60 requests a minute.

**What the live data shows (13 posts on the feed now):**
- 7 posts have 3 ideas, 4 have 2, and 2 have none. **No post has 4 or 5.**
- Over the last 24h, the 660 "yes" answers carried 0 alternates 6 times, 1 alternate 171 times, 2 alternates 482 times, and 3 alternates 5 times.
- Some ideas show the weak spots:
  - `"the Buck Me ad"` became ticker **`THE`** (the first word was long enough to be used)
  - `ROLLERDEATH` was cut to **`ROLLERDEAT`**
  - `SIXFLAGS TRAGEDY` became `SIXFLAGS`, a brand that is not on the brand list

### 7.2 How a ticker is checked ("Free" / "Taken by N coins" / "Couldn't check")

Each ticker is checked **live, when someone opens Create**, against three sources at once:
- our own index of every launch the mint stream saw, **`public.coin_name`** (exact ticker match, case ignored)
- **DexScreener** search (`adapters/market/dexscreener/src/search.ts`), kept only to Solana pump.fun curve and pumpswap pairs
- **GeckoTerminal** search (`adapters/market/geckoterminal`, `network=solana`)

Both outside searches are free, keyless sources, sent through the Search tab's 1-minute cache and per-client limit.

The rule (`tickerUse`):
- **Taken by N** (shown as "N+" when the list was cut) when any source has at least one coin with exactly that ticker
- **Free** only when none has one **and every source answered**
- **Couldn't check** when a source was down and nothing was found

An outage is never shown as "Free".

- `GET /ticker/:symbol/coins` lists the coins using the ticker: up to 30, newest first, with name, the coin's own picture, market cap and launch time. This is what tapping "Taken by N coins" opens. When the user **types** a ticker, the sheet checks it through this same route after a short pause (`create-form.ts` `useOfList`), so typed tickers also get the live check.
- `GET /ticker/:symbol` (with no `/coins`) only counts our own index ("seen"). The app's API client exports `fetchTickerSeen`, but the Create sheet does not use it.

### 7.3 The Create sheet (`mobile/src/features/create/`)

`CreateCoin.tsx`, `NameSuggestions.tsx`, `create-form.ts`:

- **Fields start filled** with the top idea, meaning the first after sorting, so a Free one comes first. With no ideas, the name is the post title and the ticker is taken from it (`suggestTicker`).
- The other ideas appear as **one row of chips** with no status label. Tapping a chip fills the name and ticker, and the status line under the ticker updates. The name is limited to 32 characters and the ticker to 10.
- **Dev buy presets: 0, 0.5, 1, 2 SOL**, or a typed amount. It starts at **0**. Presets above the trade service's cap are hidden. The cap is `policy.trade.maxDevBuySol = 2`.
- **Launch cost** is one line taken from **`POST /launch/preview`** on the trade service: a simulation of this exact launch (network fee, account rent, launchpad fee), shown as "Launch cost about X SOL". The dev buy is not counted as a cost. When nothing was measured the line is a dash with a reason: "not live yet", "sign in to see it", "measuring", or "couldn't be read".
- Behind **More options**: description, X/Telegram/website links (the post's link is pre-filled), the creator fee read from the chain, and, for the owner only, thumbs up/down on each suggested name. Those thumbs feed the re-tune's NAMES section (§10). **None have been recorded yet.**
- **Trading switch:** Create only builds a real launch when the app was built with `VITE_TRADING=on` **and** the trade service is open (switch on, wallet allowlist not empty, RPC set). Otherwise the button says "Coming soon" and nothing is minted. **I could not tell from the repo whether the deployed mobile build has it on.** It is a build-time variable on Vercel and is not stored in the repo.

### 7.4 Owner's spec vs what is built

| Owner asked for | What happens | Match? |
|---|---|---|
| 3–5 suggestions | Up to 5 allowed (`MAX_NAME_IDEAS = 5`), but **in practice 2–3**. No post on the feed now has more than 3, and 2 have none. The cause is that the prompt never asks for alternate names; the model only fills a required field | **No, in practice** |
| from the **same judge call** | Yes. `proposedName` and `alternateNames` come from the per-post judgement, with no extra call | Yes |
| ...asked for properly | The post prompt only defines **one** name ("at most three words"). It says nothing about alternates, brands or real people's names. `core/src/create/name-ideas.ts` says "the judge's prompt is told not to offer them" (brands and real people's full names). **That is not true of the live post prompt.** Brands are filtered only by a ~50-word list, and real people's names are not filtered at all | **No** |
| live **DexScreener + GeckoTerminal** ticker check | Yes. Both are searched live when Create opens (1-minute cache), **plus** our own `public.coin_name` index. Results are kept only to Solana launchpad coins | Yes |
| Free / Taken | "Free", "Taken by N coins" (opens the list), or a third answer, "Couldn't check", when a source is down | Yes, plus an honest third state |

Names are only suggested for posts that are **on a feed right now**. If a post leaves the feed, its ideas are deleted on the next projection, and Create opened from it returns 404 and falls back to the title.

---
## 8. Trading: Buy, Sell and Create

### 8.1 The switches, as deployed

Trading is open only if **all** of the following hold (`services/trade/src/config.ts` `tradingOpen`). The service checks this before anything else on every route. If it fails, every route answers "Trading is off."

| Check | Deployed state (names only, values not printed) |
|---|---|
| `TRADING` exactly `on` (trade service, Railway) | **on** |
| `TRADING_ALLOWLIST` names at least one wallet | **1 wallet**: the owner's wallet `A1e8…vvdL`, which is also `policy.owner.wallets` |
| RPC configured (`HELIUS_API_KEY`) | set |
| App flag `VITE_TRADING` exactly `on`, baked in at build (`mobile/src/shared/trading.ts`) | **on** in the live bundle at insidor-mobile.vercel.app (checked from the served JavaScript). Off, Buy, Sell, Send and Create would say "Coming soon". |
| Create path: `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`, `DATABASE_URL_APP` | all set, so Create is live |
| `JUPITER_API_KEY` (aggregator for bonded coins) | set |
| `TRADE_ALLOWED_ORIGINS` | not set; this is harmless because the app calls the service through Vercel's same-origin `/api/trade` proxy |

On Railway the trade service logged `trading=true allowlisted=1` at every start since 12 October.

**In practice, only the owner's own wallet can trade or create. Anyone else signing in gets "This wallet is not allowed to trade yet."**

### 8.2 Buy and Sell, end to end

Code: `mobile/src/features/trade/TradeBar.tsx`, `OrderSheet.tsx` and `order-flow.ts`, and `services/trade/src/routes.ts` and `gate.ts`.

1. **Coin page → Buy/Sell.** The buttons appear only with `VITE_TRADING=on`.
2. **`GET /venue/:mint`** works out where the coin trades:
   - on pump.fun's bonding curve (`curve`), using the launchpad's own instructions;
   - through Jupiter (`pool`) once the coin has bonded;
   - `none`: the card says "Can't be traded right now";
   - `unavailable`: the card says "Trading isn't available just now".

   The reply also carries the limits, so the sheet only offers sizes the server will accept.
3. **Order sheet.**
   - Buy sizes come from the ladder 0.01, 0.025, 0.05, 0.25 and 1 SOL. Sizes over the cap are cut, and "Max" is offered.
   - Sell sizes are 25, 50, 75 and 100% of what the wallet holds (`GET /balance`).
   - Slippage choices are 1%, 3%, 5% and 10%.
4. **`GET /quote`** runs the gate:
   - the wallet must be on the allowlist;
   - the amount must be a positive whole number of base units;
   - slippage must be between 1 and 1,500 bps;
   - for a buy, the **amount plus its slippage** must be at most the cap.

   Then it asks the curve or Jupiter for a price.
5. **Confirm → `POST /build`**:
   - The server reads back the quote the person saw. A Jupiter quote whose shown fields differ from the router's own reply is refused as "quote tampered".
   - It runs the gate again. The quote is not re-priced.
   - It builds an **unsigned** transaction and **simulates** it on chain. If the simulation fails, the reply carries the reason and log lines, with **no transaction in it**.
6. **Sign.** The app hands the transaction to the wallet only after a successful build (`confirmBuilt`). The wallet is Privy's embedded wallet (`wallet-embedded/src/EmbeddedWallet.tsx`), which signs with `useSignTransaction` on `solana:mainnet`. Sign-in is with X or Google.
7. **`POST /send`.** The signed transaction goes back to the trade service, which checks that the fee payer is on the allowlist and broadcasts it over Helius.

### 8.3 Create a coin, end to end

Code: `mobile/src/features/create/CreateCoin.tsx`, `services/trade/src/routes.ts` and `adapters/venue/solana-pumpfun/src/launch.ts`.

1. **The Create sheet**:
   - the post's picture;
   - the top name idea plus chips with the others, at most 5, brand names removed (`post-match.ts`);
   - a ticker of 3–10 capitals, marked Free or Taken;
   - a dev buy of 0, 0.5, 1 or 2 SOL, or a typed amount;
   - description, socials and creator fee under "More options".
2. **`GET /launch/rules`** reads from the chain whether pump.fun has new coins open, the creator fee and the protocol fee, and returns the dev-buy cap.
3. **`POST /launch/preview`**:
   - It checks the allowlist, a name of at most 32 characters, a ticker of at most 10, and a dev buy of at most `maxDevBuySol`.
   - If launches are paused, it refuses with that reason.
   - It builds the launch against a dummy metadata link and simulates it, reporting the real network fee, rent and launchpad fee. The transaction is then thrown away.
4. **`POST /launch/build`**:
   - It stores the picture and metadata in Insidor's own Supabase storage bucket and builds pump.fun `create_v2`, plus the dev buy in the same transaction.
   - It simulates the transaction. A failed simulation returns a short reason, such as "Not enough SOL in this wallet…", and nothing signable.
   - The new coin's own address key is made in memory, signs only its own creation, and is discarded. It is never stored.
5. The person's wallet signs, then `/send` broadcasts.
6. **`POST /launch/confirm`** checks on the chain that the transaction landed, was paid by this wallet and created this coin. Only then does it write `public.created_coin` through `public.register_created_coin`, shown as "Created on Insidor".

### 8.4 Every safety check, with the numbers (`policy.trade`)

| Check | Value / where |
|---|---|
| Per-buy cap `maxSolPerTrade` | **0.05 SOL**, measured on amount + slippage, checked at quote **and** build. A SOL Send (`/transfer`) has the same cap. |
| Sells | capped only by what the wallet holds |
| Dev-buy cap at launch `maxDevBuySol` | **2 SOL**, **40× the buy cap**. The policy comment says "Lower it for a first test." It has not been lowered. |
| Slippage | default **3%** (300 bps), server maximum **15%** (1,500 bps); the app offers 1, 3, 5 and 10% |
| Insidor platform fee `platformFeeBps` | **0, off** |
| Creator and protocol fees on launch | read from pump.fun's fee config on chain, never hard-coded; shown in the preview |
| Probe size to test tradability | 0.001 SOL (a quote only, nothing sent) |
| Simulation before signing | yes, for buy, sell, send and launch; a failure returns no transaction |
| Unsigned transactions only | yes; the server never signs for a person. The only key the server makes is the new coin's one-time mint key. |
| Who signs | Privy's embedded wallet, in the person's browser |
| Private keys | Never on Insidor's servers. **One exception in the app:** Portfolio has "Import wallet" (`wallet-embedded/src/key-import.ts`). A pasted private key goes from the input box straight to Privy's SDK. The box is emptied at once, and the key is never sent to Insidor, stored or logged (tested in `key-import.test.ts`). |
| Request body cap | 64 KB |

**Gaps worth knowing:**

- **`/send` checks only that the payer is allowlisted.** It does not re-check the cap on a signed transaction. Because the allowlist is only the owner, this matters only for the owner's own wallet.
- **Trades are not recorded in Insidor's database at all.** There is no trade table, and `internal.spend` does not cover trades. Only launches are recorded, in `public.created_coin`.

### 8.5 What has been done with real money

- `public.created_coin`: **0 rows**, so no coin has ever been created through Insidor.
- No trade table exists, so the database cannot show trades.
- Railway HTTP logs for the current trade deployment (started 2026-10-13 08:25 UTC, read up to about 13:25 UTC):
  - 5 × `POST /launch/preview` → **422**, all refused. The reason is not logged; a 422 there is a refusal such as not enough SOL or a failed simulation.
  - 3 × `GET /launch/rules`, 5 × `GET /venue`, **2 × `GET /quote`**, 4 × `GET /balance`.
  - **No `/build`, `/send`, `/transfer`, `/launch/build` or `/launch/confirm` request at all.**
- HTTP logs for the two earlier deployments (12 Oct 21:49 UTC and 13 Oct 01:41 UTC) are no longer available.
- No commit message records a live trade or launch test.

**Verdict: Buy, Sell, Send and Create have not been tested with real money.** The owner opened the ticket and got two real quotes. Every Create attempt today stopped at the cost preview, which refused it each time.

---
## 9. Alerts

### 9.1 The kinds (`core/src/notify/alerts.ts` `ALERT_KINDS`) and when each fires

| Kind | Fires when | Who gets it | Switch (default) | Code |
|---|---|---|---|---|
| `follow` | someone follows you | you | follows (on) | `services/social/src/routes.ts:336` |
| `reply` | someone replies to your comment | you | replies (on) | social `:386` |
| `rank_up` | your rank rises in the season | you; **list only, never pushed** | always | social `:486` |
| `holding_multiple` | a coin you hold, worth ≥ **$10**, reaches a multiple of your average buy price in SOL. No buy history means no alert. | you | holdings (on); levels 2×, 5×, 10× by default | `services/notify/src/market.ts` |
| `watchlist_multiple` | a coin of a narrative you watch reaches a multiple of its price when you started watching | you | watchlist (**off**) | market.ts |
| `graduating_soon` | a held or watched coin passes **90%** of its curve | you | graduations (on) | market.ts |
| `graduated` | its curve completes after we saw it open. A coin already graduated when first seen does not count. | you | graduations (on) | market.ts |
| `rising_post` | **any post new to the For You feed** | **every profile** | rising (on) | `services/notify/src/pipeline.ts` |
| `narrative_promoted` | a narrative (`origin='observed'` only) is newly promoted to the board | every profile | narratives (on) | pipeline.ts |
| `narrative_coin` | a narrative gets its first settled coin (`one` or `several`) | every profile | narratives (on) | pipeline.ts |
| `coin_matched` | each new coin settled on a narrative | people **watching that narrative** | coins (on) | pipeline.ts |
| `supplier_warning` | a supplier's spend today reaches **80%** or **95%** of its daily line (`budget.warnAt`), with how much to raise it (the day's rate × 1.25). Fires once per supplier, level and UTC day, and only the highest level crossed. | the owner: the profile with the `OWNER_WALLET` wallet (runner variable) | **no switch** | `services/runner/src/supplier-warnings.ts` |
| `retune_summary` | after each judge re-tune (08:00 and 20:00 London, started 20 min early, plus one-offs): what was learned and whether it was applied | the owner (`policy.owner.wallets`) | **no switch** | `tools/retune.ts:301` |

The live levels for multiples are 2, 3, 5, 10, 25, 50 and 100× (`notify.milestones`). Of these, 2, 5 and 10× are on for a new person.

**Compared with what the owner might assume: `coin_matched` is about a narrative's coin, not a feed post's.** When the evidence rule in §6 settles a For You post's own coin ("matched"), **no alert fires.**

### 9.2 Limits

- **Once per level, ever.** A claim row (`people.alert_claim`) is unique on person, kind, subject and level. A 10× alert also spends the 5× and 2× levels, so they never fire after it.
- **First look sends nothing.** On a stream's first read, everything on it is marked seen and no alert goes out (`people.pipeline_seen`).
- **Off means nothing.** A kind switched off, or everything muted, creates no row and no push. Mute also silences the supplier warning.
- **Quiet hours.** Set per person in their own timezone; the window may cross midnight. During quiet hours the alert is listed but not pushed. This applies to all kinds, including supplier warnings.
- **Hourly push limit.** **6** pushes an hour by default, adjustable from 1 to **30**. Beyond the limit the alert is listed, not pushed. **Supplier warnings ignore this limit**; the retune summary obeys it.
- **There is no per-day cap on any kind.** `rising_post` fires for every new For You post to every profile. Only the hourly push limit holds it back, and nothing limits how many are listed.
- **Timing.** The pipeline is checked every 60 s, prices and curves every 2 min, and the queue delivered every 5 s. A wallet's cost basis is re-read at most every 30 min. A push not delivered within 24 h is dropped by the push service. The Alerts list shows 40 rows.

### 9.3 Delivery

- **Method: web push** (VAPID, the `web-push` library, `adapters/push/webpush`), received by the app's service worker. Every alert is also written to the in-app Alerts list (`people.alert`).
- The notify service on Railway has `NOTIFY` set exactly to `on`, and `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY` and `VAPID_SUBJECT` are set. The live app bundle includes the push subscription code.
- A push happens only if push is on, the person is not in quiet hours, is under the hourly limit, the kind is not list-only, **and they have a registered device.**

### 9.4 Live counts (measured 2026-10-13 13:27 UTC)

There are 4 profiles. **Push devices registered: 0.** Saved alert settings: 0, so everyone is on the defaults. The queue is empty.

| Kind | Last 24 h | Delivered as | Notes |
|---|---|---|---|
| `rising_post` | **60** | all `no-device` | 29 distinct posts × the 4 profiles, 08:23–13:06 UTC |
| `supplier_warning` | **4** | all `no-device` | to the owner: Pictures 80% (08:51) and 95% (10:02); Jobs judge 95% and Jobs pictures 95% (both 13:04) |
| `retune_summary` | **1** | `no-device` | 13:04, slot 12:48 UTC, "Judge examples kept", meaning not applied |
| every other kind | 0 | — | no follows, replies, rank-ups, multiples, graduations, promoted narratives or coin alerts yet. These are also the only alerts ever recorded. |

**No alert has ever reached a phone as a push. All 65 went only to the in-app list,** because no device has subscribed. To receive pushes, the owner needs to turn on alerts in the app on their phone, which registers a device.
## 10. Learning (the twice-daily re-tune)

### 10.1 What the system learns from

| Input | Where it lives | Immediate effect | Use in the re-tune |
|---|---|---|---|
| **Owner taps** "Is a meme" / "Not a meme" in the app | `internal.owner_feedback` (subject `post`). **Now: 73 "not" (07:24–13:09), 2 "meme" (07:32)** | Applied at the next projection, within about a minute. "Not" hides the post as `owner_not_meme`; "meme" passes it, but it still needs the views bar | Becomes a labelled example. The owner's note, if any, is used as the reason. A tap overrides a missed post, which overrides the labels file. Newest call wins |
| **Owner's coin-match calls** (right / wrong / none) | `owner_feedback` subject `coin_match`. Now: 1 right, 1 none | — | **Not taught to the judge.** Only counted in the run summary as "matching lessons" |
| **Owner's name thumbs / better name** | `owner_feedback` name calls. **Now: none** (`names_used = 0`) | — | A "NAMES" section of the block, up to 60 entries |
| **Missed posts** (owner pastes a URL) | `internal.missed_post`. **Now: 7, all `done`, all diagnosed `never_found`** | — | Labelled "yes". Half (chosen by a hash of the post id) go into the block and half are **held back as the "missed" test**. A missed post with exactly one coin, posted before the coin launched, is also added to gold |
| **Label corrections** | `eval/src/feed/live-labels.json`. **206 posts: 184 not, 14 meme, 8 unsure** (unsure is ignored); last changed 2026-10-13 07:32 | — | Labelled examples. `ownerLabel` overrides `label` |
| **Gold additions** | `internal.gold_addition`. **Now: 77 from the recall check + 5 from missed posts**, all written 12:42:58 | — | **Test only, never taught.** Gold = the committed gold-set origin posts plus these additions. The recall check adds bonded coins from the last 3 days whose creator linked an X post made before launch |

**How the block is built** (`eval/src/feed/examples.ts` `buildBlock`; at most `learning.maxExamples = 120`):
- Each "not" post and each missed post goes either into the block or into the test, decided **by a hash of its post id alone**. That keeps the test made of posts the judge was never shown.
- Every owner "meme" goes into the block. The gold set is the test for "yes".
- The two sides get half the room each, and a short side gives its spare room to the other. Newest first.
- The block is plain text added after the rules, with "YES — people could coin this:", "NO — nothing to coin…" and (optionally) "NAMES…" sections. Each entry is a 140-character excerpt of the post, `[picture]` if it had one, and the reason.

**How a block is tested** (`tools/lib/pass-rates.ts` `gradePosts`):
- Same model, same post prompt with the block added, same `postScore` rules, same picture boost, and the same kind question.
- Posts go **one per call** (not 10 per call as live, see §3.7), sent as one **Message Batch at half price**.
- The test only starts if its pessimistic estimate (×1.5 to cover the kind question) fits what is left on the **learning line**. Otherwise the run ends "kept, not tested".
- Any unanswered post blocks applying the block.

**Safety gates** (`policy.learning`). **All** must hold for a block to be applied:
- gold pass rate **≥ 65%** (`minGoldPass 0.65`)
- live non-meme pass rate **≤ 5%** (`maxLiveNegativePass 0.05`), measured on the held-back half of the owner's "not" posts
- missed-post pass rate **≥ 50%** (`minMissedPass 0.5`), checked only when some missed post was held back

If nothing new has arrived since the last block (no new tap, missed post or gold addition, and the labels file is unchanged), the run is **skipped** at no cost. After each recorded run the owner gets a `retune_summary` alert.

**Money:** this runs on the **learning book** (`learn:` rows on the ledger, `budget.learningLines`: **$10.50/day** — judge $9, embeddings $0.50, X reads $1), never the live feed's line. Spent on `learn:judge` so far today: **$3.98 over 771 calls** (05:45–13:21).

### 10.2 Schedule

- The `project` service on Railway runs **`tools/retune.ts due --yes --detach` every 10 minutes** (`tools/railway-start.mjs`).
- `due` works out the latest **08:00 or 20:00 Europe/London** slot as of now + **20 minutes** (`learning.startAheadMs = 1,200,000`). If that slot has no row in `internal.judge_retune`, it starts a detached `run` for it.
- While the UK is on summer time, the slots are **07:00 and 19:00 UTC**, and runs start at about **06:40–06:50 and 18:40–18:50 UTC**. After the clocks change on 25 Oct they will be 08:00 and 20:00 UTC.
- The slot row is claimed before any money is spent, so a slot runs at most once. A run that dies stays `running` and is not retried.
- **Scheduled runs never grade a baseline**; only a hand run with `--baseline` does.
- The next scheduled slot is **19:00 UTC today (20:00 London)**.

### 10.3 Every re-tune so far

From `internal.judge_retune` and `internal.judge_examples_version`. Rates are passed/tested:

| Slot (UTC) | How started | Ran | Outcome | Block gold | Block live non-memes | Block missed | Baseline (rules alone) | Cost | Why |
|---|---|---|---|---|---|---|---|---|---|
| 2026-10-13 07:00 (08:00 London) | schedule | 07:29:55 → 07:30:29 | **kept**, not tested, no version | — | — | — | — | $0 (none booked) | "not tested: the line has $1.15 left and the test needs up to $2.97" (the learning line was $4/day then) |
| 2026-10-13 12:48 | by hand, with `--baseline` | 12:48:51 → 13:04:48 | **kept**, version 1 not applied | **55/118 (46.6%)** | **13/89 (14.6%)** | 1/2 (50%) | gold **65/118 (55.1%)**, non-memes 13/89 (14.6%), missed 1/2 | **$1.45** ($1.4485, block and baseline together) | "gold 55/118 is under 65%; live non-memes 13/89 is over 5%" |
| 2026-10-13 13:17 | by hand, with `--baseline`, forced | 13:17:28 → about 13:28 | **kept**, not applied | **59/118 (50%)** | **12/92 (13%)** | 1/2 (50%) | gold **69/118 (58%)**, non-memes 14/92 (15%), missed 1/2 | **$0.83** (most of the prompt read from cache: 1.41M tokens) | "gold 59/118 is under 65%; live non-memes 12/92 is over 5%". Learned from 19 taps, 5 missed posts, 96 label corrections |

The baseline column for the 12:48 run is not stored in the DB. It is in the run's log line and was supplied by the coordinator.

Version 1 had 120 examples (98 from the labels file, 17 taps, 5 missed posts, 0 name calls) and was 15,208 characters long. Its sha is `92e6ca40e1de188b`.

**What this means:**
- The learned block made the judge **worse on gold**: 55 of 118 passed, against 65 with the rules alone. It was **no better on non-memes**: 13 of 89 either way.
- Even the **rules-alone judge that runs live today fails both gates**: 55% gold against a 65% bar, and 14.6% non-memes against a 5% bar.
- As the gates are set now, a block is only applied if it lifts gold by about 12 posts **and** cuts non-meme passes from 13 to 4 or fewer, at the same time. Until that happens the judge keeps running on its written rules alone.

### 10.4 The owner's missed posts and gold

All 7 rows in `internal.missed_post` show diagnosis `never_found` and no coin of their own: the runner's coin search found none by the judge's name. The 5 origin posts among them were added to `internal.gold_addition` by hand at 12:42:58 UTC with the coin addresses the owner supplied in the missed-posts list (the two later reposts of the 4nMTfs coin were left out of gold, as the owner asked). That is why gold has coins these rows do not. Those 5 count as gold in every re-tune test, except a post that is also a taught example in that run.
## 11. Money

### 11.1 The budget lines in force (`contracts/src/policy.ts`, `budget`)

Spending is split into three **books**. Each book has its own daily total and its own line for each supplier.

| Book | What spends from it | Daily total | Posts (X, twitterapi.io) | Judge (Anthropic) | Short video (TikTok via Apify) | Pictures (Voyage embeddings) |
|---|---|---|---|---|---|---|
| **Live** | the runner: discovery, judging, coin lookup, embeddings | **$21.50** | $10.00 | $7.50 | $3.00 ($2 "video" + $1 "actors", both billed by Apify, so they form one $3 line) | $1.00 |
| **Jobs** | one-off paid jobs (`tools/jobs.ts`, Railway `jobs` service) | **$3.75** | $0.50 | $2.75 | $0.25 | $0.25 |
| **Learning** | judge re-tunes, missed posts, gold additions | **$10.50** | $1.00 | $9.00 | $0 | $0.50 |
| **All books** | | **$35.75/day** | | | | |

The owner's approved ceiling is **$38/day**, so the lines leave $2.25 of headroom. The policy comments record how the lines changed today: the learning line went $4 → $5 → $7.25 → $10.50 (the last change at 13:20), jobs went $6.50 → $8.75 → $4.75 → $3.75, and the live line went to $21.50 at 13:00.

On Railway the runner's `DAILY_BUDGET_USD` is 21.5, and so is the jobs service's. The runner uses whichever is lower, this or the policy's `dailyUsd`, so the two agree. Both services have `SPEND=live` set.

### 11.2 Today's spend against each line (`internal.spend`, 00:00–13:24 UTC)

How each spend is assigned: an endpoint starting `job:` goes to the jobs book, one starting `learn:` goes to the learning book, and everything else is live. Vendors map to lines as twitterapi.io → posts, anthropic → judge, apify → short video, voyage → pictures.

The "expected per day" column takes the spend in the last 6 hours (07:24–13:24) and multiplies it by 4.

| Book / line | Line | Spent today | Share of line | Last 6 h × 4 (expected/day) |
|---|---|---|---|---|
| **Live total** | $21.50 | **$9.64** | 45% | **$15.73** |
| Live posts (X) | $10.00 | $3.96 | 40% | $6.86 |
| Live judge | $7.50 | $4.07 | 54% | $5.75 |
| Live short video (Apify) | $3.00 | $1.06 | 35% | $2.27 |
| Live pictures (Voyage) | $1.00 | $0.54 | 54% | $0.87 |
| **Jobs total** | $3.75 | **$7.22** | **193%** | $0 (nothing since 05:38) |
| Jobs judge | $2.75 | $6.57 | 239% | — |
| Jobs posts | $0.50 | $0.29 | 58% | — |
| Jobs pictures | $0.25 | $0.36 | 145% | — |
| **Learning total** | $10.50 | **$4.30** | 41% | $7.35 |
| Learning judge | $9.00 | $3.99 | 44% | $6.94 |
| Learning posts | $1.00 | $0.02 | 2% | $0.05 |
| Learning pictures | $0.50 | $0.30 | 59% | $0.35 |
| **All books** | $35.75 | **$21.16** | 59% | — |

Here is where the day is heading, at the rate of the last six hours, which is about $0.66 an hour for live:

- **Live:** about $16.6 by midnight. That is under the $20.86 point where it stops (97% of $21.50). The judge line should cross its 80% warning ($6.00) at about 21:30, and the pictures line its 80% warning ($0.80) at about 20:30. Neither should reach 95%. Posts and video should finish the day around 70% of their lines.
- **Jobs:** this book is **already over its total**. Its $7.22 was spent between 00:46 and 05:38, when the jobs line was still $6.00–$8.75; the line was cut to $3.75 later. In practice no jobs-book spend can happen for the rest of today. The jobs service has finished its work anyway (its log ends "every stage has run; the job is finished").
- **Learning:** a re-tune started at 13:17 and was still running at 13:28; it had spent $0.83 since starting. The 20:00 London re-tune (19:00 UTC) needs about $3 free when it starts. Including the current run's spend, $5.89 was left before learning's stop point ($10.19), so both should fit.
- **Whole system:** at about $21.2 spent so far, the day will most likely close around $28–31, under $35.75.

For comparison, the whole day of 2026-10-12 shows $2.00 in the ledger, because discovery was off until 22:12 that evening.

### 11.3 What happens when a line runs low

- **A supplier line never stops calls.** This has been true since commit 90658f9. Policy sets `vendorLinesStop: false`, and the runner opens every book's meter with `vendorLinesWarnOnly: true` (`services/runner/src/wiring.ts`). In `adapters/meter/spend/src/in-memory.ts`, `mayspend` checks the day's total and then returns "yes" before it reaches the per-supplier check. The same applies to the jobs and learning books.
- **A line warns instead, at 80% and 95%** (`services/runner/src/supplier-warnings.ts`; `warnAt: [0.8, 0.95]`):
  - The check runs once a minute, right after the meter re-reads the ledger.
  - It writes a log line and queues a push alert to the owner's profile (found through `OWNER_WALLET`).
  - Each warning fires once per supplier, per level and per UTC day. Only the highest level crossed fires.
  - The message says how much to raise the line by: the day's projected spend × `lineBuffer` 1.25, minus the line.
  - All three books are watched.
- **Warnings that fired today:** after the runner restarted at 13:19, it logged two at 13:20:
  - `supplier line at 95%: jobs judge $6.57 of $2.75; raise it by $12.25`
  - `supplier line at 95%: jobs pictures $0.36 of $0.25; raise it by $0.75`

  Both are about the jobs book's morning spend. The suggested raises are inflated, because the projection spreads a burst of morning spend across the whole day, and jobs will spend nothing more today. **No live-line warning had fired by 13:28.**
- **Only a book's daily total stops spending.** A call is refused once the book's spend plus the call's estimate would pass `softStopFraction` 0.97 of that book's total:

  | Book | Stops at |
  |---|---|
  | Live | $20.855 |
  | Jobs | $3.64 |
  | Learning | $10.185 |

  Neither the lines nor the totals are paced through the day: the runner works at full speed until a total stops it. A refused call is a product outcome (the stage records that it paused), not an error. The meter shares one ledger across processes and re-reads it every 60 s (`ledgerRefreshMs`), so the jobs and learning tools count against the same day.

### 11.4 Flag: Apify's own monthly limit

The Apify account is on the Starter plan with a **$19 monthly usage limit**. That is an account setting on Apify's side, and only the owner can change it. When it is reached, Apify refuses runs and TikTok discovery stops, whatever Insidor's lines say.

- At the $3/day line, $19 lasts about **6 days**. At today's expected $2.27/day, it lasts about 8 days.
- Insidor's ledger shows Apify spend only today: $1.06 this month, all of it after 02:06 UTC on 10-13.
- Apify's own month-to-date figure, which may include platform charges our ledger does not record, is **not checked yet**. The owner should look at it in the Apify console. Unless the limit is raised, expect TikTok to stop around 2026-10-19 to 10-21.

---
## 12. Measured performance

### 12.1 Recall: what share of coin posts we catch, and how early

The 69% figure is **modelled, not observed**. It comes from `eval/src/gold/recall-report.md`, which `eval/src/gold/run-recall.ts` rendered at about 02:19 UTC on 2026-10-13. The settings were measured at 00:15 and the posts read at 00:16; the result was committed as ed6a8a3. The report applies a model of the current discovery settings to the founders' gold origin posts. Those coins launched before live discovery existed, so none of them was actually seen live.

| Group | Caught | Median delay after posting |
|---|---|---|
| X origin posts, tuning half | 76% (28 of 37) | 1.6 min |
| X origin posts, held out | 56% (10 of 18) | 2.1 min |
| **X origin posts, all** | **69% (38 of 55)** | **1.8 min** |
| Every origin post (TikTok/Instagram counted as misses) | 54% (38 of 71) | 1.8 min |
| X, if approvals grew 3× slower | 58% (32 of 55) | 2.1 min |

- 37 of the 38 caught X posts were caught before the coin launched.
- Why X posts were missed: not found by our search 8, posted between checks 6, deleted 2, under the approvals floor 1.

**The observed counterpart is the daily recall check** (`tools/recall-check.ts daily` → `internal.recall_check` and `internal.recall_check_run`). It has run once, at 00:20 on 2026-10-13. It covered 1,167 bonded coins; for 652 it read the metadata, and 106 linked an X post. **0 of those 106 were caught.** The recorded reasons:

| Reason for the miss | Posts |
|---|---|
| `discovery_off` (discovery was not running on 10-12 until 22:12) | 102 |
| `under_floor` | 3 |
| `deleted` | 1 |

So no live recall figure exists yet.

**Problem found: the scheduled daily recall check fails on Railway.** The `project` service starts it on boot and then every 24 h. At 13:20 today it failed with `DATABASE_URL_INTERNAL is not set; refusing to open a pool for role 'internal'` (from `tools/recall-check.ts` `openDb`, line 97). The project service has only `DATABASE_URL`, and unlike `tools/retune.ts`, the recall check does not fall back to it. Until this is fixed, no observed recall figure will be produced.

### 12.2 Judge pass rates: gold set and negative set

The gates for applying a new learned-examples block are gold ≥ 65% and live non-memes ≤ 5% (taken from the run's reason text).

**Today's re-tunes** (`internal.judge_retune`, `internal.judge_examples_version`):

| Slot | Ran | Outcome | Result |
|---|---|---|---|
| 07:00 (08:00 London) | 07:29–07:30 | kept, **not tested** | "the line has $1.15 left and the test needs up to $2.97" |
| one-off | 12:48–13:04 | kept, block v1 not applied, cost $1.45 | **gold 55/118 = 46.6%** (gate ≥65%); **live non-memes 13/89 = 14.6%** (gate ≤5%); missed posts 1/2 |
| one-off | 13:17–13:28 | kept, block not applied, cost $0.83 | **gold 59/118 = 50%**; **live non-memes 12/92 = 13%**; missed 1/2. Baseline (rules alone): gold 69/118 = 58%, non-memes 14/92 = 15%, missed 1/2 |

Both runs also graded the same posts with the live judge's rules alone (the "baseline"), which is what runs on the feed: 65/118 gold (55%) and 13/89 non-memes (15%) at 12:48; 69/118 (58%) and 14/92 (15%) at 13:17. The learned block scored 55/118 (47%) and 59/118 (50%) on gold. The baseline numbers are in the runs' log lines, not in the database.

**The earlier measurement of the gate now deployed** comes from `var/pass-rates-kind3.json`, written at 06:14 UTC on 10-13 by `tools/feed-pass-rates.ts`. It measured the two-question gate that went live in commits df89177/93c5cc4 (05:25 UTC):

| Set | Passed | Rate |
|---|---|---|
| Gold | 26/41 | **63%** |
| Live non-memes | 13/186 | **7%** |
| Owner's live memes | 11/12 | 92% |
| Curated non-meme sample | 0/43 | 0% |

For context, other runs that morning:

| Run (file) | Time | Gold | Live non-memes |
|---|---|---|---|
| `live-before` | 05:16 | 27/41 | 66/186 |
| `kind4` | 06:20 | 24/41 | 7/186 |

The sets in the morning runs (41 gold, 186 non-memes) and in the re-tune (118 gold, 89 non-memes) are different sizes, so their rates cannot be compared directly.

### 12.3 Posts per hour: in, scored, shown (last 24 h, measured 13:28 UTC)

How each column is counted:

- **In:** `public.item.first_seen_at`, by source.
- **Judged:** `internal.post_judgement.judged_at` where `copied_from` is null; **Yes** is the share the judge called coinable.
- **Held back:** posts first rejected from the feeds that hour (`public.feed_rejection.first_rejected_at`).
- **On feed now:** posts currently in `public.rising_row`, counted by the hour they were first seen.

**There is no history of feed rows.** `rising_row` is replaced on every rebuild and no migration keeps a history of it, so "shown per hour" cannot be reconstructed. Only what is on the feeds now can be counted: 13 posts, holding 19 rows (fyp 13, popular 6).

| Hour (UTC) | X in | TikTok in | Judged | Yes | Held back | On feed now |
|---|---|---|---|---|---|---|
| 10-12 14–21 | 0 | 0 | 0 | 0 | 0 | 0 |
| 10-12 22 | 817 | 0 | 0 | 0 | 0 | 2 |
| 10-12 23 | 422 | 0 | 588 | 68 | 0 | 3 |
| 10-13 00 | 110 | 0 | 516 | 31 | 76 | 0 |
| 10-13 01 | **0** | 0 | 402 | 36 | 1 | 0 |
| 10-13 02 | 575 | 28 | 222 | 24 | 0 | 2 |
| 10-13 03 | 597 | 30 | 242 | 28 | 0 | 0 |
| 10-13 04 | 397 | 25 | 116 | 18 | 0 | 1 |
| 10-13 05 | 529 | 17 | 63 | 5 | 16 | 0 |
| 10-13 06 | 563 | 36 | 161 | 43 | 45 | 0 |
| 10-13 07 | 660 | 27 | 203 | 86 | 7 | 0 |
| 10-13 08 | 691 | 20 | 217 | 88 | 14 | 0 |
| 10-13 09 | 714 | 11 | 287 | 124 | 0 | 1 |
| 10-13 10 | 556 | 20 | 178 | 75 | 2 | 1 |
| 10-13 11 | 563 | 6 | 140 | 55 | 3 | 0 |
| 10-13 12 | 627 | 13 | 147 | 54 | 0 | 2 |
| 10-13 13 (to 13:28) | 202 | 3 | 76 | 30 | 5 | 1 |
| **24 h total** | **8,020** | **236** | | | **169 posts** | **13** |

Held-back reasons for the 169 posts:

| Reason | Posts |
|---|---|
| not memeable | 71 |
| under reach bar | 51 |
| owner said not a meme | 38 |
| reach unread | 9 |

Two narratives were also held back as `too_few_passing_posts`.

**Gaps:**

1. **10-12 14:00–22:12:** nothing was stored. Discovery was off; the earliest item in the table is from 22:12. This is the `discovery_off` reason in the recall check.
2. **10-13 00:08–02:04:** no new posts were stored. Discovery was paused by pacing, and the database's session pooler was full. The jobs service logged `EMAXCONNSESSION max clients reached in session mode … pool_size: 15` while saving files. Judging carried on through 01:00 on the backlog.
3. In steady state (07:00–13:00) the system takes in about 560–715 X posts and about 6–36 TikTok posts an hour. It judges about 140–290 an hour, and the judge says yes to about 37–43% of those.

The hours of 22:00–01:00 show more posts judged than came in. That is the overnight re-scoring of the stored backlog (commit 93c5cc4), not new posts.

---

## 13. Known problems and gaps

### 13.1 Broken, or doing less than its settings say

| # | Problem | Evidence | Status |
|---|---|---|---|
| 1 | The judge misses the targets. With its rules alone (what runs live) it passed 65/118 and 69/118 gold posts (55%, 58%) and 13/89 and 14/92 live non-memes (15%) in today's two tests. Targets: 65% and 5%. | Re-tune baselines 12:48 and 13:17 UTC | Open. No learned block can pass both gates while the base judge is this far off. |
| 2 | Posts that pass only by the picture boost skip the "kind" question, so a boosted ad, sports or news post can reach For You. Two of the 13 For You posts at 13:28 got in this way (the Yorkshire tiger news post, a K-drama fan post). | `services/runner/src/work/score-posts.ts`; For You rows 2 and 3 | Open |
| 3 | No feed card has a confident coin or a Buy. For You: 8 "See N coins", 3 none, 2 pending. | `public.rising_row` at 13:24 | Open |
| 4 | Coin matching on gold: 7 of 71 right (10%), 1 wrong, 63 missed. | `eval/src/gold/matching-report.md`, 06:08 UTC | Open; last measured this morning |
| 5 | The origin finder cannot find an origin that does not contain the coin's exact name or ticker, and searches only 20 posts just before launch. On the owner's test coin (4nMTfs, "Scientists reveal the Hi") it finds nothing: the coin has no creator link, none of the three posts contains the name, and the September 21 origin is three weeks before launch. | `core/src/origin/check.ts`; `internal.coin_metadata` | Open |
| 6 | The slow lane reaches back about an hour, not six. Each pass is capped at 60 newest posts. | Runner logs, `internal.item_found_by` | Open. Changing it needs the owner's approval (search change) |
| 7 | The daily recall check fails on Railway: the project service has no `DATABASE_URL_INTERNAL` and `tools/recall-check.ts` does not fall back to `DATABASE_URL`. So there is no live recall number; the 69% figure is a replay of the gold set. | Error at 13:20 UTC | Open |
| 8 | No push alert has ever been delivered: 0 devices registered. All alerts are in-app only. | `people` push tables | Open; needs the owner to allow notifications on a phone |
| 9 | Prompt caching does nothing on the live judge: the fixed prompt is about 1,600 tokens and Haiku caches only from 4,096. It works on re-tunes (1.41M tokens read from cache on the 13:17 run). | Adapter header; re-tune usage | Open; turns on by itself if a learned block is applied |
| 10 | "No double scoring" reuses almost nothing: 0 of 1,174 judgements in six hours were copies, because the search already drops plain reposts and X almost never serves the same picture file twice. | `internal.post_judgement.copied_from` | Working as built; saves about $0 |
| 11 | Name suggestions: 2–3 per post, not 3–5 (the prompt asks for one name). Real people's names are not filtered; brands only by a list of about 50 words. Weak tickers on today's feed: `THE`, `ROLLERDEAT`, `SIXFLAGS`. | `core/src/create/name-ideas.ts`, prompts | Open |
| 12 | A post held by the "kind" question (ad, sport, showbiz, fandom, politics) is recorded as plain `not_memeable`; no reason says which kind. Two posts are hidden by two "OTHER" answers saved before the current rule. | `public.feed_rejection` | Open |
| 13 | A TikTok video that arrives under its 500k bar is never read again (only judged videos are re-read, every three hours), so it cannot climb over the bar later. | `services/runner/src/work/track-gate.ts`, `stages.ts` | Open; deliberate, to keep re-reads inside the $3 line |
| 14 | The missed-post diagnosis does not know about outages: all seven of the owner's list came back "never found — unexplained", though rrealist (01:09 UTC) fell inside the 00:08–02:04 outage. | `internal.missed_post` | Open |
| 15 | No history of what was shown: feed rows are rewritten every minute, so "shown per hour" cannot be measured after the fact. | `public.rising_row` | Open |
| 17 | ADMIT and DETECT have no effect on any feed: TRACK no longer re-reads by ADMIT's verdict, GROUP ignores ADMIT on purpose, and DETECT produced 0 outputs in 135 runs over six hours. | `internal.stage_runs` | Open; spends nothing |
| 16 | The jobs book spent $7.22 today against its $3.75 line — all before the line was lowered this morning. Its 95% warnings suggest raising it by $12.25, which is wrong because jobs are finished for the day. | `internal.spend` | Known; harmless today |

### 13.2 Untested

- Buying, selling or creating a coin with real money: never done. `public.created_coin` has 0 rows; the logs show 5 refused cost previews and 2 quotes, no build or send.
- The Create sheet's success screen: not seen live.
- Web push on a real phone: no device has subscribed.
- The 20:00 re-tune with the new 20-minute early start: first run tonight.
- TikTok judging and TikTok on For You: first results expected within the hour after 13:34 UTC.

### 13.3 Guessed rather than measured

- TikTok's bars (×12.5 of X's) are set from one day of TikTok view counts (median 74k, 90th percentile 1.4M), not from coined TikTok posts.
- The slow lane's share of search money (0.2) is marked uncalibrated in policy.
- The picture boost thresholds (0.44 similarity, 0.3 "no" confidence) were set from the gold set's pictures on 2026-10-13.

### 13.4 Different from what the owner asked for

| Asked | Built |
|---|---|
| 3–5 name suggestions per post | 2–3 |
| Judge gold ≥ 65% and live negatives ≤ 5% | 55–58% and 15% |
| Wrong coin links under 2% | 1.7% — met |
| Slow-burn lane looking back 6 hours | about 1 hour in practice |
| Re-tune "before and after" | Built today as `--baseline`; the scheduled runs do not grade a baseline (it doubles the cost) |
| Cache picture-pair results across re-tunes | Re-tunes make no paid picture-pair calls; picture embeddings are reused within a run only, because stored ones differed slightly (cosine 0.9997–0.9999) from fresh ones |
| Railway right-sizing | Nothing changed: Railway bills use, not limits, so lowering limits saves $0 |
| Dev buy capped safely | Launch dev buy cap is 2 SOL, 40× the 0.05 SOL buy cap |

### 13.5 Top 5 risks before a demo

1. **The feed may not look like memes.** For You has 13 posts; several are news or launches (SpaceX, a zoo attack, a coaster death). The judge passes about 15% of non-memes.
2. **No coin on any card.** With no confident match, no card shows Buy, so the "see a meme, buy its coin" moment cannot be shown live unless a matched post appears.
3. **First real-money trade or launch during the demo.** It has never been done; a failure (simulation, Privy signing, fees) would happen in front of the audience. Do one small real test beforehand.
4. **Narratives board was empty all day.** The fix went live at 13:36 UTC; the board needs hours of posts to fill and promote stories.
5. **TikTok is unproven and capped.** The first TikTok judgements are pending, and the Apify account's own $19 monthly limit stops TikTok in about 6–8 days unless the owner raises it.

## 14. Services

All Railway services run in project `adventurous-kindness`, environment production. Each has **1 replica, region sfo (US West)** (`railway status --json`). One start script, `tools/railway-start.mjs`, launches all of them: the service's name picks what runs.

### 14.1 What each service does, and its health at 13:24–13:28 UTC

| Service | What it does | Latest deploy | Health now |
|---|---|---|---|
| **runner** | Main pipeline: discovery (X search, TikTok watch list) plus seven decision stages (admit, track, detect, group, qualify, resolve, rank), the post judge, coin lookup and embeddings. Holds the live spending meter and sends the supplier warnings. | 13:17 | Up, but no public domain; `/health` answers on its private `HEALTH_PORT`. Log at 13:27 shows passes running. In the last hour: discover 24 runs (1 error, then backoff), track 50 ok, universe 32 ok. The admit, group, qualify and resolve passes ended "empty" (nothing to do), which is normal. |
| **chainwatch** | Watches Solana mints on one long-lived stream and keeps a coverage log, kept separate so runner deploys do not punch holes in it. | 10-12 21:33 | Up, no public domain. 1,200 watch cycles in the last hour, none erroring (515 "empty", which is normal). Last reconnect 11:45: a 6.2 s socket drop, recorded as a coverage gap. |
| **read** | Serves the app's feeds and pages as finished JSON. Runs on the read-only app login, which cannot see internal tables. | 08:22 | `read-production-a6c0.up.railway.app/health` → **200** `{"status":"ok"}` (0.26 s) |
| **trade** | Buy/sell and launch: the launchpad curve, the Jupiter aggregator, transaction simulate and send. | 08:22 | `trade-production-0e2e.up.railway.app/health` → **200** (0.24 s) |
| **social** | Comments, identity check (Privy), holdings and proof transactions. | 07:14 | `social-production-030e.up.railway.app/health` → **200** (0.34 s) |
| **notify** | Push alerts. With `NOTIFY=on` (set): delivers the queue every few seconds, checks prices and curves every few minutes, checks the pipeline every minute. | 13:00 | `notify-production-7c56.up.railway.app/health` → **200** (0.23 s) |
| **project** | A schedule rather than a long-lived process. It runs these jobs one at a time:<br>• the board/feed rebuild every 1 min<br>• coin logos every 2 min<br>• market, pairs and the source indicator every 5 min<br>• labels hourly<br>• the daily recall check<br>• a re-tune check every 10 min<br>Its `/health` returns 200 only while the board was rebuilt in the last 5 min. | 13:17 | No public domain. Log at 13:20 shows the jobs running ok. **The recall check fails** (see 12.1). |
| **jobs** | One-off paid job (gold pictures, picture pairs, origin eval, history pilot). It runs once and exits 0. | 00:45 | Finished: log ends "every stage has run; the job is finished". Idle, and it does not restart. |
| **Insidor-layer3** | Dead: an old service with no variables and no domain. | 2026-08-03, **FAILED** | Not running. Costs $0 and can be removed. |
| **Vercel `insidor-mobile`** (insidor.app) | The phone web app. `/api/read|trade|social|notify/*` go through `api/proxy.js` to the Railway services (`mobile/vercel.json`). | — | `insidor.app/` → 200. `/api/read/health`, `/api/trade/health`, `/api/social/health`, `/api/notify/health` → all **200** at 13:24–13:28. |

### 14.2 Database connections

Only variable names are shown, with the port read from each URL:

- **6543** is Supabase's transaction pooler. It is fine for short, stand-alone queries.
- **5432** is the session pooler. It is needed for locks and LISTEN, and it is **capped at 15 clients in total**.

| Service | Variable names set | Pools it actually opens |
|---|---|---|
| runner | `DATABASE_URL` (6543), `DATABASE_URL_SESSION` (5432) | Work pool on `DATABASE_URL`, internal role, **max 5**. Lock pool on `DATABASE_URL_SESSION`, **1** session client (single-runner lock). |
| chainwatch | `DATABASE_URL` (5432) | **1** session client; every write goes on its lock's connection. |
| read | `DATABASE_URL_APP` (6543), `DATABASE_URL_APP_SESSION` (5432), `DATABASE_URL` (5432) | App pool on `DATABASE_URL_APP`, **max 3**. **1** LISTEN connection on `DATABASE_URL_APP_SESSION`. `DATABASE_URL` is only read to refuse to start if the app URL equals it; it never connects. |
| trade | `DATABASE_URL_APP` (6543) | Launch pool, **max 2**. |
| social | `DATABASE_URL_PEOPLE` (6543), `DATABASE_URL` (5432) | People pool, **max 3**, 10 s idle close. `DATABASE_URL` is only compared, never connected. |
| notify | `DATABASE_URL_PEOPLE` (6543), `DATABASE_URL` (5432) | Same as social: **max 3**. |
| project | `DATABASE_URL` (6543) | Each one-shot job opens its own pool; the service role defaults to max 3 (`store/src/client.ts`). **`DATABASE_URL_INTERNAL` is missing**, which breaks the recall check. |
| jobs | `DATABASE_URL` (6543) | Internal role, max 5 (`store/src/client.ts`). Idle now. |
| Insidor-layer3 | none | none |

Pool limits by role (`store/src/client.ts`):

| Role | Max clients | Statement timeout |
|---|---|---|
| app | 3 | 5 s |
| service | 3 | 30 s |
| internal | 5 | 60 s |

Opening a session-only pool on port 6543 is refused at startup.

The **fixed session-pooler use** is 3 clients: runner lock 1, chainwatch 1, read LISTEN 1. That leaves 12 of the 15 for laptop tools and anything else on 5432. At 13:28 the database showed 14 Supavisor backends (13 idle, 1 active). Supavisor hides the application names, so they cannot be attributed per service.

### 14.3 Resources and cost

I did not re-measure these. They are the caller's figures from Railway's metrics earlier today; the Railway CLI used here does not expose metrics. "Not checked yet" by this audit:

- Every service has 1 replica in sfo, with no custom CPU or memory limits.
- Typical memory is about 0.2 GB per service. The exceptions are the runner (0.37 GB typical, 0.65 GB peak) and jobs (0.6 GB typical, 1.11 GB peak). CPU use is tiny.
- Insidor's Railway cost is about **$0.85–0.90 a day**.
- Lowering the limits would save **$0**, because Railway bills what is used, not the limits.
- Insidor-layer3 costs $0 and can be deleted, which only the owner should do.

### 14.4 Other things noticed while checking

- Railway warns on every service that Config-as-Code (`railway.json`) is deprecated. Existing files keep working until 2026-12-01.
- `CLAUDE.md` says migrations run to 0042 (next 0043), but the repo has migrations up to **0061** (`0061_watch_lane.sql`). The instruction file is out of date.
