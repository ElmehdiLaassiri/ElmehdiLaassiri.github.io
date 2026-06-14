---
title: "Webverselabs Canal Cove Books Challenge XSS "
date: 2026-06-14 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


## Scenario : 

Canal Cove Books is a neighborhood used-book shop with a search over 40,000 titles. The owner added "security" — strip script, strip on*= handlers. He reads Hacker News now. The one thing he didn't account for is that not every way to run JavaScript looks like a script tag or an event handler.


## Solution : 

<img width="1414" height="824" alt="image" src="https://github.com/user-attachments/assets/03545bda-4b4b-4bdc-bf2b-d018c1760800" />

We first start by exploring the app and navigating it just like a normal user would, check different endpoints , submit forms , just  discovering normal workflow of the app.

**/Front Page :**

<img width="1230" height="719" alt="image" src="https://github.com/user-attachments/assets/ff0d7dc1-e49e-43ae-bead-3480c81e2ebb" />

This is just a static web page that is returned to us everytime we visit the Dashboard . 

**/Visits :**

<img width="1120" height="837" alt="image" src="https://github.com/user-attachments/assets/d8d18044-f968-4da6-b1f0-15cd4779c461" />

Another static web page, nothing to work with here . 

**/Staff Picked !**

<img width="1272" height="874" alt="image" src="https://github.com/user-attachments/assets/af08b523-8383-43a1-a4e3-57d7299b9667" />

Same thing , nothing useful on it . 

**/Find Request :**

<img width="1224" height="860" alt="image" src="https://github.com/user-attachments/assets/9cd24ccc-2123-485b-a8ff-cc627eef132b" />

This looks more interesting since we are able to submit a form , we might try to inject something into it and hope for the best :) 

But first let's send a normal request and see what we get back : 

<img width="1307" height="800" alt="image" src="https://github.com/user-attachments/assets/b3aaa3c0-c855-4b64-a28a-028960c0bce6" />

We don't really get anything reflected back to us other than the name, for now let's try injecting a simple XSS payload to see : 

```js
<script>alert(window.origin)</script>
```

<img width="1303" height="552" alt="image" src="https://github.com/user-attachments/assets/c015594b-0ac7-418e-a152-9a46009c4db1" />

If we submit it : 

<img width="1300" height="761" alt="image" src="https://github.com/user-attachments/assets/bb909178-e3cc-49a1-a624-74dd4aee0dd7" />

We get nothing back , not even the name, this is normal since most apps will block script tags ( it is also mentionned in the description : "The owner added "security" — strip script" ....)

We can try other payloads that don't contain script : 

```js
<img src="" onerror=alert(window.origin)>
```

<img width="1296" height="806" alt="image" src="https://github.com/user-attachments/assets/95f6c617-7be8-4923-887c-5df8b2905fca" />

The JS code doesn't execute but at least now we can see our comment , Progress :)

<img width="1516" height="542" alt="image" src="https://github.com/user-attachments/assets/35fcfd2d-d90b-4ceb-a4bf-3b580a087aba" />

Looking at Burp, we see that Onerror got filtered as well .

We can try and use a different one , we can use Port Swigger XSS cheat sheet to generate a payload that doesnt contain script nor onerror , we already know img is allowed .  

```bash
https://portswigger.net/web-security/cross-site-scripting/cheat-sheet
```

<img width="1343" height="715" alt="image" src="https://github.com/user-attachments/assets/814b2e8a-ae86-4074-8640-2a1b2896e6c0" />

I tried them but apparently it blocks all events handlers not just onerror .

Now Searching for XSS Filter Bypasses we find Plenty that we can use : 

```js
https://cheatsheetseries.owasp.org/cheatsheets/XSS_Filter_Evasion_Cheat_Sheet.html
```
<img width="860" height="717" alt="image" src="https://github.com/user-attachments/assets/67824443-9b72-421d-9179-677aa4e7c237" />

But the one that worked for me was this one : 

```js
<svg/onload=alert('XSS')>
```

<img width="1219" height="687" alt="image" src="https://github.com/user-attachments/assets/ae402fae-0bce-4cdf-8d12-f1622f61ff51" />

Once we submit it : 

<img width="1082" height="468" alt="image" src="https://github.com/user-attachments/assets/ead6cc20-03b6-4114-882b-bba6a7aefaf0" />

Perfect , now upon triggering the XSS we should get our flag right after : 

<img width="1088" height="534" alt="image" src="https://github.com/user-attachments/assets/f59543a2-8d29-4a7c-86fe-02e2fdd1d8b3" />

That was all for this challenge, see you in the next one :)
