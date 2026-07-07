---
title: " PortSwiggerlabs: Response queue poisoning via H2.TE request smuggling"
date: 2026-07-07 00:00:00 +0000
categories: [PortSwiggerLabs , HTTP Request Smuggling ]
tags: [PortSwiggerLabs , HTTP Request Smuggling , Challenge , Web_Attacks ]
---


## Information : 

This lab is vulnerable to request smuggling because the front-end server downgrades HTTP/2 requests even if they have an ambiguous length.

To solve the lab, delete the user carlos by using response queue poisoning to break into the admin panel at /admin. An admin user will log in approximately every 15 seconds.

The connection to the back-end is reset every 10 requests, so don't worry if you get it into a bad state - just send a few normal requests to get a fresh connection.


## Solution : 

Quick note:

HTTP Request Smuggling happens because of a disagreement between servers. In a typical setup you have a front-end (proxy, load balancer, CDN) forwarding requests to a back-end, and they share persistent TCP connections. The issue is that HTTP gives two ways to tell a server where a request body ends: Content-Length, which just counts bytes, and Transfer-Encoding: chunked, which sends the body in chunks ending with a zero-size chunk. When both headers are present in the same request, different servers may prioritize differently and that gap is what we exploit.

The idea is: you craft a request the front-end considers complete and forwards normally, but the back-end reads it differently and treats part of your body as leftover data sitting in the TCP buffer. That leftover then gets prepended to the next incoming request — yours or another user's. That's how you end up bypassing access controls, hijacking requests, or poisoning responses.

The three main variants are CL.TE, TE.CL, and TE.TE , basically which server trusts which header, or in the TE.TE case, how you obfuscate the Transfer-Encoding header enough that one server just ignores it entirely.

Before jumping into Burp, two things worth knowing: first, disable Update Content-Length in Repeater options , if Burp auto corrects your Content-Length you're defeating the whole point. Second, make sure you're sending over HTTP/1, not HTTP/2. When testing manually, timing delays are your main signal if the response hangs after your crafted request, the back-end is likely waiting for data that isn't coming, which means it's parsing things differently from the front-end. That's your confirmation.

Now back to the lab:

<img width="1596" height="693" alt="image" src="https://github.com/user-attachments/assets/e3fc1067-1982-448d-9347-f345baa15e66" />

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

<img width="1427" height="533" alt="image" src="https://github.com/user-attachments/assets/157e60d4-e182-4625-b200-913c3e9045f0" />

We get a new error , this time it didn't accept both of them at the same time .

This is because even when we downgraded to HTTP/1.1 it still used HTTP/2 to communicate with the backend — so the classic approach won't work here. This is a different variant called H2.TE.

**Quick note on H2.TE:**

The setup here is slightly different — the front-end accepts HTTP/2 but downgrades to HTTP/1.1 when talking to the back-end. Normally HTTP/2 doesn't care about Transfer-Encoding since it has its own binary framing, but if you manually inject a TE: chunked header inside the HTTP/2 request, the front-end just ignores it — but when it downgrades, that header comes along for the ride and the back-end sees it and starts parsing the body as chunked. From there it's the same idea, leftover sits in the buffer, next request gets poisoned.

Now back to the lab : 

One thing i tried is to obfuscate the header , that way it might give us a 200 instead : 

<img width="1462" height="660" alt="image" src="https://github.com/user-attachments/assets/bbb11a8a-fe68-4a4c-abd3-8d08eba147cc" />

And it worked . 

But for this lab we don't need to go through all of that — since the back-end follows RFC 7230 , it will always prefer Transfer-Encoding over Content-Length when both are present , so we only need to inject the TE header alone and don't need to bother with Content-Length at all . And since this is H2.TE , the front-end speaks HTTP/2 so it doesn't even look at the TE header in the first place, no obfuscation needed , no risk of triggering that error . The HTTP/2 framing handles the length for the front-end , and the injected TE header takes care of the back-end after the downgrade .

```bash
POST / HTTP/1.1
Host: 0a84002804ef05af81241bac00930033.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: xchunked

0

```

Now why are we keeping the Trasnfer encoding in particular , here we're expecting that if it has to downgrade to HTTP/1.1 , it will copy the transfer Encoding header when rewriting the HTTP request .  And if the backend follows the RFC (which should be the case) it will see the transfer encoding header and prefer it over the CL . 

And this is exactly how we get our HTTP Request Smuggling , by having a front end that uses CL with HTTP2 and a backend that uses Transfer encoding via HTTP/1.1 . 

And just like with a normal CL.TE , we termminate the request with a terminating character , from there we poison the buffer with a smuggled request : 

```bash
POST / HTTP/2
Host: 0a84002804ef05af81241bac00930033.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked

0

GET /404_Return HTTP/1.1
X-Ignore: x
```

- Here we're keeping the HTTP2 since it's an H2.CE .
- Make sure you don't add a return line after the x since that's where we're appedning the normal request .
- Keep Update Content Length disabled .
  
<img width="1500" height="611" alt="image" src="https://github.com/user-attachments/assets/8248078a-a291-4054-9d71-f393d6a28847" />

Perfect , now if we send a normal request : 

<img width="1441" height="507" alt="image" src="https://github.com/user-attachments/assets/d78c6dc1-678e-412a-b6cd-54d4046639b9" />

We got our 404 which means the request was smuggled . 

Now this lab is a bit different , we're trying to poison the Queue for the different responses , we can do that by smuggling a complete GET request instead of what we're used to smuggling . 

So it will look something like this : GET Request --> Complete Smuggled GET Request --> Normal Request + Smuggled Request : 

Now this lab is a bit different , we're trying to poison the response queue . We can do that by smuggling a complete GET request instead of just a partial prefix like we're used to . The back-end ends up processing two requests but the front-end only knows about one , so it gets two responses back , takes the first one , and the second just sits in the queue . The next user's request then picks up that leftover response instead of their own , and their response might end up being ours . That's the poison.

So to compare it to what we usually do when we smuggle a partial request and append the next one to : 

With a partial prefix you're corrupting the victim's request, your leftover gets prepended to theirs and corruptes it. With a complete smuggled request you're not touching their request at all , you're just hijacking their response by leaving an extra one sitting in the queue :

- Partial prefix → you're corrupting the victim's request
- Complete request → you're hijacking the victim's response without even touching their request

Now following the RFC to have a complete GET request , we should end it with a return line : 

```bash
POST /AAAAA HTTP/2
Host: 0a84002804ef05af81241bac00930033.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked

0

GET /404_Return HTTP/1.1
Host: 0a84002804ef05af81241bac00930033.web-security-academy.net

```

Why are we requesting endpoints that won't exist , this is so that we know the difference between our Response and the Response of another user . 

We're expecting a 404 , the user isn't , so if we get his response , we get the 200 or 302 or whatever , and he gets our 404 : 

<img width="1417" height="419" alt="image" src="https://github.com/user-attachments/assets/2c57f4cb-0134-4d54-8be9-c86f67e1b664" />

Now we keep sending this Request until we get a different response than the 404 : 

<img width="1494" height="566" alt="image" src="https://github.com/user-attachments/assets/6a33e9ef-7485-4eb5-ba0c-442023fca94e" />

We get a 200 which means the attack is working , we are able to capture other user's responses :

Either do it manually , or just use Intruder : 

<img width="1517" height="629" alt="image" src="https://github.com/user-attachments/assets/0d6440fc-0e6c-47a1-b17e-aae904ca03cd" />

- Use Sniper Attack .
- Use Null Payloads .
- Continue Indefinetely .

<img width="1868" height="680" alt="image" src="https://github.com/user-attachments/assets/6a75be39-05fa-46a6-825d-1761bccf37b2" />

- We must remove the Update Content-Length as well :

<img width="1784" height="535" alt="image" src="https://github.com/user-attachments/assets/54409a7f-0b04-4e0a-89f4-c484c76f9871" />

We are able to get a 302 instead of the 404 : 

<img width="1797" height="803" alt="image" src="https://github.com/user-attachments/assets/fae2e886-819d-424d-9c48-4fe41b7ff304" />

We got a cookie as well . Now all we need to do is use Devtools to login as the admin and Delete Carlos : 

<img width="1901" height="783" alt="image" src="https://github.com/user-attachments/assets/8d099c9e-6b6f-48ed-b0a2-e2c740b60de3" />

Once we delete the user , we should solve our lab . 

<img width="1895" height="625" alt="image" src="https://github.com/user-attachments/assets/83ff1273-317e-4ab6-a97d-7f6f460c3f6c" />

That was all for this lab , see you in the next one .
