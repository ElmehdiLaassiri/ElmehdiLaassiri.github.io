---
title: " PortSwiggerlabs: SQL injection attack, querying the database type and version on MySQL and Microsoft "
date: 2026-06-23 00:00:00 +0000
categories: [PortSwiggerLabs , SQL Injection]
tags: [PortSwiggerLabs , SQLi , Challenge , Web_Attacks ]
---


## Information : 

This lab contains a SQL injection vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.

To solve the lab, display the database version string.


## Solution : 

First we need to find the injection spot , just navigate the app like a normal user , check for Parameters in URLs or POST Requests .

<img width="1413" height="838" alt="image" src="https://github.com/user-attachments/assets/003f87fd-f6b8-47e1-8a1a-3d1c6c7b5773" />

We first inject a simple payload to see if we can get an internal error . 

<img width="1404" height="596" alt="image" src="https://github.com/user-attachments/assets/fefb4f19-0534-4af7-b857-97e9bc8a2f98" />

Perfect we got our internal server error . Now to confirm the SQLi , either add ' to fix the query and remove the error , or comment everything which will remove the error as well , if we are able to get the error removed by doing one of the two this will confirm the SQLi . 

<img width="1422" height="869" alt="image" src="https://github.com/user-attachments/assets/d696e993-ea0f-46a3-ad35-1bca27341de2" />

If we comment the rest , we get the error removed , perfect :) This confirms the SQLi , we just need to find the number of columns now . Use order by for this : 

```sql
'Order by 1-- -
```

Keep increasing the number until we get an error :

<img width="1484" height="693" alt="image" src="https://github.com/user-attachments/assets/16ed939b-700c-4f3a-8390-4005bc74c12d" />

We get an error at 3 , checking 2 doesn't return an error : 

<img width="1585" height="688" alt="image" src="https://github.com/user-attachments/assets/bb9fb143-d6ea-4b11-bd31-6ea5304b6c7b" />

This confirms that the number of columns is 2 .

Now to determine the DB being used , we can use this list of payloads and based on which ones don't return errors , we can tell the DB used. 

Quick Checklist : 

<img width="1200" height="392" alt="image" src="https://github.com/user-attachments/assets/3a980858-115d-40bb-9d53-0c1dad300522" />

Starting with the first one : 

<img width="1466" height="639" alt="image" src="https://github.com/user-attachments/assets/49b56ffb-f215-4ac1-8e44-7f76f5a9c8fe" />

An internal error which confirms it's not Oracle . 

<img width="1594" height="670" alt="image" src="https://github.com/user-attachments/assets/4281c195-2323-499b-b2cd-cc9dce7f7bce" />

Perfect , we get the version , as well as the OS :) 

Going back to our app , we should see that the lab was solved . 

<img width="1757" height="905" alt="image" src="https://github.com/user-attachments/assets/06af3e7b-9660-4aae-9e13-b2a236b6d5ad" />

Just make sure you always URL encode everything and comment the rest to avoid errors . 

That's all for this lab , see you in the next one :) 
