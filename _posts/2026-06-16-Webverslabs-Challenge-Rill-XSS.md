---
title: " Webverselabs Challenge Rill Reflected XSS  "
date: 2026-06-16 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---



## Scenario : 

Rill is a creek-conservation volunteer network. Their sign-up search has a 24-character cap because the dev heard "shorter queries are safer." There's no other filter.

## Solution : 

We've completed too many XSS challenges so i will just speedrun this one a bit :)

<img width="1664" height="858" alt="image" src="https://github.com/user-attachments/assets/c66b3f5a-ca16-49c7-91da-8e94ec7b2545" />

As usual, we navigate the app, check different endpoints , check Burp's History to try and get a better understanding on how the app behaves. 

**/Watersheds :**

<img width="1448" height="792" alt="image" src="https://github.com/user-attachments/assets/0fc965f2-65a3-4838-8a3f-be912a4175e7" />

Just a static page, nothing useful to us  . 

**/Cleanups :**

<img width="1471" height="814" alt="image" src="https://github.com/user-attachments/assets/3f1efa7d-d6fd-42eb-a9d9-d5f77bf3d645" />

Same thing, nothing we can work with here. 

**/Reports :**

<img width="1190" height="851" alt="image" src="https://github.com/user-attachments/assets/f3d206f6-7bf7-47e7-9556-384e88334051" />

This looks like a chat that we can use to talk to the team : 

<img width="1570" height="748" alt="image" src="https://github.com/user-attachments/assets/06cb5f3f-954d-4017-8930-74916a8695af" />

Looking at Burp , it's just a simple GET request containing our input as parameter , maybe we can try to inject inside of it , specially that our input is rendered back to us. 

I first started with a simple XSS payload : 

```js
<script>alert(window.origin)</script>
```

<img width="1570" height="627" alt="image" src="https://github.com/user-attachments/assets/08bf6c40-9cf7-4979-a2aa-37d2df9a23e5" />

This returns an empty message, if we check the code : 

<img width="1674" height="648" alt="image" src="https://github.com/user-attachments/assets/300ae6e1-eb15-4026-b6bc-dfdd54bd50b5" />

We see that our payloads isn't sent correctly . It looks like the field has a character limit : i tried entering 10 A and 10 B , 10 C and 10 D to verify :

<img width="1164" height="728" alt="image" src="https://github.com/user-attachments/assets/338bf7d2-9e30-4a7d-b250-3cb11b291b64" />

Well , we see that our payload gets truncated after we surpass 24 characters : we can try to use a smaller payload then : 

I found this Blog that talks about Bypassing lenght limits using Spanned payloads : 

```bash
https://portswigger.net/support/xss-filters-beating-length-limits-using-spanned-payloads
```
 But for this to work, we need to be able to input more than just 1 parameter ,since the idea is to spread our payload into smaller sections acress the submition form : 

 <img width="1628" height="839" alt="image" src="https://github.com/user-attachments/assets/213ea0c6-4576-44a6-9f88-5fb90d4c0ad4" />

We also have **/Signup** endpoint : 

<img width="1314" height="742" alt="image" src="https://github.com/user-attachments/assets/a2467c6d-6ecd-4eff-8bb2-5b439c7eb881" />

We can try to split our payload and input it in differnt sections like they did : 

<img width="1030" height="307" alt="image" src="https://github.com/user-attachments/assets/b21587a1-b742-4a14-b1b0-f8c7a8cf0342" />

This didn't work since if we try to send a normal request we get this error which means we can't really use this endpoint. 

Now back to our **Reports** endpoint , let's try to find a smaller payload or another way to bypass character restriction . 

If we look at the code : 

<img width="1324" height="532" alt="image" src="https://github.com/user-attachments/assets/808bc7d6-4bfc-430e-8bdc-30dddb319044" />

We find that 64 is the number of characters allowed, not 24 , maybe the payload didn't work because it had filtered tags , so i decided to not use script tag completely and just use a payload that only has HTML event Handler : 

```js
<svg/onload='+/"`/+/onmouseover=1/+/[*/[]/+alert(42);//'>
```

<img width="1293" height="786" alt="image" src="https://github.com/user-attachments/assets/e5e5c613-f383-4345-8a7c-a545ccc304a3" />

Once we inject it, to test this we need to hover our mouse on that field to fire the XSS : 

<img width="1466" height="757" alt="image" src="https://github.com/user-attachments/assets/5eb45e62-8842-4203-b39d-4aa96106bd04" />

Although we don't get the 42 alert, we are able to get the flag which means our code was executed . Maybe the app surpreses alerts but still our js code was executed .

We can also use different Obfuscation methods , i found these on a Blog by Invicti on Filter Bypasses for XSS . 

```js
<body onload="eval(atob('YWxlcnQoJ1N1Y2Nlc3NmdWwgWFNTJyk='))">
```

<img width="1725" height="846" alt="image" src="https://github.com/user-attachments/assets/819c101a-47ed-42d6-90bb-6fb307c65964" />

This triggers the XSS a well , if you want to try different paylaods, just reset the challenge and retry a different payload until you get your flag :) 

<img width="1471" height="957" alt="image" src="https://github.com/user-attachments/assets/2cc153fc-19fe-45b2-a13a-132c2a00f810" />

For this one i think we should just avoid script tags , and use event handlers instead and all our payloads should work .

That was all for this challenge , see you in the next one :)



