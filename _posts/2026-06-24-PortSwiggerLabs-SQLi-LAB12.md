---
title: " PortSwiggerlabs: Blind SQL injection with time delays "
date: 2026-06-24 12:00:00 +0000
categories: [PortSwiggerLabs , SQL Injection]
tags: [PortSwiggerLabs , SQLi , Challenge , Web_Attacks ]
---

## Information : 

This lab contains a blind SQL injection vulnerability. The application uses a tracking cookie for analytics, and performs a SQL query containing the value of the submitted cookie.

The results of the SQL query are not returned, and the application does not respond any differently based on whether the query returns any rows or causes an error. However, since the query is executed synchronously, it is possible to trigger conditional time delays to infer information.

To solve the lab, exploit the SQL injection vulnerability to cause a 10 second delay.


## Solution :

<img width="1594" height="629" alt="image" src="https://github.com/user-attachments/assets/074595ef-382e-4b56-b5a0-14a9f305fd49" />

I tried injecting both parameters but this didn't work.

<img width="1549" height="743" alt="image" src="https://github.com/user-attachments/assets/d57d700d-dbcc-4e22-9942-f2d710c21abd" />

So i moved to the TrackingID parameter inside the cookie , but again couldn't see any difference , so we mus test for Blind SQLi .

<img width="1258" height="597" alt="image" src="https://github.com/user-attachments/assets/255652c3-ddbf-4be8-87aa-781d93201a45" />

Since this is a blind one , we can't really know what DB is being used so we have to test all of them : 

Eventually this is the paylaod that caused the delay : 

```sql
||pg_sleep(10)-- : This proves the DB is PostgreSQL . 
```

<img width="1657" height="762" alt="image" src="https://github.com/user-attachments/assets/39c76087-dae8-48a1-8dda-38f64a9c871c" />

We can even see that there was a huge delay after we ran the command : 

<img width="1916" height="866" alt="image" src="https://github.com/user-attachments/assets/332e8e33-7ddd-4408-b552-3b803f003e1a" />

The moment we cause the delay : 

<img width="1787" height="803" alt="image" src="https://github.com/user-attachments/assets/f98785db-e6cb-49ef-bff0-30c98f2c8715" />

We should see that the lab was solved . 

That was all for this lab , see you in the next one :) 
