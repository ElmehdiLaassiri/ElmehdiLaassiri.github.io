# Metasploitable 2 Walkthrought | By Elmehdi Laassiri :

![](https://miro.medium.com/v2/resize:fit:640/0*dBXtMyjsd6FaPG16.png)

Hello Everyone , Today we will be exploiting Metasploitable 2 . Metasploitable2 is a purposely vulnerable virtual machine designed for security testing and training purposes. It contains various intentionally exploitable vulnerabilities in its software stack, allowing users to practice penetration testing and learn about common security weaknesses in systems.

This shouldn’t be too difficult. The methodology here is quite simple: we scan for open ports, check if those ports host any vulnerable services, and exploit them. That’s all , Good luck !

# **1/ Enumeration :**

First thing i like to do is to run an nmap scan without any specifications , just to know which ports are open .

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*dpNqN_BXmqafchWzQ_-0bQ.png)

Then Once we identify the ports that are opened i like using the -sV -sC -p and then specify all the ports found , this way it s easier since we will only be scanning in detail the ports that are opened only and not all ports .

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*uIbufz8XTPeqFKpdC8yT5A.png)

- `sV`: This option tells `nmap` to attempt to determine the version of services running on the target ports.
- `sC`: This option tells `nmap` to scan with default scripts, which perform a set of common scripts against the target.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*cPz9z9cr7N-HTxoipEfdEA.png)

This was the Results , Now let s start exploiting each port :

# **2/ Exploitation :**

### **// Port 21 : FTP :**

> *FTP (File Transfer Protocol) is a standard network protocol used for the transfer of files between a client and server on a computer network. It operates on the TCP (Transmission Control Protocol) layer.*
> 

First thing we noticed after the scan is that FTP allow anonymous logins , so we ll try to connect to that using that .

![](https://miro.medium.com/v2/resize:fit:511/1*vOcuSX5eN13KmJ8iTAh9MA.png)

We got in but nothing really helpfull , Now let s see if the version of the vsftpd is vulnerable . We will be using a command in linux called searchsploit .

![](https://miro.medium.com/v2/resize:fit:622/1*GymZrBSLAJ2rBzyG0-7NJQ.png)

Nice There is an actual exploit for it . Let s run Metasploit and look for that exploit

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*B-hPlQeKWFfqteZjYbhHtA.png)

All we need to do is set the RHOST which is the ip adress of the Metasploitable2 machine , you can get it by running “ ifconfig ” command from your Metasploitable2 machine .

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*bzfb-VjAV1wCqKdfvVYS6w.png)

As you can see we got ROOT access without having to escalate our privileges . Once in all i did is to import our shell to be able to navigate the machine easly , we used this command for it :

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

### **//Port 25 : SMTP :**

> *SMTP (Simple Mail Transfer Protocol) is a protocol used for sending email messages between servers. It is responsible for transferring email from a sender’s mail server to the recipient’s mail server.*
> 

For the SMTP port, we can utilize an auxiliary module that conducts a dictionary attack to identify valid SMTP users. All we need to do is set the RHOST which is the ip of the target machine .

![](https://miro.medium.com/v2/resize:fit:600/1*IJihGa2Hh6aTK7fDrUFqKg.png)

Now we can run it .

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*IvXuvC-Z1oJE8jaHgascCw.png)

As you can see we have found some users : backup , bin , user , postmaster , Now let s check for their validity . We can do that using netcat or telnet .

![](https://miro.medium.com/v2/resize:fit:691/1*XMY-36xWiYMLEIbL80HKuA.png)

As evident from the responses, the first two users returned valid responses, while the last one did not.

### **// Port 5900 : VNC :**

> *VNC (Virtual Network Computing) port is a network protocol used to remotely access and control graphical desktops. It typically operates over TCP port 5900 and allows users to view and interact with the desktop environment of a remote system.*
> 

For this exact port , we will be using an auxilary that will try to brute force the password for the VNC , the auxilary is “scanner/vnc/vnc_login” :

![](https://miro.medium.com/v2/resize:fit:628/1*cYHIkMxGl8yRWx9dKhGIAA.png)

We will just need to set the RHOST for this and then run it .

![](https://miro.medium.com/v2/resize:fit:694/1*347hSzr_BkW9fF73Zzr5xg.png)

The retrieved password is “password”. When using this port, it is advisable to employ a password that is not easily guessable or commonly found within wordlists for more security .

### **// Port 22 : SSH :**

> *SSH (Secure Shell) is a cryptographic network protocol that provides secure access to a remote computer. It allows users to securely log in and execute commands on a remote machine over an unsecured network.*
> 

For the SSH port, we will be using an auxiliary module to brute force SSH logins. Usually, SSH is not supposed to be brute-forced since it’s difficult, but in this case, weak credentials are being used, so we can attempt it.

![](https://miro.medium.com/v2/resize:fit:612/1*RB6GBG7pFJ5hVe8jwC7vXw.png)

Here we will need to specify the RHOST , The file containing username and password .

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*D02jpavlNPyuoz7pRA7TSg.png)

Now all we need to do is to run it .

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*a1Jg-d7y8phaMRMw-3xw6A.png)

Here we can see that we have 2 sessions opened , the first one has user ; user as the username and password , and for the second one it s msfadmin ; msfadmin .

Let s Try it , to connect using ssh you need to type this command :

```
ssh user@Metasploitable2
```

You can also use msfadmin , both will work .

In case you encounter a problem regarding the Host Key type, you will just need to add the Host Key Type. In this case, we know that it is RSA from the nmap scan earlier. Here is the new command:

```
ssh -oHostKeyAlgorithms=+ssh-rsa username@192.168.87.128
```

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*mVSfNoyc3lrGGYM1nauQvQ.png)

Just like that we re in !

### **// Port 23 : Telnet :**

> *Telnet is a network protocol used to provide text-based communication between devices over a computer network. It was widely used for remote access to computers before the advent of SSH, but it is considered insecure due to its lack of encryption.*
> 

For the Telnet port , we will be using an auxilary that will brute force the telnet login .

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*c_1iaDxeMrmxyw4kc3JJlA.png)

As you can see we will be using the second one .

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*ENf4FAwwRpoQ7eCbfRgsCA.png)

We will just need to set up the RHOST , the Password File and Username File , and we re good to go .

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*gMh_GmrxOccfjozuwmIKuA.png)

Now just run it and let s hope we get a match .

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*wKAP9cJ9-vbKYTcgKKXSyw.png)

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*ZkyDsslZTANJJDtUvJzYQA.png)

Great! We’ve successfully obtained credentials for two sessions: “user/user” and “msfadmin/msfadmin” .

### **// Information Disclosure :**

> *Information Disclosure refers to the accidental exposure of sensitive data to unauthorized individuals.*
> 

We saw that Port 80 was open , let s go visit the website :

![](https://miro.medium.com/v2/resize:fit:625/1*LMevMyj83yvlHCM4xEIklw.png)

Let s visit the multillidae :

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*-90sKC8NDapbkJ3qWSYowA.png)

Now let s brute force for Directories .

We re gonna be using FFUF for this : This is the command for FFUF :

```
ffuf -u http://192.168.87.128/mutillidae/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*Wjp6OsjA4Qpyz1iIkydmYg.png)

As you can see we have multiple directories to look into , first we can visit the robots.txt or the phpinfo .

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*mTohgYqboSB3GvxamxbEOA.png)

Directories like PHPinfo shouldnt be available to the public since an attacker can see the version of our php and find if it s vulnerable and exploit it later on .

The “robots.txt” file is a text file located at the root directory of a website. It provides instructions to web crawlers, such as those used by search engines, regarding which pages or directories should not be indexed or accessed. In other terms this file contains directives specifying which parts of the site should be excluded from crawling or indexing by search engines. This also shouldn’t be out for everyone to see , or if you want to have it as a directory , it should at least be Password Protected .

Now let s visit these 2 :

![](https://miro.medium.com/v2/resize:fit:590/1*an40joNUtmwmXSd5VXxevQ.png)

We can visit the passwords directory and see if we can find anything interesting :

![](https://miro.medium.com/v2/resize:fit:672/1*9ATxmwB7N2_4-4DY5jTEyA.png)

We ll just save these names for later , it might be usefull later on . Now let s visit the php info page .

As you can see this passwords directory also shouldn’t be accessible or at least should require a password for you to be able to see it .

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*Zlw7FxyiFie3YNyQsDUx5Q.png)

Lucky for us this php version is vulnerable to “ cgi Argument Injection” , so we will be exploiting this later on .

### **//Port 80 : http/apache :**

> *HTTP (Hypertext Transfer Protocol) is an application protocol for distributed, collaborative, and hypermedia information systems. It is the foundation of data communication for the World Wide Web, used by web browsers to access web pages.*
> 

As we saw before the PHP version is vulnerable to a cgi argument injection . Since this exploit can be found on Rapid7 website that means we can use Metasploit for this attack .

Just go to Metasploit once again and search for the “php_cgi_arg_injection ” exploit :

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*ot6wF_LTmThVsR_Pgt4C-Q.png)

Now all we need to do is set the options that we need to set :

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*8CsuvGlPZfxPivma8tdiuQ.png)

In this case we only need to set the RHOST for this exploit to run :

![](https://miro.medium.com/v2/resize:fit:664/1*sYkVOgMxPSrolhEuv5Ipuw.png)

Just Like that we got access to that machine just because we found out that the version of the PHP was vulnerable .

### **//Port 445 : SAMBA :**

> *SMB (Server Message Block) is a network file sharing protocol used by Windows-based computers for sharing files, printers, serial ports, and miscellaneous communications between nodes on a network.*
> 

Here on port 445 we find that the version for this smb is vuulnerable to a command execution attack , the name of the exploit is “usermap_script”

Now let s try to use Metasploit to exploit this version of smbd :

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*nQi-0P1yLe9OTfvV4rq0rA.png)

Just Set the RHOST and we re good to go :

![](https://miro.medium.com/v2/resize:fit:673/1*av2tYxx3PhL2ARVg9CWjOQ.png)

![](https://miro.medium.com/v2/resize:fit:513/1*nI5P_gjpWmJnL8XiHOPx2g.png)

From here we can import our shell to make it easier to navigate the machine , Here is the command to do that :

```
python -c 'import pty; pty.spawn("/bin/bash")'
```

### **// Port 5432 : postgresql :**

> *PostgreSQL is a powerful open-source database management system. It’s commonly used for storing and managing data in various applications, from small projects to large-scale enterprise systems. PostgreSQL is known for its reliability, extensive features, and compliance with SQL standards, making it a popular choice among developers and organizations worldwide.*
> 

Lucky For us this version of Postgresql is vulnerable to Payload Execution attack . (We can simply search for that on Google )

Now let s go to Metasploit and try to exploit this version : We will be using this Metasploit module : `exploit/linux/postgres/postgres_payload`

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*D8bj07hq_ucmATTIN3VYzg.png)

We will just need to set the RHOST and LHOST for this one and run it :

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:668/1*1DBuWQ7GP6lglUjw3mEu5g.png)

Nice just like that we got our meterpreter shell .

### **// Port 6667 : IRC :**

> *Port 6667 is commonly associated with IRC (Internet Relay Chat), which is a protocol used for real-time text messaging and communication over the Internet. IRC allows users to join chat rooms or channels and communicate with other users in real-time. It has been a popular means of online communication for decades, often used for group discussions, online communities, and support forums. However, with the rise of other messaging platforms, its usage has declined in recent years.*
> 

We see that the version for this IRC is UnrealRCD , now let s check if this version has any known exploits . (You can use Google or just Searchsploit on kali or any other search engine out there )

Lucky for us this version is vulnerable to a backdoor Code Execution Attack .

Since it s on Rapid7 that means we can use Metasploit for it , just search for this module :

< exploit/unix/irc/unreal_ircd_3281_backdoor >

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*_FFZpU5L-g2R-zu02BYxrw.png)

Even though it says that it only needs the RHOSTS to work, you will still need to set a payload and LHOST, which is your machine’s IP address. For the payload, we will use < /cmd/linux/reverse>. Since we’re dealing with a Linux machine and we want a reverse connection rather than a bind one, this avoids being blocked by the Firewalls.

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*dNaLvWnd589zUdMU1U4mfA.png)

Now we just let it Run and we will eventually get our Shell .

![](https://miro.medium.com/v2/resize:fit:659/1*h7UlMUI9FmUmCtVIptlSfg.png)

![](https://miro.medium.com/v2/resize:fit:652/1*4NEWyZPUse9BwvPY4MZaEw.png)

Perfect , Just like that we re in .

### **// Port 8180 : Apache Tomcat :**

> *Port 8180 is commonly associated with Apache Tomcat, which is an open-source web server and servlet container. Apache Tomcat is widely used for deploying Java-based web applications. Port 8180 is often used as an alternative HTTP port for Apache Tomcat deployments, alongside the default port 8080. It allows developers to host and access web applications through Tomcat on a different port, while still leveraging Tomcat’s capabilities for servlet and JSP (JavaServer Pages) execution.*
> 

Again let s search if this version of apache has any known exploit . Lucky for us TOMCAT has a known vulnerabilty which is “ Tomcat Manager Authenticated Upload Code Execution” .

What this does is that it allows us to execute a payload on Apache Tomcat servers that have an exposed “manager” application.

Since it can be found on Rapid7 , that means we will be using Metasploit for it .

Load Metasploit and search for this module :

<exploit/multi/http/tomcat_mgr_upload>

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*VCa7_yfI5KNSrIvxUju2Xw.png)

Now let s see the options .

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*iSZ68A6jKyN7L7_mIp3-2A.png)

In this case even if it says that we dont need to give a httpUsername and Password , we still have to do that , if not the exploit wont work .

For the HttpUsername and Password , just search for apache ‘ s default credentials on GitHub .

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*G5jzXD7ps5oUsYccg3tUDQ.png)

Let s just Try tomcat tomcat since it s the most commonly used .

![](https://miro.medium.com/v2/resize:fit:657/1*im04HrkJ5AgMpbZpRtGonw.png)

Now let s just Run it .

Press enter or click to view image in full size

![](https://miro.medium.com/v2/resize:fit:700/1*qtGKLhJvK2cTasxmVPwz-Q.png)

Just like that we can run command on the machine .

Thank you for following along with this Metasploitable walkthrough. If you have any questions or notice any mistakes, please don’t hesitate to reach out. Thank you for your attention and participation !