---
layout: post-no-feature
title: Hashing it Out
description: A few questions to help students understand hash tables.
comments: true
hidden: true
date: 2020-07-07
category: articles
---

In my experience, when students learn data structures they (1) learn how they work theoretically (2) use off-the-shelf data structures to solve problems and (3) sometimes implement these data structures from scratch. However, I don't often see classes exploring and experimenting with these off-the-shelf implementations, and the interesting practical considerations that come into play.

When I was a TA for [6.006](https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-006-introduction-to-algorithms-fall-2011/), I had students answer these questions about Python dictionaries. I hope they were a fun way to better understand hash tables and experiment with them "in the wild." I definitely learned a lot while preparing this!

## Problems

### Problem 1

Using a Python program, determine how the size of a dictionary changes as you insert keys. How does this affect insertion time, and why?

### Problem 2

Let the _compactness factor_ of a dictionary be \\(\text{(size of dictionary entries)}\\) divided by \\(\text{(total size of the dictionary)}\\). What is the maximum compactness factor of a Python dictionary, and how does that compare to the load factor? How does this change over different versions of Python, and why? It might help to read the [CPython source](https://github.com/python/cpython). 

### Problem 3

Consider object `A`, which returns a deterministic hash value in the range \\([1, n]\\). For various values of \\(n\\) and \\(k\\), compute how long it takes to insert \\(k\\) `A(n)` objects into a dictionary. How long does this take in terms of \\(n\\) and \\(k\\), and does this match what you theoretically expect? How does this compare to the "real world," and what explains the difference?


```python
import random    

class A:
    def __init__(self, n):
        self.hash = random.randint(1, n)    

    def __hash__(self):
        return self.hash
```

### Problem 4

Object `B` differs from `A` in that we return a random integer in \\([1, n]\\) on **every** call to `__hash__`. What's so bad about this? Provide two examples of problematic code, using `B` objects and dictionaries, and explain what goes wrong.

```python
import random    

class B:
    def __init__(self, n):
        self.n = n    

    def __hash__(self):
        return random.randint(1, self.n)
```

## Solutions

Here are brief responses to these questions: note that I'm running these on my Macbook Pro using Python 3.7.7.

Recall that Python implements dictionaries using a hash table with [open addressing](https://en.wikipedia.org/wiki/Open_addressing) (here's the [source code](https://github.com/python/cpython/blob/master/Objects/dictobject.c)). It uses an underlying array that stores key-value pairs, and Python uses hash values to "probe" through this array and find a slot for each key. If you don't want to read through all the code, there are a bunch of resources explaining the internals of a Python dictionary, like [this](https://stackoverflow.com/questions/327311/how-are-pythons-built-in-dictionaries-implemented) and [this](https://just-taking-a-ride.com/inside_python_dict/chapter1.html).

### Problem 1

First, we can use the following code to measure when the size of a Python dictionary changes:

```python
import sys

d = {}
N = 1000

last_size = sys.getsizeof(d)

for i in range(N):
    d[i] = "foo"
    if sys.getsizeof(d) > last_size:
        print("key", i, "new size", sys.getsizeof(d))
        last_size = sys.getsizeof(d) 
```

This prints out the following, indicating that the table size **doubles** on certain insertions (it isn't exact because dictionaries have a 96-byte overhead).

```
key 5 new size 376
key 10 new size 656
key 21 new size 1192
key 42 new size 2288
key 85 new size 4712
key 170 new size 9328
key 341 new size 18536
key 682 new size 36976
```

We can also measure insertion time: here, we add \\(N\\) values to a dictionary, recording how long it takes to add each key. We add up the times over multiple trials to reduce noise:

```python
import time

N = 1000

# times[i] stores how long it takes to add 
# i to the dictionary
times = [0] * N

# Run multiple trials to reduce noise
for _ in range(10000):
    d = {}
    # For each i, measure how long it takes
    # to add i and add the result to times[i]
    for i in range(N):
        start_time = time.time()
        d[i] = "foo"
        t = time.time() - start_time
        times[i] += t
  
# Find the 5 values of i that took the longest 
# time to insert
top = sorted(list(range(N)), key=lambda i: -times[i])
for i in top[:5]:
    print(i, times[i])
```

This returns the following:

```
682 0.027842044830322266
341 0.015052318572998047
170 0.0094757080078125
85 0.006536006927490234
42 0.0048770904541015625
```

Unsurprisingly, these correspond to when the dictionary size changes! This is because Python needs to allocate a new array and reinsert all existing keys into that array, which explains why insertion into a Python dictionary is actually **amortized** \\(\Theta(1)\\) in expectation.

### Problem 2

First, looking at [dict-common.h](https://github.com/python/cpython/blob/3.7/Objects/dict-common.h) and [pyport.h](https://github.com/python/cpython/blob/3.7/Include/pyport.h), we can deduce that the size of a dictionary entry is 24 bytes. 

At this point, you could run `24 * len(d) / sys.getsizeof(d)` right before table doubling to determine the compactness factor. On Python 2, this returns \\(2/3\\), which matches the load factor stated in [dictobject.c](https://github.com/python/cpython/blob/3.7/Objects/dictobject.c). 

On Python 3.7, however, this ratio is \\(8/9\\) for smaller dictionary sizes, and \\(4/5\\) for larger dictionary sizes. The load factor is actually still \\(2/3\\), but [Python dictionaries have gotten more compact](https://mail.python.org/pipermail/python-dev/2016-September/146327.html). At a high-level, if a dictionary has capacity \\(n\\), then it'll store \\(2/3n\\) 24-byte entries that are indirectly accessed through \\(n\\) _indices_. The size of these indices depends on the the value of \\(n\\): 1 byte if \\(n < 2^8 = 256\\), 2 bytes if \\(n < 2^{16}= 65,536\\) and so on. So for a dictionary with capacity \\(2^8 \le n < 2^{16}\\), for example, the maximum compactness factor will be \\(24 \cdot (2/3 n) / (24 \cdot (2/3 n) + 2 \cdot n) = 8/9\\). 

### Problem 3

If we insert \\(k\\) keys into a dictionary with \\(n\\) hash values, we expect \\(k/n\\) keys to be assigned to each hash value. For each value of \\(n\\), inserting these \\(k/n\\) values should take \\(1 + 2 + \dots + k/n = \Theta(k^2/n^2)\\) time. This is because every time we add a new key, we need to probe through all existing keys with the same hash value before we find an open slot. Adding over all \\(n\\), the time it takes to insert all \\(k\\) keys should be \\(\Theta(k^2/n)\\) in expectation. Note that probe sequences can overlap, so this isn't 100% accurate.

We can verify this using the following code, which adds \\(k\\) `A(n)` objects to a table and measures how long it takes:

```python
import time 

def create_dict(n, k):
    d = {}
    s = time.time()
    for i in range(k):
        d[A(n)] = i
    return time.time() - s
```

First, for a few values of \\(n\\) we can see how insertion time varies with \\(k\\). Indeed, a regression will verify that insertion time increases quadratically with \\(k\\).

<img src="/blog/images/chart1.svg" style="width:min(100%, 800px)">

Second, for a fixed value of \\(k = 2000\\) we can see how insertion time varies with \\(n\\). It shouldn't be hard to verify that time is proportional to \\(1/n\\).

<img src="/blog/images/chart2.svg" style="width:min(100%, 800px)">

Cool, these are consistent with our predictions! 

So why doesn't Python run into this slowdown? In short, true collisions rarely happen in practice. Python also makes sure to use every bit of the hash value--see the `perturb` logic in [dictobject.c](https://github.com/python/cpython/blob/3.7/Objects/dictobject.c)--so it isn't as simple as taking hash values mod the length of the array. For this reason, insertion cost is dominated by different probe sequences overlapping, rather than two keys having the exact same probe sequence. [It isn't trivial](https://en.wikipedia.org/wiki/Linear_probing#Analysis), but one can prove that adding \\(k\\) keys takes \\(\Theta(k)\\) time in expectation.

### Problem 4

Here are two problematic pieces of code:

In this example, setting `d[key]` then retrieving `d[key]` usually results in a `KeyError`: if the hash value changes, Python will be unable to locate the key.  

```python    
d = {}
key = B(5)
d[key] = "foo"
print(d[key])    
```

Here, setting the value of `d[key]` a second time should overwrite the first instance. If the hash value changes, however, it will insert `key` a second time and this program will print `2`.

```python    
d = {}
key = B(5)
d[key] = "foo"
d[key] = "bar"
print(len(d))
```

Moral of the story: hash functions should be deterministic. 


