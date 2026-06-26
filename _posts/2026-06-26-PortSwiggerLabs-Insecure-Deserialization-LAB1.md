---
title: " PortSwiggerlabs: Modifying serialized objects "
date: 2026-06-26 18:00:00 +0000
categories: [PortSwiggerLabs , Insecure Deserialization ]
tags: [PortSwiggerLabs , Insecure Deserialization , Challenge , Web_Attacks ]
---


## Information : 

This lab uses a serialization-based session mechanism and is vulnerable to privilege escalation as a result. To solve the lab, edit the serialized object in the session cookie to exploit this vulnerability and gain administrative privileges. Then, delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter


## Solution : 

First we login as the wiener user.

<img width="1674" height="714" alt="image" src="https://github.com/user-attachments/assets/358580e2-cd86-4696-87a2-2df2e4e91643" />

Then we check Burp's sitemap to check all the endpoints : 

<img width="1920" height="902" alt="image" src="https://github.com/user-attachments/assets/24817c72-a6a3-4b04-809a-f5bfa38e053e" />

We see that once we login we get a session cookie , looking at Inspector , we see that it contains some information about our user , the username as well as the role , in this case admin role is set to 0 . 

Looking at the decoded value this is a PHP's native serialization format : 

- O:4:"User":2:{...} → Object, class name is 4 characters long ("User"), and it has 2 properties
- s:8:"username" → a string property, 8 characters long, named "username"
- s:6:"wiener" → string value, 6 chars, "wiener"
- s:5:"admin" → string property name "admin", 5 chars
- b:0 → boolean value, false

Now what we could try here is to change the boolean value to 1 , and see if we're granted admin privileges . 

First we send the request again then interecept it this time and modify the cookie using Inspector and forward the request : 

Now first we login :

<img width="1531" height="874" alt="image" src="https://github.com/user-attachments/assets/3fcdf957-a9a2-45ad-9a24-486dfc5c1f73" />

Then we modify the Cookie that we got : 

<img width="1197" height="776" alt="image" src="https://github.com/user-attachments/assets/cbc17472-e238-4d70-8e74-8c5eba294e26" />

We can do that using Inspector : 

<img width="1886" height="516" alt="image" src="https://github.com/user-attachments/assets/a9e32df8-a288-466e-be74-cd0b612f8790" />

Now we just forward the rest : 

<img width="1864" height="745" alt="image" src="https://github.com/user-attachments/assets/8e7fa6f8-6b6d-497d-beb3-2f9c6cacedde" />

And we are able to access the admin interface , we can see that now we've got an admin panel that we can visit : 

<img width="1686" height="549" alt="image" src="https://github.com/user-attachments/assets/1250c6f3-aaee-4a53-b376-4008849179cf" />

Now to get the flag , we need to delete the user carlos , again , we need to intercept the request and modify the cookie whenever we want to perform any action as the admin , once we intercept it we modify the value to 1 for admin : 

<img width="1911" height="896" alt="image" src="https://github.com/user-attachments/assets/7b5b12a9-b6d8-4870-8ff5-900413490f00" />

Then the final request as well : 

<img width="1901" height="915" alt="image" src="https://github.com/user-attachments/assets/e272fafd-fa29-4c10-8fca-a60a843accd3" />

Finally once we delete carlos , we should see that lab was solved : 

<img width="1798" height="701" alt="image" src="https://github.com/user-attachments/assets/6ba8c17a-9a19-4b1d-aedc-28c649053bfa" />

Just make sure you modify the cookie in every request , you could automate it by using a match and replace rule , but since there are only few requests we can do it mmanually still .

That was all for this lab , see you in the next one :)
