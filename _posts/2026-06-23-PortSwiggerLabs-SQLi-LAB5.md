---
title: " PortSwiggerlabs: SQL injection attack, listing the database contents on non-Oracle databases "
date: 2026-06-23 00:00:00 +0000
categories: [PortSwiggerLabs , SQL Injection]
tags: [PortSwiggerLabs , SQLi , Challenge , Web_Attacks ]
---


## Information : 

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response so you can use a UNION attack to retrieve data from other tables.

The application has a login function, and the database contains a table that holds usernames and passwords. You need to determine the name of this table and the columns it contains, then retrieve the contents of the table to obtain the username and password of all users.

To solve the lab, log in as the administrator user.


## Solution : 

First we need to find where to inject , we find the category parameter , first check is to inject a simple ' payload and hope we get an internal error . 

<img width="1680" height="499" alt="image" src="https://github.com/user-attachments/assets/d2ba7089-7d33-4754-be26-9e2420f795b4" />

Now to confirm the SQLi , either add ' to fix the query and remove the error , or comment everything which will remove the error as well , if we are able to get the error removed by doing one of the two this will confirm the SQLi . 

<img width="1764" height="905" alt="image" src="https://github.com/user-attachments/assets/b51aed27-a778-4314-989e-f76ddf8da5f0" />

Perfect we confirm that the server is vulnerable to a SQLi . 

Now we need to find the number of columns : 

We can use order by for this : 

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

Eventually the payload that worked was version() which confirmed it was a POSTGRESQL . 

<img width="1566" height="773" alt="image" src="https://github.com/user-attachments/assets/dc8634bc-cf8f-45c9-bd54-34d05c25f5fe" />

Perfect , now let's dump the Table names first : 

Since this is a PostgreSQL db , We can use the information_schema.tables to list all table names . 

```sql
'UNION SELECT NULL,table_name FROM information_schema.tables-- -
```

<img width="1609" height="830" alt="image" src="https://github.com/user-attachments/assets/553f1a68-082c-4176-91e7-d3251deb9ddc" />

Perfect , we are able to dump the table names , now moving on , we find an interesting table which is 'users_ewzhui' which probably holds usernames and password : 

To dump its content we first need to list the column names for this table , again this is PostgreSQL so we can use the information_schema.columns with WHERE operator : 

```sql
'UNION SELECT NULL,column_name FROM information_schema.columns WHERE table_name='users_ewzhui'-- -
```

<img width="1735" height="922" alt="image" src="https://github.com/user-attachments/assets/edcc394b-3598-421a-90e2-6fd3db26fea8" />

Perfect now that we know the column names and the table name , we can just dump them : 

```sql
'UNION SELECT password_gvamsl,username_ogeext FROM users_ewzhui-- -
```

<img width="1758" height="930" alt="image" src="https://github.com/user-attachments/assets/c4be856d-4f09-4461-9316-a52d76cd089c" />


Let's try to login in with this password : 

<img width="1884" height="817" alt="image" src="https://github.com/user-attachments/assets/38ea64ba-62b0-432d-b7e6-fba94db20f56" />

Just like that we are able to solve the lab . 

That was all for this lab, see you in the next one :) 

