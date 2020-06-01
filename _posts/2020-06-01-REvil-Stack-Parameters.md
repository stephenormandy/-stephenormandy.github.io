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
PUSH 0x5
PUSH 0x4
PUSH 0x3
PUSH 0x2
PUSH 0x1
```
Resulting in a stack looking like:

<p align="center">
  <img src="/images/initial_stack.png">
</p>

As there is much importance placed upon tracking where the top of the stack lies, there is a dedicated part of the CPU designed for just that.

CPU’s have very small areas of memory (max 4 bytes) within them called “registers”. Each register typically has a purpose:


| Register Name  | Purpose  |
|:-:|:-:|
| EAX  | Storing maths calculations  |
| EBX  | Pointers to memory locations  |
| ECX  | Loop counter  |
| EDX  | Storing maths calculations  |
| EDI  | Copying strings across memory locations  |
| ESI  | Copying strings across memory locations |
| EBP  | Base pointer - bottom of the stack |
| ESP  | Stack pointer – top of stack  |
| EIP  | Address of next instruction to execute  |


Source/Further Reading: http://www.eecg.toronto.edu/~amza/www.mindsec.com/files/x86regs.html

As you can see in the table above, ESP is the register that holds the current address to the top of the stack. Whereas EBP holds the address to the bottom of the stack. There is more nuances and details to each register but for the purpose of this post it is most important to be aware of EBP and ESP.

So far, we have discussed two instructions for adding/removing items on the stack, however it is feasible to read values from higher in the stack by getting them relevant to the base (or frame) pointer (the address pointed to in EBP). Each function has its own section of the stack, called a stack frame:

<p align="center">
  <img src="/images/stack_frame.png">
</p>

Assuming that the five values we pushed before were parameters to a function, calling the new function automatically places the return address as the instruction after the call onto the stack. The called function will then push the current value of EBP, which is still pointing to the bottom of the callers stack frame and moves the value of ESP (stack pointer to the top of the stack) into EBP. Starting the new stack frame for the current function. Each function will start with the following two instructions to make this happen:

```assembly
PUSH EBP
MOV EBP, ESP
```

In a "mov" command, the first parameter (called operand) is the destination and the second operand is the source, so we are moving the value of ESP into EBP. Once the called function finishes, it knows where to go back to by using the return address.

This is a very brief overview of two fairly complex topics but it should give you a good background into the rest of this post. 

### REvil Decryption Functions

X32dbg labels (names) functions based on their address. Therefore, throughout this blog any function will be referred to by a name that I set previously, to ensure others can follow along I have also made sure to include the relevant address for each function discussed.

Below is a diagram detailing the flow between functions for decrypting a string, addresses are included in parentheses:

![REvil decryption functions]({{ site.baseurl }}/images/decryption_functions.png)

A function that requires a string decrypted will start by making a call to decrypt_strings_wrapper, this takes in five parameters:
1.	Base address for cipher text and key
2.	Offset to required cipher text and key
3.	RC4 key length
4.	Length of plaintext
5.	Pointer to address that stores the plain text

These parameters are used to generate two new ones and pass on three existing ones, resulting in decrypt_strings also requiring five parameters. generate_key_schedule and string_decrypt are functions that implement the RC4 algorithm and perform the actual decryption required. 

For this post, the focus will be on decrypt_strings_wrapper. Specifically, on how parameters are passed to it and what it does with them that allows REvil to use these same functions to decrypt whatever string is required by the caller.

When analysing the REvil sample, the first call to decrypt_strings_wrapper is made when importing the libraries it requires. We shall start there and follow it through to understand the end to end process of decrypting a string. 

The call instruction of interest is at 0x00413322:

![Call to decrypt_strings_wrapper]({{ site.baseurl }}/images/call_decrypt_strings_wrapper.png)

A quick side note, if you see a lot of push instructions before a call, then that signifies parameters are being pushed into a function. For example, this is the same call instruction but without the helpful label and comments:

![Call to decrypt_strings_wrapper without comments]({{ site.baseurl }}/images/call_decrypt_strings_wrapper_unedited.png)

Understanding what those parameters are go a long way to understanding the called function as a whole. 

We have already discussed LIFO and how it operates, this means that any push instruction is the reverse order of how they would be in the stack. What we may assume would be the last parameter of pushing a memory address to 0x41D218, is in fact the first parameter at the top:

![Stack from call to decrypt_strings_wrapper ]({{ site.baseurl }}/images/call_decrypt_strings_wrapper_stack.png)

Now we move into decrypt_strings_wrapper.

With our newfound knowledge of the stack, we can see that calling decrypt_strings_wrapper automatically pushes the return address onto the stack:

![Call to decrypt_strings_wrapper sets the return address]({{ site.baseurl }}/images/call_decrypt_strings_wrapper_return.png)

The highlighted value at the top of the stack is the EBP, and its value is set to what the previous base pointer was. 4 bytes below that, there is the return address and below that we have our five parameters that were pushed. 

Accessing these parameters can be done by using an offset from the address of EBP. Each entry in the stack is 4 bytes, so to get the first parameter previously pushed, it would be EBP+8 (EBP+4 is the return address). Within assembly this is simply achieved with the mov instruction:

![Retrieving a parameter from the stack]({{ site.baseurl }}/images/decrypt_strings_wrapper_get_parameter.png)

A dword ptr  retrieves 4 bytes of data that reside at the specified address. In the case above, it returns the 4 bytes at 0x0018FEC0, meaning EDX now holds the base address for the cipher text (0x0041D218).

![Placing the parameter into EDX]({{ site.baseurl }}/images/decrypt_strings_wrapper_setting_edx.png)

Next up, the offset value stored in EBP+C (12 in decimal) is then added to that base address to generate the pointer to the start of the required key and ciphertext:

![Generate pointer to required RC4 key]({{ site.baseurl }}/images/decrypt_strings_wrapper_generate_pointer_to_key.png)

The pointer to the plaintext buffer and plaintext length are retrieved and immediately pushed:

![Retrieve pointers for plaintext storage/length]({{ site.baseurl }}/images/decrypt_strings_wrapper_retrieve_pointers.png)

The LEA instruction returns a pointer/address of where the key is stored rather than the value of the key itself. Opposite to a “ptr” which will retrieve the value at the address instead. 

From there it is a case of pushing the rest of the values: address of the cipher text, key length and the address of the key. By understanding the parameters, it became clear that REvil stores its RC4 keys and ciphertext appended to each other:

![View of RC4 key and ciphertext stored in PE section]({{ site.baseurl }}/images/decrypt_strings_wrapper_ciphertext_in_section.png)

All RC4 keys and ciphertext combinations start from 0x0041D218 in this sample, this is a constant value throughout each call to decrypt_strings_wrapper. By changing the offset and key length it is feasible to access each string in a non-linear manner. It is important to note that this base address changes for each sample but as the parameters are at fixed positions this makes it slightly easier to automate decryption. How that is accomplished will be discussed later, the next post will detail how these parameters are then used with the RC4 algorithm itself.

### Summary
REvil uses one set of functions for all its string decryption needs. It’s top level decryption function at decrypt_strings_wrapper (0x004123C5) requires five parameters:
1.	Base address for cipher text and key
2.	Offset to required cipher text and key
3.	RC4 key length
4.	Length of plaintext
5.	Pointer to address that stores the plain text
By adding the offset value in parameter two to the base address in parameter one, REvil can dynamically select an encrypted string stored in the .data section. Further analysis shows that the RC4 key is appended to each piece of cipher text. 
While the base address will be different for each sample, the parameters are the same. Therefore, we now know how REvil stores and accesses its encrypted strings, very useful for when we want to decrypt them ourselves.
