# Setup routine

Run this once per target repo. Goal: leave the repo with a working Flow 2 (the fix
action) and a scheduled Flow 1 (the triage pass), and let the user verify quality
before anything goes fully autonomous.

## 0. Gather inputs (ask the user)

- **Target repo** — `owner/name` where fixes should be opened.
- **Sentry** — org slug, project slug, and an auth token with `event:read` +
  `project:read` (a Sentry *Internal Integration* token is ideal).
- **Anthropic key** — confirm `ANTHROPIC_API_KEY` exists as a repo secret (Flow 2 needs it).
- **Gate preference** — `label` (default, human presses go) or `auto` (full autonomous).

## 1. Scaffold Flow 2 (the fix action)

Copy `templates/claude.yml` → `<repo>/.github/workflows/claude.yml`. It triggers when an
issue is opened/labeled with `patch-bot:fix` or a comment mentions `@claude`, and runs
`anthropics/claude-code-action`. It needs the repo secret `ANTHROPIC_API_KEY` and the
default `GITHUB_TOKEN` (with `contents: write`, `pull-requests: write`, `issues: write`).

## 2. Scaffold config

Copy `templates/patch-bot.config.json` → `<repo>/patch-bot.config.json`. Fill in with the
user. Do NOT put tokens here — only slugs and policy knobs.

## 3. Create labels

```
gh label create patch-bot      --color 5319e7 --description "Filed by patch-bot" --force
gh label create patch-bot:fix  --color d93f0b --description "Ask @claude to fix this"  --force
```

## 4. Register the schedule (the runtime)

Set up a Claude Code cloud agent / cron that invokes this skill in Run mode on an
interval (15 min is a sane default). Use the `/schedule` skill. The scheduled run needs
these secrets in its environment (NOT in the repo, NOT in config):

- `SENTRY_AUTH_TOKEN`
- `GH_TOKEN` — a token that can open issues + apply labels on the target repo.

Remind the user: a cron run is headless — interactive OAuth/MCP logins won't be present,
so these must be real environment secrets.

## 5. First pass

The default gate is `auto` — patch-bot files the issue and fixes it on its own. That's what
most users want. But for the very first run, it's worth suggesting they temporarily set
`policy.gate` to `label` so a pass or two files issues **without** triggering fixes. That
lets them eyeball quality (is the fingerprint right? is the stacktrace useful? is the repro
instruction sound?) before handing over the keys. Once they're happy, set it back to `auto`
(or leave it on `label` if they'd rather approve each fix).

## 6. Summary

Print: what was scaffolded, where, the schedule cadence, the secrets still to set, and the
exact next action for the user (usually: "set the two secrets, then watch the first pass").
