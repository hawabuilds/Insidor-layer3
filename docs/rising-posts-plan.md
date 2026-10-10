# Plan: an early "rising post" signal

**Status: a plan, not built.** Nothing in this document exists in code yet.

## Why

The lead feature will be notifications for viral posts. Today the earliest thing the app can
show is a qualified story, and QUALIFY needs at least 2 accounts and 3 posts
(`Policy.qualify.minDistinctAuthors`, `minMembers`). By then the first-alert moment has
passed. This plan shows a **single post** as "rising" as soon as the pipeline sees it
growing fast, before any grouping or judging, and with nothing on it a person could buy.

## What a rising post is

A single item that:

1. **ADMIT** let through (it is being tracked at all), and
2. **DETECT** passed with `D0_burst`: "atypical against its own trailing baseline".

DETECT is the right stage because it is already described as "the earliest alert the system
can make" (`core/src/detect/stage.ts`). It costs nothing, reads only numbers we already paid
for, sits behind a hard absolute floor (`D3_below_absolute_floor`) that a manufactured
baseline cannot open, and abstains rather than guessing when a baseline or a reading is
missing (`D1`, `D2`, `D4`). Those are the properties an alert needs.

**No new stage, no new judgement.** Rising is a *projection* of decisions that already
exist. Nothing new decides anything, so nothing new can be wrong in a way the decision log
doesn't already record.

A post **stops** being rising when either:
- its newest DETECT decision is no longer `D0_burst`, or the rising window has passed
  (a new policy field, `Policy.rising.windowMs`, so the number lives in `policy.ts`), or
- it becomes a member of a story QUALIFY passed. It has graduated into a narrative, and the
  two lists stay separate.

## What it shows: facts only

Each rising post carries exactly these fields, and nothing else:

| Field | What it is | Why it is a fact |
|---|---|---|
| `risingId` | a public id for the row | — |
| `sourceLabel` | "X", "TikTok", chosen by the server | where it was posted |
| `authorLabel` | the poster's handle | who posted it |
| `permalink` | link to the original post | where to check it |
| `excerpt` | the post's text, shortened | what it says |
| `thumbUrl` | the post's own picture, or null | what it shows |
| `postedAt` | `Instant` | when it was posted (unknown stays a dash) |
| `reach` | `Measured`, the latest view count | how many saw it (no view count → a dash with a reason, never 0) |
| `readAt` | `Instant` | when we read that view count, so the age of the figure travels with it |
| `spark` | the view readings over the window, with censored readings as `null` | the views line, which breaks at a gap instead of dropping to zero |

**Not on the wire, ever:** anything DETECT computed (the burst size, the averages, the
baselines). Those are our reasoning, not facts, and the app vocabulary check already bans
`burst` and `eta`. "Rising" itself is the only judgement shown, and it is shown as the name
of the list, not as a number.

## No coin link, no buy button

A rising post has **no `coins`, no `tradable`, no `marketCapUsd`, no `priceUsd` and no
`storyId`**. The missing `storyId` is deliberate: a link to the story page would lead to that
page's coin buttons, which is a coin link one tap away. If a post later graduates into a
qualified narrative, it leaves the rising list and the narrative appears in its own list,
with its own rules.

This is the same shape the launches rail already uses. `LAUNCH_FIELDS` in
`app/src/shared/api/wire/fields.ts` leaves out price and `tradable` "so no buy affordance can
be assembled from a launch at runtime even if a server started sending those keys".

## Where it lives

| Layer | Change |
|---|---|
| **Stage** | None. DETECT (`core/src/detect/`) already decides; ADMIT already admits. |
| **Policy** | New `rising` block in `contracts/src/policy.ts`: `windowMs` and `maxRows`. Numbers only there. |
| **Store** | The next free migration (0023 at the time of writing; `CLAUDE.md` names it). It adds `public.rising_view` (feed_id, tick, projected_at, source freshness), the same shape as `launch_view`, and `public.rising_row` (feed_id, item_id, position, payload jsonb), the same shape as `launch_row`. SELECT is granted to `insidor_app` on these two tables only. |
| **Item origin** | ⚠ **A gap to close first.** `public.item` has no `origin` column (`0016_story_origin.sql` says so). A rising list asserts that something is happening in the world, so it must show only observed items. The same migration adds `public.item.origin` (`observed` / `fixture`), set by the seed and the ingest paths, and the projector filters to `observed`, as the board does for stories. |
| **Projection** | `services/project` gains `projectRising` + `writeRising`. It reads the newest ADMIT and DETECT decisions per item from `internal.decisions` (the service role can read `internal`), drops items that are members of a QUALIFY-passed story, and builds each payload from `public.item`, `public.author` and `public.observation`. It writes the frame and the rows in one transaction, view row first, like `writeLaunches`. |
| **Read route** | `GET /rising/:feedId` in `services/read/src/routes.ts`. Two statements, no join, same as `launches`. The reply is `{ tick, source, rising: [payload…] }`, handed over verbatim. |
| **Apps (later)** | `wire/rising.ts` + `RISING_FIELDS` + `decodeRisingFeed` in both `app/` and `mobile/`. A "Rising" list on its own tab or section, never mixed into the board's rows. |
| **Alerts (later)** | `services/notify` reads new rows from the same projection by tick, so an alert can never say more than the list does. The alert text is built from payload fields only: "{sourceLabel} post by {authorLabel}: {reach} views in {age}". No coin words. Push sending, subscriptions and rate limits belong to that service and are not part of this plan. |

## Tests that prove it can never show a coin button

Each layer gets one, so a leak has to get past all of them:

1. **Store.** `rising_row` has exactly the columns (feed_id, item_id, position, payload),
   and `insidor_app` can SELECT these two tables and nothing it didn't have before.
2. **Projector.**
   - Every payload's keys equal `RISING_FIELDS` exactly.
   - Given an item whose story has a settled, tradable coin (`CoinLink` `one`), the payload
     still has no `coins`, `tradable`, `priceUsd`, `marketCapUsd` or `storyId`.
   - An item in a QUALIFY-passed story is not in the frame at all.
   - A `fixture`-origin item is never in the frame.
3. **Read route.** The `/rising` reply is the payload verbatim, and a leak test greps it for
   `coin`, `tradable`, `price`, `mint`, `storyId`, `burst` and `eta`, as `routes.test.ts`
   already does for `internal`.
4. **Decoder, in both apps.** A rising payload carrying `coins`, `tradable`, `priceUsd`,
   `marketCapUsd` or `storyId` throws `WireLeakError`. These are added to a rising-specific
   forbidden-key list, because they are legitimate keys on a board row and so are not in
   the global one. Unknown keys are dropped by the pick.
5. **Type.** `RisingPost` has no field that `coinStatus` (mobile) or `rowAction` (desktop)
   could accept. A `// @ts-expect-error` test passes a `RisingPost` to each and must fail to
   compile.
6. **Screen.** The rising list's component files import no buy, create or trade control,
   and contain no "Buy", "Create" or "$". A grep test like `mobile/src/screen-rules.test.ts`
   checks this. The row has no button at all.
7. **Alert, when built.** The notification body is built only from the payload, and a test
   asserts it contains no ticker, `$` or coin word, even for an item whose story has a coin.

## Separate from qualified narratives

- Its own projection, its own route, its own list. It never appears in `board_row` and never
  shares a tick with the board, for the reasons `0011_launches.sql` gives for launches.
- An item leaves the rising list when its story qualifies, so one thing is never shown as
  both a rising post and a narrative.
- The rising list makes no claim about coins either way. It doesn't say "no coin yet",
  because that would be a coin statement too.

## Before building, worth knowing

- DETECT, like GROUP and RANK, is "built, tested, but never met real data" (`HANDOFF.md` §5).
  The first live week should log rising candidates without alerting, to see how many there
  are and how many later qualified.
- Notification volume is the real product question: how many alerts a day is welcome. That
  is a policy number for `services/notify`, decided with data, not here.
