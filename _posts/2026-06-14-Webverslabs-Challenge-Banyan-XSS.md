---
title: "Webverselabs Banyan Challenge XSS "
date: 2026-06-14 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


## Scenario : 


Banyan coordinates community-garden plots. The search endpoint strips any space-prefixed on* attribute. The developer assumed all attribute boundaries are whitespace. HTML does not agree.


## Solution :


<img width="1338" height="830" alt="image" src="https://github.com/user-attachments/assets/0b032449-48ea-4973-a2f9-f1c5246f670b" />

We first browse the app like a normal user , see different endpoints, features then check Burp's History for interesting Requests that we can tamper with . 


**/Crops :**

<img width="1242" height="870" alt="image" src="https://github.com/user-attachments/assets/1794dddb-5ff1-476c-be53-f90fcfc7beb2" />

Just a static page , no parameters, no useful information for us here. 

**/Workdays :**

<img width="1167" height="641" alt="image" src="https://github.com/user-attachments/assets/d20839f0-e7b6-465c-b5f2-e664c627e949" />

Same thing , nothing we can work with here. 

**/Reserve :**

<img width="1242" height="841" alt="image" src="https://github.com/user-attachments/assets/5a59a5b1-1a44-4f4d-ab79-988ed84bf2ef" />

This one is interesting, let's first submit a normal request : 

<img width="1483" height="736" alt="image" src="https://github.com/user-attachments/assets/980c833e-65ae-4f3f-8e2b-c2b7528c4433" />

Now first thing we can try is to inject a payload inside of the notes parameter , for this i used a simple XSS payload : 

```js
<script>alert(window.origin)</script>
```

<img width="1305" height="750" alt="image" src="https://github.com/user-attachments/assets/e0be284a-6dd1-4981-a467-5c98fb008126" />

Weird "<script" got deleted , if we check the code : 

<img width="739" height="356" alt="image" src="https://github.com/user-attachments/assets/d4d32de1-40fc-40e4-a759-b40a51a11295" />

Maybe script tags are blocked here , we can always use different ones : 

```js
<img src="" onerror=alert(window.origin)>
```

<img width="1332" height="741" alt="image" src="https://github.com/user-attachments/assets/64637c58-b796-4a27-a293-1b1a7fc26f70" />

This time the entire thing got removed , if we check the code : 

<img width="1476" height="414" alt="image" src="https://github.com/user-attachments/assets/7d3f9b38-5d68-4a90-884c-e00602d76640" />

The moment it reached the space it deleted everything that came after, they also mentionned this in the description "The search endpoint strips any space-prefixed".

We can try payloads that don't use spaces or bypass space filters completely , and that won't include script tags as well obviously .

<img width="1334" height="396" alt="image" src="https://github.com/user-attachments/assets/d5d081e4-135d-4a09-9a58-8052ea403cde" />

I found these on Port Swigger's Website , we can test them : 

```js
<img/onerror=alert(1) src=a>
<img/anyjunk/onerror=alert(1) src=a>
<img """><script>alert("alert(1)")</script>">
```

<img width="1329" height="886" alt="image" src="https://github.com/user-attachments/assets/40ad5455-7fc7-4a4d-bb16-8f3189feee0e" />

This worked from the first one .

<img width="1500" height="591" alt="image" src="https://github.com/user-attachments/assets/03c9b56b-5ef6-4832-af0f-225dbe52fc98" />

Now Once the XSS is triggered , we should get our flag . 

<img width="1378" height="672" alt="image" src="https://github.com/user-attachments/assets/e3345c07-9d31-4e27-b446-361d30a40d7c" />

That was all for this challenge. See you in the next one :)


