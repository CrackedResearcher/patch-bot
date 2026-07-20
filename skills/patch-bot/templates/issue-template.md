<!-- patch-bot-fingerprint: {{fingerprint}} -->
> Filed automatically by **patch-bot** from a production error. {{count}} occurrences · level `{{level}}`

## Production error: {{title}}

- **Where:** `{{culprit}}`
- **First seen:** {{firstSeen}} · **Last seen:** {{lastSeen}}
- **Occurrences:** {{count}}
- **Source:** [view in Sentry]({{permalink}})

### Stacktrace

```
{{stacktrace}}
```

---

### What to do (for @claude / the fixer)

1. **Reproduce first.** Add a new automated test that *fails* because of this bug,
   matching the stacktrace above. Don't change app code until the failing test exists.
2. **Fix minimally.** Smallest change that makes the test pass. No unrelated refactors.
3. **Verify.** Full suite green; the new test passes.
4. **PR** from `fix/patch-bot-{{issue_number}}` — `Closes #{{issue_number}}`, a 2-3 sentence
   root-cause summary, and a note about the reproducing test.

If this looks like a flaky, infra, or third-party error (timeouts, dropped connections,
vendor outages), **do not guess a fix** — comment with your reasoning and stop.

<sub>Dedupe marker above lets patch-bot avoid refiling this error. Don't edit it.</sub>
