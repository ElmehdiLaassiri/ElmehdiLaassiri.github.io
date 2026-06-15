---
title: " Webverselabs Challenge Porchlight Reflected XSS  "
date: 2026-06-15 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---



## Scenario : 

Porchlight lends power tools to 300 neighborhood members. Their developer knows you're supposed to html-escape user input before putting it in a page. She did. The reflection happens to land inside an unquoted attribute — a context where html-escaping angles doesn't buy you anything.


## Solution : 


<img width="1557" height="847" alt="image" src="https://github.com/user-attachments/assets/55c0bc02-48a5-4929-a3d8-280bb4c01da8" />

This one is pretty straight forward, only 1 endpoint , if we click on Reserve : 

<img width="977" height="820" alt="image" src="https://github.com/user-attachments/assets/e0a2b4c7-a589-424d-898f-6fe66d3732b9" />

First let's reserve normally to see if we get anything unsual in Burp's History .

<img width="1401" height="677" alt="image" src="https://github.com/user-attachments/assets/43908489-444f-4787-bd1b-5e52caecaeba" />

All the parameters are returned back to us . We first inject the member parameter to see . 

```js
<script>alert(window.origin)</script>
```

<img width="1446" height="669" alt="image" src="https://github.com/user-attachments/assets/ae3bf560-97fe-462d-ba0e-68a1dbb704d5" />

Our payload was just reflected back to us without executing . Few assumptions, maybe it is being rendred back to us an HTML comment, script tags are blocked or just the parameter is not vulnerable . 

<img width="1416" height="688" alt="image" src="https://github.com/user-attachments/assets/1ed9b781-461c-46f0-ae0a-9eae99cc4328" />

Decided to Inject the other parameter as well but didn't work either , let's check the code : 

<img width="1639" height="637" alt="image" src="https://github.com/user-attachments/assets/4e45e975-fdb6-4425-bf5b-da1b1264f59c" />

The input is encoded and rendered back to us so the server is treating it like Text . Anyways we can try some Filter Bypasses to see if it works . 

```bash
https://cheatsheetseries.owasp.org/cheatsheets/XSS_Filter_Evasion_Cheat_Sheet.html
```

The value attribute is unquoted, allowing attribute injection via whitespace without requiring angle brackets. HTML entity encoding of <> is insufficient here , injecting x onmouseover=alert(window.origin) breaks out of the attribute and executes arbitrary JavaScript when a user hovers the field. This constitutes a reflected XSS via attribute injection.

<img width="1426" height="929" alt="image" src="https://github.com/user-attachments/assets/e0383096-99fc-4d48-b6dd-bae34382e0ae" />

If we check the code : 

<img width="1574" height="610" alt="image" src="https://github.com/user-attachments/assets/98e271d9-21e7-4905-a574-e81f40c38d91" />

The browser reads onmouseover=alert(1) as a real HTML attribute it becomes live JavaScript.

Also there was this payload that worked for me as well from the Filter Bypass cheatsheet . 

```js
 <a href="jav   ascript:alert('XSS');">Click Me</a>
```

<img width="1187" height="814" alt="image" src="https://github.com/user-attachments/assets/c7cd5e1b-b3d1-411f-8c79-3dc6fe3660f0" />

This shows that HTML encoding of brackets is not sufficient when user input is reflected inside an existing unquoted HTML attribute. In this context, event handler injection via whitespace (e.g. onmouseover=alert(1)) bypasses encoding entirely, since no angle brackets are required.

That was all for this challenge, see you in the next one :)
