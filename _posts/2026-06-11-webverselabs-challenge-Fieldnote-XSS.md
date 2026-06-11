---
title: "Webverselabs Fieldnote Challenge XSS "
date: 2026-06-11 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Hard , Challenge , Web_Attacks ]
---


## Scenario : 

Fieldnote is a journaling app for field biologists. Colleagues paste links to related papers or references and share them via a /share?u= URL. The filter is one line — if the URL doesn't contain 'http', replace it with a safe default. The filter author didn't think about what 'contains' actually means.

## Solution : 

First we start by navigating the webapp just like a normal user. 

<img width="1655" height="861" alt="image" src="https://github.com/user-attachments/assets/c99f9c55-0424-4d31-ae69-4067fa44ee2f" />

We first explore the different endpoints : 

**/Journal :**

<img width="1549" height="866" alt="image" src="https://github.com/user-attachments/assets/15541159-8436-42f2-b624-7b3b75c94f49" />

Nothing useful here, just a static web page . 

**/Species :**

<img width="1662" height="779" alt="image" src="https://github.com/user-attachments/assets/3528e284-14c0-427f-abf4-bd562f6a8392" />

Same thing here as well . Nothing useful really. 

**/Cite :**

<img width="1654" height="811" alt="image" src="https://github.com/user-attachments/assets/006d8f45-13a0-486f-a7af-6f336b638dc7" />

Here we see that we can actually Reference a URL , I would start by testing for a SSRF but since this challenge is for XSS , i will save our time and go for XSS directly :) 

**/Journal :** 

Before we dig deeper in the /Cite endpoint i wanted to finish exploring the app , found this /Journal endpoint but this wasn't helpful either .

<img width="1682" height="808" alt="image" src="https://github.com/user-attachments/assets/eb0f44f6-ad80-47e1-9cdf-5a1c4d1a756b" />

Now back to our **/Cite** endpoint . I first sent a URL then checked Burp's History to get a better idea of what's going on.

We see that it is sent as a GET Request via the url parameter :

<img width="1619" height="677" alt="image" src="https://github.com/user-attachments/assets/887ecdec-5ea3-43c3-8b30-9c17d22b8346" />

For us to add a reference url the input MUST contain http otherwise it just gets ignored by the server :  

<img width="1652" height="695" alt="image" src="https://github.com/user-attachments/assets/64fbe91e-7c09-4aa9-b914-548140b8b7f9" />

Now i figured i use a payload that contains HTTP inside of it, and try to call our WebHook URL, and hopefully it works . 

<img width="1625" height="826" alt="image" src="https://github.com/user-attachments/assets/449d10e2-6e9d-4ac7-8911-eaf059fee08d" />

But checking our webhook site , we don't see any connections being made -_-  

<img width="1283" height="633" alt="image" src="https://github.com/user-attachments/assets/0574dd9f-89b1-47df-8b4d-70b1434e7240" />

If we check the Front end code : 

<img width="1377" height="703" alt="image" src="https://github.com/user-attachments/assets/5c84ab39-0737-44a1-a398-851aa8f1ed3d" />

We see that our payload is put inside an href , which means it won't be executed -_- cause script tags don't usually execute inside an href , BUT we can run Javascript directly here , using a payload like windows origin , as long as we got HTTP inside of our payload , it should satisfy the filter .

```javascript
javascript:alert(window.origin)//http : HTTP here is treated as a comment (//) .
//javascript: Inside the href is a valid link . so it will run if we click it . 
```

//http satisfies the filter since it has HTTP while it gets ignored by the JavaScript interpreter. 

<img width="1157" height="304" alt="image" src="https://github.com/user-attachments/assets/158b712c-cea5-483a-a38d-07745e530d2e" />

Now if we click on it , we should trigger the XSS. 

<img width="1607" height="803" alt="image" src="https://github.com/user-attachments/assets/000f8a10-8ae2-4157-929f-69087d125d1d" />

And we do : 

<img width="1486" height="634" alt="image" src="https://github.com/user-attachments/assets/62fecdb6-d6e2-402d-aacf-f33689ee9955" />

And just like that we are able to get our flag : 

<img width="1187" height="343" alt="image" src="https://github.com/user-attachments/assets/5d2d2fca-d1f3-42b7-9737-c31d2a424dab" />

That was all for this challenge , see you in the next one :)
