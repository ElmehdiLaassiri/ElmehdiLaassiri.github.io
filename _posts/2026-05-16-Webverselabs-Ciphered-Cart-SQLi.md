---
title: "Webverselabs Ciphered Cart Challenge File Upload  "
date: 2026-05-16 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Hard , Challenge , Web_Attacks ]
---

## Scenario : 

NovaStore is a direct-to-consumer skincare brand out of Portland that did roughly $14M in 2024 and got publicly embarrassed by a credential leak the year before. The remediation work was assigned by ticket count rather than risk, and the promo-code endpoint — owned by a junior who joined two weeks before the hardening sprint — got the smallest checkbox: a throttle in front of the form and a note that "the rest can wait for Q2." Q2 came and went.


## Solution : 

<img width="1633" height="999" alt="image" src="https://github.com/user-attachments/assets/750eec54-acfd-44e2-82aa-b57fde3d67e9" />


When enumeraing the application , we find 2 parameters being used , the first one is category :  


<img width="1275" height="785" alt="image" src="https://github.com/user-attachments/assets/31f1ac12-6769-46f3-a791-b7ed7af308a5" />


I tried multiple SQLi payloads on this parameter but it appears to be well secure , i even used sqlmap but nothing came out of it . 


<img width="1406" height="551" alt="image" src="https://github.com/user-attachments/assets/ae22a0fd-a68e-4b90-84a1-8868ae3e194f" />


The second parameter i found was the one used to fetch for products : 


<img width="1548" height="820" alt="image" src="https://github.com/user-attachments/assets/95d02a69-7c12-48cf-a464-66d0beaef03b" />


Same thing i tried multiple SQLi payloads , used sqlmap but didn't get anything out of it . 


<img width="1144" height="906" alt="image" src="https://github.com/user-attachments/assets/59e80251-cead-4c45-9649-2c6b974fffaf" />


If we check our Burp History ,when we click on cart , we make a POST request to /cart.php . 


<img width="1480" height="594" alt="image" src="https://github.com/user-attachments/assets/777f3f44-6c1a-4eb6-a38d-c72fc7713a9b" />


I used sqlmap again to inject inside this post request but got nothing . 


<img width="1166" height="783" alt="image" src="https://github.com/user-attachments/assets/f2c2e3b0-d720-4420-b831-492df1079e12" />


Now let's try and make a purchase , first we see the coupon field : 


<img width="1079" height="701" alt="image" src="https://github.com/user-attachments/assets/4b7f70a3-4bdb-4a1e-8cee-259c8383401d" />


If we enter anything , it gets sent as a POST request to the /apply_promo endpoint :


<img width="1220" height="526" alt="image" src="https://github.com/user-attachments/assets/5599988c-6218-4105-b4e2-58e49424b48e" />


Now let's try and use sqlmap to inject inside this request . 


<img width="1111" height="574" alt="image" src="https://github.com/user-attachments/assets/e21d02c0-4a9d-423f-b555-fc6f08301c6b" />


Perfect , the code parameter is indeed vulnerable , now let's dump the databases then tables , and get out flag . 


<img width="1147" height="851" alt="image" src="https://github.com/user-attachments/assets/22361b02-c37a-48b0-a78b-a620064296ac" />


Since it is time based , it might take a bit too long to get the flag . That's all for this challenge :)


