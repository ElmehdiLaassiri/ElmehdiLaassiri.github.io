---
title: " PortSwiggerlabs: DOM XSS in innerHTML sink using source location.search "
date: 2026-06-25 00:00:00 +0000
categories: [PortSwiggerLabs , XSS ]
tags: [PortSwiggerLabs , XSS , Challenge , Web_Attacks ]
---



## Information : 

This lab contains a DOM-based cross-site scripting vulnerability in the search blog functionality. It uses an innerHTML assignment, which changes the HTML contents of a div element, using data from location.search.

To solve this lab, perform a cross-site scripting attack that calls the alert function.


## Solution : 

Quick note : 

For DOM-based XSS, usually we're looking for two things: a sink, which renders or executes inserted data , document.write(), innerHTML, eval(), etc. But this by itself is not enough , we need the data being passed to the sink to be controlled by us, for example via window.location (parameters entered in the URL), document.cookie (injecting the cookie), etc.

The cheat sheet and links already have payloads that work for DOM-based XSS, and often the **same exact payload string** can trigger Reflected, Stored, or DOM-based XSS it just depends on **where and how the application inserts your input into the page**, not on the payload itself. But it's good to know the difference.

Now back to the lab : 

<img width="1343" height="669" alt="image" src="https://github.com/user-attachments/assets/3e638cee-5cf3-4643-8f98-651ab5a234de" />

First let's inject a simple XSS payload : 

```js
<script>alert(window.origin)</script>
```

<img width="1381" height="710" alt="image" src="https://github.com/user-attachments/assets/2c4c0c27-140f-46c6-80a6-986619390f83" />

We see that our payload gets immediately filtered , maybe Tags are blocked once again . 

Checking the Source code : 

<img width="1169" height="531" alt="image" src="https://github.com/user-attachments/assets/423bc4c5-d1af-4855-afc7-ccfa3db392e8" />

Huge indicator of a DOM based as we explained earlier .The sink here is .innerHTML, assigned inside doSearchQuery, and we control the input being passed into it, since it comes directly from the search field via (new URLSearchParams(window.location.search)).get('search'). getElementById('searchMessage') is just used to locate the target element that receives this unsafe assignment.

Now let's use a payload that doesn't use script tags : 

```js
<img src="" onerror=alert(window.origin)>
```

<img width="1340" height="678" alt="image" src="https://github.com/user-attachments/assets/76c10c21-34fb-47a1-803b-ec0b3e154a8e" />

Perfect this executes the payload and we are able to solve the Lab . 

That was all for this lab , see you in the next one :)
