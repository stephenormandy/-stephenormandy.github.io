---
layout: post
title: REvil Part One: Passing Parameters on the Stack
---

Welcome to the first post of my blog! 

As an aspiring malware analyst, much of my free time is spent in Ghidra or a debugger pulling something apart.

This post will be the first in a series detailing my analysis of how the REvil ransomware (AKA Sodinokibi) decrypts its strings at runtime and how I automated the decryption statically through Ghidra.

This first entry will explain how REvil uses the stack to pass and formulate the required parameters to use one group of functions to decrypt varied length strings. 

While this is my first analysis of ransomware, I have analysed other families such as Lokibot and Emotet. The latter of which will be the subject of a future post about a custom detection tool created in C++.


