---
layout: post
title: "The Analysis of RedLine Stealer"
date: 2023-06-20
tags: .Net Malware 
description: The post shows the details of the analysis of the RedLine Stealer that acts as a GTA cheating module. 
---

# Intro

- In this post, I will cover the details of my analysis for info-stealer malware called **RedLine stealer** which is currently one of the trends. Regarding the case covered in this post, the malware masquerades as a cheating module for the GTA game.


# The kill-chain

- The following diagram shows the flow of attack from the initial access to communication with the C2 server.

   ![img]({{ '/assets/images/Redline/redline-1.png' | relative_url }}){: .center-image }*(**The kill-chain Diagram**)*

  ## Delivery stage

   - The malware is distributed as a cheating module for the GTA game on some websites and delivered when it gets downloaded by gamers.

   - The below screenshot shows a text file that exists in the same folder with the infection first stage executable, trying to give the user confidence that it's a non-malicious cheating module by showing instructions and features.

   ![img]({{ '/assets/images/Redline/redline-2.png' | relative_url }}){: .center-image }*(**Snippet for the instructions text file**)*

  ## Installation stage

  - Once the executable **GTA_hlMWYG.exe** is executed, a DLL file gets loaded and its main function gets executed. This DLL is protected by [Enigma](https://enigmaprotector.com/) Protector which is proofed by the unique exported functions names as appears below.

     ![img]({{ '/assets/images/Redline/redline-3.png' | relative_url }}){: .center-image }*(**Exported functions of the loaded DLL**)*
    
  - As appears in the below screenshot, the loaded DLL file then executes a process called AppLaunch.exe in suspend mode and then injects the RedLine PE file into the memory of the suspended process.

    ![img]({{ '/assets/images/Redline/redline-4.png' | relative_url }}){: .center-image }*(**Injecting RedLine inside AppLaunch process**)*

  ## The Command and Control stage

  - Once the RedLine is executed, it checks the status of the C2 server.
  
  - If the C2 server is up, then they both start communicating using SOAP messages over TCP protocol to make it harder for detection.



      


  
