---
title: "Webverselabs Parasite XXE  "
date: 2026-06-09 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


## Scenario :

Parasite Systems built a centralized dashboard to manage server configurations across their fleet. The import tool accepts configuration files — but how thoroughly did they lock down what it can access?

## Solution : 

<img width="1653" height="895" alt="image" src="https://github.com/user-attachments/assets/b83abe59-306b-4f2b-a3cb-1f7a287434a5" />

If we visit the app, we find a dashboard that keeps track of memory usage, CPU,..., as well as some additional information about the server used , db and cache , but apart from that , we don't get anything useful. 

As usual we will browse the application just like a normal user would , then check Burp's History , we're looking for parameters in a GET request , POST requests that we can modify, excessive data exposure . We find that there are 4 endpoints we can visit. 

**/Services :** 

<img width="1691" height="855" alt="image" src="https://github.com/user-attachments/assets/0a9af870-cc1e-4c3a-9865-869d3443f3f2" />

This lists all the services as well as their open ports , but nothing useful for us really. 

**/Logs :**

<img width="1743" height="844" alt="image" src="https://github.com/user-attachments/assets/5d4d4dce-9f41-4f29-8322-14ec8e827db0" />

This is a history of Logs. No parameters, No POST request we can modify for now . 

**/Settings :**

<img width="1725" height="879" alt="image" src="https://github.com/user-attachments/assets/462560d4-0959-4d4a-b035-716db6eb3d0c" />

Shows additional information about the server, like the hostname, environment, etc, but overall nothing useful . 

**/Import-Config :**

<img width="1649" height="897" alt="image" src="https://github.com/user-attachments/assets/bd02de11-a0be-476b-af32-c9fb415f251b" />

Finally we have the import-config endpoint which allows us to upload config files in an XML Format, this is a huge indicator that there might be an XXE vulnerability we can abuse. 

I have a detailed section on my web methodology on How to detect and exploit XXE if you wanna check it out : 

```xml
https://elmehdilaassiri.github.io/posts/web-app-pentesting-methodology/#xxe--detection-
```
Now first thing i did was upload a normal XML file , just to understand what happens when we upload a config file to the server, whether there are some informations rendered to us, whether we get nothing in return. 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE Confighh [
  <!ENTITY provider "Example E-Sign Corp">
]>
<envelope>
  <Signer>Patricia Vance</Signer>
  <DocumentTitle>Q1 Mutual NDA</DocumentTitle>
  <Timestamp>2026-05-13T09:14:22Z</Timestamp>
  <Organization>Vance &amp; Holloway LLP</Organization>
  <SourceProvider>&provider;</SourceProvider>
</envelope>
```

I first uploaded this legitimate xml file that i had from another challenge to see what would happen . 

<img width="1659" height="641" alt="image" src="https://github.com/user-attachments/assets/9ec70777-3321-4dc1-abae-0282280d1829" />

We see that we get the entire content back, also the entity we added from earlier "provider" in this example was actually referenced and rendered back to us in the response . 

The fact that it is reflected to us is a good sign , now let's try to read External files , in this case the /etc/passwd , for this one we can use SYSTEM which is the XML keyword that tells the parser to fetch an external resource ,the file:// is the URI scheme that tells it where to fetch from. 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE envelope [
  <!ENTITY xxe SYSTEM file:///etc/passwd >
]>
<envelope>
  <Signer>&xxe;</Signer>
  <DocumentTitle>Q1 Mutual NDA</DocumentTitle>
  <Timestamp>2026-05-13T09:14:22Z</Timestamp>
  <Organization>Vance &amp; Holloway LLP</Organization>
  <SourceProvider>&provider;</SourceProvider>
</envelope>
```
The key in an XXE is to always reference our entity in the field that gets rendered back to us , in this case the entire content is rendered back to us so we can reference it in any of the fields we want. 

<img width="1627" height="889" alt="image" src="https://github.com/user-attachments/assets/764bc0ef-271c-4e54-ac29-5e875e276d7c" />

We see that we get the content of /etc/passwd rendered back to us which means the XXE worked perfectly . 

From here we can just read the flag , it is always located at /flag.txt . 

<img width="1551" height="607" alt="image" src="https://github.com/user-attachments/assets/9fe25bdd-1141-4e85-8dfb-48fc64867170" />

That was all for this challenge, see you in the next one :) 

