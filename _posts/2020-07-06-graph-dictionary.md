---
layout: post-no-feature
title: Constructing a Python Graph
description: What happens when you have a terrible hash function.
comments: true
visible: false
date: 2020-07-06
category: articles
---

I came up with this while TA'ing [6.006](https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-006-introduction-to-algorithms-fall-2011/): it didn't make it to any quizzes or problems sets but I thought it was cute.

## Problem

In Python, it's common to store graphs as a nested dictionary, such that `d[u]` stores a dictionary in the form `{v: weight of (u, v)}` for every edge `(u, v)`. This way, you check whether edge `(u, v)` exists using `if v in d[u]`, and you can retrieve the weight of `(u, v)` by calling `d[u][v]`. 

Given a **connected weighted directed** graph \\(G\\) with vertices \\(V\\) and edges \\(E\\), how long does it take to construct this data structure in the **worst case**? Express your answer using \\(\Theta\\) notation.

Assume we use the following code to construct the graph, given a list of `vertices` and `edges`: 

```python
1  d = {v: {} for v in vertices}  
2  for (u, v, weight) in edges:
3      d[u][v] = weight
```

## Solution

Python dictionaries provide operations in expected \\(\Theta(1)\\) time, so constructing this data structure should take expected \\(\Theta(\|V\|+\|E\|)\\) time. 

But what's the worst case? Python implements dictionares using a hash table with [open addressing](https://en.wikipedia.org/wiki/Open_addressing) (you can even check out the [source code](https://github.com/python/cpython/blob/master/Objects/dictobject.c)). If every key hashes to the same value, then our dictionaries are doomed: every time you add a new key, you'll need to probe through every existing key before you find an open slot. Therefore, inserting \\(x\\) keys into a dictionary will take \\(\Theta(1 + 2 + \cdots + x)= \Theta(x^2)\\) time.

Alright, so label the vertices \\(v_1, v_2, v_3, \dots, v_{\|V\|}\\) **in the order of insertion** into \\(d\\), and let \\(\text{deg}(v)\\) be the outdegree of vertex \\(v\\). We'll also use the following facts: 

\\[\begin{aligned} \text{deg}(v) &< \|V\| \quad \text{for all}\,\, v \in V \cr \cr \sum_{i=1}^{\|V\|} \text{deg}(v_i) &= \|E\| \end{aligned}\\]

Now let's look at the code. First, for line 1, constructing the vertex dictionary takes \\(\Theta(\|V\|^2)\\) time.

For line 3, we have two separate parts: 

* For every vertex \\(v_i\\) there are \\(\text{deg}(v_i)\\) accesses to \\(d[v_i]\\), each of which requires probing through \\(i\\) keys before finding the right value. 
* In addition, for every edge \\( (v_i, v_j) \\), we need to insert \\(v_j\\) into neighbor dictionary \\(d[v_i]\\). Over all neighbors of \\(v_i\\), this takes \\(\Theta(\text{deg}(v_i))^2\\) time.

Summing this up over all vertices, we have:

\\[\begin{aligned} \sum_{i=1}^{\|V\|} i \cdot \text{deg}(v_i) + \text{deg}(v_i)^2 &\le \sum_{i=1}^{\|V\|} \|V\| \cdot \text{deg}(v_i) + \|V\| \cdot \text{deg}(v_i) \cr \cr &= 2  \cdot \|V\| \sum_{i=1}^{\|V\|} \text{deg}(v_i) \cr \cr &= 2 \cdot \|V\| \cdot \|E\| \end{aligned}\\]

In total, we have an upper bound of \\(O(\|V\|^2+\|V\| \cdot \|E\|)\\). We know the graph is connected, so it follows that \\(\|E\| \ge \|V\| - 1\\) and we can simplify this to \\(O(\|V\| \cdot \|E\|)\\).

I'll leave it to you to demonstrate that this upper bound is tight, leading to a final answer of \\(\Theta(\|V\| \cdot \|E\|)\\) in the worst case!
 
## Implementation

Let's demonstrate this upper bound using code! We can emulate worst-case behavior by wrapping everything with an object that has a constant hash value:

```python
class C:
    def __init__(self, x):
        self.x = x

    def __hash__(self):
        return 1

    def __eq__(self, other):
        return self.x == other.x
```

Using this wrapper, we can randomly create the graph data structure for various values of \\(V\\) and \\(E\\), taking the worst-case time over multiple trials. Here are the results, literally plotting \\(VE\\) vs. time:

<img src="/images/dict-plot.svg" style="width:min(100%, 800px)">

Looks pretty linear! And [here's the code](https://gist.github.com/akshayravikumar/113239ca18336b247a635c72669cdc60).
