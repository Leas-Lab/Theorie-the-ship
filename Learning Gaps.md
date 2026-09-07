---
tags: [python, meta, learning]
---
# Learning Gaps

> Honest list, not a pep talk. Back to [[Python MOC]]

## Correcting your own estimate

You say you can **read** Python but not write it fluently. After going
through [[M321 - The Ship]]: that is not true anymore.

That project contains generators with `yield`, background threads that get
checked for whether they died, a `queue.Queue` for passing data between
threads, a `deque` with a max length used as a ring buffer, `try/finally`
so the laser always switches off, `time.monotonic()` instead of
`time.time()`, and a hand written velocity calculation that aims ahead of a
moving target. On top of that, a clean split between "what should happen"
and "how do I talk to the ship", and a secret file that is correctly kept
out of git.

That is not beginner work. It is solid middle ground with a few spikes
well above it. Your problem is not ability. Your problem is that you do
not believe yourself.

## What is actually missing

- [ ] **Type hints** (`def f(x: int) -> list[str]:`, `Optional`).
      They appear zero times in `the-ship`. Coming from Java you will
      like them immediately, and with dicts like `{"x": ..., "y": ...}`
      they would help a lot.
- [ ] **Classes and `dataclass`**. Everything in the project is functions
      and raw dicts. A small `@dataclass class Position: x: float; y: float`
      would be clearer in several places.
- [ ] **`if __name__ == "__main__":`**. Missing in all four tasks. Two
      lines, instantly better.
- [ ] **Tests with `pytest`**. You did module M450 so you know testing in
      theory. Never done it in Python. `aim_point()` would be a perfect
      first unit test: pure maths, no network needed.
- [ ] **`logging` instead of `print`**. `print` is everywhere. `logging`
      gives you levels, timestamps, and an off switch.
- [ ] **`asyncio`**. You deliberately avoided it and that was right. Worth
      looking at eventually. See
      [[Concurrency - Threads Queues Generators]].
- [ ] **OAuth2 and token handling in Python**. Coming up anyway, see
      [[OAuth2 and Keycloak]].
- [ ] **Standard library**: `pathlib`, `itertools`, `functools`. You
      already know `collections` because you used `deque`.
- [ ] **Finishing things.** The email bot is dead, `laser.py` is a draft,
      the README no longer matches the code. That is the real pattern.

## Where the project stands

Trading and the RabbitMQ scanner are **done**, and the scanner now
actually connects to the broker. The setup story is in
[[RabbitMQ Setup and the guest User]].

Two things are still open, in this order:

**1. Finish the WebSocket task** (`communication.py`). The sending half is
missing, so you and Nicol cannot actually exchange messages yet.
`SEND_EVERY`, `last_sent` and `already_seen` are already sitting there.
They are not dead code, they are your own placeholder. Blueprint is in
[[M321 - The Ship]].

**2. Keycloak** (`laser.py`). The flow in the code is written, what is
missing is the authorisation server behind it. Concept, diagrams and a
checklist: [[OAuth2 and Keycloak]].

## Quick wins in the-ship

Small, concrete, half an hour total:

1. Add `websocket-client` to `requirements.txt`. It is missing even though
   `communication.py` imports it.
2. Fix the README commands: `python -m aufgaben.introduction`, not
   `python -m Aufgaben.aufgabe_1`. Folders and files were renamed, the
   README was not. The laser task is missing from it entirely.
3. Copy the `is_alive()` check from `scanner.py` into
   `communication.py`. If the listener thread dies there, your main loop
   spins forever doing nothing.
4. Sort out the port clash: RabbitMQ management sits on NodePort 2015 and
   the OAuth URLs also point at 2015. Do this before setting up Keycloak.
5. Delete or clearly label `k8s/rabbitmq.yaml` and the `localhost` part of
   the README, because they contradict your real `local_config.py`.

## The suggestion

Not another tutorial. One small script that actually gets **finished**:
read your `WochenJournal` folder and print a summary of your mood per
quarter. It uses `pathlib`, comprehensions, `collections.Counter`, type
hints and a `@dataclass`. Doable in one evening, and it closes four gaps
at once.

## Related
- [[Python Basics]]
- [[Data Structures - Python vs Java]]
- [[Closures and Decorators]]
- [[Concurrency - Threads Queues Generators]]
- [[RabbitMQ Setup and the guest User]]
- [[OAuth2 and Keycloak]]
- [[M321 - The Ship]]
