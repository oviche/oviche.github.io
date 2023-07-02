---
layout: post
title: "Patching ELF with Rair"
date: 2020-06-12
tags: [ELF,Reverse Engineering] 
description: In this post, I will try to solve oracle level 3 challenge from chapter 5 of practical binary analysis book using Rair. 
---

# Introduction

In this post, I will try to solve oracle level 3 challenge from chapter 5 of [practical binary analysis book](https://practicalbinaryanalysis.com/) using [Rair](https://github.com/Rair-Project/rair) which is a Reverse Engineering Framework that's under development. Briefly, Its rewrite of radare2 but in rust to become more memory safe and more stable along with superior features that are under development. Today, I will just use Rair hex-editor feature for patching the ELF Binary file to solve our challenge.


# Installation in Linux

Install Rust.
> CommandLine 
{:.filename}
{% highlight bash %}
$ curl --proto '=https' --tlsv1.2 https://sh.rustup.rs -sSf | sh
{% endhighlight %}

Add Rust to your system PATH manually.
> CommandLine 
{:.filename}
{% highlight bash %}
$ source $HOME/.cargo/env
{% endhighlight %}

Use cargo Rust’s build system and package manager to download Rair.
> CommandLine 
{:.filename}
{% highlight bash %}
$ sudo cargo install rair
{% endhighlight %}


# Level-3 Analysis

At the start, I execute the lvl3 binary and kinda get an error that file has an invalid format.  
> CommandLine
{:.filename}
{% highlight bash %}
$ ./lvl3
bash: ./lvl3: cannot execute binary file: Exec format error
{% endhighlight %}

Also, when I tried to check the file format of lvl3 using **file** utility command I still get an error.

> File utility command
{:.filename}
{% highlight bash %}
$ file lvl3
lvl3: ERROR: ELF 64-bit LSB executable, Motorola Coldfire, version 1 (Novell Modesto) error reading (Invalid argument)
{% endhighlight %}


Now we know something wrong is going on with format and we need to dig deeper by checking ELF headers to know what causes this error with the help of readelf utility. So let's start taking look on the executable header.
> readelf utility command
{:.filename}
{% highlight bash %}
$ readelf -h --wide lvl3
 ELF Header:
  Magic:   7f 45 4c 46 02 01 01 0b 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            Novell - Modesto
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Motorola Coldfire
  Version:                           0x1
  Entry point address:               0x4005d0
  Start of program headers:          4022250974 (bytes into file)
  Start of section headers:          4480 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         29
  Section header string table index: 28
readelf: Error: Reading 504 bytes extends past end of file for program headers
{% endhighlight %}

The **start of program headers** did not make sense because its value is greater than the file size value. Also, It should follow executable header so its offset should be 64 bytes into the file. Also, **OS/ABI** and machine did not make sense so I will change them to **UNIX-System V** and **Advanced Micro Devices X86-64** respectively to make ELF file executing on normal Linux machine. Now, let's check Section headers.

Also, , The **.text** section type is **NOBITS** but this section exists in a disk image of the file so it should be **PROGBITS** type. **PROGBITS** value means the section contains program data or instructions. so lets in the next section patch the issues we discovered.


# Playing with Rair

Now, we will start to see how to use Rair to patch the binary file. At first, we should open the file for dumping or patching with read/write permissions as the following.

![img]({{'/assets/images/rair/rair1.PNG' | relative_url }}){: .center-image }*(**Setting file permissions**)*

So to make sure that file opened with right permissions besides knowing which address the file mapped into. we will use the following **files** command.

![img]({{'/assets/images/rair/rair2.PNG' | relative_url }}){: .center-image }*(**Listing the opened files**)*

One cool feature of Rair,  It can allow you to open many files and work on them at the same time by using **open** command. you can know about it using **open?** command. Now, I'm going to seek to start the address of a file using **seek** command then dump the ELF executable header which has 64 bytes size using px command.

![img]({{'/assets/images/rair/rair3.PNG' | relative_url }}){: .center-image }*(**Dumping the ELF executable header**)*

Right now, we need to know offsets of OS/ABI, Machine, and Start of program headers inside the ELF executable headers so let's check the Elf64_Ehdr struct to know the offsets.

> Elf64_Ehdr struct
{:.filename}
{% highlight C %}
typedef struct {
unsigned char e_ident[16]; /* Magic number and other info
                         ----> OS/ABI = e_iden[7] <----*/
uint16_t e_type; /* Object file type */
uint16_t e_machine; /* ---> Machine <------ */
uint32_t e_version; /* Object file version */
uint64_t e_entry; /* Entry point virtual address */
uint64_t e_phoff; /* ------> Program header table file offset <-------- */
uint64_t e_shoff; /* Section header table file offset */
uint32_t e_flags; /* Processor-specific flags */
uint16_t e_ehsize; /* ELF header size in bytes */
uint16_t e_phentsize; /* Program header table entry size */
uint16_t e_phnum; /* Program header table entry count */
uint16_t e_shentsize; /* Section header table entry size */
uint16_t e_shnum; /* Section header table entry count */
uint16_t e_shstrndx; /* Section header string table index */
} Elf64_Ehdr;
{% endhighlight %}

In the last snippet, we can get that offset of **OS/ABI =7** with **size =1** byte, **Machine =18** with **size =2** bytes, and **offset the start of the program header =32** with **size =4** bytes. 

let's seek the offsets and overwrite them with the right values which are **OS/ABI =0x0 (UNIX-System V)**, **Machine =0x3e00 (Advanced Micro Devices X86-64)** and finally, the **start of program headers =0x40000000** (64 in little-endian) using the **wx** command.

![img]({{'/assets/images/rair/rair4.PNG' | relative_url }}){: .center-image }*(**Fixing ELF executable header**)*

Now we have done with ELF executable header. It's time for fixing .text section type in the ELF section headers. The **start of .text section header** **= Start_of_section_headers +** **(Size of section headers \*  index_of_text_section) = 4480 + (64\*14) =5376**. 

All numbers in the equation are extracted from **readelf** command results from previous snippets. Now, we need to define the offset of section type from the start of the **.text** section header by looking into **Elf64_Shdr** struct.

> Elf64_Shdr struct
{:.filename}
{% highlight C %}
typedef struct {
uint32_t sh_name; /* Section name (string tbl index) */
uint32_t sh_type; /* Section type */
uint64_t sh_flags; /* Section flags */
uint64_t sh_addr; /* Section virtual addr at execution */
uint64_t sh_offset; /* Section file offset */
uint64_t sh_size; /* Section size in bytes */
uint32_t sh_link; /* Link to another section */
uint32_t sh_info; /* Additional section information */
uint64_t sh_addralign; /* Section alignment */
uint64_t sh_entsize; /* Entry size if section holds table */
} Elf64_Shdr;
{% endhighlight %}

Based on the previous snippet, the type of **.text** section header now located at 4 bytes from the start of the .text section header (type offset = 5376+4=5380 from start ELF file) with **size =4** bytes. So we will seek to offset 5480 to overwrite it with value **PROGBITS** (0x01000000 in little-endian).

![img]({{'/assets/images/rair/rair5.PNG' | relative_url }}){: .center-image }*(**Fixing .text section type field**)*

# Finishing it

After I have done with patching the headers, I executed the binary file and it executed successfully by printing a hash.

> CommandLine
{:.filename}
{% highlight bash %}
$ ./lvl3
3a5c381e40d2fffd95ba4452a0fb4a40  ./lvl3
{% endhighlight %}

I used ltrace to know about that hash. I find out the binary calculates a hash for its file image. so I just feed that hash to oracle and successfully level 3 is done.

> CommandLine
{:.filename}
{% highlight bash %}
$ltrace ./lvl3
__libc_start_main(0x400550, 1, 0x7fffdadc5628, 0x4006d0 <unfinished ...>
__strcat_chk(0x7fffdadc5120, 0x400754, 1024, 0)                              = 0x7fffdadc5120
__strncat_chk(0x7fffdadc5120, 0x7fffdadc637b, 1016, 1024)                    = 0x7fffdadc5120
system("md5sum ./lvl3"3a5c381e40d2fffd95ba4452a0fb4a40  ./lvl3
 <no return ...>
--- SIGCHLD (Child exited) ---
<... system resumed> )                                                       = 0
+++ exited (status 0) +++
 
$ ./oracle 3a5c381e40d2fffd95ba4452a0fb4a40
+~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~+
| Level 3 completed, unlocked lvl4         |
+~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~+
Run oracle with -h to show a hint
{% endhighlight %}

