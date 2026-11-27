---
title: Telegram bot
tags: [guide, telegram, notifications, linux]
difficulty: easy
time: 20 min
date: 2026-11-27
---

# Telegram bot

A Telegram bot is the cheapest push channel a self-hosted setup can get: no email deliverability,
no app to publish, no vendor account. Ten minutes from nothing to "your backup finished" buzzing
in your pocket.

It's also a small remote control — a long-polling script can read messages and act on them.

## 1. Create the bot

1. In Telegram, message [@BotFather](https://t.me/BotFather).
2. `/newbot`, give it a display name and a username ending in `bot`.
3. BotFather replies with a token shaped like `123456789:AAHx...`.

Treat that token as a password: anyone holding it can read the bot's messages and send as it.

## 2. Find your chat ID

A bot **cannot** message you first — you have to open a conversation with it. Send it anything,
then ask the API what it received:

```bash
curl -s "https://api.telegram.org/bot${TG_TOKEN}/getUpdates" | jq '.result[0].message.chat.id'
```

That number is your chat ID. Send the bot a message in a **group** and the ID will be negative
(groups look like `-1001234567890`). For groups, add the bot as a member and check that it is
allowed to read messages, or use `/start@yourbot` in the group.

If the result is empty, the bot has never received anything — send it a message and retry. If
someone else talked to the bot since, look further down `.result`.

## 3. Send a message

One HTTP POST, nothing else:

```bash
curl -sS -X POST "https://api.telegram.org/bot${TG_TOKEN}/sendMessage" \
  --data-urlencode "chat_id=${TG_CHAT_ID}" \
  --data-urlencode "text=Backup finished" \
  --data-urlencode "parse_mode=HTML"
```

`--data-urlencode` matters: it handles spaces, ampersands and newlines without you thinking about
it. The API answers `{"ok":true,...}` on success and `{"ok":false,"description":"..."}` on
failure — which is a genuinely useful error message, unlike most HTTP APIs.

## 4. Keep the token out of your scripts

One env file, mode 600, never in git:

```bash
mkdir -p ~/.config/tgnotify
cat > ~/.config/tgnotify/env <<'ENV'
TG_TOKEN=123456789:AAHx...
TG_CHAT_ID=987654321
ENV
chmod 600 ~/.config/tgnotify/env
```

Then a three-line wrapper that anything else can call:

```bash
#!/usr/bin/env bash
# ~/bin/tg — usage: tg "message"   or   command | tg
set -euo pipefail
. ~/.config/tgnotify/env

msg="${1:-$(cat)}"
curl -sS -X POST "https://api.telegram.org/bot${TG_TOKEN}/sendMessage" \
  --data-urlencode "chat_id=${TG_CHAT_ID}" \
  --data-urlencode "text=${msg}" \
  --data-urlencode "parse_mode=HTML" \
  --data-urlencode "disable_web_page_preview=true" \
  | jq -e '.ok' >/dev/null
```

Now notifications are one word, which is the whole point — a notification you're too lazy to
write is a notification you don't get:

```bash
tg "backup done"

./long-build.sh 2>&1 | tail -20 | tg

# in a cron job, on failure only
30 3 * * * /usr/local/bin/backup.sh || tg "backup FAILED on $(hostname)"
```

## 5. Make it readable

`parse_mode=HTML` supports a small, safe subset — `<b>`, `<i>`, `<u>`, `<s>`, `<code>`, `<pre>`,
`<a href="...">`. MarkdownV2 exists too but requires escaping a long list of characters, which
is exactly the kind of fiddly you don't want in a one-line notification.

HTML mode still needs `&`, `<` and `>` escaped in interpolated text:

```bash
esc() { printf '%s' "$1" | sed 's/&/\&amp;/g; s/</\&lt;/g; s/>/\&gt;/g'; }
tg "<b>Backup failed</b>%0A<pre>$(esc "$(tail -5 err.log)")</pre>"
```

(A literal newline in `text` works too; `%0A` is only needed when passing URL-encoded strings by
hand.)

## 6. Reading messages, lightly

For a "send me a message, I do a thing" bot, long polling is enough — no public IP, no TLS
certificate, no webhook:

```python
#!/usr/bin/env python3
import os, time, requests

token = os.environ["TG_TOKEN"]
base = f"https://api.telegram.org/bot{token}"
offset = None

while True:
    r = requests.get(f"{base}/getUpdates", params={"timeout": 30, "offset": offset}, timeout=40).json()
    for upd in r.get("result", []):
        offset = upd["update_id"] + 1
        msg = upd.get("message") or {}
        text = msg.get("text", "")
        if text == "/ping":
            requests.post(f"{base}/sendMessage", data={"chat_id": msg["chat"]["id"], "text": "pong"})
    time.sleep(1)
```

Two rules: always advance `offset` (or you get the same message forever), and check
`message.chat.id` against an allow-list if the bot is discoverable — otherwise strangers can
drive it.

## 7. Gotchas

- **4096 characters** maximum per message. Long logs need splitting or `tail`-ing first.
- **Rate limits:** roughly 30 messages per second to one chat, and about 20 per minute to a group.
  Irrelevant for cron jobs, relevant if you loop over a file.
- **Token leaked?** `/revoke` in BotFather gives you a new one; update the env file.
- **Empty response / 400s** are usually an unescaped `&` in HTML mode, or a wrong chat ID.
- **The bot can't start a conversation.** If nothing arrives, you never messaged it first.

## Done when

`tg "hello"` produces a phone notification, the token is in a 600-mode file outside any repo, and
a failed cron job sends exactly one message.

## Sources

- [Telegram Bot API reference](https://core.telegram.org/bots/api) — `sendMessage`, `getUpdates`,
  and the formatting rules.
- [Bot features overview](https://core.telegram.org/bots/features) — long polling vs webhooks,
  rate limits.
- [BotFather](https://t.me/BotFather) — the bot that creates bots.
