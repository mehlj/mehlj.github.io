---
layout: post
title: Python Challenge - Additive Persistence
published: true
category: python
---

## Additive Persistence

A number's additive persistence is the number of additions required to obtain a single digit from a number ``n``. That last digit obtained is called the digital root.

For example, given the starting number of ``9876``:
```
9 + 8 + 7 + 6 = 30
3 + 0 = 3
```

We went through ``2`` rounds of addition, so the additive persistence is ``2``. The last digit obtained was ``3``, and thus, the digital root is ``3``.

Weisstein, Eric W. "Additive Persistence." From MathWorld--A Wolfram Web Resource. http://mathworld.wolfram.com/AdditivePersistence.html

## Implementation

This Python program, from a high level, does a few things:
* Accepts user input in the form of a standard integer
* Splits the input integer into a list[] of digits
* Performs the mathematics above by calculating the sum of digits, determining if there are more than one digits, and looping if so 
* Returns the additive persistence of the user-provided integer

Post code here
