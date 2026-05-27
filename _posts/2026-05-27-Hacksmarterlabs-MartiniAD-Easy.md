---
title: "HackSmarter MaritiniAD Walkthrough "
date: 2026-05-27 00:00:00 +0000
categories: [ HackSmarter]
tags: [Hacksmarter , Linux , Easy]
---


## Scenario : 

An adult beverage company "Martini Bars" recently had a corporate breach and the compliance and risk team dictates they perform a penetration test at one of their branch offices. The Hack Smarter team has been authorized to perform an internal black box pentest.

## Foothold  :  

First thing first we run an nmap scan : 

```bash
nmap -p- -Pn $target -v --min-rate 1000 --max-rtt-timeout 1000ms --max-retries 5 -oN Open_Ports.txt && sleep 5 && nmap -Pn $target -sV -sC -v -oN Nmap_sV_sC_Results.txt && sleep 5 && nmap -T5 -Pn $target -v --script vuln -oN Nmap_Vuln_Results.txt
```

<img width="1066" height="734" alt="image" src="https://github.com/user-attachments/assets/45e3d7d6-f48d-4386-868b-603f5d017c15" />

From the initial scan , we can tell this is an Active Directory box , port 88 for kerberos , 389 for ldab , this is probably the DC . since this is an AD machine i usually follow this checklist  : 

```bash
Scan All TCP Ports . 
Check ldap , rpc , smb with anonymous access to get users.
Check for Public Shares .
Find Usernames : Find usernames in Web Site and generate a Username List maybe . if you get access , Look for usernames using netexec , –rid-brute , –users and enumdomuser with RPC client .
Check usernames found with kerbrute .
Search for ASREP Roastable Users .
Once we have valid Creds : Check Kerberoastable users + Spray the password on all users .
Try to authenticate using every protocol .
Enumerate shares with those Users found . Use netexec to download .
If we get a Shell , try privesc , dump all Hashes using netexec or locally and store into a file .
If we can’t privesc , we can move to Blood Hound .
```

Of course there are other steps with more details in my methdology , but for now we can start with anonymous access , first checking to get a list of usernames to try asreproasting . But before that we need to add the Hostname as well as the domain name to our /etc/hosts 

<img width="1669" height="336" alt="image" src="https://github.com/user-attachments/assets/dcb96599-d9ca-42c6-a63a-4a9bb237332e" />

Now let's check for users via anonymous access : 

<img width="1658" height="642" alt="image" src="https://github.com/user-attachments/assets/9f5e84ad-2879-40cf-820c-7dd40de1e2e2" />

We do get a list of usernames , now to keep usernames only : 

```bash
nxc smb $target -u 'guest' -p '' --rid-brute   | grep -i 'sidtypeuser' | awk '{print$6}' | cut -d '\' -f 2 | tee users.txt
```

<img width="1322" height="479" alt="image" src="https://github.com/user-attachments/assets/c356732c-a0d7-4ce4-84d9-45c109c96dc4" />

Now let's try ASREProasting : 
```bash
impacket-GetNPUsers  DRY.MARTINI.BARS/ -dc-ip $target -usersfile users.txt -outputfile Hashes.txt
```
<img width="1092" height="546" alt="image" src="https://github.com/user-attachments/assets/89c2cf7a-3ee0-47fd-baaf-e6e103e43042" />

All users dont have the UF_DONT_REQUIRE_PREAUTH which means ASREPROASTING isn't possible in this case . Now let's check the shares : 

<img width="1163" height="424" alt="image" src="https://github.com/user-attachments/assets/c8a31268-aabe-4c10-9aed-09abcf773742" />

Perfect , we see the Notes share that we can enumerate since we have R/W . 

<img width="1093" height="529" alt="image" src="https://github.com/user-attachments/assets/421ca45a-aeb8-4b20-90f4-c3628af2ecc2" />

Perfect we find a new user , once we validate it using NXC , first thing i like to check is Kerberoasting : 

<img width="1149" height="534" alt="image" src="https://github.com/user-attachments/assets/55c8b1de-c139-47df-890c-a75efffbd828" />

We can use Jhon or Hashcat to crack the hash . 

<img width="1078" height="391" alt="image" src="https://github.com/user-attachments/assets/b8ed157b-3c91-49f3-9e0d-0830e65950c9" />

Perfect now we got our new user , we see that he is not Admin , let's see if we can login via Winrm . 

<img width="1345" height="261" alt="image" src="https://github.com/user-attachments/assets/6b521dbd-e49f-4372-b346-a6a9fcd26cf1" />

Perfect , now let's login and check for privilege escalation vectors . 


## Privilege Escalation : 

I tried login in , tried multiple tools , PowerUp.ps1 , Winpeas , checked the groups and Tokens for pivesc but nothing worked . 

<img width="1383" height="712" alt="image" src="https://github.com/user-attachments/assets/f608a9a0-79c5-4622-a3eb-10698afcd649" />

Now before running Bloodhound , what i like to do is check for password reuse whenever i have a valid password , we can do that using the netexec and the list of usernames we got earlier . 

<img width="1341" height="422" alt="image" src="https://github.com/user-attachments/assets/63107c86-a06c-4837-92c8-6dc70afe8f6e" />

Perfect we see that the user Athena.T0 also uses the same password , and this user is Administrator on this machine which is also the DC . 

Now we can either login using this user via RDP or Winrm or just dump all Hashes since the user is Admin . 

<img width="1641" height="527" alt="image" src="https://github.com/user-attachments/assets/06734d14-b697-46de-bb61-c9387baed42a" />

Ah forgot , the flag is just the KRBTGT Hash , so no need to even login . 

That was it for this AD Lab , see you in the next one :) 



