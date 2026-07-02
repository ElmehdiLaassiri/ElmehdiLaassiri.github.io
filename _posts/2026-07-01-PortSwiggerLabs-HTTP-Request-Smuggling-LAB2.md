---
title: " PortSwiggerlabs: HTTP request smuggling, confirming a TE.CL vulnerability via differential responses"
date: 2026-07-01 00:00:00 +0000
categories: [PortSwiggerLabs , HTTP Request Smuggling ]
tags: [PortSwiggerLabs , HTTP Request Smuggling , Challenge , Web_Attacks ]
---


## Information : 

This lab involves a front-end and back-end server, and the back-end server doesn't support chunked encoding.

To solve the lab, smuggle a request to the back-end server, so that a subsequent request for / (the web root) triggers a 404 Not Found response.


## Solution : 

Quick note:

HTTP Request Smuggling happens because of a disagreement between servers. In a typical setup you have a front-end (proxy, load balancer, CDN) forwarding requests to a back-end, and they share persistent TCP connections. The issue is that HTTP gives two ways to tell a server where a request body ends: Content-Length, which just counts bytes, and Transfer-Encoding: chunked, which sends the body in chunks ending with a zero-size chunk. When both headers are present in the same request, different servers may prioritize differently and that gap is what we exploit.

The idea is: you craft a request the front-end considers complete and forwards normally, but the back-end reads it differently and treats part of your body as leftover data sitting in the TCP buffer. That leftover then gets prepended to the next incoming request — yours or another user's. That's how you end up bypassing access controls, hijacking requests, or poisoning responses.

The three main variants are CL.TE, TE.CL, and TE.TE , basically which server trusts which header, or in the TE.TE case, how you obfuscate the Transfer-Encoding header enough that one server just ignores it entirely.

Before jumping into Burp, two things worth knowing: first, disable Update Content-Length in Repeater options , if Burp auto corrects your Content-Length you're defeating the whole point. Second, make sure you're sending over HTTP/1, not HTTP/2. When testing manually, timing delays are your main signal if the response hangs after your crafted request, the back-end is likely waiting for data that isn't coming, which means it's parsing things differently from the front-end. That's your confirmation.

Now back to the lab:

<img width="1697" height="865" alt="image" src="https://github.com/user-attachments/assets/22ce30da-6ab6-45b4-adda-aad719a5c488" />

**Detection :**

Now the first thing we should do is send a specific request and based on the response we will know whih case we're in , is it a CL.TE , TE.CL or TE.TE . 

For the specific request either use the one provided by HackerReciepe : 

```bash
https://www.thehacker.recipes/web/config/http-request-smuggling/#practice
```

Personally i like to follow this graph from **Jarno Timmermans** 

<img width="1748" height="915" alt="image" src="https://github.com/user-attachments/assets/0cc2c00a-4071-4bff-86e7-15b42e093c62" />

They both follow the same principle , we first send a request and check whether we get a Delay, Rejection ,etc . 

```bash
Content-Lenght: 6
Transfer-Encoding: chunked
\r\n
1\r\n
123\r\n
X\r\n
# The \r\n are just line separators (We can easily show them using burp) . 
```

Now looking at the graph , in case the front end is using Chunked transfer encoding , well since there is no 0 to end the request it will immediately reject it , but we don't know still whether the backend is using TE , or CL , since the request was rejected before it even reaches the backend . 

This is when we use the second payload on the graph : 

```bash
Content-Lenght: 6
Transfer-Encoding: chunked
\r\n
1\r\n
0\r\n
X
```

In case we send this one , and get a delay , this means the backend is using Content length (since it is waiting for the 6 characters , but it only gets the 5 and it hangs waiting for the rest , but the rest is never sent since for the front end the request has already ended (since it got the terminating 0 ). 

And since the front end uses TE , and the backend uses CL , this is an indication that the server is vulnerable to a TE.CL attack . 

**More explanation :** 

The first request we sent had *Transfer-Encoding: chunked*  But with no 0\r\n to terminate the request , so in case the front end Rejects it , this proves that it uses TE . 

Now to Determine the backend we send a different request , this time we send a request with 6 bytes , but we put a terminating character at the 5th byte , so that only 5 bytes are sent to the backend .

Here the front end is working since it got the terminating character (0\r\n) , and for the backend , in case it was using CL , it will cause a delay in the response , since it's expecting 6 bytes to consider that the request has ended , but instead it will only get 5 since that's what the front end sent , and from there it will keep waiting for the last byte which is never sent , which causes the delay .  

This is exactly what TE.CL attack is , front end uses TE while backend uses CL to validate whether the request has ended . 

**Confirmation/Exploitation :**

We can use Differential responses , the concept is always the same : 

- We have an Attack Request , in this one we will sepecify whether the CL or TE to be just enough for it to pass the Front end . 
- Once it reaches the backend, the request is well crafted so that the backend takes in a part of the request and store the rest in the buffer . 
- From there we send a normal request which will be appended to what was inside the buffer , so we're kinda injecting our old attack request into the new one that comes in , hence the name smuggling.

In our case , the TE.CL can be explained using this graph from **Jarno Timmermans** again : 

<img width="1667" height="1008" alt="image" src="https://github.com/user-attachments/assets/051dacc9-f420-4ced-9a8f-c11c93d3b71f" />

The orange part in the graph is exactly what was stored in the buffer when sending the first request . Since we know the server uses CL , we can specify it to be 4 , and from there have the rest of the request stored in the buffer so that it gets appended to the next request (normal request).

- The appended request has 15 in the CL , while the entire request stored in the buffer has 10 bytes,  which means only 5 bytes of the normal request will be appended , that's exactly "GET /"  and the rest will be ignored . 

*Don't worry if this doesn't make sense to you yet, the practical example should make things clear* :

**Practical Example :**

*Detection* : 

This is the original request : 

<img width="1741" height="699" alt="image" src="https://github.com/user-attachments/assets/f2bc34e2-01cd-47c3-a766-aab2936cfae9" />

As we said before we first send the detection payload : 

- First we downgrade it to HTTP/1.1 .
- Add Transfer-Encoding : Chunked . 
- Show Non printable characters, this will be needed to count the number of Bytes.
- Delete unneccessary Headers .
- Disable Auto Content-Length . 
- Change Method to POST .
- Add the Paylod inside the POST Body :

```bash
POST / HTTP/1.1
Host: 0a3e004c036d22cc80ebdae70069001a.web-security-academy.net
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Content-Lenght: 6
Transfer-Encoding: chunked

1
123
X
```

<img width="745" height="441" alt="image" src="https://github.com/user-attachments/assets/ae79d004-7066-4b0a-a5aa-bd0247f08c2a" />

Now if we send it : 

<img width="1845" height="695" alt="image" src="https://github.com/user-attachments/assets/b0f5aad9-c00a-4877-a0d8-899f1dd3ea65" />

We immediately get an error , this proves that the front end uses TE chunked , since there is no terminating character we got an error . 

Now to determine what the Backend uses : 

```bash
POST / HTTP/1.1
Host: 0a3e004c036d22cc80ebdae70069001a.web-security-academy.net
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Content-Lenght: 6
Transfer-Encoding: chunked

0

X
```

<img width="750" height="418" alt="image" src="https://github.com/user-attachments/assets/abd9d361-6f54-4322-ac57-a4245926c1d6" />

Notice how we're not adding a return line after the X , this is very important . Now if we send it : 

<img width="1834" height="633" alt="image" src="https://github.com/user-attachments/assets/e62cc8a4-48ec-4aa3-9ba7-bbd1ceeae001" />

We see that we're getting a delay . Since the backend is waiting for the 6 bytes , for now it has only recieved 5 bytes : 

```bash
0\r\n
\r\n
```

but for the front end the request has already ended after the terminating character (0\r\n) so it will drop the rest and the X is never sent to the backend server , so to confirm that the backend uses CL we should get a delay since X it is dropped, the backend never gets the 6th byte which causes a delay so this confirms that the backend is using CL not TE . And eventually we will get a time out error : 

<img width="1475" height="514" alt="image" src="https://github.com/user-attachments/assets/74854646-bcae-43cd-9058-13740c3d3c42" />

Perfect now onto the confirmation phase using a differential attack : 

*Confirmation :*

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
```

<img width="1898" height="628" alt="image" src="https://github.com/user-attachments/assets/710242ba-0dd8-4723-a2cf-a609464b600b" />

Now we need to specify the Content Length , then since we want the Front end to forward the entire request, we will add the terminating character at the end : (0\r\n). 

For the Content Length of the smuggled request , if we select the content of our body, we see that we have 10 Bytes : 

```bash
x=1\r\n
0\r\n
\r\n
```
<img width="1906" height="706" alt="image" src="https://github.com/user-attachments/assets/e3a4f548-72d4-4672-b0be-9147838d5705" />

We need the CL to be bigger than the actual value of our POST Request , since the backend expects a number of CL to know when the request ended, if we add more Bytes this will append the content of the Next request as well . So we just need the CL to be bigger than 10 , we'll put 15 for this one . 

Now for the Chunk Size , this will go from the start of the Body (POST /Throws404....) until the line before the terminating character without adding the return line character (Without adding the \r\n )   

<img width="1892" height="767" alt="image" src="https://github.com/user-attachments/assets/4e509092-ba6b-4860-a53c-22c1c6420142" />

We see on Inspector that it is 165 Bytes , and the HEX value is 0xa5 so we will replace the Chunked size with a5 . 

Now for the Final Content Length for the entire Request , since the server uses CL , we want it to only process the Bytes before the second request , so in our case just the characters before POST /404 ....

```bash
a5\r\n
```

In this case it's only 4 bytes, and the rest we want it to be stored in the Buffer so that it gets appended to the next normal request . 

<img width="1902" height="652" alt="image" src="https://github.com/user-attachments/assets/a0272beb-1055-46be-bcfc-10ff5cdcbcf1" />

So the Final request will look like this :

```bash
POST / HTTP/1.1
Host: 0ae800c703d22723817966ca004f0089.web-security-academy.net
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Trasnfer-Encoding: chunked
Content-Length: 4

a5
POST /throws404 HTTP/1.1
Host: 0ae800c703d22723817966ca004f0089.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
0

```

<img width="786" height="522" alt="image" src="https://github.com/user-attachments/assets/f592edbe-c16c-4577-9809-3b33feec3976" />

Now we just send it and we should get a 200 :

<img width="1522" height="835" alt="image" src="https://github.com/user-attachments/assets/7a02dfc1-1f33-46f0-a105-d6630886baff" />

Now the backend only processes the first 4 bytes and keep the rest in the Buffer which means the Buffer contains the :

```bash
POST /throws404 HTTP/1.1
Host: 0a13001804b3697b81469862004800c2.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
0
````

So next time we make a normal request , it should be appended to the request inside the Buffer and this should return a 404 : 

<img width="1586" height="715" alt="image" src="https://github.com/user-attachments/assets/ad0eb3dc-e400-42cb-aae0-fefb58dadcf3" />

Perfect , we got our Error . 

Now if we refresh the lab , we should see that it was solved : 

<img width="1714" height="875" alt="image" src="https://github.com/user-attachments/assets/645896d7-8475-4ee9-b7d1-f852e4b336f1" />

That was all for this lab, hope i explained the attack in a clear way, if you have any feedbacks please feel free to reach out, see you in the next one :) 
