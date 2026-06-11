---
title: "Webverselabs Milltrace Challenge XSS "
date: 2026-06-11 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Hard , Challenge , Web_Attacks ]
---


## Solution : 

Millrace is a microbrewery and taproom that hosts live music. Their homepage has a tiny debug pane echoing the visitor's user-agent into an HTML comment for troubleshooting. The /debug page lets anyone set their session-scoped UA to any string they like. You can see the problem from here.

## Solution : 

<img width="1835" height="837" alt="image" src="https://github.com/user-attachments/assets/0c72b637-3aab-4573-a829-a038bb5792ef" />

We first start by browsing the application like a normal user would, to check different endpoints and try to find where we can inject.

**/Pours :**

<img width="1623" height="907" alt="image" src="https://github.com/user-attachments/assets/7f15674d-1ddd-4a17-b751-f92af47de59a" />

Nothing useful here , just a List of Beer names. 

**/Events :**

<img width="1496" height="874" alt="image" src="https://github.com/user-attachments/assets/c18ae62e-b279-418d-8b90-f8e6e37b96cb" />

This isn't useful either , just a static web page showing us different planned events . 

**/Tour :** 

<img width="1685" height="742" alt="image" src="https://github.com/user-attachments/assets/37ffdbf2-8d09-4bc8-a705-6dbd9d3504e4" />

Just a static web page, nothing useful here either :) 

**/Debug :**

<img width="1481" height="903" alt="image" src="https://github.com/user-attachments/assets/45273108-45a9-41f5-a485-ed0fcec041b0" />

This time we find an injection spot finally :)

First thing we should try is use it normally then check both the front end code as well as Burp's History to see how our request is being processed by the server . 

<img width="1296" height="827" alt="image" src="https://github.com/user-attachments/assets/cc1a3041-f4e4-4d20-84e1-cc924d1c62f5" />

If we check the front end code : 

<img width="1780" height="563" alt="image" src="https://github.com/user-attachments/assets/ec6f5134-c09a-4a76-b540-f4fd9a67c5b9" />

We see that our payload gets URL encoded , and stored , let's go back to the Dashboard :

<img width="1524" height="650" alt="image" src="https://github.com/user-attachments/assets/8cf49468-df8d-458e-9c17-ad4a3a29b971" />

We see that the payload is being rendered back to us -_- , let's check the front code again :

<img width="1106" height="628" alt="image" src="https://github.com/user-attachments/assets/0618067e-6ac8-4ec5-a6a5-470f44694043" />

For whatever reason it is rendered back to us as an HTML comment , we can simply try to break out of it by ending the comment before our payload : 

```javascript
--> <script>alert(window.origin)</script>
```

Let's try this one : 

<img width="1364" height="724" alt="image" src="https://github.com/user-attachments/assets/6c0ebd73-3890-4e23-8ca7-4fa4ac13f7a9" />

Now if we go back to the Dashboard , hopefully our JS is executed . 

<img width="1471" height="446" alt="image" src="https://github.com/user-attachments/assets/e8d08464-c119-4683-baf3-c2653ac82bfc" />

And it is executed , from there we just get the flag : 

<img width="1287" height="562" alt="image" src="https://github.com/user-attachments/assets/e3a99cfd-cc8c-48fd-8709-fc7fcc1bfac7" />

If we check the front end code one last time , we can see that we were able to break out of the HTML comment : 

<img width="1181" height="446" alt="image" src="https://github.com/user-attachments/assets/0d280690-d874-4cb3-b560-ac6d3def2c5e" />

That was all for this challenge . See you in the next one :) 
