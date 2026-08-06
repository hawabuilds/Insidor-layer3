# System architecture

How to build it so that adding a platform is a file, adding a chain is a file, and the system
gets smarter every day without anyone retraining anything by hand.

This document answers a question that was asked like this: *"I guess i dont know how to
describe it but i want the perfect product."* Part of the job here is naming what that
actually means, because it is a real and specific thing, and it is not the same as "make it
good."

Every code claim carries a `file:line`. Every measurement names where it came from. Where an
earlier research pass said something and a later adversarial pass overturned it, the corrected
version is what appears here and the overturned version is named, so you can see what moved.
Where a recommendation is a **bet** rather than established practice, it says so in bold.

---

## 1. The shape in one page

### What you are actually asking for

You are asking for a system where **the expensive decisions are the ones you cannot change
later, and everything else is a file you can throw away.**

That is the whole thing. A "perfect product" is not one where every part is excellent. It is
one where you were careful in the three or four places that lock you in, and deliberately
sloppy everywhere else, because everywhere else is cheap to redo.

There are exactly three decisions in this system you cannot retrofit:

1. **The shape of what a platform hands you.** Get this wrong and every future platform gets
   bolted on with `if (platform === 'tt')` instead of a file.
2. **The shape of what a chain hands you.** Same, one layer down.
3. **Whether you wrote down what the system saw at the moment it decided.** This one is
   different in kind. The other two are expensive to change. This one is *impossible*, because
   the data is gone. A decision you did not record at the time cannot be recovered from
   anywhere, at any price, ever.

Everything else — the thresholds, the scoring formula, the clustering algorithm, the model, the
UI — is a file. You will rewrite all of them, several times, and that is fine and expected.

### The three layers

```
┌──────────────────────────────────────────────────────────────────────┐
│  THE EDGE — adapters                                                 │
│                                                                      │
│  Platform adapters          Venue adapters                           │
│  x/ tiktok/ reddit/         solana/pumpfun/  solana/amm/  base/…     │
│                                                                      │
│  Their only job: turn a vendor's JSON into our vocabulary, and       │
│  turn our intent into a vendor's API call. They hold no thresholds   │
│  and make no product decisions.                                      │
└───────────────────────────┬──────────────────────────────────────────┘
                            │  our vocabulary only
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│  THE CORE — pure logic                                               │
│                                                                      │
│  admit → track → detect → group → qualify → resolve → rank           │
│                                                                      │
│  Every stage is one function: (what we know, the policy) → a         │
│  Decision. No network. No clock. No database. No vendor names, no    │
│  platform names, no chain names — enforced by a grep in CI.          │
│                                                                      │
│  Because it is pure, you can re-run any past decision under a new    │
│  rule or a new model and see exactly what would have changed.        │
└───────────────────────────┬──────────────────────────────────────────┘
                            │  every Decision, always
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│  THE RECORD — the learning substrate                                 │
│                                                                      │
│  decisions  — what we saw, what we decided, why, under which rule    │
│  labels     — what actually happened, days later, with the           │
│               population it was computed over stated in the row      │
│                                                                      │
│  Join them and you have a training set. That join is the entire      │
│  machine-learning plan. It works before any model exists, because    │
│  the row does not care whether a threshold or a model decided.       │
└──────────────────────────────────────────────────────────────────────┘
```

### The one decision that cannot be retrofitted

**Write down the feature vector at the moment of the decision, and reserve a small random slice
of arrivals that the rules are not allowed to touch.**

The first half is a table. The second half is three lines of code. Together they are the
difference between a system that gets smarter and one that just gets bigger.

Why they cannot wait:

- **Recomputing a feature later gives you a different number, not a noisier one.** The current
  `viewsVelocity` (`worker/lib/velocity.js:48`) reads whichever snapshots happen to be attached
  when you call it, with no as-of filter. There is no way to ask it "what did this look like at
  9:13am?" Six days later it returns a different quantity with the same name.
- **Worse, the length of a post's history is a function of the thing you are trying to
  predict.** `snapshotter.js:55` stops snapshotting three hours after first sight, and
  `pruneColdDead` (`:111-133`) drops posts whose velocity is under 500 after ten minutes. So the
  posts that did well have long series and the posts that did badly have short ones. Train on
  that and you learn "posts with lots of snapshots do well," which is true and useless.
- **A random slice cannot be reconstructed after the fact**, because it is defined by what the
  gate threw away, and thrown-away things leave no trace. Without it you can never answer the
  one question that matters: *is our recall limited by the model, or by the gate in front of
  it?*

### What gets built first

Not the rewrite. **The record, retrofitted into the pipeline that is running right now.**

The current build makes a few thousand decisions a day and throws every one of them away. Two
tables and a `finally` block, in the existing code, means that stops today rather than in six
weeks. It also fixes a live bug in the same commit: `api/cron/_lib/run-stage.js:56` writes its
run record only inside the `try`, so a stage that dies looks exactly like a stage that ran and
found nothing. That is how seven hours of ingest death went unnoticed.

Three days of work you will throw away, buying six weeks of data you cannot buy.

---

## 2. Why the current build breaks when you add a platform

This is not a style complaint. Four specific things are wired in a way that means "add TikTok"
means "edit everything."

### 2.1 The folder named `adapters` is a folder, not a boundary

There is a directory called `worker/adapters/`. It looks like the right idea. It is not doing
the job, and there are three verifiable proofs.

**There is a real circular dependency, and someone has already worked around it.**

```
worker/ingest/lib/ingest-floors.js:4   require('../../adapters/x/budget')     ← top of file
worker/adapters/x/budget.js:275        require('../../ingest/lib/ingest-floors')
                                        ← inside the body of loadState()
```

Every other `require` in `x/budget.js` sits at lines 11–12, at the top, like normal. That one is
buried inside a function. That is the standard trick for breaking a require cycle in Node, and
it is load-bearing. A cycle between the vendor adapter and the admission policy is not an
accident of layout; it means the two are one thing.

**The adapters own product policy, and apply it.**

```
worker/adapters/anthropic/meme-score.js:5   require('../../score/lib/meme-gate')
worker/adapters/anthropic/meme-score.js:6   require('../../score/lib/nameability')
```

These are not passive imports. `TT_NO_THUMB_CAP` (0.4) is written out as a final `meme_score` at
lines 196, 298, 320 and 344, and `applyNameabilityCap()` reshapes the returned object at 279 and
355. This module does not return an API response. It returns a product verdict. Separately,
`adapters/x/budget.js` queries Supabase directly. These are services, not adapters.

**There is no port at all — the two platform readers share nothing.**

```
adapters/x/reader.js       exports  searchTweets, getTweetsByIds, getConfig,
                                    buildAdvancedSearchUrl, tweetsInResponse
adapters/apify/reader.js   exports  ACTORS, runActorSync, oldestPostDateIso,
                                    fetchDiscoverVideos, fetchHashtagVideos,
                                    fetchSearchVideos, getTikTokVideosByUrls
```

Not one shared method name. Two files both called `reader.js`, both under `adapters/`, with no
common surface and no registry or dispatch module anywhere in the repo. So the core does the
dispatching itself: `worker/snapshot/snapshotter.js` imports both concrete readers and branches
on `p.platform === 'x'` at lines 79, 85, 255-256, 303-304 and 387. And `worker/ingest/ingest.js`
(789 lines, X) sits beside `worker/ingest/ingest-tiktok.js` (340 lines, TikTok) as two parallel
pipelines that do the same job.

**Adding a platform today is a fork, not an implementation.** That is the whole finding.

One correction to an earlier reviewer's version of this, because it matters for where you spend
effort. That reviewer also said `worker/ingest/lib/near-miss.js:4` importing `adapters/x/reader`
was an "inversion." It is not — core importing an adapter is the correct direction, and there
are 27 such imports across 15 files. The real defect at that line is sharper: `near-miss.js`
writes a `platform` column (line 22) so it is *meant* to be generic, but its recheck path
hardcodes X's `getTweetsByIds` at line 61. **TikTok near-misses get written and never rechecked.**
That is a live functional bug caused by the missing port, and it is better evidence than the
import direction was.

### 2.2 A counter is not a number, and the core assumes it is

The core consumes `views`, `retweets`, `quotes` directly. Those are X's words. They do not
survive the second platform, and they already do not survive the second platform in this repo.

`worker/ingest/lib/parse-tiktok-post.js:82-86` maps `playCount → views`, `shareCount → retweets`,
`commentCount → replies`, and hardcodes `quotes` to `0`. Every one of those is a lie of a
different kind:

- TikTok `playCount` counts autoplays and loops. X `views` counts impressions. They are not the
  same quantity and cannot go in the same column.
- TikTok's counters are rounded to four significant figures. A real gain below the rounding step
  reads as *exactly zero change*. The repo already needed a workaround for this
  (`worker/snapshot/lib/tt-view-diagnostic.js`), which treats the symptom.
- `quotes` is structurally zero for every TikTok row, so any feature including quotes is
  systematically lower for TikTok by schema artifact, not by behaviour.

The consequence is live and it is worse than a measurement error.
`narrativeEngagementRate` (`worker/cluster/lib/cluster-engine.js:193-208`) sums engagement across
all posts in a narrative and divides by summed views across all posts. Narratives are explicitly
expected to span platforms (`isCrossPlatform`, `:226`). So the live `isBoughtReach` gate
(`:211`, wired at `:427`) fires as a function of a narrative's X-to-TikTok mix, not as a function
of whether its reach was bought. **That is a correctness bug in shipped code**, and its cause is
that the core speaks X's vocabulary.

There is a second, quieter version of the same problem. `snapshotter.js:49-50` gates on
`MIN_FEED_VIEWS` (30,000) and `MIN_FEED_VIEWS_TT` (100,000), applied at `:79`. Reddit has no view
count on the public API. There is no threshold to tune; the gate simply has no input. A
views-shaped core cannot admit a platform that does not have views.

### 2.3 Three definitions of the same feature, and the live one is not the one people cite

An earlier pass reported that `worker/lib/velocity.js:13-17` and
`worker/backtest/lib/features.js:60-64` give two different formulas for engagement rate. Both
citations are exact. The conclusion is wrong in a way that would waste a week, so here is the
corrected version.

`engagementRateFromCounts` in `velocity.js` is **dead code**. It has been exported since the
first commit and is imported by nothing. The live production definition is
`narrativeEngagementRate` at `cluster-engine.js:193-208`, which reimplements the weighted
numerator inline even though that same file already imports `engagement` from `velocity.js` at
line 16.

So there are three implementations, and the two live ones differ on four axes, not one:

| | `cluster-engine.js:193` (serving) | `features.js:60` (backtest) |
|---|---|---|
| weights | `likes + 2·retweets + replies` | `likes + replies + retweets + quotes` |
| quotes | omitted | included |
| granularity | narrative-level ratio of sums | per-post ratio |
| time and nulls | latest snapshot, missing views → `0` | T+30min, missing views → `null` |

Only the first three are accidents. **The fourth is deliberate and must survive any fix** — the
backtest reads at T+30 precisely so it does not peek at the outcome. A refactor that "unifies
the definition" by making both call one function would put lookahead back into the backtest. The
correct fix is one function that takes the time window as an argument, not one function called
from both places.

The third row is the one nobody notices: a ratio of sums and a mean of ratios are different
estimators. They do not converge even if you match the weights.

### 2.4 What all of this costs, concretely

Adding TikTok today means: a new ingest file (340 lines and growing), new branches in the
snapshotter at five call sites, a new scoring mode in the Anthropic adapter with its own model
and its own cap, a coerced counter mapping that silently corrupts a live gate, a near-miss path
that does not work, and a budget module borrowed from X (`ingest-tiktok.js:39` imports
`recordPostsIngested` from `adapters/x/budget`) even though the two vendors bill on different
units — twitterapi.io per tweet returned, Apify per actor run.

Adding Reddit means all of that plus discovering that the core's primary signal does not exist.

---

## 3. The adapter layer

### 3.1 The interface

Two ports. Everything a platform can do is one of these two things.

```ts
// contracts/platform.ts — zero runtime dependencies

export type SourceId = string;          // 'x' | 'tiktok' | 'reddit' | '4chan'

export interface PlatformAdapter {
  readonly id: SourceId;
  readonly capabilities: Capabilities;

  /** Find things we have not seen. Costs money. Budget is passed in, never read. */
  discover(q: DiscoveryQuery, budget: Budget): Promise<Discovered>;

  /** Re-read things we already know about, by id. This is the tracking path. */
  observe(ids: string[], budget: Budget): Promise<Map<string, CounterSet>>;

  /** The ONLY function permitted to know this platform's field names. */
  toItem(raw: unknown): Item;

  /** What this platform's numbers mean, so the core can normalise them. */
  baselineKey(item: Item, at: Date): string;
}

export interface Capabilities {
  /** Which counters this platform exposes at all. Absent ≠ zero. */
  counters: readonly CounterKind[];
  /** Can we search by keyword, or only by hashtag / feed / catalog? */
  discovery: readonly ('keyword' | 'hashtag' | 'feed' | 'catalog' | 'account')[];
  /** Does re-reading by id work, and how many per call? */
  observeBatchSize: number | null;
  /** Does this platform expose a reproduction lineage (quote, stitch, crosspost)? */
  lineage: boolean;
  /** Billing unit — this differs in KIND across vendors and cannot be averaged. */
  billing: 'per-item-returned' | 'per-call' | 'per-run' | 'flat';
  /** Fields this platform simply does not have. The core reads this, not a null check. */
  absent: readonly CounterKind[];
}
```

The `capabilities` object is the part that usually gets left out and then costs a rewrite. A port
with one implementation is always secretly shaped like that implementation. Declaring what a
platform *cannot* do, explicitly, is what stops the core from assuming everyone can do what X
can do.

### 3.2 The vocabulary

This is the load-bearing type in the whole system. Get it right and adding a platform is a file.

```ts
// contracts/item.ts

export type CounterKind =
  | 'reach'           // impressions-like:  X viewCount, TikTok playCount
  | 'approval'        // like, digg, upvote, heart
  | 'conversation'    // reply, comment
  | 'rebroadcast'     // a copy that creates NO new authored object: retweet, share
  | 'reproduction'    // a copy that DOES create one: quote, stitch, duet, crosspost
  | 'retention';      // bookmark, save, collect

/** How much you may trust the number. Not decoration — the core branches on this. */
export type Fidelity =
  | { kind: 'exact' }
  | { kind: 'quantized'; significantDigits: number }   // TikTok: 4
  | { kind: 'fuzzed' }                                 // Reddit vote counts
  | { kind: 'absent' };                                // the platform has no such concept

export interface Counter {
  value: number | null;
  fidelity: Fidelity;
  observedAt: number;        // when WE read it, not when they claim it changed
  platformLagMs?: number;    // set only when the platform admits its own staleness
}

export type CounterSet = Partial<Record<CounterKind, Counter>>;

export interface Item {
  itemId: string;            // ours
  source: SourceId;
  sourceItemId: string;      // theirs
  authorKey: string;         // `${source}:${stableId}` — never the handle, handles change
  postedAt: number | null;   // null when the platform lies or omits
  firstSeenAt: number;
  lang: string | null;
  text: string;
  media: MediaRef[];
  counters: CounterSet;
  fingerprints: Fingerprint[];   // pHash | textShingle | soundId | entitySpan
  rebroadcastOf: string | null;  // this item added ZERO new authorship
  reproductionOf: string | null; // this item added ONE new authorship
  formatIds: string[];           // TikTok music.id, effect id, template id
  rawRef: string;                // key into blob storage; never inline
}
```

**Why `rebroadcast` and `reproduction` are separate kinds.** This is not a naming preference, it
is the product thesis expressed as a type. X's own published ranking code
(`retweet_deduplication_filter.rs`) collapses N retweets into one feed candidate, while N quotes
stay N candidates with N authors. Reddit exposes `num_crossposts` as a first-class integer.
"People are making their own versions of this" is the signal we sell; "a lot of people saw this"
is not. Every future adapter has to answer one question: *what here creates a new authored
object?*

**Why `Fidelity` exists.** The core has one rule, in one file, with no platform names in it:

```
emitRate(prev, curr):
  if fidelity is 'absent' or 'fuzzed'                    → Censored('unusable')
  if fidelity is 'quantized'
     and |curr − prev| < quantizationStep(curr)          → Censored('below_step')
  if curr === prev and a sibling counter rose            → Censored('stale_counter')
  if curr < prev                                          → Censored('non_monotonic')
  else                                                    → Rate((curr − prev) / minutes)
```

A censored observation emits **no rate point** and carries the previous value forward. Today
`deltaPerMinute` silently emits `0`, which reads downstream as "cooling" — wrong in the most
expensive direction, because it demotes something that is actually accelerating.

That one function subsumes the TikTok flat-view workaround, the X view-staleness problem, and
vendor corrections. It is roughly fifteen lines and it mentions no platform.

### 3.3 A worked TikTok adapter

```js
// adapters/platform/tiktok/index.js

const CAPABILITIES = {
  counters:   ['reach', 'approval', 'conversation', 'rebroadcast', 'retention'],
  discovery:  ['hashtag', 'feed', 'account'],        // NO keyword search
  observeBatchSize: 50,                               // by URL, via actor
  lineage:    true,                                   // stitch / duet
  billing:    'per-run',                              // Apify bills per actor run
  absent:     ['reproduction'],                       // see the note below
};

/** The only function in the codebase allowed to know TikTok's field names. */
function toItem(raw) {
  const stats = raw.stats ?? {};
  const at = Date.now();

  // playCount and diggCount are rounded to 4 significant figures.
  // commentCount, shareCount and collectCount are exact. Measured, not assumed.
  const q4 = { kind: 'quantized', significantDigits: 4 };
  const ex = { kind: 'exact' };

  return {
    itemId: `tt:${raw.id}`,
    source: 'tiktok',
    sourceItemId: raw.id,
    authorKey: `tt:${raw.author?.id ?? 'unknown'}`,
    postedAt: raw.createTime ? raw.createTime * 1000 : null,
    firstSeenAt: at,
    lang: raw.textLanguage ?? null,
    text: raw.desc ?? '',
    media: thumbnails(raw),
    counters: {
      reach:        { value: num(stats.playCount),    fidelity: q4, observedAt: at },
      approval:     { value: num(stats.diggCount),    fidelity: q4, observedAt: at },
      conversation: { value: num(stats.commentCount), fidelity: ex, observedAt: at },
      rebroadcast:  { value: num(stats.shareCount),   fidelity: ex, observedAt: at },
      retention:    { value: num(stats.collectCount), fidelity: ex, observedAt: at },
      // reproduction: DELIBERATELY ABSENT. See below.
    },
    fingerprints: [
      ...pHashes(raw),                                    // keyframes
      ...(raw.music?.id ? [{ kind: 'soundId', key: `tt:music:${raw.music.id}` }] : []),
      ...textShingles(raw.desc),
    ],
    rebroadcastOf: null,
    reproductionOf: raw.stitchInfo?.sourceId ?? raw.duetInfo?.sourceId ?? null,
    formatIds: [
      raw.music?.id ? `music:${raw.music.id}` : null,
      raw.effectStickers?.[0]?.ID ? `effect:${raw.effectStickers[0].ID}` : null,
    ].filter(Boolean),
    rawRef: `tt/${raw.id}.json`,
  };
}
```

That is the entire adapter's normalisation half, and it is about 60 lines. `discover` and
`observe` wrap the Apify actors already in `adapters/apify/reader.js`; they move, they do not get
rewritten. Nothing in `core/` changes.

### 3.4 Where the abstraction strains

Being honest about this is more useful than pretending it is clean. Five places it strains, and
what to do about each.

**Strain 1 — `reproduction` has no counter on TikTok, only a lineage pointer.** X gives you
`quoteCount` as an integer: you can read reproduction as a level, from a single snapshot, with no
history. TikTok gives you `stitchInfo` on the *child* item, which means you can only count
reproductions by finding the children, which means you can only count the ones you happened to
ingest. So `reproduction` is a cheap level on X and an expensive, incomplete corpus join on
TikTok.

The fix is not to fake a number. It is `absent: ['reproduction']` in capabilities, plus a second
path in the core that derives reproduction from the fingerprint index (how many distinct authors
posted an item sharing this pHash or this sound id). That path is platform-blind and works
everywhere, and it is strictly better evidence — but it costs an index scan instead of a field
read. **The core must therefore accept both and prefer whichever is available**, and must never
compare a platform where reproduction is a counter against one where it is an index scan without
normalising. This is a real seam and it will need attention on the third platform.

**Strain 2 — discovery verbs do not line up.** X has query-syntax search. TikTok has hashtag,
discover and search *actors*. Reddit has subreddit listings. 4chan has board catalogs. There is
no single `search(query)` that means the same thing. The `capabilities.discovery` array is the
honest version: the core asks "which of these do you support?" and builds the query it can. A
uniform `search()` would return an empty array on the platform that cannot do it, which reads as
"nothing is happening" — the exact failure mode we are trying to design out.

**Strain 3 — billing differs in kind, not degree.** twitterapi.io bills per tweet returned
(~$0.00015 each, per `adapters/x/budget.js`'s own header). Apify bills per actor run. A single
`costOf(call)` method cannot express both, and this is already leaking: `ingest-tiktok.js:39`
imports `recordPostsIngested` from **X's** budget module. The port must carry `billing` as a
capability and the meter must have per-vendor units, with the core reasoning only in dollars.

**Strain 4 — `reach` is not the same quantity across platforms and never will be.** Autoplay
views and impressions are different behaviours with the same name. The design decision that
follows is the important one: **no feature in the core may be an absolute counter.** Every
feature is a rate, a ratio to the item's own baseline, or a percentile within its own platform.
This is exactly what Google Trends does to make queries comparable across regions and eras, and
it is the mechanism that makes a model trained on X still valid when TikTok arrives.

**Strain 5 — a platform with no reach at all.** Reddit has no public view count. Under the design
above this is fine in principle: `reach` is `{ kind: 'absent' }`, every reach-derived feature is
null, and a gradient-boosted tree handles missing values natively. But it is fine only *because*
nothing in the core gates on an absolute view count. If the admit stage ever hardcodes a view
floor again — as `snapshotter.js:49-50` does today — Reddit is undetectable. The abstraction
holds only if that rule is enforced, which is why it is a CI check and not a convention.

---

## 4. The core

Seven stages. Every stage is the same shape: a pure function from what we know to a `Decision`.

```ts
// contracts/decision.ts

export interface Decision {
  stage: StageName;               // 'admit'|'track'|'detect'|'group'|'qualify'|'resolve'|'rank'
  subjectKind: 'item' | 'story' | 'pair' | 'candidate';
  subjectId: string;
  verdict: 'pass' | 'hold' | 'drop' | 'abstain';
  reason: string;                 // NAMED, from a closed list, NEVER free text, never null
  score: number | null;           // null when a rule decided
  features: FeatureVector;        // exactly what the decider looked at
  featureSet: string;             // 'item.v3'
  policyHash: string;             // sha256 of the Policy object in force
  decider: string;                // 'rule:admit@3' | 'gbdt:traction@2026-11-02'
  propensity: number;             // (0,1] — how likely this action was
  explore: boolean;
  costUsd: number;
}

export interface Policy { /* every threshold in the system, one frozen object */ }

export interface Stage<In, Out> {
  readonly name: StageName;
  readonly featureSet: string;
  extract(input: In, ctx: Ctx): FeatureVector;      // pure
  gate(f: FeatureVector, p: Policy): string | null; // hard rules; returns a reason or null
  decide(f: FeatureVector, p: Policy, ctx: Ctx): Decision;  // PURE. SYNC. NO I/O.
}
```

**`decide` is synchronous and pure, and this is the mechanism that makes "a rule today, a model
tomorrow" real rather than aspirational.** A function that cannot `await` cannot fetch, cannot
query the database, cannot call a hosted model, and cannot quietly recompute a feature from
fresh data. That forecloses the single most common way leakage gets reintroduced: someone adds
"just one more lookup" inside the decider. It also forces a model to be an in-process artifact,
which is fine — a gradient-boosted tree is a few hundred kilobytes of nested JSON and scoring one
is tree-walking, about 80 lines of plain JavaScript with no new dependencies.

The `reason` field is the thing that let a reviewer diagnose a seven-hour outage with one query.
`deriveGateReason` (`cluster-engine.js:291-339`) is the existing version, and it is good; this
generalises it to every stage. Keep it when the rule becomes a model: a model's reason is
`'model_low:' + topContributingFeature`, so the diagnostic query survives the swap.

Now each stage, stated as a rule today and a model tomorrow.

### 4.1 ADMIT — is this worth spending money to track?

**Rule today.** A linear score over roughly eight terms, with hard gates in front:

```js
gate(f, p) {
  if (f.isRebroadcast)                 return 'rebroadcast_not_original';
  if (f.textLen === 0 && !f.hasMedia)  return 'empty';
  if (f.ageMin > p.admitMaxAgeMin)     return 'too_old';
  return null;
}

score(f) = clamp01(
    0.30 * f.authorRosterTier
  + 0.25 * f.dfCarrierAccel          // corpus-relative — available with ZERO engagement
  + 0.20 * f.hasMedia
  + 0.15 * f.reproductionLevel       // reproduction / reach, a LEVEL not a rate
  + 0.10 * f.namedSpanPresent
  - 0.35 * f.engagementBait
  - 0.25 * f.threadContinuation)
```

**The threshold is a quantile, not a typed number.** Recomputed nightly so that the count of
admits per day lands on the tracking budget. This is the direct repair of the review's finding
that every threshold is a number somebody typed.

**Model tomorrow.** A gradient-boosted tree over ~40 features. Label: did this item join a story
that produced a matched mint that peaked above a threshold. That label is sparse and slow, so
train against a dense auxiliary first — *did at least K independent reproducers appear within six
hours* — which arrives in six hours instead of fourteen days and has no coin in it, so it
survives a market regime change.

**Switch trigger.** ~5,000 positives on the auxiliary head, and the challenger must beat the rule
on recall at a fixed alert budget for fourteen consecutive days without losing recall in the
"far bucket" (see §5.4).

### 4.2 TRACK — when do we look again?

**Rule today.** A geometric schedule with promotion tiers, roughly τ = 4, 9, 14, 21, 30, 42, 58,
78 minutes, with the tier a function of the last score. Budget backpressure drops tiers from the
top down and **never starves probation**, because probation is where lead time is made.

The current implementation gets one thing badly wrong that must not carry forward:
`snapshotter.js:55` stops tracking three hours after first sight, and `:111-133` prunes cold
posts after ten minutes. That means a post's history length is determined by its early
performance, which is the outcome. **Reserve a fixed 2% of arrivals that are tracked on the full
grid regardless of score** — this is the unconditional holdout, and it is the only unbiased
history in the system.

**Model tomorrow.** A contextual bandit, not a classifier: allocate the next read where it most
changes a decision. **Do not build this before ADMIT is a model.** Bandit-before-classifier is a
common and expensive ordering mistake.

### 4.3 DETECT — is this accelerating relative to its own baseline?

This stage costs nothing and produces the earliest alert the system can make.

**Rule today.** Two burst statistics, both cheap, both platform-blind:

```
etaSelf(item, t)        = -log10( P(X ≥ n | λ = item's own trailing baseline) )
etaPopulation(item, t)  = -log10( P(X ≥ n | λ = median for (platform, hour-of-day)) )
```

Both are the Poisson figure of merit from X's own open-sourced trend detection
(`twitterdev/Gnip-Trend-Detection`), which quantifies "how atypical is this count" and calls it
`eta`. About fifteen lines. It directly replaces the typed velocity thresholds, and because it is
a ratio to an expectation rather than a raw count, it is comparable across platforms without
retuning — which is the property we need and the one an absolute threshold cannot have.

Two corrections to earlier research, both of which matter:

- **The variance axis is expected count, not age.** An earlier design normalised by a table of
  standard deviations indexed by post age. Under Poisson-ish noise the spread goes as `1/√μ`, and
  `μ` spans two to three orders of magnitude across authors at a *single* age. A table indexed by
  age cannot flatten that.
- **`log1p` on small counts biases the author baseline downward** in a way that inflates the
  score for low-baseline accounts — a false-positive generator aimed precisely at small accounts.
  Use a negative-binomial fit, which handles zeros natively.

Also: the per-author z-score must never be a standalone trigger, only a re-ranker inside a hard
absolute floor. The author's own baseline is under the adversary's control in both directions —
depress it with filler posts, then buy engagement on the target. **A gate an attacker can open
by buying 100 likes is worse than no gate.**

**Acceleration at the second observation, not the third.** The current
`viewsAcceleration` (`velocity.js:62`) needs three snapshots, which is fourteen minutes at best. A
pair of continuous-time exponential moving averages — one fast (~20 min), one slow (~6 h) — gives
`burst = fast / slow`, and `burst − 1` is a curvature proxy available at the *second* observation.
Five minutes earlier, on a product that sells earliness.

Use continuous-time decay (`w = exp(−Δt / τ)`), not the textbook `α·x + (1−α)·S`. The sampling grid
is irregular by design, and the discrete form biases the frequently-sampled posts upward
mechanically.

**Model tomorrow.** One gradient-boosted tree per horizon (M60 first, then M15, M5). **Build M60
first** — it has the most signal, so it validates the whole pipeline on the easiest version of
the task. If M60 cannot beat the heuristic *and* cannot beat a follower-count-only baseline,
nothing downstream will.

### 4.4 GROUP — which story is this?

This is the highest-leverage single change in the system, and the current implementation is the
worst thing in it: `worker/cluster/lib/embeddings.js:5-15` is a term-frequency map with no IDF and
no learned representation. **982 of 1,079 populated stories contain exactly one post by one
author — 91%.** The product's premise is that people are making their own versions, and the
grouper cannot see a second version.

**Two tiers, and the free one does most of the work.**

**Tier 1 — carrier join. Free, deterministic, language-blind, model-independent.** Any item
sharing an exact carrier with an existing story joins it, with no scoring and no API call:

| Carrier | Match rule | Cost |
|---|---|---|
| image pHash (PDQ-256) | Hamming ≤ 31 | ~2 ms CPU |
| text near-duplicate (SimHash-64) | Hamming ≤ 3, ≥ 6 shingles | ~50 µs |
| format id (sound, effect, template) | exact | free |
| reproduction pointer (quote, stitch, crosspost) | exact | free |

**A perceptual hash has no language and no platform.** The same image posted on X and on TikTok
produces the same 256 bits. This tier is what makes the grouper multilingual and cross-platform
*before any model exists*, which is why it ships first and alone.

Store both hashes as pgvector `bit` columns with `hnsw (bit_hamming_ops)` indexes. Near-duplicate
search becomes an index in the Postgres you already run — no Faiss, no vector database, no new
service. **This is the single highest-leverage infrastructure decision in this document**, because
it is what makes a replication detector operable by two people at all. If HNSW recall on a
Hamming radius of 31/256 turns out poor — plausible, since that is a loose radius — the fallback
is Manku's four-table permutation blocking, about 120 lines and no index.

**Tier 2 — embeddings, for what tier 1 missed.** A real sentence embedding, stored as
`halfvec(768)`, HNSW cosine. Roughly $0.11/day at 9,000 items.

Two things must happen before this is worth anything:

1. **Remove `lang:en` from `worker/ingest/lib/ingest-query.js:11`.** No embedding model of any
   quality can improve multilingual grouping while the ingest is English-only. Moo Deng's earliest
   signal was a Thai-language post at 4.6M views, seven to eleven days before the English wave.
2. **Do not reuse the threshold.** `SIM_THRESHOLD = 0.42` (`cluster.js:35`) is calibrated to
   sparse term-frequency geometry. Dense sentence-embedding cosine has a completely different
   distribution — unrelated sentence pairs sit well above 0.42. Dropping an encoder in behind the
   same constant would merge nearly everything.

**A correction that changes what you build.** An earlier pass said the two existing tiers are not
redundant and that 21.4% of matches are embedding-only, so the embedding tier must be replaced
rather than deleted. The 21.4% was measured on synthetic pairs, not on this corpus. The real
arithmetic: for repeat-free token vectors, `cosine ≥ 2J/(1+J)`, and at the live keyword threshold
of `J = 0.28` the minimum cosine is 0.4375, which is above `SIM_THRESHOLD = 0.42`. So **the
keyword tier is a strict subset of the embedding tier — it contributes zero additional matches.**
And the embedding tier, having no IDF and no length penalty, is an active false-positive source:
"Behind-the-scenes of Adwoa Aboah's…" matches "Behind the scenes with the champs" at 0.436.

The conclusion survives (build a real encoder) but the reason is different, and one cheap
experiment should run first: the `cluster_match` column already exists
(`worker/schema-cluster.sql:18`, written at `cluster.js:175`) and would answer the tier question
exactly — except it is NULL on every row in production. Fix that write, run one `GROUP BY`, and
you have the real number instead of a simulated one.

**Term weighting: persistence, not raw IDF.** This is counter-intuitive and it matters. Raw IDF
treats high document frequency as evidence of ambience, so a genuinely new ticker climbing from
1 to 80 documents inside 24 hours gets down-weighted *exactly when it matters most*. Use:

```
persistence(t) = (number of the last 14 daily buckets where df(t) ≥ 3) / 14
w(t)           = idf_24h(t) × (1 − persistence(t))
w(t)           = 0  if t ∈ GENERIC_TICKERS
```

`$SOL` is high-df in 13 of 14 buckets, so persistence ≈ 0.93 and weight ≈ 0. A brand-new ticker
spikes in the last two buckets, persistence ≈ 0.14, near-full weight. **The 14-day history accrues
forward only — start writing the daily bucket table today or this feature does not exist in
November.**

`GENERIC_TICKERS` already exists — 45 entries in `worker/score/lib/nameability.js`, imported by
three modules and by nothing in the clustering path. One import line. Highest value per line of
code in the repository: one crypto story accreted 39 unrelated posts because they all said `$SOL`.

**Model tomorrow.** A logistic regression over the six match features, replacing the hand-set
weights, fit on 300 hand-labelled *pairs* ("same real-world moment? yes/no", about 30 seconds
each, roughly three hours of work once). Then, only if the adjudication rate stays above 12%, a
cross-encoder for the ambiguous band.

### 4.5 QUALIFY — is there a nameable, coinable thing here?

**Rule today.** One LLM call with structured output, plus deterministic server-side caps the model
cannot override (`applyNameabilityCap` is the existing version and it fires on 18% of rows, which
is why it should be kept).

Three fixes, all measured:

1. **Run it after promotion, not before.** Today `cluster-engine.js:403` writes the title and
   `:432-448` decides whether the story is even eligible to display — 45 lines apart, in the wrong
   order. On 2026-08-05 that was 1,240 of 2,893 calls (43%) against 103 stories created, roughly
   twelve regenerations each. Reordering cuts that line by about 40×.
2. **Batch ten stories per call.** About 82% of the spend is one static prompt retransmitted per
   item.
3. **Structured output**, deleting the hand-rolled JSON-fence-stripping path.

Expect 50–70% to come back "not coinable" and render with no Buy affordance. That is the correct
outcome. A design that optimises this number upward reintroduces the wrong-coin bug.

**Model tomorrow. This is the first stage to become a model**, not the last, because its label is
free and dense: *did a matching mint appear within six hours.* That arrives off the mint stream
forever, at no cost. Switch trigger: ~2,000 labelled stories.

One caution that must ship with it. About 30,000 coins are minted per day at roughly 97% noise
and a median fresh-mint market cap of $28. A QUALIFY model trained naively on "a coin appeared"
becomes a spam detector that passes anything a bot would coin. **Gate the label on a traction
floor, not on mint existence**, and keep the deterministic nameability and genericity rules as
hard constraints outside the model.

### 4.6 RESOLVE — which coin, if any?

This is the stage where being wrong costs a user money, so it is the last one to get a model and
the one with the most hard gates. Full treatment in §6.

### 4.7 RANK — one primitive, both lanes

```js
function heat(o, cfg) {
  const base = o.rateLcbNorm * Math.sqrt(o.burst);      // shrunk rate × scale-free burst
  return Math.pow(base * o.quality, cfg.alpha)
       / Math.pow((o.ageMin + cfg.t0) / cfg.t0, cfg.gamma);
}
// alpha = 0.85, gamma = 1.35, t0 = 12 min
```

The stories lane and the coins lane are the same shape — a shrunk arrival rate of a countable
event, times a scale-free burst ratio, penalised by age, scaled by a quality multiplier. Only the
counter and the quality function differ. Stories count reach; coins count volume.

Two properties that are product requirements, not tuning:

**Candidate isolation.** Every term depends only on the item being scored. No term references
another item. This is what X's Phoenix ranker does (candidates cannot attend to each other during
inference), and here it means the board does not reshuffle because an unrelated story arrived,
and any row's position is reproducible from its own stored feature vector. The cost is real and
should be recorded as a decision: you give up list-level objectives like "do not show three coins
from the same meme," which then has to be handled as a post-selection filter.

**The board is committed server-side and the client never sorts.** A client-side sort over a
live-updating value is what makes boards flicker, and no amount of smoothing fixes it. The server
publishes `(tick, rank)`; the client renders the order it is given and patches values in place.
Add hysteresis: a challenger must beat the incumbent by 8% to swap, two ticks to enter, three to
leave, a 90-second minimum dwell, and a maximum of five positions moved per tick — with one
escape hatch so a genuinely explosive new entrant skips the dwell and enters with a NEW badge.

**The metric nobody instruments and should:** Kendall tau between consecutive ticks. Target ≥ 0.90
at a 20-second tick. Below 0.85 the board is unusable; above 0.98 for hours it is frozen.

**Before building any of this, run Hacker News's `(P−1)/(T+2)^1.8` as an offline control** against
the existing 57,032-row snapshot series, scored on early-detection precision. If the control wins,
"HN with hotter constants" was the right answer and a cheap experiment saved a month. That is a
real possibility, not a formality.

**Model tomorrow. RANK is the last stage to become a model, not the first.** It needs board
impressions and clicks, which do not exist yet and will not for months. Getting this ordering
wrong is how a small team spends a quarter training the model that matters least.

---

## 5. The learning substrate

This is the part that cannot be retrofitted, so it is stated in more detail than the rest.

### 5.1 The problem, in one sentence

**The decision happens in minutes and the answer arrives in days**, so anything you did not write
down at the moment of the decision is gone.

The numbers: post-to-mint is a median 3.8 minutes; post-to-peak is about six days. That is a gap
of roughly 2,000×. Every decision is made on minute-scale evidence and graded on day-scale
evidence, and the interval between them is exactly when the world changes underneath any feature
you might try to recompute.

This is not a theoretical concern. It has already cost this project two studies:

- The backtest's three "strongest" features — *crypto noticed it*, *ticker in replies*, *others
  copying it* — were read by a human opening the origin post's reply thread **months after the
  coin ran**. A coin that 10×'d gets shilled in its origin post's replies for months. "Crypto
  noticed it" is a near-mechanical consequence of the outcome, not a predictor of it.
- The validation harness derived its "coined" label from the model's own output.

The team's demonstrated failure mode is building a measurement that measures itself. **Freezing
the feature vector before the outcome exists makes that class of error structurally impossible
rather than a thing to be careful about.** Carefulness has already been tried here, four times,
and lost.

### 5.2 The decision log

```sql
create schema internal;   -- NOT in PostgREST's exposed schemas. See §5.6.

create table internal.decisions (
  id              bigserial,
  decided_at      timestamptz not null default now(),

  stage           text not null,   -- admit|track|detect|group|qualify|resolve|rank
  subject_kind    text not null,   -- item|story|pair|candidate
  subject_id      text not null,   -- 'x:1823…' | 'story_7f3a' | 'story_7f3a|sol:9xQe…'

  -- THREE separate clocks. Conflating them is how leakage comes back.
  feature_asof    timestamptz not null,   -- newest input datum used
  subject_origin  timestamptz,            -- post created_at / mint block_time
  horizon_s       integer,                -- decided_at − subject_origin

  verdict         text not null,          -- pass|hold|drop|abstain
  reason          text not null,          -- named, closed vocabulary, NEVER null
  score           double precision,       -- null when a rule decided

  features        jsonb not null,         -- EXACTLY what the decider saw
  feature_set     text not null,          -- 'item.v3'
  feature_hash    text not null,          -- sha256 of the sorted key list

  policy_hash     text not null,          -- the thresholds it was judged against
  decider         text not null,          -- 'rule:admit@3' | 'gbdt:traction@2026-11-02'

  propensity      double precision not null check (propensity > 0 and propensity <= 1),
  explore         boolean not null default false,
  explore_arm     text,                   -- 'epsilon' | 'holdout' | null
  log_sample_rate double precision not null default 1.0,

  shadow_of       bigint,                 -- a challenger's row points at the champion's
  cost_usd        numeric(12,8) not null default 0,
  applied_at      timestamptz,            -- set AFTER the side effect succeeded

  constraint no_lookahead check (feature_asof <= decided_at)
) partition by range (decided_at);

create index on internal.decisions (subject_kind, subject_id);
create index on internal.decisions (stage, decided_at desc);
create index on internal.decisions (explore_arm, decided_at desc) where explore;
```

Six things in that schema are doing real work:

**`features jsonb`, not typed columns.** The feature set will change weekly for the first year.
Columns mean a migration per change, and this repository already has 23 loose `schema-*.sql`
files with 11 different appliers, no ordering, no version table, and two migrations that were
written and never applied to production. Requiring a migration per feature change guarantees the
feature set stops evolving. Promote three to five features to generated `STORED` columns for the
ones that appear in `WHERE` clauses; leave the rest in jsonb.

**`feature_asof <= decided_at` as a CHECK constraint.** Three words, and it is a
database-enforced no-lookahead guarantee. If a future refactor ever reads a row written after the
decision, the INSERT fails loudly instead of quietly producing a better-looking model.

**`policy_hash`.** The review found that `meme_min` is computed, returned and logged but never
persisted — so no past decision can be audited after a config change. You cannot write a decision
here without recording the bar it was judged against.

**`decider`.** This is why the whole thing works before any model exists. Today it says
`'rule:admit@3'`. In November it says `'gbdt:traction@2026-11-02'`. The rows join to the same
labels through the same `subject_id`. **The log does not care what decided.**

**`shadow_of`.** A challenger runs on the *same frozen feature vector*, immediately after the
champion, writes a second row, and changes nothing. Zero product effect, one extra pure function
call. Champion-versus-challenger comparison becomes a query, not a deploy. This is only possible
because features are an object you can pass around — the same property that makes logging work.

**`propensity` and `explore`.** See §5.4.

**Volume.** Roughly 30,000 arrivals/day × ~5 stages ≈ 150,000 rows/day at ~1 KB ≈ 150 MB/day.
Daily partitions; keep full features 90 days; keep 100% of explore, holdout, error and any
subject that ever produced a matched mint, forever; downsample plain ADMIT drops to 10% beyond 90
days — and set `log_sample_rate` on those rows so training reweights by `1/rate`. A downsample you
forgot to record is a biased training set.

**Log first, then act, then mark applied.** Supabase's client has no multi-statement transaction,
so the decision and its side effect cannot be atomic without a stored procedure. Choose the
asymmetry deliberately: a logged decision whose side effect failed leaves `applied_at IS NULL`,
which is detectable and recoverable. A side effect with no log is a permanent, invisible hole —
and one that is *correlated with failures*, so it biases exactly where bias hurts most.

### 5.3 The delayed outcome join

```sql
create table internal.labels (
  subject_kind    text not null,
  subject_id      text not null,
  label_name      text not null,           -- 'peak_multiple' | 'reproducers_6h'
  label_version   text not null,
  window_days     integer not null,

  origin_ts       timestamptz not null,    -- the clock the window measures from
  resolves_at     timestamptz not null,    -- origin_ts + window_days
  status          text not null check (status in
                    ('pending','resolved','censored','unresolvable')),
  value           double precision,
  y               boolean,                 -- thresholded; null unless resolved
  censor_reason   text,

  population      text not null,           -- ★ NOT NULL, NO DEFAULT
  source          text not null,           -- 'dune:peak_multiple_v1@<git sha>'
  first_signal_at timestamptz,             -- when the FIRST evidence arrived
  computed_at     timestamptz,

  primary key (subject_kind, subject_id, label_name, label_version, window_days)
);
create index on internal.labels (status, resolves_at) where status = 'pending';
```

**`population NOT NULL` with no default is the most important line in this document that is not
about logging.** The backtest failed because its population was graduated coins — about 107 a day
against roughly 30,000 mints, well under half a percent — while its claim was about coinability
in general. Making the population impossible to omit turns a discipline into a constraint. The
review's own closing rule was "every measurement declares its population, its label source, and
its window, in the artefact itself, next to the number." This is that rule as a column.

**Four states, and everyone gets the third one wrong.**

- **resolved / positive** — window closed, threshold cleared.
- **resolved / negative** — window closed, nothing happened, **and we watched the whole window**.
  That last clause is only assertable against a coverage log of the mint stream.
- **pending** — `now() < resolves_at`. **Excluded from training. Never coerced to negative.** This
  is the classic error and here it would be severe: with a six-day median to peak, the pending
  population is large relative to the resolved one for the first several months.
- **censored** — the mint stream had a gap over the window, or the post was deleted, or the view
  counter was stale. Excluded from training but **counted**, because a rising censoring rate is
  the earliest sign the label pipeline is rotting.

**`first_signal_at` costs one column today and is unrecoverable tomorrow.** Recording when the
first matching mint appeared — even while the row is still pending — gives you the empirical delay
distribution `P(delay ≤ t)`. In six months that is what makes the delayed-feedback correction
possible: include pending rows as negatives with weight `1 / P(delay ≤ elapsed)`, which is how
Twitter's own CTR work handles exactly this problem. Ship the column now, build the correction
later.

**The training view — the artefact everything else exists to produce:**

```sql
create or replace view internal.train_admit_v1 as
select
  d.decision_id, d.decided_at, d.horizon_s, d.subject_id,
  d.features, d.feature_set, d.policy_hash, d.decider,
  d.propensity, d.explore, d.explore_arm,
  l.y, l.value as label_value, l.label_version,

  case when d.explore then 1.0 / d.propensity else 1.0 end          as ips_weight,
  power(0.5, extract(epoch from (now() - d.decided_at))/(21*86400)) as recency_weight,
  1.0 / d.log_sample_rate                                           as log_weight
from internal.decisions d
join internal.labels l
  on  l.subject_kind = d.subject_kind
  and l.subject_id   = d.subject_id
  and l.label_name   = 'peak_multiple'
where d.stage  = 'admit'
  and d.shadow_of is null
  and d.applied_at is not null       -- the side effect actually happened
  and l.status   = 'resolved'
  and l.resolves_at < now()          -- no open windows
  and l.origin_ts >= d.decided_at    -- ★ THE ANTI-CIRCULARITY CLAUSE
;
```

That last line is the one-line prevention of the failure that voided the last backtest, where 10
of 62 rows had an "origin" post published *after* the coin already existed. In SQL, once, forever.

**Walk-forward with a purge.** When you split by time, a training example decided on day D−3 whose
label window covers day D shares outcome information with the test set. So:

```
train:  decided_at ∈ [D−90, D−1)  AND resolves_at < D    ← the purge
test:   decided_at ∈ [D, D+1)     AND status = 'resolved'
```

Step one day, aggregate over at least 60 test days, and **report the distribution across days,
not the mean** — a mean hides exactly the regime episodes the product exists to catch.

### 5.4 Exploration

**A deterministic policy has propensity 1 for the action it took and 0 for everything else, which
makes counterfactual estimates undefined.** No amount of cleverness recovers this after the fact.
You must inject randomness at decision time or off-policy evaluation is impossible in principle.
This is why Netflix injects controlled randomisation into artwork selection and logs the
propensity, and why the ZOZO open bandit dataset ships a uniform-random arm alongside its live
one.

**Two lanes, and they cost different currencies.**

| Lane | Share | Selected how | Shown? | Costs | Buys |
|---|---|---|---|---|---|
| exploit | 90% of slots | top-ranked | yes | — | the product |
| ε-explore | 10% of slots | uniform from eligible-but-below-cut | yes | **feed quality** | unbiased estimates near the boundary |
| holdout | 2% of *arrivals*, pre-gate | uniform, bypasses every gate, tracked and matched | **no** | **tracking budget only** | the only measurement of what the gates miss |

The holdout is the one almost nobody builds and it is the most diagnostic number you will ever
have. It is a shadow lane: the items are tracked, feature-extracted, logged and label-joined, and
nothing is rendered. **It costs about 2% of tracking reads and zero feed quality.**

```js
/** Deterministic on the subject key, so it is stable across restarts and reproducible. */
function isHoldout(subjectKey, rate = 0.02, salt = 'holdout.v1') {
  const h = createHash('sha256').update(salt + '|' + subjectKey).digest();
  return (h.readUInt32BE(0) / 0xFFFFFFFF) < rate;
}
```

**Write the cost of ε-exploration into the product spec with a number attached, or it gets
quietly cut the first bad week.** If the ranker's precision at the alert budget is `p` and random
selection's is roughly the base rate `p₀`, ε-exploration costs `ε·(p − p₀)` of realised precision.
At `p = 0.30`, `p₀ = 0.02`, `ε = 0.10`, that is **2.8 percentage points** — about 0.6 winners a day
not surfaced on a 20-row board. That is the price. It is a real price and it should be a written
commitment with an owner, because a principle without a number does not survive a bad week.

**If budget pressure forces a cut, cut ε before the holdout.** ε is recoverable by re-enabling it.
The holdout's absence is a permanent hole in the record.

**Be precise about what inverse-propensity weighting is valid over**, because the common failure
is computing it across the whole log and reporting a number that means nothing:

- **exploit lane:** propensity 1.0. Estimates nothing counterfactual. On-policy metrics only.
- **ε lane:** propensity strictly between 0 and 1. Valid for off-policy estimates over the
  eligible pool. Use the self-normalised estimator, not plain IPS — plain IPS has unbounded
  variance when a propensity is small, and at 2 picks from a pool of 200 you get weights of 100.
- **holdout lane:** uniform over arrivals. Valid for unbiased replay evaluation of *any* candidate
  policy over the whole arrival stream. **This is what lets you ask "what would a 5,000-view floor
  have caught?" and get a trustworthy answer.** The exploit lane cannot answer that at any sample
  size.

The two numbers that stay on the dashboard forever:

```sql
-- Is the GATE or the MODEL the recall bottleneck?
with lanes as (
  select case when d.explore_arm = 'holdout' then 'holdout' else 'gated' end as lane,
         d.verdict, l.y
  from internal.decisions d
  join internal.labels l using (subject_kind, subject_id)
  where d.stage = 'admit' and d.shadow_of is null
    and l.label_name = 'peak_multiple' and l.status = 'resolved'
    and d.decided_at between now() - interval '56 days' and now() - interval '14 days'
)
select lane,
       count(*) filter (where y)                          as positives,
       count(*) filter (where y and verdict = 'pass')     as caught,
       round(100.0 * count(*) filter (where y and verdict='pass')
             / nullif(count(*) filter (where y),0), 1)    as recall_pct
from lanes group by lane;
-- holdout recall << gated recall  ⇒  THE GATE is the bottleneck, not the model.
```

And **`recall_far_bucket`**: recall stratified by distance from each test positive to the nearest
*training* positive. If the far bucket collapses, the model has become a derivative-finder no
matter what the headline number says. **Nothing else on the dashboard is worth anything without
this one**, because a system that only finds things resembling past winners is structurally
incapable of finding the next unlike-anything meme — which is the entire product.

Related and enforceable: **no feature may be an absolute semantic position.** No cosine similarity
to past winners, no nearest-winner distance, no winner-corpus topic membership. Embeddings are
permitted only *relationally* — distance to the centroid of the candidate's own reproducer set,
dispersion within that set, drift of that centroid over time. Enforce it as a code boundary:
nothing under `core/features/` may import the winner corpus. That is lint-checkable, and the cost
of the ban is close to zero — in the one published study on this exact task, ablating visual
features changed the headline metric by 0.00 and ablating contextual features *improved* it.

### 5.5 Drift, and telling "the model broke" from "the market moved"

Two layers on two clocks, and the split is itself the diagnostic:

| Layer | What | Window | Cadence |
|---|---|---|---|
| slow | the ranking model | 180 days, 21-day half-life recency weight | **weekly**, fixed day, a human reads a report |
| fast | calibration (isotonic, score → probability) | last 7 resolved days | daily, automatic |
| threshold | the alert-budget quantile | last 7 days of scores | daily, automatic |

**This deliberately downgrades an earlier recommendation of nightly retraining.** A nightly
retrain implies nightly monitoring, and two people will not read a nightly report — so it will run
unwatched until it lands a bad model through a regime break. Weekly, on a fixed day, with a human
reading a generated report, is the version that survives contact. The fast layer stays daily
because it is about fifty lines of isotonic regression with a hard monotonicity constraint: it
cannot produce a surprising model, and it is what absorbs market movement between retrains.

Then the diagnosis is a table:

| Ranking PR-AUC (7d vs 28d) | Calibrator's implied base rate | Input drift (PSI) | Verdict | Action |
|---|---|---|---|---|
| stable | stable | < 0.1 | healthy | nothing |
| stable | moved > 2× | < 0.1 | **the market moved** | already absorbed; record a regime note |
| **down > 20%** | stable | < 0.1 | **the model broke** | roll back to the frozen champion |
| down | moved | any | **regime break** | roll back, retrain on 60 days, refit calibration |
| stable | stable | **> 0.25 on one feature** | **schema / vendor drift** | check null-rate-by-cohort FIRST |

The last row is the one nobody builds and the one most likely to fire here. In eighteen months:
`king_of_the_hill_timestamp` went from ~100% populated to 0%; `raydium_pool` went null when
graduation moved to PumpSwap; TikTok's `SIGI_STATE` container vanished; Reddit closed
unauthenticated JSON; X replaced its entire ranking stack. **Every one of those corrupts a feature
or a label silently rather than throwing.**

The monitor for it is a per-field null rate **by cohort week**, not in aggregate:

```sql
select date_trunc('week', decided_at)::date as wk, count(*) as n,
       round(avg((features->>'reproduction_level' is null)::int)::numeric, 3) as null_repro,
       round(avg((features->>'df_carrier_accel'   is null)::int)::numeric, 3) as null_df,
       round(avg((features->>'reach'              is null)::int)::numeric, 3) as null_reach
from internal.decisions
where stage = 'admit' and decided_at > now() - interval '12 weeks'
group by 1 order by 1;
-- A field that goes null for NEW rows while old rows stay populated is the signature,
-- and it is invisible to any aggregate null check.
```

Also on the panel: **prediction drift**, which is label-free and fires hours before any label
exists (hourly mean, p50, p90 of score and the pass rate); and **feature-importance churn**, the
rank correlation of the top five importances against 28 days ago. That second one is the most
interpretable number available — author-follower features overtaking kinetic features *is* the
celebrity-launch regime arriving, visible as one number.

**Feed the regime in as features rather than branching on it:** trailing-7-day count of coins
clearing the peak threshold, share of top coins sourced from large accounts, median observed
post-to-mint lag, pooled graduation rate. Computed nightly into the decision context. One model
spans regimes instead of needing a model per regime — which matters because you never know you are
in a new regime until afterwards, and two people cannot maintain a model zoo.

### 5.6 The exposure rule, made structural

`internal` is not in PostgREST's exposed-schemas list. It is unreachable by the anon key at any
URL. Today `worker/schema-score.sql:23-26` grants anon SELECT on `post_meme_scores` with
`using (true)` — including the judge's plain-English `reason` and the raw API response — and the
anon key is published at `site/config.js:8`. Two reviewers pulled verbatim judge sentences using
the site's own key.

`internal.decisions` contains strictly more: every feature value, every threshold, every
propensity, every exploration flag. That is the recipe for gaming the gates. **Under a schema
split there is no policy to get wrong, because PostgREST never sees the schema.** The product rule
"our scoring never leaves the building" becomes a property of the grant, not a code-review habit.

---

## 6. The venue layer

### 6.1 Two levels, because the unit of variation is not the chain

pump.fun and Raydium differ more from each other than Raydium differs from Uniswap. A flat "chain
adapter" would push those differences into `if` statements inside the Solana file — the same
failure we are fixing one layer up.

```
Chain  — a settlement layer.  Owns: address format, decimals, finality, explorer.
Venue  — a market on a chain. Owns: creation events, price semantics, fees, quotes.
```

This is not speculative. **The second venue already exists inside Solana today, and it is already
causing the highest-consequence bug in the product.**

### 6.2 The finding that reframes this whole section

The coin matcher's real defect is not only that it matches by symbol string. It is arithmetically
forced by a liquidity filter.

```js
lib/token-lookup.js:29    return Number(pair.liquidity?.usd) || 0;   // ABSENT becomes 0
lib/token-lookup.js:153   if (mapped.liquidity <= 0) return { found: false, ticker: sym };
worker/cluster/lib/enrich-tickers.js:10
                          if (!result.found || !(Number(result.liquidity) > 0)) return null;
```

Measured live against DexScreener, 5 August 2026:

| venue class | pairs | missing `liquidity` object |
|---|---|---|
| `pumpfun` (bonding curve) | 19 | **19 (100%)** |
| `meteoradbc` (bonding curve) | 4 | **4 (100%)** |
| `raydium` / `pumpswap` / `fluxbeam` (AMM) | 46 | **0** |
| `orca` / `meteora` (AMM) | 52 | 2 |

A bonding curve has no two-sided reserve, so the vendor returns **no liquidity object at all** —
absent, not zero. Line 29's `|| 0` collapses "this venue has no reserve concept" and "this pool
was drained" into the same value, and both call sites then reject.

Four real, live, tradeable bonding-curve tokens with market caps between $7k and $21k, run through
the actual production function: **all four returned `found: false`.**

**So the filter is a survivorship filter, not a quality filter.** It removes essentially the whole
pre-graduation population and keeps only tokens old enough to have an AMM pool.

Two corrections to how this was previously stated, both of which change what you build:

**It is a venue-class property, not a pump.fun property.** Meteora DBC shows the same behaviour on
the same chain today, with no pump.fun involvement. It will hold for every launchpad curve on
every chain, including Base's Clanker and Zora. Any interface that treats `liquidity > 0` as the
"does this market exist" test breaks on its second venue, silently.

**And the dangerous output is not the obvious one.** The common failure is abstention —
`lookupTicker('MOON')` returns `found: false`. The *harmful* failure is a plausible survivor:
`lookupTicker('GYM')` returns "Gym Showdown", a 65-day-old graduated token with $61k liquidity,
which looks like a perfectly reasonable match for a gym-day story. Naming `$PUMP` makes the bug
sound loud and self-evident. It is neither. **A quiet, plausible, wrong answer is much harder to
detect than an absurd one**, and it is what a user's money gets spent on.

**Removing the filter is necessary but not sufficient.** `comparePairs`
(`lib/token-lookup.js:47-54`) ranks by SOL-quote, then 24-hour volume, then liquidity. With the
gate gone, a fresh $20k curve still loses to an established pair on volume, so `pickBestPair`
keeps returning the survivor. GYM would still resolve to Gym Showdown. And symbol search cannot be
the discovery mechanism anyway: DexScreener's search endpoint returned HTTP 429 after about
eleven sequential calls, while `enrich-tickers.js:7` runs a 250 ms delay — 240 requests a minute,
straight into that ceiling.

### 6.3 Mint time, which is the axis everything hangs on

An earlier pass said `pairCreatedAt` is absent on about a quarter of pump.fun pairs, and concluded
that mint time must come from a streaming subscriber. Re-measured on a sample ten times larger,
the same day: **12 of 215 pump.fun pairs (5.6%), and 0 of 57 mints under three minutes old.** The
nulls were all stale low-cap rows. On the population the product actually serves, the null rate is
zero.

**The real defect is worse and runs the other way: when the field is present, it is often wrong.**
Comparing `min(pairCreatedAt)` against the launchpad's own `created_timestamp` for the top 60
coins: 20 of 60 returned no timestamped pair at all, and of the 40 that did, the median lag was
+22 minutes, 24 of 40 were more than 10 minutes late, and the tail ran to +85 hours, +137 hours,
+979 hours and +9,743 hours. The cause is mechanical: DexScreener drops the original bonding-curve
pair after migration, so the earliest surviving pool is the migration pool.

Against a measured 3.8-minute median post-to-mint lag, a mint time off by 22 minutes **silently
reverses the ordering the gate exists to enforce** — a post made *after* the mint passes as
pre-mint. Failing closed on null cannot catch that, because the failure mode is a confident wrong
number, not a null.

So mint time becomes a stored column with its source recorded, never a derived read:

```sql
create table public.asset (
  chain             text not null check (chain in ('solana')),   -- add rows, not columns
  address           text not null,
  caip19            text not null unique,   -- 'solana:5eykt…/token:9xQe…' — the wire id
  venue_id          text not null,          -- 'solana:pumpfun' | 'solana:amm'

  minted_at         timestamptz,            -- NULL IS LEGAL and means UNKNOWN
  minted_at_source  text not null check (minted_at_source in
                      ('launchpad_api','chain_rpc','vendor_field','none')),
  minted_at_conf    text not null check (minted_at_conf in ('exact','bounded','unknown')),
  minted_at_bound_s integer,

  -- A vendor field can NEVER be 'exact'. A database invariant, not a convention.
  constraint exact_requires_real_source check
    (minted_at_conf <> 'exact' or minted_at_source in ('launchpad_api','chain_rpc')),
  constraint bounded_requires_width check
    (minted_at_conf <> 'bounded' or minted_at_bound_s is not null),

  symbol            text,     -- OBSERVED. Not an identifier. No unique constraint. Ever.
  name              text,
  image_uri         text,
  decimals          int,
  creator           text,
  declared_social   jsonb,    -- attacker-controlled; the column name says so
  first_seen_at     timestamptz not null default now(),
  primary key (chain, address)
);
create index asset_time_idx on public.asset (chain, minted_at desc) where minted_at is not null;
create index asset_sym_idx  on public.asset (chain, upper(symbol), minted_at desc);
```

**How to populate it, cheaply.** Poll the launchpad's REST list endpoint, which carries
`created_timestamp` (present on 130 of 130 coins pulled today), and confirm with one generic
`getSignaturesForAddress(mint, {limit:1000})` whose oldest `blockTime` matched the launchpad
timestamp to the second on fresh mints. If they disagree by more than about 60 seconds, the
confidence is `unknown` and no Buy affordance renders.

**Do not build a streaming subscriber for this.** Two reasons. First, this deployment is entirely
serverless (`vercel.json`: `maxDuration: 60`, cron floor two minutes), so a long-lived Geyser
subscription means a new always-on deployment target, a paid RPC tier, reconnect logic, a durable
cursor, gap backfill and permanent pager surface — roughly a week of build plus ongoing burden.
Second, it buys nothing: the winnable window is post-mint, with a six-day median to peak, so 30–60
seconds of polling latency is free. **A creation stream is also venue-specific, not chain-specific
— you decode one program's create instruction per launchpad — so it is the wrong thing to hang the
chain abstraction on anyway.**

*(This corrects an earlier position in these documents that made the mint stream the justification
for the venue layer. It is not. The justification is quoting and executing, below.)*

### 6.4 The interface, split by capability

```ts
export type ChainId = 'solana';                      // add values, not columns
export type VenueId = `${ChainId}:${string}`;        // 'solana:pumpfun' | 'solana:amm'
export interface AssetRef { chain: ChainId; address: string }   // NEVER a bare string

export type Capability = 'watch' | 'read' | 'assess' | 'trade' | 'create';

export interface Venue {
  readonly id: VenueId;
  readonly chain: ChainId;
  readonly market: 'bonding-curve' | 'amm';
  readonly capabilities: readonly Capability[];
  readonly enabled: boolean;        // a broken venue is DISABLED, not deleted
  watch?: VenueWatch; read?: VenueRead; assess?: VenueAssess; trade?: VenueTrade;
}

/** EVERY field nullable, and every null is DISTINCT FROM ZERO. The §6.2 bug as a type. */
export interface MarketState {
  asset: AssetRef; venue: VenueId; observedAt: number;
  priceUsd: number | null;
  marketCapUsd: number | null;
  marketCapBasis: 'fdv' | 'circulating' | null;      // the venue says which; we never guess

  /** NULL on a bonding curve. Absence is not illiquidity. NOTHING GATES ON THIS. */
  liquidityUsd: number | null;
  /** What actually backs the price. The UI and the gates read THIS. */
  depth:
    | { kind: 'bonding-curve'; progress: number | null;
        slippageBpsAt: Record<'0.1'|'0.5'|'1.0', number | null> }
    | { kind: 'pool'; liquidityUsd: number; poolCount: number }
    | null;

  mintedAt: number | null;                            // copied from asset, never derived
  mintedAtConf: 'exact' | 'bounded' | 'unknown';
  transferRules: TransferRules | null;
  source: { vendor: string; endpoint: string; fetchedAt: number };
}
```

**What generalises almost for free: reads.** GeckoTerminal serves `solana` and `base` from one
`/networks/{id}` namespace, and DexScreener returns `chainId` per pair. A second chain's read
adapter is roughly 150 lines of mostly configuration.

**What does not generalise: writes.** Jupiter is Solana-only. 0x is EVM-only, one call per swap
plus one or two more for approval, with no protocol-enforced expiry — where Solana has a blockhash
that expires in about 68 seconds. A shared `swap()` would have to invent an approval concept for
Solana and an expiry concept for EVM, and would own retry for one chain but not the other.

**So: write two executors, share the quote.** The quote is what the user sees and what the product
promises. The executor is plumbing that genuinely differs.

```ts
export interface TradeQuote {
  asset: AssetRef; venue: VenueId; side: 'buy' | 'sell';
  inAmount: bigint; outExpected: bigint; outMinimum: bigint;   // bigint: 1e18 > 2^53
  inDecimals: number; outDecimals: number;
  slippageBps: number;

  /** Itemised. The confirm sheet renders THIS ARRAY and cannot render a hardcoded fee line.
   *  Measured all-in cost ranged 1.60%–22.72% across three same-age mints, so a fixed
   *  "0.50% fee" label is a lie. */
  costs: TradeCost[];
  allInBps: number;

  expiresAt: number | null;
  expiryReason: 'blockhash' | 'ttl' | 'none';
  route: { label: string }[];   // NEVER cached — the venue changes at graduation
}

export interface TradeCost {
  code: 'network'|'priority'|'rent'|'venue'|'price-impact'|'aggregator'|'insidor';
  label: string;
  amountUsd: number | null;
  bps: number | null;
  refundable: boolean;   // per-cost. Solana ATA rent is refundable. Nothing on EVM is.
}
```

### 6.5 The gates, ordered by cost

```js
async function runGates(candidate, story, policy, venue) {
  // free, local
  if (candidate.mintedAtConf === 'unknown')  return 'G1_mint_time_unknown';
  const lag = candidate.mintedAt - story.earliestPostAt;
  if (lag < policy.minLagMs)                 return 'G2_predates_post';   // → 'adopted' path
  if (lag > policy.maxLagMs)                 return 'G3_too_late';
  if (policy.majors.has(assetKey(candidate))) return 'G4_major';

  // one cheap read
  const s = await venue.read.state(candidate.asset);
  if (!s.transferRules?.complete)             return 'G5_transfer_rules_unread';
  if (s.transferRules.hasTransferFee ||
      s.transferRules.hasTransferHook)        return 'G6_nonstandard_transfer';

  // QUOTABILITY, not liquidity. Runs LAST — it is the only gate that costs a paid,
  // rate-limited call, and the only one that works identically on a curve and a pool.
  // THIS REPLACES `liquidity > 0`.
  const q = await venue.trade.quote({ asset: candidate.asset, side: 'buy',
                                      inAmount: policy.probeNotional });
  if (!q)                                     return 'G7_unquotable';
  if (q.allInBps > policy.maxAllInBps)        return 'G8_cost_absurd';
  return null;
}
```

`G7` must **fail closed** on a vendor error and must emit a distinct reason
(`G7_vendor_unavailable`), so "the token is untradeable" and "our quote vendor is down" are
distinguishable in one query. Failing open means every candidate passes during an outage. Not
distinguishing the two is exactly what made the ingest death invisible.

### 6.6 The decision, and why a margin is not a threshold

Candidate generation is **time-first, symbol-second** — the inverse of today's code, which searches
a vendor by symbol and then filters by time:

```sql
select * from asset
where chain = $1
  and minted_at between $2 and $3          -- story.earliestPostAt + [minLag, maxLag]
  and minted_at_conf <> 'unknown'
order by minted_at limit 500;
-- Symbol is then a SCORING channel over this set, never the retrieval key.
```

Then:

```
score = w · [temporal, symbol × collisionIdf, semantic, image, declared]

best, second = top two candidates
confident = best.score >= TAU_HIGH  AND  (best.score - second.score) >= DELTA_MARGIN
```

**The margin is the part that is usually missing.** A pump.fun search for "chill guy" returns 306
distinct tokens. A high score on the best candidate proves nothing when candidate #2 scores
equally. Ambiguity rejection is what makes "no confident match, no Buy button" actually
enforceable rather than aspirational.

**Cold start: a venue with fewer than ~200 of its own adjudicated labels can produce `unsure` at
most, never `confirmed`.** Scores are not comparable across venues — the collision IDF depends on
that venue's symbol density and the time prior on its lag distribution. A single global threshold
silently assumes otherwise. The consequence is a feature, not a limitation: **a new venue's first
month is read-only, enforced by types.** It is also how the labels get collected — every `unsure`
is logged with its full feature vector.

### 6.7 What to keep Solana-specific until a second chain is real

| Keep specific | Why | Trigger to revisit |
|---|---|---|
| trade execution | Jupiter and 0x do not reconcile (§6.4). Two executors, one quote type | **never** — the shared artefact is the disclosure, not the executor |
| holder analytics | `api/holders.js` paginates an SPL-specific RPC. EVM needs an indexer, a different vendor, a different cost model | a user notices the second chain's coin page has no holders |
| OHLCV / charts | `api/ohlcv.js:64` hardcodes `x-chain: solana`. One line becomes a parameter | the second chain's read adapter ships. Not before — the parameter has one legal value today |
| safety normalisation | the current shape (`api/safety.js:20-30`) is `{mintRevoked, freezeRevoked, lpBurned, top10}`. Two of those four are Solana nouns. On EVM they would be null forever, which renders as "unknown" and suppresses Buy permanently | replace with coded reasons plus a `requiredChecks` list **now**, because the shape is already wrong for Token-2022 on Solana |
| cross-chain "total market cap for a meme" | measured today: the symbol ZORA shows $1.0B liquidity on a Solana pair and $72k on a Base pair. A merged number would be meaningless and, on a Buy screen, dangerous | **never** |
| "create a coin" | the current flow fabricates an address in the browser with a random number generator and tells the user their coin is live. Delete it. The type exists only so the UI can check `capabilities.includes('create')` and get `false` | a paying user asks twice **and** the matcher's abstain rate has been stable for 30 days |

**Generalise now, because the trigger is already met:** `(chain, address)` identity with a CAIP-19
wire key; the chain address codec (~90 lines, and it is what stops a base58 string landing in a
column that later holds hex); the bonding-curve versus AMM venue split *inside* Solana; the
quotability gate; `MintEvent` with source and confidence; coded safety reasons; `bigint` in money
paths (`api/balance.js:36` does `value / 1e9`, which is exact at 1e9 and silently lossy at 1e18 —
cheaper to fix before a second chain than during).

---

## 7. The module tree and the dependency rule

You said you could not find the logic. **The tree alone has to answer that**, without a README
paragraph explaining it.

```
insidor/
├─ README.md          ≤ 40 lines: a table of question → directory. Nothing else.
│
├─ contracts/         ★ TYPES AND PORTS ONLY. Zero runtime dependencies.
│     item.ts  observation.ts  story.ts  asset.ts  decision.ts  policy.ts  ports.ts
│
├─ core/              ★ ALL THE LOGIC. Pure: no async, no fetch, no clock, no npm deps.
│  ├─ admit/          prior.js admit.js
│  ├─ kinetics/       ewma.js rate.js fidelity.js lifecycle.js
│  ├─ track/          schedule.js shed.js
│  ├─ detect/         burst.js baseline.js
│  ├─ group/          carriers.js match.js centroid.js promote.js merge.js
│  ├─ qualify/        rules.js
│  ├─ resolve/        gates.js score.js verdict.js provenance.js
│  ├─ rank/           heat.js stability.js
│  ├─ features/       ★ ONE feature-builder set, used by BOTH serving and training
│  └─ decide.js       the Decision constructor — every stage returns one
│
├─ adapters/          ★ ALL VENDOR CODE. No thresholds, no product decisions, no SQL.
│  ├─ platform/       x/ tiktok/ reddit/ replay/       + registry.js
│  ├─ venue/          solana/pumpfun/ solana/amm/      + registry.js
│  ├─ market/         dexscreener/ jupiter/ rugcheck/
│  ├─ judge/          anthropic/  (prompts live here, as .txt files)
│  ├─ embed/          gemini/ local/
│  ├─ meter/          ★ one cost wrapper every adapter call passes through
│  └─ test/contract/  ★ ONE suite run against EVERY platform and EVERY venue adapter
│
├─ store/             ★ THE ONLY PLACE SQL EXISTS.
│  ├─ migrations/     0001_init.sql … forward-only, numbered, one ledger table
│  └─ repo/           items.js observations.js stories.js assets.js decisions.js labels.js
│
├─ ml/                label/ train/ registry/ serve/    (features are NOT here — see below)
│
├─ services/          ★ THE PROCESSES. Thin. Wire adapters → core → store.
│  ├─ runner/         six supervised loops in one process
│  ├─ chainwatch/     the one long-lived connection, its own deployable
│  └─ watchdog/       imports contracts + store ONLY. Different infrastructure.
│
├─ app/               Next.js. Reads store. Cannot import core or adapters.
├─ eval/              blind/ gold/ replay/ reports/    Forbidden from importing adapters/.
└─ tools/             the CI checks below
```

**Why `features/` is in `core/` and not `ml/`.** If training computes a feature one way and serving
computes it another way, they drift, and the drift is invisible until the model quietly degrades.
Putting the builders in `core/` means the exact function that produced the logged feature vector is
the function the trainer replays. It also means feature code is pure and testable with no fixtures.

### The rule, in one sentence

**`contracts` is a leaf everyone imports; `core` imports only `contracts`; `adapters` imports only
`contracts`; `store` imports only `contracts`; `services` imports everything; `app` imports only
`contracts` and `store`; `watchdog` and `eval` import strictly less than `services`.**

```
contracts  →  (nothing)
core       →  contracts
adapters   →  contracts                          NEVER core
store      →  contracts                          NEVER core, NEVER adapters
ml         →  contracts, core                    NEVER adapters
services/* →  everything
app        →  contracts, store                   NEVER core / adapters / services
eval       →  contracts, core, store, ml         NEVER adapters
watchdog   →  contracts, store                   NEVER any other service
```

Three of those need their reason said out loud, because they will be argued with:

- **`app` must not import `core`.** The product rule is that our scoring is never shown to a user.
  If the app cannot import the scoring code, it cannot render a score. Six user-facing surfaces
  leak scoring today; this closes the class structurally.
- **`eval` must not import `adapters`.** A replay that can reach the network can read
  post-outcome state — which is exactly how the current validation harness derived its label from
  the model's own output. A lint rule is a cheaper guarantee than a protocol.
- **`watchdog` must not import any other service.** An observer sharing a module with the thing it
  observes shares its failure modes.

### Enforcement — four layers, cheapest first

**Layer 1 — `package.json` (free, and the strongest).** `adapters/package.json` does not list
`core` as a dependency. You cannot import what is not declared. One line of JSON is the primary
enforcement of your number-one requirement.

**Layer 2 — dependency-cruiser (`tools/check-boundaries.mjs`).**

```js
// .dependency-cruiser.cjs
forbidden: [
  { name: 'adapters-never-import-core', severity: 'error',
    from: { path: '^adapters/' }, to: { path: '^core/' } },
  { name: 'core-imports-only-contracts',
    from: { path: '^core/' },   to: { pathNot: '^(core|contracts)/' } },
  { name: 'core-has-no-runtime-deps',
    from: { path: '^core/' },   to: { dependencyTypes: ['npm'] } },
  { name: 'app-never-imports-logic',
    from: { path: '^app/' },    to: { path: '^(core|adapters|services|ml)/' } },
  { name: 'store-is-a-leaf',
    from: { path: '^store/' },  to: { path: '^(core|adapters|services|app|ml)/' } },
  { name: 'eval-never-hits-the-network',
    from: { path: '^eval/' },   to: { path: '^adapters/' } },
  { name: 'features-never-see-winners',
    from: { path: '^core/features/' }, to: { path: '(winner|outcome|label)' } },
  { name: 'no-circular', from: {}, to: { circular: true } },
]
```

`npm run graph` renders `docs/graph.svg`, and CI fails if it differs from the committed one. **A
changed SVG in a pull request is the most legible possible signal that somebody added a
coupling.**

**Layer 3 — the vocabulary gate (`tools/check-vocabulary.mjs`), about 30 lines.** This is the one
that actually enforces agnosticism, because a leak usually arrives as a *field name*, not as an
import.

```js
const BANNED_IN_CORE_AND_CONTRACTS = [
  // platform
  'tweet','retweet','quotecount','viewcount','views','playcount','digg',
  'subreddit','upvote','followers_count','tiktok','twitter',
  // chain and venue
  'solana','lamport','spl','mint_ca','base58','bonding_curve','pumpfun',
  'pump.fun','raydium','pumpswap',
  // vendors
  'jupiter','dexscreener','birdeye','helius','rugcheck','anthropic',
  'claude','haiku','apify','twitterapi','serpapi','supabase',
];
// scans core/** and contracts/** — identifiers, strings, comments, filenames.
// exits 1 with the offending file:line.
```

`authorKey` is fine. `followerBucket` is fine as an author prior. `viewCount` is not, because it
forces every future platform to pretend it has views.

**Layer 4 — runtime backstop.** Every server module in `app/` begins with `import 'server-only'`,
which turns a lint bypass into a build failure and closes the only catastrophic case: a service key
in the client bundle.

CI: `lint → typecheck → check-vocabulary → check-purity → check-boundaries → test`. Six commands,
all fast, all deterministic.

### Two honest notes about cost

**The linter is not currently running.** `eslint.config.js` already implements cross-feature import
zones for `site/features/{feed,token,wallet}` — good instinct, worth keeping — but
`eslint-plugin-import` is **not installed**, so `npx eslint` fails and those zones enforce nothing
today. Install it first; the existing pattern then extends to `worker/` in an afternoon.

**There is no TypeScript in this repository.** No `.ts` files, no `tsconfig.json`, and
`package.json` has two runtime dependencies and no TypeScript toolchain. The interfaces above are
written in TypeScript syntax because it is the clearest way to read them, but they ship either as
JSDoc typedefs (free, no build step, editors still autocomplete, no compile-time guarantee) or a
TypeScript migration is budgeted explicitly as its own step. **Pick one and say which** — the worst
outcome is a design document full of types that quietly implies weeks of unbudgeted migration.
Recommendation: JSDoc for `contracts/` and `core/` now, because the vocabulary grep and the
boundary checks give you most of the value without the migration, and revisit once the shape has
stopped moving.

### The escape valve

Any capability moves into a shared interface when **two** adapters implement it and a third would
be an obvious copy. Until then the second adapter duplicates the code. Promotion is a pull request
titled `promote: <capability>` that deletes both originals in the same commit. Demotion: any
capability with one implementer for 60 days moves back into that adapter.

By that rule, today: `watch`, `read` and identity promote (two Solana venues implement each
differently). `trade`, `assess` and `create` do not.

---

## 8. What migrates from the current build

Rule for all of it: the old tables get renamed into a `legacy` schema and made read-only. **Nothing
perishable is ever dropped**, including the experiment waves whose results turned out to be wrong —
keeping them on disk is what made the sign-flip instability visible in the first place.

### 8.1 The snapshot series — 57,032 rows, and it cannot be bought

`post_snapshots` → `internal.observations`. One SQL statement plus a fidelity map.

```sql
insert into internal.observations (item_id, captured_at, kind, value, fidelity, censored_reason)
select map_item(s.post_id), s.captured_at, k.kind, k.value, 'exact',
       case when k.kind = 'reach'
             and k.value = lag(k.value) over w
             and approval_rose(...) then 'stale_counter' end
from legacy.post_snapshots s
cross join lateral (values
  ('reach',        s.views),
  ('approval',     s.likes),
  ('conversation', s.replies),
  ('rebroadcast',  s.retweets),
  ('reproduction', s.quotes),      -- ← the load-bearing one
  ('retention',    s.bookmarks)
) as k(kind, value)
where k.value is not null
window w as (partition by s.post_id, k.kind order by s.captured_at);
```

Two judgement calls, stated rather than hidden: X counters are marked `exact` because we have not
measured otherwise; and the staleness censoring rule is applied *retroactively*, which the current
`deltaPerMinute` does not do — it emits `0` and reads as cooling.

Verification before and after: 2,410 of 2,411 posts have at least one observation, 1,821 have at
least two, median 9, max 135, and zero non-monotonic series across 994 checked.

**Carry the caveat forward with the rows.** History length correlates with early performance
(§4.2), so this corpus is fine for calibrating rate estimators and censoring rules, and is *not*
fine as an unbiased sample of trajectories. Write that in the migration file, not in someone's
head.

### 8.2 The near-miss corpus — 8,101 rows

`ingest_near_miss` → `internal.items_rejected`, plus one synthetic `internal.decisions` row each
with `decider = 'rule:legacy'`, `policy_hash = 'legacy'`, `reason = 'below_reach_floor'`.

**The important part is what we refuse to claim.** These rows record that a post was below a
floor. They do not record that it was *exposed and produced nothing*, which is what a negative
label requires. They are also censored by the same thresholds that produced the positives, so they
sample the boundary, not the population. So they migrate with `exposure = 'unknown'` and are used
at reduced weight for propensity calibration only — **not as class-0 training examples.** The full
raw payload comes across, because the fingerprints, media and author fields are recoverable from
it and are worth more than the rejection itself.

The park-and-recheck-by-id loop (`near-miss.js`) moves to `core/track/schedule.js` as a probation
tier, with promotion becoming rate-based rather than a raw threshold crossing — and with the
platform hardcoding at line 61 fixed by the port, so TikTok near-misses actually get rechecked.

### 8.3 The outcome labeller — the best artefact in the repository

`worker/backtest/dune-peak-multiples.sql` → `ml/label/peak_multiple.sql`, **verbatim SQL**, with
two changes around it.

The body does not change and should not: it takes graduation price from the first post-migration
trade with `amount_usd >= 10`, which sidesteps bonding-curve price artefacts, and counts trades
within 90% of max as a wash-trade guard. Classes separate cleanly — winners at 18× median peak and
$902k all-time high against losers at 1.19× and $42k, with 830 of 841 resolvable rows correctly
assigned. Lift it.

The two changes:

1. **A mandatory header block that becomes the row's `population` and `source` values:**
   ```sql
   -- population:   all mints matched to a story with verdict in (confirmed, unsure),
   --               2026-08-01 onward
   -- label_source: dex trades, decoded migrate + create
   -- window:       [minted_at, minted_at + 30d]
   -- version:      peak_multiple_v1
   ```
2. **The population changes from graduated-only to all matched mints.** Graduations run about 107
   a day against roughly 30,000 mints. Restricting to them is precisely why the backtest could not
   test what it claimed to test. **The SQL was right; the denominator was wrong.**

Note the two graduation-rate figures in these documents disagree — 0.36% in one place, 0.198% in
another. Resolve it once and cite one number, or the whole section gets challenged over a
footnote.

### 8.4 The cost meter — it moves, and it changes shape

`worker/adapters/{anthropic,x,apify}/budget.js` → `adapters/meter/`.

Today the budget check lives *inside* each vendor adapter, which means the adapter makes a product
decision. New shape:

```ts
export interface MeterPort {
  check(vendor: string, callType: string, estUnits: number): { allowed: boolean; reason?: string };
  record(vendor: string, callType: string, actualUnits: number, usd: number): Promise<void>;
}
```

**Adapters report. They never decide.** The shedding order — drop the lowest tracking tier before
the middle, never starve probation — lives in `core/track/shed.js`, where it is testable.

Carry forward verbatim: the per-call-type breakdown, measured averages replacing estimates, the
correct model rates, and per-tweet billing.

Fix four things in the move, all measured:

1. `narrative-title.js:163-169` calls `recordCall` only inside `if (parsed)`, so any title the
   parser rejected was **billed and never counted**. `record()` goes in a `finally`.
2. `budget.js:520-535` exports `scoreBudgetUsd()` and `maxScoreCallsRemaining()` and **nothing
   calls them.** So `TITLE_BUDGET_PCT` (25%) is enforced while `SCORE_BUDGET_PCT` (75%) is not, and
   scoring silently gets the whole cap.
3. `assertStartupBudget()` — the only thing that refuses to boot on a projected overrun — is
   reachable only from `worker/index.js:101`, the long-running worker, whose last heartbeat was
   2026-07-24 while crons ran through 08-06. **The projection guard is dead in production.**
4. The cost constants are stale in the expensive direction: `COST_SCORE_X = $0.003` against a
   measured ~$0.00127, and `COST_SCORE_TT = $0.012` against ~$0.0022. If the guard is using those,
   it pauses scoring at roughly a third of affordable throughput. **Correcting two numbers buys
   more headroom than any caching change.**

And a caution, since a caching optimisation was proposed in an earlier pass and is worth killing
cleanly: **prompt caching does not fire on this model.** Haiku 4.5's minimum cacheable prefix is
4,096 tokens; the system prompt is about 904. `cache_control` silently does nothing, with no error
and no warning. If you want caching, you have to deliberately pad the prefix past 4,096 — which
actually pays (a 4,096-token cached read costs less than the current 904-token uncached one) but
is a design decision, not a one-line fix. Fix the stale constants first.

### 8.5 Also carried, smaller but expensive to rediscover

| From | To | Why |
|---|---|---|
| `deriveGateReason` (`cluster-engine.js:291-339`) | `Decision.reason`, every stage | let a reviewer diagnose a seven-hour outage with one query |
| the blind/answer split and seeded shuffle | `eval/blind/` | upgraded so the answer is structurally unavailable rather than warned about, and the labeller sheet no longer links to the live coin page |
| the anti-self-clustering rule (`cluster-engine.js:98`) | `core/group/match.js` | keep the intent, but as a raised bar rather than a ban; the promotion quorum does the real work |
| `applyNameabilityCap` + `GENERIC_TICKERS` | `core/qualify/rules.js` | and finally imported by the grouping path |
| `tt-score-gate.js`'s shape | `core/track/` | a cheap deterministic pre-filter in front of an expensive call |
| `COINABILITY_CORE` and the subject-vs-story prompt text | `adapters/judge/anthropic/prompts/*.txt` | survives as labelling instructions independent of any model |
| the feed table reconciler (`feed.js:346-475`) and announce queue (`:873-991`) | `app/features/feed/` | the two hardest-won files in the front end |
| the per-feature README with a "what breaks here" section | mandatory on every module | |
| the self-authored defect register | the rebuild's punch list | |

### 8.6 Left behind

Vercel Cron as the execution model. Bag-of-words posing as embeddings. Symbol-string coin
matching. The fabricated contract address and the "your coin is live" message. The fabricated
sparklines and stock photos. The user-facing budget banner naming our vendor and our spend. Anon
SELECT on the scoring tables. 23 loose schema files with 11 appliers and no ledger. `lang:en` in
the ingest query.

**And every conclusion drawn from the backtest, in both directions.**

---

## 9. The build order

Each step ships alone and is useful alone. The perishable data starts accruing on day four.

**Step 0 — the skeleton and the gates. ~2 days.**
Nine workspaces, `contracts/policy.js` holding every number that is currently typed into a file,
all six CI checks green on an empty repo, `docs/graph.svg` committed, `eslint-plugin-import`
actually installed.
*Unblocks:* everything. Boundaries added after the code exists never get added.
*Done when:* a pull request that imports `core` from `adapters` fails CI with a named rule.

**Step 1 — the decision log and run records, retrofitted into the CURRENT build. ~3 days.**
`internal.decisions`, `internal.labels`, `internal.stage_runs`, and a `finally`-block writer wired
into the existing `api/cron/_lib/run-stage.js` and the existing gate functions. Nothing is
rewritten. Add the 2% holdout and 5% exploration reservations in the same commit.
*Unblocks:* replay, the first training frame — and it fixes the live silent-death bug today.
*Why first:* the pipeline is running right now and discarding every decision it makes.
*Done when:* `select stage, outcome, count(*) from internal.stage_runs group by 1,2`
distinguishes "ran and found nothing" from "died", and a killed process leaves
`finished_at IS NULL`.

**Step 2 — mint time as a stored column with a source. ~2 days.**
The `asset` table from §6.3, populated by polling the launchpad list endpoint and confirming
against one generic RPC call. Plus a coverage log so gaps are recorded rather than silently
counted as lead-time wins.
*Unblocks:* every outcome label, the temporal gate, and every lead-time number the product will
ever publish.
*Done when:* a deliberately induced gap produces a coverage row and the labeller marks that window
unmeasurable instead of negative.

**Step 3 — the collector: the X platform adapter plus `admit()`, shadowing the old ingest. ~5 days.**
Ship `adapters/platform/replay/` — a source backed by recorded JSON — **in the same step.** An
interface with one implementation is always secretly shaped like that implementation. Two from day
one is what makes the third cheap, and it gives `eval/` a network-free data source, which is why
CI can forbid `eval → adapters`.
*Done when:* the contract test passes against both `x` and `replay`, and `check-vocabulary` is
green.

**Step 4 — tracker, kinetics, observations. ~5 days.**
The schedule, hydration by id, the observations table, the censoring rules, lifecycle with
hysteresis, and the unconditional holdout tracked on the full grid.
*Unblocks:* every rate feature and the board's base. Perishable — every day not shipped is curve
data that will not exist.
*Done when:* a quantized counter below its step emits `Censored('below_step')`, not a zero rate.

**Step 5 — the resolver, the day-one patch set. ~4 days.**
Gates G1–G8 in cost order; quotability replacing `liquidity > 0`; time-first candidate generation;
the margin rule; provenance split so an observed cashtag and an invented ticker cannot occupy the
same row; the abstain band with no Buy affordance.
*Unblocks:* turning the Buy button back on without it being wrong. **This is the one defect class
where the failure costs a user money** — 189 of 323 resolved tickers share zero content words with
their story, and a working swap path sits one click away.
*Done when:* a frozen regression set — the real $KANG, $PUMP, $GRASS and GYM cases — runs in CI, so
a threshold change cannot land silently.

**Step 6 — the storyteller with real carriers, then real embeddings. ~7 days.**
Tier 1 first and alone: pHash, SimHash, format ids, the pgvector `bit` HNSW indexes. Measure the
singleton rate. Start the 14-day carrier-frequency table accruing. *Then* tier 2, behind the gold
set. Remove `lang:en` in the first hour of this step.
*Unblocks:* cross-post grouping, which is the product's actual premise.
*Done when:* the 300-pair gold set produces a precision/recall curve and the join threshold is
read off it rather than typed.

**Step 7 — board and app on the new store. ~7 days.**
Committed ranks with tick ids, Kendall tau on the dashboard, the app reading only the public
schema. Port the table reconciler and announce queue forward verbatim.
*Done when:* grepping the app's render paths for `score`, `reason` or `meme_min` returns nothing,
and the app cannot import `core` by CI rule.

**Step 8 — labeller and replay. ~5 days.**
`internal.labels` with `population NOT NULL`, the peak-multiple labeller repointed at all matched
mints, and `eval/replay` re-running `core` over the decision log under a candidate policy.
*Done when:* changing one number in `Policy` produces a verdict-flip report over last month with no
deploy and no network access.

**Step 9 — models. Ongoing.**
Order: QUALIFY first (its label is free and dense), then ADMIT, then DETECT at M60, then M15 and
M5. **RANK last**, because it needs board impressions that do not exist yet.
*Done when:* a challenger scores in shadow for seven days and is promoted by a query, not a deploy.

**Step 10 — the second platform. ~3 days.**
`adapters/platform/tiktok/` and one line in the registry.
**This step is the architecture's exam.** If it touches `core/`, the design failed — and it failed
cheaply, three days in, instead of at the point where you get told that adding a platform is a
rewrite.

### The order can bend in one place

Steps 1 and 5 are the two that fix live defects. If the Buy button matters more right now than the
board, Step 5 can move ahead of Steps 3–4. **Step 2 cannot move**, because every day it is not
running is a day of label data that will never exist.

---

## 10. What this deliberately does not do

Every item below is correct practice at a company with a platform team, and wrong for two people.
Naming them is part of the design, because the failure mode is not choosing badly — it is choosing
each one reasonably and waking up with eight systems to keep alive.

**A feature store.** Feast, Tecton, Chronon. They exist to solve the online/offline consistency
problem: the same feature computed by a streaming job and a batch job, which then drift, which
then has to be measured. Airbnb built a whole consistency-measurement pipeline for exactly this —
replaying online fetch logs through the offline backfill and diffing. **Logging features at
decision time does not manage that problem, it dissolves it**: there is only one computation of
each feature, and its output *is* the training data. Chronon orchestrates Kafka, Spark, Hive and
Airflow. That is more operational surface than this entire product. Cost avoided: roughly
$400–900/month and a person.

**A vector database.** Pinecone, Milvus, a Faiss service. At a 24-hour window of tens of thousands
of items, near-duplicate and semantic search are a pgvector index in the Postgres you already run.
Revisit at roughly 10× volume, and even then check whether Manku's permutation blocking (about 120
lines, no index) is enough first.

**A streaming layer.** Kafka, Flink, Ray. The pipeline is six loops on 20-to-60-second cadences.
Queues are arrays. The cadence is minutes and the label horizon is days; the only thing streaming
would buy is latency you cannot use.

**A warehouse.** Monthly Parquet exports of the decision log to object storage, queried with
DuckDB on a laptop. Same answers, no cluster.

**Online or continuous training.** ByteDance's Monolith and X's Phoenix both train against live
feedback, and both explicitly trade system reliability for it. That trade needs someone on call for
the parameter servers. Retrain a gradient-boosted tree in a notebook, weekly, by hand.

**A two-tower user model, engagement-sequence transformers, per-viewer diversity, multi-task
heads.** These are the core of every published recommender system, and **none of them applies,
because there is no user.** Everyone sees the same board. With a constant user vector, a two-tower
retriever collapses algebraically into a fixed linear function over item embeddings — a scorer,
not a retriever. You would be building a linear layer the expensive way. What genuinely transfers
is the *item* tower — a content encoder for item-to-item similarity — which is exactly the piece
whose absence causes the 91% singleton rate.

**Graph random-walk retrieval.** Pinterest's Pixie, PinSage. They walk a bipartite
engagement graph. We do not own one.

**Nineteen prediction heads.** Phoenix has them. We have on the order of a thousand sparse labels
that take six days to mature in a non-stationary market. **Capacity must not exceed evidence.** The
ceiling is a gradient-boosted tree with at most about twenty features, which is also what
Pinterest documents using for its lightweight ranker, for the same reason.

**A model-serving service.** `require('./model.json')` in the worker. A tree ensemble is a few
hundred kilobytes of nested objects and scoring one is tree-walking.

**An A/B testing platform.** There is one board and no per-user assignment. Use shadow scoring plus
backtesting against the outcome labeller. That is what `shadow_of` is for.

**A splitter for over-merged stories.** When a story's internal coherence falls below threshold,
flag it, suppress the Buy affordance, and show it on the ops dashboard. Splitting correctly means
deciding which posts, which title, which promotion timestamp and which coin match survive — a week
of work for a rate that should be under 3%. **If it exceeds 3% of open stories, that call was
wrong** and the week becomes contingency work.

**A generic wallet abstraction in shared code.** It would mean shared code imports the wallet SDK,
which means shared code has business rules. Keep it in its feature.

**A streaming mint subscriber.** §6.3. Poll. The winnable window is days.

### What that leaves

**One Postgres. One object store. One worker process with six loops. One long-lived connection.
One watchdog on different infrastructure. One Next.js app. Two runtime dependencies.**

That is what two people can actually keep alive, and it is not a compromised version of the real
thing. Netflix and Pinterest are currently *collapsing* their multi-stage funnels back into single
models, which is documented evidence that the elaborate version was a cost artefact rather than a
law. The staged funnel exists because expensive scorers times many items exceeds the budget. When
the expensive part gets cheap, you widen the gate rather than tune it.

The parts of the big-company playbook that transfer are the cheap ones and they are the ones that
matter: a staged funnel where each stage compresses and each stage's compression is measured; item
scoring that does not depend on what else is on the board; hard negatives mined from your own
rejections; and an append-only decision log written from the first day.

**The last one is the only thing on this list that cannot be added later.** If exactly one item
from this document survives contact with reality, make it that one.

---

## Appendix: the diagnostic nobody had

A stage whose compression ratio is about 1.0 is not a stage — it is a rename.

| stage | in/day | out/day | compression |
|---|---|---|---|
| arrive | — | ~30,000 | — |
| admit | ~30,000 | ~9,000 | 3.3× |
| detect | ~9,000 | ~900 | 10× |
| **group (today)** | **~1,200 admitted posts** | **1,289 stories** | **~1.0× — broken** |
| qualify | ~120 stories | ~40 | 3× |

Every published system compresses by 10× to 1,000× per stage. **Instrument compression per stage
and the 91% singleton bug is visible on day one instead of in a review six months later.** It is
one number per stage, written to the run record, and it costs nothing.
