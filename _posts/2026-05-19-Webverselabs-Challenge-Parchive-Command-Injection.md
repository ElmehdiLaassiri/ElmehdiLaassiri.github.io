---
title: "Webverselabs Parchive Command Injection  "
date: 2026-05-20 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


## Scenario : 

Parchive is a document-archiving SaaS founded in 2020 in Boston for mid-market law firms and audit shops, with seats starting at $89/month and a customer base of around 4,000 active reviewers. The bulk-export feature was rewritten last summer by a contractor who pushed back on the team's request for a whitelist, arguing it would break power-user filenames; the compromise was a small "obvious junk" check on the archive name field. A client flagged some unexpected behaviour in the export endpoint during a routine review.



## Solution : 


<img width="1731" height="829" alt="image" src="https://github.com/user-attachments/assets/535ae447-7623-4ffd-96a6-3fc92f474066" />

As usual , we will navigate the application just like a normal user , understand different features , functionalities , then check Burp History to see the different requests that were made to the server . 

<img width="1906" height="781" alt="image" src="https://github.com/user-attachments/assets/b7046fd1-ea39-43ba-b4c8-62c879710925" />

After navigating the application , we find this Download button , looking at Burp's History , this is the only POST request made to the server , and the only one containing a parameter we can tamper with . 

<img width="1161" height="560" alt="image" src="https://github.com/user-attachments/assets/b4de73eb-79a8-4c4b-946b-61a806df67e4" />

I sent it to Repeater then tried some basic Command Injection payloads , injecting the first parameter resulted in an error . 

<img width="1386" height="683" alt="image" src="https://github.com/user-attachments/assets/46e81d54-3dbd-46fc-a865-1e1da54672f4" />

But upon trying the same payload on the archive_name parameter , we dont get any errors : 

<img width="1210" height="654" alt="image" src="https://github.com/user-attachments/assets/d72abc3b-54ea-4b4a-9cbb-75696ad186d7" />

Perfect we can try adding a different command , to see if we do actually have a command injection : 

<img width="1407" height="651" alt="image" src="https://github.com/user-attachments/assets/47a79cc1-0754-4426-8007-bb0680267aa3" />

Perfect , we do have our commmand injection , now to read the flag , we don't really know if spaces are blocked by default , for this we will use some basic bypasses , i already have this section on my methodology so i will just use it : 

```bash
# Bypass Space : 

Check with New Line :  

\n : %0a ==> This is usually not blacklisted , we can check with this one , if it doesnt break anything , we probably have Command Injection .

===> Use $IFS : This is a known linux env variable .  
Ip=127.0.0.1%0a${IFS} 
```

So our payload will simply be : 

```bash
%0acat${IFS}/flag.txt 
```

<img width="1470" height="755" alt="image" src="https://github.com/user-attachments/assets/1117a321-0726-4615-8cca-f450c3a0c30c" />

That was it for this challenge , see you in the next one :) 



