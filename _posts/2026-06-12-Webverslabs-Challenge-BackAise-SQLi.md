---
title: "Webverselabs Challenge BackAisle SQLi  "
date: 2026-06-12 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---



## Scenario : 

Yardlines built their storefront on a long weekend in 2022. The category filter on the shop page was the kind of throwaway code you write at 2am with a deadline tomorrow — quick, direct, never revisited. They added a friends-and-family capsule program in 2024 and bolted a second visibility rule onto the same code path, figuring nobody could tell that capsule pieces existed if they weren't on the public grid. The grid is just one view of the data.


## Solution :


<img width="1888" height="899" alt="image" src="https://github.com/user-attachments/assets/37bbe77a-352a-4dcf-8679-380d057e5d4c" />

We first start by exploring the app just like a normal user would . Try and see different endpoints, features, etc... 

Then we check Burp's History to see if we find any interesting requests, we're mainly looking for parameters that we can inject whether it be a parameter in the URL or a GET Request or inside the body of a POST Request . 

**/Shop :**

<img width="1641" height="916" alt="image" src="https://github.com/user-attachments/assets/18ed0091-8f06-4c9f-a5a7-7869a0c1c4ba" />

This endpoint just gives us a list of products that we can view, buy etc , usually each product will have an id which we will find in the GET request . 

<img width="1662" height="784" alt="image" src="https://github.com/user-attachments/assets/9146de73-085d-41ac-860f-e97859c43e62" />

Well we don't have an ID , but at least we find a parameter that we can use to inject , i first injected a simple payload to trigger the sqli . Which didn't cause any errors which is not a good sign . 

<img width="1338" height="432" alt="image" src="https://github.com/user-attachments/assets/0b53a600-c7e9-44e4-a14a-5490ef3d890d" />

Just to be sure i still used sqlmap to try and test this parameter : 

```bash
https://github.com/sqlmapproject/sqlmap
python3 sqlmap.py -u "https://bba07957-4327-back-aisle-124e4.challenges.webverselabs-pro.com/product.php?slug=field-jacket-olive" --level=5 --risk=3 --batch
```

<img width="1607" height="673" alt="image" src="https://github.com/user-attachments/assets/9aaf1923-d642-4468-a677-c75053ba3939" />

Unfortunately, this didn't work either. Let's just keep looking for now.

**/Drops :**

<img width="1683" height="931" alt="image" src="https://github.com/user-attachments/assets/29e26a77-0d49-4bfa-89ab-7b9d6cc48e5d" />

New parameter , again i injected a simple payload "'" to try and cause an error . 

<img width="1114" height="266" alt="image" src="https://github.com/user-attachments/assets/2c16a992-f3a9-43e5-a503-f812ef4d4ffa" />

Perfect, but we can't confirm still that this is an SQLi for sure, now if we add another ' this should close the query and we shouldn't get an error , if this works this is also a huge indidcator that this parameter is indeed vulnerable. 

<img width="1362" height="625" alt="image" src="https://github.com/user-attachments/assets/bf87fdc7-20a1-456b-9d18-51820426bd72" />

And we do get rid of the error , now that we know this might be vulnerble we can use sqlmap to try and dump the entire Database . 

Another way to confirm the SQLi is by commenting the rest and see if this solves the error , for example we inject 

```sql
'-- - : This should comment the Rest of the query so we won't have an issue breaking the syntax which should remove the error we got earlier .
```

<img width="1498" height="614" alt="image" src="https://github.com/user-attachments/assets/0e7d7fb0-7015-40df-80d0-3a6933d09c7b" />

Now moving on to sqlmap — make sure to add --random-agent since the firewall fingerprints and blocks sqlmap's default User-Agent.

```bash
python3 sqlmap.py -u "https://bba07957-4327-back-aisle-124e4.challenges.webverselabs-pro.com/shop.php?category=*" --batch --random-agent
```

Without it, even adding up --level 5 --risk 3 won't help , the request gets blocked before sqlmap even gets a chance to test payloads. 

<img width="1464" height="952" alt="image" src="https://github.com/user-attachments/assets/76fc5c5d-c88e-4fa7-83f8-87eba3fda4e6" />

Simply randomizing the User-Agent was enough to bypass the firewall and get sqlmap running successfully.

<img width="1550" height="929" alt="image" src="https://github.com/user-attachments/assets/fc1d1b4f-891a-4d4d-b795-8efbdbd9e53a" />

Finally we just dump the entire DB to get the flag : 

<img width="1791" height="685" alt="image" src="https://github.com/user-attachments/assets/660b68e7-5fa0-4bf6-9fc0-3b2c3c54c8b4" />

That was all for this challenge . See you in the next one :) 

