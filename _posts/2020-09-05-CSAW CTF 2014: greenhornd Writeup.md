---
layout: post
title: "CSAW CTF 2014: greenhornd Writeup"
date: 2020-09-05
tags: [x86, Pwn] 
description: The post explains how to solve the greenhornd challenge from CSAW CTF 2014. 
---

# Introduction

Nowadays, I'm trying to learn windows exploitation by reading the tutorials and solving tasks that recommended by open-source [seminar](https://github.com/leesh3288/WinPwn/tree/master/Seminar/2019_Winter_WinPwn) written with the Korean language (Thanks to google translate) besides other external resources. So, I decided to provide writeups for the chosen challenges existed within the seminar repository.

Consequently, I am going today to solve my first 32-bit windows pwn challenge within window10 which is greenhornd from CSAW CTF 2014 using the Open-Read-Write ROP chain to read the file named key from a remote server. Additionally, I will use [AppJailLauncher](https://www.trailofbits.com/expertise/appjaillauncher/) to launch the exe file for providing a game server experience using the following command.

>CommandLine
{:.filename}
{% highlight bash %}
AppJailLauncher.exe /network /key:key /port:9998 /timeout:30 greenhornd.exe
{% endhighlight %}
