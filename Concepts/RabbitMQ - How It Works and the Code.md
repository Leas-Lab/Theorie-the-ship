---
tags: [python, concept, rabbitmq, amqp, m321, exam-prep]
status: from-your-code
source: ~/Documents/School/M321/the-ship
---
# RabbitMQ - How It Works and the Code

> **Exam-prep page.** Everything here is built to be said out loud to a
> teacher: what RabbitMQ actually is, then line by line what task 2 does
> with it. The 403/guest-user story has its own page:
> [[RabbitMQ Setup and the guest User]]. This page is the "explain the
> whole thing" page. Back to [[Python MOC]]

## Where this sits in the project

Sorted the same way as the four aufgaben, because that is how the exam
will probably go:

| Aufgabe | Protocol | RabbitMQ involved? |
|---|---|---|
| 1: Trading (`introduction.py`) | HTTP | no |
| **2: Scanner (`scanner.py`)** | **AMQP via RabbitMQ** | **yes, this page** |
| 3: Communication (`communication.py`) | WebSocket | no |
| 4: Laser (`laser.py`) | HTTP + OAuth2 | no |

So RabbitMQ is only in task 2. But task 2 is the one where you have to be
able to explain a whole extra piece of infrastructure (the broker), not
just a Python trick. That is what this page is for.

---

## Part 1: What RabbitMQ actually is

**The one-sentence version:** RabbitMQ is a **message broker**. It sits
between programs and passes messages from senders to receivers, so the two
sides never have to talk to each other directly or even know about each
other.

### Why not just call an API directly?

With HTTP (like task 1) the client asks a direct question and waits for a
direct answer. That only works if:
- you know exactly who to ask
- that one server is currently up
- you are fine with asking again and again to check for new data

The scanner does not fit that. The simulator does not know who wants
scanner data, might have zero or many listeners, and produces updates
continuously without being asked. That is a **publish/subscribe** problem,
not a request/response problem. RabbitMQ is built for exactly that.

### The three core pieces

```
Publisher --> [ Exchange ] --> [ Queue ] --> Consumer
                   |
              (routing rule)
```

- **Exchange**: the mailroom. Messages arrive here first. An exchange
  decides *where* a message goes based on its type.
- **Queue**: the actual mailbox. Messages sit here until a consumer picks
  them up.
- **Binding**: the rule connecting an exchange to a queue ("send copies of
  everything to this queue").
- **Publisher**: whoever sends a message into an exchange. In this
  project, the ship simulator itself.
- **Consumer**: whoever reads messages out of a queue. That is you,
  `scanner_commands.py`.

Crucial point for the exam: **the publisher never sends directly to a
queue.** It always sends to an exchange, and the exchange, guided by
bindings, decides which queue(s) get a copy. That indirection is the whole
reason RabbitMQ is more flexible than "just open a socket".

### Exchange types (know all four, you used one)

| Type | Behaviour |
|---|---|
| **`fanout`** | Ignores routing keys entirely. Every bound queue gets a copy of every message. Broadcast. |
| `direct` | Message goes only to queues bound with a matching exact routing key. |
| `topic` | Like direct, but the routing key can use wildcards (`sensor.*`). |
| `headers` | Routing decided by message headers instead of a key. Rare. |

The scanner uses **`fanout`**:
```python
channel.exchange_declare(exchange=EXCHANGE, exchange_type="fanout")
```
Why fanout is correct here: a scanner reading is not "for" any particular
listener. It is a broadcast of "here is what I currently see". If you and
Nicol were both listening, `fanout` means you would **both** get every
detection. A `direct` or `topic` exchange would need routing keys that
this problem does not actually have.

### AMQP, in one paragraph

AMQP (Advanced Message Queuing Protocol) is the actual wire protocol
RabbitMQ speaks, the same relationship HTTP has to a web server. `pika` is
the Python library that speaks AMQP for you, so you never build raw AMQP
frames by hand, you just call `pika.BlockingConnection(...)`.

---

## Part 2: The code, line by line

This is `functionalities/scanner_commands.py`, the only file in the whole
project that touches RabbitMQ directly:

```python
import json
import pika
from local_config import consume_host, consume_port

EXCHANGE = "scanner/detected_objects"

def detected_objects():
    connection = pika.BlockingConnection(
        pika.ConnectionParameters(host=consume_host, port=consume_port, socket_timeout=5)
    )
    channel = connection.channel()
    channel.exchange_declare(exchange=EXCHANGE, exchange_type="fanout")
    queue = channel.queue_declare(queue="", exclusive=True).method.queue
    channel.queue_bind(exchange=EXCHANGE, queue=queue)
    for method_frame, properties, body in channel.consume(queue=queue, auto_ack=True):
        yield json.loads(body.decode("utf-8"))
```

Walking through it exactly in order:

1. **`pika.BlockingConnection(pika.ConnectionParameters(host=..., port=..., socket_timeout=5))`**
   Opens the TCP connection to the broker and speaks the AMQP handshake.
   `BlockingConnection` means this call sits and waits (blocks) rather
   than being async. `socket_timeout=5` gives up after 5 seconds instead
   of hanging forever if nothing answers.
   *No `credentials=` is passed here* — that omission is exactly what
   causes the guest-user 403 story, see [[RabbitMQ Setup and the guest User]].

2. **`connection.channel()`**
   A connection can carry several independent channels, like several
   phone lines over one cable. You only ever need one here. All the real
   work (declaring, binding, consuming) happens on the channel, not
   directly on the connection.

3. **`channel.exchange_declare(exchange=EXCHANGE, exchange_type="fanout")`**
   "Make sure an exchange called `scanner/detected_objects` exists, and
   it is a fanout." `declare` is **idempotent** — if it already exists
   with the same settings, nothing bad happens, it just confirms it is
   there. This is also why the exchange type has to match on both ends;
   the simulator that publishes into it must have declared it the same
   way.

4. **`channel.queue_declare(queue="", exclusive=True).method.queue`**
   This is the part most people misread on first glance. `queue=""` means
   "you (the broker) invent a name for me" — RabbitMQ generates something
   like `amq.gen-XyZ123`. `exclusive=True` means: only this connection may
   use this queue, and the queue is **deleted automatically** the moment
   this connection closes.
   Why that matters: you do not want a permanent queue sitting on the
   broker collecting scanner data nobody is reading anymore every time you
   restart your script. A temporary, self-cleaning, private queue is
   exactly right for "give me a live feed while I'm running".
   `.method.queue` is just how pika hands back the generated name from the
   broker's response — you need that name for the next step.

5. **`channel.queue_bind(exchange=EXCHANGE, queue=queue)`**
   This creates the binding: "connect my private queue to that fanout
   exchange." No routing key needed, fanout does not use one. From this
   point on, every message the simulator publishes into the exchange also
   lands in your queue.

6. **`channel.consume(queue=queue, auto_ack=True)`**
   Starts pulling messages off your queue. This returns an **iterator**
   that keeps producing `(method_frame, properties, body)` tuples forever,
   one per message, blocking and waiting whenever the queue is empty.
   `auto_ack=True` means the broker considers a message delivered the
   instant it hands it to you, no manual confirmation needed. See the
   delivery guarantees section below for why that is the right call here.

7. **`yield json.loads(body.decode("utf-8"))`**
   `body` arrives as raw bytes, so `.decode("utf-8")` turns it into a JSON
   string, then `json.loads` turns that into a Python dict. `yield`
   instead of `return` is what makes `detected_objects()` a **generator**
   — see [[Concurrency - Threads Queues Generators]] for the full
   explanation of why that matters, but the short version: this stream
   never ends, so you cannot collect it into a list first, you hand out
   one detection at a time and pause in between.

### How the rest of task 2 consumes this

`aufgaben/scanner.py` never touches pika directly. It only calls
`detected_objects()`:

```python
def watch_for_station():
    for objects in detected_objects():
        for obj in objects:
            if obj["name"] == TARGET_STATION:
                samples.append((obj["pos"], time.monotonic()))
```

This runs in its own background **thread**, because `detected_objects()`
blocks while waiting for the next message, and the main thread needs to
keep steering the ship at the same time. Each message from the broker
appears to contain a **list** of detected objects (`for obj in objects`),
so one scanner "tick" can report several stations at once. Only the one
you actually care about, `"G-Station 1-4"`, gets kept, together with a
timestamp — that timestamp is what later lets `aim_point()` work out the
station's velocity and aim ahead of it.

### Delivery guarantees — the concept the teacher will probably poke at

| Mode                                            | Meaning                                                                                                                                          | Used here? |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- |
| `auto_ack=True`                                 | Broker marks the message as delivered instantly. If your code crashes right after receiving it, that message is gone for good. **At most once.** | **yes**    |
| `auto_ack=False` + manual `channel.basic_ack()` | You confirm each message yourself once you have actually processed it. If you crash first, the broker redelivers it. **At least once.**          | no         |

Why `auto_ack=True` is the right call for a scanner feed and would be
wrong for, say, a "buy order" queue: a scanner reading is only useful for
about a second. If one gets lost because your process happened to crash at
the wrong instant, the next reading half a second later replaces it
completely. Losing an order, on the other hand, loses money. Different
data, different guarantee needed.

---

## Part 3: Where the broker itself lives

`k8s/rabbitmq.yaml` deploys RabbitMQ as a **Kubernetes Deployment plus a
Service**:

```yaml
image: rabbitmq:3-management
ports:
  - containerPort: 5672    # AMQP, the actual message protocol
  - containerPort: 15672   # the web management UI
```
```yaml
type: NodePort
ports:
  - name: amqp
    port: 5672
    targetPort: 5672
    nodePort: 2014          # what you actually connect to
  - name: management
    port: 15672
    targetPort: 15672
    nodePort: 2015           # http://localhost:2015, guest/guest
```

Three port numbers, three different jobs, worth being able to say clearly:
- **5672** — the real AMQP port, inside the cluster.
- **2014** — the `NodePort`, i.e. the port actually reachable from outside
  the cluster. This is what `consume_port` in `local_config.py` is set to.
- **2015** — same idea but for the web dashboard (`15672` inside, `2015`
  outside).

`rabbitmq:3-management` specifically (not the plain `rabbitmq:3` image) is
what gives you the web UI on 15672 in the first place — plain RabbitMQ
only speaks AMQP, no browser dashboard.

**Important nuance, already flagged in [[M321 - The Ship]]:** the
`local_config.py` that is actually used points `consume_host` at your ship
IP (`192.168.101.40`), not `localhost`. So in reality the broker used by
the exercise sits on the ship VM, not in a local Kubernetes cluster you
deployed yourself. `k8s/rabbitmq.yaml` is most likely an earlier, unused
attempt at running your own local broker — good to know so you do not
tell your teacher the wrong thing.

---

## Part 4: The two-minute spoken explanation

If asked "what does RabbitMQ do in your project", the compressed version:

> "The simulator publishes scanner detections into a RabbitMQ exchange.
> I don't ask for the data, I subscribe to it. My code opens a connection
> with `pika`, creates a temporary private queue, binds it to that
> exchange, and then reads a live, endless stream of detections through a
> generator. It runs on a background thread so the ship can keep steering
> at the same time.  kmI use `auto_ack=True` because losing one scan reading
> occasionally doesn't matter, the next one arrives half a second later
> anyway."

That single paragraph covers: broker, exchange, queue, binding, why
publish/subscribe fits here, `pika`, threading, and delivery guarantees.
Everything below is there so you can go deeper on any piece of it if asked
a follow-up.

---


<details>
<summary>Answers</summary>

1. An exchange receives messages and decides *where* they go; a queue is
   where messages actually wait to be picked up by a consumer.
2. Fanout broadcasts to every bound queue with no routing key needed.
   A scanner reading is not addressed to anyone specific, it is a
   broadcast of "here is what I currently see" — exactly fanout's job.
3. It asks the broker to generate a random queue name and to delete that
   queue automatically once this connection closes. Right choice because
   you only want a live feed while your script runs, not a permanent
   queue nobody empties after you disconnect.
4. Because the stream of detections never ends. `yield` hands out one
   value and pauses, so you never try to hold "all of them" in memory,
   which would never finish anyway.
5. `auto_ack=True` means the broker considers a message delivered the
   moment it hands it over, no confirmation needed. With `auto_ack=False`
   and no `basic_ack()`, the broker keeps re-delivering the same
   unacknowledged message and never lets it go, or it piles up unacked.
6. Because `detected_objects()` blocks while waiting on the socket. If it
   ran in the main loop, the ship could never steer while waiting for the
   next scanner reading.
7. 5672 is the real AMQP port inside the Kubernetes cluster. 2014 is the
   NodePort, the port actually reachable from outside the cluster, and
   the one `consume_port` in `local_config.py` points at.
8. `scanner_commands.py` passes no `credentials=`, so pika defaults to
   `guest`/`guest`. RabbitMQ's default `loopback_users = [guest]` only
   allows that user to log in from the same machine as the broker. Since
   the client and broker are on different machines, that is a 403.
   Nicol fixed it with `loopback_users = none` on the broker, the only
   one of the three options that fit the task's rules (code and IP could
   not change).
9. `exchange_declare` is idempotent: if an exchange with that exact name
   and type already exists, RabbitMQ just confirms it, it does not create
   a duplicate or error out. It only fails if you tried to redeclare it
   with *different* settings (e.g. `fanout` the second time when it's
   already `direct`).
10. Both of you get everything. That is the entire point of `fanout` —
    it ignores routing keys and copies every message to every queue
    that is bound to it. Each of you has your own private `exclusive`
    queue bound to the same exchange, so the exchange fans the same
    message out to both queues independently.
11. The generator pauses on that line and the thread blocks on the
    underlying socket read, using no CPU, until a message arrives or the
    connection times out/breaks. Because this runs in a background
    thread with the GIL released during the wait, the main thread is
    free to keep calling `set_target()` and steering. The moment a
    message arrives, pika hands it back through the loop as the next
    `(method_frame, properties, body)` tuple and the `yield` line hands
    it onward to whoever is iterating `detected_objects()`.

</details>

## Related
- [[RabbitMQ Setup and the guest User]] — the 403 story and the fix, in full
- [[Concurrency - Threads Queues Generators]] — why the scanner needs a
  thread and a generator at the same time
- [[M321 - The Ship]] — task 2 in the context of the whole project
- [[Python MOC]]
