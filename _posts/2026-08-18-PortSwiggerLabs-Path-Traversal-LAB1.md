---
title: " PortSwiggerlabs: File path traversal, simple case "
date: 2026-08-18 00:00:00 +0000
categories: [PortSwiggerLabs , Path Traversal ]
tags: [PortSwiggerLabs , Path Traversal , Challenge , Web_Attacks ]
---


## Scenario : 

This lab contains a path traversal vulnerability in the display of product images.

To solve the lab, retrieve the contents of the /etc/passwd file.


## Solution : 

<img width="1794" height="894" alt="image" src="https://github.com/user-attachments/assets/77974d0d-3a31-460d-b156-d52f4b25bb41" />

This one is pretty straight forward , no sanitization , so it shouldn't be that complicated . 

First we just navigate the application like a normal user , then from there we check Burp's History to see if we can find anything useful . 

A POST Request we can tamper with , a parameter that we can inject whether it be a GET or POST Request , anything that we can use really . 

<img width="1548" height="621" alt="image" src="https://github.com/user-attachments/assets/b1a21906-e996-4b6e-a092-c92ecf1bf77e" />

Looking at Burp's History , we see that we have an id parameter that changes depending on each product we select , this is a Path Traversal labs so the payloads we will use are specific to LFI , but usually whenever we find a parameter , we should test for all type of injections as well , SQLI,XSS,SSTI, ... 

For now looking at HackerReciepe or payload all the things , we can tets some LFI payloads : 

```text
https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion
```

<img width="1421" height="695" alt="image" src="https://github.com/user-attachments/assets/309561ca-8dcf-4c95-8ea9-5d6f8e74a9d4" />

This didn't work , i tried multiple Bypasses but didn't get anything , i assume the ID parameter isn't the vulnerable one here . 

I removed every filter from Burp , to see ALL requests made to the server , and i found this : 

<img width="1757" height="706" alt="image" src="https://github.com/user-attachments/assets/c9835ddb-12f9-424c-9c78-5fc8977403f8" />

The filename is the parameter used to get the images for each Product , let's test this one , since usually the Images might be stored locally , and maybe by changing the parameter value to another internal file , we might be able to read it . 

<img width="1527" height="893" alt="image" src="https://github.com/user-attachments/assets/5b5fcdc8-8920-4f7a-b59b-f5f1f52aa6ce" />

Worked perfectly , as i said before no sanitization nor blacklist filters are in place . 

Now back to our lab , it should be solved . 

<img width="1698" height="665" alt="image" src="https://github.com/user-attachments/assets/f0bda894-af4d-4c71-b120-0f6cbddeaa74" />

That was all for this lab , see you in the next one :) 

