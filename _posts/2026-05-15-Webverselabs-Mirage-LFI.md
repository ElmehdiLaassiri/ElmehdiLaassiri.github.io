---
title: "Webverselabs Mirage Challenge LFI  "
date: 2026-05-15 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---


## Scenario : 

NovaPan is a self-hosted hosting control panel founded in 2016 and bundled by a handful of European budget hosts, with licences starting at $9/month per server and somewhere around 30,000 active installs. The log-viewer feature was refactored two releases ago when a community contributor sent in a patch hardening the input handler, and the third-party auditors marked it 'low risk' after running their usual checklist against it. The contributor's patch did exactly what its commit message said it did, and nothing more.


## Solution : 

<img width="1822" height="834" alt="image" src="https://github.com/user-attachments/assets/898947a3-df12-40df-8f4a-b7648d14727f" />


Since we know this is an LFI challenge , my goal is to find an endpoint that uses a parameter , whether it's in the URL or inside the body , If we browse the webapp normally we find the /logs/view endpoint , which uses a parameter in the URL .


<img width="1368" height="842" alt="image" src="https://github.com/user-attachments/assets/b11e642f-6326-47b8-9d93-c332c33aade0" />


Now the way i like to search for LFI is by using FFUF with the Jaddix Wordlist from Seclist . 


<img width="1625" height="605" alt="image" src="https://github.com/user-attachments/assets/424bf8d0-c6fb-4297-8df5-b15f204e1cb4" />


If we filter for 200 Status pages , we get these 2 payloads that we can  to try .


<img width="1625" height="605" alt="image" src="https://github.com/user-attachments/assets/e57b18ff-c6b0-4295-b9d3-669e26d2e5c5" />


Perfect , we are able to get the content of the passwd file : 


<img width="1416" height="850" alt="image" src="https://github.com/user-attachments/assets/0cbf22d6-3a57-42ba-864d-416d84f93ef2" />


Now we can use the same payload to read the flag , it is always in the root directory : /flag.txt . 


```bash
%252e%252e%252f%252e%252e%252f%252e%252e%252f%252e%252e%252f%252e%252e%252f%252e%252e%252f%252e%252e%252f%252e%252e%252f%252e%252e%252f%252e%252e%252fflag.txt 
```

<img width="1500" height="723" alt="image" src="https://github.com/user-attachments/assets/f980b319-22b8-4ac6-aa11-be0a9171c8ae" />


That was it for this challenge :) 


