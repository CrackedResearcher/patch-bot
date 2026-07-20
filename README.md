# patch-bot

**Turn production errors into fix PRs — automatically.**

patch-bot is a [Claude Code](https://claude.com/claude-code) skill that watches your error
tracker, and when a real production bug shows up it opens a GitHub issue and lets Claude
write a reproducing test, a fix, and a pull request for your team to review.

No server to host. The whole thing is a skill + a schedule.

```
Sentry error  ─►  patch-bot triage (scheduled)  ─►  GitHub issue (@claude)
                                                          │
                                                          ▼
                              Claude Code Action  ─►  repro test + fix + PR  ─►  you review & merge
```

## Why it works this way

- **Poll, don't host.** Instead of running a webhook server 24/7, a scheduled Claude agent
  polls Sentry every ~15 min. Zero infrastructure; near-real-time is plenty for bug fixes.
- **GitHub is the database.** Dedupe is done by searching existing issues for a fingerprint
  marker — no extra storage to run.
- **Human-gated by default.** patch-bot files the issue automatically but (by default) waits
  for a human to add the `patch-bot:fix` label before Claude attempts a fix. Full-auto is one
  config flag away once you trust it.
- **Repro-first fixes.** The fixer must write a *failing test that reproduces the bug* before
  changing any code — so you get a regression test with every fix, not a guess.

## Requirements

- A GitHub repo where fixes should land.
- A Sentry project + auth token (`event:read`, `project:read`).
- An `ANTHROPIC_API_KEY` set as a **repo secret** (the fix action uses it).
- Claude Code installed locally, and access to scheduled cloud agents for the runtime.

## Install

```bash
# as a Claude Code plugin
/plugin install patch-bot
```

Or clone this repo into your Claude Code skills directory.

## Set up (once per repo)

In Claude Code, from the repo you want to protect:

```
run patch-bot setup
```

It will:
1. add `.github/workflows/claude.yml` (the fix action),
2. create `patch-bot.config.json` and fill it in with you,
3. create the `patch-bot` and `patch-bot:fix` labels,
4. help you register the scheduled triage agent,
5. tell you which secrets to set.

Set these secrets where the scheduled agent runs (**not** in the repo):

| secret | used for |
| --- | --- |
| `SENTRY_AUTH_TOKEN` | reading errors from Sentry |
| `GH_TOKEN` | opening issues + applying labels |

And this one as a **GitHub repo secret** (for the fix action):

| repo secret | used for |
| --- | --- |
| `ANTHROPIC_API_KEY` | the Claude Code fix run |

## Everyday use (what it looks like once live)

1. A bug hits production. Sentry records it.
2. Within ~15 min, patch-bot's scheduled pass sees it, checks it clears your policy
   (occurrence threshold, not on the denylist, right level), and opens a GitHub issue with
   the stacktrace and repro instructions.
3. **You** glance at the issue. If it's worth fixing, add the **`patch-bot:fix`** label.
   (Or set `gate: "auto"` and skip this step entirely.)
4. The Claude Code Action writes a failing test, fixes the bug, and opens a PR from
   `fix/patch-bot-<n>` that closes the issue.
5. You review the PR like any other and merge.

You stay in control of *what* gets fixed and *what* gets merged; patch-bot removes the
tedious middle — noticing, triaging, writing it up, reproducing, and drafting the fix.

## Configuration

Everything lives in `patch-bot.config.json`. The important knobs:

```jsonc
"policy": {
  "min_occurrences": 5,       // ignore rare/one-off errors
  "max_per_run": 5,           // cap issues per pass — one bad deploy can't flood you
  "gate": "label",            // "label" = human presses go (default) | "auto" = autonomous
  "levels": ["error", "fatal"],
  "deny": ["TimeoutError", "ConnectionError", "third_party/", "healthcheck"],
  "cooldown_hours": 24        // don't refile a just-closed error
}
```

Start with `gate: "label"` and let a pass or two run so you can judge issue quality before
handing over the keys.

## Adding other error trackers

patch-bot normalizes every source into one shape, so Rollbar / Bugsnag / Datadog / a generic
webhook are additive: drop in a new adapter next to `skills/patch-bot/references/sentry-adapter.md`
that emits the same `NormalizedError`, and set `source.provider` accordingly. Nothing
downstream changes.

## License

MIT
