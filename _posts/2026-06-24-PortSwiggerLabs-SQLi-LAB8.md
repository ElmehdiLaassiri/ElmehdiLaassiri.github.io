---
title: " PortSwiggerlabs: SQL injection UNION attack, retrieving data from other tables "
date: 2026-06-23 00:00:00 +0000
categories: [PortSwiggerLabs , SQL Injection]
tags: [PortSwiggerLabs , SQLi , Challenge , Web_Attacks ]
---




## Information : 

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables. To construct such an attack, you need to combine some of the techniques you learned in previous labs.

To solve the lab, perform a SQL injection UNION attack that retrieves all usernames and passwords, and use the information to log in as the administrator user.


## Solution : 

First we need to find the vulnerable parameter , luckily for us , there is only the category parameter that we can test : 

<img width="1616" height="453" alt="image" src="https://github.com/user-attachments/assets/9903283e-8cc3-4468-a349-277cce95d688" />

First i injected a simple payload : ' , since this returned an internal server error , this is a huge indicator that there might be a SQLi . 

Now to confirm the sqli , either add another ' which will fix the query and remove the Error , or simply just comment what comes after the ' we injected , this way we can get rid of the error , and if we don't haver the error this confirms the SQLi . 

<img width="1747" height="738" alt="image" src="https://github.com/user-attachments/assets/1da42805-9345-461b-bc60-06e2e37bca7c" />

Perfect the error is gone , this confirms the SQLi . 

Now we start by identifying the column numbers , i used order by for this one , but you can use UNION SELECT with NULLs . 

Detailed method on how to identify the columns number: 

```bash
https://elmehdilaassiri.github.io/posts/PortSwiggerLabs-SQLi-LAB7/
```

<img width="1560" height="732" alt="image" src="https://github.com/user-attachments/assets/a34123c6-574a-4d25-ad7c-2e9e31aea2ed" />

If we specify the number of columns to be 2 we don't get any errors , but if we try 3 we immediately get an error : 

<img width="1658" height="560" alt="image" src="https://github.com/user-attachments/assets/0931d09d-852a-4a90-864a-25af18754e02" />

This confirms that the number of columns is 2 , now to identify the DB i used this Cheat Sheet : 

```bash
https://portswigger.net/web-security/sql-injection/cheat-sheet
```

<img width="1595" height="447" alt="image" src="https://github.com/user-attachments/assets/819e2fda-09cf-429c-9d97-b681ab9a832c" />

This confirms the DB isn't MySQL nor Microsoft .

<img width="1734" height="855" alt="image" src="https://github.com/user-attachments/assets/d1d9077b-8df1-4607-8a02-1d35152899ca" />

This one worked  : 

```sql
SELECT version()
```

Which is specific to PostgreSQL . 

<img width="1072" height="644" alt="image" src="https://github.com/user-attachments/assets/131348af-f935-4f74-bffa-f55f3b36ba38" />

Now we need to list the table names , since this is a POSTGRESQL , we will be using the information_schema.tables : 

```sql
'UNION SELECT NULL,table_name FROM information_schema.tables-- -
```

<img width="1729" height="891" alt="image" src="https://github.com/user-attachments/assets/c66992e3-5634-42d3-b922-243250ebba06" />

Perfect , we find a custom Table called users : 

<img width="1469" height="509" alt="image" src="https://github.com/user-attachments/assets/52865cb1-ad82-4c24-a481-8880877c88e2" />

Now to list the column names for this table , we can use the 'information_schema.columns' : 

```sql
'UNION SELECT NULL,column_name FROM information_schema.columns WHERE table_name=users-- -
```

<img width="1786" height="809" alt="image" src="https://github.com/user-attachments/assets/a8d4ea8b-7599-4f4e-89f5-6e11a7d4d137" />

Now that we got the column name as well as the table name , retrieving the content should be pretty easy : 

```sql
'UNION SELECT username,password FROM users-- -
```

Make sure you comment the rest , and URL encode everything if you're using Burp : 

<img width="1755" height="931" alt="image" src="https://github.com/user-attachments/assets/760a5f9e-5724-4ff5-9a88-2dbcee901211" />

Perfect we got the administrator password , we can use it to login and solve the lab .

<img width="1738" height="705" alt="image" src="https://github.com/user-attachments/assets/31ac3da4-580a-4ee5-85b3-334f2b98b23d" />

Once we login as the admin , we should see lab Solved .

That was all for this lab, see you in the next one :) 
