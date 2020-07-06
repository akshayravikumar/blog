---
layout: post
title: Constructing a Graph Dictionary
description: A cute problem I wrote for MIT's Intro to Algorithms class.
comments: true
hidden: true
date: 2020-07-06
category: articles
image:
 feature: crossword-header.jpg
---

I wrote this problem while I was TA'ing [6.006](https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-006-introduction-to-algorithms-fall-2011/): it didn't make it to the quiz but I thought it was cute.

## Problem

In Python, it's common to store graphs as a nested dictionary, such that `d[u]` stores a dictionary in the form `{v: weight of (u, v)}` for every edge `(u, v)`. This way, you check whether edge `(u, v)` exists using `if v in d[u]`, and you can retrieve the weight of `(u, v)` by calling `d[u][v]`. 

Given a **connected weighted directed graph** \\(G\\) with \\(V\\) vertices and \\(E\\) edges, how long does it take to construct this data structure in the **worst case**?

Let's say the code looks like this, assuming we already have access to a list of `vertices` and `edges`:

```python
d = {v: {} for v in vertices}  

for (u, v, weight) in edges:
    d[u][v] = weight
```

## Solution

Python dictionaries provide operations in expected \\(O(1)\\) time, so constructing such a representation should take expected \\(O(V+E)\\). 

But what's the worst case? Python implements dictionares using a hash table with [open addressing](https://en.wikipedia.org/wiki/Open_addressing) (you can even check out the [source code](https://github.com/python/cpython/blob/master/Objects/dictobject.c)). If every key hashes to the same value, then our dictionaries are doomed: every time you add a new key, you'll need to probe through every existing key before you find an open slot. Therefore, adding \\(x\\) keys to a dictionary will take \\(O(1 + 2 + \cdots + x)= O(x^2)\\) time.

How do we use this information? First, constructing the vertex dictionary takes \\(O(V^2)\\).

Now let's look at the edges. In the worst case all the edges are in same neighbor dictionary, so perhaps \\(O(E^2)\\) is the upper bound? Or, each vertex has at most \\(V - 1\\) neighbors, which leads to \\(O(V^2)\\) for each neighbor dictionary and \\(O(V^3)\\) in total.

We can do better by looking at the degrees of each vertex! Label the vertices \\(1, 2, \dots, V\\) and let their outdegrees be \\(d_1, d_2, \dots, d_V\\). Now our upper bound is \\(O(d_1^2 + d_2^2 + \dots + d_V^2)\\). However, using the fact that \\(d_i < V\\) for all \\(i\\), we have the following:

\\[\begin{aligned}d_1^2 + d_2^2 + \dots + d_V^2 &< V\cdot d_1 +  V\cdot d_2 + \dots +  V\cdot d_V  \cr \cr &= V(d_1 + d_2 + \dots + d_V) \cr \cr &= VE \end{aligned}\\]

So our upper bound is actually \\(O(VE)\\), leading to a final answer of \\(O(V^2+VE)\\). As long as \\(E \ge V\\),  we can simplify this to \\(O(VE)\\)! I'll leave it to you to demonstrate that this upper bound is tight.

## Implementation

Let's demonstrate this upper bound using Python code! We can emulate worst-case behavior by wrapping everything with an object with a constant hash value:

```python
class C:
    def __init__(self, x):
        self.x = x

    def __hash__(self):
        return 1

    def __eq__(self, other):
        return self.x == other.x
```

Using this wrapper, we can create the graph data structure for various values of \\(V\\) and \\(E\\), taking the worst-case time over multiple trials. Here are the results, literally plotting \\(VE\\) vs. time:

<img src="/images/dict-plot.svg" style="width:min(100%, 800px)">

Looks pretty linear! And [here's the code](https://gist.github.com/akshayravikumar/113239ca18336b247a635c72669cdc60).
