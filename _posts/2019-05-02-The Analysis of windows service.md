---
layout: post
title: "The Analysis of windows service"
date: 2019-05-02
tags: [x86, Malware Analysis] 
description: The post explains the structure of the windows service executable by analyzing Shamoon 3.0 dropper. 
---

# Introduction

In my first post, I will analyze the dropper of **Shamoon 3.0** malware which is windows service executable that differs from a normal executable structure and execution method.

So by analyzing dropper of Shamoon 3.0, we can understand:
 1. windows service structure.
 2. how to analyze & debug windows service.

So let us understand what's windows service and how it structured before jumping to Analysis.

# Windows service program

It's a program that executed by **Service Control Manager (SCM)** and conforms to its rules. it runs in the background with no GUI interface as it doesn't need a user to interact with it. It can be started automatically at system boot.

 ![img]({{'/assets/images/shamoon3/winserv1.PNG' | relative_url }}){: .center-image }*(**Windows Service Structure**)*


The Window Service Structure program consists of three important functions as seen in windows service structure Figure:

## 1. **Main entry point function**

The main function of the windows service program, its goal to inform the service name and its service main function to SCM by calling **StartServiceCtrlDispatcherA** **(SERVICE_TABLE_ENTRYA** **\*lpServiceStartTable)** function. so that the SCM starts execution of the service main function. the service name and its main function provided by a **lpServiceStartTable** parameter which is a pointer to the list of **SERVICE_TABLE_ENTRYA** structures.

> SERVICE_TABLE_ENTRYA structure 
{:.filename}
{% highlight C %}
typedef struct _SERVICE_TABLE_ENTRYA {
  LPSTR lpServiceName;
  LPSERVICE_MAIN_FUNCTIONA lpServiceProc; }
{% endhighlight %}

> **Note**: if windows service program didn't call StartServiceCtrlDispatcherA before a certain timeout SCM will stop the execution of the service.

## 2. **Service main function**

The function that was specified as service main by main entry point function and its goal to register a callback function called service control handler by calling **RegisterServiceCtrlHandlerA** **(lpServiceName,** **lpHandlerProc)** function to handle control requests sent by SCM.

Then it sends service status to SCM by calling **SetServiceStatus** function so that SCM can track the status of service. Also, the service functionality code is implemented here.


## 3. **Service control handler callback**
a callback function that is expected to handle service **STOP**, **PAUSE**, **CONTINUE**, and **SHUTDOWN** codes sent by SCM. The control handler must also return within **30 seconds** or SCM will return an error stating that service is not responsive.

> **Note**: service can't be executed like any executable but by it needs to be installed and started using these commands.

> Executing the service
{:.filename}
{% highlight Bash %}
 sc create "ServiceName" binPath= C:\service.exe
 sc start "ServiceName"
{% endhighlight %}


# Shamoon 3.0 dropper Analysis

I started my analysis by checking the resources of executable then I found three encrypted or encoded resources and their names **LNG**, **MNU** and **PIC** after that I looked at disassembly graph of the main function in **Fig[1]**. Then I noticed two things, first a call to **StartServiceCtrlDispatcherA** and the second thing that it checks argument count so I concluded that it was a service and need an argument to run.

![img]({{'/assets/images/shamoon3/winserv2.PNG' | relative_url }}){: .center-image }*(**Fig[1]**)*

Although it's a service if it runs as a normal process it contains code that responsible to setup itself as service. so I decided to start debugging it like any normal process to spotlight on how it's going to set up and run itself as service.
In **Fig[2]** it calls a **CreateService** function to create a service then specify service name and the path of service program file with command argument **LocalService** to run with. Also, set its start type as **AUTO_START** for persistence to run
automatically during startup.

![img]({{'/assets/images/shamoon3/winserv3.PNG' | relative_url }}){: .center-image }*(**Fig[2]**)*


Also, it sets a service description by using **ChangeServiceConfig2W** in **Fig[3]**.

![img]({{'/assets/images/shamoon3/winserv4.PNG' | relative_url }}){: .center-image }*(**Fig[3]**)*


Then it starts itself as service using **StartService** function in **Fig[4]**.

![img]({{'/assets/images/shamoon3/winserv5.PNG' | relative_url }}){: .center-image }*(**Fig[3]**)*


Here comes the tricky part which is how to debug the service starting from the very beginning so that we won't miss anything. The first step is to replace two bytes from entry point function with jump self instruction that correspond to opcode **EB FE**. In **Fig[5]** we will replace **push 14h** with **jump self instruction** as in **Fig[6]**.

![img]({{'/assets/images/shamoon3/winserv6.PNG' | relative_url }}){: .center-image }*(**Fig[5]**)*

![img]({{'/assets/images/shamoon3/winserv7.PNG' | relative_url }}){: .center-image }*(**Fig[6]**)*



The second step is to enable **debug privilege** in x32dbg that exist in **options >> preferences >> engine** so that we get the privilege needed to debug service. As we said in the early section about the service if it started without calling **StartServiceCtrlDispatcherA** function before a certain timeout then SCM will stop the service execution so we have to edit this timeout. So the third step to do is to create a **Dword** value called **ServicesPipeTimeout** in registry path **HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control** then set it to **86,400,000** which is 24 hours in a millisecond as in **Fig[7]** to extend the time for debugging. then restart our windows to apply this change.

![img]({{'/assets/images/shamoon3/winserv8.PNG' | relative_url }}){: .center-image }*(**Fig[7]**)*


Now we can step over the StartService function in **Fig[4]** and attach to newly created service so it will stop at **jump self** instruction as in **Fig[8]** then we should replace this instruction by original one which is **push 14h** to continue debugging.

![img]({{'/assets/images/shamoon3/winserv9.PNG' | relative_url }}){: .center-image }*(**Fig[8]**)*

Let us spotlight about what are the resources exist inside the service. After I continue debugging I find out that service started to search for a resource called **LNG** using function **findResourceW** and retrieves pointer this resource by using functions **LoadResource** and **LockResource** as in **Fig[9]**.

![img]({{'/assets/images/shamoon3/winserv10.PNG' | relative_url }}){: .center-image }*(**Fig[9]**)*


After service gets a pointer to **LNG**, it created a file called **tsprint_ibv.exe** as in **Fig[10]** to write LNG bytes from memory to that file. if you notice memory dump in **Fig[11]** the memory bytes start with **MZ** which is an indication that **LNG** is embedded windows exe. then it executed as in **Fig[12]**.

![img]({{'/assets/images/shamoon3/winserv11.PNG' | relative_url }}){: .center-image }*(**Fig[10]**)*

![img]({{'/assets/images/shamoon3/winserv12.PNG' | relative_url }}){: .center-image }*(**Fig[11]**)*

![img]({{'/assets/images/shamoon3/winserv13.PNG' | relative_url }}){: .center-image }*(**Fig[12]**)*

By hooking WINAPI functions, I discovered that the **tsprint_ibv.exe** process is responsible to drop and load driver called **hdv725x.sys** as in **Fig[13]** which is responsible for wiping a disk. 

The **MNU** resource is loaded and written to file called **netb57vxx.exe** and executed in the same way as **LNG** resource. it's a communication module to communicate with C2 servers.

![img]({{'/assets/images/shamoon3/winserv14.PNG' | relative_url }}){: .center-image }*(**Fig[13]**)*

Also, the service contains a **64-bit** version of itself that saved as **PIC** resource which it will be dropped and setup if the operating system architecture is **64-bit**.

# Sample Hashes

**MD5**: de07c4ac94a50663851e5dabe6e50d1f
**SHA1**: df177772518a8fcedbbc805ceed8daecc0f42fed
**SHA-256**: c3ab58b3154e5f5101ba74fccfd27a9ab445e41262cdf47e8cc3be7416a5904f

 
