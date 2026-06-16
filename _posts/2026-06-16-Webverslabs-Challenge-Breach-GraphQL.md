---
title: " Webverselabs Challenge Breach GraphQL "
date: 2026-06-16 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


## Scenario : 

Breach is an internal collaboration tool where teams share notes and documents. The developers implemented access controls, but the architecture has layers — and not all of them agree on who can see what.

## Solution : 

<img width="1870" height="883" alt="image" src="https://github.com/user-attachments/assets/8fc6460c-2505-468e-aced-aa19e2fc499e" />

We start by navigating the app just like a normal user, check the different endpoints , see if we find anything interesting, from there we check Burp's History to see what requests were made to the server . 

**/Profile :**

<img width="1908" height="602" alt="image" src="https://github.com/user-attachments/assets/36406fb5-7469-4be7-b9e2-d65ce1f37d13" />

This returns information about our profile, we see that we have an ID , we will keep this in mind. 

**/Teams :**

<img width="1873" height="646" alt="image" src="https://github.com/user-attachments/assets/7b8d4906-c8e5-4786-91d6-46bf4ed80cb9" />

This lists Team members we see that danial is the Administrator , we will note this info as well . 

**/About !**

Finally the About page is just a static page that returns information about the company : 

<img width="1873" height="809" alt="image" src="https://github.com/user-attachments/assets/ff8d6a53-56d6-434c-809c-c91646f42523" />

Nothing really useful . 

Let's check Burp's History : 

<img width="1377" height="617" alt="image" src="https://github.com/user-attachments/assets/822c0a4a-9a72-4851-b48a-c1621027dc1c" />

We see that we have a GraphQL endpoint , which means the app uses graphql : 

I already have a detailed section on my mehtodology on how we can attack GraphQL : 

```bash
https://elmehdilaassiri.github.io/posts/web-app-pentesting-methodology/#graphql-
```

Now first thing to test whenever we have GraphQL is the introspection query , as this will return valuable information about the entire structure of the backend, and we can use that with a tool like Voyager to be able to view the entire structure as a graph. 

First we identify the engine being used :

```bash
+ We can use graphw00f , it will send multiple queries to identify the engine used : 
https://github.com/dolevf/graphw00f
python3 main.py -d -f -t https://6d887ea1-4327-breach-2308d.challenges.webverselabs-pro.com/about
```

Once again the engine used is Apollo : 

<img width="1594" height="887" alt="image" src="https://github.com/user-attachments/assets/ac741ba2-f9cb-46d7-9c9b-ab2220bba14d" />

It also gives us a link to check for misconfigurations on Apollo : 

<img width="1340" height="555" alt="image" src="https://github.com/user-attachments/assets/78685f84-40b3-40cb-8215-7a17fb2f8272" />

We see that by default Introspection is enabled on this engine , we can try and use Burp to send the introspection request :

<img width="1479" height="782" alt="image" src="https://github.com/user-attachments/assets/9bbbbce5-a9fc-47b9-962e-31447e3fc99e" />

If we send the introspection request :

<img width="1673" height="799" alt="image" src="https://github.com/user-attachments/assets/e990e5b9-1f7e-4350-aee8-5cefb599c07e" />

We get the response which we can paste into Voyager to be able to get the entire structure of the backend in a graph. 

<img width="1469" height="736" alt="image" src="https://github.com/user-attachments/assets/594cae7e-9ceb-4b51-85da-68b1507368ac" />

We see that we have the flag query which we can try , we already have the fields we need to retrieve : 

<img width="1446" height="769" alt="image" src="https://github.com/user-attachments/assets/8a5cfa20-ba14-4a32-a74a-0b0eba028268" />

If we try that , we get an error. Looking back at the Graph from Voyager : 

<img width="1746" height="756" alt="image" src="https://github.com/user-attachments/assets/d2df6690-2e73-4cb4-8594-18b66486fbeb" />

We can see that the Flag Query requires a parameter which is debug , which takes a Boolean . 

Now if we add this to our Request : 

<img width="1569" height="753" alt="image" src="https://github.com/user-attachments/assets/66c0547d-7cc2-4caf-b0a2-cb5b63cf57b0" />

We get our flag . 

That was all for this challenge, see you in the next one :) 


