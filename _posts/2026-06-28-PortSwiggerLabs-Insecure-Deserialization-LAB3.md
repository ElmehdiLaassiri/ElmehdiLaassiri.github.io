---
title: " PortSwiggerlabs: Using application functionality to exploit insecure deserialization"
date: 2026-06-28 00:00:00 +0000
categories: [PortSwiggerLabs , Insecure Deserialization ]
tags: [PortSwiggerLabs , Insecure Deserialization , Challenge , Web_Attacks ]
---



## Information : 

This lab uses a serialization-based session mechanism. A certain feature invokes a dangerous method on data provided in a serialized object. To solve the lab, edit the serialized object in the session cookie and use it to delete the morale.txt file from Carlos's home directory.

You can log in to your own account using the following credentials: wiener:peter

You also have access to a backup account: gregg:rosebud


## Solution : 

First we login to the app as the wiener user : 

<img width="1744" height="878" alt="image" src="https://github.com/user-attachments/assets/4377f656-3a8f-44cc-8186-1442d4e984b5" />

We see that we have the ablity to delete our user . Let's try deleting our user then check Burp :

<img width="1898" height="1001" alt="image" src="https://github.com/user-attachments/assets/082c75ed-5751-4ae0-af79-12d8cee3524f" />

If we had Burp pro we will get a flag that there is a serializable object inside the request , but in our case we will just use Inspector and from there we can see that the cookie we get is a PHP serializable object : 

```php
O:4:"User":3:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"pxzxy52y06syyfesirzosl3q041a6wy7";s:11:"avatar_link";s:19:"users/wiener/avatar";}
```

Looking at the cookie , we see that the avatar link is actually tied to our home directory , we need to delete the file morale.txt file from carlos's directory .

Let's modify the cookie : 

```php
O:4:"User":3:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"pxzxy52y06syyfesirzosl3q041a6wy7";s:11:"avatar_link";s:23:"users/carlos/morale.txt";}
```

Notice that users/carlos/morale.txt has 23 characters so we must change the s:19 to 23 otherwise it'll break : 

Now first we login as the backup acc , Greg : 

<img width="1545" height="766" alt="image" src="https://github.com/user-attachments/assets/be610341-7836-4cfc-8032-7ceced43c582" />

Then from there we click on delete and intercept the request : 

<img width="1884" height="920" alt="image" src="https://github.com/user-attachments/assets/52cb10f0-a2bc-48cb-8503-60d6eb08c004" />

Once we do that , we should forwad the requests and from there we will see that we solved the lab : 

<img width="1676" height="911" alt="image" src="https://github.com/user-attachments/assets/b2fdff42-82c7-493f-a792-70663d47aa6e" />

That was all for this lab , see you in the next one :)



