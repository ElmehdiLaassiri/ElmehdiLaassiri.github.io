---
title: " PortSwiggerlabs: HTTP request smuggling, confirming a CL.TE vulnerability via differential responses"
date: 2026-06-30 00:00:00 +0000
categories: [PortSwiggerLabs , HTTP Request Smuggling ]
tags: [PortSwiggerLabs , HTTP Request Smuggling , Challenge , Web_Attacks ]
---


## Information :

This lab involves a front-end and back-end server, and the front-end server doesn't support chunked encoding.

To solve the lab, smuggle a request to the back-end server, so that a subsequent request for / (the web root) triggers a 404 Not Found response.

Note :

Although the lab supports HTTP/2, the intended solution requires techniques that are only possible in HTTP/1. You can manually switch protocols in Burp Repeater from the Request attributes section of the Inspector panel.


## Solution : 

Quick note:

HTTP Request Smuggling happens because of a disagreement between servers. In a typical setup you have a front-end (proxy, load balancer, CDN) forwarding requests to a back-end, and they share persistent TCP connections. The issue is that HTTP gives two ways to tell a server where a request body ends: Content-Length, which just counts bytes, and Transfer-Encoding: chunked, which sends the body in chunks ending with a zero-size chunk. When both headers are present in the same request, different servers may prioritize differently and that gap is what we exploit.

The idea is: you craft a request the front-end considers complete and forwards normally, but the back-end reads it differently and treats part of your body as leftover data sitting in the TCP buffer. That leftover then gets prepended to the next incoming request — yours or another user's. That's how you end up bypassing access controls, hijacking requests, or poisoning responses.

The three main variants are CL.TE, TE.CL, and TE.TE , basically which server trusts which header, or in the TE.TE case, how you obfuscate the Transfer-Encoding header enough that one server just ignores it entirely.

Before jumping into Burp, two things worth knowing: first, disable Update Content-Length in Repeater options , if Burp auto corrects your Content-Length you're defeating the whole point. Second, make sure you're sending over HTTP/1, not HTTP/2. When testing manually, timing delays are your main signal if the response hangs after your crafted request, the back-end is likely waiting for data that isn't coming, which means it's parsing things differently from the front-end. That's your confirmation.

Now back to the lab:

<img width="1714" height="869" alt="image" src="https://github.com/user-attachments/assets/d5a2b0fe-4502-49c4-9b84-e8784c1b9172" />

Now first thing first , we go to Burp , send our request to Repeater to start modifying it : 

<img width="1613" height="768" alt="image" src="https://github.com/user-attachments/assets/0a6bf106-45fc-4b40-9473-3d718ca66033" />


### Detecting CL.TE : 

Now to detect the vulnerability i will follow this graph from **Jarno Timmermans** : 

<img width="1859" height="934" alt="image" src="https://github.com/user-attachments/assets/9a90be80-cfa7-4f38-921c-dffcba4b90b2" />

Why do we get a delay ? 

Let's say we're sending this payload : 

```bash
Content-Lenght: 6
Transfer-Encoding: chunked
\r\n
1\r\n
123\r\n
X\r\n
# The \r\n are just line separators (We can easily show them using burp) . 
```

Now if the Front end uses Content Length (CL) : it will only transfer 1\r\n123 And stop , the rest : \r\nX\r\n will be ignored . 

From there if the Backend Server uses Transfer-Encoding (TE) , it will wait for the 0 (the terminating chunk) inside the request body to know that when the request ends . But it won't recieve anything , that's why we get a timeout .

So if we ever get a delay this is a huge indicator that we got a CL/TE vulnerability 

### Confirming CL.TE : 

Now to confirm the Vulnerability i will use this amazing graph from **Jarno Timmermans** that summarizes the attack perfectly , this is what i used to understand the attack . 

<img width="1748" height="915" alt="image" src="https://github.com/user-attachments/assets/0cc2c00a-4071-4bff-86e7-15b42e093c62" />

We first send an attack Request , which will be a POST Request containing our payload : 

```bash
POST / HTTP/1.1
Content-Length: 41
Transfer-Encoding: chunked 

0\r\n
\r\n
GET /Throws404 HTTP /1.1\r\n
X-Ignore: X
```

Now since the Front end uses CL , it will send the entire request , but Since the Backend uses TE , the moment it sees 0 it will stop since it will consider the request to have ended , so the rest will be stored in the server's Buffer . 

<img width="1226" height="639" alt="image" src="https://github.com/user-attachments/assets/512012d7-9c9c-4e56-85e7-52088d236e5c" />

```bash
GET /Throws404 HTTP /1.1\r\n
X-Ignore: X
```

Now if we make a new request , just a normal one that should get a 200 : 

```bash
GET / HTTP/1.1\r\n
Host: lab.....
\r\n
```
Once it reaches the backend it will look like this : 


```bash
GET /Throws404 HTTP /1.1\r\n
X-Ignore: XGET / HTTP/1.1\r\n
Host: lab.....
\r\n
```

Which will return a 404 instead confirming that we were able to smuggle another request when we sent our Attack Request . 

Now one important thing to note is that We must not have a return line after the "X-Ignore: X", otherwise it will look like this : 

<img width="1226" height="639" alt="image" src="https://github.com/user-attachments/assets/7dfbfc7b-1346-45a3-9d30-1c5660321b5c" />

```bash
GET /Throws404 HTTP /1.1\r\n
X-Ignore: X
GET / HTTP/1.1\r\n
Host: lab.....
\r\n
```

Which will be treated like 2 different GET Requests. 

Now to test this , we first go to Repeater , then from Inspector we change the request attribute to HTTP/1.1 : 

<img width="1914" height="733" alt="image" src="https://github.com/user-attachments/assets/2e3e4e1b-af31-4f28-bb36-c2817fa1737d" />

Then we send it to confirm that the server accepts HTTP/1.1 :

<img width="1383" height="598" alt="image" src="https://github.com/user-attachments/assets/bffb04c9-3656-4a6f-a3b9-4c45d9a12d6e" />

From there we change the request method : it will change it to POST : 

<img width="1474" height="670" alt="image" src="https://github.com/user-attachments/assets/f5c1e11f-e7e3-48b1-9134-b6bed147ff2f" />

Then we delete the unecessary Headers , we only need the HOST , Content-Type and content-Length , then we remove the hidden non printable characters to see our return lines (\r\n)

<img width="1211" height="406" alt="image" src="https://github.com/user-attachments/assets/165b38df-6dc6-46dd-9662-f2a6570df223" />

From there we remove the update content-length since that's what we're exploiting : 

<img width="1282" height="510" alt="image" src="https://github.com/user-attachments/assets/020b8037-3645-4984-b942-45558a7b2938" />

We're putting it at 6 so that it never Forwards the X to the backend . 

Now if we send the request : 

<img width="1916" height="821" alt="image" src="https://github.com/user-attachments/assets/a0d6d837-87fb-4113-aab0-d36e31582f98" />

We see that the request took way too long and eventually we get a request time out since the backend still expects the 0 to end the request . 

Perfect this does confirm the vulnerability , now what we should do is send a new request containing the endpoint we want to store in the buffer , this time we don't care about the length since we're looking to end the request with a terminating character (0) on the backend before we try to make any request . 

In other words , what we're doing is sending a new request where we add this inside the body : 

```bash
GET /Non_Existing HTTP /1.1\r\n
X-Ignore: X
```

But before we need to tell the server that the request ended before sending our new one , we do that by adding the 0 before this , so the command should be : 

```bash
POST / HTTP/1.1
Host: 0a49005304a3846d80dde979003b00b9.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 7
Transfer-Encoding: chunked

0
GET /Non_Existing HTTP /1.1\r\n
X-Ignore: X
```

Now 2 things to note here , we can't add line returns after the X-Ignore like we mentionned before , and for content lenght let's re enable auto update length :

<img width="1516" height="658" alt="image" src="https://github.com/user-attachments/assets/a92d4137-4f4b-4dc3-aeb9-343c0abda62f" />

Perfect , now to confirm that our new request for /Non_Existing is indeed in the buffer , if we request a normal valid endpoint , we should get an error . 

Since it will automatically append it to the /Non_Existing that we had smuggled earlier : 

<img width="1486" height="605" alt="image" src="https://github.com/user-attachments/assets/b8ddfae4-95d0-4144-9ec3-88cfc77a73e8" />

Perfect we get a Not Found as expected . 

If we send it again , it should work just fine : 

<img width="1486" height="605" alt="image" src="https://github.com/user-attachments/assets/518dbbee-fb7a-4d0e-b9a7-a934b160c271" />

Now finally if we go back to the lab , we should see that it was solved . 

<img width="1672" height="867" alt="image" src="https://github.com/user-attachments/assets/3b076d19-d837-4e60-881d-35efeb91583e" />

That was all for this lab , see you in the next one :)
