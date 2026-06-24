---
title: " PortSwiggerlabs: SQL injection UNION attack, retrieving multiple values in a single column "
date: 2026-06-24 00:00:00 +0000
categories: [PortSwiggerLabs , SQL Injection]
tags: [PortSwiggerLabs , SQLi , Challenge , Web_Attacks ]
---




## Information : 

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response so you can use a UNION attack to retrieve data from other tables.

The database contains a different table called users, with columns called username and password.

To solve the lab, perform a SQL injection UNION attack that retrieves all usernames and passwords, and use the information to log in as the administrator user.


## Solution : 

This one is way too easy they already gave us the table name as well as the column names , so all we need to do is find an injectable spot, figure our the number of columns and craft our query that should look something like this : 

```sql
'UNION SELECT username,password,.... From users-- - : the ... are for other columns if they exist . 
```

<img width="1661" height="510" alt="image" src="https://github.com/user-attachments/assets/245b04f1-576a-4068-aa5f-6e9d2c4dc0f5" />

We first inject in the category field which results in an internal server error , if we wanted to confirm it , either add another ' or just comment the rest , both will fix the query and remove the error , confirming our SQLi. 

<img width="1668" height="948" alt="image" src="https://github.com/user-attachments/assets/306cce68-254e-4672-b317-8c8e63a196a9" />

Now back to our lab , we first need to identify the colum number , i will use orderby . 

<img width="1590" height="463" alt="image" src="https://github.com/user-attachments/assets/5aa4c188-8172-436e-b9c5-f428ddc1caad" />

We get an error at number 3 which means 2 is the number of columns , now we just retrieve the username and password by dumping both columns from the users table , pretty straight forward :

```sql
'UNION SELECT username,password From users-- - 
```

<img width="1392" height="433" alt="image" src="https://github.com/user-attachments/assets/9771d8ec-1a08-4bda-886b-70776a20ef24" />

This returned an error , so we need to step back for a second and see if both columns accept Strings or not.

<img width="1449" height="734" alt="image" src="https://github.com/user-attachments/assets/29eff52f-fbea-4e52-b300-40778ac18f6b" />

The second colum does , now let's check the first one : 

<img width="1493" height="441" alt="image" src="https://github.com/user-attachments/assets/c33c5b3e-3725-40eb-8925-b381c1065d0d" />

This causes an internal server error , which means we got a type mismatch , so first column doesn't accept strings .

I only used the second one to retrieve usernames and we got these : 

<img width="1580" height="860" alt="image" src="https://github.com/user-attachments/assets/d402c8f5-6f73-4184-9ba0-a882539c6abd" />

Now if we try dumping the passwords : 

<img width="1659" height="845" alt="image" src="https://github.com/user-attachments/assets/5ed46893-f71a-4be9-ad9a-7bcfe9f7ce98" />

Perfect we got the passwords as well , Only 3 users and 3 password we should try all of them to see which one is for the administrator : 

<img width="1675" height="768" alt="image" src="https://github.com/user-attachments/assets/b5146e64-2ff2-480e-81b7-d6d1411b332c" />

The third one was the one for the administrator , so once we login as the admin , the lab should be solved . 

## Few thing to Note :

The tables inside the DB might have multiple columns , but for our request only 2 are necessary in the query made to the server .

This one didn't need both of them to be strings , maybe it sends an ID or some sort of INT to the server in the first column . 

Whenever we only have 1 spot to retrieve data with , we can use concatenation : 

```sql
' UNION SELECT NULL,username||'~'||password FROM users-- 
```

<img width="1547" height="836" alt="image" src="https://github.com/user-attachments/assets/b717754c-3a6c-4411-902a-10c58bcec7e6" />

This will retrieve both of them at once seperated by a ~ , this is useful when we can only use 1 field to retrieve Strings and we want to extract multiple columns from a table that holds Strings 

That was all for this lab , see you in the next one :) 

