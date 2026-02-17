---
title: "Part 1: From yield to Async IO: How Python Turns Functions Into Schedulers"
metaDescription: 
date: 2026-02-17T02:00:00.000Z
summary: How yield turns into async programming via cooperative scheduling.
tags:
  - python
  - yield
  - generators
  - coroutines
  - asyncio
---

We write multi-threaded programs to make better use of modern hardware and multi-process programs to distribute work across machines. Purpose of both is to enhance the performance at the expense of simplicity. With co-routines the programs thought process is more naturally expressed as a series of cooperative processes.

**Coroutines Python2:**

The beginning of coroutines was when generators were added to the python. Generators are lazily evaluated list. In normal function you would use return to get the value from called function but with generators you use yield keyword. More specifically with return, the function is done with the work but with yield, it emits a value and pause its execution. This continues until caller calls the next on generator. At which point the generator resumes its execution after the yield statement.
So basically lifecycle in generator is something like: emitting a value -> suspends execution -> returns flow back to the calling code.

```python

def square(limit):
    for i in range(limit):
        yield i * i
```

Up until now, yield was used as a keyword to emit the value. It has no capability to take input. After sometime the changes were introduced:
- yield, which was previously a statement, was redefined to be an expression.
- Added a send() method to inject inputs during execution.
- Added a throw() method to inject exceptions.
- Added a close() method to allow the caller to terminate a generator early.

Let's understand each in detail:
1. Send:

```python

def gen():
    value = yield "ready"
    print("Received:", value)

g = gen() # intialize generator
print(next(g)) # "ready"
g.send("hello") # Resume and send value
```

2. throw:
```python
def gen():
    try:
        yield "running"
    except ValueError:
        print("Caught ValueError inside generator")

g = gen()
print(next(g))              # "running"

g.throw(ValueError)
```

3. close:
```python
def gen():
    try:
        yield 1
        yield 2
    finally:
        print("Cleaning up...")

g = gen()
print(next(g))   # 1
g.close() # Cleaning up, not yielding 2 as we closed the generator
```

Now try to understand the below code:
```python
import itertools

def averager():

    sum = float((yield)) # emit none and expect an input from calling code via send
    counter = itertools.count(start=1) # initializing the counter
    while True:
        sum += (yield sum / next(counter)) # emits the average and expects the next value via send.

avg = averager()
next(avg)
avg.send(10) # 10
avg.send(20) # 15
avg.send(30) # 20
```

**Python 3.3: "yield from"**
The generator can yield only to its immediate caller and if you want to split generators up for a reasons of code re-use and modularity, the calling generator would have to manually iterate the sub-generator and re-yield all its results.
```python
# Implementation in pre-3.3 Python
def chain(*generators):
    for generator in generators:
        for item in generator:
            yield item

# Implementation in post-3.3 Python
def chain(*generators):
    for generator in generators:
        yield from generator
```

Let's take an example:
```python

def parse_number():
    digits = []

    while True:
        ch = yield
        if ch.isdigit():
            digits.append(ch)
        else:
            break

    return int("".join(digits))


def parser():
    while True:
        result = yield from parse_number()
        print("Parsed number: ", result)


p = parser()
next(p)
for c in "123x45y":
    p.send(c)


```

Basically it means: "Suspend me. The child is now the generator. Route EVERYTHING through it until it finishes."

It automatically:
- forwards every .send()
- forwards every .throw()
- propagates exceptions
- captures child return value
- resumes parent afterward
- handles GeneratorExit
- handles close()

All in one line.


#### The state of Python coroutines: Introducing asyncio

AsyncIO has a handy event loop for scheduling coroutines. Asyncio gives you a ready-made event loop that can run code in two styles:
- Old-school callbacks (functions triggered when something happens)
- Modern coroutines (async/await) that look blocking but actually aren't

Either way, the same event loop schedules everything. And even if you don't care about networking or files, the coroutine scheduler itself is valuable, it lets you coordinate many tasks without writing your own scheduler.

- asyncio isnt just about async IO.
- It provides a general-purpose cooperative task scheduler.

**Callback style:**
```python
loop.call_later(...)
```

**Coroutine style:**
```python
await something()
```
The loop pauses and resumes tasks automatically.

Let me show you an example of cooperative scheduling using only one OS thread. The below example shows you can run multiple long-running background jobs concurrently with one thread.

```python

import asyncio
import datetime
import errno
import os
import sys
from pathlib import Path

def rotate_file(path, keep_versions):
    """
    Create versions of the file and promote 1 to 2, 2 to 3, 3 to 4.
    """
    path = Path(path)
    if not path.exists():
        return

    for i in range(keep_versions, 1, -1):
        old = path.with_suffix(path.suffix + f".{i-1}")
        new = path.with_suffix(path.suffix + f".{i}")
        if old.exists():
            old.rename(new)
        
    path.rename(path.with_suffix(path.suffix + ".1"))

@asyncio.coroutine
def rotate_by_interval(path, keep_versions, rotate_secs):
    """
    Rotate file every N seconds.
    """
    while True:
        yield from asyncio.sleep(rotate_secs)
        rotate_file(path, keep_versions)

@async.coroutine
def rotate_daily(path, keep_versions):
    """
    Rotate file every midnight.
    """
    while True:
        now = datetime.datetime.now()
        last_midnight = now.replace(hour=0, minute=0, second=0)
        next_midnight = last_midnight + datetime.timedelta(1)
        yield from asyncio.sleep((next_midnight - now).total_seconds())
        rotate_file(path, keep_versions)

@async.coroutine
def rotate_by_size(path, keep_versions, max_size, check_interval_secs):
    """
    Rotate file when it exceeds N bytes checking every M seconds.
    """
    while True:
        yield from asyncio.sleep(check_interval_secs)
        try:
            filesize = Path(path).stat().st_size
            if filesize > max_size:
                rotate_file(path, keep_versions)
        except FileNotFoundError:
            pass

def main(argv):
    loop = asyncio.get_event_loop()

    rotate1 = loop.create_task(rotate_by_interval("/tmp/file1", 3, 30))
    rotate2 = loop.create_task(rotate_by_interval("/tmp/file2", 5, 20))
    rotate3 = loop.create_task(rotate_by_size("/tmp/file3", 3, 1024, 60))
    rotate4 = loop.create_task(rotate_daily("/tmp/file4", 5))
    loop.run_forever()

if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

All of this happens using:
- one OS thread
- one asyncio event loop
- cooperative scheduling via yield from asyncio.sleep(...)

You can see that main() just creates a bunch of tasks and plugs them into an event loop, then asyncio takes care of the scheduling.  This approach is quite modular and manages to produce single-threaded code where different asynchronous operations interoperate with little or no awareness of each other. 

I will be writing part 2 soon, which will go in depth in other asyncio primitives like callbacks based and await based. You can find this whole code [here](https://github.com/ayush-garg341/python/tree/master/coroutines)

The above code is not mine and I just tweaked it a little. You can find more about original author and post [here](https://www.andy-pearce.com/blog/posts/2016/Jun/the-state-of-python-coroutines-introducing-asyncio/).