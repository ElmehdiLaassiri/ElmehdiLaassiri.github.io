---
title: " PortSwiggerlabs: Blind OS command injection with output redirection "
date: 2026-08-21 13:00:00 +0000
categories: [PortSwiggerLabs , Command Injection ]
tags: [PortSwiggerLabs ,Command Injection , Challenge , Web_Attacks ]
---



## Scenario : 


This lab contains a blind OS command injection vulnerability in the feedback function.

The application executes a shell command containing the user-supplied details. The output from the command is not returned in the response. However, you can use output redirection to capture the output from the command. There is a writable folder at:

/var/www/images/
The application serves the images for the product catalog from this location. You can redirect the output from the injected command to a file in this folder, and then use the image loading URL to retrieve the contents of the file.

To solve the lab, execute the whoami command and retrieve the output.



## Solution : 

<img width="1705" height="862" alt="image" src="https://github.com/user-attachments/assets/ec380658-1745-4174-9bd4-2c6a368af698" />

First we browse the app just like a normal user , then we check Burp's History for interesting Requests . 

We're mainly looking for parameters where we can inject , whether in a POST or GET request (Mainly POST since that's what we're sending to the server ) . 

Since this is a Command injection , what we're hoping for is that the parameter being passed to the server gets placed inside a system command (or passed to a function that executes shell commands), so the server ends up running our input as part of a command.

So to abuse this we can add a command injection payload to make it so that it executes an additional command . 

there are plenty of command injection payloads that we can use . 

<img width="1088" height="566" alt="image" src="https://github.com/user-attachments/assets/c4cfad0e-bac3-462e-8c67-6865a0337d9b" />

We will use something that works for both Linux and Windows : 

```bash
||
```

Just like the lab before, we find this endpoint , where we can submit our feed back . 

<img width="1401" height="870" alt="image" src="https://github.com/user-attachments/assets/d5098264-08b0-4eab-8303-287561328c08" />

Checking Burp, we see that this is indeed a POST Request with multiple parameters that we can inject . 

<img width="1683" height="790" alt="image" src="https://github.com/user-attachments/assets/9b433534-a989-43a8-8464-d665b266e27d" />

I first injected the name but didn't work . 

<img width="1532" height="665" alt="image" src="https://github.com/user-attachments/assets/64a42038-ab47-4ede-a39d-5261b0d02526" />

No way for us to know , we'll use the time delay technique to find the vulnerable parameter first . 

```bash
||sleep+10||
```

<img width="1419" height="633" alt="image" src="https://github.com/user-attachments/assets/08cab071-0889-4e1b-b1d0-4b7223ca6d3a" />

Didn't get any delay which means it wasn't executed , let's move to the email field . 

<img width="1892" height="880" alt="image" src="https://github.com/user-attachments/assets/64cd14ec-aa96-4f00-bae0-ae88efbcd2ef" />

And we got our delay which confirms the email is indeed the vulnerable field . 

Now to get the output of this command we're given a writable file that we can use .

```text
/var/www/images/
```

All we have to do is pipe the output of the command into that file , and from there read the file content by modifying the filename parameter . 

```bash
||whoami>/var/www/images/output.txt||
```

<img width="1549" height="723" alt="image" src="https://github.com/user-attachments/assets/fe1b54e9-65a4-41c5-9a9d-5c9a1cd32900" />

Now let's try reading the content of the output.txt file we just created . 

<img width="1448" height="655" alt="image" src="https://github.com/user-attachments/assets/5b2d3537-8d63-4324-8b9a-6885c1816275" />
 
We just modify the filename to the specific file we wrote into .

<img width="1522" height="601" alt="image" src="https://github.com/user-attachments/assets/8e0a37fb-efd2-4626-bb2c-a41a4a215edd" />

This didn't work , but remembering the labs from Path traversal , there was a case where we had to specify the file name only, since it will append /var/www/images/ by default. 

<img width="1521" height="676" alt="image" src="https://github.com/user-attachments/assets/ada9cfa6-8b01-4212-be60-b37ca2517697" />

Perfect we get the output of the whoami command . 

This should solve the lab . 

That was all for this lab , see you in the next one :) 


<img width="1763" height="651" alt="image" src="https://github.com/user-attachments/assets/86cb186f-f66c-4548-b8f7-5795c2fc9291" />

That was all for this lab, see you in the next one . 
