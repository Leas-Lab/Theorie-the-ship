---
tags: [python, project, m321, distributed-systems]
status: from-real-code
source: ~/Documents/School/M321/the-ship
---
# M321 - The Ship

> **Everything here was read out of the real code**, not guessed.
> Module M321, programming distributed systems. Partner: Nicol.
> Back to [[Python MOC]]

## Where it stands

| Task | Topic | Status |
|---|---|---|
| Trading | HTTP request and response | **done** |
| Scanner | RabbitMQ / AMQP | **done and running** |
| Communication | WebSocket | **in progress**, sending half missing |
| Laser | **Keycloak** / OAuth2 | **next up** |

## What the project is

A **spaceship simulator**. Your ship runs as a server on its own IP in the
school network (yours is `192.168.101.40`, Nicol's is `.41`) and you
control it from outside using different protocols. Each task deliberately
uses a different way of talking to it.

## How the code is organised

Two layers, cleanly separated. This is the best design decision in the
whole project.

```
aufgaben/          <- WHAT should happen (the steps, the logic)
  introduction.py       Task 1: trading
  scanner.py            Task 2: RabbitMQ and holding position
  communication.py      Task 3: WebSocket
  laser.py              Task 4: Keycloak and mining

functionalities/   <- HOW you talk to the ship (thin API wrappers)
  steering_commands.py       set_target, wait_until_in_reach
  communication_commands.py  pos, stations_in_reach, buy, sell
  scanner_commands.py        detected_objects  (AMQP)
  laser_commands.py          configure_oauth, activate, deactivate, set_angle, state
  cargo_commands.py          hold

config.py          <- table of endpoints, builds URLs from my_ip
local_config.py    <- your IP, ports, secret. NOT in git.
k8s/rabbitmq.yaml  <- RabbitMQ as a Kubernetes deployment
```

Every function in `functionalities/` follows the same three steps:

```python
def stations_in_reach():
    response = requests.get(command["stations_in_reach"])
    response.raise_for_status()
    return response.json()
```

`raise_for_status()` everywhere. That makes a failed call complain loudly
instead of silently returning nonsense. Exactly right.

The ports, all on your ship:

| Port | What for |
|---|---|
| 2009 | set_target |
| 2011 | pos, stations_in_reach, buy, sell |
| 2012 | hold (cargo bay) |
| 2018 | laser and OAuth |
| 2026 | WebSocket |
| 2014 | RabbitMQ AMQP |

---

## Task 1: Trading (`introduction.py`)  `done`

**Pattern: request and response over HTTP.** The simplest case. You ask,
the ship answers, you carry on.

Buy IRON at **Azura Station**, sell it at **Core Station**, in growing
amounts: 4, 8, 12. Then set course for (7000, 7000).

```python
amounts = [4, 8, 12]
for i, amount in enumerate(amounts):
    print(buy_at_azura(amount))
    if i < len(amounts) - 1:
        print(sell_at_core(amount))
```

**The bit that is easy to miss:** the `if i < len(amounts) - 1` means the
last 12 IRON never get sold. The ship leaves with a full hold. That was on
purpose, the README says so.

---

## Task 2: Scanner with RabbitMQ (`scanner.py`)  `done and running`

**Pattern: publish and subscribe. Asynchronous and decoupled.**

Find **G-Station 1-4** and stay within range of it for 60 seconds. The
catch: the station **moves**.

### The messaging part

```python
EXCHANGE = "scanner/detected_objects"

def detected_objects():
    connection = pika.BlockingConnection(...)
    channel = connection.channel()
    channel.exchange_declare(exchange=EXCHANGE, exchange_type="fanout")
    queue = channel.queue_declare(queue="", exclusive=True).method.queue
    channel.queue_bind(exchange=EXCHANGE, queue=queue)
    for method_frame, properties, body in channel.consume(queue=queue, auto_ack=True):
        yield json.loads(body.decode("utf-8"))
```

Four decisions are hidden in there, and all four are correct:

1. **`fanout`**: every subscriber gets every message. For a sensor feed you
   do not want messages split between listeners, otherwise you and Nicol
   would each see half the data.
2. **`queue=""` with `exclusive=True`**: RabbitMQ invents a random name and
   throws the queue away when you disconnect. A disposable subscription
   rather than a rubbish heap that grows forever.
3. **`auto_ack=True`**: no manual confirmation. That means **at most
   once**, so a message is allowed to get lost. For a live sensor that is
   right, because a position from five seconds ago is worthless. For an
   order it would be wrong.
4. **`yield` instead of `return`**: this makes it a **generator**. It hands
   out messages one at a time, forever, without ever holding them all in
   memory.

### The chase

The genuinely clever part: you do not aim at where the station was last
seen. You work out where it is going.

```python
samples = deque(maxlen=64)   # ring buffer of (position, timestamp)

def aim_point():
    here, now = history[-1]
    for earlier, when in reversed(history[:-1]):
        span = now - when
        if span >= VELOCITY_WINDOW:      # 1.0 s
            return {
                "x": here["x"] + (here["x"] - earlier["x"]) / span * LEAD_SECONDS,
                "y": here["y"] + (here["y"] - earlier["y"]) / span * LEAD_SECONDS,
            }
```

Two measurements give you a speed. Multiply by 1.5 seconds of lead and you
aim at where the station **will be**. Same idea as leading a moving target
when shooting clay pigeons. Nobody writes that by accident.

**Full RabbitMQ concept + code walkthrough for the exam:**
[[RabbitMQ - How It Works and the Code]].

### Getting it to actually connect

The hard part was not the Python. It was **where the broker lives and who
is allowed to log in**. Full story:
[[RabbitMQ Setup and the guest User]].

Short version: `scanner_commands.py` passes no `credentials=`, so pika
quietly logs in as **`guest`/`guest`**. By default RabbitMQ only lets
`guest` connect **from the same machine**. Your Python runs on your PC, the
broker runs elsewhere, so you get:

```
pika.exceptions.ProbableAuthenticationError:
(403) 'ACCESS_REFUSED - Login was refused using authentication mechanism PLAIN.'
```

That is the wall Nicol hit on his `.41`. There are three ways around it,
and the task rules cut off two:

| Option | Why it does not work here |
|---|---|
| Own user plus `PlainCredentials` | Needs a change to `scanner_commands.py`, which is a fixed template |
| SSH tunnel to `127.0.0.1` | Then the real `.41` is no longer in the config, which is required |
| **`loopback_users = none` on the broker** | The one that is left, and it works |

What Nicol did, on the VM `.41`, in `/etc/rabbitmq/rabbitmq.conf`:

```
listeners.tcp.default = 2014
loopback_users = none
```
then `sudo systemctl restart rabbitmq-server`.

The first line moves AMQP off its normal port 5672 onto the 2014 the task
asks for. The second line removes the localhost restriction on `guest`.

### Where the broker actually is

Here the repository contradicts reality, and it matters:

| Source | Says |
|---|---|
| `README.md`, line 118 | `consume_host = "localhost"  # RabbitMQ runs on your own machine` |
| `k8s/rabbitmq.yaml` | RabbitMQ deployed on your own machine, NodePort 2014 |
| **`local_config.py`, the real one** | `consume_host = my_ip`, which is **192.168.101.40**, your ship |

The third one is what actually gets used. So your setup is the same shape
as Nicol's: broker on the ship VM, client on your own computer.

**Reasoning, not proven:** it has to be that way, because the simulator
itself publishes into the `scanner/detected_objects` exchange. A broker
sitting on your laptop would never receive anything, because the ship has
no idea where your laptop is. So `k8s/rabbitmq.yaml` and the `localhost`
section of the README are most likely leftovers from an earlier attempt.

Open:
- [ ] What exactly happened on `.40`? Did you configure it yourself over
      SSH, was it already set up, or does the broker run in Docker there
      (in which case the restriction is off by default)?
- [ ] Delete `k8s/rabbitmq.yaml` and the `localhost` part of the README, or
      label them clearly as an alternative setup, so nobody is misled.

### Threads

```python
scanner = threading.Thread(target=watch_for_station, daemon=True)
scanner.start()
```

The generator blocks while waiting on the socket. In the main thread you
could never steer at the same time. `daemon=True` means the thread gets
killed when the program ends, so nothing hangs on exit.

And the part most people forget:

```python
if not scanner.is_alive():
    raise SystemExit(f"Scanner dead, no AMQP to {consume_host}:{consume_port}")
```

A thread that dies silently is the ugliest class of bug there is. You check
for it. Good.

### Measuring time

`time.monotonic()`, not `time.time()`. A monotonic clock never runs
backwards, not even during a clock sync or a daylight saving change. For
"hold for 60 seconds" that is the only correct choice.

---

## Task 3: WebSocket (`communication.py`)  `in progress`

**Pattern: a permanently open connection, both directions.**

Fly to your assigned station (Elyse Terminal for you, Shangris Terminal for
Nicol), then listen over a WebSocket for what it sends.

```python
ws = create_connection(command["websocket"])      # ws://<ship>:2026/api
threading.Thread(target=listen, args=(ws,), daemon=True).start()

while True:
    while not inbox_from_station.empty():
        message = inbox_from_station.get()
        print("from the station:", message["destination"], len(message["msg"]))
    time.sleep(0.1)
```

**Why `queue.Queue` and not a normal list?** Because two threads touch it.
`queue.Queue` is thread safe, a list is not guaranteed to be. This is the
standard pattern: producer thread puts in, main thread takes out.

You deliberately used **`websocket-client`** (synchronous, threads) rather
than `websockets` (asyncio). Both are fine. Threads fit better here because
`requests` is synchronous everywhere else in the project.

### What is still missing: the sending half

Right now this is a **receiver**, not a conversation. You read what comes
in but never send anything out. The task wants both: you send Nicol
something, he gets it, and the other way round.

The three lines you already wrote and then left sitting there are exactly
the placeholder for this:

```python
SEND_EVERY = 3          # how often to send, in seconds
already_seen = set()    # to avoid handling the same message twice
last_sent = 0           # when we last sent something
```

The blueprint:

```python
while True:
    # receiving (you already have this)
    while not inbox_from_station.empty():
        message = inbox_from_station.get()
        print("from the station:", message["destination"], len(message["msg"]))

    # sending (missing)
    if time.monotonic() - last_sent >= SEND_EVERY:
        ws.send(json.dumps({
            "destination": partner_station,
            "msg": "hello Nicol",
        }))
        last_sent = time.monotonic()

    time.sleep(0.1)
```

Four things that will trip you up:

1. **The message format.** You know from receiving that a message has
   `destination` and `msg`. Use the exact same fields when sending. If it
   does not work, print one whole received message (`print(message)`) and
   copy the shape.
2. **Where to?** `destination` has to point at Nicol's terminal, not
   yours. Yours is Elyse, his is Shangris. That belongs in
   `local_config.py`.
3. **`last_sent` with `time.monotonic()`**, not `time.time()`. Same reason
   as in the scanner.
4. **`already_seen`** matters once you are both sending: if the same
   message arrives twice (a reconnect, a broadcast), you do not want to
   handle it twice. That needs the message to have an ID. If it does not, a
   hash of `(destination, msg)` will do.

**The point of the whole task:** this is where "both directions at once"
becomes real. With HTTP you would have to keep asking "anything new?" every
few seconds. With a WebSocket the line stays open and either side speaks
whenever it wants. Your listener thread and your sending loop run at the
same time over **one** connection.

Open:
- [ ] Confirm the field names when sending
- [ ] Move the partner target into `local_config.py`
- [ ] Either use `already_seen` or delete it, do not leave it half done
- [ ] Add the `is_alive()` check for the listener thread, like the scanner
      has

---

## Task 4: Laser with Keycloak (`laser.py`)  `next up`

Current state is the commit `81aef5e "Laser: draft"`. **This is the
Keycloak task.** The steps are written, what is missing is the
authorisation server behind them.

New compared to everything before: **authentication**. `/activate` on port
2018 answers with **403** unless you have a valid OAuth token. So first
`configure_oauth()` with a `client_secret`, an `authorize_url` and a
`token_url`, and only then are you allowed to fire.

These two URLs are the ones that will point at Keycloak:

```python
"authorize_url": f"http://{oauth_host}:2015/authorize",
"token_url":     f"http://{oauth_host}:2015/token",
```

Details on the concept: [[OAuth2 and Keycloak]].

> **Careful, port clash.** RabbitMQ management sits on NodePort **2015** in
> `k8s/rabbitmq.yaml`, and the OAuth URLs also point at **2015**. As long
> as those are different hosts you are fine. On the same machine you are
> not. Note it now, or you will spend an hour on it next week.

The plan: fly to Arakrock, point the laser at angle 0, mine STONE until the
hold is full, then fly to Vesta Station.

Three places where this code is more grown up than task 1:

```python
try:
    while True:
        ...
finally:
    deactivate()          # the laser ALWAYS switches off, even on a crash
```

```python
laser = state()
if laser["is_cooling_down"]:
    print("laser cooling down")
elif not laser["is_active"]:
    activate()            # ask the state instead of blindly commanding
```

```python
def ship_position():
    data = pos()
    if "x" in data:
        return data
    return next(v for v in data.values() if isinstance(v, dict) and "x" in v)
```

That last one handles `/pos` answering in two different shapes. `next(...)`
with a generator expression grabs the first matching dict. Compact, but it
throws `StopIteration` if nothing matches at all.

Also good: the placeholder check at the top that saves you from a
confusing 403.

---

## The thread running through all of it

| | Task 1 | Task 2 | Task 3 | Task 4 |
|---|---|---|---|---|
| Pattern | request and response | publish and subscribe | both directions | request and response with auth |
| Technology | HTTP | AMQP / RabbitMQ | WebSocket | HTTP plus OAuth via Keycloak |
| Who starts | the client | the server pushes | either side | the client |
| Coupling | tight | loose | permanent | tight |
| Delivery | exactly once | at most once (`auto_ack`) | a stream | exactly once |
| If the other side dies | exception | queue goes stale | connection breaks | 403 or exception |
| Status | done | done | in progress | next up |

That table is the one to have in your head for the exam.

---

## Honest code review

**What is good**
- The split between `aufgaben/` and `functionalities/`. Clean and reusable.
- `raise_for_status()` in **every** wrapper. Consistent.
- `local_config.py` in `.gitignore` with an example file as a template.
  That is exactly how you handle secrets.
- `time.monotonic()`, `deque(maxlen=)`, `queue.Queue`, a generator with
  `yield`, `try/finally`. These are not beginner tools.
- The lead calculation in the scanner is real original work.

**What is broken or sloppy**

1. **`requirements.txt` is incomplete.** It only lists `requests` and
   `pika`. `communication.py` does `from websocket import
   create_connection`, which is **`websocket-client`**, and it is not in
   the file. A fresh clone cannot run task 3.
   Fix: add `websocket-client==1.9.0`.

2. **The README does not match the code.** It says `python -m
   Aufgaben.aufgabe_1`, but since commit `979d757` the folder is lowercase
   `aufgaben/` and since `f19c8f5` the files are called `introduction.py`,
   `scanner.py`, `communication.py`. On Linux and in Docker, capital
   letters matter, so that command fails. The laser task is missing from
   the README entirely.
   It should read: `python -m aufgaben.introduction` and so on.

3. **The sending half of `communication.py` is unfinished.** `SEND_EVERY`,
   `last_sent` and `already_seen` are not dead code, they are the
   placeholder for the part that is missing. See task 3.

4. **`except Exception as e: print(e); return`** in the listener thread.
   The thread dies quietly and the main program then spins forever doing
   nothing. `scanner.py` checks `is_alive()`, this one does not.
   Inconsistent.

5. **No `if __name__ == "__main__":`.** All four tasks run at module level,
   which means importing one starts the ship. Fine for school, but the
   guard costs two lines.

6. **Port 2015 is double booked.** RabbitMQ management and the OAuth URLs
   both want it. Sort it out before Keycloak.

7. **The README and `k8s/rabbitmq.yaml` contradict the real
   `local_config.py`.** The README sends you to `localhost`, the actual
   config uses `192.168.101.40`. Anyone cloning this repo would build a
   broker that never receives anything.

---

## Python you can demonstrably do

Generators (`yield`), threads with `daemon=True`, `queue.Queue` for passing
data between threads, `collections.deque` with `maxlen`, `try/finally` for
cleanup, `enumerate`, `next()` with a generator expression, `math.hypot`,
`raise SystemExit`, packages and imports, `response.raise_for_status()`,
keeping configuration out of the code.

See [[Concurrency - Threads Queues Generators]].

