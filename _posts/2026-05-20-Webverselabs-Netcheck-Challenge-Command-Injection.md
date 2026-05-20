---
title: "Webverselabs Netcheck Challenge Command Injection  "
date: 2026-05-20 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---


## Scenario : 

Netcheck is a bootstrapped uptime-monitoring SaaS founded in 2021 by two ex-SRE friends in Lisbon, with about 800 paying teams on plans that start at $19/month. The Manual Diagnostics panel was a sales-led feature, built in an afternoon to close a deal with a customer who wanted "proof from outside our network," and it has been quietly earning revenue ever since. The annual customer audit lands in two weeks and the founders asked you to take a look first.


## Solution : 


<img width="1264" height="778" alt="image" src="https://github.com/user-attachments/assets/1fce47b4-90f4-4841-863a-ab307871729c" />

We first create an account then browse the application just like a normal user to see different endpoints , functionalities and finally check Burp's History to see the differernt requests made to the server . 

<img width="1751" height="824" alt="image" src="https://github.com/user-attachments/assets/34d9012e-d94a-40ad-b9df-8a384ad229c0" />

Upon navigating the application , we find this Diagnostics endpoint , which Pings whatever address we give it to check if it's alive . This is porbably just running a ping command underneath , which can be abused for a command injection if the parameter being passed to the server isn't well sanitized 

<img width="1597" height="711" alt="image" src="https://github.com/user-attachments/assets/6f5d9de9-170d-4de0-b7ca-c51c5aaa1ee8" />

I first tried some basic commmand injection payloads to see which one will break the application , but this one worked pretty well .

<img width="1534" height="756" alt="image" src="https://github.com/user-attachments/assets/c7e397cc-dc54-44fc-a77a-17c11ec63c00" />

Injecting in the action field didn't work so i decided to inject the Host Parameter instead and it worked , we got our command injection , Now to read the flag we just need to bypass the space filter if it exists , for this we can simply use : 

```bash
${IFS} ==> This is our space

action=diag&host=8.8.8.8;cat${IFS}/flag.txt

```

<img width="1559" height="777" alt="image" src="https://github.com/user-attachments/assets/ca3ccaff-f506-4421-9a87-1a8dda9349f4" />

Just like that we get our flag . That was it for this challenge , see you in the next one :)


