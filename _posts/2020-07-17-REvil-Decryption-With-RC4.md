---
layout: post
title: REvil Part Two - Decryption With RC4
---

We know which function generates the parameters for decryption. Now onto the decryption itself. 

Decrypt_strings_wrapper allows REvil to decrypt strings in a non-linear manner. Once the parameters are generated within the wrapper, they are passed onto another function to perform the decryption process. Decrypt_strings and its child functions are the focus for this post.

![REvil decryption functions]({{ site.baseurl }}/images/decryption_functions.png)
