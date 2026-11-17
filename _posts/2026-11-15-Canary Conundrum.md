---
layout: post
title: "pwn.college: Canary Conundrum (Hard) – Write-up"
date: 2026-11-15
tags: [Pwn] 
description: This post discuss the solution Canary Conundrum (Hard) from pwn.college
---

# Introduction
This blog post is a write-up of the **Canary Conundrum (Hard)** challenge from [pwn.college](https://pwn.college/). 

# Gaining insights on the binary

The first step I usually do, is to check the mitigations of the binary using the `checksec` command in pwndbg. The command's output shows that all the mitigations are turned on.

![img]({{'/assets/images/Canary_Conundrum/img1.png' | relative_url }}){: .center-image }*(**The mitigations status**)*

After executing the binary, I observed that it prompts the user for input and then prints the provided input to the screen.

> CommandLine 
{:.filename}
{% highlight CMD %}
$ /challenge/canary-conundrum-hard 
###
### Welcome to /challenge/canary-conundrum-hard!
###

Payload size: 12
Send your payload (up to 12 bytes)!
hshahdhhdh
You said: hshahdhhdh

Goodbye!
### Goodbye!
{% endhighlight %}


