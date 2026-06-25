---
title: " PortSwiggerlabs: Stored XSS into HTML context with nothing encoded "
date: 2026-06-25 00:00:00 +0000
categories: [PortSwiggerLabs , XSS ]
tags: [PortSwiggerLabs , XSS , Challenge , Web_Attacks ]
---


## Information : 

This lab contains a stored cross-site scripting vulnerability in the comment functionality.

To solve this lab, submit a comment that calls the alert function when the blog post is viewed.


## Solution : 

This one is pretty basic as well , no WAF nor sanitization , which means any payload will work . 

Since this is a stored one it's worse than a Reflected XSS , since once our payload is stored , it will execute on every victim browser as soon as they visit the app , this can be used to steal cookies if HTTP Only isn't used and perfrom Session Hijacking .

Now back to our Lab : 

<img width="1556" height="981" alt="image" src="https://github.com/user-attachments/assets/386717bd-fb66-4a48-bde3-4134d88b2d3c" />

Now if we visit any of the posts we see that we can leave a comment , this is huge since we can inject our payload there , and if a user visits the post our payload will execute .

<img width="1086" height="640" alt="image" src="https://github.com/user-attachments/assets/8de2c571-12be-46b9-9c42-4ff97ad7ef4b" />

First let's submit a normal comment to see if the comment is returned back to us or whether we need to test for Blind XSS .

<img width="1570" height="466" alt="image" src="https://github.com/user-attachments/assets/375e38a2-e2ea-4f3f-805d-ec10a8d08671" />

Now back to our Blog : 

<img width="1086" height="534" alt="image" src="https://github.com/user-attachments/assets/a4187173-e708-4e10-9298-b5edd8a967e4" />

We see that our comment is there , perfect , now i will be injecting a simple payload to test with :

```js
<script>alert(window.origin)</script>
```

I will be injecting inside the message field , if it doesn't work i will move to the others, emails are usually always secured .

<img width="1040" height="655" alt="image" src="https://github.com/user-attachments/assets/b58c1ceb-d6cd-41ae-9887-02158eac98fb" />

Now if we go back to the Comment, our payload should execute and we should get the window origin pop as an alert : 

<img width="1324" height="468" alt="image" src="https://github.com/user-attachments/assets/60048e53-7ee3-415c-90d6-0bbd712df282" />

And we do :

<img width="1184" height="395" alt="image" src="https://github.com/user-attachments/assets/0f821c81-d366-4ad4-a8c1-15faa6c5b62c" />

Now checking the comments : 

<img width="1009" height="384" alt="image" src="https://github.com/user-attachments/assets/5007d458-707d-4597-b0f5-6d007a0144a8" />

When the page is loaded, the stored script tag is injected directly into the HTML and executed by the browser, triggering alert(window.origin). Since script tags don't render visibly, we see the alert pop up rather than the payload appearing as text on the page.

If we refresh the page it will execute again since this is a stored XSS so the payload isn't stored in the client browser but on the server instead . 

<img width="1456" height="750" alt="image" src="https://github.com/user-attachments/assets/cd0a409f-ed7f-4044-b3cb-8de8237715e5" />

That was all for this lab, see you in the next one :)



