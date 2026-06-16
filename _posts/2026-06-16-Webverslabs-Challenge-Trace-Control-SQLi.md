---
title: " Webverselabs Challenge Trace Control SQLi  "
date: 2026-06-16 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---



## Scenario : 

Trackboard is a self-hosted issue tracker maintained by a four-person platform team at a mid-size adtech firm in Austin, in use internally since 2019. Last Thursday's emergency hotfix was cut from a staging branch nobody had remembered to align with prod, and the on-call who pushed it left for a long weekend the next morning. You've got a few hours before the next deploy — make them count.


## Solution : 

<img width="1657" height="799" alt="image" src="https://github.com/user-attachments/assets/84637872-f4c8-446b-935f-171ccbacdac9" />

We first start by navigating the app just like a normal user would , check different endpoints , use different features , then check Burp's History to see if we got any interesting requests made to the server.

We're mainly looking for parameter where we can inject , the goal is to see if we can cause an internal error by messing up the query's quotations .

**/Projects Issues:**

<img width="1556" height="733" alt="image" src="https://github.com/user-attachments/assets/e77bf71c-426b-49f3-8b16-48a58d8c5b63" />

This is a list of projects ,notice how there is a parameter in the GET request , if we try to inject a simple payload o break the sql query : 

<img width="1558" height="727" alt="image" src="https://github.com/user-attachments/assets/c5138928-3b1d-4ba8-a557-278398d6cca4" />

We see that we don't get any errors , we can also confirm that the endpoint is indeed not vulnerable if we use sqlmap . 

Moving on to the **Search** feature :

<img width="1298" height="500" alt="image" src="https://github.com/user-attachments/assets/8901baec-6fa3-42b7-835d-3ce0246f497c" />

We also see a parameter inside the URL which means we can test it as well . 

<img width="1220" height="565" alt="image" src="https://github.com/user-attachments/assets/0a3ddbad-b4e6-4b33-a095-7048c6364ae4" />

We can't really cause an error , i also tried SQLmap but that didn't work either. 

**/New Issue :**

<img width="1307" height="887" alt="image" src="https://github.com/user-attachments/assets/d27eca0f-606b-4132-a0a3-56f66b5503e9" />

We can create a new Issue , if we look at the reqeuest : 

<img width="1603" height="664" alt="image" src="https://github.com/user-attachments/assets/ba9c3fbf-b5e2-48ca-88ec-540d94f19efe" />

We see that it is a redirection to /issue.php : 

<img width="1616" height="676" alt="image" src="https://github.com/user-attachments/assets/21fd0e6f-7a65-486a-b51d-34b5342e6ba3" />

We notice that there is a parameter this time, we can try injecting inside that parameter before we inject the original POST Request (i didn't immediately go for the POST request since it is a redirect so it will be harder to test). 

If we also go back to the **Overview** endpoint, we see that our Issue was created. 

<img width="1706" height="585" alt="image" src="https://github.com/user-attachments/assets/90b86101-7926-4b1b-88b3-2bd6274e5c39" />

Now if we click on any of the isse we go back to where we were once we created the Issue , from here we said we were going to try and inject the id parameter as well before moving to the POST Request . 

<img width="1574" height="355" alt="image" src="https://github.com/user-attachments/assets/371df7b9-a9e1-43bc-8d09-16ee3a0620a2" />

Perfect, we managed to cause an error , which also gives us the DB used , which in this case is MariaDB.

To save time , we can use SQLmap now that we know for sure the parameter is vulnerable and the DB is MariaDB , just make sure that you modify the user-agent since sqlmap uer agent is always blocked by any firewall.

<img width="1723" height="939" alt="image" src="https://github.com/user-attachments/assets/af09278d-8ddf-4b00-a0ed-bd288bd58a30" />

Perfect, it s a time based SQLi . 

I first ran this : 

```bash
python3 sqlmap.py -u https://69ce1775-4327-trace-control-1c416.challenges.webverselabs-pro.com/issues.php?id= --random-agent --batch --skip-heuristics --dbms=MariaDB --dump
```

But it took so long since it is a Time based SQLi . 

<img width="939" height="536" alt="image" src="https://github.com/user-attachments/assets/99ca6f7e-9214-49a2-8029-6e5ec5fbafbc" />

So once it identified the DB name as well as the Table name , i used them to retrieve the flag faster .

```bash
python3 sqlmap.py -u https://69ce1775-4327-trace-control-1c416.challenges.webverselabs-pro.com/issues.php?id= --random-agent --batch --skip-heuristics --dbms=MariaDB --dump -D chalapp -T admin_flags --threads=10
```

<img width="1351" height="685" alt="image" src="https://github.com/user-attachments/assets/c59313cd-7e7f-4b23-a86e-70387817cc32" />

Perfect, we're able to retrieve the flag.

That was all for this challenge, see you in the next one. 

