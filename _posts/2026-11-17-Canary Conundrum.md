---
layout: post
title: "Writeup: Canary Conundrum"
date: 2026-11-17
tags: [Pwn] 
description: This post discuss the solution Canary Conundrum (Hard) from pwn.college
---

# Introduction
This blog post is a write-up of the **Canary Conundrum (Hard)** challenge from [pwn.college](https://pwn.college/). 

# Gaining insights on the binary

The first step I usually do, is to check the mitigations of the binary using the `checksec` command in pwndbg. The command's output shows that all the mitigations are turned on.

![img]({{'/assets/images/Canary_Conundrum/img1.png' | relative_url }}){: .center-image }*(**The mitigations status**)*

By executing the binary, I found that it prints out a hexadecimal number representing a memory address leak and receives two inputs, which are the message's length and the message itself, as shown below.

> CommandLine 
{:.filename}
{% highlight CMD %}
$ ./challenge
Welcome.
Leak: 0x7f318f02c010
Length of your message: 5
Enter your message: aaaaa
aaaa
{% endhighlight %}
