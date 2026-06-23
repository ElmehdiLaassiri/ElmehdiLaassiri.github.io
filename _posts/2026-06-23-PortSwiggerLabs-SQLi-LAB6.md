---
title: " PortSwiggerlabs: SQL injection attack, listing the database contents on Oracle "
date: 2026-06-23 00:00:00 +0000
categories: [PortSwiggerLabs , SQL Injection]
tags: [PortSwiggerLabs , SQLi , Challenge , Web_Attacks ]
---


## Information : 

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response so you can use a UNION attack to retrieve data from other tables.

The application has a login function, and the database contains a table that holds usernames and passwords. You need to determine the name of this table and the columns it contains, then retrieve the contents of the table to obtain the username and password of all users.

To solve the lab, log in as the administrator user.


## Solution :

If we check the category parameter : 

<img width="1466" height="811" alt="image" src="https://github.com/user-attachments/assets/d0a59d59-4af9-4af7-ad2c-f22c408bfbc0" />

We first can check if the parameter is vulnerable by injecting a simple payload and see if we get an internal server error . 

<img width="1432" height="506" alt="image" src="https://github.com/user-attachments/assets/ac571ed7-34d9-47b0-9c07-d3a11c49d783" />

Now to confirm the SQLi , either add ' to fix the query and remove the error , or comment everything which will remove the error as well , if we are able to get the error removed by doing one of the two this will confirm the SQLi . 

<img width="1764" height="905" alt="image" src="https://github.com/user-attachments/assets/b51aed27-a778-4314-989e-f76ddf8da5f0" />

Perfect we confirm that the server is vulnerable to a SQLi . 

Now we need to find the number of columns : 

We can use order by for this : 

```sql
'Order by 1-- -
```

<img width="1508" height="813" alt="image" src="https://github.com/user-attachments/assets/c3115c36-d5bc-4839-8c54-7fb13bc177b9" />

Still no errors , which means it's probably 2 or higher . 

<img width="1177" height="455" alt="image" src="https://github.com/user-attachments/assets/4ce0b7e3-3be6-4076-8e4c-a3bda6026945" />

The moment we try 3 we get an error, this confirms that the number of columns is 2 . Now let's try and see which DB is used . 

Now to determine the DB being used , we can use this list of payloads and based on which ones don't return errors , we can tell the DB used. 

Quick Checklist : 

```bash
https://portswigger.net/web-security/sql-injection/cheat-sheet
```

<img width="1200" height="392" alt="image" src="https://github.com/user-attachments/assets/3a980858-115d-40bb-9d53-0c1dad300522" />

```sql
'UNION SELECT banner,NULL FROM v$version-- -
```

<img width="1308" height="880" alt="image" src="https://github.com/user-attachments/assets/7144a559-bf59-47b3-a343-acc6ecb2bcf6" />

This worked perfectly and we get the version of the DB , which means it is an oracle DB , now let's list all the table names :

Looking at the CheatSheet , we see that to get the table names we can query the all_tables which is a standard data dictionary view. 

```sql
'UNION SELECT table_name,NULL FROM all_tables-- -
```

<img width="1433" height="908" alt="image" src="https://github.com/user-attachments/assets/5235a327-565b-4ac9-b1f4-57f5f908eaaa" />

We find the 'APP_USERS_AND_ROLES' to be quiet interesting , let's list the columns for this table . 

Again we can follow the CheatSheet , we see that we can query the all_tabs_columns :

```sql
'UNION SELECT column_name,NULL FROM all_tab_columns WHERE table_name = 'APP_USERS_AND_ROLES'-- -
```

<img width="1462" height="644" alt="image" src="https://github.com/user-attachments/assets/c2b0ce5b-2f62-4811-9a4f-f171ae0c8c60" />

I don't think this is the one we're looking for, specailly that this is a default one, role and names doesn't contain passwords or anything like that . 

<img width="1531" height="667" alt="image" src="https://github.com/user-attachments/assets/de424836-34cd-40fe-80c1-459a6811f4a3" />

We find this one : 'USERS_PHOCZD' this is a custom DB so it can be useful . 

```sql
'UNION SELECT column_name,NULL FROM all_tab_columns WHERE table_name = 'USERS_PHOCZD'-- -
```

Keep in mind we have to URL encode everything if we're on Burp and don't forget to comment the rest .

<img width="1415" height="846" alt="image" src="https://github.com/user-attachments/assets/9af22aa5-2870-42ad-8c99-69028aa54b9e" />

This is the one we're looking for , now that  we know the column names as well as the table we want to get data from , the rest is pretty easy : 

```sql
'UNION SELECT PASSWORD_HWROIQ,USERNAME_QJSWNE From USERS_PHOCZD-- -
```

<img width="1509" height="952" alt="image" src="https://github.com/user-attachments/assets/b0f18c79-2c65-4c26-907c-3695ea26384f" />

Just like that we get the admin's password . 

<img width="1592" height="760" alt="image" src="https://github.com/user-attachments/assets/9cadc163-fe97-4ecc-81f5-0a1091c80cec" />

Once we login using the admin's creds the lab will be solved . 

That was it for this lab. See you in the next one :)

