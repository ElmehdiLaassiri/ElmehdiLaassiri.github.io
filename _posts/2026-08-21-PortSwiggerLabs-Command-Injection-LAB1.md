---
title: " PortSwiggerlabs: OS command injection, simple case "
date: 2026-08-21 00:00:00 +0000
categories: [PortSwiggerLabs , Command Injection ]
tags: [PortSwiggerLabs ,Command Injection , Challenge , Web_Attacks ]
---



## Scenario : 

This lab contains an OS command injection vulnerability in the product stock checker.

The application executes a shell command containing user-supplied product and store IDs, and returns the raw output from the command in its response.

To solve the lab, execute the whoami command to determine the name of the current user.


## Solution : 

As the name implies , "simple case" , this will be a simple lab , no WAF , no filters nothing that will block us :) 

<img width="1645" height="894" alt="image" src="https://github.com/user-attachments/assets/6d2310f2-116c-4bbf-8b82-a6b2175b76a0" />

Now first , we navigate the app , then check Burp's History as always . 

We're mainly looking for a parameter that we can inject : 

<img width="1615" height="778" alt="image" src="https://github.com/user-attachments/assets/469b61ef-a264-459a-b78d-71e8376bea4c" />

We get the Product ID parameter that changes depending on each product we select . 

Usually whenever we get a parameter it's always good to test all Injections (SQLi,XSS,...) but in our case we will focus on Command Injection Only . 

What we're hoping for is that the parameter being passed to the server gets placed inside a system command (or passed to a function that executes shell commands), so the server ends up running our input as part of a command.

So to abuse this we can add a command injection payload to make it so that it executes an additional command . 

there are plenty of command injection payloads that we can use . 

<img width="1088" height="566" alt="image" src="https://github.com/user-attachments/assets/c4cfad0e-bac3-462e-8c67-6865a0337d9b" />

We will use something that works for both Linux and Windows : 

```bash
||
```

<img width="1379" height="612" alt="image" src="https://github.com/user-attachments/assets/7d552885-e030-462f-a673-bfe318f1ece9" />

Product ID was not vulnerable -_- . 

Now if we check the Stock on different stores : 

<img width="1332" height="503" alt="image" src="https://github.com/user-attachments/assets/7bf94217-1052-4a3b-a883-e1fd97dac4c2" />

We get a POST request , containing 2 parameters : 

<img width="1492" height="674" alt="image" src="https://github.com/user-attachments/assets/79a85fd3-59cb-427e-a5d6-cbc38be27834" />

I will start with the Store ID parameter since the Product ID didn't seem to be vulnerable . 

First a simple payload to see if it will break anything :

<img width="1280" height="682" alt="image" src="https://github.com/user-attachments/assets/ad3d51ac-5176-4de8-a567-2e8cc106ca73" />

```text
%7c ==> Encoded value for |
```

Now let's add our command and check if this works : 

<img width="1291" height="616" alt="image" src="https://github.com/user-attachments/assets/7d7b69f8-3bac-4c0c-a04f-8403a1769d1c" />

And it worked perfectly . 

Now going back to the lab , it should be solved .

<img width="1676" height="893" alt="image" src="https://github.com/user-attachments/assets/94c16b8f-6da3-42f2-9816-476ffac99931" />

That was all for this lab , see you in the next one :) 
