---
title: " PortSwiggerlabs: DOM XSS in document.write sink using source location.search "
date: 2026-06-25 00:00:00 +0000
categories: [PortSwiggerLabs , XSS ]
tags: [PortSwiggerLabs , XSS , Challenge , Web_Attacks ]
---


## Information : 

This lab contains a DOM-based cross-site scripting vulnerability in the search query tracking functionality. It uses the JavaScript document.write function, which writes data out to the page. The document.write function is called with data from location.search, which you can control using the website URL.

To solve this lab, perform a cross-site scripting attack that calls the alert function.


## Solution : 

We have this search field where we can inject : 

<img width="1400" height="691" alt="image" src="https://github.com/user-attachments/assets/07099d04-f7e5-40eb-9fca-a53739eeb544" />

First i injected a simple payload to test with : 

```js
<script>alert(window.origin)</script>
```

<img width="1335" height="635" alt="image" src="https://github.com/user-attachments/assets/4ad4037c-66ad-46fa-8184-21f641752ff5" />

If we check the source code : 

<img width="1211" height="588" alt="image" src="https://github.com/user-attachments/assets/217d41cf-e419-4f9c-bf20-1fa749bafa10" />

We see that first our payload is encoded and treated like a Text , which is returned back to us. So this part is not vulnerable .

We also find a document.write function which takes whatever we inserted in the search field and puts it inside an img src attribute, with no encoding applied.

We can first try to escape the img src tag by adding : "> then adding our payload : 

```js
"><script>alert(window.origin)</script>
```

<img width="1388" height="426" alt="image" src="https://github.com/user-attachments/assets/f8a6ce03-15bc-4f63-bd9a-8133576ec9d1" />

Ah perfect this worked perfectly , but in case script was blocked we can use HTML event handlers with svg onload . 

Back to our lab , once the XSS is triggered we should see that the lab solved . 

<img width="1484" height="700" alt="image" src="https://github.com/user-attachments/assets/1dd33c1f-636d-4621-bf0a-c87cd16e37ce" />

That was all for this lab , see you in the next one :)
