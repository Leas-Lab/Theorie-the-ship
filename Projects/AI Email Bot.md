---
tags: [python, project, ai]
status: reconstruction
---
# AI Email Bot

> **From your notes:** `WochenJournal/2026/KW - 11.md`, where you wrote
> *"I am making an automatic Email bot with an AI to learn Python."*
> No code found anywhere. The rest is reconstruction.
> Back to [[Python MOC]]

## Context from the journal

That was the week you officially started your main project, spent a lot of
time waiting on Björn, and picked up a side project to fill the gap. The
email bot was explicitly a **learning project**, not an assignment. Your
own note on what went wrong that week: no real contact person, no
direction.

## What a bot like that needs

Three pieces, all either standard library or one package:

| Piece | Tool |
|---|---|
| Reading mail | `imaplib` and `email` (both built in) |
| Sending mail | `smtplib` and `email.message.EmailMessage` |
| Writing the reply | an HTTP call to an LLM API, so `requests` |

```python
import imaplib, email

mail = imaplib.IMAP4_SSL("imap.gmail.com")
mail.login(USER, APP_PASSWORD)
mail.select("inbox")

_, ids = mail.search(None, "UNSEEN")
for msg_id in ids[0].split():
    _, data = mail.fetch(msg_id, "(RFC822)")
    message = email.message_from_bytes(data[0][1])
    print(message["Subject"])
```

## What you actually learn from it

Exactly the things a tutorial never teaches properly:

- **Handling secrets**: the password goes in a `.env` file, read with
  `os.environ` or `python-dotenv`, and `.env` goes in `.gitignore`.
- **Bytes versus strings**: IMAP hands you bytes. This is where everybody
  trips.
- **Encoding**: mail headers are MIME encoded, so
  `email.header.decode_header` is your friend.
- **Rate limits and cost**: a bot that polls every second is expensive and
  will get blocked.
- **Not replying twice** to the same email.

## The honest warning

A bot with write access to a real mailbox, driven by a language model, can
send real emails to real people. For a learning project: write to the
drafts folder instead of sending, and use a test mailbox. Not your work
account.

## Open
- [ ] Does any of this code still exist? If so, where?
- [ ] Which model or API was the plan?
- [ ] Is the project dead or just paused?
