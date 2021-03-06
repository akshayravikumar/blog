---
layout: post-no-feature
title: Hashing it Out
description: A deep dive into Python dictionaries.
comments: true
visible: true
date: 2020-07-10
category: articles
---

In my experience, when students learn data structures they (1) learn how they work theoretically (2) use off-the-shelf data structures to solve problems and (3) sometimes implement these data structures from scratch. However, I don't often see classes exploring the off-the-shelf implementations, and the interesting practical considerations that come along with them.

When I was a TA for [6.006](https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-006-introduction-to-algorithms-fall-2011/), I had students answer the following questions about Python dictionaries. I hope they were a fun way to experiment with hash tables "in the wild"--I definitely learned a lot while preparing this! Solutions are at the end.

## Background

Recall that Python implements dictionaries using a hash table with [open addressing](https://en.wikipedia.org/wiki/Open_addressing). Long story short, dictionaries have an underlying array that stores key-value pairs. When inserting a key, Python uses hash values to "probe" through this array in a specific sequence until it finds an open slot for that key. This post assumes you have a basic understanding of open addressing. If you'd like a refresher on hash table concepts, [this](https://runestone.academy/runestone/books/published/pythonds/SortSearch/Hashing.html) might help.

Also, the internals of a Python dictionary aren't a mystery: you can check out the [source code](https://github.com/python/cpython/blob/master/Objects/dictobject.c)! If you don't want to read through everything, there are a bunch of resources explaining Python dictionaries, like [this](https://stackoverflow.com/questions/327311/how-are-pythons-built-in-dictionaries-implemented) and [this](https://just-taking-a-ride.com/inside_python_dict/chapter1.html). 

## Problems

### Problem 1

Use a Python program to determine how the size of a dictionary, in bytes, changes as you insert keys. How does this affect insertion time, and why?

### Problem 2

Assume you have inserted \\(e\\) entries into a hash table that has the capacity to store \\(c\\) entries. Define the _load factor_ of the hash table as \\(e/c\\). In addition, define the _compactness factor_ of the hash table as (number of bytes allocated to the \\(e\\) entries) divided by (number of bytes allocated to the entire table). 

Recall that the size of a hash table periodically increases to keep the load factor below a certain threshold. What is the maximum compactness factor of a Python dictionary, and how does that compare to the maximum load factor? How has this changed over different versions of Python, and why? It might help to read the [CPython source](https://github.com/python/cpython). 

### Problem 3

Consider object `A`, which returns a random integer in the range \\([1, n]\\) on **every** call to `__hash__`. What's so bad about this? Provide two examples of problematic code using `A` objects and dictionaries, and explain what goes wrong.

```python
import random    

class A:
    def __init__(self, n):
        self.n = n    

    def __hash__(self):
        return random.randint(1, self.n)
```


### Problem 4

Object `B` differs from `A` in that we return a **deterministic** hash value in the range \\([1, n]\\). For various values of \\(n\\) and \\(k\\), compute how long it takes to insert \\(k\\) `B(n)` objects into a dictionary. Does this match what you theoretically expect? How does this compare to the "real world," and what explains the difference?


```python
import random    

class B:
    def __init__(self, n):
        self.hash = random.randint(1, n)    

    def __hash__(self):
        return self.hash
```

## Solutions

Here are brief responses to these questions: unless otherwise stated, I'm running these on my Macbook Pro using Python 3.7.7.

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

Unsurprisingly, these correspond to when the dictionary size doubles! This is because Python needs to allocate a new array and reinsert all existing keys into that array, which explains why insertion into a Python dictionary is actually **amortized** \\(\Theta(1)\\) in expectation.

### Problem 2

First, looking at [dict-common.h](https://github.com/python/cpython/blob/3.7/Objects/dict-common.h) and [pyport.h](https://github.com/python/cpython/blob/3.7/Include/pyport.h), we can deduce that the size of a dictionary entry is 24 bytes. Therefore, given a dictionary `d`, its compactness factor is equal to `len(d) * 24 / sys.getsizeof(d)`. This ratio is maximized right before the dictionary resizes. To measure this, we can modify our script from Problem 1:

```python
import sys

d = {}
N = 100000

last_size = sys.getsizeof(d)

for i in range(N):
    compactness = len(d) * 24.0 / sys.getsizeof(d)
    d[i] = "foo"
    if sys.getsizeof(d) > last_size:
        print("key", i, "compactness", compactness)
        last_size = sys.getsizeof(d) 
```

On Python 2.7, this prints the following:

```
key 5 compactness 0.428571428571
key 21 compactness 0.480916030534
key 85 compactness 0.608591885442
key 341 compactness 0.651177593889
key 1365 compactness 0.66272859686
key 5461 compactness 0.665677948885
key 21845 compactness 0.666419223299
key 87381 compactness 0.666604789308
```

The compactness factor clearly approaches \\(2/3\\), which matches the load factor stated in [dictobject.c](https://github.com/python/cpython/blob/3.7/Objects/dictobject.c). (Also, note that different versions of Python have different table resizing schemes.)

On Python 3.7, this prints the following:

```
key 5 compactness 0.4838709677419355
key 10 compactness 0.6382978723404256
key 21 compactness 0.7682926829268293
key 42 compactness 0.8456375838926175
key 85 compactness 0.8916083916083916
key 170 compactness 0.865874363327674
key 341 compactness 0.8773584905660378
key 682 compactness 0.8830384117393181
key 1365 compactness 0.8859800951968845
key 2730 compactness 0.8874200888503629
key 5461 compactness 0.888160034695869
key 10922 compactness 0.8885213005396317
key 21845 compactness 0.8887065715603049
key 43690 compactness 0.7999243224109415
key 87381 compactness 0.7999627701453185
```

Weird. The compactness factor approaches \\(8/9\\), but suddenly shifts to \\(4/5\\). In fact, the maximum load factor is still \\(2/3\\), but [Python dictionaries have gotten more compact](https://mail.python.org/pipermail/python-dev/2016-September/146327.html). 

So what's changed? In the old implementation, dictionaries stored a single array of 24-byte entries. If a dictionary has capacity \\(c\\) and a maximum load factor of \\(2/3\\), then its maximum compactness factor is \\((24 \cdot \frac{2}{3} c)/(24 \cdot c) = 2/3\\).

This isn't very efficient: if a third of the entries are always going to be unoccupied, we're wasting 24 bytes on each of them. Python 3.6 solves this by storing an array of \\(c\\) _indices_, which it uses to access a separate array of \\(\frac{2}{3} c\\) 24-byte entries. When inserting a new key, Python (1) adds the key-value pair to the entries array (2) probes through the indices array until it finds an open slot and (3) stores the index of the new entry.

The size of these indices depends on the the value of \\(c\\): 1 byte if \\(c < 2^8 = 256\\), 2 bytes if \\(c < 2^{16}= 65,536\\) and so on. So for a dictionary with capacity \\(2^8 \le c < 2^{16}\\), the maximum compactness factor will be \\((24 \cdot \frac{2}{3} c) / (24 \cdot \frac{2}{3} c + 2 \cdot c) = 8/9\\), which matches our observations. This is a pretty sizable improvement!

### Problem 3

In this example, setting `d[key]` then retrieving `d[key]` usually results in a `KeyError`: if the hash value changes, Python will be unable to locate the key.  

```python    
d = {}
key = A(5)
d[key] = "foo"
print(d[key])    
```

Here, setting the value of `d[key]` a second time should overwrite the first instance. If the hash value changes, however, it will insert `key` a second time and this program will print `2`.

```python    
d = {}
key = A(5)
d[key] = "foo"
d[key] = "bar"
print(len(d))
```

Moral of the story: hash functions should be deterministic. 


### Problem 4

If we insert \\(k\\) keys into a dictionary with \\(n\\) hash values, we expect \\(k/n\\) keys to be assigned to each hash value. For each value of \\(n\\), inserting these \\(k/n\\) values should take \\(1 + 2 + \dots + k/n = \Theta(k^2/n^2)\\) time. This is because every time we add a new key, we need to probe through all existing keys with the same hash value before we find an open slot. Adding over all \\(n\\), the time it takes to insert all \\(k\\) keys should be \\(\Theta(k^2/n)\\) in expectation. Note that probe sequences can overlap, so this analysis isn't 100% accurate--we'll get to that later!

We can verify this using the following code, which adds \\(k\\) `B(n)` objects to a table and measures how long it takes:

```python
import time 

def create_dict(n, k):
    d = {}
    start_time = time.time()
    for i in range(k):
        d[B(n)] = i
    return time.time() - start_time
```

First, for a few values of \\(n\\) we can see how insertion time varies with \\(k\\). Indeed, a regression will verify that insertion time increases quadratically with \\(k\\).

<img src="/blog/images/chart1.svg" style="width:min(100%, 800px)">

Second, for a fixed value of \\(k = 2000\\) we can see how insertion time varies with \\(n\\). It shouldn't be hard to verify that time is proportional to \\(1/n\\).

<img src="/blog/images/chart2.svg" style="width:min(100%, 800px)">

Cool, these are consistent with our predictions! 

So why doesn't Python run into this slowdown? In short, hash collisions rarely happen in practice. Python also makes sure to use every bit of the hash value, as opposed to naively taking hash values mod the length of the array (see the `perturb` logic in [dictobject.c](https://github.com/python/cpython/blob/master/Objects/dictobject.c)).

In our example, there were so many hash collisions that insertion time was dominated by two keys having the exact same probe sequence. In the real world, however, insertion time is dominated by different probe sequences overlapping. [It isn't trivial](https://en.wikipedia.org/wiki/Linear_probing#Analysis), but one can prove that inserting \\(k\\) keys into a dictionary takes \\(\Theta(k)\\) time in expectation. 

