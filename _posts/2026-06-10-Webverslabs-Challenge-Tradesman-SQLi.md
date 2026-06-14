---
title: "Webverselabs Challenge Tradesman SQLi  "
date: 2026-06-10 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---


## Scenario : 

Stylus Hi-Fi is a vintage record marketplace based in Portland — sellers list their inventory, buyers search by Goldmine grade, the platform takes 8%. The public marketplace went through three frontend rewrites since 2017. The seller dashboard hasn't. Login still goes through the original query.


## Solution : 

<img width="1572" height="886" alt="image" src="https://github.com/user-attachments/assets/7480c57f-8712-4494-8875-939d99baee60" />

We first start by navigating the application just like a normal user, to check different endpoints and features, then we check Burp's History to see if we find anything interesting . We're mainly looking for parameters where we can inject. 

**/Browse !**

<img width="1489" height="861" alt="image" src="https://github.com/user-attachments/assets/b13d182a-7a9e-4033-808b-7f8b951fa3b1" />

This is a list of Products , at first i thought clicking on any of them will shows us a parameter inside of the GET request that we can use to injetc but that wasn't the case : 

<img width="1579" height="892" alt="image" src="https://github.com/user-attachments/assets/b00cb843-eee6-408a-a72a-1f05db557302" />

I knew it wasn't going to work but i even tested to inject the URL still , but that just returned an error . 

<img width="1418" height="639" alt="image" src="https://github.com/user-attachments/assets/178057bc-d682-46e6-8109-4c43e5495bd0" />

You can try SQLmap but due to the request not having any parameters to inject, it won't be useful either. 

<img width="940" height="372" alt="image" src="https://github.com/user-attachments/assets/60cc0616-5568-4a74-844b-ddbddecad562" />

Now moving on, we find the **Apply** endpoint where we can apply to start posting our products and sell on the website as well. 

<img width="1178" height="749" alt="image" src="https://github.com/user-attachments/assets/07dcba8b-f885-46a2-9c4f-aaf4a0bc7bf3" />

But apparently this endpoint doesn't work either. It's just a static web page that is returned to us . 

Now the last thing we've got left is the **Sign in** endpoint :

<img width="1232" height="648" alt="image" src="https://github.com/user-attachments/assets/bc145a69-75da-4cf8-8fa4-bb40ed83f5dc" />

Now if we send a normal Request : 

<img width="1596" height="625" alt="image" src="https://github.com/user-attachments/assets/d0246a27-ccdf-4c66-aa78-992aa9bed5ff" />

We see that when we login , it makes a POST request to the **/seller/login** endpoint , with our username and password . 

Now whenever we have a login page , first thing you should think of is Auth Bypass via SQL injection . Now there are so many payload we can test , here is a LIST of them :

```bash
https://gist.github.com/spenkk/2cd2f7eeb9cac92dd550855e522c558f
```

I decided to use FFUF , and inject all of these payloads in the username field , and finally filter for the Response we find in a failed Response : *Invalid handle or password* .

First we put all the payloads in a file : 

<img width="854" height="365" alt="image" src="https://github.com/user-attachments/assets/206b910e-94a4-46dc-a142-e4c8aed54e2c" />

Then we craft the FFUF Request , it is pretty simple , just make sure you add all necessary parameters SPECIALLY *"Content-Type: application/x-www-form-urlencoded"* since we are passing Special characters and for the rest of the request it should be pretty simple since we already have the Burp Request : 

```bash
 ffuf -X POST -u https://9ba0dee6-4327-tradesman-d1afa.challenges.webverselabs-pro.com/seller/login \
-H "Content-Type: application/x-www-form-urlencoded" \
-d "handle=FUZZ&password=aaaaaaaa" \
-w Auth_bypass_list \
-fr "Invalid handle or password"
```

<img width="1267" height="824" alt="image" src="https://github.com/user-attachments/assets/5e172887-bbb2-4966-8201-c9325d8cd926" />

Perfect, a Redirect , which means we're logged in , let's try any of these : 

<img width="1113" height="503" alt="image" src="https://github.com/user-attachments/assets/3956920b-2227-42b7-90d8-be251a19a093" />

And just like that we're in : 

<img width="1508" height="837" alt="image" src="https://github.com/user-attachments/assets/fd09d256-8015-4fec-9ac9-3d41987617e0" />

The flag can be found under the Admin Notes endpoint : 

<img width="1565" height="845" alt="image" src="https://github.com/user-attachments/assets/8184a043-5fe9-4bda-a52a-2135f4685ee8" />

That was all for this challenge, see you in the next one :)
