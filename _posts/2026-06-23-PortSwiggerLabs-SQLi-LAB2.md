---
title: " PortSwiggerlabs: SQL injection vulnerability allowing login bypass "
date: 2026-06-23 00:00:00 +0000
categories: [PortSwiggerLabs , SQL Injection]
tags: [PortSwiggerLabs , SQLi , Challenge , Web_Attacks ]
---


## Information : 

This lab contains a SQL injection vulnerability in the login function.

To solve the lab, perform a SQL injection attack that logs in to the application as the administrator user.

## Solution : 

<img width="1858" height="926" alt="image" src="https://github.com/user-attachments/assets/4ed93d08-22d1-4cf7-933c-11e880a48f20" />

Let's try login in : 

<img width="1840" height="847" alt="image" src="https://github.com/user-attachments/assets/44d8b030-715b-4836-a403-ecb9c8d7c691" />

Whenever we find a Login page , we can always test for Login Bypass via SQLi , here is a list of payloads i like to use : 

<img width="1275" height="681" alt="image" src="https://github.com/user-attachments/assets/f488e37c-e853-46f1-b4e2-156363e2688d" />

We can either use Intruder to test all of these , but since there aren't that many , we'll do it manually : 

```sql
' or 1=1--
```

This one worked for me : 

<img width="1776" height="740" alt="image" src="https://github.com/user-attachments/assets/de99bd4d-22c6-4319-9441-570fcc138590" />

Looking back at our Dashbaord , we should see that the lab was solved : 

<img width="1897" height="811" alt="image" src="https://github.com/user-attachments/assets/30e505b3-f307-4811-8639-a6f85bf18ba5" />

That was all for this lab, See you in the next one :) 
