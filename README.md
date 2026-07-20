# patch-bot

a claude code skill that fixes your prod bugs and opens the PR.

## what it is

patch-bot watches your error tracker. when a real bug shows up in production, it opens a
github issue, and claude writes a failing test that reproduces it, fixes it, and opens a
pull request for you to review. you just merge.

no server to run. it's just a skill plus a schedule.

## why i built this

at my company i wired this up as an internal tool — sentry would fire a webhook, a little
backend turned it into a github issue, tagged claude, and claude shipped the fix. it worked
great, so i wanted to give it to everyone.

but a hosted backend is a pain for other people to run. so i rebuilt it without the server:
instead of sentry pushing to something you have to host, a scheduled claude agent just polls
sentry every few minutes. same result, nothing to deploy, installs in two minutes.

## how it works

```
sentry error
   │
   │   every ~15 min, patch-bot wakes up and:
   ▼
patch-bot triage  ── polls sentry, drops the noise, dedupes ──► opens a github issue (@claude)
                                                                      │
                                                                      ▼
                                              claude code action ──► repro test + fix + PR
                                                                      │
                                                                      ▼
                                                                 you review & merge
```

two halves: patch-bot does the triage (the boring part — noticing, filtering, writing it
up). claude code's github action does the actual fixing. patch-bot sets both up for you.

a couple of things that keep it sane:
- **it dedupes.** each issue carries a fingerprint, so the same bug never gets filed twice.
- **you press go.** by default patch-bot files the issue but waits for you to add a
  `patch-bot:fix` label before claude touches anything. flip one config flag for full auto.
- **fix comes with a test.** claude has to reproduce the bug with a failing test before it's
  allowed to change code — so every fix ships with a regression test, not a guess.

## setup

you'll need: a github repo, a sentry project + token, and an `ANTHROPIC_API_KEY`.

1. install it

   ```
   /plugin install patch-bot
   ```

2. from your repo, in claude code:

   ```
   run patch-bot setup
   ```

   this scaffolds the fix action, writes `patch-bot.config.json`, makes the labels, and
   helps you register the scheduled triage agent.

3. set your secrets where the schedule runs (not in the repo):

   | secret | for |
   | --- | --- |
   | `SENTRY_AUTH_TOKEN` | reading errors |
   | `GH_TOKEN` | opening issues + labels |

   and `ANTHROPIC_API_KEY` as a repo secret, for the fix action.

that's it. let a pass or two run and watch the issues it files before you hand it the keys.

## using it day to day

a bug hits prod → within ~15 min patch-bot files an issue with the stacktrace → you add
`patch-bot:fix` if it's worth fixing → claude opens a PR that closes it → you review & merge.

you stay in charge of what gets fixed and what gets merged. patch-bot just handles the
tedious middle.

## other error trackers

it's built around one normalized shape, so rollbar / bugsnag / datadog are additive — drop
in an adapter next to the sentry one and point `source.provider` at it. sentry works today.

## license

MIT
