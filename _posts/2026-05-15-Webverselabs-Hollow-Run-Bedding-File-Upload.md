---
title: "Webverselabs Hollow Run Bedding Challenge File Upload  "
date: 2026-05-15 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---


## Scenario : 

Hollow Run Bedding was founded in 2019 by two cousins in Bird-in-Hand, PA, after a family wedding turned into a long conversation about how every mattress on the market either sagged or smelled. They build two models — a firmer flagship and a softer companion — sell direct, and ship with a 100-night trial. The review thread on each product page fills in steadily; verified buyers post a star rating, a paragraph, and a photo of the mattress in their bedroom. The form was scoped one evening at the kitchen table.


## Solution : 


<img width="1531" height="875" alt="image" src="https://github.com/user-attachments/assets/8db4e99c-c15f-45e8-b07a-a1af67abc5d0" />


We first need to create an account that we can use to enumerate the application further . 


<img width="1582" height="907" alt="image" src="https://github.com/user-attachments/assets/170e60ee-0348-4806-9fe4-e3ef7544a525" />


We see that we can leave reviews on different products . 


<img width="1521" height="874" alt="image" src="https://github.com/user-attachments/assets/9a3d9c8e-92f0-424f-8bc5-d86f0cc36710" />


The Shell.jpg contains our Webshell ! 


<img width="952" height="430" alt="image" src="https://github.com/user-attachments/assets/d1f15464-1dcc-408f-b52c-032294e67fe1" />


We first upload it as an image , then intercept it using burp and modify the extension to php to see if we can bypass the Frontend validation .


<img width="1469" height="790" alt="image" src="https://github.com/user-attachments/assets/9d1947e6-9a6a-4b01-aabb-d22adf96e2d4" />


We see that our Shell is uploaded without any issue , now we just need to know where it is stored to be able to interact with it , From Burp History we do see a GET request for the newly created review : 



<img width="1434" height="755" alt="image" src="https://github.com/user-attachments/assets/898a2762-93f1-4f96-a422-68876c6ff81d" />


perfect , now we just need to visit the /reviews/13-shell.php endpoint to interact with our Web Shell . 


<img width="1050" height="331" alt="image" src="https://github.com/user-attachments/assets/731759f7-a5c0-4214-844b-5e9dfe401b7b" />


Perfect , the flag is always located at /flag.txt . 


<img width="1061" height="280" alt="image" src="https://github.com/user-attachments/assets/5aca0b59-e011-426f-bfeb-50db3ec53fa9" />


That's all for this challenge :) 





