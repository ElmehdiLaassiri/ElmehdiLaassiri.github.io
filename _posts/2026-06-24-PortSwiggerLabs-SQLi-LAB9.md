---
title: " PortSwiggerlabs: SQL injection UNION attack, finding a column containing text "
date: 2026-06-24 00:00:00 +0000
categories: [PortSwiggerLabs , SQL Injection]
tags: [PortSwiggerLabs , SQLi , Challenge , Web_Attacks ]
---


## Information :

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables. To construct such an attack, you first need to determine the number of columns returned by the query. You can do this using a technique you learned in a previous lab. The next step is to identify a column that is compatible with string data.

The lab will provide a random value that you need to make appear within the query results. To solve the lab, perform a SQL injection UNION attack that returns an additional row containing the value provided. This technique helps you determine which columns are compatible with string data.



## Solution : 


<img width="1676" height="913" alt="image" src="https://github.com/user-attachments/assets/9f0e6a34-20c4-4af3-9458-c5d047c5eea6" />

We first start by injecting the category parameter to see if we get an internal server error : 

<img width="1456" height="498" alt="image" src="https://github.com/user-attachments/assets/e477ff88-2d59-4405-b898-db6b9fa09aaa" />

And we do , now to confirm the SQLi , either add another ' or just comment the rest of the query , this will make sure the query isn't broken and it should remove the error we got earlier , both of them will confirm that we have a SQLi. 

<img width="1406" height="782" alt="image" src="https://github.com/user-attachments/assets/c5eff3f6-c11e-4396-9c4f-9517dc0c79a4" />

Perfect now that our SQLi is confirmed , we can start by getting the column number , for this i will use Order by .

<img width="1495" height="865" alt="image" src="https://github.com/user-attachments/assets/434c07a2-9557-4cfd-aea4-65adcecc4426" />

2 doesn't return errors . We will keep incrementing until we get an error .

<img width="1623" height="518" alt="image" src="https://github.com/user-attachments/assets/82abe09a-2e53-45e0-bb64-4c7d8db6589f" />

We get it at 4 , this means there are 3 columns . now we should check which DB is being used . 

Quick Checklist : 
```sql
'UNION SELECT banner,NULL,NULL FROM v$version-- - : ORACLE .
'UNION SELECT version,NULL,NULL FROM v$instance-- - : ORACLE .
'UNION SELECT @@version,NULL,NULL-- - : Microsoft / MySQL . 
'UNION SELECT NULL,,NULL,version()-- - : PostgreSQL . 
```
None of these worked sadly , so i couldn't determine the DB used . 

Back to our lab , we need to find a field that can take Strings : Now since we are doing a UNION Select attack , we need to match both the column number as well as the data type , otherwise we will get an error . 

So first we will use NULL for all of them then for each iteration we will modify a field to a string and see which one doesn't return an error . 

```sql
'UNION SELECT NULL,NULL,NULL-- - : This is the first one = shouldn't return an error .
```

<img width="1577" height="741" alt="image" src="https://github.com/user-attachments/assets/d0d2523d-71b0-41ff-a5ab-4ea7835f86ca" />

We're using 3 NULL since we already know the column number is 3 .

```sql
'UNION SELECT 'aaaa',NULL,NULL-- - : Checking if the first column allows strings
'UNION SELECT NULL,'aaaa',NULL-- - : checking if 2nd field allows strings 
'UNION SELECT NULL,NULL,'aaaa'-- - ; Same thing for the last field 
```

The First and Third one and returned an error . 

<img width="1415" height="520" alt="image" src="https://github.com/user-attachments/assets/e2eb02b6-2d70-4dac-b7d1-d17a85f2136d" />

But the second one didn't which means the Second column contains strings .

<img width="1571" height="732" alt="image" src="https://github.com/user-attachments/assets/e73ed24f-f1b3-4cbf-a404-825ffd16b16d" />

Now to solve the Lab we need to retrieve this String : DHLSOd

<img width="1473" height="902" alt="image" src="https://github.com/user-attachments/assets/711ae461-ce86-4d43-a812-06115b2e46db" />

That was all for this lab , see you in the next one :) 

