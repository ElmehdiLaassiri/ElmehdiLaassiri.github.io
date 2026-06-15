---
title: "Webverselabs Challenge FreightManifest SQLi  "
date: 2026-06-12 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


## Scenario : 

Aerotrust's catalog backend dates back to a 1999 migration off a homegrown FoxPro setup. The PHP front-end was wrapped around it the same year. The underlying storage was quietly swapped out in 2014 for cost reasons, but the team kept the legacy response surface untouched — partly because the procurement team's tooling scrapes that text and complains when the format changes, partly because nobody owns that page anymore. The label on the tin and what's inside the tin haven't matched for over a decade.


## Solution : 

<img width="1810" height="914" alt="image" src="https://github.com/user-attachments/assets/f9d92584-e01a-49ba-906d-b9662f874103" />

Checking the different endpoints : **avionics** , **powerplant**, etc. We find a parameter sent in each GET Request : 

<img width="1661" height="606" alt="image" src="https://github.com/user-attachments/assets/72d1928b-7361-468f-a0fd-3f22dcafa54c" />

Tried Injecting inside this parameter , started by a simple ' to cause an error but that didn't work : 

<img width="1669" height="752" alt="image" src="https://github.com/user-attachments/assets/6244c7ae-c058-48b9-9a13-d066b7271aaa" />

Whenever we inject something it redirects us to the login page, i also checked with sqlmap but that didn't work either . 

<img width="1786" height="608" alt="image" src="https://github.com/user-attachments/assets/22d8afc4-a328-43d9-b53e-708b74fa7377" />

Moving on, we can use the search feature : 

<img width="1869" height="762" alt="image" src="https://github.com/user-attachments/assets/d4434ea1-fe9f-4a36-a6a7-53ca081ea1df" />

Tried injecting the parameter which did result in an error : 

<img width="1501" height="667" alt="image" src="https://github.com/user-attachments/assets/e18bd3ed-c82f-4190-8a9f-c82d64d487f8" />

To confirm the SQLi , we can either add another ' and hope this removes the error which means the query got closed successfully : 

<img width="1550" height="640" alt="image" src="https://github.com/user-attachments/assets/b2429b1e-6e1a-4528-8189-f636c1f0bef9" />

Or we can comment the rest without closing the query and if this doesn't return an error this confirms the sqli as well : 

<img width="1488" height="509" alt="image" src="https://github.com/user-attachments/assets/7272aafa-5323-4fc6-b1dd-8be563736bce" />

Perfect, now we are sure the parameter is vulnerable to a SQLi. 

Looking at the error : "ORA-00904: invalid identifier — ORA-00933: SQL command not properly ended"

This indicates that the DB is Oracle , we can also use SQLmap to confirm the sqli : 

<img width="1843" height="896" alt="image" src="https://github.com/user-attachments/assets/216cf006-0731-4631-820f-fe080fe89f73" />

Now with this information we can confirm that there is a SQLi and that the DB is Oracle . 

BUT when we let sqlmap run , it returns that the parameter was a false positive . 

<img width="1631" height="575" alt="image" src="https://github.com/user-attachments/assets/263cb980-5dda-438a-9be3-6459343aeddb" />

This is weird , since we can confirm that there is a SQLi so the fact that it returns that it is a false positive can be caused due to having the Wrong DB . 

I tried confirming the SQLi by trying to detect the number of columns : 

<img width="1615" height="702" alt="image" src="https://github.com/user-attachments/assets/1df5fb50-aa41-481c-b77e-e730472d426c" />

We keep adding numbers until we no longer have that error : 

<img width="1602" height="819" alt="image" src="https://github.com/user-attachments/assets/b484b3bf-0445-4264-be0e-338eb51d5c95" />

Perfect, we find that the number of columns is 6 and that first , third and Fourth columns are all returned to us . 

Now since at first i thought this was an Oracle DB , i tried some payloads to get the DB name , and tables : 

```sql
' UNION SELECT NULL,2,3,4,5,6 FROM dual-- -   : This worked
' UNION SELECT NULL,2,3,4,5,6 FROM v$version-- -   : This failed (privilege issue)
' UNION SELECT table_name, NULL, NULL, NULL, NULL, NULL FROM all_tables WHERE owner='AEROTRUST'-- -  : Also worked and returned PARTS 
```

First one didn't return an error which is a good sign , it means the Table exists , and apparently DUAL is a known dummy table from ORACLE . 

Second one returned an error : 

<img width="1591" height="661" alt="image" src="https://github.com/user-attachments/assets/fc92095c-b9e1-4b96-9730-fb2dc6b20f87" />

Which is weird since v$version is specific to Oracle so by not existing something was wrong . 

And finally the last one worked , and it did return the table name which in this case was PARTS . 

<img width="1585" height="659" alt="image" src="https://github.com/user-attachments/assets/397d2057-7414-48cd-9e07-f6de7abcca6e" />




<img width="1576" height="719" alt="image" src="https://github.com/user-attachments/assets/e2fa8e97-7ac5-4016-909a-73ffc660d6f4" />
