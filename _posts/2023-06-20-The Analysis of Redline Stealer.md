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

- After establishing the communication channel and ensuring that C2 is alive, the RedLine will start fetching the settings from the C2, as appears below.

  ![img]({{ '/assets/images/Redline/redline-12.png' | relative_url }}){: .center-image }*(**Fetching the settings**)* 

- The below screenshot is for the settings, and it consists of the following types of fields:
   
   1. Boolean fields that act as switches for whether do specific functionality or not.
   
   2. List of file patterns to exfiltrate.
   
   3. List of paths and profiles for browsers and applications, to steal important data from their files like cookies and login data.   

   4. List of wallet information in which every object in the list contains wallet application name, paths, and patterns for the files to exfiltrate to the C2 server.
 
  ![img]({{ '/assets/images/Redline/redline-13.png' | relative_url }}){: .center-image }*(**The fetched settings**)* 

# Fingerprinting the infected machine

- For every infected device, the RedLine will try to fingerprint the infected machine by collecting the following information and sending them to the C2. 

    - First, It will try to extract the public IPv4 if any of the C2 addresses has a port equal to 80 or 81. Otherwise, it returns the string **unknown**.

      ![img]({{ '/assets/images/Redline/redline-14.png' | relative_url }}){: .center-image }*(**Extracting public IP for the infected machine**)*  

      ![img]({{ '/assets/images/Redline/redline-15.png' | relative_url }}){: .center-image }*(**Saving the public IP to struct**)*  
 
    - Also, it collects information about whether the machine is pre-infected or not. If it finds a folder named **SystemCache** exists within the last 10 days, then it will return true, indicating that the machine is already infected.          

      ![img]({{ '/assets/images/Redline/redline-16.png' | relative_url }}){: .center-image }*(**Checking if machine is pre-infected**)*  

      ![img]({{ '/assets/images/Redline/redline-17.png' | relative_url }}){: .center-image }*(**Saving boolean value for pre-infection status to struct**)*


    - Finally, it gathers the following listed information and then sends all of them to the C2, as appears in the below screenshot.

       1. The `Username` for the current thread.

       2. The `Virtual display size`.
       
       3. The `Current input language` that is used at the moment.
       
       4. The `Windows version`, represented by a string that concatenates the operating system bitness (32/64 bit) and value of the **ProductName** at the registry path **HKEY_LOCAL_MACHINE\\SOFTWARE\\Microsoft\\Windows NT\\CurrentVersion**.

       5. The `Disk location` of the executing RedLine sample.
       
       6. The `Device signature` is estimated by the computation of the **MD5** hash for the concatenation of **Username**, **UserDomainName**, and a **Disk drive's serial number**.

      ![img]({{ '/assets/images/Redline/redline-18.png' | relative_url }}){: .center-image }*(**Collecting and sending device information to C2**)*


# Collecting Hardware information

- RedLine collects and sends the following hardware specifications to C2:

  1. The installed **Processor(s)** names and their number of cores.

     ![img]({{ '/assets/images/Redline/redline-19.png' | relative_url }}){: .center-image }*(**Collecting Processors specifications**)*

  3. The installed **Graphics card(s)** names and their adapter RAM sizes.

     ![img]({{ '/assets/images/Redline/redline-20.png' | relative_url }}){: .center-image }*(**Collecting Graphics cards specifications**)*

  4. The installed **RAM** size is in bytes and megabytes units

     ![img]({{ '/assets/images/Redline/redline-21.png' | relative_url }}){: .center-image }*(**Collecting RAM size**)*
     

# Collecting installed browsers' information

- RedLine retrieves information like names, paths, and versions for the installed browsers and sends them to the C2 server.

  ![img]({{ '/assets/images/Redline/redline-22.png' | relative_url }}){: .center-image }*(**Retrieving the installed browsers' information**)*



# Listing the installed programs

- RedLine lists all the installed programs' names and versions and then sends the list to the C2 server.

  ![img]({{ '/assets/images/Redline/redline-23.png' | relative_url }}){: .center-image }*(**Listing the installed programs**)*


# Finding the installed security products

- RedLine collects the names of the installed Antivirus, AntiSpyWare, and Firewall products and sends these names to the C2 server.

  ![img]({{ '/assets/images/Redline/redline-24.png' | relative_url }}){: .center-image }*(**Collecting the installed security products names**)*


# Retrieving the running processes

- RedLine retrieves information like ID, Name, and CommandLine for the running processes. After that, it sends them to the C2 server.

  ![img]({{ '/assets/images/Redline/redline-25.png' | relative_url }}){: .center-image }*(**Retrieving the running processes information**)*

# Listing the available languages

 - RedLine collects all available input languages and then exfiltrates them to the C2 server.

   ![img]({{ '/assets/images/Redline/redline-26.png' | relative_url }}){: .center-image }*(**Retrieving the available languages**)*

# Stealing Telegram application files

- RedLine searches for the Telegram application directory called **tdata** where application data like session data, messages, and images are stored. Also, it looks for other **tdata** sub-directories whose names are 16 characters long.
  
  ![img]({{ '/assets/images/Redline/redline-27.png' | relative_url }}){: .center-image }*(**Searching for Telegram application data directory**)*  

- Then, it steals all the files under the previously mentioned directories and exfiltrates them to the C2 server.

  ![img]({{ '/assets/images/Redline/redline-28.png' | relative_url }}){: .center-image }*(**Stealing files under Telegram application data directory**)* 

# Stealing data from browsers and applications

- RedLine searches a list of profile paths received within the settings, to locate the browsers' databases to steal data records like cookies, saved login, auto-fills, and credit cards as appeared below.

  ![img]({{ '/assets/images/Redline/redline-29.png' | relative_url }}){: .center-image }*(**Stealing a browsers' data**)*


- These previously mentioned databases are:

  - `Cookies`, to extract records of the **cookies** table.

  - `Extension Cookies`, to extract records of the **cookies** table.

  - `Network\\Cookies`, to extract records of **cookies** table

    ![img]({{ '/assets/images/Redline/redline-30.png' | relative_url }}){: .center-image }*(**Extracting cookies records**)*

  - `Login Data`, to extract records of the **logins** table which contains saved logins.

    ![img]({{ '/assets/images/Redline/redline-31.png' | relative_url }}){: .center-image }*(**Extracting login data records**)*

  - `Web Data`, to extract records for tables like **autofill** containing auto-fill data and **credit_cards** holding saved credit cards' data.

    ![img]({{ '/assets/images/Redline/redline-32.png' | relative_url }}){: .center-image }*(**Extracting auto-fill records**)*

    ![img]({{ '/assets/images/Redline/redline-33.png' | relative_url }}){: .center-image }*(**Extracting credit cards records**)*


- Also, it tries to steal cookies from Firefox-based browsers by searching for a database called **cookies.sqlite**.

  ![img]({{ '/assets/images/Redline/redline-34.png' | relative_url }}){: .center-image }*(**Stealing Firefox-based browsers cookies**)*
   
- If it finds the **cookies.sqlite** database, Then, it extracts the cookies records from the table named **moz_cookies**.

  ![img]({{ '/assets/images/Redline/redline-35.png' | relative_url }}){: .center-image }*(**Extracting cookies from moz_cookies table**)*

- Below is the list of browsers and applications whose data are stolen.
  <details> <summary>Targetted browsers and applications</summary><br>
  <b>Battle.net</b> - <b>Chromium</b> - <b>Chrome</b> - <b>Opera Software</b> - <b>ChromePlus</b> - <b>Iridium</b> - <b>7Star</b> - <b>CentBrowser</b> <b>Chedot</b> - <b>Vivaldi</b> - <b>Kometa</b> - <b>Elements Browser</b> - <b>Epic Privacy Browser</b> - <b>Uran</b> - <b>Sleipnir5</b> - <b>Citrio</b> <b>Coowon</b> - <b>liebao</b> - <b>Orbitum</b> - <b>Comodo Dragon</b> - <b>Amigo</b> - <b>Torch</b> - <b>Yandex</b> - <b>360Browser</b> - <b>Maxthon3</b>  <b>K-Melon</b> - <b>Sputnik</b> - <b>Nichrome</b> - <b>CocCoc</b> - <b>Chromodo</b> - <b>Mail.Ru Atom</b> - <b>Brave-Browser</b> - <b>Microsoft Edge</b> -
<b>NVIDIA GeForce Experience</b> - <b>Steam</b> - <b>CryptoTab Browser</b> - <b>Mozilla Firefox</b> - <b>Waterfox</b> - <b>Thunderbird</b> - <b>Comodo IceDragon</b> - <b>Cyberfox</b> - <b>BlackHaw</b> - <b>Pale Moon</b> - <b>QIP Surf</b>
  </details>



# Collecting FileZilla Credentials

- RedLine searches for the **FileZilla** XML files like **recentservers.xml** and  **sitemanager.xml** which contain login credentials for FTP servers.

   ![img]({{ '/assets/images/Redline/redline-36.png' | relative_url }}){: .center-image }*(**Looking for FileZilla credentials files**)*   

- Once it finds them it extracts all host's URLs, username,s and passwords and send them to the C2 server.

   ![img]({{ '/assets/images/Redline/redline-37.png' | relative_url }}){: .center-image }*(**Extracts stored FTP servers' credentials**)*   


# Taking a screenshot

- RedLine copies the screen and converts it to bytes in order to send it to the C2 server.

  ![img]({{ '/assets/images/Redline/redline-38.png' | relative_url }}){: .center-image }*(**Taking a screenshot**)*



# Exfilterating files matching received patterns

- RedLine exfiltrates the files that match any of the patterns that exist in the received settings. Every pattern string has the format **Disk path \| regex1, regex2,regex3, ... \| search-option constant value**. The below are received patterns from C2.

    - **%userprofile%\Desktop \| \*.txt,\*.rtf,\*.doc\*,\*key\*,\*wallet\*,\*seed\*,\*.jpg,\*.jpeg,\*.png,\*.pdf \|0**
    - **%userprofile%\Documents \|  \*.txt,\*.rtf,\*.doc\*,\*key\*,\*wallet\*,\*seed\*,\*.jpg,\*.jpeg,\*.png,\*.pdf \|0**
    - **%userprofile%\Downloads \| \*.txt,\*.rtf,\*.doc\*,\*key\*,\*wallet\*,\*seed\*,\*.jpg,\*.jpeg,\*.png,\*.pdf \|0**

<p></p>
- If the disk path field equals **%DSK_23%**, then it will search all logical drives to find and exfiltrate the matched files.

  ![img]({{ '/assets/images/Redline/redline-39.png' | relative_url }}){: .center-image }*(**Searching all logical drives for matched files**)*
  

- In case of the disk path does not equal **%DSK_23%**, then it will search inside that path based on the search option value, to find matched files. 

  ![img]({{ '/assets/images/Redline/redline-40.png' | relative_url }}){: .center-image }*(**Searching inside a specific path for matched files**)* 

- Also, note that in all cases, the exfiltrated file mustn't be more than **3097152** bytes and the summation of exfiltrated files' sizes mustn't be more than **52428800** bytes.

 
  
# Stealing Crypto wallets files

- RedLine looks for the directories that contain any of the files named **wallet.dat** or **wallet** and then exfiltrates all the files in these directories that match the regex pattern **"\*wallet\*"**.

  ![img]({{ '/assets/images/Redline/redline-41.png' | relative_url }}){: .center-image }*(**Looking for wallet files**)*

- Additionally, it searches for the following folders of crypto and authenticator extensions inside the browsers paths. When it finds these folders, it will exfiltrate all their files as regex **"\*"** is specified.

   ![img]({{ '/assets/images/Redline/redline-42.png' | relative_url }}){: .center-image }*(**Stealing files of Crypto wallets browser extensions**)*
 
| Folder name \<Key\>| Wallet/Authenticator \<Value\>|
|--------------------|-------------------------------|
| ffnbelfdoeiohenkjibnmadjiehjhajb | YoroiWallet|  
| ibnejdfjmmkpcnlpebklmnkoeoihofec | Tronlink|
| jbdaocneiiinmjbjlgalhcelgbejmnid | NiftyWallet|   
| nkbihfbeogaeaoehlefnkodbefgpgknn | Metamask|  
| afbcbjpbpfadlkmhmclhkeeodmamcflc|MathWallet|  
|hnfanknocfeofbddgcijnmhnfnkdnaad|Coinbase|  
|fhbohimaelbohpjbbldcngcnapndodjp |BinanceChain|  
|odbfpeeihdkbihmopkbjmoonfanlbfcl| BraveWallet|
|hpglfhgfnhbgpjdenjgmdgoeiappafln| GuardaWallet|
|blnieiiffboillknjnepogjhkgnoapac| EqualWallet|
| cjelfplplebdjjenllpjcblmjkfcffne | JaxxxLiberty|
| fihkakfobkmkjojpchpfgcmhfjnmnfpi |BitAppWallet|
| kncchdigobghenbbaddojjnnaogfppfj | iWallet|
| amkmjjmmflddogmhpjloimipbofnfjih | Wombat|
| fhilaheimglignddkjgofkcbgekhenbh | AtomicWallet|
| nlbmnnijcnlegkjjpcfjclmcfggfefdm | MewCx|
| nanjmdknhkinifnkgdcggcfnhdaammmj | GuildWallet|
| nkddgncdjgjfcddamfgcmfnlhccnimig | iWallet|
| kncchdigobghenbbaddojjnnaogfppfj | SaturnWallet|
| fnjhmkhhmkbjkkabndcnnogagogbneec | RoninWallet|
| aiifbnbfobpmeekipheeijimdpnlpgpp | TerraStation|
| fnnegphlobjdpkhecapkijjdkgcjhkib | HarmonyWallet|
| aeachknmefphepccionboohckonoeemg | Coin98Wallet|
| cgeeodpfagjceefieflmdfphplkenlfk | TonCrystal|
| pdadjkfkgcafgbceimcpbkalnfnepbnk | KardiaChain|
| bfnaelmomeimhlpmgjnjophhpkkoljpa | Phantom|
| mgffkfbidihjpoaomajlbgchddlicgpn | PaliWallet|
| aodkkagnadcbobfpggfnjeongemjbjca | BoltX|
| kpfopkelmapcoipemfendmdcghnegimn | LiqualityWallet|
| hmeobnfnfcmdkdcmlblgagmfpfboieaf | XdefiWallet|
| lpfcbjknijpeeillifnkikgncikgfhdo | NamiWallet|
| dngmlblcodfobpdpecaadgfbcggfjfnm | MaiarDeFiWallet|
| bhghoamapcdpbohphigoooaddinpkbai | Authenticator|
| ookjlbkiijinhpmnjffcofjonbfbgaoc | TempleWallet|
{:.inner-borders}

<p></p>

- Finally, It steals files from the list of crypto wallets in the received settings. The following table contains the name of the targetted crypto wallet applications, directories to search, and file matching patterns for every directory to exfiltrate to C2.

| Wallet name       | Directory path                                 | Patterns 
|-------------------|----------------------------------------------- |--------- |
| Armory | %appdata%\Armory|\*.wallet|    
| Atomic | %appdata%\atomic|\*|
| Binance | %appdata%\Binance|\*app-store\*|   
| Coinomi | %localappdata%\Coinomi\Coinomi\Cache |\*|  
|         | %localappdata%\Coinomi\Coinomi\db |\*|  
|         |%localappdata%\Coinomi\Coinomi\wallets |\*|  
|Electrum | %appdata%\Electrum\wallets           |\*|  
|Ethereum| %localappdata%\Ethereum\wallets       |\*|
|Exodus| %appdata%\Exodus\exodus.wallet          |\*|
|        | %appdata%\Exodus\exodus               |\*.json|
| Guarda | %appdata%\Guarda                      |\*|
| Jaxx    | %appdata%\com.liberty.jaxx           |\* |
| Monero | %userprofile%\Documents\Monero\wallets  |\*|
{:.inner-borders}
  


# Stealing Discord tokens

- RedLine searches the directory with path **\%appdata\%\\discord\\Local Storage\\leveldb**, to look for the files matching any of the following regex patterns:

  - **"\.log**  
  - ***\.ldb**

  ![img]({{ '/assets/images/Redline/redline-43.png' | relative_url }}){: .center-image }*(**Searching for discord files**)*

- After that, it looks into the matched files to find the tokens using the regex **[A-Za-z\\\d]{24}\\\.[\\\w-]{6}\\\.[\\\w-]{27}**.

  ![img]({{ '/assets/images/Redline/redline-44.png' | relative_url }}){: .center-image }*(**Finding the discord tokens**)*

- Finally, it saves these tokens under a text file called **Tokens.txt** and uploads it to the C2 server. 

  ![img]({{ '/assets/images/Redline/redline-45.png' | relative_url }}){: .center-image }*(**Saving the tokens into a text file**)*



# Collecting Steam files

- Redline searches the installation path for [Steam](https://en.wikipedia.org/wiki/Steam_(service)) to steal the files with the following patterns:

  -  **\*ssfn\***, the files that match this pattern can be used to do auto-login.
  -  **\*.vdf**, the files that match this pattern contain usernames that are logged in and configurations.

  ![img]({{ '/assets/images/Redline/redline-46.png' | relative_url }}){: .center-image }*(**Stealing Steam application files**)*


# Stealing VPN files

- RedLine targets the VPN applications' files, for example, it looks for the files that match the **\*ovpn** pattern inside application data folders for **OpenVPN** and **ProtonVPN**. 

  ![img]({{ '/assets/images/Redline/redline-47.png' | relative_url }}){: .center-image }*(**Stealing ovpn files for OpenVPN**)*

  ![img]({{ '/assets/images/Redline/redline-48.png' | relative_url }}){: .center-image }*(**Stealing ovpn files for ProtonVPN**)*

- Also, it looks for XML files with the name **user.config** inside the application data folder for the **NordVPN**, to steal the saved usernames and passwords.

  ![img]({{ '/assets/images/Redline/redline-49.png' | relative_url }}){: .center-image }*(**Stealing NordVPN Credentials**)*


# Indicators of compromise (IOCs)




