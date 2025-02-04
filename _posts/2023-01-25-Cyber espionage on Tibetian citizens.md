---
layout: post
title: "Cyber Espionage on Tibetian Citizens"
date: 2023-01-25
tags: [x86, Malware Analysis] 
description: The post shows the details of a malware attack that was developed by a Chinese state-sponsored group to spy on Tibetian citizens. 
---

# Intro

- This post will mainly include the details of my analysis of the stages of the attack, malware classification, tool tracking, and indicators of compromise.  

# Description of the kill-chain

- The following diagram shows the flow of attack from the initial access to communication with the C2 server.
  
  ![img]({{ '/assets/images/espionage_tibet_image/lowzero_1.png' | relative_url }}){: .center-image }*(**Kill-chain diagram**)*

  ## Delivery
  
  - The malware is delivered as an RTF document, maybe through phishing mail, that contains an application form written in the chinese-tibetian language as it appears below.
    
    ![img]({{ '/assets/images/espionage_tibet_image/lowzero_2.png' | relative_url }}){: .center-image }*(**RTF document page [1]**)*
 
    ![img]({{ '/assets/images/espionage_tibet_image/lowzero_3.png' | relative_url }}){: .center-image }*(**RTF document page [2]**)*

  - This may indicate that this attack is targeting Tibetan people.

  <p></p>
  ## Exploitation
  
  - This RTF document contains 2 embedded objects one of them with the name of **Equation.2\x00\x124Vx\x90\x124VxvT2**. After searching this name, I find that this RTF is generated by a tool called **Royal Road** which usually includes one of the exploits which target the equation editor vulnerabilities of the office.
  
  - This RTF document contains an exploit that takes advantage of  RCE vulnerabilities in the equation editor of Microsoft Office; which allows one to write mathematical equations and formulas.
  
  
  
  ## Installation & Reconnaissance
  
  - Once the exploit is triggered, an executable (the second embedded object in the RTF document) is decrypted and executed which executes and injects a shellcode inside **RUNDLL32.exe**.
 
  - Then the shellcode unpack, load, and execute a dll file which acts as a backdoor.

  - This backdoor collects various host information from the infected host which may be used to check if the machine is an interesting target or not.
  
  
  ## Command and Control
 
  - The backdoor communicates with the C2 server on port 110 and sends the collected information and waits till receiving one or more commands to get executed.
   

# Description of functionality

- This section will cover a detailed analysis of the components of the attack kill chain that's mentioned previously.
 
  ## RTF document
  
  - The RTF document holds two embedded objects **ghb4nrwmp.wmf** and **Equation.2\x00\x124Vx\x90\x124VxvT2** as shown below.
     
     ![img]({{ '/assets/images/espionage_tibet_image/lowzero_4.png' | relative_url }}){: .center-image }*(**RTF document's embedded objects**)*
  
  - Based on my search, the second object name is usually the name of the embedded exploit that targets the equation editor of Microsoft office to decode and execute the first object, **ghb4nrwmp.wmf** . 
    
  - This was confirmed by using any.run sandbox to check the effect of opening this document with word office. For example, the below screenshot shows that by opening the document, the equation editor **EQNEDT32.exe** got executed then **rundll32.exe** spawned from it.  
  
     ![img]({{ '/assets/images/espionage_tibet_image/lowzero_5.png' | relative_url }}){: .center-image }*(**Equation editor spawning rundll32.exe**)*  
    
  ## The embedded Executable (ghb4nrwmp.wmf)
  
  - Once the exploit gets triggered, it decodes and executes the **ghb4nrwmp.wmf** object.
  
  - Using **xorsearch** command I find this file is encrypted with xor and the key is 0xfc.
  
     ![img]({{ '/assets/images/espionage_tibet_image/lowzero_6.png' | relative_url }}){: .center-image }*(**The Xorsearch command's output**)*  
  
  - Also, as appear below, many of the bytes in the hex editor show a lot of 0xfc which indicates that these original bytes are equal to zero(s) so with xoring with 0xfc they turn to become 0xfc.  
  
     ![img]({{ '/assets/images/espionage_tibet_image/lowzero_7.png' | relative_url }}){: .center-image }*(**The Hex editor view of file**)* 

  - After decoding the file, the e-magic replacing **MZ** to **NZ** was corrupted maybe to avoid detection or to thwart static analysis tools as the following.
  
     ![img]({{ '/assets/images/espionage_tibet_image/lowzero_8.png' | relative_url }}){: .center-image }*(**The corrupted magic value**)*
  
    
  - The core functionality of this file is to do the following:
    1. Create the process **C:\Windows\system32\rundll32.exe shell32.dll,Control_RunDLL** in suspended mode.
      
       ![img]({{ '/assets/images/espionage_tibet_image/lowzero_9.png' | relative_url }}){: .center-image }*(**Creating the process in suspended mode**)*
 
    2. Inject a shellcode at the suspended process entry point using an ``WriteProcessMemory`` API as appear in below screenshots.
       
       ![img]({{ '/assets/images/espionage_tibet_image/lowzero_10.png' | relative_url }}){: .center-image }*(**The process injection by WriteProcessMemory**)*
     
       ![img]({{ '/assets/images/espionage_tibet_image/lowzero_11.png' | relative_url }}){: .center-image }*(**The injected code at rundll32 entrypoint**)*      
      
    3. Another process injection occur at newly allocated address 0xBA0000 in the suspended process as appear below.  
     
       ![img]({{ '/assets/images/espionage_tibet_image/lowzero_12.png' | relative_url }}){: .center-image }*(**Another process injection by WriteProcessMemory**)*
     
       ![img]({{ '/assets/images/espionage_tibet_image/lowzero_13.png' | relative_url }}){: .center-image }*(**Injected code at address 0xBA0000 of suspended process**)*      
      
    4.  Finally, it Resumes the suspended process to execute the shellcode.
     
        ![img]({{ '/assets/images/espionage_tibet_image/lowzero_14.png' | relative_url }}){: .center-image }*(**Resuming the suspended process**)*    
      
  
  ## The Injected shellcode
  - Now let’s go deeper to spotlight the main objective of the injected shellcode inside the rundll32 process.
  
  - Firstly, when the process resumed the shellcode written at the entry point will be executed whose main goal is to jump to injected code at address 0xBA0000 as appear below.
    
    ![img]({{ '/assets/images/espionage_tibet_image/lowzero_15.png' | relative_url }}){: .center-image }*(**The EntryPoint shellcode**)*   

  - The shellcode at address 0xBA0000 main goal is to load the dll that is embedded in the shellcode and exists at address 0xBA0C40 as appears below.
    
    ![img]({{ '/assets/images/espionage_tibet_image/lowzero_16.png' | relative_url }}){: .center-image }*(**The Dos stub of DLL in memory**)*  
  
  - By dumping the memory at this address and fixing the dos header, I could recognize that the following calls are for the **DllMain** and the exported function **F** which passed encoded/encrypted/compressed data as appeared in below screenshots.
    
    ![img]({{ '/assets/images/espionage_tibet_image/lowzero_17.png' | relative_url }}){: .center-image }*(**The call to DllMain function in debugger**)* 
    
    ![img]({{ '/assets/images/espionage_tibet_image/lowzero_18.png' | relative_url }}){: .center-image }*(**The address of DllMain function in disassembler**)*
       
    ![img]({{ '/assets/images/espionage_tibet_image/lowzero_19.png' | relative_url }}){: .center-image }*(**The call to export function F in debugger**)*
    
    ![img]({{ '/assets/images/espionage_tibet_image/lowzero_20.png' | relative_url }}){: .center-image }*(**The address of export function F in disassembler**)*
 
  ## The Backdoor DLL file
 
  - The core functionality of this backdoor lies at export function **F**.
  
  - First part of the F function is decrypting and parsing the data of the first parameter and below is the parsed data. 
     
    ![img]({{ '/assets/images/espionage_tibet_image/lowzero_21.png' | relative_url }}){: .center-image }*(**The structure of the decrypted parameter for the F function**)*
   

  - Then it initializes some global variables with the fields of parsed data as appeared below.

     ![img]({{ '/assets/images/espionage_tibet_image/lowzero_22.png' | relative_url }}){: .center-image }*(**Initializing a global variable with mutex value**)*

     ![img]({{ '/assets/images/espionage_tibet_image/lowzero_23.png' | relative_url }}){: .center-image }*(**Initializing the global variables with other parsed fields**)*


  - After that, it collects the following information (fingerprinting) from the host then encrypt and sends it to the C2 server.
  
      1. **System information** like architecture information and type of the processor, the number of processors in the system, the page size, and other system-related information.
      
      2. **operating system version information** that contains major and minor version numbers, a build number, a platform identifier, and descriptive text about the operating system
      
      3. **Host-name**.
      
      4. **Machine IP addresses**.
      
      5. **name and the PID** of the process that hosts the backdoor in its address space.
      
      6. **mutex name**.
      
      7. **machine username**.

   ![img]({{ '/assets/images/espionage_tibet_image/lowzero_24.png' | relative_url }}){: .center-image }*(**Collecting the host information**)*



  - Then it wait to receive a commands from C2 server to execute them, the commands are laid in the following structure.
   
      ![img]({{ '/assets/images/espionage_tibet_image/lowzero_25.png' | relative_url }}){: .center-image }*(**the received command structure**)*


  - The followings are the command ID(s) and their corresponding functionalities:
    
    1. **Command ID: 2000**
       - This command decrypts the command data and assigns it to a global variable called **decrypted_command_data** as appears below.
        <p></p>
       ![img]({{ '/assets/images/espionage_tibet_image/lowzero_26.png' | relative_url }}){: .center-image }*(**Decrypting the command data**)*
   
    2. **Command ID: 2001**
       - Clear the variable which holds the decrypted command data as appears below.
       <p></p>
       ![img]({{ '/assets/images/espionage_tibet_image/lowzero_27.png' | relative_url }}){: .center-image }*(**Clearing the decrypted_command_data variable**)*

    3. **Command ID: 2002**
       - This command assigns the new value to the variable that is related to a certain time duration. 
       <p></p>
       ![img]({{ '/assets/images/espionage_tibet_image/lowzero_28.png' | relative_url }}){: .center-image }*(**Setting a time duration variable**)*

    4. **Command ID: 2003**
       - This command kills the running process that hosts the backdoor in memory as appears below.
        <p></p>
       ![img]({{ '/assets/images/espionage_tibet_image/lowzero_29.png' | relative_url }}){: .center-image }*(**Killing the process**)*

    5. **Command ID: 2004**
       - This command set a global variable called **stop_connection_flag** that if set to zero, will result in a close connection and stop the backdoor execution as appear below.
       <p></p>
       ![img]({{ '/assets/images/espionage_tibet_image/lowzero_30.png' | relative_url }}){: .center-image }*(**Setting the stop_connection_flag to zero**)*     

    6. **Command ID: 2005**
       - This command is responsible to load modules to the memory and save addresses of the exported functions with an ordinal value that equals 1 and 2 as appeared below.
       <p></p>
       ![img]({{ '/assets/images/espionage_tibet_image/lowzero_31.png' | relative_url }}){: .center-image }*(**Loading modules and save exported functions**)* 
       
    7. **Command ID: 2006**
       - This command is responsible for getting the export function with an ordinal number equals to 3 from the last loaded module and executing it as appears below.   
       <p></p>
       ![img]({{ '/assets/images/espionage_tibet_image/lowzero_32.png' | relative_url }}){: .center-image }*(**Getting and executing the exported function with ordinal 3**)* 
       
  - In case of the command ID wasn't any of the pervious then:
    
    1. If the Command ID value is divided by 100 not equals to 30 and the variables that contain addresses of the exported functions of ordinal equal 1 and 2 (of the last loaded module) are not equal null, then it call the function with ordinal equal 1 as appear in the following screenshot.  
       <p></p>
       ![img]({{ '/assets/images/espionage_tibet_image/lowzero_33.png' | relative_url }}){: .center-image }*(**A call to exported function with ordinal 1**)* 
    
    2. If the command ID value is divided by 100 equals 30 and reminder equals 1, then the backdoor will set up a listener that receives a remote connection that will instruct the backdoor to connect to another host; in other words, it will act as a bridge or proxy, as appear below.
       <p></p>
       ![img]({{ '/assets/images/espionage_tibet_image/lowzero_34.png' | relative_url }}){: .center-image }*(**Setup a listener**)* 
    
       ![img]({{ '/assets/images/espionage_tibet_image/lowzero_35.png' | relative_url }}){: .center-image }*(**Connecting to other host based on data received from remote connection**)*
       
       
# Classification and Attribution       

- I find out that the scheme of this RTF document is similar to ones that got generated by a tool called [**Royal Road weaponizer**](https://nao-sec.org/2020/01/an-overhead-view-of-the-royal-road.html) which typically use the exploits to target vulnerabilities in equation editor.

- Also, the [**yara rule**](https://github.com/nao-sec/yara_rules/blob/master/RoyalRoad_Classification/RoyalRoad_version.yar) classify the RTF file as RoyalRoad version 7b.

    ![img]({{ '/assets/images/espionage_tibet_image/lowzero_36.png' | relative_url }}){: .center-image }*(**Classifying the RTF to be generated by royalRoad version 7b**)*
    
- Based on the [**sheet**](https://docs.google.com/spreadsheets/d/1lDzylI6Jymz7EE0agRVUsL3kwmJSRDjXYjr5l5MUOEk/edit#gid=127522608) which is in the previous report, this tool is used by many of Chinese threat actors.    
 
- However, one of the actors in the sheet,**TA413**,perfrom an attack that contains the same embedded executable object name **ghb4nrwmp.wmf**.   
    

# Tool Tracking

- To track the tool, I thought to use the features that exist in enterprise virus-total where I can register initial yara rules and retrieve the result files or dynamic information extracted from these files by the available sandboxes.

- As appears the below diagram, my plan is to retrieve IOC(s) like mutexes' names if exist, possible C2 IP (s), and hashes from samples that match with the initial yara rules. Then, implement a python script to utilize these newly extracted IOC(s) to generate other yara rules to find new variants in the wild.

- The database hold the IOC(s) from various variants, can be used later in security solutions in client environments to detect malicious activities for this malware.    

  ![img]({{ '/assets/images/espionage_tibet_image/lowzero_37.png' | relative_url }}){: .center-image }*(**Scheme to track the tool**)*

- I thought to build two initial yara rules, the first one; which named **detect_evil_rtf** under the section "**Scripts and yara rules**", to search for similar malicious RTF that lead to open/create the detected mutex or communicate with the detected C2 server IP.
 
- Another yara rule is used to track the files that are similar to the two binaries in the kill-chain; which is named **detecting_toolset** under the section "**Scripts and yara-rules**". I find that both binaries used almost the same toolset for the compilation as both files have the same **ProductId** and **BuildId** which appear in both rich headers as appear below.
    
  ![img]({{ '/assets/images/espionage_tibet_image/lowzero_38.png' | relative_url }}){: .center-image }*(**Rich header for ghb4nrwmp.wmf**)*

  ![img]({{ '/assets/images/espionage_tibet_image/lowzero_39.png' | relative_url }}){: .center-image }*(**Rich header for backdoor.dll**)*


# Indicators of compromise (IOC)


| IOC | Information | 
| -------- | -------- | 
| 45.77.19.75| C2 IP address|
|8C9BB583-7C26-4990-AA73-E66F594A5AD5 | Mutex name|
| 9681ef910820d553e4cd54286f8893850a3a57a29df7114c6a6b0d89362ff326| SHA-256 hash for the RTF document|
|ba9c8bccf506fd65ce33f87608f90b187a2b4b85c16764e8553d0528055a0982|SHA-256 hash for the encrypted ghb4nrwmp.wmf|
|5ea5a1498ac9ce9f0ecd4d704fb938d2dfb375b81eca413bccab0322bbb9a4f9|SHA-256 for the decrypted ghb4nrwmp.wmf but with corrupted e-magic |
|26ffbc7ccf17e711b8971ca065623659d056ad86fe651a29e01d9143b41a999d|SHA-256 after decryption and fixing e-magic of file ghb4nrwmp.wmf |
|8f61327aaff6796ef37a80544a948b4746846deaa2738e5942c6e2faaf102345| SHA-256 for the embedded exploit in the RTF file |
|0f44c2e3418a51ff42a6c6abf484263a404a24c0a1a131b309b01c5de18fab9c|SHA-256 for the backdoor.dll after fixing it Dos header|
{:.inner-borders}



# Scripts and yara rules

The following python script used to decode the **ghb4nrwmp.wmf** and the decoded bytes into file.

>python
{:.filename}
{% highlight python %}
 readFile = open("ghb4nrwmp.wmf","rb")
 dataBytes = readFile.read()
 readFile.close()
 writeFile = open("decoded_executable","wb")
 for Byte in dataBytes:
   b = Byte ^ 0xfc
   writeFile.write(b.to_bytes(1,'big')) 
 writeFile.close()
{% endhighlight %}

This rule can be used to detect the RTF file that can lead to that open/create the detected mutex or communicate with  the detected c2 server IP.

>detect_evil_rtf.yar
{:.filename}
{% highlight ruby %}
import "vt"
import "cuckoo"
 rule detect_evil_rtf {
   meta:
     description= "try to find the malicious  rtf that is executed"

   strings:
     $exploit_object="4571756174696F6E2E32" //Equation.2
  
   condition:
       vt.metadata.file_type == vt.FileType.RTF and
       $exploit_object and
       (cuckoo.network.host(/45\.77\.19\.75/) or
       (vt.behaviour.mutexes_opened == "8C9BB583-7C26-4990-AA73-E66F594A5AD5" or
       vt.behaviour.mutexes_created == "8C9BB583-7C26-4990-AA73-E66F594A5AD5"))
        }

{% endhighlight %}





This yara rule for detecting the common toolset used in building **ghb4nrwmp.wmf** and **backdoor.dll**.

>detecting_toolset.yar
{:.filename}
{% highlight ruby %}
import "pe"
rule detecting_toolset{   
  meta:
   description= "this rule detect the toolsets that used to build both ghb4nrwmp.wmf and backdoor.dll"
   
  strings: 	
     $pe = "PE"
     $rich= "Rich"     
   
  condition:
     $pe and $rich and 
     pe.rich_signature.toolid(225, 20806) and
     pe.rich_signature.toolid(223, 20806) and
     pe.rich_signature.toolid(223, 20806) and
     pe.rich_signature.toolid(147, 30729) and
     pe.rich_signature.toolid(1,0)        and
     pe.rich_signature.toolid(228, 21005) and
     pe.rich_signature.toolid(222, 21005)
       }
{% endhighlight %}


