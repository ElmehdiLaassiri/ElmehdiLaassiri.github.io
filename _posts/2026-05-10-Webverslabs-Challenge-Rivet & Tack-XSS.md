---
title: "Webverselabs Challenge Rivet & Tack XSS  "
date: 2026-05-10 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---



## Scenario : 

Rivet & Tack is a two-generation leather shop out of Millerton — belts and dog collars from $48, custom saddle work by appointment, founded 1986. The order-lookup page was bolted on by the owner's nephew, a high-school junior who'd just discovered View Source and was prouder of how the markup read than of what the browser would do with it.


## Solution : 

<img width="1848" height="785" alt="image" src="https://github.com/user-attachments/assets/ca9a0653-f412-4027-b5ca-536ded86a420" />

Clicking on **Story** returns a static page , **Goods** returns the first page , and if we click on **Monogram** : 

<img width="1539" height="729" alt="image" src="https://github.com/user-attachments/assets/886dd961-c724-4317-9ee2-a4c0b66eca06" />

We don't control the Piece parameter since it's already pre written , we can't enter anything ourselves (if you send it then intercept the request and inject inside the Piece parameter that doesn't change anything as well ).

But we are able to enter our Initials : 

<img width="1655" height="851" alt="image" src="https://github.com/user-attachments/assets/73156622-eed2-47f8-9364-7c409e6aec1d" />

We see that we can enter waaay beyond the character limit which is supposed to be 4 as they mentionned . 

Now i started by using a very simple XSS payload : 

```js
<script>alert(window.origin)</script>
```

<img width="1650" height="824" alt="image" src="https://github.com/user-attachments/assets/f27b5c21-cc01-4ef5-92ad-8c69b1819d19" />

But it is just rendered back to us without executing , if we check the front code : 

<img width="1351" height="451" alt="image" src="https://github.com/user-attachments/assets/80da8db5-bb69-4390-9512-b5f55b11422c" />

We see that our brackets are encoded and the payload is just returned back to us . Maybe there is a filtering in place for the script tags , to bypass this we can use HTML event handlers instead once again . 

We can first try this one withou script :

```js
<img src="" onerror=alert(window.origin)>
```

<img width="1648" height="797" alt="image" src="https://github.com/user-attachments/assets/f5677925-7809-4b61-9187-8f5c13ec89c4" />

This didn't work either , let's just try and use other events , maybe it will work : 

```js
\<a onmouseover="alert(document.cookie)"\>xxs link\</a\>
```

<img width="1502" height="937" alt="image" src="https://github.com/user-attachments/assets/ceafda70-69b9-4b3d-ac77-dc5fdbc9ee5a" />

On mouse over works perfectly , upon triggering the XSS, we will get our flag : 

<img width="1368" height="748" alt="image" src="https://github.com/user-attachments/assets/bb68345a-bd5a-4006-8fec-0c80e435bb5a" />

That was all for this chllenge, see you in the next one :)

