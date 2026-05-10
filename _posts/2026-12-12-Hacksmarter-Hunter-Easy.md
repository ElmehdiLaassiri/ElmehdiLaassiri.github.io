title: "HackSmarter Hunter Walkthrough (Easy) "
date: 2025-12-11 00:00:00 +0000
categories: [ HackSmarter]
tags: [Hacksmarter , Web_Challenge , Easy]
---

# Summary :

> Hunter is a an easy challenge from HackSmater , we're given a list of usernames that the OSINT team got and we should try to enumerate valid usernames on an enterprise portal , the protal seems pretty secure at first as it doesn’t return any information that might be used to filter for and enumerate users , But the Password reset feature has a flaw ; the time it takes for requests are different based on whether that user exists or not , so we will be abusing this to get a list of valid usernames . Baisically Time based user enumeration if you want to call it that :)
>



# Solution : 

If we navigate to the URL, we find a simple login page. The first thing I did was fuzz for other endpoints. Fuzzing returns only these 2 endpoints.

<img width="1603" height="689" alt="image" src="https://github.com/user-attachments/assets/3ce183ff-2730-4a3b-90bc-d20ec15a0b40" />

I tried fuzzing both of them for other directories, files, etc, but nothing was found.

<img width="983" height="716" alt="image" src="https://github.com/user-attachments/assets/1599bf77-1e7e-45b1-86ea-dfa4120572a3" />

If we try to log in, we get no response from the server , no information to try and filter for.

<img width="1018" height="704" alt="image" src="https://github.com/user-attachments/assets/ee5bd11f-4162-4aa0-b303-2fa26d0999fe" />

Now let's open Burp and make some login requests to see what the request and response look like.

<img width="1258" height="432" alt="image" src="https://github.com/user-attachments/assets/832060b4-c23a-4bde-b57b-bc5050d80d2c" />

Now let's craft the same request using FFUF and filter the response to see if any of the usernames return a different response.

<img width="1222" height="724" alt="image" src="https://github.com/user-attachments/assets/044285f0-a7c8-456d-af3d-4ef0068bf1af" />

This doesn't return anything useful. Now, remember the Password Reset page , we can try the same thing on that endpoint as well.

<img width="1184" height="749" alt="image" src="https://github.com/user-attachments/assets/6d316baf-f352-478a-8b06-bb205d08f9f1" />

Nothing useful there either. It seems pretty secure. One last thing to test: when it comes to password reset functionalities, sometimes the server takes longer to respond for users that actually exist, as it will attempt to send the OTP or reset link. Since our application doesn't implement any sort of rate limiting, we can keep fuzzing and look for responses that take longer than the rest.
And we do find a response that takes 1000ms compared to the 100–200ms range we saw earlier — this is a strong indicator that this user exists. 

<img width="1457" height="572" alt="image" src="https://github.com/user-attachments/assets/ff9d0004-3730-44fc-a7b4-c28500a54a09" />

Just like that, we are able to enumerate valid usernames even when the web app doesn't return any useful information.







