---
title: "Webverselabs Overrun Challenge IDOR "
date: 2026-06-13 21:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Hard , Challenge , Web_Attacks ]
---


## Scenario : 

Overrun's contract change-order module shipped two sprints ago. The endpoint that opens a single change order is mounted under a project route — and the middleware on that route only checks that the calling PM owns the project. The handler then loads the change order by its global record id without re-checking that the change order actually belongs to that project. The bug has been live for weeks. The internal notes on a confidential change order at Harbour Point are about to be read by the wrong people.


## Solution : 


<img width="1842" height="797" alt="image" src="https://github.com/user-attachments/assets/c9ec6221-6474-416c-8b4e-cc324808f662" />

We first start by creating an account . Then browse the application just like a normal user, check all endpoints , features , etc. Then we check Burp's History to get details about the requests sent to the server . 

We're mainly looking for parameters we can tamper with or inject in , wether it's a GET or POST Request . 

**/Dashboard :**

<img width="1493" height="875" alt="image" src="https://github.com/user-attachments/assets/4ae66673-b028-4343-bb65-1c43f76ce66f" />

The dashboard gives general information about our porftolia , like the Budget; Variance, Let's check other endpoints . 

**/Site Log :**

<img width="1451" height="807" alt="image" src="https://github.com/user-attachments/assets/f6afc52a-2733-412f-a04e-c10bd706da58" />

This allows us to view projects and sort them based on Projects, date,etc. But nothing really useful for us here . 

**/Reports :**

<img width="1456" height="834" alt="image" src="https://github.com/user-attachments/assets/4f9839e7-755e-4756-9e74-333cd8c4ca97" />

This endpoint allows us to generate reports, as well as view the old ones . Tried submiting a report but looking at the Burp Request, we can't really do much with it. 

<img width="1560" height="730" alt="image" src="https://github.com/user-attachments/assets/56adfca8-1312-42c5-9e78-d21629940f2e" />

**/Audit Log :**

This is a list of Reports for different projects , just a static web page , nowhere to go from here . 

Now if we go back to the **Dashboard** endpoint, we can enumerate the features further : 

<img width="1407" height="618" alt="image" src="https://github.com/user-attachments/assets/d3f0d74f-3826-41e1-9ed5-ac893e0d6465" />

Now first one : If we try **Open Project** :

<img width="1536" height="807" alt="image" src="https://github.com/user-attachments/assets/68a14bf9-bcb4-4300-ab46-7e489cafb6ae" />

I tried modifying the ID to see if we can access any other document .But that just redirects us back to the Dashboard . 

<img width="1406" height="866" alt="image" src="https://github.com/user-attachments/assets/ebcee0ca-1aa0-4227-be29-dcd3ac6e54ad" />

Now let's take a look at **Change Order Botton** : 

<img width="1577" height="706" alt="image" src="https://github.com/user-attachments/assets/0e77efce-cc4d-4d6d-b4e5-d515ffcbc8a6" />

First i tried doing is modifying the id in the URL to something else, maybe we can modify the order of other documents . 

<img width="1066" height="220" alt="image" src="https://github.com/user-attachments/assets/561a2bfe-7346-4306-a86d-f831cc6cc57d" />

If we change it from 3 to 1 we get access denied . 

Now i tried FUZZING for all numbers that won't returns access denied . For this i used FFUF . 

<img width="1121" height="603" alt="image" src="https://github.com/user-attachments/assets/9b0c416f-4da2-41fc-9b7f-a8707303b0e2" />

We see that all of them return a 302 , with Access denied . 

Now back to our **Change Order Endpoint** :

<img width="1638" height="658" alt="image" src="https://github.com/user-attachments/assets/abf6682c-351a-48a4-9534-42a6ccdd60f1" />

We can try and open any of the projects below . 

<img width="1428" height="598" alt="image" src="https://github.com/user-attachments/assets/3ec2e7b4-64f6-4565-a975-36cd767710dd" />

Nice a new ID we can modify :) 

<img width="1149" height="310" alt="image" src="https://github.com/user-attachments/assets/afd1e30f-be4c-49e9-bb77-dc3037fdee9a" />

If we modify it we either get a change order not found or a different Report . 

We can FUZZ again and filter the response for *'Change order not found.'*

<img width="1718" height="845" alt="image" src="https://github.com/user-attachments/assets/db167531-df4c-41ad-9702-27d3f8fb3b0d" />

```bash
ffuf -u https://a36dfc57-4327-overrun-1286f.challenges.webverselabs-pro.com/projects/3/change-orders/FUZZ -w Numbers -H "Cookie: rack.session=Kofy%2FFXbcr2fbz%2F6ZtuIDgzBLYZ3uHEKDQbnfGUC0%2BGduabjqFDtM1ULvK8GOWtQnbDTNucAblsIt0rZapfm6GCT%2BBIZRHYUzEXNjoguXve7VRzuFKT4LdwNOOBewuy62W%2FEuAM3vA9H65Nr%2Bx7kMVV5qsLY4l%2ByLiEipr9Pna%2BRunKWxHmdIVLIeqXXuRnDhFA6vMYlJHD8guHzjH%2BNEnSuj6si%2BxDf8Jc9V9KRneDegyhRyQqgHVV%2FiGVwfE9C41SghqRJM9Ku3QpBjlNa02I19ULg4R1p8DmGoeO5BvhgbNxplIFnlgIIWzPNb6gbONFUI6Kt86odxorAyPqp0ZHnTIFyGbKHEt2wFmzAN%2BH443aMXExGA2f0gVav9hmPNgou7bflbvDv9qIlv966UeSN84yFuHoFuIZr0juocrJVbeCBbFie8AqQ1%2FpDrN7TtwId8Rqx4DAYjtlvxQc4z3q1OBfEMka6UgHlbqeTpsQdwRYGPYPKA3i0mg%3D%3D--TARajkSX5Atsk7fv--atUal1F3w00xLBVjRSdSog%3D%3D"  -H "User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36"  -H "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7" -fc 302
```
Make sure you add the Cookie,User agent, accept,etc. 

We see that 4 returns more Word than the rest, hopefully it's the flag :) 

<img width="1498" height="710" alt="image" src="https://github.com/user-attachments/assets/9a9b0946-88d3-4386-921f-507e9053ac90" />

Perfect we got our flag .

That was all for this challenge. See you in the next one :) 



