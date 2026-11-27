---
title: Guides
---

# Guides

Instructions for things I do rarely enough that I have to look them up — which is exactly the
kind of task that deserves a page instead of a reopened browser history.

The measure of a good guide here: **past-me, mid-task, at 23:00, can follow it without thinking.**
That means exact commands, the gotchas mentioned *before* they bite, and a "done when" line so
it's obvious the job actually finished.

## All guides

| Guide | Tags | Time |
|---|---|---|
| [Cron jobs](cron-jobs.md) | linux, automation | 20 min |
| [Telegram bot](telegram-bot.md) | telegram, notifications | 20 min |

## Writing a new one

Start from this front matter — the tags are what route it into the tag pages:

```yaml
---
title: Bleeding Shimano brakes
tags: [guide, motorcycle, brakes]
difficulty: easy        # easy | medium | involved
time: 30 min
date: 2026-11-27
---
```

And this skeleton:

```text
# Title

One paragraph: what this does, and when you'd want it.

## Before you start        <- tools, parts, assumptions, safety
## Steps                   <- numbered, real commands
## Gotchas                 <- !!! warning admonitions, the things that bite
## Done when               <- how you know it worked
## Sources                 <- links, versions, dates
```
