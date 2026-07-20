# Policy & dedupe

The policy block in `patch-bot.config.json` is the difference between a useful tool and a
PR-spam machine. Every field is a guardrail.

## Schema

```jsonc
"policy": {
  "min_occurrences": 5,        // ignore errors seen fewer than N times
  "max_per_run": 5,            // hard cap on issues opened per triage pass
  "gate": "auto",              // "auto" (autonomous, default) | "label" (human presses go)
  "levels": ["error", "fatal"],// only these Sentry levels qualify
  "allow": [],                 // if non-empty, culprit/title MUST match one of these (substring or /regex/)
  "deny": [                    // drop if culprit/title matches any of these
    "TimeoutError",
    "ConnectionError",
    "third_party/",
    "healthcheck"
  ],
  "cooldown_hours": 24         // don't refile a fingerprint closed within this window
}
```

## How each stage uses it

1. **Level filter** — drop errors whose level isn't in `levels`.
2. **Threshold** — drop errors with `count < min_occurrences`. New-but-rare errors are
   usually noise; a real bug recurs.
3. **allow / deny** — `deny` wins over `allow`. These target flaky, infra, and
   third-party errors the agent should never try to "fix" (timeouts, dropped
   connections, vendor outages, health checks).
4. **Cap** — after filtering, sort by `count` desc and take at most `max_per_run`. Log how
   many were dropped by the cap so the tail isn't silently lost.

## Dedupe (state = GitHub, no database)

Every filed issue embeds a marker in its body:

```
<!-- patch-bot-fingerprint: <sentry-fingerprint> -->
```

Before opening an issue, search:

```
gh issue list --repo <repo> --state all --search "patch-bot-fingerprint: <fingerprint> in:body"
```

- Match found, issue OPEN → skip (already being worked).
- Match found, issue CLOSED within `cooldown_hours` → skip (just fixed; give it time to
  confirm the fix held).
- Match found, issue CLOSED older than `cooldown_hours` → the bug is back; file a fresh
  issue and link the old one.
- No match → file it.

## Gate

- `"auto"` (default): patch-bot opens the issue and immediately tags `@claude`, so the fix
  happens on its own with nobody in the loop. The thresholds, `deny` list, dedupe, and
  `max_per_run` cap are what keep this from spamming — tune those, not the gate.
- `"label"`: patch-bot opens the issue only, then stops. A human adds `patch-bot:fix` when
  they want the fix attempted. Use this when you want to approve *what* gets fixed while
  patch-bot still does all the tedious detection + write-up.
