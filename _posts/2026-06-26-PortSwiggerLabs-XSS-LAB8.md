---
title: " PortSwiggerlabs: Stored XSS into anchor href attribute with double quotes HTML-encoded "
date: 2026-06-26 00:00:00 +0000
categories: [PortSwiggerLabs , XSS ]
tags: [PortSwiggerLabs , XSS , Challenge , Web_Attacks ]
---



## Information : 

This lab contains a stored cross-site scripting vulnerability in the comment functionality. To solve this lab, submit a comment that calls the alert function when the comment author name is clicked.


## Solution : 

First we submit a normal comment to see how the app behaves : 

<img width="1451" height="639" alt="image" src="https://github.com/user-attachments/assets/f7a979dd-dce8-4107-b93f-1af7f144b668" />

Once we submit it we're redirected to this page :

<img width="1444" height="511" alt="image" src="https://github.com/user-attachments/assets/437554ec-0878-4cbc-8613-dbc1a06e1f0a" />

If we go back to check all the comments : 

<img width="1553" height="571" alt="image" src="https://github.com/user-attachments/assets/cff6b503-4157-4ecc-97ec-902b1bdc3717" />

Everything looks normal , but when we specify a website inside the comment : 

<img width="1169" height="661" alt="image" src="https://github.com/user-attachments/assets/f116f6a6-2c58-4036-af81-d2d52eb43a96" />

Now if we check the code : 

<img width="1730" height="592" alt="image" src="https://github.com/user-attachments/assets/751aa6e5-f6e0-4667-bc3b-a8adf0c029a2" />

We see that our website is directly injected inside the href , while href is mainly used for redirections , we can also specify a different URL scheme other than HTTP,HTTPS,ftp://,etc... , which is javascript:

So our payload will simply be : 

```js
javascript:alert(XSSS)
```

<img width="1419" height="652" alt="image" src="https://github.com/user-attachments/assets/98cf040e-8d62-4de4-829c-3c822771b823" />

Now if we follow the redirection , and then go back to the post : 

<img width="1504" height="600" alt="image" src="https://github.com/user-attachments/assets/06e2ab98-a453-4bc9-b699-3c7637b5ed32" />

As soon as we submit our payload : 

We will see that the lab is solved . 

<img width="1504" height="600" alt="image" src="https://github.com/user-attachments/assets/65febdcd-5167-4c97-b85e-407c0889c165" />

This happened bcs developpers allowed us to inject arbitrary links to the app , which can usually be abused to execute JS code as well directly . 

That was all for this lab , see you in the next one :)
