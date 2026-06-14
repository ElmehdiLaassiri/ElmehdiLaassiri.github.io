---
title: "Webverselabs Challenge Statuscraft SSRF  "
date: 2026-06-12 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---



## Scenario : 

Statuscraft is a two-engineer side project out of Berlin, founded in 2022 as a hobbyist alternative to Statuspage — drop in a URL, get a status page, pipe outages to Slack. Hobby tier is free, the team tier is €9/month, and the self-hosted build ships next month. The add-monitor form shows you the first ping inline so you can sanity-check the response before committing the monitor; the engineers built it in a weekend and never came back to harden it.


## Solution : 

<img width="1332" height="841" alt="image" src="https://github.com/user-attachments/assets/af3116a6-3c3b-49f4-a3d3-8fc1c44e797f" />

First, we need to create an account to explore the app further. 

<img width="1055" height="634" alt="image" src="https://github.com/user-attachments/assets/3041bf97-26b1-4c52-848f-58f2d9098cd8" />

Once we login, we find 2 new endpoints, and the possibility to add a monitor, i checked the other endpoint but **/Incidents** only returned a static web page . 

<img width="990" height="786" alt="image" src="https://github.com/user-attachments/assets/eae52fd5-64ed-4795-9ea5-5668761d04c6" />

And Looking at the burp request , there is nothing useful there too. 

Now back to our **/monitors** endpoint , we can try to add a new monitor and see what Requests are made to the server when we do so.

<img width="1149" height="672" alt="image" src="https://github.com/user-attachments/assets/54c3c8fb-f7ef-4bcc-9b5a-717fa3be74ba" />

First i added the Url for our Webhook Site :

<img width="1135" height="694" alt="image" src="https://github.com/user-attachments/assets/ed750315-7b85-49ac-a7c4-b3ce8de694d8" />

This didn't work for whatever reason , but looking at Burp : 

<img width="924" height="592" alt="image" src="https://github.com/user-attachments/assets/589cf0ac-f3b1-42d3-9486-6ef78f1786a8" />

We first will try to use a different URL like google.com and see if we can make the server fetch that.

<img width="1344" height="696" alt="image" src="https://github.com/user-attachments/assets/8f7b86de-fb4b-4a50-a27c-a8015f4d491d" />

We get a redirect but once again we dont get the ping, maybe it can't reach external IP addresses. 

<img width="1518" height="591" alt="image" src="https://github.com/user-attachments/assets/2d24155b-ed42-4f13-9e91-2c5692c762c5" />

But what if we specify an internal IP address like localhost ? will it work?

<img width="1502" height="699" alt="image" src="https://github.com/user-attachments/assets/8a0f3007-bf19-4920-bf61-d51fd802b3f9" />

At first i tried the common ports for web apps to see if we get any different result , i tried 443,8000,8080,80.

<img width="1504" height="671" alt="image" src="https://github.com/user-attachments/assets/ff09d99d-7a90-4df3-82e4-08047df0f967" />

Port 443 and 80 returned the same response as before. But when i tried 8080 :

<img width="1420" height="609" alt="image" src="https://github.com/user-attachments/assets/eecbee8c-372b-496f-943c-41d03d3b144e" />

If we follow the redirection : 

<img width="1508" height="732" alt="image" src="https://github.com/user-attachments/assets/acb30903-f695-42bb-8453-f5f0d67f0461" />

Perfect , this proves that port 8080 is indeed open.

now we can try an internal port scan , for this we can either use FFUF or Burp's Intruder , i always use FFUF in my writeups so let's use Intruder this time.

First , send the POST Request to Intruder and specify the Port as the variable : 

<img width="1660" height="732" alt="image" src="https://github.com/user-attachments/assets/337f7888-0a5f-4e26-85c3-f15a9e2518a6" />

Then paste the listt of ports you want to test , i used the most common ones . 

<img width="975" height="630" alt="image" src="https://github.com/user-attachments/assets/b344e4a7-ade0-4bf6-a330-4d9668f7ede3" />

Before we start the attack , since ALL the request are a Redirection , we need to tell Intruder to Follow redirections otherwise we can't filter anything . (Settings --> Redirection --> Always)

<img width="916" height="722" alt="image" src="https://github.com/user-attachments/assets/9af269e9-9e00-4e73-b6b5-f2ec3d6124b4" />

Then finally we just start our attack . 

<img width="1394" height="756" alt="image" src="https://github.com/user-attachments/assets/b4e7d1b7-25de-41df-96c4-6ed59244c164" />

From there we filter based on the Content of the Response , for each response we will have Response 1 which is the same across all requests since that's just the redirection , and the Reponse 2 which is after we follow the redirection .

Here I'm looking for a response that has similar lebght to the response for 8080 . 

<img width="1422" height="749" alt="image" src="https://github.com/user-attachments/assets/9d8bc49c-a3c4-4dbc-a8f9-06b76988fd89" />

We find Port 5000 is also UP and running . 

<img width="1152" height="712" alt="image" src="https://github.com/user-attachments/assets/103647cc-ccb9-4f5b-8803-ed20ea41e767" />

I didn't notice it earlier but the flag was in the Response for 8080 not 5000 -_- 

<img width="1152" height="712" alt="image" src="https://github.com/user-attachments/assets/23045a97-5dac-4145-b330-d676f0a71e7d" />

But anyways we manage to try Internal Port scanning via SSRF .

That was all for this challenge. See you in the next one :)
