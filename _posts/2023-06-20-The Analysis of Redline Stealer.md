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
  
  -  If the C2 server is up, they both start communicating using SOAP messages over TCP protocol to make it harder to detect than the previous RedLine version that used the SOAP over HTTP protocol that is humanly readable and easily detected.


# Deobfuscating the strings obfuscations

- Before digging deeper into the malware functionalities, I deobfuscated the RedLine strings obfuscation using a [redline-deobfuscator](https://github.com/oviche/redline-deobfuscator) that I developed.

# Countries of interest

- The countries that **are not** included in the below array are interesting targets for RedLine.

  ![img]({{ '/assets/images/Redline/redline-5.png' | relative_url }}){: .center-image }*(**Protected list of countries**)*

- The RedLine will start executing its functions if the infected device's local time-zone Id, current culture, and current UI culture aren't related to any of the previous countries list.

  ![img]({{ '/assets/images/Redline/redline-6.png' | relative_url }}){: .center-image }*(**Checking whether the infected device is interesting target**)*

# Malware Configuration decryption

- The below screenshot is for a class that represents the encrypted malware configuration. It consists of the following fields:

  - **IP** that contains encrypted C2 IP(s)
  - **ID**  that contains encrypted sample ID.
  - **Message** represents a message to display however it's empty in this case.
  - **Key** represents the decryption key.
  - **Version** represents a version of the malware.
  <p></p>
  ![img]({{ '/assets/images/Redline/redline-7.png' | relative_url }}){: .center-image }*(**Checking whether the infected device is interesting target**)*


- Three simple operations do the decryption of both **IP** and **ID** fields.

   1. Applying base64 decoding for the encrypted strings as appeared below.

      ![img]({{ '/assets/images/Redline/redline-8.png' | relative_url }}){: .center-image }*(**Base-64 decoding for the encrypted strings**)*
     
   3. Then XORing the decoded string from the previous operation with the Key which in our case is **Detersions**    

      ![img]({{ '/assets/images/Redline/redline-9.png' | relative_url }}){: .center-image }*(**XORing the decoded string with a key**)*
       
   5. Finally, apply the base64 decoding for the output of the last operation.

- Below is the decoded configuration.

  ![img]({{ '/assets/images/Redline/redline-10.png' | relative_url }}){: .center-image }*(**Decrypted RedLine configuration**)* 
 

# Establishing a network connection

- The malware will set a communication channel that uses SOAP over TCP which makes the SOAP messages sent in a binary-encoded format that is not easily get decoded.

- Also, it does not validate a certificate as it sets certificate validation mode to **none**. Additionally, it provides an authorization value equal to **4e6fb2a9ee1dcdfaa807aab6c0f5a16d** to enable the sample access to a specific.

  ![img]({{ '/assets/images/Redline/redline-11.png' | relative_url }}){: .center-image }*(**Establishing a communication channel**)* 
  
# Fetching the settings from C2
- ok



