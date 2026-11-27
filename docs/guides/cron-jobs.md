---
title: Cron jobs
tags: [guide, linux, automation]
difficulty: easy
time: 20 min
date: 2026-11-27
---

# Cron jobs

Scheduling a command to run on its own, without a systemd unit, a queue, or a cloud service.
Cron has been doing this since 1975 and will still be doing it when the current hype cycle is
archaeology.

## The five fields

`crontab -e` opens your personal crontab.

```text
┌───────────── minute        (0-59)
│ ┌───────────── hour        (0-23)
│ │ ┌───────────── day of month (1-31)
│ │ │ ┌───────────── month     (1-12)
│ │ │ │ ┌───────────── day of week (0-6, Sunday = 0; 7 also = Sunday)
│ │ │ │ │
* * * * *  command to run
```

`*` means every value. Lists (`1,15`) and ranges (`1-5`) are allowed, and `*/n` means every
*n*th value.

## Examples worth keeping

```text
# every five minutes
*/5 * * * * /home/gordon/bin/check-disk.sh

# every day at 03:30 — the classic backup slot
30 3 * * * /home/gordon/bin/backup.sh

# weekdays at 08:00
0 8 * * 1-5 /home/gordon/bin/morning.sh

# every 15 minutes during office hours, workdays only
*/15 9-17 * * 1-5 /home/gordon/bin/poll.sh

# first of the month at midnight
0 0 1 * * /home/gordon/bin/monthly-report.sh
```

Special strings, for when you can't be bothered with fields:

```bash
@reboot   /home/gordon/bin/on-startup.sh
@daily    /home/gordon/bin/daily.sh       # 00:00
@weekly   /home/gordon/bin/weekly.sh      # Sunday 00:00
@hourly   /home/gordon/bin/hourly.sh
```

Check what's scheduled with `crontab -l`. Edit with `crontab -e` — never by hand-editing the
file under `/var/spool/cron/`.

## The four things that break cron jobs

Almost every "my cron job doesn't work" is one of these.

### 1. `PATH` is not your `PATH`

Cron runs with a minimal environment: usually `PATH=/usr/bin:/bin` and `HOME` of the job owner.
Anything in `/usr/local/bin`, `~/.local/bin` or installed by `nvm`/`pyenv` is invisible.

Fix it at the top of the crontab:

```bash
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin:/home/gordon/.local/bin
MAILTO=""
```

Or, more robust, call things by absolute path and let scripts set up their own environment.

### 2. No output means mail; output means *somewhere* else

Cron mails stdout/stderr to the job owner if a mail system exists. On most machines it doesn't,
so failure is silent. Redirect to a log, always:

```bash
30 3 * * * /home/gordon/bin/backup.sh >> /home/gordon/logs/backup.log 2>&1
```

Add timestamps so the log is readable later:

```bash
30 3 * * * { echo "=== $(date -Is)"; /home/gordon/bin/backup.sh; } >> /home/gordon/logs/backup.log 2>&1
```

### 3. `%` is special inside a crontab

Every `%` in the command is turned into a newline by cron; the first one also ends the command.
`date +%F` therefore breaks. Escape it as `\%`:

```bash
0 3 * * * /home/gordon/bin/report.sh > /tmp/report-\$(date +\%F).txt 2>&1
```

(This is a cron-only quirk — it works fine in a script, which is another reason to put logic in
a script and keep the crontab line boring.)

### 4. Jobs don't catch up

If the machine was off at 03:30, the 03:30 job simply didn't happen. `anacron` (daily-ish jobs)
or a systemd timer with `Persistent=true` (`OnCalendar=`) handles that case properly.

## Two more habits

**Lock out overlapping runs.** A job that occasionally takes longer than its interval will
happily start a second copy of itself:

```bash
*/5 * * * * flock -n /tmp/poll.lock /home/gordon/bin/poll.sh >> /home/gordon/logs/poll.log 2>&1
```

**Test with cron's environment, not yours.** Run the command once with a stripped environment
before you trust it:

```bash
env -i SHELL=/bin/sh PATH=/usr/bin:/bin /home/gordon/bin/backup.sh
```

If it fails there, it will fail at 03:30 too.

## Where crontabs come from

| Location | Notes |
|---|---|
| `crontab -e` | the current user's crontab; no user field |
| `/etc/crontab` | system crontab; **has** a user field as the 6th field |
| `/etc/cron.d/*` | drop-in files, same format as `/etc/crontab` |
| `/etc/cron.{hourly,daily,weekly,monthly}/` | scripts run by `run-parts`; filename must have no dots |

The `/etc/cron.d` variant is the nicest for anything managed by a config tool, since each job
gets its own file:

```bash
# /etc/cron.d/backup   (user field is mandatory here)
30 3 * * *  root  /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

## When cron is the wrong tool

If the job occasionally takes longer than its interval, needs to catch up after downtime, or
belongs to a service you already manage, a systemd timer is a better fit. Same idea, two files:

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Home backup

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Run home backup nightly

[Timer]
OnCalendar=*-*-* 03:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now backup.timer
systemctl list-timers --all          # next run times
journalctl -u backup.service -n 50   # output, no log plumbing needed
```

`Persistent=true` reruns a missed timer after boot. Output goes to the journal, which is free.

## Done when

- `crontab -l` shows the job.
- It produced a log line at the right time (wait for it, or temporarily set it to `* * * * *`).
- Removing the job is one `crontab -e` away.

## Sources

- [`man 5 crontab`](https://man7.org/linux/man-pages/man5/crontab.5.html) — the authoritative
  field and quirk reference.
- [crontab.guru](https://crontab.guru/) — schedule translator, good for the weird ones.
- [Arch Wiki: cron](https://wiki.archlinux.org/title/Cron) — solid comparison of cron
  implementations and their differences.
- [`man 5 systemd.timer`](https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html)
  — the calendar syntax for timers.
