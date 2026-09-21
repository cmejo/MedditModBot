# MedditModBot

A Devvit moderation bot for [r/medicine](https://reddit.com/r/medicine). It enforces four community-policy features, each independently toggleable between **off**, **shadow** (log-only), and **on** via the install settings page.

## Features

| Feature | Trigger | What it does |
|---|---|---|
| **AI disclosure gate** | `PostSubmit` | Removes new posts and stickies a comment asking OP to confirm whether/how they used AI tools. OP's reply to the sticky re-approves the post. |
| **Flair required (posts)** | `PostSubmit` | Removes posts by users without a user flair in the sub. |
| **Flair required (comments)** | `CommentSubmit` | Removes comments by users without a user flair in the sub. |
| **OP engagement check** | scheduled job, `CommentSubmit` | If a post has ≥N comments at the engagement window mark but OP hasn't commented, removes it and stickies a notice. OP commenting later re-approves and restores the original sticky. |
| **Minimum subreddit karma** | `PostSubmit` | Removes posts by users whose combined post + comment karma in this sub is below a threshold. |
| **Redaction cleanup** | `CommentUpdate`, mod menu | Removes comments whose body has been overwritten by a history-scrubbing tool (Redact etc.), matched against a configurable signature list. A subreddit-level mod menu action sweeps the "edited" listing to clear the backlog. |

Mods (including the app account itself) are exempt from every check. Each feature has a mode setting that controls behavior, with a `+` suffix flag that also mirrors the action to a configured Discord webhook:

| Mode | Reddit action | Discord mirror |
|---|---|---|
| `off` | none | no |
| `shadow` | none (log only) | no |
| `shadow+` | none (log only) | yes |
| `on` | live | no |
| `on+` | live | yes |

The AI gate has no `shadow` modes (its action is structural) but does support `off` / `on` / `on+`.

### Multipurpose sticky

There's a single bot sticky per post that transitions through states:

- `awaiting-ai` — asking OP to disclose AI usage.
- `flair-psa` — reminding commenters that flair is required (shown after AI is satisfied if flair-required is on, or directly if AI gate is off).
- `confirmed` — acknowledgement (only when AI gate is on and flair-required is off).

When the engagement check removes a post, a separate **Track B** sticky is created with the engagement-removed body so OP gets a fresh inbox notification. Reddit allows one sticky per post; the new sticky replaces the multipurpose one until the post is re-approved.

## Tech stack

- [Devvit Web](https://developers.reddit.com/docs/capabilities/devvit-web/devvit_web_overview) `0.13.0`
- [Hono](https://hono.dev/) for the server endpoints
- [Vite](https://vite.dev/) for bundling
- [TypeScript](https://www.typescriptlang.org/) strict mode (`exactOptionalPropertyTypes` on)
- [Vitest](https://vitest.dev/) for unit tests

## Project layout

```
src/
├── index.ts                      # Hono bootstrap
├── config.ts                     # Modes, Redis keys, sticky body templates
├── routes/
│   ├── triggers.ts               # PostSubmit / CommentSubmit dispatchers
│   ├── scheduler.ts              # op-engagement-check callback
│   ├── menu.ts                   # Mod-only debug-state menu item
│   └── api.ts                    # (empty, reserved)
├── core/
│   ├── settings.ts               # Typed getSettings() with 5s in-memory cache
│   ├── exemptions.ts             # isModerator() with cached mod list
│   ├── logging.ts                # logFeatureAction() structured logger
│   ├── reddit-helpers.ts         # removal helpers + human-mod predicates
│   └── sticky.ts                 # Track A multipurpose sticky state machine
└── features/
    ├── ai-gate/
    ├── flair-required/
    ├── op-engagement/
    └── min-karma/
docs/
└── playtest-checklist.md         # Manual playtest matrix
```

Each feature module exposes `decide()` (mostly pure) and `apply()` (Reddit-side I/O) plus a `run()` wrapper. The dispatcher in `routes/triggers.ts` fans triggers out to features in a fixed order on `PostSubmit` (min-karma → flair → AI gate → schedule engagement) and runs them independently on `CommentSubmit`.

## Settings

Configured per install at `https://developers.reddit.com/r/<subreddit>/apps/expdevsmodbot`:

| Key | Type | Default | Description |
|---|---|---|---|
| `aiGateMode` | off / on / on+ | off | AI disclosure gate. No shadow modes (binary effect). |
| `flairPostMode` | off / shadow / shadow+ / on / on+ | off | Flair requirement on posts. |
| `flairCommentMode` | off / shadow / shadow+ / on / on+ | off | Flair requirement on comments. Also controls whether the AI sticky transitions to the flair-required PSA after OP confirms AI usage. |
| `engagementMode` | off / shadow / shadow+ / on / on+ | off | 2-hour-ish OP-engagement check. |
| `minKarmaMode` | off / shadow / shadow+ / on / on+ | off | Subreddit-karma gate on posts. |
| `minKarmaThreshold` | number | 10 | Combined post + comment karma required. |
| `engagementWindowMinutes` | number | 120 | Window before the engagement check fires. |
| `engagementMinComments` | number | 10 | Minimum total comments before engagement check will remove. |
| `redactionMode` | off / shadow / shadow+ / on / on+ | off | Remove comments overwritten by history-scrubbing tools. Also gates the "purge redaction edits" menu action. |
| `redactionSignatures` | paragraph | `anonymized with \[?Redact` | One case-insensitive regex per line, tested against the edited comment body. Blank lines and `#` comments ignored; a malformed line is skipped and logged. Match the tool's attribution footer, not the filler text. |
| `discordWebhookUrl` | string | "" | Optional Discord webhook URL. Required for the `+` suffix modes to actually deliver mirrored notifications. |

## Setup

1. Install dependencies:
   ```bash
   npm install
   ```
2. Log in to Devvit:
   ```bash
   npm run login
   ```
3. Configure `devvit.json -> dev.subreddit` to point at your test subreddit.
4. Start the playtest watcher:
   ```bash
   npm run dev
   ```
5. In the test sub: open the app's install-settings page and flip features on (or to shadow).

**Required:** the app account must be a moderator of the target subreddit. Devvit auto-promotes it on install. The `getUserKarmaFromCurrentSubreddit` API used by min-karma needs this.

## Development commands

- `npm run dev` — Devvit playtest with file watching.
- `npm run type-check` — `tsc --build`.
- `npm test` — Vitest unit tests.
- `npm run lint` — ESLint over `src/`.
- `npm run build` — One-shot Vite build into `dist/`.
- `npm run deploy` — Type-check, lint, then upload a new version. Uploaded-only versions are visible to the app owner and installable to test subs under 200 subscribers.
- `npm run launch` — Type-check, lint, then `devvit publish`, which creates + uploads a version *and* files the review request. Don't chain `deploy` in front of it — `publish` does its own upload, and the second one errors.

Publishing does not move an existing install to the new version. After the review request clears, run `devvit install <subreddit>` and confirm with `devvit list installs <subreddit>`.

## Operations

### Streaming logs

```bash
npx devvit logs <subreddit> | grep "\[modbot\]"
```

Every action line follows:

```
[modbot] [<mode>] feature=<name> action=<verb> postId=... author=... reason="..."
```

`<mode>` is `live` or `shadow`. Useful filters:

- `grep "\[shadow\]"` — what the bot would have done during a validation period.
- `grep "feature=op-engagement"` — narrow to one feature.

### Debug menu

A mod-only post menu item, "Modbot: dump post state," dumps the bot's Redis keys for that post:

- `processed:postsubmit:<id>` — idempotency claim, proves the trigger ran.
- `bot-sticky:<id>` — Track A multipurpose sticky record.
- `engagement-sticky:<id>` — Track B record (only set after engagement removal).
- `engage:<id>` — pending engagement-check job metadata.
- `removed-by-us:<id>` — marker indicating the bot performed the most recent removal (used by the human-mod-override guard).

The full JSON is also written to `devvit logs` as `[modbot] [debug-state] {...}`.

### Redaction backfill

The `CommentUpdate` trigger only sees edits made after it's live. To clear the backlog, use the subreddit-level mod menu item **"Modbot: purge redaction edits"**. It pages the subreddit's "edited" mod listing (`reddit.getEdited`), applies the same signature check as the trigger, and reports counts in a toast. It honors `redactionMode` — in shadow it reports what it *would* remove, in off it refuses — so it can't bypass a feature that's switched off. The sweep is capped at 500 comments per click to stay inside the menu handler's execution window; the toast says when to click again.

### Discord mirroring

Create a Discord webhook (channel settings → Integrations → Webhooks → New Webhook → Copy URL), paste it into the `discordWebhookUrl` setting, then switch features to a `+` suffix mode:

- `shadow+` — log + Discord, no Reddit action. Use during validation to watch decisions without enforcing.
- `on+` — live + Discord. Use in production when you want ongoing visibility of every action.

Each Discord message is a single concise sentence with an inline link to the affected post. The `<url>` syntax suppresses link-preview cards so the channel stays a flat readable list:

> Removed [post](https://reddit.com/comments/abc123) by u/Alice — karma 3 < threshold 10

Production behavior is never blocked by Discord failures — the call is fire-and-forget with errors swallowed and logged.

### Roll-out

1. Deploy with all features `off` (current defaults).
2. Set `discordWebhookUrl` and flip a feature to `shadow+` so you can watch decisions in your moderator Discord channel.
3. After a few days of clean signals, flip the feature to `on+` (or `on` if you'd rather not get ongoing notifications).
4. Repeat for the next feature.

See `docs/playtest-checklist.md` for the manual verification matrix used during initial validation.

## Architectural notes

- **Idempotency.** Triggers can be delivered more than once. The dispatcher claims `processed:postsubmit:<postId>` via SETNX with a 1-hour TTL before any destructive action.
- **Human-mod override.** Every re-approval path checks `wasRemovedByHumanMod()`, which combines `post.removedByCategory` (`'moderator'`, `'anti_evil_ops'`, `'community_ops'`, `'content_takedown'`, `'copyright_takedown'`) with the absence of our `removed-by-us:<id>` marker. If a human mod removed the post, the bot transitions sticky state but does not undo the removal.
- **Scheduler reliability.** The engagement check is at-least-once and may run late. The scheduled handler always re-fetches post state and decides fresh — it never trusts state captured at schedule time.
- **OP reply exemption.** OP's reply to the Track A sticky is exempted from the flair check by the comment dispatcher; otherwise we'd silently remove the AI-unlock comment.
- **Settings cache.** A 5-second in-memory cache shields the settings client from burst trigger fan-out without blocking moderator-visible configuration changes for more than a few seconds.

## License

BSD-3-Clause (see `LICENSE`).
