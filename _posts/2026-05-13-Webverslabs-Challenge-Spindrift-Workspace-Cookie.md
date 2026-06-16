---
title: " Webverselabs Challenge Spindrift Workspace Cookie "
date: 2026-05-13 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Easy , Challenge , Web_Attacks ]
---


## Scenario : 

Two founders, eighteen months in, no marketing budget, no AI features. They built their own session layer one weekend "because we don't need a whole library for this." A junior contractor noted in the team Linear that the cookie was "stateless and self-describing" and shipped it.


## Solution : 

<img width="1697" height="864" alt="image" src="https://github.com/user-attachments/assets/74cd6fb0-5178-4325-9694-8c2b64f03b3f" />

I will speedrun this one since this one was pretty easy : 

**/Pricing** , **/Changelog** , **Field Notes** : All of these returns static web pages that are pretty useless to us .

We will try to create an account : 

<img width="1644" height="738" alt="image" src="https://github.com/user-attachments/assets/0cd7f22d-b912-4d36-b171-d34f0b97620e" />

If we try to use the Free plan , we're given a default account , Bob@gmail.com : 

<img width="1508" height="570" alt="image" src="https://github.com/user-attachments/assets/875c4687-6a49-4595-ba7f-d26ccbe5a1ab" />

Looking at the request, we can't modify the role or anything like that during the login . 

<img width="1725" height="862" alt="image" src="https://github.com/user-attachments/assets/ead1661e-d699-45c4-94d8-567cebe8dc0c" />

Once we login , we can view our account , or Open New Workspaces . 

<img width="1502" height="680" alt="image" src="https://github.com/user-attachments/assets/8736b9fb-3208-4e53-95eb-02f9af199aca" />

Now if we look closely at the request , the cookie ends with a "=" and looks like a base64 encoding , if we try to decode it : 

<img width="1573" height="243" alt="image" src="https://github.com/user-attachments/assets/05c74f99-0ce5-4016-9a36-f6b3833639df" />

We see that we have the role of a member, let's try to modify the role and encode it again and see if we can get access as the Administrator : 

```bash
echo "{"uid":42,"email":"bob@example","role":"admin","exp":1781607517}" | base64
e3VpZDo0MixlbWFpbDpib2JAZXhhbXBsZSxyb2xlOmFkbWluLGV4cDoxNzgxNjA3NTE3fQo=
```
<img width="1079" height="231" alt="image" src="https://github.com/user-attachments/assets/feb07ac0-3305-4b0e-84c1-fb06e66ecf53" />

Now back to the app , let's modify the Cookie from the browser tools : 

<img width="1802" height="846" alt="image" src="https://github.com/user-attachments/assets/e3ab5a64-5d99-40e9-af64-256e4a8e94c0" />

And from there we access the New Admin endpoint and get our flag that way . 

That was all for this challenge, see you in the next one :)




