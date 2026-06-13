---
title: "Webverselabs Kismet Challenge XSS "
date: 2026-06-11 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Hard , Challenge , Web_Attacks ]
---


## Scenario : 

Kismet is a small-town matchmaking service that still sends real letters. The online bio editor allows a short whitelist of tags for formatting — including <details>, for collapsible sections. The dev knew about markdown. He didn't know about ontoggle.


## Solution : 

<img width="1560" height="839" alt="image" src="https://github.com/user-attachments/assets/82a3cada-607a-4d5c-8326-e6975f429ecf" />

As usual we first need to browse the application just like a normal user, check different endpoints, different requests made to the server then try and find an injectable spot. 

**/Members :**

<img width="1264" height="874" alt="image" src="https://github.com/user-attachments/assets/6b215db1-fd99-445b-a976-6f40aafa1852" />

This is just a static web page that returns a list of members , pretty useless for us . 

**/Letter Guide :**

<img width="1452" height="679" alt="image" src="https://github.com/user-attachments/assets/1a1d404e-040c-4315-8e6f-55b8e5b6a016" />

Again just a manual showing us how to write a letter on the app, just a static web page. 

**/Weekly Matches :**

<img width="1496" height="538" alt="image" src="https://github.com/user-attachments/assets/9b5ab475-fb3b-4e6c-932c-d956381ed38a" />

Another static page showing us different matches for the week . 

**/Letter Editor :**

<img width="1536" height="846" alt="image" src="https://github.com/user-attachments/assets/c3b1857d-a859-4ebb-b604-4785e8da871f" />

Finally somewhere we can inject :)

Now it's already showing us which Formats are allowed : 

```bash
<b>
<i>
<u>
<em>
<strong>
<details>
```

If we try to inject a simple XSS Payload using the script tag for example , this shouldn't work since it is filtered based on what's written : 

<img width="1330" height="809" alt="image" src="https://github.com/user-attachments/assets/6e25f5a0-4c84-4568-b10a-a771dab47be9" />

We see that the script tag are removed automatically . 

Now since we have the details tag available , we can check if there is a way we can use it for an XSS : 

```bash
https://portswigger.net/web-security/cross-site-scripting/cheat-sheet#noembed-consuming-tag
```
Here we can view all the tags that can allow the injection of JS code . 

<img width="1360" height="789" alt="image" src="https://github.com/user-attachments/assets/6975089c-7bb1-4ec0-8c45-683335968234" />

The <details> tag supports event handlers like ontoggle that can be used to execute JavaScript.

We can test these payloads : 

<img width="1486" height="716" alt="image" src="https://github.com/user-attachments/assets/6be87388-c167-4996-85aa-1634b0e0e197" />

Now if we inject it : 

<img width="1303" height="652" alt="image" src="https://github.com/user-attachments/assets/d5e75474-2f78-4f0a-806f-ccd744f6dd5e" />

Perfect this works , now all we have to do is click on the Details section to trigger the JS execution . 

<img width="1141" height="623" alt="image" src="https://github.com/user-attachments/assets/c8bb7be7-c006-499c-bbb9-479db3942f7f" />

From there once the payload is executed and the XSS is triggered we will get our flag :

<img width="1395" height="761" alt="image" src="https://github.com/user-attachments/assets/372e7a8f-5434-4b62-823d-f9ed424b130a" />

That was all for this challenge. See you in the next one . 

