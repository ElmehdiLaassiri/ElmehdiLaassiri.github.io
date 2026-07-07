---
title: " PortSwiggerlabs: HTTP/2 request smuggling via CRLF injection"
date: 2026-07-07 00:00:00 +0000
categories: [PortSwiggerLabs , HTTP Request Smuggling ]
tags: [PortSwiggerLabs , HTTP Request Smuggling , Challenge , Web_Attacks ]
---



## Information :

This lab is vulnerable to request smuggling because the front-end server downgrades HTTP/2 requests and fails to adequately sanitize incoming headers.

To solve the lab, use an HTTP/2-exclusive request smuggling vector to gain access to another user's account. The victim accesses the home page every 15 seconds.

If you're not familiar with Burp's exclusive features for HTTP/2 testing, please refer to the documentation for details on how to use them.


## Solution : 

**Quick note:**

HTTP Request Smuggling happens because of a disagreement between servers. In a typical setup you have a front-end (proxy, load balancer, CDN) forwarding requests to a back-end, and they share persistent TCP connections. The issue is that HTTP gives two ways to tell a server where a request body ends: Content-Length, which just counts bytes, and Transfer-Encoding: chunked, which sends the body in chunks ending with a zero-size chunk. When both headers are present in the same request, different servers may prioritize differently and that gap is what we exploit.

The idea is: you craft a request the front-end considers complete and forwards normally, but the back-end reads it differently and treats part of your body as leftover data sitting in the TCP buffer. That leftover then gets prepended to the next incoming request — yours or another user's. That's how you end up bypassing access controls, hijacking requests, or poisoning responses.

The three main variants are CL.TE, TE.CL, and TE.TE , basically which server trusts which header, or in the TE.TE case, how you obfuscate the Transfer-Encoding header enough that one server just ignores it entirely.

Before jumping into Burp, two things worth knowing: first, disable Update Content-Length in Repeater options , if Burp auto corrects your Content-Length you're defeating the whole point. Second, make sure you're sending over HTTP/1, not HTTP/2. When testing manually, timing delays are your main signal if the response hangs after your crafted request, the back-end is likely waiting for data that isn't coming, which means it's parsing things differently from the front-end. That's your confirmation.

Now back to our lab : 

<img width="1592" height="741" alt="image" src="https://github.com/user-attachments/assets/130b1738-cbd9-47a8-bc2e-67fbac227e27" />

Now first we need to see if we can smuggle a request : 

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

<img width="1584" height="591" alt="image" src="https://github.com/user-attachments/assets/3aeb26e8-3035-458c-8bd2-11b63c2c814d" />

We get an error , apparently if we use HTTP/1.1 the front end checks for headers present and will block it in case we had both . 

Let's try HTTP/2 instead : 

<img width="1199" height="490" alt="image" src="https://github.com/user-attachments/assets/3073ec3f-d456-456a-815b-7d3598179969" />

Well it did get through this time , this is bcs with HTTP/2 , it uses a different mechanism to determine the CL , using binary frames . 

So the idea here is that we're going to add a Transfer-Encoding: chunked and we're hoping that when the request gets downgraded to HTTP/1.1 to communicate with the backend , it will see Transfer-Encoding: chunked and if it follows the RFC it should prefer Transfer Encoding over CL and since we've already sent a terminating character, it will end the request and allow us to smuggle a new one . 

```bash
POST / HTTP/2
Host: 0a3c008703d6fb5280884e47007600d3.web-security-academy.net
Cookie: session=3dybuamJGgbv6aREx6OcuyCsUftDGs1U
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked

0

GET /404_Found HTTP/1.1
X-Ignore: x
```

Now first we send our Attack request : 

<img width="1461" height="699" alt="image" src="https://github.com/user-attachments/assets/0ab47f7d-bab7-415e-9bb6-84a01285643e" />

Then we follow it up with a normal one like we always do , and if it worked we should get a 404 : 

<img width="1419" height="654" alt="image" src="https://github.com/user-attachments/assets/268fa81a-3345-4e29-b81d-fb0967d28a44" />

But we keep getting a 200 , which is not what we want . 

This is probably due to the fact that when the downgrade is happening , it is stripping away the Transfer-Encoding . 

One way around this is to add a header, and for its value it will be equal to X then return line Transfer-Encoding , this way when the downgrade is happening , if the return line isn't stripped out , it will see it as a new Header , that way we bypass the original filter . 

```bash
POST / HTTP/2
Host: 0a3c008703d6fb5280884e47007600d3.web-security-academy.net
Cookie: session=3dybuamJGgbv6aREx6OcuyCsUftDGs1U
Content-Type: application/x-www-form-urlencoded
Test_Header: XXX
Transfer-Encoding: chunked

0

GET /404_Found HTTP/1.1
X-Ignore: x
```

<img width="1302" height="608" alt="image" src="https://github.com/user-attachments/assets/77e722da-5570-4a9a-9b2e-6d1ffcc04651" />

Now if we send it and send a normal request afterwards : 

I tried doing it like this but it didn't work so i used Inspector to add a new header and for the value we kept the same thing : 

```bash
Test_Header: XXX\r\n
Transfer-Encoding: chunked
```

<img width="1890" height="851" alt="image" src="https://github.com/user-attachments/assets/8d38aa22-5037-4ade-8f6b-baa60d75d840" />

Once we add it , it's completely normal that we can't see headers anymore since Burp doesn't know how to render them in HTTP/2 :

<img width="1873" height="810" alt="image" src="https://github.com/user-attachments/assets/236437fe-fcc2-45c1-b5bc-0e6d3d342093" />

Now once we send it , then follow it with a new request :

<img width="1380" height="598" alt="image" src="https://github.com/user-attachments/assets/8a4a83de-7e05-4436-bd09-d29ac9d8d99c" />

Perfect , we are able to smuggle our request :)

Now this lab , we're tasked with stealing other user's request headers . 

For this one , we need something that gets rendered to us , in our case it's the search feature : 

<img width="1556" height="720" alt="image" src="https://github.com/user-attachments/assets/0815f83e-d9e2-4576-9769-8d1abbb2ca8a" />

If we make a search for something it stays in the history and it is rendered back to us . 

Now the idea here is to smuggle a search request , make the CL too big that it will contain some of the other user's request as well as part of the response : 

- First let's smuggle the search request , this is how the normal search request looks like :

<img width="1277" height="735" alt="image" src="https://github.com/user-attachments/assets/ebe82db3-7982-4f0e-a908-ee0747378149" />

We won't need all headers , just the important one that we usually need and the cookie since it is what gives us the History . And of course we need it to support HTTP/1.1 , which is the case (we can attempt a normal request with HTTP/1.1 and it'll work without any issues) . 

<img width="1286" height="561" alt="image" src="https://github.com/user-attachments/assets/a6b85f4f-9f98-4aee-b3a9-86b57a463355" />

Perfect , now let's smuggle it : 

<img width="1331" height="673" alt="image" src="https://github.com/user-attachments/assets/32c525e9-624c-4b01-9f9d-50854af45d64" />

Now one thing to note here , we need the CL to be bigger than our actual request , since we're trying to include some of the other request as part of the response . 

First we check how long is an actual normal Request that we're trying to append : 

<img width="1876" height="719" alt="image" src="https://github.com/user-attachments/assets/4fc4d51f-3168-4ec0-8a26-f96d9b59da77" />

It's around 1100 , plus the 12 we had earlier , if we do 1100 we should be more than fine . As long as the CL isn't bigger than the value we should be good otherwise we might get some delays . 

<img width="1479" height="647" alt="image" src="https://github.com/user-attachments/assets/ebfad0f8-abd0-43ec-8e73-ca0495da6568" />

Now if we send a normal one , we will see that our request gets appended to the smuggled one : 

<img width="1613" height="845" alt="image" src="https://github.com/user-attachments/assets/3199b733-1091-4cc0-b7a3-8fdcb44f017b" />

We see that we are able to see our Session Cookie , this is bcs we're the ones who triggered the smuggled request by making a GET request after the Smuggled one . 

Now if we just send the attack request , and wait for the user to browse the app , it will trigger the smuggled request , which is a POST request that is rendered to us , but this time instead of returning our latest search alone , it will return it as well as the Headers inside the request of the other user , which will leak his Session Cookie . 

<img width="1369" height="723" alt="image" src="https://github.com/user-attachments/assets/05628bee-b7c0-4fd3-8c54-cac46010a010" />

Now we just send it and wait for the user to browse , should take 15 sec like they said , then we refresh the page . 

<img width="1190" height="746" alt="image" src="https://github.com/user-attachments/assets/8ed197b1-7ce3-4dd8-96c6-5c02f49c9176" />

Perfect we got the session cookie , now all we need to do is use the Devtools to inject it in our session to login as that user and solve the lab . 

<img width="1821" height="589" alt="image" src="https://github.com/user-attachments/assets/bc70440c-4e66-41a9-8d1d-13b61b945a08" />

That was all for this lab , see you in the next one :)


