---
title: " PortSwiggerlabs: Reflected XSS into HTML context with nothing encoded "
date: 2026-06-25 00:00:00 +0000
categories: [PortSwiggerLabs , XSS ]
tags: [PortSwiggerLabs , XSS , Challenge , Web_Attacks ]
---


## Information : 

This lab contains a simple reflected cross-site scripting vulnerability in the search functionality.

To solve the lab, perform a cross-site scripting attack that calls the alert function.

## Solution : 

This one is pretty straight forward and pretty basic as well. 

We just need to find an injectable spot . 

<img width="1357" height="669" alt="image" src="https://github.com/user-attachments/assets/172b5a8d-5d34-4150-ab3f-97a937e79ae5" />

In our case it's the search feature .

A simple payload here will be enough to trigger the XSS .

```js
<script>alert(window.origin)</script>
```

<img width="1419" height="444" alt="image" src="https://github.com/user-attachments/assets/52be172e-deb9-4e68-8b50-e88c511fbc24" />

Perfect , once the XSS is triggered the lab should be solved .

<img width="1453" height="639" alt="image" src="https://github.com/user-attachments/assets/b86805c3-1429-4cef-9b51-bebcf9274bb2" />

Few things to note , since this is just a Reflected XSS , once we refresh the page , our payload will be gone and we need to inject the field once again . 

So the only way a victim can be compromised is by visiting the link where we injected the XSS before , whereas if this was a stored XSS , once our payload is stored it will affect Every user who visits the application without the victim having to click our specific link or anything like that which makes Stored XSS much worse compared to a Reflected one . 

That was it for this lab , see you in the next one :) 

