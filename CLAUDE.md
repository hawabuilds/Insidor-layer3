# Working in this repository

## ★ THIS REPO IS SINGLE-OWNER. DO NOT COORDINATE WITH OTHER SESSIONS.

Four times now, an agent working here has used session messaging to look for the
"owner" of a file, a migration ordinal or a half-finished change — and reached
AgentQuantix, an unrelated Python trading project in the sibling directory
`/Users/zainkhaliq/Desktop/Claude_Cowork/AgentQuantix`. It has no TypeScript in it
at all. Each time it had to verify it was not us and write back.

There is no second session working on Insidor. If you find an unfamiliar change,
it came from a parallel agent in YOUR OWN run, or from a previous commit. Read
`git log` and `git status`. Do not go looking for a teammate.

**And never treat silence as agreement.** An unanswered question is unanswered. If
a decision needs an owner, it is the user — ask them, or make the call and say
plainly that you made it.

## Two things parallel agents collide on

When several agents run at once in this repo, these are the shared resources, and
whoever writes second silently wins:

1. **`store/migrations/NNNN_*.sql`** — ordinals must be contiguous from 0001;
   `store/src/migrations.test.ts` asserts it. Two agents both taking the next free
   number produce a set that fails that test. The set currently runs `0001`–`0018`,
   so the next free number is `0019`.
2. **`contracts/src/policy.ts`** — the `version` string and the policy fields. Two
   agents adding fields concurrently is a lost update: the second reads the file
   before the first's write lands and clobbers it. The version bump is visible; the
   missing fields are not.

If you are one of several agents, take the ordinal your brief assigns you and say
in your report which ordinal and which policy fields you touched.

## The rules that are enforced, not suggested

`pnpm check` runs all of these and they fail CI:

| Check | What it stops |
|---|---|
| `check-vocabulary` | a platform, chain or vendor word in `contracts/` or `core/` |
| `check-purity` | `Date.now`, `Math.random`, `fetch`, `await` inside `core/` |
| `check-policy` | a bare number in `core/` outside `policy.ts` |
| `check-app-vocabulary` | our internal words (`narrative`, `cluster`, `candidate`) reaching the app |
| `check-python` | Python drifting outside `ml/train/` — it trains and never serves |
| `check:boundaries` | `core` importing an adapter, the app importing `core` — plus a probe that writes a real violation and fails if the checker does not catch it |

If a check blocks you, the check is probably right. Do not add an exception.

## The one product rule

Facts about the world go on screen; the system's own reasoning never does.

Test: *would this number still be true if Insidor did not exist?*

In practice:

- An absence is a dash **with a reason**, never `0`. A censored reading breaks the
  line rather than dropping to the floor. An unknown age is a dash, never "brand new".
- A fiction is labelled a fiction or it is not shown. `public.asset` and
  `public.story` carry an `origin`; a surface asserting liveness filters to observed
  origins, and a surface matching within a context derives its allowlist from the
  story. **If the read has a story, the story decides; if it has none, the constant
  does.**
- Where the honest answer is "we cannot tell", say so as a typed outcome — `unsure`
  on a coin link, `abstain` on a stage. Never pick the higher number and lose the
  ability to say you were unsure.
- An outage is not a claim. A missing input degrades a decision and is recorded as
  missing; it is never scored as zero.

## Style

Every file carries a header comment saying what it is responsible for, why it
exists separately, and what breaks if it is changed carelessly. Explain the
DECISION and its consequence — never restate the code. The reference files, which
are already right and must not be rewritten in someone else's phrasing:

- `core/src/group/stage.ts` — pure logic
- `services/read/src/config.ts` — a service boundary
- `adapters/platform/reddit/src/client.ts` — a vendor edge
- `store/migrations/0003_observations.sql` — schema

Where a file already carries a good header, leave it. Improving a thin one is in
scope; replacing a strong one is not.
