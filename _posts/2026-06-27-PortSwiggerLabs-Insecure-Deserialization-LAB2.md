---
title: " PortSwiggerlabs: Modifying serialized data types "
date: 2026-06-27 00:00:00 +0000
categories: [PortSwiggerLabs , Insecure Deserialization ]
tags: [PortSwiggerLabs , Insecure Deserialization , Challenge , Web_Attacks ]
---



## Information : 

This lab uses a serialization-based session mechanism and is vulnerable to authentication bypass as a result. To solve the lab, edit the serialized object in the session cookie to access the administrator account. Then, delete the user carlos.

You can log in to your own account using the following credentials: wiener:peter


## Solution :

<img width="1608" height="882" alt="image" src="https://github.com/user-attachments/assets/f104f268-81ed-4969-a075-b832ee9f6e14" />

First we start by logging in as wiener : 

<img width="1645" height="782" alt="image" src="https://github.com/user-attachments/assets/d8a406c1-754d-43da-bdc2-f71d1032e1a4" />

Once logged in , we see that we have 3 bottons , home, Myaccount and Log out , now if we check Burp's Site map : 

<img width="1911" height="945" alt="image" src="https://github.com/user-attachments/assets/e29d40f4-384b-49a4-bb93-b94de54f5a13" />

We see that once we're logged in, we're given a cookie , if we had Burp Pro , it will automatically detect that we have a serialized object , but my case i didn't have that , but we can see from Inspector that the value of that cookie once decoded is : 

```bash
O:4:"User":2:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"ly9g32nbnilkfzgmphe3u4nqm68jay1l";}
```

This is a standard PHP serialized object format : We see that user has 4 characters and 2 attributes , username and access token , before each one of them we have the lenght for each one of these attributes . 

Now one way to abuse this is by changing the user we got to someone else , in this case the administrator :

```bash
O:4:"User":2:{s:8:"username";s:6:"administrator";s:12:"access_token";s:32:"ly9g32nbnilkfzgmphe3u4nqm68jay1l";}
```

This won't work since we need to make sure we specify the lenght of the word right before it , which will be 13 rather than 6 :

```bash
O:4:"User":2:{s:8:"username";s:13:"administrator";s:12:"access_token";s:32:"ly9g32nbnilkfzgmphe3u4nqm68jay1l";}
```

Now back to our burp , either intercept each request and modify the cookie , or just copy the modified cookie from inspector and paste it inside the browser tool . 

<img width="1904" height="753" alt="image" src="https://github.com/user-attachments/assets/57f99656-b3b3-4527-addb-1be8c3df23e8" />

Now back to our browser : 

<img width="1727" height="746" alt="image" src="https://github.com/user-attachments/assets/511f89d3-2896-4f6e-8887-b20fc1a56e70" />

We just change the value of the cookie : 

<img width="1588" height="851" alt="image" src="https://github.com/user-attachments/assets/bfc84429-0c6f-43f4-beae-2b0fb151975a" />

Here we have a problem with the access token , a way around this would be to replace the entire value with 0 , and replacing the type from s (String) to i for Inetger :

```bash
O:4:"User":2:{s:8:"username";s:13:"administrator";s:12:"access_token";i:0;}
```

<img width="1902" height="656" alt="image" src="https://github.com/user-attachments/assets/d5c544ee-a5b2-41dc-918a-af89949e65ac" />

Now we replace the old cookie on our dev tools : 

<img width="1765" height="908" alt="image" src="https://github.com/user-attachments/assets/f1054d0e-5177-4539-a3aa-7ec64f8bacd6" />

We see that now we can see the Admin panel , which we will use to delete the user carlos : 

<img width="1684" height="753" alt="image" src="https://github.com/user-attachments/assets/003572de-c131-4a92-8fc7-1329f5a2ceef" />

Now once we delete it , we should solve the lab : 

<img width="1697" height="794" alt="image" src="https://github.com/user-attachments/assets/75728862-2731-49cb-a6ac-3697e208be4f" />

That was all for this lab, see you in the next one :)

