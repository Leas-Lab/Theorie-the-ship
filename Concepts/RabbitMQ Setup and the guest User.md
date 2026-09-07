---
tags: [python, concept, rabbitmq, amqp, m321, ops]
status: from-your-code
---
# RabbitMQ Setup and the guest User

> **Based on** task 2 in [[M321 - The Ship]] and the 403 error Nicol hit on
> his `.41` machine. Back to [[Python MOC]]

## The error

```
pika.exceptions.ProbableAuthenticationError: ConnectionClosedByBroker:
(403) 'ACCESS_REFUSED - Login was refused using authentication mechanism PLAIN.'
```

First thing to notice: this is a **403, not "connection refused"**. That
difference tells you immediately where to look.

| Error | Means |
|---|---|
| `Connection refused` or timeout | Nobody is listening. Wrong port, service not running, firewall. |
| **403 ACCESS_REFUSED** | The broker is there and answered you. It just does not want to let you in. |

So a 403 is actually good news. Network and port are fine. It is only the
login.

## Why it happens

Two things combine.

**First:** the code never passes any login details.

```python
pika.ConnectionParameters(host=consume_host, port=consume_port, socket_timeout=5)
```

No `credentials=`. So pika quietly uses the default: **`guest` / `guest`**.

**Second:** RabbitMQ has a safety setting called `loopback_users`. Out of
the box it is:

```
loopback_users = [guest]
```

In plain words: the user `guest` may only log in **from the same machine**.
The reason is obvious once you hear it. Everybody on the planet knows the
username `guest` and the password `guest`. If that worked over the network,
every freshly installed broker would be wide open within minutes.

Nicol's Python ran on his Windows PC (`.48`), the broker on the VM (`.41`).
That is a connection from outside. So: 403.

```
.48  Python, logs in as guest
 |
 |  AMQP
 v
.41  RabbitMQ:  "guest from outside? no."   403
```

## The three possible fixes

### A) Make your own user

On the broker:
```bash
sudo rabbitmqctl add_user ship ship123
sudo rabbitmqctl set_permissions -p / ship ".*" ".*" ".*"
```
In Python:
```python
credentials = pika.PlainCredentials("ship", "ship123")
pika.ConnectionParameters(host=..., port=..., credentials=credentials)
```

Cleanest option, and the only correct one in the real world. The three
`".*"` are regex patterns for *configure*, *write* and *read*, so it means
"allowed to do everything".

**Why it was ruled out:** it needs a change to `scanner_commands.py`, and
that file is given to you as a template.

### B) SSH tunnel

```bash
ssh -L 2014:127.0.0.1:2014 ship@192.168.101.41
```

Then you point `consume_host` at `127.0.0.1`. The tunnel carries your
traffic over to the VM, and the broker sees a connection arriving
**locally**. The loopback rule no longer applies, so `guest` is allowed in.

Elegant, changes nothing on the broker, disturbs nobody else.

**Why it was ruled out:** the task requires the real `.41` address in the
config, not `127.0.0.1`.

### C) Turn off the loopback restriction on the broker

In `/etc/rabbitmq/rabbitmq.conf` on the VM:
```
listeners.tcp.default = 2014
loopback_users = none
```
then
```bash
sudo systemctl restart rabbitmq-server
```

`loopback_users = none` means there is no longer any user restricted to
localhost. `guest` may connect from anywhere.

**This is what Nicol did**, and given the constraints it was the only
option left. Unchanged code plus the real IP in the config leaves nothing
else. If the client cannot move and the config cannot move, the broker has
to move.

The other line, `listeners.tcp.default = 2014`, is a separate matter. AMQP
normally runs on **5672**. The task wants 2014, so the listener has to be
moved.

## Side by side

| | A: own user | B: SSH tunnel | C: loopback_users = none |
|---|---|---|---|
| Change the code | yes | no | no |
| IP in the config | stays `.41` | becomes `127.0.0.1` | stays `.41` |
| Change the broker | yes, add a user | no | yes, edit conf and restart |
| Affects other people | no | no | **yes**, applies to everyone |
| Fine for production | yes | emergency only | **no** |
| Fits the task rules | no | no | **yes** |

## Why containers just work

Worth knowing, or you will never understand why the same setup works in
one place and fails in another.

The official Docker image `rabbitmq:3-management` ships with a default
config that contains:

```
loopback_users.guest = false
```

So the restriction is **already switched off**. That is why `guest`/`guest`
works from anywhere against a broker running in Docker or Kubernetes, while
the exact same client gets a 403 against one installed with
`apt install rabbitmq-server`.

Same software, different default. Not magic, just one line in a config
file.

## How to diagnose, in the right order

Always work from the bottom up. Never guess.

```bash
# 1. Is the service even running?
sudo systemctl status rabbitmq-server

# 2. Is it listening on the right port?
sudo ss -lntp | grep 2014
sudo rabbitmq-diagnostics listeners

# 3. Can the client reach it?              (Windows)
Test-NetConnection 192.168.101.41 -Port 2014

# 4. Am I allowed to log in?
sudo rabbitmqctl list_users
sudo rabbitmqctl list_permissions -p /
sudo rabbitmqctl authenticate_user guest guest
```

Step 3 green and step 4 red is exactly your 403. Step 3 red is a
completely different problem and you would be wasting your time looking at
users.

If nothing helps, the broker log always says why it refused you:
```bash
sudo journalctl -u rabbitmq-server -n 50
```

## The security note

`loopback_users = none` in plain words means: **anybody on the network who
types guest/guest has full access to the broker.** In an isolated school
network for an exercise, fine. In a company network, or anything reachable
from the internet, that is a disaster. RabbitMQ warns about it in its own
docs.

The correct order in real life: own user, only the permissions actually
needed, TLS on top, and delete `guest`.

## Related
- [[M321 - The Ship]], task 2
- [[Concurrency - Threads Queues Generators]], why the consumer runs in a thread
- [[OAuth2 and Keycloak]], the same theme one level up: 401 versus 403


