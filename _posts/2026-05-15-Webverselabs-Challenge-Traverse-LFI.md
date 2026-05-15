---
title: "Webverselabs Traverse Challenge LFI  "
date: 2026-05-15 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---


## Scenario :

Traverse is a four-person developer-tools startup out of Berlin that launched their API product in 2023 and rewrote the docs portal last month after a Show-HN thread complained the old one felt sluggish. The new version was put together by the founding engineer in a single caffeinated weekend, modelled on a tutorial blog post they half-read on the plane home from a conference. It looks the part — they have not yet had time to revisit how it actually serves pages.


## Solution : 

This one was pretty easy , upon visiting the web app , we find a parameter that is being passed in every request . 


<img width="1233" height="866" alt="image" src="https://github.com/user-attachments/assets/4e5cb005-65e5-447b-8733-3a64c2f8844d" />


We can use FFUF with a list of LFI payloads from Seclist ,i prefer the Jaddix wordlist , Finally we just filter the response to not match 200 status . 



<img width="1457" height="804" alt="image" src="https://github.com/user-attachments/assets/237c7696-7d35-4e64-b713-dd200ef9301e" />



We can test these payloads to see if it works or is it a generic page that we get . 


<img width="1411" height="481" alt="image" src="https://github.com/user-attachments/assets/615427f9-c5fe-4e16-9526-da450cdb6d10" />


Perfect we can even see the /etc/shadow file . Now let's try finding the flag , usually it's located in the root directory , which is /flag.txt , let's use one of these payloads and append flag.txt to it to be able to read it . 

```bash
https://eeda1f1e-4327-traverse-258cd.challenges.webverselabs-pro.com/page?name=/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/flag.txt
```


<img width="1536" height="596" alt="image" src="https://github.com/user-attachments/assets/cdfcbfc6-67c9-4c65-8331-c0ce2ab0ad0f" />

It works !) that's all for this challenge . 


