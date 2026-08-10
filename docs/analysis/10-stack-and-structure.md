# Stack and structure

The question was: *"i dont even know if we are using the right coding languages. i need full
proper structure and understanding before we do pdf."*

This document answers it completely. Every choice is made, not surveyed. Where an earlier
research pass claimed something and the adversarial pass refuted it, the corrected version is
what appears here and the refutation is named, so you can see what moved and why. Where a
choice is genuinely close, it says so and gives the tiebreaker rather than pretending
certainty.

The prose is written for someone who does not write code. The artefacts — the trees, the
config files, the three worked source files — are written so a developer can start from them
without asking a follow-up question.

---

## 1. The decisions, in one table

| | The choice | Why, in one line | What would change it |
|---|---|---|---|
| **Language** | TypeScript, everywhere except one offline directory | The contracts and the pure core are only real if something checks them; nothing checks a comment | Nothing at this size. If the team grew past ~6 engineers, Go for the services would become arguable |
| **Runtime** | Node 24 LTS. Node 22 stays as the declared floor and the rollback | It runs `.ts` files directly with no build step, it is supported to April 2028, and every SDK we are forced to use is Node-shaped | A native dependency with no Node 24 prebuild — drop to 22, which is alive until April 2027 |
| **Package manager** | pnpm 11, workspace of nine packages | It is the only one that makes "adapters cannot import core" true at install time rather than at review time. Measured: npm workspaces does *not* enforce it | Nothing. This is the cheapest structural guarantee available |
| **Backend framework** | None. Long-lived Node processes, six supervised loops | The workload is 0.35 events per second and six timers. A framework here is ceremony | If work ever became user-triggered and bursty rather than scheduled |
| **Frontend framework** | React 19 on Vite 8, single-page app. **Not** Next.js | No SEO surface, no cacheable content, every number stale within a tick. Server components would render HTML that is wrong before it hydrates | If the read API ever has to become private and authenticated |
| **Database** | One Supabase Postgres, three schemas, `pg` client for servers | pgvector 0.8 already does the carrier search; a second store is a second backup story for two people | Nothing at 55M decision rows/year. Revisit at 50× |
| **ML — training** | Python, `ml/train/` only, run by hand, weekly | LightGBM in three seconds beats writing a boosting implementation | Nothing |
| **ML — serving** | TypeScript. Trees walked in ~60 lines; embeddings from a hosted API | A tree walk is arithmetic. A local encoder costs ~$7/month of RAM to avoid a $0.33/month bill | Volume rising ~100×, or embeddings needing to be reproducible forever |
| **Hosting** | Railway for the two always-on processes, Fly for the watchdog, Vercel for the app | The watchdog must not share a control plane with the thing it watches | Vercel previews going unused — then the app moves to Railway too |
| **Deployment shape** | Strangle, per-stage, one shared database | Collection cannot stop, and a single cutover of six coupled stages has no rollback | Nothing. Greenfield-and-switch is disqualified by the team size, not by taste |

Three of these are irreversible in the sense the architecture document means: the shape of what
a platform hands you, the shape of what a chain hands you, and whether you wrote down what the
system saw when it decided. Everything else in this table is a file. You will change several of
them and it will cost days, not months.

---

## 2. The language question, answered

### Is the current JavaScript wrong?

Yes — but not for the reason people usually give, and being precise about the reason matters,
because the wrong reason leads to the wrong fix.

The usual argument is "JavaScript lets you make mistakes." True, and not decisive. The build
has 164 files and 25,776 lines of working JavaScript that does real work every day. Nobody is
struggling to write it.

The actual problem is that **this architecture's central promises are unenforceable in plain
JavaScript.** Not harder to enforce — unenforceable. Three of them:

**A platform adapter must return our shape, not theirs.** Today `worker/ingest/lib/
parse-tiktok-post.js:82-86` writes TikTok's `playCount` into a column named `views`,
`shareCount` into `retweets`, and hardcodes `quotes = 0` — because TikTok has no equivalent
concept. Nothing objected. Downstream, "quotes" is the reproduction signal this entire product
is built on, and for every TikTok post it reads as an honest zero. That is not a typo. It is a
shape mismatch that no reviewer would catch, because there is no shape to mismatch against.

**A censored reading must emit no rate, not a zero.** The kinetics design says: if a counter is
quantized and the change is below the rounding step, you have learned nothing and must publish
nothing. Today `deltaPerMinute` publishes `0`, which downstream reads as *cooling*, which
demotes the item. Wrong in the most expensive direction — it hides acceleration. In TypeScript
this is a union type where reading `.perMin` off the censored branch is a compile error. In
JavaScript it is a sentence in a document.

**A pure decider must not be able to reach the network.** The whole replay guarantee — "re-run
any past decision under a new rule and see what changes" — depends on `decide()` being unable
to fetch, query, or call a model. A function typed to return `Decision` rather than
`Promise<Decision>` cannot be `async`, and a function that cannot be `async` cannot `await`.
That is the guarantee expressed as a type. There is no JavaScript equivalent; there is only a
convention, and conventions in this codebase have a track record: `eslint.config.js` already
declares import-boundary zones that enforce nothing, because `eslint-plugin-import` was never
installed.

So: the language is wrong because the design is built on contracts, and contracts that nothing
checks are documentation.

### What types actually buy, in plain terms

Think of a contract as a plug shape. Right now the system has plug shapes drawn on paper next
to each socket. Everyone agrees they should match. Sometimes someone plugs in the wrong thing,
it fits physically, and the appliance runs slightly wrong for six months.

Types make the plug shapes physical. The wrong adapter does not fit. You find out while you are
holding it, not in a report three weeks later.

Concretely, four things become impossible rather than discouraged:

1. **Forgetting to declare an absent counter.** `capabilities.absent: readonly CounterKind[]`
   is required. You cannot ship a TikTok adapter that silently claims to have a reproduction
   count.
2. **Reading a rate off a censored observation.** The compiler names the file and line.
3. **Awaiting inside a decider.** The signature forbids it.
4. **Sharing a billing helper between two vendors that bill differently.** Today
   `ingest-tiktok.js:39` imports `recordPostsIngested` from X's per-item budget module for a
   vendor that bills per Apify run. A `billing: 'per-item-returned' | 'per-call' | 'per-run' |
   'flat'` field on the adapter makes that import fail.

### What types cost a developer who writes JavaScript quickly — honestly

Four real costs. None of them is "you have to learn a new language."

**The build step is gone, but the type checker is not.** This is the correction that matters
most, and the earlier research got it wrong. Node 24 runs `.ts` files directly by stripping
types — verified locally on a four-package pnpm workspace: `node services/run.ts`, no compiler,
no bundler, no loader flag, correct output. But **Node strips types without checking them.**
The docs are explicit: *"No type checking is performed."* You still install TypeScript, still
keep a `tsconfig.json`, still run `tsc --noEmit` in CI. What disappeared is the *emit* step,
which was never the slow half. The honest saving is one devDependency and a `dist/` directory,
not "no build."

What you get for that is real and worth having: because the checker runs out of band, **nothing
the developer writes is ever blocked by it.** He runs the file. CI tells him about types later.
That is the whole reason this is safe for someone who works fast and loose.

**Four TypeScript features are banned, and breaking the ban fails at *runtime*.** Node can only
strip syntax that erases to whitespace. `enum`, `namespace` with runtime code, constructor
parameter properties (`constructor(private wallet: string)`), and decorators all throw
`ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX` when the module loads — not when you save the file. In a
system that touches money, a crash on a cold path at 3am is the wrong failure shape.

The fix is one tsconfig line and it is not optional: `"erasableSyntaxOnly": true` (TypeScript
5.8+) turns every one of those into an editor error instead. Verified: with it on, a file
containing `export enum Stage { Admit }` fails `tsc` immediately; without it, it fails at
runtime. Two more flags matter for the same reason — `verbatimModuleSyntax` and
`isolatedModules`, because `import { SomeType }` without the `type` keyword is also a runtime
error, not a compile error.

Practical consequence: no NestJS, no TypeORM, no class-validator, no tsyringe, no
type-graphql. None of them were on the list. Say it now rather than discover it.

**Node ignores `tsconfig.json` entirely, so path aliases do not work.** No `@/lib/thing`. Use
package.json `imports` subpaths, which must begin with `#`. This is a repo-structure decision
forced by the runtime, so it belongs in the first commit rather than the fortieth.

**Strict mode friction in weeks one and two.** Real, bounded, and mitigable in a specific
order: `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes` stay **off** until the
system is running — they generate hundreds of errors on day one. `any` is permitted everywhere
except `contracts/`. A `// @ts-expect-error` with a one-line reason is an acceptable commit.
Types are mandatory only at boundaries — exported signatures in `contracts/` and adapter
implementations. Function bodies infer.

### Why not the alternatives

**JSDoc-typed JavaScript** — the architecture document's own recommendation — was the right
answer in 2024 and is not now. It genuinely works; I checked rather than assumed, and
discriminated unions, `Partial<Record<K,V>>`, `@import` and constrained generics all type-check
correctly. But it costs 48% more lines for identical guarantees (37 vs 25 on the same module),
puts types in comments where rename-refactoring does not reach, and — the part that kills it —
**does not avoid the toolchain**. Same `typescript` dependency, same `tsconfig.json` (plus
`allowJs`/`checkJs`), same CI step. Its one advantage was avoiding a build step, and the build
step is gone for `.ts` too. That recommendation should be considered overturned.

**Go** is the strongest non-JS candidate and loses on three specifics, not on vibes. There is
no PDQ perceptual-hash implementation (`goimagehash` has aHash/dHash/pHash only), and PDQ-256
is what the grouping design specifies. The Solana client is community-maintained and lags the
first-party TypeScript one. Privy — the wallet vendor, already a dependency — ships JavaScript
and nothing else. The frontend is TypeScript regardless, so choosing Go means two languages for
two people, one of which is still TypeScript.

**Python as the system language** is the worst candidate at the one property the architecture
calls irreversible. Nothing structurally prevents a "pure" function reading the clock or
opening a socket, `mypy` is opt-in and gets skipped under deadline, and the ML advantage that
motivates it turns out to apply to one offline directory rather than to a system. Take the
directory. Refuse the system.

**Rust** would produce the most correct version of this and is disqualified by the team, not by
merit. Two people, one non-engineer, no fast-and-loose gear.

**Bun and Deno** both work. Bun is faster at things this system never does — 0.35 events per
second — and its two remaining soft spots are exactly this system's two unusual requirements: a
process that stays alive for weeks, and a native addon (`sharp`, for keyframe decode before
hashing). Deno's permission model is the single most attractive feature any alternative
offers, and Node absorbed it: verified that `node --permission --allow-fs-read=$PWD` blocks a
stray file read inside `core` while still loading the module graph.

**Performance played no part in any of this, and that is the point.** 1,200 admitted posts a
day and one websocket is a workload every candidate handles with the CPU asleep. Once
performance is off the table, the criteria are ecosystem fit and cost to two people, and there
the answer is not close.

---

## 3. Where the ML runs

### The one-sentence answer

**Python is in the repository and never in production.** It lives in `ml/train/`, is run by
hand on a laptop, weekly, and produces exactly two artefacts: a JSON model file and an HTML
report. Nothing under `services/` imports it, shells out to it, or waits on it. If Python were
deleted from the machine at 3am, the system would keep making decisions with the model file it
already has.

### Why a two-language split is safe here, when it usually is not

The classic reason ML must be one language is train/serve skew: the trainer recomputes a
feature slightly differently from the server, and the model quietly degrades. Here the trainer
**cannot** recompute a feature. It reads `internal.decisions.features` — a frozen JSON blob
that `core/features/` computed at the moment of the decision and wrote down. Python never sees
a raw post, never touches a counter, never calls a platform. It sees a table of numbers and a
column of labels.

The architecture document already made this argument for a different purpose: *"logging
features at decision time does not manage the online/offline consistency problem, it dissolves
it."* That same sentence is what makes the language boundary safe. The boundary is data, not
code.

### The split, precisely

```
  NODE (production, always on)
    core/features/          computes the feature vector, once
    store/repo/decisions    freezes it into internal.decisions
    ml/label/run_labels.mjs nightly, JavaScript, writes internal.labels
                     │
                     │  ① a SQL result set (parquet)  ── crosses the boundary
                     ▼
  PYTHON (ml/train/, laptop, weekly, by hand)
    pull.py → train_gbdt.py → report.py → publish.py
                     │
                     │  ② model.json + one registry row  ── crosses back
                     ▼
  NODE
    ml/registry → Policy.scorers.admit → core/admit/decide()
```

Two artefacts cross. Neither is code. Neither is on the critical path of a user request.

### Serving trees in TypeScript: verified, with the caveats that were missing

A LightGBM gradient-boosted tree is a few hundred kilobytes of nested JSON, and walking it is
arithmetic. Measured: a 36-line dependency-free JavaScript walker reproduced LightGBM's own
Python predictions to a maximum absolute error of **2.2 × 10⁻¹⁶** over 1,000 rows — that is
float64 rounding, not approximation — at 7.4 µs per row.

**But that measurement is a happy-path measurement, and the earlier research reported it
without its scope.** The adversarial pass ran the same naive walker against four variants and
it failed silently — no exception, no warning, just wrong probabilities:

| Model variant | Max absolute error |
|---|---|
| numeric features, no missing values (the measured case) | 2.2 × 10⁻¹⁶ |
| numeric features **with missing values** | 0.92 |
| **categorical** features | 0.98 |
| `sigmoid = 2.0` instead of the default | 0.15 |
| `linear_tree = True` | 0.34 |

The categorical case is the nastiest: LightGBM's JSON dump emits the threshold as the string
`"0||1||2||7"`, so `x <= threshold` coerces to `NaN`, evaluates false, and every row takes the
right branch forever. A model that is wrong by 0.9 in probability space looks like a bad week,
not like a bug.

All of them repair to 2.2 × 10⁻¹⁶ once the walker implements LightGBM's actual decision
semantics. So the fix is cheap. It is just not what "60-line tree walker" conjures, and the
happy-path number is what would have shipped.

Three non-negotiables follow:

1. **Constrain the trainer.** `categorical_feature=[]`, `linear_tree=False`, `sigmoid=1.0`,
   set explicitly in `train_gbdt.py`.
2. **Fail loudly at load.** The walker throws on any `decision_type` or `missing_type` it does
   not implement. Never fall through.
3. **A parity fixture in CI.** `publish.py` writes the model, 1,000 feature rows, and
   LightGBM's own predictions for them. CI asserts max error < 1e-9 on every push. **This test
   is the entire justification for allowing a second language.** Without it, do not split.

### The correction that matters more: trees are the easy quarter

The earlier research called the tree-walker measurement "the single fact the entire
recommendation rests on." It is not. The judge model as specified is a frozen text embedding
plus a frozen image embedding plus ~38 hand-engineered features, fed to a tree ensemble with
isotonic calibration. **The trees are one of four stages, and the only one that was never the
accuracy risk.**

The three that are:

- **The text encoder.** Roughly 40 ms per post, not 16 µs. And the divergence risk is the
  *tokenizer*, not the arithmetic — a different subword split produces a different vector,
  which is a different answer, not a rounding difference.
- **Image preprocessing.** PIL and `sharp` do not resize and normalise identically, and the
  difference is far larger than float noise.
- **Isotonic calibration.** Portable, but it is a scikit-learn artefact with its own
  interpolation and out-of-bounds clipping semantics, and LightGBM's dump does not contain it.
  It must be exported and matched deliberately. (Good news: a ~25-line JavaScript
  pool-adjacent-violators implementation matched `sklearn.isotonic` **exactly** — 0.000e+0
  difference over 500 points — so the nightly recalibration loop needs no Python at all.)

The memory figures were similarly scoped wrong. `onnxruntime-node` is 258 MB of native binaries
on disk. The measured resident footprint of two quantised models plus OCR is ~340 MB, which is
a real line on the hosting bill and is already budgeted. The "69 MB RSS" figure describes a
bare tree evaluator and nothing else.

**The decision this changes:** embeddings come from a hosted API, not from a local ONNX model
in the worker. At 1,200 items/day × ~60 tokens, `gemini-embedding-001` costs **$0.33/month**
($2.43 at the design's upper volume bound; under a dollar to re-embed the entire 90-day corpus
after a model change). The local alternative measured 780 MB resident, which moves the worker
from a ~$3/month machine to an ~$11/month machine — about **$7/month extra to avoid a
$0.33/month bill**, plus 261 MB of `node_modules` and a 4.9-second cold start on every deploy.
The intuition that local is free is exactly backwards at this volume.

And it does not create a fragility, because the grouping design already makes embeddings
Tier 2. Tier 1 — perceptual image hash, text simhash, format ids, reproduction pointers — is
local, free, deterministic and language-blind, and does most of the work. An embedding outage
degrades grouping to Tier 1 rather than stopping it. That must stay true: if anyone ever makes
an embedding a precondition for grouping, this mitigation evaporates.

`onnxruntime-node` stays in `optionalDependencies` with a ~40-line local adapter behind the
same port, and production deploys with `--omit=optional`. The fallback is a config flag, not a
rewrite.

### One volume figure to stop repeating

"1,200 predictions a day" is wrong. 1,200 is *admitted posts entering GROUP* — and it is the
one row in the funnel table the architecture document marks as broken. Predictions are not one
per post: MATCH scores **pairs** (every admitted post against every open story candidate), and
RESOLVE scores up to **500 candidates** per decision because it needs the top two to compute an
ambiguity margin. Real prediction cardinality is one to two orders of magnitude higher.

It still does not matter — 61,000 rows/second against a workload of thousands — but the number
should be right in a document people will plan against, and it changes how you think about
GROUP, whose cost grows with pairs inside a block rather than with arrivals.

### What Python's presence costs

One `pyproject.toml`, one `uv.lock`, one command. `uv` installs its own Python, so there is no
system-Python problem. No container, no scheduler, no CI secret, no GPU, no notebook server, no
deploy target.

The risk is not technical, it is gravitational: `ml/train/` will try to grow. The moment
anything under `services/` shells out to Python, the single-runtime property is gone and nobody
decided to lose it. So it is a CI check like the others — eight lines that fail the build if a
`.py` file appears outside `ml/train/`, or if `services/`, `adapters/`, `core/` or `store/`
mention `python`, `uv run`, or `child_process`.

---

## 4. The full tree

Nine packages. The rule is that **the directory name answers the question**, so the README is a
lookup table rather than an explanation.

```
insidor/
├─ package.json                 root: scripts + devDeps only. "private": true.
├─ pnpm-workspace.yaml          ★ the boundary is declared here
├─ tsconfig.base.json           one compiler config; every package extends it
├─ .dependency-cruiser.cjs      ★ the boundary is enforced here
├─ .nvmrc                       24
├─ .github/workflows/
│   ├─ ci.yml                   typecheck → vocab → purity → boundaries → test
│   ├─ watchdog.yml             every 10 min, different vendor from production
│   └─ train.yml                weekly, Sunday 06:00 UTC — the only scheduled job
├─ docs/
│   ├─ graph.svg                committed; CI fails if regenerating it differs
│   └─ decisions/               one ADR per irreversible call
├─ tools/                       not a package. Plain .mjs, run by node.
│   ├─ check-vocabulary.mjs     no platform/chain/vendor word in core|contracts
│   ├─ check-purity.mjs         no clock, no RNG, no fetch, no await in core
│   ├─ check-policy.mjs         no bare numeric literal in core outside policy.ts
│   ├─ check-python.mjs         no .py outside ml/train/; no shell-out to it
│   ├─ check-app-vocabulary.mjs no internal word or vendor name in app/src
│   └─ graph.mjs                renders docs/graph.svg; --verify in CI
│
├─ legacy/                      ★ the current build, `git mv`'d whole, unedited
│   ├─ api/  worker/  site/  lib/  scripts/
│   └─ FROZEN.md                bug fixes and the decision-log writer only
│
├─ contracts/                   ★ TYPES AND PORTS. Zero dependencies. Zero logic.
│   ├─ package.json             "dependencies": {}  ← literally empty, forever
│   ├─ tsconfig.json
│   └─ src/
│       ├─ index.ts             the one legitimate barrel: the public vocabulary
│       ├─ vocabulary.ts        ★ Item, Observation, Decision   [WRITTEN OUT §4.1]
│       ├─ ids.ts               ItemId StoryId AssetRef — branded, opaque
│       ├─ reasons.ts           the closed reason-code list, ~60 entries
│       ├─ policy.ts            ★ EVERY threshold in the system. One frozen object.
│       ├─ story.ts             Story, StoryMember, MatchEvidence
│       ├─ asset.ts             Asset, MintTime, MarketState, TradeQuote
│       ├─ judgement.ts         what a judge returns — vendor-neutral
│       ├─ features.ts          FeatureVector, FeatureSetId, Scorer
│       └─ ports/
│           ├─ platform.ts      PlatformAdapter, Capabilities, Budget
│           ├─ venue.ts         Venue, VenueWatch | Read | Assess | Trade
│           ├─ judge.ts  embed.ts  meter.ts
│           └─ store.ts         the repository interfaces store/ implements
│
├─ core/                        ★ ALL THE LOGIC. Pure. No dependencies, not even dev.
│   ├─ package.json             deps: { "@insidor/contracts": "workspace:*" }
│   ├─ tsconfig.json            rootDir ./src — a ../ import fails typecheck
│   └─ src/
│       ├─ index.ts             narrow barrel: seven stages + decide. Eight names.
│       ├─ decide.ts            the Decision constructor. Every stage returns one.
│       ├─ hash.ts              FNV-1a, 15 lines. Keeps core dependency-free.
│       ├─ math.ts              clamp01, logistic, percentile, negBinomial
│       ├─ admit/               stage.ts  stage.test.ts  prior.ts  bait.ts
│       ├─ track/               stage.ts  schedule.ts  shed.ts  holdout.ts
│       ├─ detect/              stage.ts  burst.ts  baseline.ts  poisson.ts
│       ├─ kinetics/
│       │   ├─ rate.ts          ★ emitRate: the censoring rule, ~15 lines
│       │   ├─ rate.test.ts     the below-step case is the first test in the repo
│       │   ├─ ewma.ts          continuous-time decay, w = exp(−Δt/τ)
│       │   ├─ fidelity.ts      quantizationStep()
│       │   └─ lifecycle.ts     hysteresis on state transitions
│       ├─ group/               stage.ts  carriers.ts  simhash.ts  phash.ts
│       │                       persistence.ts  match.ts  promote.ts  merge.ts
│       ├─ qualify/
│       │   ├─ stage.ts         ★ [WRITTEN OUT §4.2]
│       │   ├─ stage.test.ts
│       │   ├─ rules.ts         deterministic caps the judge cannot override
│       │   ├─ nameability.ts
│       │   └─ generic-tickers.ts   the 45 entries — imported by grouping too
│       ├─ resolve/             stage.ts  gates.ts (G1..G8)  score.ts  margin.ts
│       │   └─ __fixtures__/    the frozen $KANG $PUMP $GRASS GYM regression set
│       ├─ rank/               stage.ts  heat.ts  hysteresis.ts  stability.ts
│       └─ features/            ★ ONE builder set. Serving AND training call THIS.
│           └─ registry.ts  item.ts  story.ts  candidate.ts
│
├─ adapters/                    ★ ALL VENDOR CODE. No thresholds. No SQL. No core.
│   ├─ package.json             does NOT list @insidor/core — pnpm enforces it
│   └─ src/
│       ├─ registry.ts          the ONLY dispatch. One line per platform.
│       ├─ tape.ts              record once / replay forever. Throws on miss.
│       ├─ meter/               meter.ts  units.ts  (per-item|per-call|per-run|flat)
│       ├─ codec/               ★ runtime validation of vendor JSON. Types erase.
│       ├─ platform/
│       │   ├─ x/               index capabilities to-item discover observe client
│       │   ├─ tiktok/
│       │   │   ├─ to-item.ts   ★ [WRITTEN OUT §4.3]
│       │   │   └─ to-item.test.ts  + the other five files
│       │   ├─ reddit/          (the same six files)
│       │   └─ replay/          ★ a platform backed by recorded JSON. Ships day one.
│       ├─ venue/solana/        chain.ts  pumpfun/  amm/   + registry.ts
│       ├─ market/              dexscreener/  jupiter/  rugcheck/
│       ├─ judge/anthropic/     index.ts + prompts/*.txt — never an inline string
│       ├─ embed/               gemini/ (shipped)  local/ (written, not installed)
│       ├─ hash/pdq/            vendored wasm, 38.6 KB, no npm dependency
│       └─ testkit/             platform.contract.ts  venue.contract.ts
│
├─ store/                       ★ THE ONLY PLACE SQL EXISTS.
│   └─ src/
│       ├─ client.ts            pg over Supavisor session mode. Not PostgREST.
│       ├─ migrate.ts           forward-only, numbered, ONE ledger table
│       ├─ migrations/          0001_schemas.sql … 0014_model_registry.sql
│       ├─ projections/         ★ the ONLY module that sees both vocabularies
│       │   ├─ board.ts         RankedStory → BoardRow, via a runtime pick()
│       │   └─ board.test.ts    asserts the emitted key set, and the values
│       └─ repo/                items observations stories assets decisions
│                               labels carriers policies stage_runs
│
├─ view/                        ★ THE PUBLIC WIRE VOCABULARY. Zero dependencies.
│   ├─ package.json             does NOT import contracts. The duplication is the wall.
│   └─ src/                     board.ts  story.ts  asset.ts  trade.ts  fields.ts
│
├─ ml/
│   ├─ label/run_labels.mjs     JS. Nightly. Unattended. No Python.
│   ├─ registry/                registry.ts  promote.ts — sha + feature-hash gates
│   ├─ serve/
│   │   ├─ lgbm.ts              ~60 lines. Zero dependencies. Throws on unsupported.
│   │   ├─ isotonic.ts          ~25 lines PAVA. Verified == sklearn exactly.
│   │   └─ fixtures/            model + 1,000 rows + Python's answers ← the CI oracle
│   └─ train/                   ★ THE ONLY PYTHON IN THE REPOSITORY
│       ├─ pyproject.toml  uv.lock
│       ├─ sql/train_admit_v1.sql   committed; its sha is stored in the registry row
│       └─ pull.py  train_gbdt.py  report.py  publish.py
│
├─ services/                    ★ THE PROCESSES. Thin. Wire adapters → core → store.
│   └─ src/
│       ├─ runner/main.ts       six supervised loops in one process   → Railway
│       │   └─ loops/           admit track detect group qualify resolve rank
│       ├─ chainwatch/main.ts   owns a durable mint cursor            → Railway
│       └─ watchdog/main.ts     imports contracts + store ONLY        → Fly.io
│
├─ app/                         ★ Vite SPA. Imports @insidor/view and NOTHING else.
│   └─ src/
│       ├─ main.tsx  routes/    thin: layout and Suspense only
│       ├─ data/                ★ THE ONLY PLACE THAT TOUCHES THE NETWORK
│       ├─ live/                ★ boardStore.ts, freeze.ts — the ordering machine
│       ├─ features/            board/ story/ asset/ trade/ wallet/
│       └─ ui/                  tokens.css + primitives/ — a leaf, imports nothing
│
├─ eval/                        Forbidden from importing adapters.
│   └─ src/                     replay/  blind/  gold/  parity/  reports/
│
└─ tapes/                       recorded vendor responses, committed, ~40 MB
```

### Naming rules that do actual work

**Every stage directory contains a file literally named `stage.ts`.** This is the
highest-value naming rule in the repository, because it means "where is the admit logic" has a
mechanical answer requiring no search, no README, and no memory. Everything else in the
directory is a helper `stage.ts` calls.

**There is never a `types.ts`.** It is a junk drawer with a respectable name, and within a year
nobody can delete anything from it. Shared types live in `contracts/`. Types used by one
function live in the file defining that function.

**Two barrels are allowed, both narrow.** `contracts/src/index.ts`, because the vocabulary
genuinely is a public surface, and each package's `src/index.ts` because the `exports` map
needs one. `core/src/index.ts` exports exactly eight names. A barrel listing eight names is a
table of contents; a barrel re-exporting two hundred is fog, and it is how import cycles get
created without anyone choosing one.

**Every number lives in `contracts/src/policy.ts`.** Today there are 60 uppercase numeric
constants across 24 files. `tools/check-policy.mjs` fails CI on a numeric literal in `core/`
outside a small allowlist. This check needs occasional allowlist edits for genuine mathematics
— that is a real, small, recurring cost, and it is worth paying, because "every threshold is a
number somebody typed somewhere" is the finding that made the last six months unauditable.

---

### 4.1 A contract — `contracts/src/vocabulary.ts`

```ts
/**
 * THE VOCABULARY. Three nouns. Everything else is spelled in terms of them.
 *
 *   Item        — something a person posted, in our words, never a vendor's.
 *   Observation — one reading of an item's counters at one instant, carrying an
 *                 honest statement of how much that reading can be trusted.
 *   Decision    — a stage looked at some features under some policy and chose.
 *                 This is the only thing here that cannot be reconstructed later.
 *
 * RULES, enforced by tools/check-vocabulary.mjs:
 *   - Type-only imports from sibling contracts files. Nothing else, ever.
 *   - No word naming a platform, a chain, or a vendor — in code OR in comments.
 *     Not "views". Not "retweet". A field name is how a leak actually arrives.
 *   - Adding a field here is a real decision. Adding one to a FeatureVector is not.
 */

import type { ItemId, SourceId, AuthorKey } from './ids.ts';

/** Epoch milliseconds. Never a Date: a Date is mutable and is not JSON. */
export type Millis = number;

/* ── what a platform can count ────────────────────────────────────────── */

export type CounterKind =
  | 'reach'          // impressions-like. NOT comparable across sources.
  | 'approval'       // the one-tap positive
  | 'conversation'   // a written response
  | 'rebroadcast'    // a copy that creates NO new authored object
  | 'reproduction'   // a copy that DOES create one  ← the thesis, as a type
  | 'retention';     // saved for later

/**
 * How much you may trust the number. Not decoration: core branches on this,
 * and it is the reason a flat counter never emits a zero rate.
 */
export type Fidelity =
  | { readonly kind: 'exact' }
  | { readonly kind: 'quantized'; readonly significantDigits: number }
  | { readonly kind: 'fuzzed' }    // deliberately perturbed by the source
  | { readonly kind: 'absent' };   // the source has no such concept. NOT zero.

export interface Counter {
  /** null means "not read". Absence of the CONCEPT is Fidelity.absent. */
  readonly value: number | null;
  readonly fidelity: Fidelity;
  /** When WE read it. Never when the source claims it changed. */
  readonly observedAt: Millis;
  /** Set only when the source admits its own staleness. */
  readonly lagMs?: number;
}

export type CounterSet = Readonly<Partial<Record<CounterKind, Counter>>>;

/* ── carriers: what makes two items the same thing ────────────────────── */

export type FingerprintKind = 'imageHash' | 'textShingle' | 'formatId' | 'entitySpan';

export interface Fingerprint {
  readonly kind: FingerprintKind;
  /** Opaque. Comparable only against the same kind. */
  readonly key: string;
  /** Present for hashes supporting distance; absent for exact-match kinds. */
  readonly bits?: number;
}

export interface MediaRef {
  readonly kind: 'image' | 'video' | 'audio';
  readonly uri: string;
  readonly width: number | null;
  readonly height: number | null;
  readonly durationMs: number | null;
}

/* ── ITEM ─────────────────────────────────────────────────────────────── */

export interface Item {
  readonly itemId: ItemId;         // ours
  readonly source: SourceId;
  readonly sourceItemId: string;   // theirs
  /** A stable id, never a display handle. Handles change; ours must not. */
  readonly authorKey: AuthorKey;

  /** null when the source omits it or is known to lie. Never defaulted. */
  readonly postedAt: Millis | null;
  readonly firstSeenAt: Millis;

  readonly lang: string | null;
  readonly text: string;
  readonly media: readonly MediaRef[];
  readonly counters: CounterSet;
  readonly fingerprints: readonly Fingerprint[];

  /** Added ZERO new authorship. */
  readonly rebroadcastOf: ItemId | null;
  /** Added ONE new authorship. The signal we sell. */
  readonly reproductionOf: ItemId | null;

  /** Reusable templates: a sound, an effect, a format. Free carrier joins. */
  readonly formatIds: readonly string[];

  /** Key into blob storage. The raw vendor payload is NEVER inlined here. */
  readonly rawRef: string;
}

/* ── OBSERVATION ──────────────────────────────────────────────────────── */

/** Why a reading produced no usable rate. Closed list; core branches on it. */
export type CensorReason =
  | 'unusable_fidelity'  // absent or fuzzed
  | 'below_step'         // quantized, and the change is under the rounding step
  | 'stale_counter'      // unchanged while a sibling counter rose
  | 'non_monotonic'      // went backwards
  | 'no_prior';          // first reading; nothing to difference against

export interface Observation {
  readonly itemId: ItemId;
  readonly capturedAt: Millis;
  readonly kind: CounterKind;
  readonly counter: Counter;
  /**
   * The differenced quantity, per minute, or null when censored.
   * A censored observation carries the previous level forward and emits NO
   * rate point. It must never be written as 0 — downstream, 0 reads as
   * "cooling", which demotes exactly the items that are accelerating.
   */
  readonly ratePerMin: number | null;
  readonly censored: CensorReason | null;
}

/* ── DECISION ─────────────────────────────────────────────────────────── */

export type StageName =
  | 'admit' | 'track' | 'detect' | 'group' | 'qualify' | 'resolve' | 'rank';

export type Verdict = 'pass' | 'hold' | 'drop' | 'abstain';

export interface Decision {
  readonly stage: StageName;
  readonly subjectKind: 'item' | 'story' | 'pair' | 'candidate';
  readonly subjectId: string;

  /**
   * THREE CLOCKS. Conflating any two is how lookahead comes back.
   *   featureAsOf   — the newest input datum the decider was allowed to see
   *   decidedAt     — when we chose
   *   subjectOrigin — when the thing itself began
   * The store enforces featureAsOf <= decidedAt as a CHECK constraint.
   */
  readonly featureAsOf: Millis;
  readonly decidedAt: Millis;
  readonly subjectOrigin: Millis | null;
  readonly horizonS: number | null;

  readonly verdict: Verdict;
  /** From a closed list. Never free text. Never null, including on `pass`. */
  readonly reason: import('./reasons.ts').ReasonCode;
  /** null when a rule decided; a number when a model did. */
  readonly score: number | null;

  /** EXACTLY what the decider saw. Frozen here, before the outcome exists. */
  readonly features: import('./features.ts').FeatureVector;
  readonly featureSet: import('./features.ts').FeatureSetId;

  /** The thresholds it was judged against. Without this, nothing is auditable. */
  readonly policyHash: string;
  /** 'rule:qualify@3' today, 'gbdt:qualify@2026-11-02' later. Log doesn't care. */
  readonly decider: string;

  /** (0,1]. 1.0 for a deterministic rule. Off-policy evaluation needs it. */
  readonly propensity: number;
  readonly explore: boolean;
  readonly exploreArm: 'epsilon' | 'holdout' | null;

  readonly costUsd: number;
}

/* ── the shape every stage has ────────────────────────────────────────── */

/**
 * Everything a stage may know about the outside world. The clock is a VALUE,
 * not a call. That is the whole trick: a stage cannot ask what time it is, so
 * a replay six months later gets the same answer.
 */
export interface StageContext {
  readonly now: Millis;
  readonly policyHash: string;
  /** Deterministic per subject; drives holdout and epsilon assignment. */
  readonly seed: string;
}

/**
 * decide() is SYNCHRONOUS on purpose. A function that cannot await cannot
 * fetch, cannot query, cannot call a hosted model, and cannot quietly
 * recompute a feature from fresher data than the one it logged.
 */
export interface Stage<Input> {
  readonly name: StageName;
  readonly featureSet: import('./features.ts').FeatureSetId;
  extract(input: Input, ctx: StageContext): import('./features.ts').FeatureVector;
  gate(f: import('./features.ts').FeatureVector,
       p: import('./policy.ts').Policy): import('./reasons.ts').ReasonCode | null;
  decide(input: Input,
         p: import('./policy.ts').Policy,
         ctx: StageContext): Decision;
}
```

---

### 4.2 A pure core stage — `core/src/qualify/stage.ts`

```ts
/**
 * QUALIFY — is there a nameable, coinable thing in this story?
 *
 * HOW THIS FILE STAYS FREE OF I/O, given that it consults a language model:
 * it does not consult one. The service calls the judge adapter, gets a
 * Judgement, and hands it in as `input.judgement` — data, like any other
 * field. This function only reads. If the judge was not called, `judgement`
 * is null and we ABSTAIN with a named reason rather than deciding blind.
 *
 * Consequences worth stating, because they are the point:
 *   - Replaying this stage over the decision log needs no network and no key.
 *   - Swapping the judge changes nothing here.
 *   - The deterministic caps below CANNOT be overridden by the model, because
 *     they run after it and their inputs are ours.
 *
 * Every number is in Policy. There are no numeric literals in this file.
 */

import type {
  Decision, StageContext, Story, StoryMember, Judgement,
} from '@insidor/contracts';
import type { FeatureVector } from '@insidor/contracts/features.ts';
import type { Policy } from '@insidor/contracts/policy.ts';
import type { ReasonCode } from '@insidor/contracts/reasons.ts';

import { decide as makeDecision } from '../decide.ts';
import { clamp01 } from '../math.ts';
import { isGenericTicker } from './generic-tickers.ts';
import { nameabilityCap, subjectSpecificity } from './nameability.ts';
import { distinctAuthors, distinctSources } from './rules.ts';

export const NAME = 'qualify' as const;
export const FEATURE_SET = 'story.qualify.v1';
export const DECIDER = 'rule:qualify@1';

/** Everything this stage may see. Assembled by services from the store. */
export interface QualifyInput {
  readonly story: Story;
  readonly members: readonly StoryMember[];
  /** null when the judge was not called or failed. NOT an empty judgement. */
  readonly judgement: Judgement | null;
  /** What the judge call cost us. Metered by the adapter, passed as data. */
  readonly judgeCostUsd: number;
}

/* ── features ─────────────────────────────────────────────────────────── */

export function extract(input: QualifyInput, ctx: StageContext): FeatureVector {
  const { story, members, judgement } = input;

  const authors = distinctAuthors(members);
  const sources = distinctSources(members);
  const proposed = judgement?.proposedName ?? null;

  return {
    memberCount: members.length,
    distinctAuthors: authors,
    distinctSources: sources,
    isCrossSource: sources > 1 ? 1 : 0,
    ageMin: (ctx.now - story.promotedAt) / 60_000,

    // judge channel — ABSENT is distinct from NEGATIVE, and stays distinct
    judgePresent: judgement ? 1 : 0,
    judgeCoinable: judgement ? (judgement.coinable ? 1 : 0) : null,
    judgeConfidence: judgement?.confidence ?? null,

    // our channel — computed from our own data, never from the judge's prose
    nameProposed: proposed ? 1 : 0,
    nameIsGeneric: proposed ? (isGenericTicker(proposed) ? 1 : 0) : null,
    nameSpecificity: proposed ? subjectSpecificity(proposed, members) : null,
    nameLen: proposed ? proposed.length : null,
  };
}

/* ── hard gates: rules the judge cannot argue with ────────────────────── */

export function gate(f: FeatureVector, p: Policy): ReasonCode | null {
  if (f.judgePresent === 0)                                return 'Q1_unjudged';
  if (f.distinctAuthors < p.qualify.minDistinctAuthors)    return 'Q2_single_author';
  if (f.memberCount     < p.qualify.minMembers)            return 'Q3_too_thin';
  if (f.nameProposed === 0)                                return 'Q4_unnameable';
  if (f.nameIsGeneric === 1)                               return 'Q5_generic_name';
  if ((f.nameLen ?? 0) < p.qualify.minNameLen)             return 'Q6_name_too_short';
  if ((f.nameSpecificity ?? 0) < p.qualify.minSpecificity) return 'Q7_name_unspecific';
  return null;
}

/* ── score: only reached when every gate passed ───────────────────────── */

function score(f: FeatureVector, p: Policy): number {
  const w = p.qualify.weights;
  const raw =
      w.judgeConfidence * (f.judgeConfidence ?? 0)
    + w.specificity     * (f.nameSpecificity ?? 0)
    + w.crossSource     * f.isCrossSource
    + w.authorBreadth   * clamp01(f.distinctAuthors / p.qualify.authorBreadthFull);
  // The cap runs AFTER the model's contribution and can only LOWER the result.
  return Math.min(clamp01(raw), nameabilityCap(f, p));
}

/* ── the decision ─────────────────────────────────────────────────────── */

export function qualify(input: QualifyInput, p: Policy, ctx: StageContext): Decision {
  const f = extract(input, ctx);

  const base = {
    stage: NAME,
    subjectKind: 'story' as const,
    subjectId: input.story.storyId,
    featureAsOf: input.story.lastMemberAt,
    subjectOrigin: input.story.earliestPostAt,
    features: f,
    featureSet: FEATURE_SET,
    costUsd: input.judgeCostUsd,
  };

  // A model, if one is loaded, arrives as a pure sync closure on the Policy.
  // core NEVER imports ml/. This one line is "a rule today, a model tomorrow".
  const scorer = p.scorers.qualify;

  const blocked = gate(f, p);
  if (blocked !== null) {
    // ABSTAIN, not drop, when we simply never asked. The two are different
    // populations, and merging them poisons every recall number downstream.
    const verdict = blocked === 'Q1_unjudged' ? 'abstain' : 'drop';
    return makeDecision(
      { ...base, verdict, reason: blocked, score: null, decider: DECIDER }, ctx);
  }

  const s = scorer ? scorer.score(f) : score(f, p);
  const decider = scorer ? scorer.id : DECIDER;

  if (s < p.qualify.passScore) {
    return makeDecision(
      { ...base, verdict: 'drop', reason: 'Q8_score_below_bar', score: s, decider }, ctx);
  }

  // The judge saying "not coinable" is respected only HERE, after our gates.
  // 50–70% of stories landing on Q9 is the correct outcome, not a tuning target.
  if (f.judgeCoinable === 0) {
    return makeDecision(
      { ...base, verdict: 'drop', reason: 'Q9_judged_not_coinable', score: s, decider }, ctx);
  }

  return makeDecision(
    { ...base, verdict: 'pass', reason: 'Q0_coinable', score: s, decider }, ctx);
}
```

---

### 4.3 An adapter — `adapters/src/platform/tiktok/to-item.ts`

```ts
/**
 * The ONLY file in the repository permitted to know this vendor's field names.
 *
 * What this file does: shape translation, and an honest fidelity claim per
 * counter. What it does NOT do, ever: threshold, score, filter, decide, or
 * touch the network or the database. If you are tempted to add an `if` that
 * drops an item here, that `if` belongs in core/admit.
 *
 * The fidelity values below were MEASURED against live payloads, not assumed.
 * That is why they differ per field within the same object.
 */

import type { Item, CounterSet, Fingerprint, MediaRef, Fidelity, Millis }
  from '@insidor/contracts';
import { itemId, authorKey } from '@insidor/contracts/ids.ts';
import { SOURCE } from './capabilities.ts';

/* Measured: play and approval counts round to 4 significant figures.
   Comment, share and save counts are exact. */
const QUANTIZED_4: Fidelity = { kind: 'quantized', significantDigits: 4 };
const EXACT: Fidelity = { kind: 'exact' };

/* ── narrow, defensive readers. The vendor changes shape without notice. ─ */

const rec = (v: unknown): Record<string, unknown> =>
  (typeof v === 'object' && v !== null) ? v as Record<string, unknown> : {};

const str = (v: unknown): string | null =>
  typeof v === 'string' && v.length > 0 ? v : null;

const num = (v: unknown): number | null => {
  if (typeof v === 'number') return Number.isFinite(v) ? v : null;
  if (typeof v === 'string') { const n = Number(v); return Number.isFinite(n) ? n : null; }
  return null;
};

/* ── media ────────────────────────────────────────────────────────────── */

function media(raw: Record<string, unknown>): MediaRef[] {
  const v = rec(raw.video);
  const cover = str(v.cover) ?? str(v.originCover);
  if (!cover) return [];
  const durS = num(v.duration);
  return [{
    kind: 'image',
    uri: cover,
    width: num(v.width),
    height: num(v.height),
    durationMs: durS !== null ? durS * 1000 : null,
  }];
}

/* ── carriers ─────────────────────────────────────────────────────────── */

function formatIds(raw: Record<string, unknown>): string[] {
  const out: string[] = [];
  const music = str(rec(raw.music).id);
  if (music) out.push(`${SOURCE}:sound:${music}`);
  const effects = Array.isArray(raw.effectStickers) ? raw.effectStickers : [];
  for (const e of effects) {
    const id = str(rec(e).ID) ?? str(rec(e).id);
    if (id) out.push(`${SOURCE}:effect:${id}`);
  }
  return out;
}

/** Word 5-grams, lowercased. SHAPE only — the matching rule lives in core. */
function shingles(text: string): string[] {
  const w = text.toLowerCase().replace(/\s+/g, ' ').trim().split(' ').filter(Boolean);
  if (w.length < 5) return w.length ? [w.join(' ')] : [];
  const out: string[] = [];
  for (let i = 0; i + 5 <= w.length; i++) out.push(w.slice(i, i + 5).join(' '));
  return out;
}

function fingerprints(raw: Record<string, unknown>, text: string): Fingerprint[] {
  const out: Fingerprint[] = formatIds(raw).map(key => ({ kind: 'formatId', key }));
  // Perceptual hashing of the keyframe happens in the media pipeline, not here.
  // Emitting a fake hash would silently poison the free carrier join, which is
  // the grouper's whole basis. Absent is honest; invented is not.
  for (const key of shingles(text)) out.push({ kind: 'textShingle', key, bits: 64 });
  return out;
}

/* ── counters ─────────────────────────────────────────────────────────── */

function counters(raw: Record<string, unknown>, at: Millis): CounterSet {
  const s = rec(raw.stats);
  return {
    reach:        { value: num(s.playCount),    fidelity: QUANTIZED_4, observedAt: at },
    approval:     { value: num(s.diggCount),    fidelity: QUANTIZED_4, observedAt: at },
    conversation: { value: num(s.commentCount), fidelity: EXACT,       observedAt: at },
    rebroadcast:  { value: num(s.shareCount),   fidelity: EXACT,       observedAt: at },
    retention:    { value: num(s.collectCount), fidelity: EXACT,       observedAt: at },
    // `reproduction` is DELIBERATELY OMITTED, not set to zero. This source
    // exposes no reproduction COUNT; it exposes a lineage POINTER on the child
    // item only. capabilities.absent declares it, and core derives reproduction
    // from the fingerprint index instead. Writing 0 here would make every item
    // from this source look uncopied, which is the opposite of the truth — and
    // is exactly the live bug in parse-tiktok-post.js:86 today.
  };
}

/* ── the translation ──────────────────────────────────────────────────── */

/**
 * @param raw   one item from the vendor payload
 * @param at    the instant WE read it, INJECTED. This file never calls a clock:
 *              two items from one response must share one observedAt, and a
 *              recorded fixture must replay to a byte-identical Item.
 */
export function toItem(raw: unknown, at: Millis): Item {
  const r = rec(raw);
  const id = str(r.id) ?? str(r.awemeId);
  if (!id) throw new TypeError('to-item: payload has no item id');

  const text = str(r.desc) ?? '';
  const author = rec(r.author);
  const created = num(r.createTime);

  return {
    itemId: itemId(SOURCE, id),
    source: SOURCE,
    sourceItemId: id,
    // The numeric account id, never the @handle: handles are renamed and reused.
    authorKey: authorKey(SOURCE, str(author.id) ?? 'unknown'),

    // Seconds → millis. Null rather than a guess: postedAt feeds the pre-mint
    // ordering gate, and a confidently wrong timestamp there is worse than none.
    postedAt: created !== null ? created * 1000 : null,
    firstSeenAt: at,

    lang: str(r.textLanguage),
    text,
    media: media(r),
    counters: counters(r, at),
    fingerprints: fingerprints(r, text),

    // This source has no share-without-authoring object in its item feed.
    rebroadcastOf: null,
    // Stitch and duet BOTH create a new authored object: reproduction,
    // not rebroadcast. Getting this backwards inverts the product's signal.
    reproductionOf:
      str(rec(r.stitchInfo).sourceId) ?? str(rec(r.duetInfo).sourceId) ?? null,

    formatIds: formatIds(r),
    rawRef: `${SOURCE}/${id}.json`,
  };
}
```

**One thing types do not buy, said plainly.** Types are erased at runtime. `toItem(raw: unknown)`
proves nothing about the vendor's JSON. That is why `adapters/src/codec/` exists — a runtime
decoder per platform, and a contract test suite running against recorded fixtures so a vendor
shape change fails CI. This cost is identical under TypeScript, JSDoc and plain JavaScript, and
it is worth saying out loud because over-trusting a type system at the network edge is the most
common way a team believes it is protected when it is not.

---

## 5. The rules that keep it clean

### The dependency rule

```
contracts   →  nothing
core        →  contracts
adapters    →  contracts                    (never core)
store       →  contracts                    (never core, never adapters)
view        →  nothing
ml/serve    →  contracts
services    →  contracts, core, adapters, store, ml
app         →  view                         (and NOTHING else internal)
eval        →  contracts, core, store, view (never adapters)
```

Read it once and the shape is obvious: **logic never knows who it is talking to, and vendors
never know what the logic decides.**

### How it is enforced — four layers, and one of them was wrong

The architecture document claimed `package.json` is "free, and the strongest" enforcement. That
was tested, and it is **false under npm workspaces**: npm hoists every workspace into the root
`node_modules`, so `adapters` imported `@insidor/core` without declaring it and got the right
answer back. `npm ls` never noticed. Under **pnpm**, the same import dies with
`ERR_MODULE_NOT_FOUND`, because pnpm symlinks only declared dependencies into each package.

So the enforcement lives in the package manager, and the package manager must be pnpm. Two
further holes were found and closed:

- **A relative path escapes it.** `import '../../core/src/index.ts'` bypasses module resolution
  entirely. Closed by `rootDir: "./src"` in each package's tsconfig — `tsc` reports `TS6059`
  during the typecheck you already run.
- **Root dependencies leak to every package.** Anything installed with `pnpm add -w` is
  importable undeclared from anywhere, because Node's resolution walks up. Since shared tooling
  lives at the root, the accurate statement is *"pnpm blocks cross-package phantom imports,"*
  not *"pnpm blocks phantom imports."* Which is why layer 3 exists and is not optional.

**Layer 1 — `pnpm-workspace.yaml` and nine `package.json` files.** `core/package.json` has one
dependency and no devDependencies at all, because its tests use `node:test`, a builtin.
`adapters/package.json` does not list `@insidor/core`, and under pnpm that omission is
load-bearing on the developer's own machine, before CI, before a review.

**Layer 2 — `tsconfig.json` per package.** `rootDir` closes relative escapes; TypeScript
independently reports `TS2307: Cannot find module '@insidor/core'` for the undeclared import.
Two mechanisms, one line of JSON each.

**Layer 3 — dependency-cruiser, in allow-list form.** The earlier draft of this config was
deny-list shaped (`adapters must not import core`), which lets `adapters → store`,
`adapters → services` and `ml → adapters` through unnoticed. Allow-list form plus an
undeclared-dependency catch-all:

```js
// .dependency-cruiser.cjs
/** @type {import('dependency-cruiser').IConfiguration} */
module.exports = {
  forbidden: [
    // ★ the catch-all the earlier config was missing: anything imported but
    //   not declared, in any package, from any source.
    { name: 'no-undeclared-dep', severity: 'error', from: {},
      to: { dependencyTypes: ['npm-no-pkg', 'npm-unknown', 'undetermined'] } },

    { name: 'contracts-is-a-leaf', severity: 'error',
      from: { path: '^contracts/' }, to: { pathNot: '^contracts/' } },

    { name: 'view-is-a-leaf', severity: 'error',
      from: { path: '^view/' }, to: { pathNot: '^view/' } },

    // ALLOW-LIST: core may reach contracts and itself, and nothing else.
    { name: 'core-imports-only-contracts', severity: 'error',
      from: { path: '^core/', pathNot: '\\.test\\.ts$' },
      to:   { pathNot: '^(core|contracts)/' } },

    { name: 'core-has-no-runtime-deps', severity: 'error',
      from: { path: '^core/', pathNot: '\\.test\\.ts$' },
      to:   { dependencyTypes: ['npm', 'npm-dev', 'core'] } },  // 'core' = node builtins

    { name: 'adapters-reach-contracts-only', severity: 'error',
      from: { path: '^adapters/' }, to: { pathNot: '^(adapters|contracts)/' } },

    { name: 'store-is-a-leaf', severity: 'error',
      from: { path: '^store/' },
      to:   { pathNot: '^(store|contracts|view)/' } },

    // ★ the rule that stops internal scoring reaching the app. See below.
    { name: 'app-imports-only-view', severity: 'error',
      from: { path: '^app/src/' },
      to:   { path: '^(core|adapters|services|ml|store|contracts|eval|tools)/' } },

    { name: 'eval-never-hits-the-network', severity: 'error',
      from: { path: '^eval/' }, to: { path: '^adapters/' } },

    { name: 'watchdog-is-alone', severity: 'error',
      from: { path: '^services/src/watchdog/' },
      to:   { path: '^services/src/(runner|chainwatch)/' } },

    { name: 'features-never-see-outcomes', severity: 'error',
      from: { path: '^core/src/features/' }, to: { path: '(winner|outcome|label)' } },

    { name: 'no-circular', severity: 'error', from: {}, to: { circular: true } },
  ],
  options: {
    doNotFollow: { path: 'node_modules' },
    tsConfig: { fileName: 'tsconfig.base.json' },
    tsPreCompilationDeps: true,
    enhancedResolveOptions: { extensions: ['.ts', '.mts', '.tsx'] },
  },
};
```

Two mistakes worth recording because both were made and corrected by running it: `extensions`
is not a valid top-level option (it lives under `enhancedResolveOptions`, and the wrong version
fails with an opaque schema error), and without `pathNot: '\\.test\\.ts$'` the purity rules fire
on core's own test files, which legitimately import `node:test`.

**Layer 4 — greps, in `tools/`.** `check-vocabulary.mjs` fails on any platform, chain or vendor
word inside `core/` or `contracts/`. `check-purity.mjs` fails on `Date.now`, `new Date`,
`Math.random`, `fetch`, `process.env`, `node:` imports, `require`, or `async`/`await` in
`core/`. `check-policy.mjs` fails on a bare numeric literal in `core/` outside a small
allowlist. These are regex checks: they stop accidents, not adversaries, which is the correct
threat model for a two-person team — the real guarantees are the three layers behind them.

One trap, hit and corrected: compare the **relative** path, not the absolute one. An absolute
path containing a banned word (a home directory, a CI runner name) produces a false positive on
every file and looks exactly like a real failure.

**CI is two commands.**

```yaml
# .github/workflows/ci.yml
name: ci
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v5
        with: { node-version-file: '.nvmrc', cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm check       # types → vocab → purity → policy → python → boundaries
      - run: pnpm test        # node --test, plus the LightGBM parity fixture
      - run: pnpm graph && git diff --exit-code docs/graph.svg
```

That last line is the best idea in the whole enforcement scheme: a changed dependency-graph SVG
in a pull request is the most legible possible signal that somebody added a coupling, and it is
legible to a non-engineer.

**Two operational notes that will otherwise cost an evening each.** pnpm 11 refuses dependency
build scripts by default — `pnpm install` exits 0 and prints `ERR_PNPM_IGNORED_BUILDS`, and the
failure then appears at runtime with an error that does not name the cause. Commit
`onlyBuiltDependencies` in `pnpm-workspace.yaml` on day one covering `esbuild`, `sharp` and any
native module, and again the moment Playwright arrives. And `"packageManager": "pnpm@11.21.0"`
plus a `preinstall` guard (`npx only-allow pnpm`) in the root, because one habitual `npm
install` silently restores hoisting and removes the strongest boundary with no error and green
CI.

### The rule that stops internal scoring reaching the app, made structural

This is the requirement with the worst track record. Six leaks shipped in the previous build,
and three of them were *reasonable-looking UI written against data the client happened to
have*: `site/live.js:397` asks the database for `post_meme_scores!inner(meme_score,
suggested_ticker)`, so the client had the score, so the client rendered the score. A leak that
arrives as a `.select()` string is not something code review reliably catches.

So the fix is not a rule. It is four walls, and each one is checkable.

**Wall A — a separate wire vocabulary with no internal field in it.** A new zero-dependency
package, `view/`, which **does not import `contracts/`**. This corrects the architecture
document's own dependency rule, which permitted `app → contracts` — but `contracts/` holds
`Decision`, `Policy` and `FeatureVector`, so an app allowed to import it can name the type of a
score, and typed access is one `select('*')` from rendered access. The duplication between
`Item` and `BoardRow` is roughly 80 lines and it *is* the wall.

```ts
// view/src/board.ts — zero dependencies, does not import contracts
/** A judgement, as a closed enum. The client cannot re-threshold an enum. */
export type Tone = 'rising' | 'steady' | 'cooling';

export interface BoardRow {
  id: string;
  title: string;
  thumbUrl: string | null;
  sources: readonly { source: 'x' | 'tiktok' | 'reddit'; count: number }[];
  authorCount: number;          // a fact
  postCount: number;            // a fact
  reach: number;                // a fact
  reachDelta24h: number | null; // a fact
  momentum: Tone | null;        // a judgement, already made, not re-derivable
  firstSeenAt: string;
  mint: MintState;
  isNew: boolean;
}
```

No `score`, no `heat`, no `eta`, no `burst`, no `confidence`, no `propensity`, no `policyHash`,
no `costUsd`. **No `rank` field either** — position is the array index, so a row cannot carry
"our rank" off the board.

The important move is `momentum`. Internally, burst is a number. The projection turns it into a
three-valued enum, one way, forever. The client has nothing to threshold — which is exactly
what `feed.js:549`'s `gain >= 150000 ? 'up' : 'down'` was.

**Wall B — the projection is a runtime `pick()` against an exported allowlist, with a test that
reads the values.**

```ts
// store/src/projections/board.test.ts
test('the projection emits exactly the public field set', () => {
  expect(Object.keys(toBoardRow(fixtures.rankedStory)).sort())
    .toEqual([...BOARD_ROW_FIELDS].sort());
});

/** Reads the VALUE, so a leak nested inside `mint` or `sources` is caught too. */
test('no internal vocabulary survives the projection', () => {
  const json = JSON.stringify(toBoardRow(fixtures.rankedStory)).toLowerCase();
  for (const w of ['score','propensity','policy','explore','holdout','eta',
                   'burst','heat','threshold','anthropic','claude','budget'])
    expect(json).not.toContain(w);
});
```

**Wall C — the table the browser reads is a physical table with no internal column in it.** Not
a view over an internal table, not a column list in a `.select()`, not a row-level-security
policy. A table.

```sql
create table public.board_row (
  view_id text not null, id text not null, slot int not null,
  title text not null, thumb_url text, sources jsonb not null,
  author_count int not null, post_count int not null,
  reach bigint not null, reach_delta_24h bigint,
  momentum text check (momentum in ('rising','steady','cooling')),
  first_seen_at timestamptz not null,
  mint jsonb not null default '{"kind":"none"}'::jsonb,
  is_new boolean not null default false,
  updated_at timestamptz not null default now(),
  primary key (view_id, id)
);

-- `internal` is NOT in PostgREST's exposed schemas, so it has no URL at all.
-- There is no grant to get wrong because the HTTP layer cannot name the schema.
grant select on public.board_row, public.board_tick to anon;
```

`select * from public.board_row` is safe, and that is the whole design. Every other approach is
a rule someone has to remember, and `live.js:397` is what forgetting looks like.

**Wall D — a vocabulary gate over `app/src`,** banning internal machinery words (`score`,
`confidence`, `threshold`, `burst`, `eta`, `propensity`, `holdout`), internal words for
user-facing things (`narrative`, `cluster`, `candidate`, `admit`, `qualify`), every vendor name
(`anthropic`, `claude`, `apify`, `twitterapi`, `dexscreener`, `helius`, `rugcheck`), and every
spend word. Plus a lint rule that a numeric comparison against a literal ≥ 100 inside a
component is an error, because a threshold in a component is a product judgement.

Scored against the six known leaks: **five are foreclosed structurally, one is not.** The
honest weak spot is the coloured delta — `reachDelta24h` is a legitimate fact, so the field
stays. The mitigation is that `ui/primitives/Delta.tsx` colours by **sign only**, and no
primitive accepts a numeric colour input. That one is design-system-plus-review, not structure,
and it should be labelled as such rather than claimed.

---

## 6. Where everything runs

### The processes

Three long-lived Node processes. No serverless anywhere in the pipeline.

| Process | What it is | Where | Why there |
|---|---|---|---|
| `runner` | six supervised loops in one process — admit, track, detect, group, qualify, rank | Railway, 1 GB, always on | The loops already exist and work (`worker/index.js`); they were written and then never used in production |
| `chainwatch` | the mint cursor and coverage log | Railway, 512 MB, always on | Separate **deployable**, because every `runner` deploy restarts the process, and a restart mid-cursor is a gap in the coverage log — and a gap makes a label *censored* rather than *negative* |
| `watchdog` | one query loop, alerts to Telegram | **Fly.io**, 256 MB, $2/mo | A watchdog on the same provider as the runner cannot report a provider-wide incident, which is one of the three ways this pipeline actually dies |
| `app` | Next-less Vite SPA, static assets | Vercel Pro | Per-PR preview URLs are how a non-engineer reviews without running anything |

**One correction to carry forward: there is no chain websocket.** The architecture document's
own §7 calls `chainwatch` "the one long-lived connection" while §6.3 argues against a streaming
subscriber — the document disagrees with itself, and §6.3 is right. Its first argument (a
serverless deployment cannot hold a connection) expires the day we move to Railway. Its second
survives: a Helius plan with mainnet gRPC is **$499/month**, roughly three times the entire
infrastructure budget, to buy 30 seconds of latency on a window measured in days. `chainwatch`
polls every 20 seconds. Whoever edits the architecture document should change "one long-lived
connection" to "one long-lived **process**", or the next reader will build a websocket.

**Why serverless failed here, precisely,** because it determines the fix. Three distinct
failures, and only one is about the 60-second ceiling:

1. The ceiling forced `worker/lib/time-guard.js` — a hand-rolled deadline that stops at 50s and
   persists a resume cursor. A continuation mechanism invented to work around a platform limit
   is a permanent source of partial-run bugs.
2. No singleton guarantee. Two overlapping invocations both drain the same tracking queue.
   There is no lock, because PostgREST cannot take one.
3. No liveness signal. This is the seven hours.

(For the record: the ceiling argument is now partly stale — Vercel Pro supports one-minute cron
cadence and an 800-second `maxDuration`. It does not change the conclusion, because 2 and 3
stand, but anyone reasoning from `maxDuration: 60` should re-check the numbers first.)

### How a dead stage is noticed within minutes

The current build **cannot tell "died" from "found nothing."** The cause is one line:
`api/cron/_lib/run-stage.js:48` calls `recordLastRun(sb, stage)` *inside* the `try` block. A
stage that throws writes nothing. A stage that succeeds and finds zero rows writes exactly what
a stage that succeeds and finds a thousand rows writes. There is no state in the database
distinguishing the two. That is why seven hours went unnoticed.

Three independent layers replace it, because the failure was a *detection* failure and one
layer has failure modes of its own.

**Layer 1 — a run row written on start, closed in a `finally`.**

```sql
create table internal.stage_runs (
  id           bigserial primary key,
  stage        text        not null,
  started_at   timestamptz not null default now(),
  finished_at  timestamptz,             -- ★ NULL forever == the process was killed
  outcome      text check (outcome in ('ok','empty','error')),
  err          text,
  items_in     integer,
  items_out    integer,
  compression  double precision,        -- items_in / items_out, free diagnostic
  duration_ms  integer,
  host         text not null
);
create index stage_runs_open_idx on internal.stage_runs (started_at)
  where finished_at is null;
```

```ts
// services/src/runner/main.ts — the loop body, in full
async function loop(l: Loop) {
  await sleep(l.offsetMs);
  while (!stopping) {
    const started = Date.now();
    const runId = await openStageRun(pool, l.stage);   // ★ BEFORE the work
    let outcome: 'ok' | 'empty' | 'error' = 'ok';
    let counts = { in: 0, out: 0 };
    let err: string | null = null;
    try {
      counts = await l.run({ pool, signal: ac.signal, now: () => Date.now() });
      outcome = counts.out === 0 ? 'empty' : 'ok';     // ★ 'empty' is not 'error'
    } catch (e) {
      outcome = 'error';
      err = (e as Error).message;
      console.error(`[runner:${l.stage}]`, e);
    } finally {                                        // ★ ALWAYS closed
      await closeStageRun(pool, runId, {
        outcome, err, itemsIn: counts.in, itemsOut: counts.out,
        compression: counts.out > 0 ? counts.in / counts.out : null,
        durationMs: Date.now() - started,
      });
      await fetch(`https://hc-ping.com/${l.hc}${outcome === 'error' ? '/fail' : ''}`)
        .catch(() => {});
    }
    const jitter = Math.random() * 2_000;   // stops six loops convoying
    await sleep(Math.max(1_000, l.everyMs - (Date.now() - started) + jitter));
  }
}
```

Three properties fall out for free. `outcome = 'empty'` is a distinct, queryable state from
`'error'`. A killed process leaves `finished_at IS NULL` forever. And `compression` — the
input-to-output ratio the architecture document uses as its primary funnel diagnostic — is
recorded on every run without anyone computing it.

**Layer 2 — a dead-man's switch per loop.** Each cycle pings a Healthchecks.io URL with a grace
period of 3× the cadence. This catches "the whole process is gone" without the database being
involved at all. Nine checks (six loops, chainwatch, watchdog, trainer) against a 20-check
tier.

**Layer 3 — the Fly watchdog, every 60 seconds,** querying over a read-only Postgres role that
can `SELECT` two tables and nothing else. It alerts on: any stage whose last successful
`finished_at` is older than 3× its cadence; any run open longer than 10 minutes; any gap in the
mint coverage log over 180 seconds; and any stage whose compression ratio moved more than 3×
week-over-week. It pings its own dead-man's switch, so the watchdog dying is also detected.

**Detection time: under three minutes.** Against seven hours.

One residual accepted rather than removed: if Healthchecks.io has an outage at the same moment
the pipeline dies, nothing alerts. Three independent layers failing simultaneously is a risk two
people should take rather than pay to remove.

### The real monthly cost

| Line | Now | After embeddings |
|---|---|---|
| Vercel Pro (app only, **no crons**) | 20 | 20 |
| Railway Pro — runner + chainwatch (incl. $20 usage credit) | 20–25 | 20–25 |
| Fly.io watchdog, 256 MB | 2 | 2 |
| Supabase Pro + Small compute | 30 | 75 *(Medium — HNSW builds want RAM)* |
| Database disk beyond 8 GB | 4 | 8 |
| Blob storage for raw payloads | 1 | 2 |
| Healthchecks.io Supporter | 5 | 5 |
| Embeddings (gemini-embedding-001) | 0 | ~0.35 |
| Solana RPC — Helius Developer | 49 | 49 |
| CI, training job, Sentry free, Telegram | 0 | 0 |
| **Total** | **~$131** | **~$181** |

The $49 Helius line is a data vendor, not infrastructure; excluding it the infrastructure bill
is about $82/month. Two costs to budget in advance rather than discover: Supabase steps from
Small to Medium when embeddings land, and the symptom will present as *"grouping is slow"*
rather than *"the database needs RAM"* — write that down now. And Railway bills by usage, so a
loop whose interval is accidentally zero is a runaway invoice; the `Math.max(1_000, …)` floor in
the sleep computation above is not decorative.

### The database, and the one client change that matters

One Supabase Postgres, three schemas: `public` (what PostgREST exposes), `internal` (the
record), `legacy` (the old tables, renamed and frozen). Nothing splits out. 150k decision rows
a day at ~1 KB is 4.5 GB/month and ~55M rows/year — that is two inserts per second, which is a
daily-partitioned table with a BRIN index and `DROP PARTITION` retention, not a Timescale
problem. pgvector 0.8 ships `halfvec`, `bit`, and HNSW with `bit_hamming_ops`, which is exactly
what the Tier 1 carrier join needs, in infrastructure already provisioned.

**But the server processes must use `pg` over Supavisor session mode, not
`@supabase/supabase-js` over PostgREST.** That single change buys: real multi-statement
transactions (so the log-then-act asymmetry becomes a choice rather than a constraint);
`FOR UPDATE SKIP LOCKED` (the tracking claim queue, which replaces the entire case for BullMQ
or pg-boss); session advisory locks (a genuine singleton guard — the thing serverless could not
provide); `COPY` for bulk observation inserts; `store/` genuinely being the only place SQL
exists; and the `internal` schema simply never being taught to PostgREST, so there is no grant
to get wrong. Session mode is port 5432, IPv4 on every tier. Transaction mode (6543) drops
prepared statements and cannot hold a session lock, so it is wrong for the runner.

The app keeps `@supabase/supabase-js` for Auth, Realtime and reads of `public`.

---

## 7. The frontend

### The framework, decided — and one claim corrected

**React 19 on Vite 8, a single-page app, deployed to Vercel as static assets. Not Next.js.**

An earlier draft justified React with: *"the current build is already a React app, poorly — it
CDN-loads React at runtime."* That is checkable and it is **wrong in scope**. React appears in
one file — `site/features/wallet/privy.js`, 174 of 4,091 lines, 4.3% — as the peer dependency of
one vendor SDK. The other ~3,900 lines are imperative DOM: `feed.js` (1,133 lines) has 61
`innerHTML`/`createElement`/`querySelector` calls, `live.js` (2,051 lines) is a vanilla IIFE.
And `@privy-io/js-sdk-core` — the framework-agnostic client — is **already in package.json and
imported by nothing**.

So choosing React *is* adopting a framework: a bundler where there is none today, JSX, and a
component rewrite of ~3,900 working lines. That cost is ahead of the team, not behind it, and
it should be priced honestly.

It is still the right call, for three reasons that survive the correction:

1. **The wallet and the trade path are the only surfaces where a bug costs a user money**, and
   they should run on the vendor's first-class SDK inside its native runtime. Privy ships
   React. Jupiter's tooling assumes React. There is no supported Svelte path, and mounting a
   React root inside Svelte to get a wallet is exactly the mess that exists today.
2. **The board is being rebuilt regardless.** The reconciler is already hand-written — and it
   is not where the earlier draft said it was. `live.js:893` is a 28-line delegating wrapper;
   the actual reconciler is `feed.js`: `syncNarrativesTable()` at :404-475,
   `reconcileNarrativeOrder()` at :356-368 (insertBefore-based keyed-child ordering — literally
   React's algorithm, by hand), `applyNarrativeRowCells()` at :369-403, over five hand-invalidated
   caches, using `JSON.stringify` per row per tick as the diff. The "vanilla is simpler"
   position is disproved by `feed.js`, not by `live.js`.
3. **There is a live latent bug in that reconciler that a second chain will trigger.** The row
   template emits four `.nm-col` divs; the patch path addresses the first two by attribute and
   the last two **positionally** (`cols[cols.length-2]`, `cols[cols.length-1]`). Add a fifth
   column — a second chain's market cap, a model score — and the patch writes into the wrong
   div, while first paint still renders correctly. The bug appears only after the first live
   tick, on real data, in production. Fix or flag that before adding anything, whichever
   framework wins.

**Why not Next.js.** No anonymous surface, no SEO, no cacheable content, and every number
invalidated by the next tick. Server components render HTML that is stale before hydration
completes, then hydrate into a client component that refetches everything anyway — two
rendering models, one used. And the correctness of the whole thing depends on knowing which of
two module graphs a file is in. A framework whose failure mode is *a misplaced `'use client'`
puts a key in the browser bundle* is the wrong framework for a fast-and-loose developer with no
reviewer. A Vite SPA has one module graph and one runtime, and `import 'server-only'` becomes
unnecessary rather than load-bearing.

The honest counter-argument is that Next gives you a server, so the anon key stays private. It
is already public (`site/config.js:8`) and the board is the product. The security boundary is
`internal` not being exposed to PostgREST, which holds regardless of framework. If the read
surface later has to become authenticated, `app/src/data/` is the only directory that changes,
and it is about 200 lines.

### Structure

`routes/` is layout and `<Suspense>` only — no logic, no fetch. `data/` is the only place that
touches the network. `live/` is the ordering machine. `features/<surface>/` holds anything that
knows the domain; `ui/primitives/` holds anything that does not, and `ui/` is a leaf that
imports nothing from `src/`. Cross-feature imports go through `features/B/index.ts` only. All
five of those are dependency-cruiser rules, not conventions.

Design system: CSS Modules over one `tokens.css`. This is the closest call in the document, and
the tiebreaker is stated rather than hidden: Tailwind is faster to generate and suits a
fast-moving developer, and I would not fight about it. It loses on two specifics — this is a
*table-dense* app, so column widths in `ch`, `font-variant-numeric: tabular-nums` and a fixed
row rhythm are three CSS declarations versus a wall of arbitrary-value classes repeated per
cell; and `site/styles/index.css` is 1,003 lines of a coherent dark terminal aesthetic that
ports across in an afternoon this way and gets rewritten from scratch the other way. **If the
developer prefers Tailwind, take it.** The swap is a week and touches no boundary.

Deliberately absent, with reasons: `@tanstack/react-table` (its purpose is client-side sorting
and filtering, which the ranking design forbids — importing it invites the thing you banned);
Jupiter's embedded plugin (it brings its own wallet-connect UI, duplicating Privy and requiring
glue between two vendors' React contexts — use the Ultra REST API instead, three HTTP calls and
one signature); Zustand or Redux (the board needs per-row subscriptions and an order/value split
with a freeze buffer, about 120 lines no store library provides — you would write the same 120
lines inside Zustand).

### Live data without the board reordering under the cursor

The ordering problem is solved server-side and the transport is almost incidental. **The server
commits `(tick, order)` and the client never sorts.** That is already the ranking design's
decision; the frontend's job is to honour it.

Transport is one Supabase Realtime **broadcast** channel — chosen over `postgres_changes` for a
product reason rather than a scaling one: the payload is built by a database trigger, so it is
an explicit projection you wrote, not a row shape someone else configured. It is the same
websocket you already pay for, and no new deployment target.

The client half is the real work, and it is not a library.

```ts
// app/src/live/boardStore.ts — ~120 lines. The two subscription sets are the point.
export function createBoardStore() {
  let order: readonly string[] = [];
  let tick = 0;
  let frozen = false;
  let pending: BoardTick | null = null;
  const rows = new Map<string, BoardRow>();

  const orderLs = new Set<Listener>();          // notified on a real tick
  const metaLs  = new Set<Listener>();          // notified on freeze/transport
  const rowLs   = new Map<string, Set<Listener>>();  // ONE row, ONE listener set

  function applyTick(t: BoardTick) {
    if (t.tick <= tick) return;                       // sockets deliver out of order
    if (tick !== 0 && t.tick > tick + 1) onGap?.(t.tick);  // a hole → refetch
    tick = t.tick;
    for (const r of t.rows) { rows.set(r.id, r); emit(rowLs.get(r.id)); }
    order = t.order;                                  // new array ref ONLY here
    pending = null;
    emit(orderLs); emit(metaLs);
  }

  return {
    /** VALUES PATCH ALWAYS — even while frozen. Freezing values would be lying. */
    patch(p: RowPatch) {
      const cur = rows.get(p.id);
      if (!cur) return;
      rows.set(p.id, { ...cur, ...p.fields });
      emit(rowLs.get(p.id));                          // ONE row re-renders. Not sixty.
    },

    /** ORDER waits while the user is touching the board. */
    tick(t: BoardTick) {
      if (!frozen) return applyTick(t);
      pending = t;
      emit(metaLs);                                   // → renders the "3 updates" pill
    },

    setFrozen(next: boolean) {
      if (frozen === next) return;
      frozen = next;
      if (!next && pending) applyTick(pending); else emit(metaLs);
    },
    // …getOrder, getRow, subscribeOrder, subscribeRow, subscribeMeta, reset
  };
}
```

```ts
// app/src/live/freeze.ts
/** Order is held while the pointer, keyboard focus OR scroll is inside the board.
 *  Hard release at 30s, so a parked cursor cannot freeze the board forever. */
export function useFreezeWhileInteracting(ref: RefObject<HTMLElement | null>) {
  const store = useBoardStore();
  useEffect(() => {
    const el = ref.current; if (!el) return;
    let soft: number | undefined, hard: number | undefined;

    const hold = () => {
      clearTimeout(soft);
      store.setFrozen(true);
      clearTimeout(hard);
      hard = window.setTimeout(() => store.setFrozen(false), 30_000);
    };
    const leave = () => {
      clearTimeout(soft);
      soft = window.setTimeout(() => { clearTimeout(hard); store.setFrozen(false); }, 900);
    };

    el.addEventListener('pointerenter', hold);
    el.addEventListener('pointerleave', leave);
    el.addEventListener('focusin', hold);
    el.addEventListener('focusout', leave);
    // A scroll is a read, not a hover — hold, or rows slide past under the eye.
    el.addEventListener('scroll', hold, { passive: true });
    return () => { /* …remove all five, clear both timers */ };
  }, [ref, store]);
}
```

Why this does not reorder under the cursor, mechanically, in four steps:

1. `order.map(id => <Row key={id} …/>)` — React **moves** existing DOM nodes rather than
   recreating them, so hover, focus and text selection survive a reorder that *is* applied.
2. `order` only ever changes inside `applyTick`, which is gated by `frozen`.
3. `frozen` is true whenever the pointer, keyboard focus or scroll is inside the board.
4. Values keep patching while frozen, so the board is never stale — only its *ordering* is
   deferred, and the deferral is visible as a pill saying how many updates are waiting.

This is a genuine product tradeoff, not a free win, and it should be shown to the founder before
it ships. A user who parks the cursor sees an order up to 30 seconds old while the numbers keep
moving, which can read as a bug. The alternative failure — a row moving as you click it — is
worse but less visible. One interaction to decide deliberately: a brand-new explosive entrant
flagged `isNew` still sits behind the freeze. Either exempt it (it appends rather than reorders)
or accept up to one freeze window of delay on the product's headline moment.

**One reconnect bug that must not be reproduced.** Today `live.js:1751-1755` calls
`stopFallbackPoll()` on reconnect and issues no refetch, so every row that changed during a
socket outage is silently missing until something unrelated refetches. Broadcast has **no
replay**, so the rule is: refetch authoritatively on every `SUBSCRIBED` transition *and* on any
gap in the tick sequence. Both are in the store above.

---

## 8. Getting there from here

### Strangle, not greenfield — and the reason is not about code

"Greenfield or strangle" asks about code. The thing that cannot stop is not code; it is writes
to one Postgres. Once the database is fixed as immovable, strangling becomes nearly free and
greenfield becomes indefensible — so **the database decision makes the codebase decision for
you.**

Greenfield fails for one specific reason. One developer. For the eight-to-eighteen weeks of a
greenfield build, nobody is watching the old system — and the old system is the sole producer of
the perishable data. That this is not hypothetical is on disk: seven hours of silent ingest
death; `worker/index.js`'s last heartbeat 2026-07-24 while crons ran through 08-06; two recovery
`UPDATE` blocks in `schema-ingest-floors.sql` written to undo floors that poisoned themselves.
The greenfield failure mode is not a bad cutover day. It is that the old system dies quietly in
week five, nobody notices for a fortnight, and you arrive at cutover having lost the data *and*
without a baseline to compare against.

Rewrite-in-place fails differently: the boundary rules are all-or-nothing globs, and they cannot
go green while `worker/ingest/lib/ingest-floors.js:4` and `worker/adapters/x/budget.js:275` hold
a live require-cycle between the vendor adapter and the admission policy. Gates that have never
been green are gates nobody turns on. You would get the file tree without the property the file
tree exists to produce — which is the current state, where `worker/adapters/` is a folder rather
than a boundary.

### The mechanism, in three parts

**1. `git mv` the current build into `legacy/` in the first commit.** The boundary rules glob
`^(contracts|core|adapters|store)/` and never see it, so CI can be green on day two on an empty
tree rather than red until week eighteen. It also makes progress legible: every pull request
that *deletes* a `legacy/` directory is a completed step, visible in the diff. `vercel.json`
points at `legacy/api/**` until the app moves — a one-line change.

**2. One environment variable per stage says who owns that stage's vendor key.**

```
INSIDOR_STAGE_OWNER_INGEST=legacy      # legacy | next
INSIDOR_STAGE_OWNER_TRACK=legacy
INSIDOR_STAGE_OWNER_GROUP=legacy
INSIDOR_STAGE_OWNER_RESOLVE=legacy
INSIDOR_STAGE_OWNER_RANK=legacy
```

A process that does **not** own a stage runs it in shadow: it computes the `Decision`, writes it
with `shadow_of` set, performs no side effect, and makes no paid call. So during every parallel
period there is zero double-billing (only the owner holds the key), zero risk that the new code
stops collection (it cannot write), and the new core is exercised on live production data
continuously. The new tree reads the old tree's writes through
`adapters/src/platform/replay/` — which the build order already requires for contract testing
and for giving `eval/` a network-free data source.

**Rollback is flipping one string and redeploying. Under two minutes. No migration, no restore,
no data loss.** That is why seven cheap reversible moments beat one expensive irreversible one.

**3. One database, three schemas.** Migration is `INSERT … SELECT` rather than dump-and-restore
— restartable, countable, verifiable with a single query. The parallel-run comparison is a
`JOIN` rather than a cross-database reconciliation pipeline that itself needs maintaining. Every
migration **creates new tables and only reads the live ones**, and the `legacy.*` rename happens
per-stage only after that stage's writer is switched off.

### What keeps running during, and what is protected first

Continuity is guaranteed by an alarm, not by a plan. Right now there is no way to know that
collection stopped, so "we will keep it running" is a hope. **Before any transition work**: the
GitHub Actions watchdog (deliberately a different vendor from Vercel, because the thing most
likely to fail *is* Vercel cron), plus moving `recordLastRun` out of the `try` at
`api/cron/_lib/run-stage.js:48`. Roughly forty lines, and it is the highest-return week in the
plan.

### What data moves — and three corrections that change the work

**The 57,032-row snapshot series is safe, but back up three tables, not two.** Verified:
`post_snapshots` is insert-only — no TTL, no purge, and pruning writes `tracking_status =
'pruned'` rather than deleting. Six `.delete()` call sites exist across the codebase and none
touch it. But the cascade has **two hops**, not one: `narratives → narrative_posts →
post_snapshots`, and 1,266 of 2,411 live posts carry a non-null `narrative_id`. So one
`DELETE FROM narratives` — a human clearing junk clusters in a dashboard — destroys roughly half
the series, and a backup of only the two lower tables **cannot be restored**, because the
foreign key rejects rows whose parent narratives are gone.

And "a ten-minute `pg_dump`" is not available: there is no Postgres client on the machine, no
`DATABASE_URL` in the environment, and `pg` is not in `package.json`. The honest path uses the
already-wired supabase-js service client with `.range()` pagination — PostgREST silently caps
at 1,000 rows regardless of the `limit` parameter, so a naive export captures 1.75% of the
series and looks like it worked. Roughly 58 pages, ~324 MB, keep the raw JSON.

**The migration's real risk is a lossy transform, not a `DELETE`.** The observation backfill
un-maps TikTok's counters from X's column names — `playCount` out of `views`, `shareCount` out
of `retweets` — and it must **deliberately omit** `reproduction` for TikTok. Writing the
hardcoded `quotes = 0` as `reproduction = 0` would fabricate 57,000 observations and make every
reproduction-derived feature systematically lower for TikTok by construction. Absent is not
zero. That single omission is why the migration is hand-written rather than generated, and the
migration needs per-platform, per-kind row-count assertions before anyone trusts it.

**The "8,101-row near-miss corpus" does not exist as history, and the fix is not three lines.**
`near-miss.js:9` sets a 12-hour TTL and `:35-44` hard-deletes. 8,101 rows is about twelve hours
of arrivals on a rolling buffer.

But the 12-hour TTL is the **lesser** of two reapers. A second hard delete at `:76-83` removes
any row whose post exceeds `MAX_POST_AGE_MIN_X` (default 360 minutes), measured from post
creation — and since posts are only admitted to near-miss inside that same 6-hour bound, the
stale boundary *always* precedes the TTL boundary. Every row the recheck loop actually touches
— the top 50 by views, the high-signal rows with the richest trajectories — dies at ~6 hours and
never reaches the TTL. Patch only the TTL and what survives is the never-rechecked low-view
tail: close to the inverse of the intended corpus. A third delete at `:98-101` erases the
promotion event.

Worse, the naive soft-expire is a live regression: the recheck `SELECT` at `:49-53` has no
expiry predicate and orders by `views desc limit 50`, so expired high-view rows would occupy
the batch permanently, burning one API call per cycle forever while live near-misses starve.
And `recheckNearMisses` hardcodes X's `getTweetsByIds` at `:61` with no platform filter, so
TikTok rows would become immortal and get fed to X's by-ID endpoint.

**The cheaper fix that gets most of the value: leave all three deletes alone and add an
append-only `INSERT` into an archive table immediately before each one.** Write-path only. No
read-path filter to get wrong, no batch-poisoning risk, no schema change to the hot table, and
the archive gets its own retention schedule. Storage matters either way: ~16k rows/day carrying
raw JSON is 1–2 GB/month against an 8 GB allowance, so the 14-day raw-stripping job ships in the
*same commit*, or the fix under pressure will be to restore the delete and destroy the corpus a
second time.

**Cost history cannot be recovered at any price.** `worker_usage` is a daily rollup keyed
`(source, utc_date)`. Per-call rows cannot be exploded out of it. Cost-per-decision starts at
zero on the day the new ledger ships. State it rather than discover it.

### The cutover test, as a measurement

**Agreement with the old system is a failure, not a pass.** This is the easiest mistake to make,
because agreement is the easiest query to write. The old grouper produces 91% singletons. The
old matcher resolves 189 of 323 tickers to stories sharing zero content words. A new grouper
that agrees with the old one has reproduced the defect.

So every gate is defined against ground truth or a compression target — except at GROUP and
RESOLVE, where the incumbent is deliberately run over the *same* gold set as a labelled control
row in the same result set, so it is a baseline to beat rather than a target to match.

| Stage | Gate | Window | Set by |
|---|---|---|---|
| TRACK | snapshot coverage(next) ≥ 0.98 × coverage(legacy), **and** zero 15-minute gaps | 7 days | instant labels |
| GROUP | precision ≥ 0.90, recall > legacy on identical pairs, singleton rate < 60% | 3–7 days | gold pair set |
| RESOLVE | abstain rate **>** legacy (more abstention wins), precision ≥ 0.95 on adjudicated non-abstains | 14 days | frozen regression set |
| ADMIT | recall ≥ legacy at equal alert budget, cost/admitted ≤ legacy, holdout populated ≥ 14 days | 14 days | 6-hour auxiliary label |
| RANK | Kendall tau ≥ 0.90 between consecutive 20-second ticks | 3 days | instant |

**Window length is set by label horizon, not by comfort.** Gold and regression sets have instant
labels: 3–7 days. The 6-hour auxiliary label needs 14 days. The 30-day peak label **cannot be
validated before cutover at all.** Accept it, cut over on the auxiliary, and leave the shadow
lane running afterwards so the peak-label comparison arrives 30 days later and can still force a
rollback. This is the one place the plan knowingly ships on incomplete evidence, and saying so
is better than discovering it.

**Gates live in the watchdog, not on a dashboard.** Two people build dashboards and then do not
read them — the same failure that left the budget guard dead in production for two weeks. A gate
that stops passing must page someone. A gate on a page someone has to visit is not a gate.

### One compatibility view that removes the riskiest step

`site/live.js` and `feed.js` keep reading `public.narratives` unchanged, because the table
becomes a view over the new store. `img_seed` and `search_series` become constants — which is
correct anyway, since the fabricated sparklines and stock photos are being left behind. This
defers the most user-visible step, saves about three days, and is scaffolding with a deletion
date rather than a permanent layer.

---

## 9. The honest timeline

The architecture document's build order totals **43 developer-days** for steps 0–8 plus 10. That
figure is roughly right for the code in isolation and roughly **half the calendar**, because it
contains no transition overhead — no backfills, no cutover switches, no comparison harness, no
replay fixtures, no watchdog — and implicitly assumes a developer who is not also keeping a live
system alive.

| | Days |
|---|---|
| Coding, re-estimated with transition included | 50.5 |
| Backfill migrations + verification | 3 |
| Cutover switches, shadow plumbing, rollback | 3 |
| Parallel-run comparison queries + one alerting gate | 3 |
| Keeping the OLD system alive, ~0.5 d/week × 16 weeks | 8 |
| **Total** | **~67 developer-days** |

One developer with a live production system and a non-engineer reviewer does not get five
productive days a week. At 3.75: **67 / 3.75 ≈ 17.9 weeks — call it 17 to 18 weeks, about four
months.** The eight-to-nine week figure is off by roughly 2×.

What makes that tolerable is the shape, not the total:

| | |
|---|---|
| **Week 1** | Perishable data alarmed and protected. Watchdog live, `recordLastRun` fixed, near-miss archive shipping, four measured cost-constant fixes |
| **Week 3** | ★ **The point of no regret.** All five perishable streams recording. Three of them are not recording at all today — the decision log, the mint/asset series, the 14-day carrier-frequency buckets — and a fourth has been deleting itself every twelve hours. After week 3, a slip from week 9 to week 18 costs **time, not the asset** |
| **Week 6** | X adapter and admit shadowing on live data |
| **Week 9** | Tracker cut over, resolver correct — the Buy button becomes trustworthy |
| **Week 13** | Storyteller with real carriers |
| **Week 18** | App on the new store, replay working, TikTok added — the exam passed |

**Not in any of these numbers: the models.** That step is calendar-bound rather than
effort-bound. QUALIFY's model needs ~2,000 labelled stories; ADMIT's needs ~5,000 auxiliary
positives plus 14 consecutive winning days. Month 5 at the earliest, and only if the decision
log ships in week 2. That is the single strongest argument for putting the decision log first:
every week it slips moves the first model by a week, and nothing recovers it.

**What makes it worse:**
- A TypeScript decision made mid-build rather than at step 0. A half-migration is the worst
  outcome and it is not in any estimate here. **This decision must land before step 0 ships.**
- PDF and design deliverables built in parallel by the same person.
- The old system needing more than half a day a week. The record — a seven-hour silent outage,
  a heartbeat dead for two weeks, self-poisoning floors — says half a day is optimistic. The
  week-1 watchdog is what makes this *knowable* rather than guessed.
- Removing the English-language filter widens the funnel and therefore the bill. If the stale
  cost constants are still in place, the budget guard pauses scoring at roughly a third of
  affordable throughput and it will look like a bug in the new system. Fix the constants in the
  same commit. (And note the filter appears **twice** — the keyword lane and the media lane —
  so removing one leaves half the funnel closed while looking fixed.)

**What compresses it:**
- The compatibility view: −3 days, and it removes the riskiest step.
- Deferring tier-2 embeddings until after TikTok: −4 days. If tier-1 carriers alone drop the
  singleton rate below 60%, the encoder can wait — and carriers are free, deterministic,
  language-blind and platform-blind.
- **Do not cut the TikTok step.** It is the exam. Cutting it means never finding out whether the
  architecture works, three days from the end.

The one thing that will be skipped under pressure is week 1, because it produces nothing a
founder can look at. Counter it by making the week-1 deliverable a demo: induce an outage
deliberately and watch the alert fire.

---

## 10. What we are deliberately not doing

Each of these is the correct answer for a larger team and the wrong answer for this one. Named,
so nobody re-proposes them as improvements.

**Turborepo or Nx.** The mainstream 2026 default for a TypeScript monorepo, and Nx's tag-based
module boundaries could express this dependency rule directly. Both are build-graph
orchestrators, and with native type stripping **there is no build to cache** — the full check
runs in seconds across nine small packages. Nx's boundary enforcement is a genuine loss and the
one real argument in its favour, but it enforces at lint time, which is strictly weaker than
pnpm enforcing at install and runtime. Correct at thirty engineers; a second system to keep
alive at two.

**Durable execution — Inngest, Temporal, Restate, Trigger.dev.** This is the option that sounds
most responsible and is priced out decisively. Inngest bills per execution, where one run plus
five steps is six executions; modelled honestly this pipeline is ~1.2M executions/month, which
is Pro at $75 plus ~$60 overage — **more than every other infrastructure line combined, to
schedule six timers.** Temporal Cloud starts at $100/month plus $50/million actions, and its
real cost is that somebody must understand determinism constraints and workflow versioning.
There is no such person.

**A job queue — BullMQ with Redis, or pg-boss.** BullMQ means a second stateful system to back
up, monitor and upgrade, at 1,200 admitted posts a day. pg-boss avoids Redis and still adds a
scheduler, a job table and dead-letter semantics for work that is six timers. The one workload
with a genuine per-item timer shape — the re-read schedule — is better served by a
`next_observe_at` column drained with `FOR UPDATE SKIP LOCKED`, because then the schedule is
*data you can `SELECT`* rather than scheduler state you have to trust. That property is what the
2% unconditional holdout reservation actually needs: an `ORDER BY` clause, not a queue priority.

**A separate time-series or analytics store — Timescale, ClickHouse.** 55M rows/year is two
inserts per second. Daily partitioning plus BRIN plus `DROP PARTITION` gets the same answers,
and a second store makes the training join a cross-database problem instead of a `JOIN`. The
escape hatch already exists and is free: monthly Parquet export of aged partitions, queried
with DuckDB on a laptop.

**A vector database — Pinecone, Weaviate, Faiss.** pgvector 0.8 in the Postgres already running
does everything the carrier search needs, including Hamming distance over bit vectors with an
HNSW index. This is arguably the single highest-leverage infrastructure decision available,
because it is what makes a replication detector operable by two people at all.

**A streaming chain subscriber — Geyser, LaserStream.** $499/month for sub-second mint
detection, on a window whose median post-to-peak is six days. Plus reconnect logic, a durable
cursor, gap backfill and permanent pager surface. Three times the entire infrastructure budget
for latency the product cannot use.

**ONNX Runtime in production.** 258 MB of native binaries — it alone exceeds Vercel's function
size limit — to evaluate a tree ensemble that 60 dependency-free lines evaluate in microseconds.
It is the right tool for exactly one job, running a neural encoder locally, and that is the job
being deliberately not done.

**A Python inference service.** It would break the guarantee that `decide()` is synchronous and
pure, which is the mechanism that forecloses leakage. It is 100× to 5,000× slower than
in-process. It doubles the always-on production surface. And it solves a divergence risk that
measurement shows is 10⁻¹⁶.

**A monorepo build orchestrator, a feature-flag service, an APM vendor, a secrets manager, a
staging environment.** Each is correct practice. Each is a vendor, a bill, a login, an upgrade
path and a thing that can be down, for a team of two who need the number of systems that can
page them at 3am to be as close to one as possible.

**Rust and Go.** Both would produce a more correct system. Both fail the operability test the
same way: a language a non-engineer founder cannot review and a fast-and-loose developer cannot
ship in is the wrong language regardless of technical merit.

---

## What must be true before the first commit

Five things, because they are the ones that get expensive if decided later:

1. **TypeScript is decided now, not in week six.** A half-migration is the worst outcome, and it
   is in no estimate in section 9. If the developer, after one afternoon of pairing on
   `adapters/src/platform/x/index.ts`, is visibly slower, fall back to JSDoc-checked `.js` — the
   tsconfig, the CI chain, the pnpm layout, the boundary rules and the purity checks are all
   unchanged; only `allowJs`/`checkJs` is added and the extension differs. That makes this
   genuinely reversible in an hour, which is why it is safe to commit to now.
2. **pnpm, with `packageManager` pinned and an `only-allow` preinstall guard.** One habitual
   `npm install` silently restores hoisting and removes the strongest boundary, with no error
   and green CI.
3. **`erasableSyntaxOnly`, `verbatimModuleSyntax`, `isolatedModules` in `tsconfig.base.json`,
   and `enum` banned by convention on day one.** The `as const` object plus union type is the
   house pattern, written down before someone reaches for an `enum` in the second-chain adapter.
4. **`imports` subpaths, not path aliases.** Node ignores tsconfig, so `@/lib/thing` does not
   exist. This is a repo-structure decision forced by the runtime.
5. **The watchdog before anything else.** Everything else in this document assumes the old
   system keeps collecting, and today there is no way to know when it stops.
