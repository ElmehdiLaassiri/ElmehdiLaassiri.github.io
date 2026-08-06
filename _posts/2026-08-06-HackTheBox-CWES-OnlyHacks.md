---
title: " Hack The Box OnlyHacks Walkthrough"
date: 2026-08-07 00:00:00 +0000
categories: [CWES Preparation Path ]
tags: [CWES Preparation, HackTheBox, Challenge , Web_Attacks ]
---



## Challenge Scenario :

Dating and matching can be exciting especially during Valentine's, but it’s important to stay vigilant for impostors. Can you help identify possible frauds?

## Solution : 

This challenge was pretty easy , nothing complicated overall . 

First we visit the given IP address : 

<img width="1381" height="821" alt="image" src="https://github.com/user-attachments/assets/48afb585-7792-406c-aaa3-195528bab114" />

First thing i tried is basic auth bypass via SQLi but that didn't work , so i moved to FUZZING the application for directories : 

```bash
feroxbuster -u http://154.57.164.82:32598/
```

<img width="1772" height="642" alt="image" src="https://github.com/user-attachments/assets/2c7b5958-e905-4085-8bae-86ae8f64d0ea" />

Both chat and matches return 500 and 401 , the rest redirects us back to the login page , let's just create an account and browse the application further : 

<img width="1292" height="767" alt="image" src="https://github.com/user-attachments/assets/4313a230-0061-4393-9f82-d43f3fed5e60" />

When creating the account , we see that we can't modify any role or get admin rights by intercepting and modifying the request : 

<img width="1724" height="668" alt="image" src="https://github.com/user-attachments/assets/52ec841e-fba3-4a9e-b7f5-a4b3178505df" />

