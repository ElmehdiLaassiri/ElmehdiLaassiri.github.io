---
title: " PortSwiggerlabs: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data "
date: 2026-06-23 00:00:00 +0000
categories: [PortSwiggerLabs , SQL Injection]
tags: [PortSwiggerLabs , SQLi , Challenge , Web_Attacks ]
---


## Information : 

This lab contains a SQL injection vulnerability in the product category filter. When the user selects a category, the application carries out a SQL query like the following:

To solve the lab, perform a SQL injection attack that causes the application to display one or more unreleased products.

## Solution : 

<img width="1783" height="882" alt="image" src="https://github.com/user-attachments/assets/4385a5e5-f257-43df-9176-c18472f0f812" />

We first navigate the application just like a normal user. We're mainly looking for parameters to inject in : 

<img width="1784" height="969" alt="image" src="https://github.com/user-attachments/assets/b617a2ee-3e98-4553-b49f-d28cdefbb054" />

If we visit any of the products we find the product id paramter that we can try to inject , i first injected a simple payload to see if we can cause an internal error :

<img width="1301" height="360" alt="image" src="https://github.com/user-attachments/assets/4b6dfe61-3339-4ab4-9d2d-873cbc2fff03" />

We only get Invalid Product ID , this is not the internal server error that we're looking for, i added another ' to see if it will fix the query and remove the error but that didn't work as well.

Moving on, we find another parameter when we try listing the categories :

<img width="1793" height="945" alt="image" src="https://github.com/user-attachments/assets/47a5b37a-d4f6-4bfa-aecf-d9304cfd86f2" />

Injecting a simple payload did return an internal server error : 

<img width="1534" height="646" alt="image" src="https://github.com/user-attachments/assets/f4c0e694-a7e0-4bdb-88a3-60169de1d4a8" />

To confirm it , either add another ' to fix the query or just comment the rest, if any of these 2 removes the error this is an indicator that we got an SQLi . 

<img width="1732" height="931" alt="image" src="https://github.com/user-attachments/assets/ff0d1593-a512-4e32-a46d-5975b9aff30d" />

This does remove the error , which confirms the Vulnerability . 

Now to exploit this , we see that we got 4 products returned to us when we select the Food&Drink category, we can use this SQLi to list all products from this table. 

We can do that by simply using a payload that returns a True statement , for example 1=1 . What this will do is , either return products from the Food&Drinks Category or 1=1  which is always true and it should list all products that way. 

In other words : 

Since 1=1 is always true, the WHERE clause condition is bypassed entirely, returning all rows from the products table , including unreleased items and items from other categories.

<img width="1718" height="949" alt="image" src="https://github.com/user-attachments/assets/797b248b-c272-4854-92c5-465b6dac7493" />

```sql
'OR 1=1-- - : Make sure you comment the rest of the query to not have an internal error by breaking the query.
```
Perfect, we get a list of all products even unreleased ones. 

That was all for this lab, see you in the next one :)

