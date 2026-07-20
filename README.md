# patch-bot

A Claude Code skill that fixes your prod bugs and opens the PR.

patch-bot watches your error tracker. When a real bug hits prod, it opens a GitHub issue, Claude writes a test that reproduces it, fixes it, and opens a PR. You review and merge.

No server, no backend — it's a Claude Code skill that runs on a schedule.

## What it does

- Turns every bug into a test + a fix + a PR
- Fingerprints errors so the same bug isn't filed twice
- Writes a failing test to reproduce the bug before changing any code, so you get a regression test
- Runs on its own by default — turn on label mode if you want to okay each fix first

## How it works

```
sentry error
   │
   ▼   every ~15 min patch-bot checks sentry and files what's real
github issue  ──►  claude writes a test, fixes the bug, opens a PR  ──►  you review & merge
```

## Setup

You'll need:
- A GitHub repo where fixes should land
- A Sentry project + auth token (`event:read`, `project:read`)
- An Anthropic API key

Install:
```bash
/plugin install patch-bot
```

Set up, from your repo in Claude Code:
```bash
run patch-bot setup
```

Add your secrets where the scheduled agent runs:

| secret | for |
| --- | --- |
| `SENTRY_AUTH_TOKEN` | reading errors from Sentry |
| `GH_TOKEN` | opening issues and adding labels |

And as a GitHub repo secret:

| secret | for |
| --- | --- |
| `ANTHROPIC_API_KEY` | the fix run |

## Config

`patch-bot.config.json`:

```jsonc
{
  "repo": "owner/name",
  "source": { "provider": "sentry", "org": "your-org", "project": "your-project" },
  "policy": {
    "min_occurrences": 5,        // ignore errors seen fewer times than this
    "max_per_run": 5,            // most issues to open in one pass
    "gate": "auto",              // "auto" = fix automatically, "label" = you approve each one
    "levels": ["error", "fatal"],
    "deny": ["TimeoutError", "ConnectionError", "healthcheck"],
    "cooldown_hours": 24         // don't refile a bug you just closed
  }
}
```

`gate` is `"auto"` by default. Set it to `"label"` to approve fixes yourself — patch-bot opens the issue, and Claude only fixes it once you add the `patch-bot:fix` label.

## Other error trackers

Sentry works out of the box. For Rollbar, Bugsnag, or Datadog, write a small adapter that maps their errors to the same shape and set `source.provider`.

## License

MIT
