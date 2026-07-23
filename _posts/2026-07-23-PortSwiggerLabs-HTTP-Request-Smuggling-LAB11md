---
title: " PortSwiggerlabs: HTTP request smuggling, obfuscating the TE header"
date: 2026-07-23 00:00:00 +0000
categories: [PortSwiggerLabs , HTTP Request Smuggling ]
tags: [PortSwiggerLabs , HTTP Request Smuggling , Challenge , Web_Attacks ]
---



## Information : 

This lab involves a front-end and back-end server, and the two servers handle duplicate HTTP request headers in different ways. The front-end server rejects requests that aren't using the GET or POST method.

To solve the lab, smuggle a request to the back-end server, so that the next request processed by the back-end server appears to use the method GPOST.


## Solution : 

**Quick note:**

HTTP Request Smuggling happens because of a disagreement between servers. In a typical setup you have a front-end (proxy, load balancer, CDN) forwarding requests to a back-end, and they share persistent TCP connections. The issue is that HTTP gives two ways to tell a server where a request body ends: Content-Length, which just counts bytes, and Transfer-Encoding: chunked, which sends the body in chunks ending with a zero-size chunk. When both headers are present in the same request, different servers may prioritize differently and that gap is what we exploit.

The idea is: you craft a request the front-end considers complete and forwards normally, but the back-end reads it differently and treats part of your body as leftover data sitting in the TCP buffer. That leftover then gets prepended to the next incoming request — yours or another user's. That's how you end up bypassing access controls, hijacking requests, or poisoning responses.

The three main variants are CL.TE, TE.CL, and TE.TE , basically which server trusts which header, or in the TE.TE case, how you obfuscate the Transfer-Encoding header enough that one server just ignores it entirely.

Before jumping into Burp, two things worth knowing: first, disable Update Content-Length in Repeater options , if Burp auto corrects your Content-Length you're defeating the whole point. Second, make sure you're sending over HTTP/1, not HTTP/2. When testing manually, timing delays are your main signal if the response hangs after your crafted request, the back-end is likely waiting for data that isn't coming, which means it's parsing things differently from the front-end. That's your confirmation.

Now back to the lab : 

<img width="1666" height="694" alt="image" src="https://github.com/user-attachments/assets/2a447372-f98a-46e3-a768-c5e3e0d62441" />

Now first we need to check if we have a vulnerability : 

Just like everytime we first need to find what case we've got , is it a CL.TE , TE.CL or a TE.TE . 

For this , I like to use this graph from **Jarno Timmermans** : 

<img width="1859" height="934" alt="image" src="https://github.com/user-attachments/assets/9a90be80-cfa7-4f38-921c-dffcba4b90b2" />

But you can use this guide from HackerReciepe : 

```bash
https://www.thehacker.recipes/web/config/http-request-smuggling/#practice
```

Now first let's send the detection request, that based on the response we will know if the backend and front end trust different Headers :

- First Downgrade the request to HTTP1.1/
- Then We change the request Method to POST 
- Then Remove Auto Length
- Add Transfer-Encoding: chunked
- Show Non-printable characters , this will be helpful when it comes to counting Bytes .
- In the Boy , add 

```bash
Content-Lenght: 6
Transfer-Encoding: chunked

1
123
X
```

<img width="834" height="399" alt="image" src="https://github.com/user-attachments/assets/3e1db7ac-c547-49e7-b8e5-9a0b3e0be890" />

Now if we send this , either we get an error which means the front end is definetly using TE , and since there is no terminating character we immediately get an error , and we need to figure out what the backend is using . 

Or we will get a delay , which means the front end does use CL , but the backend uses TE and since there is no terminating character , it will hang forever until we get a request time out . 

<img width="1755" height="532" alt="image" src="https://github.com/user-attachments/assets/28eed207-1d56-4656-b9d6-38e24fbaacbf" />

In this case we got an error , so we know the Front end is using TE , now let's check the backend . 

For this we will need a different payload : 

```bash
Content-Lenght: 6
Transfer-Encoding: chunked

0

X
```

First we send a terminating chunk , the 0\n\r , followed by an X , if the backend is using CL , it will be hanging for the next character , if it's using TE , it should work without any issue since we're already sending a terminating chunk . 

<img width="1798" height="633" alt="image" src="https://github.com/user-attachments/assets/191b42dc-1f06-40cd-9d72-1ca03f2f1b2b" />

In this case it retunrs a 200 OK which confirms it is actually using TE as well . So we found a TE.TE . So in this case we need to trick either the front end or the backend so that it doesn't process the TE header .

There are many ways we can do this , here is another graph from **Jarno Timmermans** that we can use : 

<img width="1837" height="1042" alt="image" src="https://github.com/user-attachments/assets/9b389cf6-0e0e-4f4b-92f5-18b0cb4c99b4" />

In our case we will use one where we add a malformed TE header so that either one of the 2 servers will not consider it and it will go with CL instead . 

```bash
POST / HTTP/1.1
Host: 0af8003d04e310fa80cbccb300160058.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked
Transfer-Encoding: NON-Existing

0

X
```

<img width="1427" height="570" alt="image" src="https://github.com/user-attachments/assets/7067b6f7-24c9-4156-9671-a30692ee11c9" />

We see that we immediately get a timeout error , this is because the Front end is more likely to accept our malformed TE header , and processes the malformed request , whereas the Backend refuses the TE completely and uses CL instead which causes a Dealy since the backend never gets the full request. 

Now to abuse this , we need to send an attack request that will get smuggled in and appended to the normal request that comes after . 

```bash
POST / HTTP/1.1
Host: 0abd009904a297df805cdad7003e00c3.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 3
Transfer-Encoding: chunked
Transfer-Encoding: AAAAA

1
A
0

```

Here we are adding the obfuscated header, then we're specifying the CL to be 3 , so that for the front end the request will end at 3 after the 1\r\n , and all the rest will be smuggled , and appended to the next request . 

<img width="1518" height="614" alt="image" src="https://github.com/user-attachments/assets/3a1ba3f0-2334-41ba-869a-86220baddadf" />

Now if we make a normal request after this one : 

<img width="1544" height="529" alt="image" src="https://github.com/user-attachments/assets/5339383f-4825-40e3-a639-b1af510277c5" />

We see that the content of our request was indeed smuggled . 

Now to solve the lab , we'll need to append a GPOST request . So the content of our smuggled request should be GPOST .... , just like a normal POST request , so we need to make it RFC compliant , for example have a line between headers and body , have body parameters inside the POST request and finally have a terminating chunk for the backend to know when the request ended since it uses TE. 

```bash
GPOST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: (Need to Speficy it) 
\r\n
x=1\r\n
0\r\n
\r\n
```

For the CL of the smuggled request , we first need to check the size of the body and whatever it is , we need to add 1 at least to it so that it appends the content of the next normal request . 

<img width="1920" height="766" alt="image" src="https://github.com/user-attachments/assets/b4ae326d-61b1-4500-aa7d-8e5e07edf5e5" />

In this case, it's 10 so the CL should be 11 or higher . We'll use 15. 

Now we need to specify the chunk size , in this case , it should be the HEX value of the smuggled request , the request ends at x=1 , we don't include the \r\n nor the terminating chunk . 

<img width="1877" height="610" alt="image" src="https://github.com/user-attachments/assets/0a494b9b-443f-493c-9b13-c751cc9a8059" />

In our case it's 5c , we just specify it so that the backend knows the size of the first request and then it will save what comes next inside the buffer . 

Now finally we need to set the CL of our request to tell the front end when our request has ended , in this case our first request should end after the 5c\r\n , so that everything after this gets smuggled . So our CL should be exactly 4 . 

The final Request should be : 

```bash
POST / HTTP/1.1
Host: 0abd009904a297df805cdad7003e00c3.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 4
Transfer-Encoding: chunked
Transfer-Encoding: AAAAA

5c
GPOST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
0

```

This is without the non-printable characters ofc , it will be added by default by Burp if you select it : 

<img width="1382" height="618" alt="image" src="https://github.com/user-attachments/assets/00576603-add5-4537-a9d3-7ddd973a3cf2" />

We just send our attack request , after that we follow it with a normal request to see if it was smuggled . 

<img width="1483" height="587" alt="image" src="https://github.com/user-attachments/assets/c5f7bf34-5e27-48f0-a5d8-cc7bc470f3d7" />

Perfect our request was smuggled , and if we go back to the lab , we see that it was solved since we were able to smuggle the GPOST request : 

<img width="1648" height="850" alt="image" src="https://github.com/user-attachments/assets/2ccfcf67-e25a-40b2-8b72-2b98a0b05b2d" />

That was all for this lab, see you in the next one :)


