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

<img width="1406" height="877" alt="image" src="https://github.com/user-attachments/assets/c1b87bcf-85a8-4871-a7f7-e7752507d1dd" />

First i injected a simple payload to see how the server will handle it : 

```js
<script>alert(window.origin)</script>
```

<img width="1480" height="590" alt="image" src="https://github.com/user-attachments/assets/d2ddf2c6-ed80-40fe-b017-d06d8f8d878b" />

We see that our payload gets immediately blocked . If we check the code : 

<img width="1242" height="461" alt="image" src="https://github.com/user-attachments/assets/ac91b039-61b9-43f3-b074-2721635c5c5f" />

We see that we can't even find our payload , maybe the script tag is blocked . 

For this we can use other payloads that don't include script tags .

```js
<img src="" onerror=alert(window.origin)>
```

<img width="1306" height="670" alt="image" src="https://github.com/user-attachments/assets/24c41de1-db38-4ceb-a335-3387a6e14983" />

Perfect , we got our paylaod to execute . 

That was all for this lab , see you in the next one :)
