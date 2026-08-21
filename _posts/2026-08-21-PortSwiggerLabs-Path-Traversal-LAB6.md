---
title: " PortSwiggerlabs: File path traversal, validation of file extension with null byte bypass "
date: 2026-08-21 00:00:00 +0000
categories: [PortSwiggerLabs , Path Traversal ]
tags: [PortSwiggerLabs , Path Traversal , Challenge , Web_Attacks ]
---



## Scenario : 

This lab contains a path traversal vulnerability in the display of product images.

The application validates that the supplied filename ends with the expected file extension.

To solve the lab, retrieve the contents of the /etc/passwd file.


## Solution : 

<img width="1846" height="936" alt="image" src="https://github.com/user-attachments/assets/c75102b4-ca49-4869-83da-c52d7042a526" />

First we navigate the app like a normal user then we check Burp's History : 

<img width="1493" height="664" alt="image" src="https://github.com/user-attachments/assets/e24fcd34-54b6-4e96-bba1-ff42d8b9d4e1" />

Again, we're interested in these 2 parameters that we can inject with our LFI payloads . 

I will assume the Product ID isn't vulnerable just like all the previous labs , just to save time , and start testing the filename parameter instead :  

First i tried all payloads from earlier labs : 

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

<img width="1510" height="571" alt="image" src="https://github.com/user-attachments/assets/ac7590f9-5c9d-414d-95ce-014b3cb7a4e3" />

We only get a 400 . 

It probably only accepts .png format . 

<img width="1101" height="362" alt="image" src="https://github.com/user-attachments/assets/3d5454f7-3026-4cf7-a7b8-9befed4b403e" />

I found this technique on Payload All the Things . 

This terminates the request so that even if the server adds a certain extension, for example .php, it won't try and read /etc/passwd.php but instead it will read /etc/passwd and ignore the rest, so that even if the server appends something it doesn't block us.

This also works when the server expects .png in the filename. If we just ask for /etc/passwd without .png, the server rejects it because it doesn't pass the extension check. But we can't ask for /etc/passwd.png either, since that file doesn't exist. This is where the null byte saves us: we send /etc/passwd%00.png, so the .png passes the server's check, while the %00 truncates the string at the filesystem level, and the server ends up reading the real /etc/passwd.

With that being said , let's test it : 

```bash
http://example.com/index.php?page=../../../etc/passwd%00
```

<img width="1464" height="721" alt="image" src="https://github.com/user-attachments/assets/48ddd3a0-0768-410f-a9e0-11625641c88e" />

We're still getting the error , since we're not passing the extension check probably . 

Now if we add the .png extension : 

```bash
../../../etc/passwd%00.png
```

<img width="1496" height="754" alt="image" src="https://github.com/user-attachments/assets/1bb8c991-bca5-4174-a74d-15717add57b4" />

This works perfectly :) 

That was all for this lab , see you in the next one :) 

