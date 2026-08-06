---
title: " Hack The Box OnlyHacks Walkthrough"
date: 2026-08-06 00:00:00 +0000
categories: [CWES Preparation Path ]
tags: [CWES Preparation, HackTheBox, Challenge , Web_Attacks ]
---



## Challenge Scenario :

Dating and matching can be exciting especially during Valentine's, but it’s important to stay vigilant for impostors. Can you help identify possible frauds?

## Solution : 

This challenge was pretty easy , nothing complicated overall . 

First we visit the given IP address : 

<img width="1266" height="805" alt="image" src="https://github.com/user-attachments/assets/bac6835e-9808-4eb1-9906-ef49ae0a6ab7" />

First thing i tried is basic auth bypass via SQLi but that didn't work , so i moved to FUZZING the application for directories : 

```bash
feroxbuster -u http://154.57.164.82:32598/
```

<img width="1459" height="868" alt="image" src="https://github.com/user-attachments/assets/df7d5ed5-904e-46f1-9ef8-d7f68ad480d0" />

Both chat and matches return 500 and 401 , the rest redirects us back to the login page , let's just create an account and browse the application further : 

<img width="986" height="725" alt="image" src="https://github.com/user-attachments/assets/51307011-d56b-4790-a46f-56bc98c2d1a4" />

When creating the account , we see that we can't modify any role or get admin rights by intercepting and modifying the request : 

<img width="1494" height="669" alt="image" src="https://github.com/user-attachments/assets/964b8caf-f537-4d63-bb04-6d8452844fe7" />

Once inside : 

<img width="1910" height="839" alt="image" src="https://github.com/user-attachments/assets/8e82759a-bb2e-4d8a-8875-7ce99156ed3d" />

First let's use every feature on the app then check Burp history to see which requests are made to the server : 

<img width="1108" height="156" alt="image" src="https://github.com/user-attachments/assets/5161e3b3-4cf4-426b-8332-d857396ba1f1" />

Nothing interesting for now , let's try to get a match and send a normal message to see if we get a different request : 

<img width="1400" height="700" alt="image" src="https://github.com/user-attachments/assets/d2b4c4ca-fd76-4796-8201-a987cb5bfcff" />

Now if we check Burp : 

<img width="1716" height="666" alt="image" src="https://github.com/user-attachments/assets/ddbe7651-2bf2-4654-ba1c-85111569863f" />

We see that we send a POST request with our ID parameter when we write a normal message : 

<img width="1602" height="549" alt="image" src="https://github.com/user-attachments/assets/1ac76d31-5f73-41c2-affc-e35829559713" />

Then we get another GET request , which shows us the conversation , and it contains the same RID , let's try to modify the rid and hope we get the conversation of a different user instead : 

<img width="1595" height="480" alt="image" src="https://github.com/user-attachments/assets/3a6dacdf-e7b6-4751-a211-5a2336ff2040" />

For this i used FFUF , since i don't have Burp pro , intruder is very slow , just make sure you add the necessary headers , specially the token we got : 

```bash
# Generate a List of numbers to Fuzz with
seq 1 1000 > numers

# Fuzzing using Ffuf :
ffuf -u http://154.57.164.71:31386/chat/?rid=FUZZ -w numbers -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/144.0.0.0 Safari/537.36" -H "Cookie: session=eyJ1c2VyIjp7ImlkIjo1LCJ1c2VybmFtZSI6InRlc3QxMjM0NTY3ODkifX0.anTDeQ.fXgj63H-stLAy_T3_r1cETuBEps" -H "Referer: http://154.57.164.71:31386/chat/?rid=6" -H "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7" -fs 265
```

I filtered by Size 265 since that's the size of the error response : 

<img width="1882" height="827" alt="image" src="https://github.com/user-attachments/assets/62494695-c84f-4864-bace-62087928f366" />

We see that we get another number which is 3 , now back to burp : 

<img width="1646" height="713" alt="image" src="https://github.com/user-attachments/assets/9dbde0bb-2d32-4e24-ab2d-397b5cb22179" />

We see that we are able to access another user's conversation and we get the flag that way . 

This site was also vulnerable to an XSS vulnerability , we can easily test it using a basic XSS payload : 

```js
<script>alert(window.origin)</script>
```

<img width="1484" height="780" alt="image" src="https://github.com/user-attachments/assets/55c51e59-e36f-4f72-970f-800a6ba1ce71" />

But that's not the point of this challenge . 

That was all for this one, see you in the next one :) 



