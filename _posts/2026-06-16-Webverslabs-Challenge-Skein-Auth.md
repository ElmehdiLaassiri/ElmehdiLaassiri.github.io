---
title: " Webverselabs Challenge Skein Auth  "
date: 2026-06-16 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


# Scenario : 

Skein is a volunteer-moderated forum for spinners, knitters and quilt-makers. Members hate logging in every time they swap a colourway, so the "Keep me signed in for 30 days" checkbox is on by default. The head moderator, who manages a small army of fibre-fest organisers, picked a memorable password back when the forum was just three friends and a Google sheet.


## Solution : 

<img width="1836" height="860" alt="image" src="https://github.com/user-attachments/assets/c696a037-ebd1-4ea6-b3cb-1e5f7e130eb9" />

We navigate the app just like a normal user, trying to view different endpoints, gather as much information as we can . 

**/Projects** and **/Threads** return static web pages , nothing useful for us . 

<img width="1631" height="853" alt="image" src="https://github.com/user-attachments/assets/df95feb8-2533-4d9d-82e0-c691353f0f1e" />

From the **/Members** endpoint, we find that there is a moderator account named loomweaver. We will note this for later. For now lets create an account and login. 

<img width="1488" height="871" alt="image" src="https://github.com/user-attachments/assets/3e139b7a-5d9b-4d3c-9d06-e54d3efb3ae2" />

If we check the Burp Request for our Login : 

<img width="1529" height="667" alt="image" src="https://github.com/user-attachments/assets/99dd8586-8b48-4273-b0c4-6f51dad36c1a" />

We can't really add roles or anything like that during creation of the account . 

Now that we have an account let's try to login instead of creating the account , to see if we can bypass the authentication. 

<img width="1594" height="571" alt="image" src="https://github.com/user-attachments/assets/9a9f6b74-d8b2-4b8d-9524-c9f5b616e7a7" />

Everything seems normal. 

We already have the name of the moderator ,and we know how the request should look like so if there is no rate limiting in place , we might be able to Brute force the admin password . 

But first we need to know how the email address looks like , we only have the name . 

I tried Wrong password + Correct username and Wrong username + Wrong password but the answer is the same so Username enumeration is not possible . 

<img width="1557" height="645" alt="image" src="https://github.com/user-attachments/assets/a3dfb997-e252-4a2b-a392-4ec12d1f5337" />

Let's keep digging , we might be able to find an email address . 

<img width="1687" height="543" alt="image" src="https://github.com/user-attachments/assets/f9ae6c14-27ea-41b4-8dd6-0e1cfe8c520a" />

We find this one : "post@skein.community" , assuming all employees follow the same naming pattern for the enterprise email "loomweaver@skein.community" should be the email for the admin. 

Now we can just use FFUF to brute force our way in , make sure you add content type to be URL enoding 

```bash
ffuf -X POST -u https://b5d8db80-4327-bump-key-f6754.challenges.webverselabs-pro.com/login -d "email=loomweaver@skein.community&password=FUZZ&keep_signed_in=on" -H "Content-Type: application/x-www-form-urlencoded" -w /usr/share/seclists/Passwords/Leaked-Databases/rockyou-75.txt  -fr "We couldn't find an account with that email and password" -H "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7" -H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36" -fc 200
```

We filter for 200 since a valid login will result in a redirection so a 302 :

<img width="1879" height="870" alt="image" src="https://github.com/user-attachments/assets/cd797144-ebd2-46e9-ba6a-be50ed28b54e" />

Perfect, we are able to find the password , now we just login as the moderator : 

<img width="1623" height="750" alt="image" src="https://github.com/user-attachments/assets/354b4d57-4eda-4f33-a786-ddabd8b809bc" />

From there we retrieve our flag . 

That was all for this challene, see you in the next one :) 
