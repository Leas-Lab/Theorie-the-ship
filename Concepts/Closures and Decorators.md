---
tags: [python, concept, m323, functional]
status: from-your-notes + reference
---
# Closures and Decorators

## Your original notes

### Making a closure visible
```python
def make_power(exponent):
    def power(base):
        return base ** exponent
    return power

square = make_power(2)
print(square.__closure__)                   # shows the stored value
print(square.__closure__[0].cell_contents)  # 2
```

### A closure that remembers state
```python
def make_counter():
    count = 0
    def counter():
        nonlocal count
        count += 1
        return count
    return counter

cnt = make_counter()
cnt(), cnt(), cnt()   # 1, 2, 3
```

## What a closure is

A function that **remembers** variables from where it was born, even long
after that outer function has finished and gone.

`make_power(2)` runs, returns, and disappears. Yet `square` still knows
about the `2`. Python keeps it in something called a **cell**, and
`__closure__` is your window into that cell. So your first note does not
just show the behaviour, it shows the actual mechanism. That is a good
note.

`nonlocal` is needed because `count += 1` would otherwise create a **new
local** variable and you would get an `UnboundLocalError`. `nonlocal`
says: this name lives in the function that wraps me, use that one.

Scope memory aid: **LEGB**. Local, Enclosing, Global, Built-in. That is
the order Python searches for a name.

## From closure to decorator

A decorator is just a closure that eats a function and hands back a
function.

```python
def logged(fn):
    def wrapper(*args, **kwargs):
        print(f"-> {fn.__name__}{args}")
        result = fn(*args, **kwargs)
        print(f"<- {result}")
        return result
    return wrapper

@logged
def add(a, b):
    return a + b

add(2, 3)
```

`@logged` is nothing but shorthand for `add = logged(add)`. That is the
entire mystery. Someone replaced your function with a wrapper that calls
your function.

### Do not forget `functools.wraps`

Without it, your function is now called `wrapper` and its docstring is
gone. That breaks debugging and tooling.

```python
import functools

def logged(fn):
    @functools.wraps(fn)
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs)
    return wrapper
```

### A decorator that takes arguments

Three levels deep. It looks worse than it is: one level for the argument,
one for the function, one for the call.

```python
def retry(times):                       # takes the argument
    def decorator(fn):                  # takes the function
        @functools.wraps(fn)
        def wrapper(*args, **kwargs):   # takes the actual call
            for attempt in range(times):
                try:
                    return fn(*args, **kwargs)
                except Exception:
                    if attempt == times - 1:
                        raise
        return wrapper
    return decorator

@retry(3)
def flaky_network_call():
    ...
```

This is exactly the pattern you would want for unreliable network calls in
[[M321 - The Ship]], and for refreshing an expired token in
[[OAuth2 and Keycloak]].

## The Java comparison

Java does not have real closures over changeable variables. A lambda can
only capture variables that are effectively final. So there is no
`nonlocal` equivalent and you end up using a field or an `AtomicInteger`
instead. Your `make_counter` is noticeably uglier in Java.

