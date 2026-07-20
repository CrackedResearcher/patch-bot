---
name: patch-bot
description: >-
  Autonomously turn production errors into fix PRs. Use this skill to (1) SET UP
  the self-healing pipeline in a repo — it scaffolds a GitHub Action + config, or
  (2) RUN a triage pass that polls an error tracker (Sentry) for new production
  errors, dedupes them against existing issues, applies a policy filter, and opens
  a GitHub issue tagging @claude to write a reproducing test, a fix, and a PR.
  Trigger on: "set up patch-bot", "run a patch-bot triage pass", "check prod
  errors and file fixes", or when invoked by a scheduled cloud agent.
---

# patch-bot

patch-bot closes the loop from **production error → reviewed pull request** with no
human in the middle of the boring part. It has two modes.

- **Setup mode** — run once per repo. Scaffolds everything the pipeline needs.
- **Run mode** — one triage pass. A scheduled cloud agent calls this on an interval
  (e.g. every 15 min); it can also be run on demand.

Decide the mode: if `patch-bot.config.json` does not exist in the repo root, do
**Setup**. Otherwise do **Run**. If the user explicitly says "setup" or "run", honor that.

---

## Architecture (how the loop works)

```
error tracker (Sentry)              ─┐
   every N min, patch-bot polls it   │  FLOW 1: TRIAGE  (this skill, scheduled)
   → normalize new errors            │
   → dedupe vs existing GH issues    │
   → policy filter (thresholds/gate) │
   → open GitHub issue + tag @claude ─┘
                                      │
GitHub Action (anthropics/claude-code-action)
   fires on the @claude tag          ─┐  FLOW 2: FIX  (scaffolded, event-driven)
   → write a FAILING repro test      │
   → implement the fix               │
   → open a PR from a fix/ branch    ─┘
   → human reviews + merges
```

patch-bot owns **Flow 1**. Flow 2 is Anthropic's official action, which Setup mode
scaffolds into the target repo. There is no server to host — the schedule is the runtime.

---

## Setup mode

Follow `references/setup.md`. In short:

1. Confirm prerequisites with the user (Sentry auth token + org/project slug, the
   target GitHub repo, and that `ANTHROPIC_API_KEY` is available as a repo secret).
2. Copy `templates/claude.yml` → `.github/workflows/claude.yml` in the target repo.
3. Copy `templates/patch-bot.config.json` → `patch-bot.config.json` at the repo root
   and fill in the values with the user.
4. Create the GitHub labels `patch-bot` and `patch-bot:fix`.
5. Register the schedule (a Claude Code cloud agent / cron) that runs this skill in
   Run mode. Tell the user the secrets it needs (`SENTRY_AUTH_TOKEN`, `GH_TOKEN`).
6. Print a summary + a "dry-run" suggestion so the first pass opens issues WITHOUT
   tagging @claude, so the user can eyeball quality before going live.

---

## Run mode (one triage pass)

Read `patch-bot.config.json` first — it drives every decision below.

1. **Fetch** new errors from the tracker. Use `references/sentry-adapter.md`. Only
   pull issues whose `lastSeen` is newer than the last pass, capped at
   `policy.max_per_run`. Never fetch the entire backlog.

2. **Normalize** each into the common shape:
   `{ fingerprint, title, culprit, level, count, permalink, stacktrace, firstSeen, lastSeen }`.

3. **Policy filter** — apply `references/policy.md`: drop anything below
   `min_occurrences`, matching `deny`, or not matching `allow` (if `allow` is set).
   Log every drop with the reason (never silently skip — silent skips read as
   "nothing to do" when there was).

4. **Dedupe against GitHub — the state store is GitHub itself, not a database.**
   For each survivor, search issues for the fingerprint marker
   `patch-bot-fingerprint: <fingerprint>` (it lives in the issue body). If an OPEN or
   recently-closed issue already carries it, skip — don't refile.

5. **Open the issue.** Render `templates/issue-template.md` with the normalized data.
   The body MUST embed the fingerprint marker (for dedupe) and the repro-first
   instructions (the part that makes fixes good, not just fast).

6. **Gate the agent** per `policy.gate`:
   - `"auto"` (default) → also post a comment `@claude` (or apply `patch-bot:fix`) to fire
     Flow 2 now, so the fix happens without waiting on anyone.
   - `"label"` → open the issue only, then stop. A human adds `patch-bot:fix` when they
     want the fix attempted. Use when you want to approve *what* gets fixed.

7. **Record** the pass: log a one-line summary — errors seen, filtered (with reasons),
   issues opened, agents triggered. This summary is what the scheduled-run log shows.

---

## Guardrails (do not skip — this is what separates a tool from a toy)

- **Cap per run** (`max_per_run`) so one bad deploy can't spawn 50 issues/PRs.
- **Dedupe before create**, always — an un-deduped poll loop refiles the same bug forever.
- **Auto is the default**, so the other guardrails do the real work of preventing spam —
  thresholds, `deny`, dedupe, and `max_per_run`. Keep those tuned. `gate: "label"` is the
  opt-in for when a human should approve *what* gets fixed.
- **Headless auth**: the scheduled agent runs with nobody present, so it needs
  `SENTRY_AUTH_TOKEN` and `GH_TOKEN` as agent secrets — interactive/OAuth MCP logins
  won't be available in a cron run.
- **Never** commit tokens into `patch-bot.config.json`. Config holds slugs and knobs;
  secrets come from the environment.

---

## Files in this skill

- `references/setup.md` — the one-time setup routine, step by step.
- `references/policy.md` — the policy schema + how gating/dedupe/thresholds work.
- `references/sentry-adapter.md` — how to pull + normalize Sentry errors (the seam for
  adding Rollbar/Bugsnag/Datadog/generic later).
- `templates/claude.yml` — the GitHub Action that performs Flow 2 (the fix).
- `templates/issue-template.md` — the issue body, incl. fingerprint + repro-first prompt.
- `templates/patch-bot.config.json` — the config the user fills in during Setup.
