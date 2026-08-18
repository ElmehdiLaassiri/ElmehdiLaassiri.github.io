---
title: " PortSwiggerlabs: File path traversal, traversal sequences stripped non-recursively "
date: 2026-08-18 00:00:00 +0000
categories: [PortSwiggerLabs , Path Traversal ]
tags: [PortSwiggerLabs , Path Traversal , Challenge , Web_Attacks ]
---


## Scenario : 

This lab contains a path traversal vulnerability in the display of product images.

The application strips path traversal sequences from the user-supplied filename before using it.

To solve the lab, retrieve the contents of the /etc/passwd file.


## Solution : 

<img width="1782" height="895" alt="image" src="https://github.com/user-attachments/assets/4ddade78-dd8b-4711-a12a-5037dccfb485" />

Just like every time , we first start by navigating the application just like a normal user , from there we check Burp History :

<img width="1536" height="658" alt="image" src="https://github.com/user-attachments/assets/fdcb4722-3efe-4fee-bc45-0aa01e73ff27" />

We find 2 interesting parameters , Product ID and Filename . 

To save time , the ID parameter isn't vulnerable , i tried multiple payloads , multiple obfuscation methods and Filter Bypasses but they didn't work . 

Now let's test the Filename parameter using the payloads from payloadallthethings : 

```text
https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/File%20Inclusion/README.md
```

<img width="1465" height="568" alt="image" src="https://github.com/user-attachments/assets/18ca9912-6db5-4b7a-8004-822ba70c5d56" />

Injecting a simple Payload just returns no such file . 

<img width="1465" height="568" alt="image" src="https://github.com/user-attachments/assets/aa71ee63-a5c8-42a5-b50b-12b38f82de21" />

I decided to specify the filename instead without any backslashes . 

<img width="1320" height="580" alt="image" src="https://github.com/user-attachments/assets/50a7cbeb-4720-424d-ac05-3771304eeab3" />

Didn't work either , maybe the ../ gets flagged by the Firewall or it gets truncated automatically . 

Let's try other Payloads to Bypass the WAF , here are some examples : 

```bash
%252e%252e%252fetc%252fpasswd
%252e%252e%252fetc%252fpasswd%00
%c0%ae%c0%ae/%c0%ae%c0%ae/%c0%ae%c0%ae/etc/passwd
%c0%ae%c0%ae/%c0%ae%c0%ae/%c0%ae%c0%ae/etc/passwd%00
....//....//....//etc/passwd
..///////..////..//////etc/passwd
/%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../etc/passwd
```

<img width="1484" height="738" alt="image" src="https://github.com/user-attachments/assets/344df3cc-8368-4134-a593-a85a5675defc" />

At the end this one worked : 

```bash
....//....//....//etc/passwd
```

You can of course automate this by using Intruder or FFUF , but since there weren't too many , it was easier to test them manually .

Now if we go back to our Lab , it should say Solved . 

<img width="1750" height="822" alt="image" src="https://github.com/user-attachments/assets/1dd9d599-1a7f-4ecd-8677-78fa73c9f865" />

That was all for this lab , see you in the next one :) 

