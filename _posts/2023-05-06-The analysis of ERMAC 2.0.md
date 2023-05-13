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


# Finding an active C2 URL

- In case the response from pinging the C2 server ( equivalent to sending a checkAP command ) is empty, it means that this Url is down. The bot will start looking for another active URL from a list of URLs that is a part of the bot settings.

- This is done by extracting the value of key **urls** from the shared preference which is called **settings**, then this value is split to form an array of URLs as appears in the below screenshot.  

![img]({{ '/assets/images/ermac_22.png' | relative_url }}){: .center-image }*(**Parsing URL(s)**)*

- After that, it pings every URL of the array by sending a **checkAP** command, to check whether it's active or not. If an active URL is found, subsequently, the key **urlAdminPanel** for the default URL value will be set to the active URL, as appears in the below code.

![img]({{ '/assets/images/ermac_23.png' | relative_url }}){: .center-image }*(**Searching for an active URL**)*


# The Bot Registration
- When the bot gets a response of **~no~** from pinging the C2 indicating that the bot is not registered. Then the bot registration function will get executed as appear below.  

  ![img]({{ '/assets/images/ermac_24.png' | relative_url }}){: .center-image }*(**Registering the bot**)*

- In order to register the bot, a **Registration** command with the following device information in the form of a JSON object is sent to the C2 server:
  - The key **id** holds a value of botID
  - The key **country** holds a value of country name of locale.
  - The key **countryCode** hold an iso string equivalent to **MCC** code.
  - The key **tag** hold string **Tag1**
  - The key **isDualSim** holds true when the phone has dual sim or false if not.
  - The key **operator** holds the name of operator for sim#1.
   <p></p>
   ![img]({{ '/assets/images/ermac_25.png' | relative_url }}){: .center-image }*(**Collecting device information snippet [1]**)*
  
  - The key **phone_number** holds a value of phone number for sim#1.
  - The key **operator1** holds the name of operator for sim#2.
  - The key **phone_number1** holds a value of phone number for sim#2.
  - The key **android** holds a value of `Build.VERSION.RELEASE` which is android version.
  - The key **model** holds an information about the model and manufacturer of the device.
  - The key **batteryLevel** holds the integer percentage of total battery capacity.
  - The key **imei** holds International Mobile Equipment Identity (aka [IMEI](https://en.wikipedia.org/wiki/International_Mobile_Equipment_Identity))
  <p></p>
   ![img]({{ '/assets/images/ermac_26.png' | relative_url }}){: .center-image }*(**Collecting device information snippet [2]**)*
  
  - The key **accessibility** holds true if the accessibility service enabled or false if it's not.
  - The key **protect** holds the state of google play protection whether enabled or disabled.
  - The key **admin** holds the state of whether the device admin component enabled or not.
  - The key **screen** contains the value that refers to a state of the screen display whether on or off.
  <p></p>
  ![img]({{ '/assets/images/ermac_27.png' | relative_url }}){: .center-image }*(**Collecting device information snippet [3]**)*
  
  - The key **isKeyguardLocked** holds the value that states whether screen is locked or not.
  - The key **is_dozemode** holds the value to show whether the virus application is in the power allow list or not.
  - The key **sms** holds the value that states whether `android.permission.READ_SMS` is granted or not.
  - The key **set_contact_list** holds the value that states whether `android.permission.READ_CONTACTS` is granted or not.
  <p></p>
  ![img]({{ '/assets/images/ermac_28.png' | relative_url }}){: .center-image }*(**Collecting device information snippet [4]**)*
  
  - The key **set_hide_sms_list** holds the value that states whether `android.permission.RECEIVE_SMS`is granted or not.
  - The key **set_windows_fake** holds a value that states whether all of the needed permissions are enabled or not.
  - The key **set_accounts** holds the value that states whether `android.permission.GET_ACCOUNTS`is granted or not.
  <p></p>
  ![img]({{ '/assets/images/ermac_29.png' | relative_url }}){: .center-image }*(**Collecting device information snippet [5]**)*

- Whenever the bot registration is completed, indicated by receiving a response that contains **"ok"**, It sets the key **checkUpdateInjection** with value **"1"** in the shared preference. This signal the bot to communicate with the C2 server to install updated HTML injections which will be covered in detail under the upcoming **"The bot commands"** section. 

  ![img]({{ '/assets/images/ermac_30.png' | relative_url }}){: .center-image }*(**Signaling the bot to install updated HTML injections**)*
 

# The Bot settings
- If the response from pinging the C2 server is non-empty and contains the key **action** with a value **~settings~**, then The following keys in the shared preference, which represent the bot's settings get updated from the C2 server.    
 
  ## urls
  - The key **urls** in shared preference, holds a semicolon-delimited string containing alternative URL(s) for C2. These URL(s) are parsed from the JSONObject that represents the response of sending the **checkAP** command as appears in the below screenshot.
   <p></p>
   ![img]({{ '/assets/images/ermac_31.png' | relative_url }}){: .center-image }*(**Setting alternative URL(s) for C2**)*
 
 
  ## lockDevice
  - This key in the shared preference, is set by the C2 server, as appears below. its value instructs the bot to lock the device screen or not.
   <p></p>
   ![img]({{ '/assets/images/ermac_32.png' | relative_url }}){: .center-image }*(**Setting the LockDevice key**)*
  
  - If the key **lockDevice** is set to **"1"**, then a new service gets executed which is responsible for locking the device screen.
   <p></p>
   ![img]({{ '/assets/images/ermac_33.png' | relative_url }}){: .center-image }*(**Starting the service that locking the screen**)*
  
  - The following line, located inside the infinite loop of the service code is responsible for locking the screen.
   <p></p>
   ![img]({{ '/assets/images/ermac_34.png' | relative_url }}){: .center-image }*(**locking the screen**)*
  


  ## hiddenSMS
  - This key within the shared preference, is set with the value of the key **hideSMS**, which is part of the response of pinging the C2 server, as appears in the below screenshot.   
    
    ![img]({{ '/assets/images/ermac_35.png' | relative_url }}){: .center-image }*(**setting HiddenSMS key**)*
  
  - As appear in the below code, when the value of this key is equal to **"1"**, the needed permission is granted and the default SMS manager not equals the virus package name. Then, the bot requests the user to enable a virus application as the default SMS manager to hide the SMS(es). Also, it sets **autoClickSms** to **"1"** to signal the accessibility service to auto-click the enable button.
    
    ![img]({{ '/assets/images/ermac_36.png' | relative_url }}){: .center-image }*(**Change the default SMS manager**)*
  
  ## offSound
  - hello3
  ## keylogger
  - hello4
  ## clearPush
  - hello5
  ## readPush
  - hello6
  ## activeInjection
  - hello7
 
# The Bot commands

# Additional capabilities


