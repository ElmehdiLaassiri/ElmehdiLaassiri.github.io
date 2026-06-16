---
title: "Webverselabs Salt Brook Pilates Challenge IDOR "
date: 2026-06-15 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---



## Scenario : 

Salt Brook Pilates is a founder-led studio in the Hudson Valley running reformer classes out of two storefronts. Drop-ins are $32, monthly unlimited is $260, and the booking site was rewritten last spring by a small contract team. The new profile endpoint went live without a strong-params review — the form only ever submitted four fields, so nobody worried about what else the API would accept.


## Soltion : 

<img width="1842" height="900" alt="image" src="https://github.com/user-attachments/assets/0013d32c-6e29-4a51-ac07-96c080d029b9" />

This was pretty straight forward , i will speedrun it : 

**/Classes** return a list of classes we can take , nothing useful to be fair ,  **/Instructor** contains some names , we will note them , we might need them later .  

<img width="1656" height="739" alt="image" src="https://github.com/user-attachments/assets/68b69cc5-6bbf-4659-b3a4-75423ccf50e1" />

We first start by creating our account : 

<img width="1346" height="771" alt="image" src="https://github.com/user-attachments/assets/83990764-1725-468e-b376-c9bdb3868e7c" />

Looking at the request on Burp : 

<img width="1412" height="655" alt="image" src="https://github.com/user-attachments/assets/734d7538-1f9e-4f69-9fc4-9a0baaa681fe" />

We see that there are no roles we can modify when we try to create the account . 

<img width="1608" height="857" alt="image" src="https://github.com/user-attachments/assets/fbb80eb5-b2d1-4c1a-bf9f-79023f3eca0c" />

If we try to make changes to our profile , then check Burp : 

<img width="1498" height="676" alt="image" src="https://github.com/user-attachments/assets/f487beef-ef52-4ebb-9f3f-d710ca5f2263" />

PATCH Requests are always interesting , we also see the role field in the response , we can ty to modify our role before sending the request , since this is a PATCH reqest , if there is no Access Control in place we can modify our role to admin :)

<img width="1216" height="785" alt="image" src="https://github.com/user-attachments/assets/bda213b1-f98d-41d2-8452-df0a3b28293e" />

If we forward the request : 

<img width="1705" height="775" alt="image" src="https://github.com/user-attachments/assets/ccc5353d-e90f-4fb2-a7d5-bce860ec7f30" />

We notice this **Staff** endpoint that was added after we got Admin role : 

<img width="1601" height="760" alt="image" src="https://github.com/user-attachments/assets/53fcb05c-de54-42a1-8cce-2bd8f532243a" />

If we visit that endpoint, we should be able to get our Flag . 

That was all for this challenge. See you in the next one :) 
