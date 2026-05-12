---
title: "Webverselabs Ember-Kettle Challlenge Reflected-XSS "
date: 2026-05-12 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---

## Summary : 

In this challenge we are presented with a web application that we first 
enumerate as a normal user, mapping out its endpoints and functionality. 
During enumeration we identify two interesting endpoints — a Newsletter 
endpoint (POST) and a brew endpoint (GET) with a controllable parameter.

After testing the brew endpoint for SQLi and Command Injection with no 
success, XSS payloads proved successful, confirming a Reflected XSS 
vulnerability. We demonstrate multiple exploitation scenarios including 
basic alert execution, cookie extraction, and Out-of-Band XSS using 
Webhook.site to simulate blind XSS conditions.

Finally we automate the injection process using XSStrike, which generates 
an event-based payload using the ontoggle handler, further confirming 
the vulnerability.


## Enumeration :

First thing first , we start by navigating the application like a normal user , see the different endpoints , features and all , before we start Fuzzing the webapp , i like doing so then going back to Brup's HTTP History to see if we find an endpoint that we can dig deeper in . 

I found 2 interesting endpoints the first one is the Newsletter endpoint , which is a POST request , but it checks our email , and usually email fields are well secured , so i will be testing it last , maybe a blind XSS or OOB SQLi , you never know , but first let's see if there are other endpoints . 

<img width="1059" height="730" alt="image" src="https://github.com/user-attachments/assets/273ebd27-6ebe-4b56-b886-ebaeda53e3cb" />

And we do find this brew endpoint , which is a GET request with a parameter field that we can control , first thing we should always try is injecting the parameter field , i started with SQLi , Command injection payloads but i didn't get anything , but when we tried XSS payloads we did get an alert which means we do have an XSS . 
To check for the XSS i tried this basic payload : 

```bash
<script>alert(window.origin)</script>
```

<img width="1372" height="776" alt="image" src="https://github.com/user-attachments/assets/59fea889-4cbb-4a51-b00f-d8b6d44b128d" />

Great it worked , We can also use other payloads , to steal cookie for example :  

```bash
#"><img src=/ onerror=alert(document.cookie)> : For cookies extraction.
```
<img width="1240" height="507" alt="image" src="https://github.com/user-attachments/assets/aa1f87c8-2363-4861-9770-5d81e59a1f12" />

If we wanted to test for an Out Of band XSS , for example the webapp doesn't return anything (for example we had to send a form or something like that ) , we can either set up a server locally and call it using a payload that will go fetch external resources , or we could just use Webhook site , which gives us a unique URL that we can call to see if we get a Response . Just visit https://webhook.site/ , and we should get our unique URL : 

<img width="1031" height="810" alt="image" src="https://github.com/user-attachments/assets/385384f2-904c-498e-8f0b-6bf44d009e62" />

Now we use a payload that will go fetch our URL : 

```bash
<script src="https://webhook.site/1aaef16c-4d7c-4ad6-bd9c-a90f3a9d1091"></script>
 ```
Now once we execute this payload , we should be able to get a Request to our WebHook site . 

<img width="1738" height="655" alt="image" src="https://github.com/user-attachments/assets/74b8b206-673a-400a-8801-7f71af50732f" />

Just like that we can test Blind XSS without any issues . 

Now if we wanted to automate the injections , we could use a tool like XSStrike : 

```bash
git clone https://github.com/s0md3v/XSStrike.git 
cd XSStrike 
pip install -r requirements.txt 
python3 xsstrike.py 
python3 xsstrike.py -u "https://9b3764aa-4327-ember-kettle-ed8a5.challenges.webverselabs-pro.com/brew?mood=aaa"
```
<img width="1405" height="710" alt="image" src="https://github.com/user-attachments/assets/4b0db74d-07fc-4e44-819f-69afff5c0c0a" />

XSStrike generated an event-based payload using the ontoggle handler , this requires manual interaction (mouse click) to execute the payload , as opposed to auto-executing payloads like onerror .

<img width="988" height="623" alt="image" src="https://github.com/user-attachments/assets/dbb8429f-9e31-451e-8577-2bf88d1bf743" />

```html
<dEtailS/+/OnTOgGLe%09=%09confirm()>
```

The <details> section is rendered in the page because our payload was successfully injected and executed — clicking on it triggers the confirmation popup, confirming the XSS vulnerability.

<img width="1056" height="771" alt="image" src="https://github.com/user-attachments/assets/0e7d8db2-7924-47c1-894d-01fa0cb705be" />

Finally we get our popup once we click on details . 

Very fun challenge from Webverselabs , a great playground to freely test and explore XSS techniques


