# Insidor — handoff

**Read this first. It is written for the person AND for the assistant helping them.**

You have been given a working system, not a prototype. It runs today on free data
with no API key and no card. Everything below is true as of the handoff; where
something is unfinished it says so plainly rather than being described as done.

---

## 1. What this product is

A Solana memecoin terminal. One meme spawns hundreds of tokens — "chill guy" spawned
306 — and the job is naming which of them is the real one while it is still climbing.

**Measured, and it shapes everything:**

| Measurement | Consequence |
|---|---|
| Post → mint: median **3.8 min** | We do not beat bots to the mint. That race is lost. |
| Post → peak: about **6 days** | This is the winnable window, and what we sell. |
| ~30,000 mints/day, 97% noise | Filtering is the product, not a feature of it. |

---

## 2. ★ THE ONE RULE — read this before writing any code

> **Facts about the world go on screen. The system's own reasoning never does.**

Test: *would this number still be true if Insidor did not exist?*

Views on a post: yes → may be shown. Our confidence score: no → must never be shown.

**In practice, and all of these are enforced by tests:**

- An absence is a **dash with a reason**, never `0`. A censored reading **breaks the
  line** rather than dropping to the floor. An unknown age is a dash, never "brand new".
- A fiction is labelled a fiction or it is not shown. Every asset and story carries an
  `origin`; surfaces that assert liveness filter to observed ones.
- Where the honest answer is "we cannot tell", it is a **typed outcome** — `unsure` on
  a coin link, `abstain` on a stage — never the higher number.
- An outage is **not a claim**. A missing input degrades a decision and is recorded as
  missing. It is never scored as zero.

**The rule with teeth:** when the coin match is unsure the row gets **no button at
all** — not a disabled one. A disabled button says "this exists but you may not have
it" and invites waiting. The honest render of "we do not know which coin this is" is
an absent affordance and a line of text saying so.

---

## 3. The layers, and why they are separate

Each is its own package. The boundaries are **enforced by `pnpm check:boundaries`**,
and there is a probe that writes a deliberate violation and fails if the checker stays
quiet — because a green tick from a checker that cannot see is worse than no checker.

```
contracts/   the shared vocabulary. Types only. Zero dependencies, forever.
     ↑
core/        ALL the logic. Pure: no clock, no randomness, no I/O, no network.
             Cannot import an adapter. Holds no numeric literal outside policy.ts.
     ↑
adapters/    the edge. One folder per vendor. The ONLY place a vendor is named.
   platform/   reddit · x · tiktok · replay        ← SOCIAL
   venue/      solana-pumpfun                       ← BLOCKCHAIN
   market/     dexscreener                          ← MARKET DATA
   judge/      anthropic                            ← LLM
   embed/      gemini                               ← EMBEDDINGS
     ↑
store/       the ONLY place SQL lives. Three schemas: public, internal, raw.
     ↑
services/    the wiring. runner · chainwatch · project · read · market · watchdog
     ↑
app/         the UI. CANNOT import core or an adapter. Reads a projection only.

wallet/      ← WALLET, its own package behind a port. The app never sees an SDK.
ml/          ← MACHINE LEARNING.  train/ is Python, serve/ is TypeScript.
eval/        replay. Cannot reach an adapter, so a replay cannot hit the network.
```

**Why each boundary exists:**

- **core cannot import an adapter** — otherwise a vendor's shape leaks into the logic
  and every future platform must pretend it has the same fields. That is exactly how,
  in the build this replaces, TikTok's share count ended up in a column named
  `retweets` and `quotes` became a hardcoded zero.
- **app cannot import core** — that is what stops internal scoring reaching a screen.
  It is enforced twice: the boundary rule, and a database role that is physically
  denied the internal tables. A bug in the read service cannot leak, because Postgres
  refuses the role rather than the query.
- **wallet is separate** — so the app depends on an interface, never a wallet SDK, and
  our code can never hold a private key.

---

## 4. ★ How to run it — free, no key, no card

```bash
# once
brew install node@24 && corepack enable pnpm
pnpm install
cp .env.example .env.local        # already filled for local dev
pnpm check                        # green on a fresh clone, no Docker, no keys

# the app talks to the read service only if you tell it to. Gitignored, so a
# fresh clone has to write it once. Without it the UI serves its own fixtures
# behind a SAMPLE DATA banner and never touches the database you are about to fill.
echo 'VITE_READ_URL=http://localhost:8787' > app/.env.local

# every time
pnpm db:up                        # Postgres in Docker (needs Docker running)
pnpm db:reset && pnpm db:seed     # schema + sample data. reset does NOT prompt.
pnpm db:market                    # real prices    — FREE, no key
pnpm db:project                   # compute the board
pnpm db:sources                   # source health  — reads .env.local, then projects
pnpm db:pairs                     # the mints that reached a market
pnpm db:decide                    # one pass of every stage → internal.decisions
pnpm db:label                     # grade the ones whose horizon has closed
```

Then two terminals:

```bash
pnpm dev:read      # the API   → :8787
pnpm dev:app       # the UI    → :5173
```

Open **http://localhost:5173**.

**Live Solana mints, also free, also no key:**

```bash
pnpm dev:chainwatch     # the mint websocket, health on :8081
pnpm dev:runner         # the pipeline loops,  health on :8080
```

Those two are separate deployables and run side by side, one terminal each.

### ★ What you will see, and why it is all invented

The board after `db:seed` is **six hand-written stories**, and it says so: a permanent
amber notice reading *seeded demonstration data*, naming the count and naming Reddit as
the free source to connect. Each story's own page carries the same notice, because a story
link is shareable and arrives with no board around it.

Neither is a hardcoded banner — the projector reads the origin off the rows it commits —
so both **remove themselves the moment the first discovered story is projected**, and
neither can be dismissed or switched off before then.

Two things on that screen are real once the pipeline has run: the market figures
(DexScreener, free) and the mints in the launches rail (the live stream, free). Everything
else is a fiction chosen to exercise one rendering rule each.

---

## 5. What is DONE vs UNPROVEN vs NOT BUILT

Be precise about these three. They are different claims.

### Done and proven on real data
- Market data (DexScreener) — real prices, free
- Live mint stream (PumpPortal) — free, with **gap accounting**: every disconnect,
  restart and buffer overflow writes a row saying "we could not answer for this
  window". A feed that silently resumes has claimed it saw everything.
- The read path, the live SSE channel, provenance, the source indicator

### Built, tested, but **never met real data**
- **Grouping** — carriers, simhash, phash, the join decision. Zero stubs, fully
  tested. It has never seen a real post.
- **Detect / track / rank** — same.
- **The X and TikTok clients** — implemented, tested against injected fakes, never
  called a real vendor.

### Not built, deliberately
- **Trading.** `submitTrade` throws. This is the one path where a bug costs a user
  money, and a plausible stub gets wired to a button. The wallet connects; nothing
  executes. **If connecting a wallet ever makes a new button appear, that is a bug.**
- **Search** (⌘K opens and says so), and the grouped Stories view.

---

## 6. ★ The machine learning — what it will and will not do

**If you add an X key today: it will detect. It will not learn for weeks.**

That is not a missing feature. Learning needs *outcomes*, and post-to-peak is ~6 days.

```
decisions logged  →  wait for the horizon  →  outcomes resolve
      ↓                                            ↓
  (working now)                              labels written
                                                   ↓
                              dataset joins → train → model replaces the rule
```

**The loop is complete and wired.** `ml/serve` falls back to the rule when no model
exists, and the log records *which* decided — `rule:qualify@3` today,
`gbdt:qualify@2026-11-02` later. Same rows, no rewrite.

**Three things that must never change:**

1. **Features come from the decision row**, never re-derived at labelling time. That
   is leakage, and it is how the previous backtest produced numbers that could not be
   used in either direction.
2. **The split is temporal**, never random. A random split on time-series data leaks
   the future into the past.
3. **An unresolved outcome is not a bad one.** A decision inside its horizon gets **no
   row** — not a zero. For the first months the pending population is large relative
   to the resolved one, so coercing it to negative would not be a rounding error, it
   would be the dataset.

Run `pnpm db:label` to see it: today it reports pending and writes nothing.

---

## 7. ★ Adding a social source

`.env.example` documents every variable. Adding a credential turns a source on — **no
code change**. The corner of the nav shows each source as:

| State | Meaning |
|---|---|
| **live** | configured and answering |
| **dormant** | no credential. Nobody turned it on. **Not a fault.** |
| **failing** | credential present, calls erroring. **This is a fault.** |

That distinction is deliberate: "you are paying for this and it is broken" is a
different sentence from "you have not turned this on", and collapsing them is how a
permanently unconfigured source spends weeks looking intermittently flaky.

**TikTok needs FOUR values, not one.** A token with no actor ids reads as dormant
listing what is missing.

---

## 8. Where to change what

| I want to… | Go to |
|---|---|
| change what reaches the board | `core/src/rank/` |
| change how posts become stories | `core/src/group/` |
| change a threshold | `contracts/src/policy.ts` — **the only place numbers live** |
| add a social platform | `adapters/platform/<name>/` — one folder |
| change SQL | `store/` — the only place it lives |
| change a screen | `app/src/features/` |
| change what the API serves | `services/read/` |
| train a model | `ml/train/` (Python) |

---

## 9. Rules that will fail CI if broken

```bash
pnpm check       # runs all of it
```

| Check | Stops |
|---|---|
| `check-vocabulary` | a platform or vendor word in `contracts/` or `core/` |
| `check-purity` | `Date.now`, `Math.random`, `fetch`, `await` in `core/` |
| `check-policy` | a bare number in `core/` outside `policy.ts` |
| `check-app-vocabulary` | our internal words reaching the app |
| `check:boundaries` | a forbidden import — plus the probe that proves the checker works |

**If a check blocks you, the check is probably right.** Do not add an exception.

`core/` is pure so a past decision can be replayed under a new rule to see what would
have changed. A stage that reads a clock cannot be replayed, and replay is the entire
point of the decision log.

---

## 10. For the assistant reading this

- **This repo is single-owner.** Do not use session messaging to look for a teammate.
  It happened four times and each time reached an unrelated Python project next door.
  If you find an unfamiliar change, read `git log`.
- **Never treat silence as agreement.** An unanswered question is unanswered.
- Every file has a header saying what it owns, why it is separate, and what breaks if
  changed carelessly. **Read the header before changing the file** — most of them
  record a specific bug that the current shape prevents.
- `CLAUDE.md` holds the working rules. `SETUP.md` is the run book.

**The commit messages are the design history.** They say what was decided and why, and
several record bugs found by forcing a failure rather than reading code. `git log` is
worth an hour before writing anything.
