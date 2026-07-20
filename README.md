# patch-bot

a claude code skill that fixes your prod bugs and opens the PR.

## the problem

every company ships bugs. one shows up in production, and now someone has to notice it, read
the stacktrace, find where it is in the code, write a fix, add a test, and open a PR. it
takes time, it's boring, and it happens again and again.

## the solution

at my company i got tired of this, so i built a bot to do it for us. it watches for bugs and
fixes them on its own. it worked really well, so i turned it into something anyone can use.

that's patch-bot. when a real bug hits production, it opens a github issue, and claude writes
a test that reproduces the bug, fixes it, and opens a pull request. you just look it over and
merge. there's nothing to host — it's a skill plus a schedule.

## how it works

```
sentry error
   │
   ▼   every ~15 min patch-bot checks sentry and files what's real
github issue (@claude)
   │
   ▼
claude ──► writes a test, fixes the bug, opens a PR ──► you review & merge
```

patch-bot does the boring half — spotting the error and writing it up. claude does the fix.

## features

- **fixes bugs for you** — a test, a fix, and a PR, done automatically
- **nothing to host** — no server, it just checks sentry on a schedule
- **never files the same bug twice** — each issue has a fingerprint
- **you stay in control** — it files the issue, but waits for you to okay the fix (or flip a
  flag to go full auto)
- **every fix has a test** — claude must reproduce the bug first, so you never get a guess

## how to use it

you'll need a github repo, a sentry token, and an anthropic key.

1. install it — `/plugin install patch-bot`
2. in your repo, run `run patch-bot setup`. it sets everything up for you.
3. add your secrets: `SENTRY_AUTH_TOKEN` and `GH_TOKEN` where the schedule runs, and
   `ANTHROPIC_API_KEY` as a repo secret.

let it run once or twice and read the issues before you hand it the keys.

## adding new providers

it works with sentry today. want rollbar, bugsnag, or datadog? add a small adapter and point
the config at it — everything else stays the same.

## license

MIT
