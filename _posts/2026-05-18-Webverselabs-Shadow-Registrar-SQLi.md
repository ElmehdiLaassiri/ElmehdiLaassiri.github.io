---
title: "Webverselabs Shadow Registrar SQLi  "
date: 2026-05-18 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Hard , Challenge , Web_Attacks ]
---


## Scenario:

RegistryPro is a domain registrar that has been quietly serving small hosting resellers out of Dublin since 2008, with a public lookup page that has barely changed since the original CTO wrote it. A junior on the UX team redesigned the terminal last spring, surfaced a few extra diagnostics in the response, and shipped the change without anyone on the backend side reviewing what those diagnostics actually exposed. The original CTO retired in 2021 and nobody else really knows the driver layer.


## Solution : 

<img width="1861" height="847" alt="image" src="https://github.com/user-attachments/assets/ab2cbe58-08ad-4847-b7b2-27eb530eca74" />

First thing first , we start by navigating the application like a normal user , try different endpoints to see what response we get , then we check the Burp History to see the content of each of our requests . 

<img width="837" height="604" alt="image" src="https://github.com/user-attachments/assets/b2128b61-bcbd-4bce-baa5-376baa83e045" />

We see the /dns.php which has the domain_id parameter , first thing i would think of is trying to inject in that parameter field to see if it's vulnerable , after trying multiple SQLi payloads we don't get anything useful in return , server doesn't return internal errors , time based payloads don't cause any delays , let's keep looking . 

<img width="1284" height="585" alt="image" src="https://github.com/user-attachments/assets/753eaf92-aef8-497f-9e6e-3ced3d24730f" />

Let's take a look at the /whois.php endpoint : 

<img width="1228" height="587" alt="image" src="https://github.com/user-attachments/assets/f1572a30-c305-4015-810a-74e28529b9c8" />

Let's try to inject the domain parameter , i like using this time based SQLi payloads from lostsec : 

```bash
https://github.com/coffinxp/customBsqli/blob/main/payloads/generic.txt
```

Huge indicator that this parameter might be injectable is the error we get once we add ' to the domain field . 

<img width="1444" height="560" alt="image" src="https://github.com/user-attachments/assets/398f1f11-9ab8-41f4-a543-a6fdc911237a" />

Now let's some of the Time based payloads from the github page , personally i used this one which worked just fine confirming that we got a Time based SQLi . 

```bash
'; SELECT BENCHMARK(10000000,MD5(1)); -- -
```

<img width="1920" height="785" alt="image" src="https://github.com/user-attachments/assets/b5cc1c44-2372-4247-9fd6-362193289b16" />

Now we can use sqlmap , specify that it is vulnerable to time based , and since this payload is for Mysql we know that this is the DB being used . 

<img width="1195" height="931" alt="image" src="https://github.com/user-attachments/assets/2fcbb59a-ee97-4a76-a5ec-8dc58bfa6a12" />

Perfect , now sqlmap can find that it is vulnerable as well : 

<img width="1228" height="908" alt="image" src="https://github.com/user-attachments/assets/4b2eac20-cc51-44e7-b2f6-c6bd567ef9d1" />

For time based SQLi , you can use a Tool named Ghauri as well , it is better for time based than sqlmap but for this one sqlmap will work just fine for us , Now all we need to do is dump the tables , and let it run for a while since this is time based sqli , it will take a bit logner . 

<img width="1152" height="877" alt="image" src="https://github.com/user-attachments/assets/add2383c-f432-4590-ac5a-49234aa44dc1" />


And we get our flag this way , this is it for this challenge , see you in the next one :) 



