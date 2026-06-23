---
title: " PortSwiggerlabs: SQL injection UNION attack, determining the number of columns returned by the query "
date: 2026-06-24 00:00:00 +0000
categories: [PortSwiggerLabs , SQL Injection]
tags: [PortSwiggerLabs , SQLi , Challenge , Web_Attacks ]
---



## Information : 

This lab contains a SQL injection vulnerability in the product category filter. The results from the query are returned in the application's response, so you can use a UNION attack to retrieve data from other tables. The first step of such an attack is to determine the number of columns that are being returned by the query. You will then use this technique in subsequent labs to construct the full attack.

To solve the lab, determine the number of columns returned by the query by performing a SQL injection UNION attack that returns an additional row containing null values.


## Solution : 

We first need to find the vulnerable parameter , quick test is to inject a simple payload ' , and see if we can cause the query to break and eventually result in an internal server error . 

<img width="1653" height="930" alt="image" src="https://github.com/user-attachments/assets/55246967-bc6d-42a7-8555-590fd247dcd4" />

First i tried injecting the product ID parameter : 

<img width="1025" height="299" alt="image" src="https://github.com/user-attachments/assets/998dd504-be74-4e68-a2ee-efc7d9da432f" />

This didn't work sadly , moving on to the category parameter : 

<img width="1552" height="480" alt="image" src="https://github.com/user-attachments/assets/96ba65ae-2bb6-478e-8b40-a446f093b0c6" />

Perfect we got our internal server error , now to confirm the sqli , either add another ' which will fix the query and remove the Error , or simply just comment what comes after the ' we injected , this way we can get rid of the error , and if we don't haver the error this confirms the SQLi . 

```sql
category='' 
category='-- -
```

<img width="1366" height="695" alt="image" src="https://github.com/user-attachments/assets/3284f113-7caa-42cd-bf71-961e8ea10227" />

Perfect we no longer have the error , which proves the SQLi .

Now it really doesn't matter what DB is being used (as long as it's not NoSQL OFC) the first thing we should do once we confirm the SQLi , is to get the number of columns specially if we're planning on doing a UINION based SQLi .

Since for the UNION SQLi to work we MUST specify the exact number of columns otherwise we'll get an error . 

Now to get the number or Columns , we have 2 options either use order by , or UNION SELECT with NULL Values :

1/ Using Order By : 

The idea here is simple , we use order by NumberX , then we keep increasing the number until we get an error.

The moment we get an error this means we exceeded the number of columns , so the number is ERROR - 1 if this makes sense . 

```sql
' ORDER BY 1-- -  → works
' ORDER BY 2-- - → works
' ORDER BY 3-- -  → works
' ORDER BY 4-- -  → error! This means 3 is the number . 
```

<img width="1422" height="699" alt="image" src="https://github.com/user-attachments/assets/19de9025-9265-4e77-a892-269ddd83cc12" />

Now we keep increasing the number : 

<img width="1609" height="398" alt="image" src="https://github.com/user-attachments/assets/e07b476f-a4ef-4694-a4cb-d021bb1eff74" />

We get an error at number 4 so the number of columns is 3 . 

2/ Using UNION SELECT with NULL : 

Here it's the opposite: we start with an error (since the column count doesn't match yet), and we keep increasing the number of NULLs in the UNION SELECT until the error disappears. The moment the error disappears, that tells us we've matched the correct number of columns.

```sql
'UNION SELECT NULL,NULL-- -
```

<img width="1341" height="538" alt="image" src="https://github.com/user-attachments/assets/53d93a9f-e372-43a8-be15-bc1572deb6b5" />

2 NULLS should return an error since we already know there are 3 columns , so if we try 3 :

```sql
'UNION SELECT NULL,NULL,NULL-- -
```

<img width="1713" height="767" alt="image" src="https://github.com/user-attachments/assets/b6db9101-6aa2-4e32-9222-a0da0124e19a" />

Works perfectly , from here we can try to identify the DB used , check tables names, column names for those tables,etc . 

But for this lab listing the number of columns is enough .

That was it for this lab , see you in the next one :)

