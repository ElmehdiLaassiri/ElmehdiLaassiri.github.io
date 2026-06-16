---
title: "Webverselabs Challenge Sandpiper Stationery XSS "
date: 2026-06-10 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---


## Scenario : 

Sandpiper Stationery is a three-person wedding-invitation studio on the Cape, founded 2012 — letterpress suites from $640, six-to-eight week lead times, an Etsy following that finally outgrew the Etsy template. Their first standalone site went up in March. The tracking page was a last-minute add the freelancer described as "five lines, can't go wrong" the afternoon before launch.


## Solution : 

<img width="1795" height="888" alt="image" src="https://github.com/user-attachments/assets/2e74123b-1fd7-47dc-833d-19b877681c08" />

This one was pretty straigh forward , not too many endpoints, just the Preview Button : 

<img width="1484" height="796" alt="image" src="https://github.com/user-attachments/assets/907b319a-8046-49ad-8aa8-8c218255145c" />

If we hit send : 

<img width="1463" height="618" alt="image" src="https://github.com/user-attachments/assets/774a917e-c449-4e94-ad6d-a682b5bc1f30" />

It is reflected back to us , the emil field is well secured most of the time , so we will focus on the guest name, i started by a using a simple XSS payload : 

```js
<script>alert(window.origin)</script>
```

<img width="1579" height="722" alt="image" src="https://github.com/user-attachments/assets/f510e1c4-947e-440d-86bf-56aa84628c1a" />

Well this gets reflected back to us , let's read the front end code : 

<img width="1190" height="410" alt="image" src="https://github.com/user-attachments/assets/e85a54ff-15c5-41a8-8040-c612f6bbad2c" />

We see that our payload was encoded and treated like Text rather than part of HTML code . 

We could use a Filter Bypass which we're sure it will work : 

```js
https://cheatsheetseries.owasp.org/cheatsheets/XSS_Filter_Evasion_Cheat_Sheet.html
```

But here let's try simpe Bypasses , like using HTML event handlers instead of script tags : 

<img width="1485" height="743" alt="image" src="https://github.com/user-attachments/assets/9a877a7b-ea05-4971-ae04-aecf7e5b5d33" />

Works perfectly :)

Once the XSS is triggered we should get the flag : 

<img width="1469" height="750" alt="image" src="https://github.com/user-attachments/assets/cd0a2885-467a-4179-ab3e-ca7dfa54ac06" />

That was all for this challenge, see you in the next on :)
