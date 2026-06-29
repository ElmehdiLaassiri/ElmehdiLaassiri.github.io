---
title: " PortSwiggerlabs: Arbitrary object injection in PHP"
date: 2026-06-29 00:00:00 +0000
categories: [PortSwiggerLabs , Insecure Deserialization ]
tags: [PortSwiggerLabs , Insecure Deserialization , Challenge , Web_Attacks ]
---


## Information : 

This lab uses a serialization-based session mechanism and is vulnerable to arbitrary object injection as a result. To solve the lab, create and inject a malicious serialized object to delete the morale.txt file from Carlos's home directory. You will need to obtain source code access to solve this lab.

You can log in to your own account using the following credentials: wiener:peter


## Solution : 

First we login as wiener , then check Burp Site map to see different endpoints .

<img width="1741" height="857" alt="image" src="https://github.com/user-attachments/assets/c8dd2208-2c57-43e9-9bee-a57919230349" />

Now checking Burp : 

<img width="1734" height="870" alt="image" src="https://github.com/user-attachments/assets/647344a8-4301-4ca3-a696-0e419be87f40" />

We notice this template.php file , if we send it to Repeater : 

<img width="1446" height="653" alt="image" src="https://github.com/user-attachments/assets/44d34228-dd70-4a37-9c17-c6226ac660b0" />

To be able to read files using Repeater , we can add a ~ to the file name : 

<img width="1498" height="702" alt="image" src="https://github.com/user-attachments/assets/e80f810d-bc7d-4c67-8a2b-66f4e55ee675" />

Perfect we are able to read the file . 

If we check the destrcut function : 

```php
function __destruct() {
        // Carlos thought this would be a good idea
        if (file_exists($this->lock_file_path)) {
            unlink($this->lock_file_path);
        }
    }
```

This is the way we're going to delete the morale.txt file . Looking back at the code :

```php
class CustomTemplate {
    private $template_file_path;
    private $lock_file_path;
```

We can create a new CustomTemplate Object , with the  $lock_file_path , we need to make sure we don't make any errors when creating the serialized object : 

```php
O:14:"CustomTemplate":1:{s:14:"lock_file_path";s:23:"/home/carlos/morale.txt";}
```

Now first we Base64 encode it then we URL encode it just like we see in Inspector everytime we find a php serialized object. 

<img width="1536" height="719" alt="image" src="https://github.com/user-attachments/assets/d49925d4-a763-4528-8629-0151e22148d7" />

This is our new payload :

```php
TzoxNDoiQ3VzdG9tVGVtcGxhdGUiOjE6e3M6MTQ6ImxvY2tfZmlsZV9wYXRoIjtzOjIzOiIvaG9tZS9jYXJsb3MvbW9yYWxlLnR4dCI7fQo%3D
```

Now back at Burp , we see that once we login we have a new session cookie , which is also just a serialized object : 

<img width="1909" height="916" alt="image" src="https://github.com/user-attachments/assets/eb44c58d-ac4a-4b50-9e05-3bb6d351bc4c" />

Now we can send any request that has this cookie to repeater , then modify the cookie to our new value , and this will automatically trigger the __destruct()  method since we've specified the lock_fle_path in the serialized object :

<img width="1895" height="788" alt="image" src="https://github.com/user-attachments/assets/13ceca92-85e4-4902-abc4-0bab2a85e4da" />

Now if we change the cookie : 

<img width="1910" height="746" alt="image" src="https://github.com/user-attachments/assets/c66c30de-769c-4c47-bc21-6643158b0078" />

We see that the lab was solved .

That was all for this lab , see you in the next one :) 
