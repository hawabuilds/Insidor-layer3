# Your meme score vs Mercury's judge

A side-by-side of the meme scoring in the old repo (`hawabuilds/Insidor-layer3`) and the
coinability judge in this one, written before anything is changed.

Sources read, old repo: `worker/adapters/anthropic/meme-score.js`, `worker/score/score.js`,
`worker/score/lib/` (`meme-gate.js`, `nameability.js`, `tt-score-gate.js`,
`coin-launch-check.js`), `worker/score/rescore-validate.js`, `docs/rescore-threshold-analysis.md`,
`docs/rescore-validation-report.md`, `backtest/backtest-merged-reveal-report.md`.

This repo: `adapters/judge/anthropic/` (all three `prompts/*.txt` and the source),
`core/src/qualify/`, `core/src/admit/`, the `qualify` and `admit` sections of
`contracts/src/policy.ts`, and `contracts/src/judgement.ts`.

---

## The short version

- **News is the big gap.** Your prompt tells the model, in so many words, that headlines,
  match results, policy debates, industry arguments and transfer news are not coinable, and it
  gives worked pairs. Mercury's prompts never use the words news, politics, sport or result.
  The closest they get is one line: "the posts are reactions to an event rather than to a
  subject".
- **Your fandom rule is missing**, and so are your worked pairs (Biden, Messi, therapy, K-pop)
  and your "one word that works as a ticker" rule.
- **Your nameability cap is already covered, and more strictly**, by deterministic gates in
  `core/src/qualify/`. These run after the model, and the model can't argue with them.
- **Your TikTok no-thumbnail cap (0.4) clashes with a rule in this repo**: a missing input
  has to stay missing and is never scored. I recommend not porting it (see the decisions at
  the end).
- **Your 0.70 cutoff is real evidence, but weaker than "73% vs 2%" sounds.** The backtest
  matched coins by name, loosely, using the model's own suggested ticker. That inflates the
  gap in a way we can't measure now, because the validation cache was never committed.
- **One fact about this repo matters for every step after this:** Mercury's judge has never
  called a model. `httpClient` in `adapters/judge/anthropic/src/client.ts` throws
  `NotImplemented`, so these prompts have never been sent to a model and there are no
  recorded responses to test against.

---

## 1. How each version filters out news

### Your prompt (`COINABILITY_CORE` in `meme-score.js`)

News is named directly, as the "STORY" half of the core test:

> STORY (not coinable — score low): an event, argument, opinion, or news narrative with no
> deployable subject as the focus. Industry takes, policy debates, match RESULTS, award
> snubs, transfer news as headline — not the person/group as meme.

Then it teaches the difference with pairs:

> - "Messi [as meme/character]" → named person → HIGH
> - "Messi's team lost the final" → sports RESULT, not Messi-as-subject → LOW
> - "Therapy industry criticized" / "K-pop industry faces criticism" → argument about industry → LOW
> - "Therapy industry criticized as ineffective" → an argument → LOW
> - "Biden tapes about classified documents" → NEWS EVENT focus → LOW; Biden memed AS character → HIGH

And it closes with a hard rule:

> - Story/topic-only (no named subject focus): meme_score below 0.3
> - If you cannot name the coinable subject in one word that would work as a ticker, score below 0.3

### Mercury's prompts

None of the three files mention news, headlines, politics, sport, results, products or
industry. These are the lines that come closest. From `coinability-core.txt`:

> Answer "no subject" rather than stretching, in each of these cases:
> ...
> - the posts are about a broad topic rather than a specific thing
> ...
> - the posts are reactions to an event rather than to a subject

From `system.txt`:

> 3. If the posts are about several different things, that is not one subject. Say so.

These would catch some news by accident ("reactions to an event"). They would not stop
"Apple and LG leak new HomeKit features", because Apple is a specific thing the posts are
all about. Your prompt stops it, because the post is a product headline rather than Apple as
a meme.

### Your old false positives show where your own rule leaks

Your v2 validation report lists posts that scored 0.85 and never got a coin. Several are
announcements that the fandom rule waved through:

> | 0.85 | 0.6 | God Of War Laufey Launches February 16, 2027 | LAUFEY | Named artist (Laufey) as the subject of a major project announcement ...

That is exactly the "X unveils single" case you want tightened in step 2. Your fandom rule
says *"Stray Kids win award" / "Stray Kids unveil single" → group is the focus → HIGH*,
which is the opposite of what step 2 asks for. So those two example lines can't be copied
across as written. They need to become a pair: the group as the meme → coinable, the
announcement as a press release → not.

---

## 2. What each prompt asks, and which of your rules Mercury already has

### What the model is asked

| | Your prompt | Mercury's prompts |
|---|---|---|
| Unit judged | One post: the narrative's top post by views (`rescore-validate.js`) | A group of posts (a story of at least 3 posts from at least 2 accounts) |
| Question | "Is there a THING someone could put on a coin, or is it just a topic?" | "Is there one subject? What would people call it? How sure are you?" |
| Output | `meme_score` 0–1, `reason`, `suggested_ticker`, `suggested_name` | `coinable` yes/no, `confidence` 0–1, `proposedName`, plus `alternateNames` and `subjectSpans` |
| TikTok | A separate system prompt; judged from the video thumbnail | No per-platform prompt; `JudgeSubject` carries text and media refs |
| Views shown to model | No | No, and the prompt says why: "attention is measured elsewhere" |

### Rule by rule

| Your rule | In Mercury? | Notes |
|---|---|---|
| SUBJECT vs STORY, with news as the STORY side | **Missing** | ⚠ Same words, different meaning. In Mercury's `subject-vs-story.txt`, "story" means *the set of posts* ("A STORY is the set of posts. A SUBJECT is what they are about"). In yours, "story" means *an event or news narrative*. Pasting your wording in as-is would give one file two meanings for one word. I'd port it as "news, events and arguments" and leave his "story" alone. |
| Worked news contrasts (Biden, Messi, therapy, K-pop) | **Missing** | His file already teaches with worked contrasts (the cat, the sentence, the dance, the game character), so the pairs fit his style well. |
| Product/tech news contrast (Apple/LG HomeKit) | Missing in both | New in step 2. |
| Named people and fandoms are subjects | **Missing**, partly the other way round | His dance example says the subject is the originator "only if the posts are about that person rather than merely using their sound". That is close to your "person as the focus" idea, but nothing says a named idol, athlete or streamer counts. |
| Do not downgrade because the post describes something they did | **Missing** | Step 2 narrows this: an announcement or award headline is not enough on its own. |
| Ticker = the person's or group's name (JUNGKOOK, STRAY, RM) | Partly | He asks for "the shortest phrase that people in the posts themselves use". Close, but not ticker-shaped. |
| "Can't name it in one word that works as a ticker → not coinable" | **Missing, and clashes** | ⚠ His own examples are multi-word names: "green jacket guy", "chill guy". A strict one-word rule would make "chill guy" not coinable. See decision A. |
| Nameability cap at 0.3 (generic or sentence-like ticker) | **Covered, more strictly** | In `core/src/qualify/stage.ts`: no name → `Q4_unnameable`, a generic name → `Q5_generic_name`, too short → `Q6`, not anchored in the posts' own words → `Q7_name_unspecific`. These DROP, rather than capping at 0.3. On top of that, `nameabilityCap()` in `nameability.ts` caps the score at the specificity, with a floor of 0.4, and at 0 for a generic name. All of it runs after the model. |
| Generic ticker list | Partly | Yours has news-flavoured words his doesn't: `NEWS`, `BREAKING`, `UPDATE`, `LIVE`, `LATEST`, `VIRAL`, `TRENDING`, `ALERT`. His (`generic-tickers.ts`) has crypto majors and slang. Adding the news words to his list would be a small, cheap extra wall against news. Not in the plan; tell me if you want it. |
| TikTok no-thumbnail cap at 0.4 | **Missing, and conflicts** | See decision C. |
| Format alone (dance, haul, tutorial) isn't a subject | **Present** | His dance and template rules cover this well. |
| Cross-platform narratives get a lower bar (−0.1) | Not in his code | It lived in your `memeMinForNarrative`. Step 3 doesn't ask for it; I'm mentioning it so it isn't lost. |
| A single account posting repeatedly isn't a subject | His only | "the group is a single author posting repeatedly". He also gates it in code (`Q2_single_author`). |
| Generic common words aren't names | Both | His `system.txt` rule 4. |

---

## 3. How your 0–1 score maps to his judge output

Your `meme_score` is one number doing two jobs: *is there a coinable subject* and *how sure
the model is*. His judge splits these into a yes/no and a confidence.

| Your output | His output | How it lines up |
|---|---|---|
| `meme_score` ≥ 0.7 ("clear nameable subject") | `coinable: true`, `confidence` high | The main match. |
| `meme_score` ≤ 0.3 ("story/event/argument") | `coinable: false`, `confidence` high | ⚠ His confidence is how sure the model is of *its answer*, yes or no: "A number between 0 and 1, given only these posts." A confident NO also has high confidence. |
| `meme_score` 0.4–0.6 ("borderline") | either answer, `confidence` low | |
| `suggested_ticker` | `proposedName` | His is a name, not a ticker. Core then checks it (generic, length, anchored in the posts). |
| `suggested_name` | `proposedName` / `alternateNames` | |
| `reason` | `rationaleRef` | His prose is never stored inline and never shown, only a blob key. |
| the 0.3 cap applied after the model | the Q4–Q7 gates and `nameabilityCap()` | Same idea, done in pure core code. |

**What this means for step 3.** A cutoff of "confidence ≥ 0.70" only means what your cutoff
meant when the answer is `coinable: true`. On a `false` answer it would wave through
confident NOs, which Q9 drops anyway. The new gate should read "coinable and confidence at
least the bar for this source".

Two more things to know before reusing 0.70:

- `contracts/src/judgement.ts` says of `confidence`: *"Self-reported and therefore never a
  gate on its own."* Step 3 adds a gate on it. That's fine as long as it sits next to the
  other gates rather than replacing them, but it goes against a line Mercury wrote on
  purpose. The comment should be updated in the same commit, so the contract doesn't
  contradict the code.
- Your 0.70 was measured on **your** scale, from **your** prompt, judging **one post**. His
  judge gives a different number, from different prompts, over **a group of posts**. The
  cutoff is a sensible starting point, not a transferred calibration. It needs measuring
  again once his judge actually runs.

---

## 4. What your backtest found

### The threshold analysis (`docs/rescore-threshold-analysis.md`)

629 narratives, scored with your v2 prompt on 2026-08-03:

| | Narratives | Coined | Rate |
|---|---:|---:|---:|
| Score ≥ 0.70 | 207 | 151 | **72.9%** |
| Score < 0.70 | 422 | 8 | **1.9%** |

- Scores bunch up: nothing landed between 0.5 and 0.6, and nothing at 0.9 or above. Almost
  every pass is 0.7–0.85.
- Inside the pass range the coined rate is flat, about 71–74%. Raising the bar from 0.70 to
  0.80 drops 76 passes (37%) for +0.1 points. So 0.70 is the elbow, and going higher only
  costs volume.
- **TikTok was never calibrated.** The 82 TikTok-only narratives mostly scored exactly 0.4,
  which is the no-thumbnail cap, not a real judgement. Your own note: *"Keep MEME_MIN_TT=0.75
  until TT vision rescores are in the validation set."* So 0.75 is a placeholder.

### The validation report (`docs/rescore-validation-report.md`)

- Fandom: 64 of 88 fandom-tagged narratives passed. Fandom misses (failed but coined) went
  from 3 (v1) to 1 (v2): *Real Madrid Closed Training Match*, which is a sports headline
  where a coin named after the club existed anyway.
- The 10 listed false positives (scored 0.85, no coin found) are mostly fandom and
  announcement posts: Yoongi, RM, Laufey, Frisk, Zukka.

### The merged reveal report (`backtest/backtest-merged-reveal-report.md`)

A separate study: 866 coins, half winners and half losers, looking back for their origin.
Only 79 had a viral post behind them. Your report says it is *"Descriptive only — not
statistically significant."* The largest gaps between winners and losers were "ticker in
replies" (+25.7 points, n=17/22), "crypto noticed" (+21.3) and "others copying it" (+16.9).
"Has one-word name" and "has character" did *not* separate winners from losers. That says
nameability decides whether a coin gets *made*, not whether it does *well*.

---

## 5. What the backtest can and cannot prove

**It can show:**

- The v2 prompt separates "there is a nameable thing here" from "this is a topic" in a
  consistent way, and the line falls at 0.70.
- Above 0.70, a higher score doesn't mean a coin is more likely. So a stricter bar costs
  volume and buys nothing.

**It cannot show:**

1. **Whether a coin did well.** It measured "a token with a matching name launched after the
   post", nothing about price, volume or survival. The reveal report suggests nameability
   doesn't separate winners from losers.
2. **That the prompt predicts anything ahead of time.** Everything was scored after the
   fact, on posts whose outcomes already existed. The v2 prompt was written after looking
   at v1's misses on the same narratives, so the fandom improvement is partly tuning on the
   test set.
3. **That "coined" means "coined from this post".** The match is loose. Any Solana token
   launched after the post whose symbol equals the term, or whose *name contains* it, counts
   (`hitMatchesTerm` in `coin-launch-check.js`). The report's own misses show it:
   *Madrid FC* matched MADRID, *Villain* matched VILLA, *HOUSECAT* matched HOUSE, *Julian*
   matched JULIA. Famous names (MESSI, YOONGI, RM) almost always have some coin, so a high
   score on a celebrity post is close to an automatic "coined".
4. **How much of the 73% vs 2% gap is real.** The search terms come from the model's
   *own* `suggested_ticker` and `suggested_name`. A low score usually comes with no ticker or
   a generic one, since your nameability cap links the two, and then there is nothing to
   search for and the narrative counts as "not coined" automatically. Part of the gap is
   built into the method. How much can't be measured from here: the per-narrative results
   lived in `.backtest/rescore-validation-cache.json`, which was never committed.
5. **Anything about TikTok.**
6. **Anything about Mercury's judge.** Different prompt, different unit (one post vs a group),
   and a different output scale.

None of this makes 0.70 a bad starting point. It makes it a starting point to re-measure,
which is why step 3 puts the backtest's limits into the `policy.ts` comment.

---

## Decisions taken (2026-10-10)

| | Question | Decision |
|---|---|---|
| A | The one-word rule | "One word, or a few words run together like CHILLGUY." Ported into `coinability-core.txt` as THE NAME TEST. |
| B | The word "story" | Ported as "news, events and arguments", leaving Mercury's meaning of story alone. |
| C | TikTok no-thumbnail cap | Not ported. A video the judge can't see is never sent; it comes back `unavailable` and qualify abstains (`adapters/judge/anthropic/src/to-judgement.ts`, `lookedAtFor`). |
| D | Per-source bar | Keyed by what the judge looked at: `Policy.qualify.minJudgeConfidence` = text 0.70, videoFrame 0.75. 0.75 is marked not calibrated. Gate `Q10_judge_unsure` in `core/src/qualify/stage.ts`. |
| E | Tests without a live call | Prompt-text and fixture tests for now (`fixtures/news-contrasts.json`, `src/news-contrasts.test.ts`), with the answers marked hand-written. |

## Follow-ups (not part of this task)

1. **Record the news pairs live.** Once the judge client works, do one small recorded live run
   of every case in `adapters/judge/anthropic/fixtures/news-contrasts.json`. Store the real
   answers in place of the hand-written ones, set `answerSource` to `recorded`, and keep them
   as test fixtures. That is the test that catches a prompt rewording the model reads
   differently, which the current tests can't.

2. **The judge HTTP client is NotImplemented, so qualify will abstain on every story when the
   feed goes live. Implementing it, metered and within the budget, is the next task.**
