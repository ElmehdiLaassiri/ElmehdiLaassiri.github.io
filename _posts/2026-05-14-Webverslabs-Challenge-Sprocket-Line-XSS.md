---
title: " Webverselabs Challenge Sprocket-Line- Reflected XSS  "
date: 2026-05-14 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---


## Scenario : 

After a phishing near-miss, Sprocket Line's owner hired someone to "add security." He got one line of PHP that strips <script> from input — and a two-page invoice. The owner feels much better now.

## Solution : 

<img width="1613" height="879" alt="image" src="https://github.com/user-attachments/assets/750c34c2-0ed3-479c-9f8e-3bdf9acd5407" />

We first start by navigating the application just like a normal user, check different endpoints and features, we're mainly looking for a spot to inject . 

**/Parts :**

<img width="1642" height="885" alt="image" src="https://github.com/user-attachments/assets/e8f30892-de97-4414-9ce7-e1d89515cea8" />

Just a static page showing us different products, nothing useful . 

**/Warranty :**

<img width="1604" height="744" alt="image" src="https://github.com/user-attachments/assets/cc859f31-b6d7-41c1-aa28-a04a1e3898ae" />

Another static page , nothing useful for us :) 

**/Fit Calc :**

<img width="1659" height="871" alt="image" src="https://github.com/user-attachments/assets/fa3e5063-6f0b-4b33-8e31-5594cb3f83de" />

Finally somewhere we can inject , first let's send a normal request and check burp . 

<img width="1551" height="709" alt="image" src="https://github.com/user-attachments/assets/69b070ad-b071-477d-830b-d62d94a9f2ed" />

Just a simple GET request containing our parameter , and returning whatever we inserted back to us , let's first try to inject a simple XSS payload : 

```js
<script>alert(window.origin)</script>
```

<img width="1523" height="703" alt="image" src="https://github.com/user-attachments/assets/f12c5688-c8e5-47da-8647-b71a9d9fe664" />

We see that our tags are removed automatically , and we only get the content inside of them reflected back to us . Maybe there is a filter for script tag, whenever we suspect that there is a filter for specific tags , we can try and use this list of Bypasses : 

```bash
https://cheatsheetseries.owasp.org/cheatsheets/XSS_Filter_Evasion_Cheat_Sheet.html
```

For this one , any of the Bypasses should work since this is a pretty simple one : 

<img width="937" height="789" alt="image" src="https://github.com/user-attachments/assets/f8f89262-05b9-4da2-bfdb-ea4415005fea" />

I choosed this one : 

```js
<a href="jav&#x0A;ascript:alert('XSS');">Click Me</a>
URL Encode it :
<a+href%3d"jav+++ascript%3aalert('XSS')%3b">Click+Me</a>
```

<img width="1509" height="720" alt="image" src="https://github.com/user-attachments/assets/fe595978-27ce-49bb-935f-ec8f794898d0" />

Make sure you URL encode everything since the field doesn't allow spaces . 

If you can't insert the payload directly : 

<img width="1503" height="608" alt="image" src="https://github.com/user-attachments/assets/9959b5e2-11c0-4896-b3a0-9ff3f89facdc" />

Just insert anything and then intercept the request :

<img width="1346" height="659" alt="image" src="https://github.com/user-attachments/assets/c54d9686-2241-4aeb-bec8-b1959aadae12" />

From there modify the request before forwarding it to the server : 

<img width="1437" height="699" alt="image" src="https://github.com/user-attachments/assets/1d2844ca-0f86-4110-b25d-125ec9a2fba9" />

Now if we go back to the app : 

<img width="1449" height="664" alt="image" src="https://github.com/user-attachments/assets/a121fcf5-02cf-4b1d-8ff4-6d16b2cabd55" />

We see that our code was indeed executed , now if we click the "Click Me " Botton , we fire our XSS . 

<img width="1777" height="820" alt="image" src="https://github.com/user-attachments/assets/2164a6cf-a572-4a9f-9e8a-d42ab8e557ff" />

Once the XSS is triggered we should get our flag : 

<img width="1651" height="792" alt="image" src="https://github.com/user-attachments/assets/c0c2871f-5b9a-4049-a4fa-792fab5cbeeb" />

That was all for this challenge. See you in the next one :) 

