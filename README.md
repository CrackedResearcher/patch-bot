# patch-bot

a claude code skill that fixes your prod bugs and opens the PR.

## what this is

you know the drill when a bug hits production — someone has to notice it, dig through the
stacktrace, figure out where it's coming from, write a fix, add a test. patch-bot does that
part for you. it watches your errors, and when a real one shows up claude writes a test that
reproduces it, fixes it, and opens a PR. you just review and merge.

and there's nothing to host. it's a skill plus a schedule, that's the whole thing.

## why i made it

i built this at work first, as an internal tool. sentry would fire a webhook, a little
backend turned it into a github issue and tagged claude, and claude shipped the fix. worked
really well, so i wanted other people to be able to use it too.

thing is, nobody wants to stand up a backend just to try your tool. so i ripped the server
out. instead of sentry pushing to something you host, a scheduled claude agent just checks
sentry every few minutes. same outcome, except now there's nothing to deploy and you can be
running it in a couple minutes.

## how it works

```
sentry error
   │
   │   every ~15 min patch-bot wakes up, checks sentry,
   ▼   drops the noise, and files what's real
github issue (@claude)
   │
   ▼
claude code action ──► writes a repro test, fixes it, opens a PR ──► you review & merge
```

two parts, really. patch-bot handles the annoying half — noticing the error, throwing away
the noise, not filing the same bug twice, writing it up. then claude code's github action
does the actual fixing. setup wires up both.

a few things i was careful about. it won't file the same bug twice — every issue carries a
fingerprint. it doesn't just start opening PRs on its own either; by default it files the
issue and waits for you to slap a `patch-bot:fix` label on the ones you actually care about
(there's a flag to go full auto once you trust it). and claude isn't allowed to touch any
code until it's written a failing test that reproduces the bug — so you always get a
regression test out of it, never a blind guess.

## running it

you'll need a github repo, a sentry project + token, and an anthropic key.

1. install it — `/plugin install patch-bot`
2. from your repo, in claude code — `run patch-bot setup`. it scaffolds the fix action,
   writes the config, makes the labels, and helps you set up the scheduled agent.
3. drop in your secrets where the schedule runs: `SENTRY_AUTH_TOKEN` and `GH_TOKEN`. and
   `ANTHROPIC_API_KEY` as a repo secret for the fix action.

then just let it run a pass or two and read the issues it files before you hand it the keys.

## day to day

a bug hits prod, and within ~15 min there's an issue waiting with the stacktrace. if it's
worth fixing you label it, claude opens a PR, you merge. you're always the one deciding what
gets fixed and what actually ships — patch-bot just takes the boring middle off your plate.

## other trackers

sentry works today. it's built around one normalized shape, so rollbar / bugsnag / datadog
are just a small adapter away.

## license

MIT
