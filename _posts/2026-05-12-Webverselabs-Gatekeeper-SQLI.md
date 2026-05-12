---
title: "Webverselabs Gatekeeper Challenge SQLi "
date: 2026-05-12 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---

## Summary : 

Gatekeeper Corp presents a web application with a login portal vulnerable to **SQL Injection Authentication Bypass**. The login form fails to properly sanitize  user input, allowing an attacker to manipulate the underlying SQL query and  gain unauthorized access without valid credentials.

## Solution : 

If we visit the webapp , we find a web page that explains what Gatekeeper corp is , We do find 2 endpoints : Directory and a Login page .
The directory endpoint has multiple email adresses , we will keep this in mind as we might be using them to brute force our way in , if the web app doesn't implement any rate limiting . 

<img width="1282" height="753" alt="image" src="https://github.com/user-attachments/assets/4634b29f-801c-4a62-acb5-8c551274aa52" />

Now if we try to login , all credentials return the same response  , "Invalid Creds" so we can't really enumerate usernames this way . 

<img width="879" height="671" alt="image" src="https://github.com/user-attachments/assets/8f81940d-0352-495f-b6ce-a3425844ab50" />

Tried Fuzzing for other endpoints but i didnt find anything useful . 

<img width="1584" height="741" alt="image" src="https://github.com/user-attachments/assets/a75f3811-7bd2-4906-9ce7-49a5f9075ff7" />

Now first thing i would think about when i see a Login page , is Auth Bypass via SQLi , i already have a section of payloads in my methodology that i can test . 

```bash
'==> Auth Bypass :  

'OR 1 = 1 --
Adminstrator'OR 1 = 1 --
Admin'OR 1 = 1 -- 
Admin ' or '1' = '1 #  
admin')-- -
Admin')-- -

```
Now let's first take a look at how the request looks like to try and use FFUF to brute force . 


<img width="1286" height="676" alt="image" src="https://github.com/user-attachments/assets/b2657d1e-e714-472a-9f34-b7f74494355f" />


For the username file i will use the ones from auth bypass . (If the payloads won't work try URL encoding although it's done by default by FFUF Post requests) . 
Now just make sure you include all Headers , Filter the 200 , so that we only get the redirect (which means login succeeded )

<img width="1319" height="934" alt="image" src="https://github.com/user-attachments/assets/d8bfcd0b-0692-4779-9d92-c8b05485b6a8" />

Now we just chose any of these payloads to login and we should get our Flag . 

<img width="1695" height="877" alt="image" src="https://github.com/user-attachments/assets/994d2853-2920-4607-a7b9-9ffd8259aaad" />

Just like that we can completely Bypass the Login Page . 

<img width="1447" height="837" alt="image" src="https://github.com/user-attachments/assets/63c1cfbf-fc27-4f36-8b43-6305d19f81ee" />

Another fun challenge from Webverselabs , a great playground to freely test and explore Auth Bypass techniques via SQLi . 



