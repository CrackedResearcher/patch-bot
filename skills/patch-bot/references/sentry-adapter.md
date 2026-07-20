# Sentry adapter

Turns Sentry issues into the common `NormalizedError` shape. This is the **seam**: to
support Rollbar/Bugsnag/Datadog/a generic webhook later, add a sibling adapter that emits
the same shape — nothing downstream changes.

## Common shape (what every adapter must emit)

```jsonc
{
  "fingerprint": "string",   // stable id for dedupe — use Sentry's issue id
  "title":       "string",   // short human title
  "culprit":     "string",   // where it happened (module/function/route)
  "level":       "error",    // error | fatal | warning | ...
  "count":       123,        // total occurrences
  "permalink":   "https://sentry.io/...",
  "stacktrace":  "string",   // top frames, trimmed
  "firstSeen":   "ISO-8601",
  "lastSeen":    "ISO-8601"
}
```

## Pulling from Sentry

Auth: `Authorization: Bearer $SENTRY_AUTH_TOKEN`. Base: `https://sentry.io/api/0`.

List recently-active unresolved issues for the project (newest activity first):

```
GET /projects/{org}/{project}/issues/?query=is:unresolved&statsPeriod=24h&sort=freq
```

For each issue you keep after the policy filter, fetch the latest event to get a
stacktrace:

```
GET /issues/{issue_id}/events/latest/
```

Pull the exception values + top ~15 frames (`filename:function:lineno`), trim to keep the
issue body readable.

## Mapping

| Sentry field                         | NormalizedError |
|--------------------------------------|-----------------|
| `issue.id`                           | `fingerprint`   |
| `issue.title` / `metadata.value`     | `title`         |
| `issue.culprit`                      | `culprit`       |
| `issue.level`                        | `level`         |
| `issue.count`                        | `count`         |
| `issue.permalink`                    | `permalink`     |
| latest event exception frames        | `stacktrace`    |
| `issue.firstSeen` / `issue.lastSeen` | `firstSeen` / `lastSeen` |

## Incremental polling

Track the previous pass's high-water `lastSeen` (or just filter to `statsPeriod` a bit
wider than the schedule interval, then let dedupe absorb the overlap). Overlap + dedupe is
safer than trying to be exact and missing an error on a boundary.

## Tokens

Prefer a Sentry **Internal Integration** token scoped to `event:read` + `project:read`.
Never hardcode it; read from `$SENTRY_AUTH_TOKEN`.
