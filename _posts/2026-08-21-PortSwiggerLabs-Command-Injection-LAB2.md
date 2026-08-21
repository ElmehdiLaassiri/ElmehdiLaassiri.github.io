---
title: " PortSwiggerlabs: OS command injection, simple case "
date: 2026-08-21 00:00:00 +0000
categories: [PortSwiggerLabs , Command Injection ]
tags: [PortSwiggerLabs ,Command Injection , Challenge , Web_Attacks ]
---



## Scenario : 

This lab contains a blind OS command injection vulnerability in the feedback function.

The application executes a shell command containing the user-supplied details. The output from the command is not returned in the response.

To solve the lab, exploit the blind OS command injection vulnerability to cause a 10 second delay.


## Solution :

<img width="1645" height="898" alt="image" src="https://github.com/user-attachments/assets/cbef9c70-33b4-42d8-9b2d-cec8c11e01a1" />

First we start by navigating the app just like a normal user, from there we check Burp's History . 

We're mainly looking for parameters where we can inject , whether in a POST or GET request (Mainly POST since that's what we're sending to the server ) . 

Usually whenever we get a parameter it's always good to test all Injections (SQLi,XSS,...) but in our case we will focus on Command Injection Only . 

Since this is a Command injection , what we're hoping for is that the parameter being passed to the server gets placed inside a system command (or passed to a function that executes shell commands), so the server ends up running our input as part of a command.

So to abuse this we can add a command injection payload to make it so that it executes an additional command . 

there are plenty of command injection payloads that we can use . 

<img width="1088" height="566" alt="image" src="https://github.com/user-attachments/assets/c4cfad0e-bac3-462e-8c67-6865a0337d9b" />

We will use something that works for both Linux and Windows : 

```bash
||
```

We find this endpoint , where we can submit our feed back . 

<img width="1735" height="856" alt="image" src="https://github.com/user-attachments/assets/bcbc5459-70f9-4a03-83e6-ec12b57081cc" />

This is the POST request that we're looking for . 

<img width="1492" height="707" alt="image" src="https://github.com/user-attachments/assets/8d647c36-6623-4466-a1b1-fc7e859f147a" />

The problem is , we don't get anything back . So even if we have a command injection , we can't see the output , so how would we know ? 

We can inject commands that will cause time delay . 

```bash
sleep 10
ping -c 10 127.0.0.1
```

Both of these will take 10 sec to finish which will result in a delay before we get the response . 

First let's inject the name parameter then move to the email ....

<img width="1413" height="683" alt="image" src="https://github.com/user-attachments/assets/ef40a208-c291-48d0-bad9-8de25b0e6f4e" />

When injecting the name field , we don't get any delays which means our command wasn't executed . 

Now moving on to the email field : 

```bash
email=test@gmail.com||sleep+10||& .....
```

Make sure the space is URL encoded .

<img width="1829" height="872" alt="image" src="https://github.com/user-attachments/assets/302b8396-4fc2-4e66-80b1-5ea48a577f01" />

We get a delay this time of exactly 10 sec . 

<img width="1920" height="830" alt="image" src="https://github.com/user-attachments/assets/f1af0c28-a384-49c2-a82e-f631467bb4aa" />

This confirms the Command Injection . 

Now if we go back to the lab it should be solved since we triggered the 10 seconds delay . 

<img width="1603" height="764" alt="image" src="https://github.com/user-attachments/assets/225c0fa6-276f-4b9b-9404-c4691e5d8839" />

That was all for this lab, see you in the next one :) 
