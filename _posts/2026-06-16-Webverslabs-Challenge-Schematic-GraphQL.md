---
title: " Webverselabs Challenge Schematic GraphQL "
date: 2026-06-16 07:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---


## Scenario : 

Schematic Inc built a slick dashboard for their team. Everything looks polished on the surface, but the API powering it might expose more than the UI lets on. Dig deeper.


## Solution : 

<img width="1821" height="850" alt="image" src="https://github.com/user-attachments/assets/05a3251f-b3f2-49d7-bde6-1fbbff46ee40" />

We first start by navigating the app just like a normal user would, check the different endpoints, use different features and then check Burp's History to see what kind of requests were made to the server. 

**/Team :**

<img width="1764" height="723" alt="image" src="https://github.com/user-attachments/assets/9feaab9f-1a6b-46b5-b61f-dfe67ced19bf" />

This is a static page that returns all users with their appropriate role and status . 

**/Products :**

<img width="1837" height="716" alt="image" src="https://github.com/user-attachments/assets/5be825d3-96a5-48a3-ba6a-76fa5cd57b39" />

A list of products with their prices, etc.

**/About :** 

<img width="1860" height="807" alt="image" src="https://github.com/user-attachments/assets/710fa05a-dd80-49b0-9004-c0a70b6f7a91" />

Another static page , nothing we can work with here . 

If we check Burp's History : 

<img width="1297" height="654" alt="image" src="https://github.com/user-attachments/assets/aca451b9-2440-4e4e-86f4-f13d735999d8" />

We find that the app uses GraphQL for most of the requests , i already have a detailed section on my mehtodology on how we can attack GraphQL : 

```bash
https://elmehdilaassiri.github.io/posts/web-app-pentesting-methodology/#graphql-
```

Now first thing to test whenever we have GraphQL is the introspection query , as this will return valuable information about the entire structure of the backend, and we can use that with a tool like Voyager to be able to view the entire structure as a graph. 

First we identify the engine being used :

```bash
+ We can use graphw00f , it will send multiple queries to identify the engine used : 
https://github.com/dolevf/graphw00f
python3 main.py -d -f -t https://8a3f31b4-4327-schematic-e9433.challenges.webverselabs-pro.com/
```
<img width="1480" height="894" alt="image" src="https://github.com/user-attachments/assets/fc44920b-8c3c-4373-a670-a633b954815c" />

We identified that the template it Apollo . It also gives us a link to check for misconfigurations on Apollo : 

<img width="1340" height="555" alt="image" src="https://github.com/user-attachments/assets/78685f84-40b3-40cb-8215-7a17fb2f8272" />

We see that by default Introspection is enabled on this engine , we can try and use Burp to send the introspection request :

<img width="1529" height="818" alt="image" src="https://github.com/user-attachments/assets/c51e18c0-422e-464d-911d-c19f7cc24969" />

I already have the GraphQL extension . 

Once we send it , we get the response containing all the details about the GraphQL endpoint :

<img width="1636" height="802" alt="image" src="https://github.com/user-attachments/assets/a811584a-b7ac-4b9b-b5a7-c308e98f3bbf" />

The different relationships between different queries and objects . 

We can use Voyager and paste the introspection response there and it will generate the graph for us : 

<img width="1606" height="852" alt="image" src="https://github.com/user-attachments/assets/9a63091b-7ffc-483b-ae7f-5c10cd267a07" />

From there we should get our graph to be able to visualize everything : 

<img width="1333" height="817" alt="image" src="https://github.com/user-attachments/assets/79317c4d-9bdd-4c85-941b-758c3147751c" />

We already saw the **Products** , and **Users** but the **SystemConfig** Query wasn't mentionned anywhere . 

We can attempt to get the System Config , to be able to query it , we already have the fields we want to select *id/key/value* from the graph : 

To make a query is pretty simple using the GraphQL extension , we just type the query field we want to call and the fields we want back :

```graphql
{ systemConfigs { id key value } }
```

<img width="1647" height="719" alt="image" src="https://github.com/user-attachments/assets/e2eefd8c-4f48-4d6e-a10c-925ead85e236" />

And we should find our flag : 

That was all for this challenge, see you in the next one :) 

