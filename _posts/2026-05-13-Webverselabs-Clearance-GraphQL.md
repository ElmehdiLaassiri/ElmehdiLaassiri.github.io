---
title: "Webverselabs Clearance Challenge GraphQL "
date: 2026-05-12 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , IDOR ,GraphQL ]
---


## Scenario  :

CliniCore shipped a GraphQL API to power their new timeline feature. A viewerRole variable was added during development to allow the admin panel to preview how different roles see patient data. It was never removed from the production schema. A receptionist account with routine access to the system is all you need.

## Solution :

Upon visitng the webapp , we find multiple endpoints to visit , the most important one here will be the API endpoint as ait will probably give us an idea on how the application works in details since other devs might wanna use these information to integrate this app APi to their project . 


<img width="1828" height="769" alt="image" src="https://github.com/user-attachments/assets/87f2c712-7cce-405e-922c-8276f154fb03" />


On the API endpoint we find these roles : 


<img width="1249" height="859" alt="image" src="https://github.com/user-attachments/assets/c2b38c34-d520-4916-bc35-61584364ff63" />


We will keep them in mind since if were to try and priviilege escalate later on we need to know the different roles that exist first . We also know the application uses GraphQL as mentionned in the API documentation . 
Now let's create an account to be able to enumerate the application further .


<img width="1078" height="714" alt="image" src="https://github.com/user-attachments/assets/7efc220e-af99-4723-9b07-fb5f2f7c501d" />


As usual we will browse the webapp like a normal user , then check the Burp History for the different endpoints and request made to the server . 

We do find these 3 endpoints , First the profile page , which is a GET request to the /api/me endpoint which only returns information about our user , no parameter we can abuse or modify to get other user's information here . 


<img width="1112" height="581" alt="image" src="https://github.com/user-attachments/assets/adcaa721-a416-44c6-af67-60cd17835c0f" />


Second endpoint is where we find information about all the users that exist , i tried all injections on the search parameter to see if we can list other users via SQLi but didn't get anything out of it , apparently these are all the users that exist .  


<img width="1221" height="740" alt="image" src="https://github.com/user-attachments/assets/11b60e92-5831-4862-866a-837f9366bfba" />


Now if we try to see information about a specific user , this is the GraphQL query that was made to the server , We see that it returns limited information due to the fact that we are only Receptionist . 


<img width="1498" height="762" alt="image" src="https://github.com/user-attachments/assets/0da4fcb3-8e14-4c75-8651-367d8f092528" />


If we check our profile we see that our user isn't able to view some restricted clinical notes , maybe if we can escalate our privilege to one of the other roles we saw earlier we can get additional information . 


<img width="774" height="535" alt="image" src="https://github.com/user-attachments/assets/38b956c1-436e-4318-9435-e99d54916696" />


We see in our request a field for ViewerRole , we see that for now we're Receptionist , let's try to modify this to one of the other roles we found eralier in the API documentation (admin,nurse,doctor,...) . 
I tried each role on all 3 patients , and good enough the doctor role could see additional information about the patients , The flag is in the user with ID 3 .  

<img width="1508" height="736" alt="image" src="https://github.com/user-attachments/assets/5f84df0f-ab01-4196-b20b-c2046ed32112" />

The application didnt apply any sort of RBAC or checks , which lead to this IDOR . 

## Other Checks : 

Whenever we're dealing with an application that is using GraphQL , the first thing we should try is the introspective query which is a built-in GraphQL request used to discover the API's schema, revealing all available data types, fields, and operations., you can either use the GraphQL extension on Burp to generate this query :


<img width="1527" height="835" alt="image" src="https://github.com/user-attachments/assets/e1c9d785-2283-41cd-919f-fe96579b1717" />


it will automatically generate the restrospective query for us to use . 


<img width="1631" height="841" alt="image" src="https://github.com/user-attachments/assets/79d5bcf9-44ff-4054-9131-a0ba2fec9380" />


Once we do get a reponse , we can use a tool like GraphQL Voyager to turn this response into a graph that is easier to read , Just copy the response we got from the introspection query , and paste it in : 


<img width="1553" height="904" alt="image" src="https://github.com/user-attachments/assets/7015aeaa-5d5d-40c1-818f-4e265698fc28" />


This should return a Graph which link each type and field to its corresponding relationship, visually mapping out the entire data schema and its dependencies , which opens the door for other vulnerabilities . 


<img width="1578" height="817" alt="image" src="https://github.com/user-attachments/assets/ce673167-ce5d-4dde-b7d3-9cbbca2ced0f" />


Something to keep in mind whenever we're dealing with GraphQL . 









