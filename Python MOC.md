---
tags: [python, moc]
---
# Python MOC

This is the map. Everything else hangs off this page.

## Concepts
- [[Python Basics]]
- [[Data Structures - Python vs Java]]
- [[Closures and Decorators]]
- [[Concurrency - Threads Queues Generators]]
- [[RabbitMQ Setup and the guest User]]
- [[RabbitMQ - How It Works and the Code]]
- [[OAuth2 and Keycloak]]  <- next topic

## Projects
- [[M321 - The Ship]]  <- the main one, real code
- [[AI Email Bot]]

## Meta
- [[Learning Gaps]]

---

## Where this stuff comes from

| Source | What came out of it |
|---|---|
| `~/Documents/School/M321/the-ship` | All the real code: 4 tasks, 5 command modules, RabbitMQ setup |
| `Archived/M323(In)/Closures und Dekoratoren.md` | Your original closure notes, kept and expanded |
| `WochenJournal/2026/KW - 11.md` | "automatic Email bot with an AI to learn Python" |
| `WochenJournal/2026/KW - 17.md` | Array vs ArrayList vs List, Python next to Java |
| Nicol's RabbitMQ chat | The whole guest user and 403 story |

**Not Python**, even though it might look like it: the ÜK tasks (Generate
Name, Wetterstation, Bitcoin calculator, TODO) are all **Vue**.

Only `Python Basics` is pure reference written from scratch. Everything
else is tied to your real code or your real notes.

## Random fact
Python's `nonlocal` keyword only arrived in Python 3.0, back in 2008.
Before that, people faked it by putting a counter inside a list
(`count = [0]`, then `count[0] += 1`), because you were allowed to change
what was *inside* a list but not to rebind the name itself. You still find
that hack in old code today.
