---
title: " PortSwiggerlabs: File path traversal, traversal sequences blocked with absolute path bypass "
date: 2026-08-18 00:00:00 +0000
categories: [PortSwiggerLabs , Path Traversal ]
tags: [PortSwiggerLabs , Path Traversal , Challenge , Web_Attacks ]
---



## Scenario : 

This lab contains a path traversal vulnerability in the display of product images.

The application blocks traversal sequences but treats the supplied filename as being relative to a default working directory.

To solve the lab, retrieve the contents of the /etc/passwd file.


## Scenario : 

<img width="1695" height="863" alt="image" src="https://github.com/user-attachments/assets/723cd675-147c-4098-88d5-d2dbae7e3a04" />

First we navigate the application like a normal user as usual , from there we check Burp's History : 

<img width="1466" height="697" alt="image" src="https://github.com/user-attachments/assets/0a57e83c-32ce-4a45-89f5-83bbb816ba7a" />

We find 2 parameters , one for the Product ID and the other one is for the pictures used for each product, to save time : I already tested PoductID with multiple Payloads from payload all the Things : 

```text
https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/File%20Inclusion/README.md
```

But it wasn't vulnerable : 

<img width="1469" height="673" alt="image" src="https://github.com/user-attachments/assets/b78da34a-fff5-4aef-a3cf-e722b25148ea" />

Now let's test Filename : 

<img width="1441" height="631" alt="image" src="https://github.com/user-attachments/assets/1ec4909d-1ed0-484a-9c29-f30d973e3307" />

I kept getting No such file , whenever i tried an LFI Paylaod . I even tried multiple filter Bypasses but that didn't work . 

Since it's saying No Such File , this made me think , maybe if we give the file name , it might find it , instead of having to go back to the root folder . 

Maybe /etc/passwd does exist , but ../../../etc/passwd doesn't since it takes it as an entire string . 

<img width="1481" height="766" alt="image" src="https://github.com/user-attachments/assets/b07e7161-92c0-4c19-aa8e-36afd1273bb8" />

That worked , and now if we go back to our Lab , it should say Solved . 

<img width="1736" height="806" alt="image" src="https://github.com/user-attachments/assets/7799e3dd-e245-4889-b10c-e99791e29837" />

That was all for this challenge , see you in the next one :) 

