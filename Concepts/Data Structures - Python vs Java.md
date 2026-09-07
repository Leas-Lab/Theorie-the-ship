---
tags: [python, java, concept, data-structures]
status: from-your-notes + reference
---
# Data Structures - Python vs Java

> **From your notes:** `WochenJournal/2026/KW - 17.md`, where you wrote
> *"I can tell you the difference between Array, ArrayLists and Lists now.
> I also know what a datastructure is."* The rest below is expanded
> reference. Back to [[Python MOC]]

## The core idea in one sentence

An **array** has a fixed size, an **ArrayList** grows as you add things,
and a **Python list** is basically an ArrayList that also does not care
what type you put in it.

## The comparison

| | Java `int[]` | Java `ArrayList<E>` | Python `list` |
|---|---|---|---|
| Size | fixed when created | grows | grows |
| Type | one type, checked at compile time | one type via generics | anything, mixed |
| Read by index | `a[0]`, instant | `get(0)`, instant | `a[0]`, instant |
| Append | not possible | `add()` | `append()` |
| Under the hood | one solid block of memory | an array that doubles when full | an array of pointers that doubles when full |

The important bit: a Python list is **not** a real array. It stores
pointers to objects, not the values themselves. That is why it can mix
types, and also why it is slower and uses more memory. If you need real
arrays you use `array.array` or NumPy.

## The four Python types you need

```python
my_list  = [1, 2, 3]          # ordered, changeable, duplicates allowed
my_tuple = (1, 2, 3)          # ordered, CANNOT be changed
my_set   = {1, 2, 3}          # unordered, no duplicates, very fast "in"
my_dict  = {"a": 1, "b": 2}   # key to value, keeps insertion order
```

Java translation:

| Python | Java |
|---|---|
| `list` | `ArrayList` |
| `tuple` | no real equivalent, closest is a `record` |
| `set` | `HashSet` |
| `dict` | `HashMap`, but with guaranteed ordering |

## The trap that will get you exactly once

```python
def add(item, target=[]):      # BROKEN
    target.append(item)
    return target

add(1)   # [1]
add(2)   # [1, 2]   <- surprise
```

Default values are created **once**, when Python reads the `def` line, not
every time you call the function. So every call shares the same list.

The fix:

```python
def add(item, target=None):
    if target is None:
        target = []
    target.append(item)
    return target
```

## What "data structure" actually means

A data structure is a decision about **which operation you want to be
cheap**. A list makes reading by index cheap and searching expensive. A
set and a dict make searching cheap and do not care about order.

You pick based on what you do most often.

```python
if name in my_list:   # has to walk through everything
if name in my_set:    # jumps straight there via a hash
```

With 10 items nobody notices. With 100000 items and 50000 lookups, the
list version takes minutes and the set version takes a blink.

