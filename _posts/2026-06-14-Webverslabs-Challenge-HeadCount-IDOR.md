---
title: "Webverselabs HeadCount Challenge IDOR "
date: 2026-06-15 21:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Hard , Challenge , Web_Attacks ]
---


## Scenario : 

Headcount's engineering team added an include_raw parameter to the aggregation endpoint for debugging during the compensation review sprint. A TODO comment in the org chart template reminded them to remove it. Neither the comment nor the parameter made it into the cleanup ticket. A regular employee account is enough to pull full compensation report data.


## Solution : 

<img width="1848" height="845" alt="image" src="https://github.com/user-attachments/assets/f3245316-7708-4d9b-b1d5-1fc8c9c3d5b7" />

We first start by creating our account so that we can navigate the app. 

<img width="1867" height="889" alt="image" src="https://github.com/user-attachments/assets/f83c2d6d-c2dd-4e07-b34e-c839133740ca" />

Once inside, we can start enumerating each endpoint, the goal here is to check all endpoints then move to Burp's History to see if we have any requests parameters that we can tamper with , an ID or somethine like that. 

**/Briefing :**

<img width="1538" height="670" alt="image" src="https://github.com/user-attachments/assets/0f17f408-2c66-4dd8-ba36-322a2d3f6054" />

Just a static page with a memo we can read, nothing that useful really . 

One thing to note is that once we login , we see 2 Requests made to the server : 

<img width="1351" height="761" alt="image" src="https://github.com/user-attachments/assets/356a851d-0f97-44f2-ad5f-78ea4b272b61" />

**/api/me** and **/api/orgchart** : 

The **/api/me** returns information about us  : ID, email, Role, etc .

<img width="1273" height="498" alt="image" src="https://github.com/user-attachments/assets/8c5557ea-0312-429c-adda-71cee23ab27e" />

And **/api/orgchart** returns information about the entire organization : 

<img width="1272" height="664" alt="image" src="https://github.com/user-attachments/assets/d1c5b842-c8fb-4124-b87e-a8e9eb6fbe9b" />

For now let's keep enumerating the app.

**/Directory :**

<img width="1555" height="861" alt="image" src="https://github.com/user-attachments/assets/cb775612-a6a1-4034-8560-9ca503d42210" />

When we visit it , we get the same GET request as before, the **/api/orgchart** . 

**/Comp bands :**

<img width="1670" height="816" alt="image" src="https://github.com/user-attachments/assets/ee7bf87b-dffe-4b63-90d5-0c08ec7fa507" />

A static web page , nothing useful , Request doesn't contain any parameters. 

**/Profile :**

<img width="1825" height="866" alt="image" src="https://github.com/user-attachments/assets/ad09cf69-f94a-4c65-a954-aa448097b4fd" />

We find a list of Resources we can access , we already checked /api/me and /api/orgchart , what's left is the **/api/hr/reports/aggregate** :

<img width="1387" height="657" alt="image" src="https://github.com/user-attachments/assets/725e30a7-5e6f-40fc-88ab-749550d7c8f0" />

If we try accessing it, we get a forbidden error, Only HR Admins are allowed. 

After checking all the endpoints i didn't find any useful information we can use so i decided to read front end code for each endpoint , then i found this comment on the **/orgchart** endpoint : 

<img width="1216" height="662" alt="image" src="https://github.com/user-attachments/assets/97727624-9663-47df-b397-cc979c43fd84" />

We see this include raw parameter that they want to remove before prod , maybe we can use this to get more information about an endpoint , specifically the one we can't access like the **/api/hr/reports/aggregate** endpoint :

Before that i did add it to all the other requests to see if it would give us additional information :

<img width="1507" height="611" alt="image" src="https://github.com/user-attachments/assets/3c30d5d1-7ce3-4abb-b80f-4c31b8bb46a6" />

For the value, it just made sense that it will either True or false (1 or 0 ) or something like that, but just a Boolean.

<img width="1539" height="666" alt="image" src="https://github.com/user-attachments/assets/ed6fd98d-e49a-46c5-a4ca-95ad40d50b8b" />

If we add it, we see that it bypasses the Restriction we had earlier . We can list the content of the page . 

Now if we scroll down , we should be able to see our Flag : 

<img width="1646" height="648" alt="image" src="https://github.com/user-attachments/assets/e38d2745-852a-44da-9339-f392e3d4a53e" />

That was all for this challenge, see you in the next one :) 



