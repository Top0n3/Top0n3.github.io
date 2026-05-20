---
title: PWN Rop chain Trick 
categories: CTF 
image: null
published: true
---


H3ll0 pwn3rs !!!

This article is made to explain some trick and methodologies that you can use to bypass some constraint in your ropchain creation

I suppose that you undrstand what is ROP(Rreturned Oriented Programming) technicall and will not explain it here.
So let start !!!!!!!!!!!!!

# Context / Importance

In Rop chain challenge, you will be faced to multiple problems, constrain that you need to solve in order to get a value  `rop chain`.
It is very important to know some of these problems . That will help you avoid losing your time during competition.
Bellow are some constrain and some trick to bypass theme.

# Lack of strings in the binary

Suppose that you need to make rop chain and call execve("/bin/sh") and you realise that the binary file don't contain `/bin/bash` string!!! Deam.
This is the common problem that you can  be faced.
Some trick
- Write your string to the memory location that you can determinate.
The purpose here is to exploit bof and write the needed string into memory and use it after to make your syscall
For instance, Imagine that with your rop chaine you can call function `print_file` which take  a pointer on string( file name) and read/print it contain.
Maybe your purpose is to read flag.txt flag but the program don't contain this string.
If the binary is not PIE protected, you must remember that if you are able to call gets(some_writable_address), you can write "flag.txt" at some_writable_address. So you will use it in your next rop chain to call print_file.

