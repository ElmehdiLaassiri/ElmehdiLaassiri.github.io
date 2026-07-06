---
title: " PortSwiggerlabs: Exploiting HTTP request smuggling to capture other users' requests"
date: 2026-07-07 00:00:00 +0000
categories: [PortSwiggerLabs , HTTP Request Smuggling ]
tags: [PortSwiggerLabs , HTTP Request Smuggling , Challenge , Web_Attacks ]
---


## Information : 

This lab involves a front-end and back-end server, and the front-end server doesn't support chunked encoding.

To solve the lab, smuggle a request to the back-end server that causes the next user's request to be stored in the application. Then retrieve the next user's request and use the victim user's cookies to access their account.


## Solution : 

Quick note:

HTTP Request Smuggling happens because of a disagreement between servers. In a typical setup you have a front-end (proxy, load balancer, CDN) forwarding requests to a back-end, and they share persistent TCP connections. The issue is that HTTP gives two ways to tell a server where a request body ends: Content-Length, which just counts bytes, and Transfer-Encoding: chunked, which sends the body in chunks ending with a zero-size chunk. When both headers are present in the same request, different servers may prioritize differently and that gap is what we exploit.

The idea is: you craft a request the front-end considers complete and forwards normally, but the back-end reads it differently and treats part of your body as leftover data sitting in the TCP buffer. That leftover then gets prepended to the next incoming request — yours or another user's. That's how you end up bypassing access controls, hijacking requests, or poisoning responses.

The three main variants are CL.TE, TE.CL, and TE.TE , basically which server trusts which header, or in the TE.TE case, how you obfuscate the Transfer-Encoding header enough that one server just ignores it entirely.

Before jumping into Burp, two things worth knowing: first, disable Update Content-Length in Repeater options , if Burp auto corrects your Content-Length you're defeating the whole point. Second, make sure you're sending over HTTP/1, not HTTP/2. When testing manually, timing delays are your main signal if the response hangs after your crafted request, the back-end is likely waiting for data that isn't coming, which means it's parsing things differently from the front-end. That's your confirmation.

Now back to the lab:

<img width="1652" height="710" alt="image" src="https://github.com/user-attachments/assets/b6df6bbe-2a1f-491a-82e6-e590631280ec" />

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

<img width="755" height="395" alt="image" src="https://github.com/user-attachments/assets/19d9ca15-c9b9-4d0d-a402-171d8f135eea" />

Now if we send this , either we get an error which means the front end is definetly using TE , and since there is no terminating character we immediately get an error , and we need to figure out what the backend is using . 

Or we will get a delay , which means the front end does use CL , but the backend uses TE and since there is no terminating character , it will hang forever until we get a request time out . 

<img width="1402" height="489" alt="image" src="https://github.com/user-attachments/assets/df26b143-f7ff-4b57-a887-605f67654246" />

Perfect we got a request timeout which confirms this is a CL.TE vulnerability . 

Now this lab is a bit different since we are abusing the CL.TE we got to read other user's requests instead of trying to access restricted endpoints like we did in previous labs.

