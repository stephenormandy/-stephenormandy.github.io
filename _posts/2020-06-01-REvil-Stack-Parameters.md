---
layout: post
title: REvil Part One - Passing Parameters on the Stack
---

Welcome to the first post of my blog! 

This post will be the first in a series detailing my analysis of how the REvil ransomware (AKA Sodinokibi) decrypts its strings at runtime and how I automated the decryption statically through Ghidra.

This first entry will explain how REvil uses the stack to pass and formulate the required parameters to use one group of functions to decrypt varied length strings. 

While this is my first analysis of ransomware, I have analysed other families such as Lokibot and Emotet. The latter of which will be the subject of a future post about a custom detection tool created in C++.

### Why REvil?
REvil got media attention at the start of the year with their attack on [Travelex](https://www.trendmicro.com/vinfo/us/security/news/cybercrime-and-digital-threats/sodinokibi-ransomware-increases-yearend-activity-targets-airport-and-other-businesses). As I like a challenge and REvil has garnered a reputation for being a sophisticated piece of malware, I grabbed a sample and got to work.

### The Stack and Registers
Before diving into the details of this post, this section is a brief overview of two main topics: the stack and registers.

A stack is a particular way of storing information, in a higher programming language like Python this is akin to a list data structure. But unlike lists, a stack only allows you to push (add) or pop (remove) elements that reside at the top of it, due to a governing principle of Last In First out (LIFO).

Let’s assume that we want to fill a stack with numbers from 1 to 5.  Achieving that under LIFO means the value we want to be first, must be the last one entered. In x86 Assembly, that would be: 

```assembly
Push 0x5
Push 0x4
Push 0x3
Push 0x2
Push 0x1
```
Resulting in a stack looking like:

![Initial Stack]({{ site.baseurl }}/images/initial_stack.png "Initial Stack")