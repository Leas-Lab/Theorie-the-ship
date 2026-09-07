---
tags: [python, concept, concurrency, threading]
status: from-your-code
source: M321 the-ship
---
# Concurrency - Threads, Queues, Generators

> **From your code.** Every pattern on this page is in your
> [[M321 - The Ship]] project. This is not theory, you built it.
> Back to [[Python MOC]]

## The problem

A network call blocks. While you sit there waiting for a message, you
cannot do anything else. For a ship that has to listen **and** steer at
the same time, that is fatal.

Python has three answers. You used all three.

## 1. Generator: an endless stream

```python
def detected_objects():
    for method_frame, properties, body in channel.consume(queue=queue, auto_ack=True):
        yield json.loads(body.decode("utf-8"))
```

`yield` instead of `return` turns the function into a **generator**. It
hands out one value, then **pauses**, and next time it carries on from
exactly that line. Its local variables survive the pause.

Why that is right here: the stream never ends. With `return` and a list
you would have to collect everything first, which never finishes and eats
all your memory.

You call it like anything else:
```python
for objects in detected_objects():
    ...
```

**Rule of thumb:** the moment you think "and then the next one, and then
the next one, possibly forever", you want a generator.

## 2. Thread: two things at once

```python
scanner = threading.Thread(target=watch_for_station, daemon=True)
scanner.start()
```

`target` is the function that runs in the background. `start()` kicks it
off and your own code carries on immediately.

**`daemon=True`** matters. A daemon thread gets killed when the program
ends. Without that flag, Python waits for the thread to finish, and a
thread with `while True` never finishes. Your program hangs on exit.

### The trap: dying quietly

If a thread hits an exception, your main program **hears nothing**. It
keeps looping forever, wondering why no data arrives.

You caught this in `scanner.py`:
```python
if not scanner.is_alive():
    raise SystemExit(f"Scanner dead, no AMQP to {consume_host}:{consume_port}")
```
You did not do it in `communication.py`. That is the one inconsistency in
the project.

### The GIL, in one paragraph

Python has a **Global Interpreter Lock**: only one thread runs actual
Python code at a time. So threads give you **no extra computing power**.

But the moment a thread waits for input or output, network or disk, it
lets go of the lock. Which is exactly why your threads work fine here:
they spend almost all their time waiting.

Remember it as: **threads for waiting, processes for calculating.**

## 3. Queue: a safe handover point

```python
inbox_from_station = queue.Queue()

def listen(ws):                      # thread B puts things in
    inbox_from_station.put(json.loads(ws.recv()))

while True:                          # thread A takes things out
    while not inbox_from_station.empty():
        message = inbox_from_station.get()
```

Two threads touching the same data structure is normally a race waiting to
corrupt something. `queue.Queue` has the locking built in, so you do not
need to write your own.

Do **not** just use a list and hope. It works most of the time, and "most
of the time" is the worst kind of bug.

## 4. Ring buffer: keep only the last N

```python
from collections import deque
samples = deque(maxlen=64)
samples.append((position, time.monotonic()))
```

With `maxlen`, the oldest entry falls out automatically. No manual
cleanup. And removing from the front of a `deque` is instant, while
`list.pop(0)` has to shuffle every remaining element down one slot.

## 5. Measuring time properly

```python
time.monotonic()   # right for "how long has this been going"
time.time()        # right for "what date is it"
```

`time.time()` can jump: a clock sync, daylight saving, someone setting the
clock by hand. Then your stopwatch suddenly reads minus three seconds.
`time.monotonic()` only ever goes forward.

## And asyncio?

The fourth answer, which you deliberately did not use. `asyncio` does the
same job with `async`/`await` in a single thread. It scales better when
you have hundreds of connections, but it is contagious: once one function
is `async`, everything that calls it has to be `async` too. Since
`requests` is not async, using it would have caused friction for no gain.

Your call was correct. Still worth learning eventually, see
[[Learning Gaps]].
