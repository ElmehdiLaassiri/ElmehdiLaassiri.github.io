---
title: "Webverselabs Crosswind Challenge File Upload  "
date: 2026-05-15 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---



# Scenario : 

Crosswind has run since 2019 on member dues and three volunteer mods — no ads, no premium tier, no syncing your heart-rate to a hedge fund. The stack is whatever ran fine on a Linode in 2019: PHP, a single database file, and a profile-picture form the founder wrote in one sitting after a long Saturday gravel ride. The next morning he added a short rejection list for "the obvious stuff people might try to upload," patted himself on the back, and went out for coffee.


# Solution : 



 First we Visit the web app , try navigating it like a normal user as usual . 



<img width="1399" height="876" alt="image" src="https://github.com/user-attachments/assets/f6f4f6f5-f18f-444f-9de3-34ace3650f80" />



The  /Feed and /Rides endpoint dont return anything useful , Fuzzing didn't return any additional endpoints . Let's create an account and browse the app more , we find /account.php which allows us to upload an image .  



<img width="1325" height="870" alt="image" src="https://github.com/user-attachments/assets/0048f783-f89c-41ab-9d61-4b4035202b78" />



Let's try uploading a webshell and see if we can bypass front end filters . 



<img width="854" height="392" alt="image" src="https://github.com/user-attachments/assets/d04e3214-15c4-48d9-9ccb-6cd3069437dd" />



First Bytes are there to make sure we bypass the MIME Type filters . Now let's upload this , intercept the request and modify the extension to php to see if it works . 



<img width="1578" height="900" alt="image" src="https://github.com/user-attachments/assets/2b7c383e-890a-414e-945a-4943601511d9" />



This doesn't work since php extensions are blocked , this is good as the application uses Black List filtering which is easier to bypass than a whitelist filter . 
Here is a list of PHP extensions that we can try to see if it bypasses this filter . 

```bash
https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/web-extensions.txt
```

I will be using php7 , feel free to use any of these extension as long as it allows the execution of php code . 



<img width="1385" height="908" alt="image" src="https://github.com/user-attachments/assets/e83b6987-4613-423c-89c9-59872bc4fec1" />



It works , now we just need to find where the uploaded image is stored , luckily for us it is displayed to us by the application : 



<img width="1411" height="718" alt="image" src="https://github.com/user-attachments/assets/ea5ce586-1601-4877-b45e-88c0538db4f1" />



Now we just visit the /uploads/avatars/7-profile.php7 and we should find our webshell . 



<img width="1224" height="329" alt="image" src="https://github.com/user-attachments/assets/d59a0b28-ddb0-459d-8d98-f187ddaa76a7" />


The flag is located at /flag.txt . 


<img width="1192" height="382" alt="image" src="https://github.com/user-attachments/assets/ce762c8a-9de6-4fe6-989e-15979bfc008b" />


That's it for this challenge . 



