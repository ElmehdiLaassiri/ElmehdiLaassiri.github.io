---
title: " PortSwiggerlabs: SQL injection attack, querying the database type and version on Oracle "
date: 2026-06-23 00:00:00 +0000
categories: [PortSwiggerLabs , SQL Injection]
tags: [PortSwiggerLabs , SQLi , Challenge , Web_Attacks ]
---


## Information : 

This lab contains a SQL injection vulnerability in the product category filter. You can use a UNION attack to retrieve the results from an injected query.

To solve the lab, display the database version string.


## Solution : 

First we need to identify the vulnerable parameter : 

<img width="1761" height="945" alt="image" src="https://github.com/user-attachments/assets/e233b2b5-3804-469a-8019-f4d0f3782afb" />

We find this categrory parameter , now we start by injecting a simple payload to see if we can break the query and cause an internal server error . 

<img width="1706" height="703" alt="image" src="https://github.com/user-attachments/assets/fdc5a1e0-fa3f-4d15-b491-13536dbd6810" />

Perfect , we get our internal error . 

Now to confirm the SQLi , either add ' to fix the query and remove the error , or comment everything which will remove the error as well , if we are able to get the error removed by doing one of the two this will confirm the SQLi . 

<img width="1634" height="932" alt="image" src="https://github.com/user-attachments/assets/f1fd1c89-9b2e-407c-800e-f1d7a807b50e" />

Perfect Our SQLi is confirmed . 

Now first test will be to check for UNION based SQLi . First we need to know how many columns exist :

We can do that by using the order by operator , we keep adding numbers until we get an error , the moment we get an error we know the number of columns is the number before causing the error . 

```sql
'order by X-- -
```

<img width="1558" height="747" alt="image" src="https://github.com/user-attachments/assets/1a2f54f0-eca1-4d65-aa99-ea4ca2ade7de" />

2 doesn't return an error , but if we try 3 , we get an internal error : 

<img width="1544" height="753" alt="image" src="https://github.com/user-attachments/assets/40b4b939-f11c-415e-9945-e93560a3bee9" />

This confirms that only 2 collumns exist . 

Now to determine the DB used we can try many payloads and depending on which one doesn't return errors we should know the DB being used . 

```bash
https://portswigger.net/web-security/sql-injection/cheat-sheet
```

<img width="1152" height="400" alt="image" src="https://github.com/user-attachments/assets/27c11b6e-b4e2-4229-a897-cbab018d75e9" />

I Kept trying these payloads but they didn't work : 

<img width="1599" height="724" alt="image" src="https://github.com/user-attachments/assets/2c06e31e-337d-4a50-8c9d-1d249a6c13fe" />

Then i realised, maybe the v$version table doesn't allow numbers which means if we do UNION with a number it crashes , always use NULL since it will always work since it is compatible with any Data Type avoiding type mismatch errors that would crash the query. 

```sql
'UNION SELECT banner,NULL FROM v$version-- -
```

Make sure you URL encode everything , and comment the rest always to avoid errors : 

<img width="1511" height="796" alt="image" src="https://github.com/user-attachments/assets/ae06a21a-8ba8-4797-b789-ed477ba0b627" />

This one worked perfectly which confirms this is actually an ORACLE DB and we get the version for it in our response . 

Upon retrieving the version , we will see that the lab is Solved now . 

<img width="1854" height="977" alt="image" src="https://github.com/user-attachments/assets/27af577b-8805-45f2-b8c0-817f9a932e17" />

That was all for this lab, See you in the next one :) 
