---
tags: [python, concept, reference]
status: reference
---
# Python Basics

> **Status: reference.** Not from your notes, written from scratch as a
> foundation. Tick off what you actually know.
> Back to [[Python MOC]]

## The mental model

In Java, the type is attached to the name. `int x` means x is an int,
forever, and the compiler enforces it.

In Python, a name is just a **sticky label** you put on an object. The
label knows nothing. The object knows what it is.

```python
x = 5          # the label x is stuck on an int
x = "hello"    # now the same label is stuck on a string. Perfectly legal.
```

Also: there are no curly braces. **Indentation is the syntax.** If you
indent wrong, the program means something different. It is not styling.

## The things you use every single day

### Truthiness

Empty things count as false. You will see this everywhere, so learn it
now.

```python
if not my_list:      # instead of if len(my_list) == 0
    ...
```

These are all false: `False`, `None`, `0`, `0.0`, `""`, `[]`, `{}`, `()`.
Everything else is true.

### f-strings

Put an `f` in front of the quote and you can drop variables straight in.

```python
name = "Lea"
print(f"Hi {name}, {2+2=}")   # Hi Lea, 2+2=4
```

That `=` inside the braces prints the expression **and** its value. It is
the best debugging tool Python has and almost nobody knows about it.

### Comprehensions

Python's version of Java streams, but shorter.

```python
squares   = [n * n for n in range(10)]
evens     = [n for n in numbers if n % 2 == 0]
by_id     = {u.id: u.name for u in users}
```

Read it out loud left to right: "give me n times n, for every n in range
10". That is the whole trick.

### `if __name__ == "__main__":`

Every Python file is a module. When you import a file, everything written
at the top level runs immediately. That is usually not what you want.

```python
def main():
    ...

if __name__ == "__main__":
    main()
```

This says: only run `main()` if this file was started directly, not if
somebody imported it. Two lines, saves a lot of confusion.

### Exceptions

```python
try:
    value = int(text)
except ValueError as e:
    print(f"not a number: {e}")
else:
    print("worked")     # only runs if nothing was raised
finally:
    print("always runs")
```

Python culture is **"try it and catch the error"** rather than "check
everything first". The official name is EAFP, easier to ask forgiveness
than permission. In Java you would check first. In Python you just go.

### `with`

```python
with open("data.json", encoding="utf-8") as f:
    content = f.read()
```

`with` closes the file for you, even if something crashes in the middle.
Same idea as try-with-resources in Java.

## Environment and packages

```bash
python3 -m venv .venv          # create an isolated environment
source .venv/bin/activate      # turn it on (macOS/Linux)
pip install pika               # install packages into it
pip freeze > requirements.txt  # write down what you installed
```

One virtual environment per project. Always. Otherwise in a year your
system Python has 200 packages that fight each other.

