---
layout: post
title: "The analysis of ERMAC 2.0"
date: 2023-05-06
tags: Android Malware
description: The post shows the details about the capabilities of ERMAC android bank trojan version 2.0
---

# Intro
- **ERMAC** is a well-known Android bank trojan, developed based on another Android bank trojan called **Cerberus**. Its first version was detected targeting Poland in 2021 with the capability of stealing credentials. In This post, I will present the details of my analysis of the second version of the ERMAC trojan that is sold on darknet sites.   

# Check for interesting victims
- Before the malware proceeds to do any initialization or registration, it checks whether the victim device is interesting based on the following two factors: 
 
  ## Countries of interest 
  - For this factor, The malware tries to retrieve a country code string equivalent of the **Mobile Country Code (MCC)** to identify the country of the device by calling the API function `getNetworkCountryIso` as appears in the below screenshot. 
  
    ![img]({{ '/assets/images/ermac_1.png' | relative_url }}){: .center-image }*(**Getting country code for MCC**)*
  
  - In case This function returns an empty string, the malware will try to get the country code of the default locale as appear in the below screenshot.
     
     ![img]({{ '/assets/images/ermac_2.png' | relative_url }}){: .center-image }*(**Getting default locale's country**)*
     
  - The malware will consider the infected device as **interesting** if it's located out of the the countries with the country codes in the below screenshot.
     
     ![img]({{ '/assets/images/ermac_3.png' | relative_url }}){: .center-image }*(**Non-interesting country codes**)*     
  - These countries are the following:
    - Ukraine
    - Russian Federation
    - Belarus 
    - Tajikistan
    - Uzbekistan
    - Turkmenistan
    - Azerbaijan
    - Armenia
    - Kazakhstan
    - Kyrgyzstan
    - Moldova
     <br/>
   
  ## Existence of emulation
  - Regarding this factor, The malware will try to find whether it's executing in an emulator or a real device.
  - As appear the below screenshot, The emulator will get detected if the following Android build properties contain values that usually set in the emulating environment. 
    - `Build.FINGERPRINT` contains any strings like **"generic"** or **"unknown"**.
    - `Build.Model` contains any strings like **"google_sdk"**,  **"Emulator"** or **"Android SDK built for x86"**.
    - `Build.MANUFACTURER` contains string **"Genymotion"**.
    - `Build.BRAND` contains string **"generic"**.
    - `Build.DEVICE` contains string **"generic"**.
    - `Build.PRODUCT` equals string **"google_sdk"**.
  
  <p></p>
  ![img]({{ '/assets/images/ermac_4.png' | relative_url }}){: .center-image }*(**Emulation detection**)*
   
- Therefore, The victim device is an **interesting** if it's not an emulator beside being located out of the previously mentioned countries. 


# Perform initialization

- If the victim device is interesting, the malware will start initializing certain keys of shared preference called **settings**.
- First, it generated bot id that matches the regex **[a-z0-9]{17}** then save it under key named **idbot** in shared preferences as appear in the below screenshot.

![img]({{ '/assets/images/ermac_5.png' | relative_url }}){: .center-image }*(**Bot ID generation**)*

- After that, it initializes the shared preference with the following as appears in the below screenshot:
   
  - Set key **urlAdminPanel** which is default C2 url, with string value **hxxp://185[.]215[.]113[.]59:3434**.
  - Set key **initialization** with value **good**.
  - Set key **events** which hold the logs, with empty string.
  - Set key **datakeylogger** which later will hold keylogging data, with empty string.      
      
![img]({{ '/assets/images/ermac_6.png' | relative_url }}){: .center-image }*(**Bot's initialization**)*
 <br/>

# Enable the accessibility service

- So that the malware performs its functionalities and enables the needed permissions, it needs the user to enable its accessibility service.
- In order to achieve that, The malware disguises itself as a **Google Chrome** and requires the user to enable accessibility service.
- First, it performs base-64 decoding for encoded html code, then loads it in web view as appears in the below screenshots.
 
  ![img]({{ '/assets/images/ermac_7.png' | relative_url }}){: .center-image }*(**Decoding and loading html to web view**)*
   
  ![img]({{ '/assets/images/ermac_8.png' | relative_url }}){: .center-image }*(**The screen appears to user**)*
 
- When the user clicks on the above **GO TO SETTINGS**, the below code gets executed and shows the accessibility settings page.
  
  ![img]({{ '/assets/images/ermac_9.png' | relative_url }}){: .center-image }*(**Open up the accessibility settings page**)*
     
<br/>

# Granting the needed permissions

- For the purpose that the malware performing its functionalities, the following permissions are needed to be granted.

  ## Draw overlay permission

    - In order to draw over other applications screens, The draw overlay permission needed to be enabled. 
  
    - For other devices than Xiaomi, The following code is intended to show the screen that requests this permission. Also, it sets key **autoClickPerm** to **"1"** in the shared preference, to signal the accessibility service to perform auto-click on the slider button. 
  
      ![img]({{ '/assets/images/ermac_10.png' | relative_url }}){: .center-image }*(**Request Overlay Permission**)*  

    - As a proof of concept, I imitate the previous code considering that the application name My Application, which appears in the below screenshot, as the virus application name.

      ![img]({{ '/assets/images/ermac_11.png' | relative_url }}){: .center-image }*(**Overlay Request Screens**)*

    - Now comes the role of the accessibility service to auto-click on the above virus application name from the list of applications.

      ![img]({{ '/assets/images/ermac_12.png' | relative_url }}){: .center-image }*(**Select the virus application**)*  

    - After that, it clicks on the slider button as appears in the below code and sets **autoClickPerm** to an empty string indicating that no more overlay permission clicks are needed.
    
      ![img]({{ '/assets/images/ermac_13.png' | relative_url }}){: .center-image }*(**Clicking the slider button**)*  
      
    
  ## Display pop-up windows while running in the background permission
    - For the Xiaomi devices, this permission is needed to allow apps that run from the background to show pop-up windows, which is almost equivalent to the draw overlay permission.
    
    -  The following code open-up a window for requesting this permission regarding the virus application. Additionally, it sets **autoClickPerm2** to **"1"**, to signal the accessibility service to do auto-clicking to grant this permission.
          
       ![img]({{ '/assets/images/ermac_14.png' | relative_url }}){: .center-image }*(**Request a draw overlay for Xiaomi**)*      

    -  Briefly, similar to what happens regarding other devices, the accessibility service will check if the key **autoClickPerm2** is equal to **"1"**, then it will do auto-clicking to enable the permission.
      
       ![img]({{ '/assets/images/ermac_15.png' | relative_url }}){: .center-image }*(**Enabling a draw overlay for Xiaomi**)*


  ## Other permissions
     - In addition to the previous permissions, The malware is trying to grant the following permissions: 
       - `android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS`  
       - `android.permission.RECEIVE_BOOT_COMPLETED`
       - `android.permission.ACCESS_NETWORK_STATE`	 
       - `android.permission.WAKE_LOCK`
       - `android.permission.INTERNET`
       - `android.permission.READ_PHONE_STATE`
       - `android.permission.CALL_PHONE`	  
       - `android.permission.READ_CONTACTS`
       - `android.permission.GET_ACCOUNTS`
       - `android.permission.RECEIVE_SMS`
       - `android.permission.READ_SMS`
       - `android.permission.SEND_SMS`
       - `android.permission.READ_PHONE_NUMBERS` 

     - These permissions are set to the array called **permissionArray** and requested as appears in the below screenshot.
        
       ![img]({{ '/assets/images/ermac_16.png' | relative_url }}){: .center-image }*(**Enabling a draw overlay for Xiaomi**)*   
       
     - Again, the accessibility service will auto-click on the **ALLOW** button for the pop-ups of the requested permissions, by looking for the below view ids for that **ALLOW** button.    
       
       ![img]({{ '/assets/images/ermac_17.png' | relative_url }}){: .center-image }*(**Possible view ids for ALLOW button**)* 
     
     - Since a clickable view with an id that matches one of the previously mentioned view ids, has been found. Then, this view is auto-clicked, as appears in the below screenshot.
       
       ![img]({{ '/assets/images/ermac_18.png' | relative_url }}){: .center-image }*(**Auto-clicking on ALLOW button**)*
     
     
# Pinging the C2 server
- So that the bot makes sure that the default IP is still up, it tries to ping it by sending a JSON object with the following key values, which also appear in the below screenshot.
   - The key **command** is set to **checkAP**.
   - The key **id** is set to the generated **botID**.
   - The key **ticks** is set to the **working time** of certain service.
 
![img]({{ '/assets/images/ermac_19.png' | relative_url }}){: .center-image }*(**Sending checkAP command**)*


# The Traffic sending 
- When the bot communicates with the C2 server to send or receive traffic, the following two steps always occur:
  ## Generating random domains
  - Everytime, the bot try communicate with C2 it generate a domain with the following pattern **"urlAdminPanel/[a-z0-9]{1,20}.php"**, as appear in the following screenshot.
   <p></p>
  ![img]({{ '/assets/images/ermac_20.png' | relative_url }}){: .center-image }*(**Domain generation algorithm**)*
  
  ## Encrypting the traffic
  - Before sending any commands or logs, The bot encrypts the traffic with the AES algorithm with the below parameters followed by base-64 encoding, as appears in the below screenshot.
    - The `mode` is `AES/CBC/PKCS5Padding` 
    - The `IV parameter` equals `0123456789abcdef`
    - The `secret key` equals `1A1zP1eP5QGefi2DMPTfTL5SLmv7Divf`
  <p></p>
  ![img]({{ '/assets/images/ermac_21.png' | relative_url }}){: .center-image }*(**The encryption algorithm**)*


# Finding a working C2 IP

# The Bot Registration

# The Bot settings

# The Bot commands

# Additional capabilities


