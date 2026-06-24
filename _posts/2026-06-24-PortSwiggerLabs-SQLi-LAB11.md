---
title: " PortSwiggerlabs: Visible error-based SQL injection "
date: 2026-06-24 11:00:00 +0000
categories: [PortSwiggerLabs , SQL Injection]
tags: [PortSwiggerLabs , SQLi , Challenge , Web_Attacks ]
---


## Information : 

This lab contains a SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs a SQL query containing the value of the submitted cookie. The results of the SQL query are not returned.

The database contains a different table called users, with columns called username and password. To solve the lab, find a way to leak the password for the administrator user, then log in to their account.



## Solution : 

First let's make a normal request : 

<img width="1757" height="775" alt="image" src="https://github.com/user-attachments/assets/14113348-b734-4fa6-be6e-0f0939c922ed" />

We see this tracking ID , (tried injecting the category parameter but that didn't work) , if we inject the tracking ID parameter : 

<img width="1797" height="632" alt="image" src="https://github.com/user-attachments/assets/016f0998-b93c-4ce4-b04d-e403b52f6143" />

We get a verbose message , which shows us the query made to the server . First let's confirm the SQLi by commenting the rest : 

<img width="1549" height="794" alt="image" src="https://github.com/user-attachments/assets/96e6fce7-677a-4ac0-bd4a-158dfbc96d23" />

We see that we no longer have an error which confirms the vulnerability .

Now here we can use a Condition Based SQLi : 

The condition we will be using is CAST : which is convert this value to a specific type Eg : CAST('1' AS INT) this should work perfectly since 1 is an INT .

But if we tried CAST(ADMIN AS INT) this will return an error . 

Now this is our condition , we should verify if it's true : 1=CAST(1 AS INT) : This condition should return TRUE , and the query shouldn't break so let's test it : 

```sql
'AND 1=CAST('1' AS INT)-- - : This shouldn't return any errors .
```
<img width="1610" height="770" alt="image" src="https://github.com/user-attachments/assets/1c1b10f8-7744-4654-ae71-90b6d222cd05" />

Now let's test a different one to confirm that this is a Condition based SQLi : 

```sql
'AND 1=CAST('a' AS INT)-- - : This should result in an error .
```

<img width="1518" height="613" alt="image" src="https://github.com/user-attachments/assets/d7c23572-299a-4abb-b8c3-2a34d6ca62a3" />

Perfect, this confirms the Condition based SQLi . 

Now how can we abuse this? We already know there is a table called users that has username and password columns . 

We can use CAST and inside of it we make a condition that will return an error , but will still execute something and return it back as part of the error . 

```sql
1=CAST((SELECT username FROM users LIMIT 1) AS INT) 
```

This will select the first Row only (LIMIT 1) then try and transform it to an INT , which will result in an error and in the error it should give us the content of the first SELECT we made : 

```sql
TrackingId=' AND 1=CAST((SELECT username FROM users LIMIT 1) AS INT)--
```

<img width="1498" height="638" alt="image" src="https://github.com/user-attachments/assets/c37d5b83-b723-48d1-9609-1ac216c6529f" />

Perfect we get our username . Now to retrieve the password , same thing : 

```sql
 1=CAST((SELECT password from users where username = 'administrator') AS INT)
```

<img width="1546" height="656" alt="image" src="https://github.com/user-attachments/assets/e29e836c-71f9-42f5-8aa1-aeb65bfb3045" />

I kept trying to make it work but i couldn't , probably since it's too big .

Another way around it , we already know the first Row was for administrator , LIMIT1 , so if we do the same thing and dump first Row of passwords we should get the admin's password . 

```sql
 1=CAST((SELECT password from users LIMIT 1) AS INT)
```

<img width="1542" height="523" alt="image" src="https://github.com/user-attachments/assets/1817ec98-d80e-42a3-a789-c75f09942e54" />

That way we got the admin's password : 

<img width="1640" height="675" alt="image" src="https://github.com/user-attachments/assets/4487171e-ebe6-46fb-be7a-ad9f76035c41" />

Once we login as the admin , we should see that the lab was solved . 

That was all for this lab, see you in the next one . 

