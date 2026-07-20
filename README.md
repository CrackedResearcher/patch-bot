# patch-bot

> A Claude Code skill that fixes your production bugs and opens the PR.

patch-bot watches your error tracker. When a real bug hits production, it opens a GitHub
issue, and Claude writes a test that reproduces the bug, fixes it, and opens a pull request.
You review and merge.

There's no server to run — it's a Claude Code skill that runs on a schedule.

## Features

- **Automatic fixes.** Every bug becomes a test, a fix, and a PR — hands-off.
- **No infrastructure.** No backend to deploy. It polls your error tracker on a schedule.
- **No duplicate issues.** Each error is fingerprinted, so the same bug is never filed twice.
- **Automatic by default.** It opens the issue and Claude fixes it — no babysitting. Want
  to approve each fix first? Turn on label mode and it waits for your go-ahead.
- **Fixes come with tests.** Claude has to reproduce the bug with a failing test before it
  changes any code — so you get a regression test, not a guess.

## How it works

```
sentry error
   │
   ▼   every ~15 min patch-bot checks sentry and files what's real
github issue  ──►  claude writes a test, fixes the bug, opens a PR  ──►  you review & merge
```

Two parts: patch-bot spots the error and writes it up, and Claude Code's GitHub Action does
the fix. Setup wires up both.

## Getting started

### Prerequisites

- A GitHub repo where fixes should land
- A Sentry project and an auth token (`event:read`, `project:read`)
- An Anthropic API key

### 1. Install

```bash
/plugin install patch-bot
```

### 2. Set up

From your repo, in Claude Code:

```bash
run patch-bot setup
```

This scaffolds the fix workflow, creates the config file, adds the labels, and helps you
register the scheduled agent.

### 3. Add your secrets

Where the scheduled agent runs:

| secret | used for |
| --- | --- |
| `SENTRY_AUTH_TOKEN` | reading errors from Sentry |
| `GH_TOKEN` | opening issues and applying labels |

And as a GitHub repo secret, for the fix workflow:

| secret | used for |
| --- | --- |
| `ANTHROPIC_API_KEY` | the Claude Code fix run |

That's it. Let it run a pass or two and read the issues it files before you go live.

## Usage

Once it's running, your day looks like this:

1. A bug hits production.
2. Within ~15 minutes, patch-bot opens an issue and Claude opens a PR that fixes it.
3. You review and merge.

That's the default — it just handles bugs on its own.

**Want to approve fixes yourself?** Turn on label mode (`"gate": "label"`). Then patch-bot
opens the issue and stops there. Claude only fixes it once you add the `patch-bot:fix` label
to the ones you care about. Everything else works the same.

## Configuration

Everything lives in `patch-bot.config.json`:

```jsonc
{
  "repo": "owner/name",
  "source": { "provider": "sentry", "org": "your-org", "project": "your-project" },
  "policy": {
    "min_occurrences": 5,        // ignore errors seen fewer than this many times
    "max_per_run": 5,            // most issues to open in a single pass
    "gate": "auto",              // "auto" = fix automatically (default), "label" = you approve each fix
    "levels": ["error", "fatal"],
    "deny": ["TimeoutError", "ConnectionError", "healthcheck"],
    "cooldown_hours": 24         // don't refile a bug you just closed
  }
}
```

The default is `"auto"` — patch-bot fixes bugs on its own. Prefer to approve each one? Set
it to `"label"`. Either way, the thresholds, `deny` list, and `max_per_run` cap are what
keep it from spamming, so tune those to taste.

## Adding other error trackers

Sentry works out of the box. To add Rollbar, Bugsnag, or Datadog, write a small adapter that
maps their errors to the same shape and set `source.provider`. Nothing else changes.

## License

MIT
