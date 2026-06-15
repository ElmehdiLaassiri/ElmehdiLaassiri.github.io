---
title: " Webverselabs Challenge Chorus Reflected XSS  "
date: 2026-06-15 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


## Scenario : 

Chorus is an indie music review site. The dev added a personal "Hi, <name>!" greeting using an inline script that calls htmlspecialchars — but without ENT_QUOTES. Angles are escaped. Quotes aren't. Guess which one matters inside a JavaScript string literal.


## Solution :

<img width="1860" height="894" alt="image" src="https://github.com/user-attachments/assets/89c9e7df-c34f-4f92-a4d6-c9eedbf38a6f" />

We first start by using the app just like a normal user, discover different endpoints , features, etc. 

**/Artists :**

<img width="1847" height="670" alt="image" src="https://github.com/user-attachments/assets/2727ce3b-6cbf-49f2-ba77-a062c4e2acf1" />

This returns a static page , nothing useful for us . 

**/Reviews :**

<img width="1550" height="830" alt="image" src="https://github.com/user-attachments/assets/a10ff56b-bb44-4cf1-a540-ba0eac25a192" />

Another static page , but this time we can generate a mixstape , we also have a **Sumit** botton that redirects us to this : 

<img width="1638" height="581" alt="image" src="https://github.com/user-attachments/assets/3d2bf88f-4b13-4145-9f3b-3a4110ec2d59" />

Basically nowhere :) 

Let's explore the mixtape generation endpoint : 

<img width="1189" height="567" alt="image" src="https://github.com/user-attachments/assets/874e5c36-a925-44fd-9d16-8ce9778c0146" />

Whatever we type gets reflected back to us : 

<img width="1369" height="661" alt="image" src="https://github.com/user-attachments/assets/e2375b96-824b-4ef0-9754-8c011e332239" />

If we check Burp's Request : 

<img width="1485" height="739" alt="image" src="https://github.com/user-attachments/assets/4b0c904e-269f-4ea3-9ad5-0a6a2c5a5826" />

A simple GET request containing our parameter , now i started by using a basic XSS payload : 

```js
<script>alert(window.origin)</script>
```

<img width="1549" height="706" alt="image" src="https://github.com/user-attachments/assets/0389ec5c-513b-483c-9895-2185a21c95df" />

We see that our tags get encoded by the server, which causes the browser to treat them as plain text instead of HTML.

Whenever we see something like this , we can always try some Filter Bypasses before anything : 

```bash
https://cheatsheetseries.owasp.org/cheatsheets/XSS_Filter_Evasion_Cheat_Sheet.html
```

After trying some of these payloads , this one worked perfectly : 

```js
<svg/onload='+/"`/+/onmouseover=1/+/[*/[]/+alert(42);//'>
--> <svg> is an inline HTML5 element — valid in modern browsers without needing <html> or <body> context
--> / between svg and onload acts as a whitespace substitute — some filters only check for <svg onload with a space
--> onload fires automatically, no user interaction needed (unlike onmouseover) 
```

Few assumptions for why this worked : 

The filter was encoding script tags fine, but it never accounted for svg so that slipped through as raw HTML. The / instead of a space between svg and onload also helped dodge any pattern matching. Once the browser got the unencoded tag, it rendered the SVG and onload fired automatically, no clicks or hovers needed. The weird characters around alert(42) are just there to confuse filters looking for obvious strings like alert(  the browser doesn't care about the mess, it runs it fine anyway.

<img width="1424" height="802" alt="image" src="https://github.com/user-attachments/assets/8c5ab2c4-b887-421a-af29-ccee8e2119f6" />

Once our XSS is triggered, we should get our flag : 

<img width="1269" height="803" alt="image" src="https://github.com/user-attachments/assets/d9f02c85-2895-4bfc-b564-816bb8fcc274" />

That was all for this challenge , see you in the next one :) 

