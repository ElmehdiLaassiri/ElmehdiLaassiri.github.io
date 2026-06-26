---
title: " PortSwiggerlabs: Reflected XSS into attribute with angle brackets HTML-encoded "
date: 2026-06-26 00:00:00 +0000
categories: [PortSwiggerLabs , XSS ]
tags: [PortSwiggerLabs , XSS , Challenge , Web_Attacks ]
---



## Information : 

This lab contains a reflected cross-site scripting vulnerability in the search blog functionality where angle brackets are HTML-encoded. To solve this lab, perform a cross-site scripting attack that injects an attribute and calls the alert function.


## Solution : 


Again we're given a search field where we can inject : 

<img width="1716" height="868" alt="image" src="https://github.com/user-attachments/assets/8d368664-28e9-4f28-8913-51dc2d5718d1" />

Before injecting i put a normal string just to see how it is being processed by the app . 

<img width="1550" height="583" alt="image" src="https://github.com/user-attachments/assets/e90f90d5-e423-4774-b732-7f5e2f3ddda3" />

Looking at Burp, we see the value="aaa" that was our input , what if we add another " before the string we just inserted will we break our of the quotations?

```js
"<image onmouseover="alert(1)" style=display:block>test</image>
```
<img width="1567" height="555" alt="image" src="https://github.com/user-attachments/assets/2f0c6953-5665-43a9-bb6a-8439189b6176" />

Perfect we are able to break out of the quotations , now we just inject our XSS payload but add " at the beguinning : 

<img width="1626" height="717" alt="image" src="https://github.com/user-attachments/assets/efffbe38-94d3-445b-9542-fe10f53e55e0" />

Perfect we're able to trigger the XSS after we put our mouse over the field (since we're using the mouse over event handler ) . 

To generate all sort of XSS payloads i used this Portswigger XSS cheat sheet . 

```bash
https://portswigger.net/web-security/cross-site-scripting/cheat-sheet
```

<img width="1592" height="855" alt="image" src="https://github.com/user-attachments/assets/ffd1900d-bb4a-4474-9ec3-90f53e856407" />

We can customize our payload for different scenarios . 

That was all for this lab, see you in the next one :) 

