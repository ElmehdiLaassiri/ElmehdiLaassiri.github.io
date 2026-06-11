---
title: "Webverselabs ScanPortal Challenge Command Injection  "
date: 2026-06-11 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


## Scenario : 

ScanPortal is an attack surface management platform for SMBs. The developer recently hardened the nmap call after a code review flagged it. What they missed is that every submitted target is also appended verbatim to a shared scan log, and the customer-facing log search runs grep on that file with the user's pattern dropped straight into a shell command.

## Solution : 

<img width="1859" height="887" alt="image" src="https://github.com/user-attachments/assets/67f00f55-1bd1-4be2-b701-42ca2e1596d0" />

We first start by creating a new account and login in , to be able to enumerate the application further . 

<img width="1901" height="873" alt="image" src="https://github.com/user-attachments/assets/5dfa052a-b03d-4f1d-b397-fd38f22a400a" />

We are able to identify multiple endpoints, let's go through each one of them : 

**/Assets :**

<img width="1894" height="826" alt="image" src="https://github.com/user-attachments/assets/22575abb-4d18-4297-846e-0af0b41df09c" />

This just gives us a list of IP addresses , hostnames , and OS for each server running. 

**/Vulns** : 

<img width="1890" height="804" alt="image" src="https://github.com/user-attachments/assets/bd382b6a-5f4c-4b99-9aaf-5dbb9d7f4a96" />

This is a list of vulnerabilities found ranked by criticality . 

**/Scans :**

<img width="1907" height="564" alt="image" src="https://github.com/user-attachments/assets/1b2fa0d7-4336-40b8-9d3b-730775bc5244" />

**/Scan Logs :**

<img width="1910" height="544" alt="image" src="https://github.com/user-attachments/assets/7ddbbf2b-efe2-4d4c-8102-9569ab7b4ebd" />

This is the log file of all scans performed , for now we haven't done any . 

The only endpoint that looks interesting is the /scans endpoint , if we perform a scan and look at the request :

<img width="1138" height="594" alt="image" src="https://github.com/user-attachments/assets/1b103c67-ecf3-4e66-bdd6-de9aa81ed594" />

We see that it sends a POST request to the server , with the IP we specified as well as scan_type ,then it redirects us to the scan result page : the server is probably doing an nmap scan in the background and returning to us the result in a table format . 

<img width="1610" height="618" alt="image" src="https://github.com/user-attachments/assets/55bc416c-5824-4d03-ad51-9c2b93fb3b07" />

Whenever we suspect the backend may execute commands based on user input, we should test for command injection. If inputs aren’t properly sanitized, we may be able to execute arbitrary commands on the server.

I have a detailed section in my web_methdology for command injections based on the CWES path : 

```bash
https://elmehdilaassiri.github.io/posts/web-app-pentesting-methodology/#command-injection-
```

Now first i injected a semi column in both parameters to see if it will break anything . 

<img width="1526" height="684" alt="image" src="https://github.com/user-attachments/assets/4616061c-8800-4add-be6d-ca43dd32619d" />

We see that if we inject the IP address field , it will just be returned to us which means it wasn't executed by the server. 

Checking the scan_type parameter returned the same thing, i tried multiple payloads : 

<img width="1817" height="654" alt="image" src="https://github.com/user-attachments/assets/1da5c795-d738-4a65-81a3-5f70bd2137bb" />

At first i thought maybe we can break our of a "" or '' , maybe injecting ${} will execute commands inside the quotes rather than returning it via an echo command or something but that didn't work . 

Finally i moved back to the Logs : 

<img width="1898" height="507" alt="image" src="https://github.com/user-attachments/assets/87ef0637-1c56-4269-878e-05c970e653ae" />

We see that we are able to search for specific logs, maybe the server is using a command in the backend this time as well to fetch for specific logs based on keywords, if we check Burp's History : 

<img width="1584" height="596" alt="image" src="https://github.com/user-attachments/assets/e1c0ae70-2a35-4bf7-ba95-3b0771433842" />

Let's try to use the same injection payloads from earlier on this parameter : 

<img width="1612" height="654" alt="image" src="https://github.com/user-attachments/assets/614dcc3e-f5bf-46c9-8be7-a2ba5d245293" />

Good news we manage to get a different error this time :) , the server is telling us that we got an unterminated quote . 

Probably we have a grep command running in the background or some other linux command :

```bash
grep "aaaa"";${id}
```

I think having 3 quotes broke it, let' try adding a fourth one : 

<img width="1336" height="526" alt="image" src="https://github.com/user-attachments/assets/e7981721-267b-4388-bf7b-32e1f3ab0905" />

We no longer have the quotation problem , but we get a permission denied . 

<img width="1906" height="537" alt="image" src="https://github.com/user-attachments/assets/bf6b7f02-28c8-485a-acaa-7b51a89d4fe0" />

I first thought that we can only add different files to the command to trick the app into rendering the file but that didn't work either : 

<img width="1817" height="364" alt="image" src="https://github.com/user-attachments/assets/29908fac-e6b4-44d2-aadc-cf0251e84a2c" />

Another thing we can try is once we break out of the "" , and we add our command , we comment out the rest just like we do in a normal SQLi , to make sure we don't break the original command syntax. 

<img width="1366" height="525" alt="image" src="https://github.com/user-attachments/assets/8cacf806-c231-4d55-9e25-c0bab7361f93" />

This works perfectly : 

Now we can easily get our flag : 

<img width="1315" height="473" alt="image" src="https://github.com/user-attachments/assets/9dab197f-816f-402c-a353-75bfb66b0153" />

Let's also try to read the source code to see what was really going on in the backend . 

<img width="934" height="461" alt="image" src="https://github.com/user-attachments/assets/c1aaed37-7b18-4517-90ab-7dcd7247768d" />

The app source code is in app.py : 

<img width="1155" height="687" alt="image" src="https://github.com/user-attachments/assets/4453c30f-38b7-47f5-a3d3-c4d60cbbe8dd" />

Perfect , we see that it was indeed using a grep command and putting everything inside a double quotes , so when we added " we broke out of it and were able to execute our code . 

That was all for this challenge , see you in the next one :) 
