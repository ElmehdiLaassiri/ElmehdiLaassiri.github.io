---
title: " Hack The Box Armexis Walkthrough"
date: 2026-08-06 00:00:00 +0000
categories: [CWES Preparation Path ]
tags: [CWES Preparation, HackTheBox, Challenge , Web_Attacks ]
---


## Scenario : 

In the depths of the Frontier, Armaxis powers the enemy’s dominance, dispatching weapons to crush rebellion. Fortified and hidden, it controls vital supply chains. Yet, a flaw whispers of opportunity, a crack to expose its secrets and disrupt their p


## Solution : 

So first we are given a ZIP file as well as 2 Hosts : 

<img width="1533" height="409" alt="image" src="https://github.com/user-attachments/assets/cb030c66-7f93-4e92-aecb-50efc5f92a3c" />

First let's unzip the file using the password they have given us : 

<img width="1184" height="568" alt="image" src="https://github.com/user-attachments/assets/d7d4a63b-7730-4b1f-a190-c4fc51dc9fae" />

We're given the entire application :

<img width="711" height="781" alt="image" src="https://github.com/user-attachments/assets/bf56e136-c903-4170-bdc5-419238b7b9a1" />

But first let's check the 2 IP addresses that we were given : 

- 00 :

<img width="1289" height="717" alt="image" src="https://github.com/user-attachments/assets/edbe3a43-d9fc-4eb8-b68a-e479ccb03b83" />

The first one is a Login page .

- 01 : 

The second one is the mail box for the email : test@email.htb :

<img width="1907" height="432" alt="image" src="https://github.com/user-attachments/assets/c5367cfd-e063-4720-b32d-038932088d36" />

I first tried Fuzzig for other directories starting with the first host 00 : 

<img width="1453" height="823" alt="image" src="https://github.com/user-attachments/assets/6d4a2b4b-130f-4616-babe-c21cbfa75d0a" />

We find 2 interesting endpoints /ResetPassword and /Weapons which returns a 404 .

Let's first check the Reset Password feature : 

<img width="1246" height="696" alt="image" src="https://github.com/user-attachments/assets/f0b4827f-b3bd-403c-9e01-2ede5f9e0389" />

We first need to specify the account , and it will probably send us an OTP , we already have the Mail box for the test@email.htb user so let's use that one : 

First we need to create this account using the register endpoint : 

<img width="1444" height="441" alt="image" src="https://github.com/user-attachments/assets/fb9738bf-9dfd-439a-b55b-643a1fd58ece" />

Then we can request a Password reset :

<img width="1597" height="687" alt="image" src="https://github.com/user-attachments/assets/1eebf8eb-4b82-41c5-9c85-d5865e662a06" />

For this one we need to provide a Token , then the new password , first we need a Valid Token , which we will recieve on our Mail box :

<img width="1900" height="524" alt="image" src="https://github.com/user-attachments/assets/5616e574-4c58-401b-a120-cca1b6b4e5fc" />

Now back to the reset page , we specify this token and the new password : 

<img width="1648" height="686" alt="image" src="https://github.com/user-attachments/assets/3b9208d4-0674-4e63-92ef-cc9de977ea4e" />

And the password is successfuly changed , now let's check Burp :

<img width="1452" height="524" alt="image" src="https://github.com/user-attachments/assets/7b2f4b1c-c06e-4f1a-af26-f500437de102" />

We see that in the POST Request we're sending an email + Token and the new email , what if we send a valid Token and modify the email to the admin email , can we reset the admin password that way ? 

First let's request a new Token :

<img width="1907" height="568" alt="image" src="https://github.com/user-attachments/assets/31654648-bb96-4708-95bf-b7f8a3821aa0" />

Now using this new Token , let's try and modify the password for the admin , now back to our Burp request , send it to Repeater : 

<img width="809" height="515" alt="image" src="https://github.com/user-attachments/assets/db7892f0-a02f-4ab0-9b8c-818b6b699a29" />

We just replace the token with our new one , and we're just hoping that the admin email follows the same structure as our given user : 

```bash
{
"token":"fdb516f0900d8ba01bb247f7bdde2c4c",
"newPassword":"123456",
"email":"admin@email.htb"
}
```

<img width="1236" height="496" alt="image" src="https://github.com/user-attachments/assets/59357b4b-3e23-44f9-a86c-5b306fcc859c" />

We get User not found , i tried multiple ones , (eg Admin , Administrator ....) but none of them was correct . 

Now let's go back to our ZIP file :

<img width="814" height="703" alt="image" src="https://github.com/user-attachments/assets/4e56b174-b8fc-4a69-8155-ee40f415532f" />

Looking at the folder structure , few files might be useful for us , first the index.js , then route/index.js , email-app/index.js , as all of these might contain the administrator email inside the code : 

Let's navigate the entire codebase recursively using grep and filter for admin : 

```bash
grep -r admin
```

<img width="1226" height="618" alt="image" src="https://github.com/user-attachments/assets/863bdda2-8c61-4b2e-98ab-63add6d13786" />

Perfect we found the admin email , we can test with this :

```text
admin@armaxis.htb
```

Now back to Burp :

<img width="1326" height="582" alt="image" src="https://github.com/user-attachments/assets/ea4eaaaa-a2c3-4757-951c-623a16a3e8de" />

Perfect now let's login using the administrator account :

<img width="1902" height="685" alt="image" src="https://github.com/user-attachments/assets/7b5c205c-0b5f-49c5-af45-1ebfb4ba3195" />

Once inside we find the weapons endpoint which returned 401 earlier :

<img width="1879" height="788" alt="image" src="https://github.com/user-attachments/assets/0fdebb77-8686-4d4c-8b03-07e042ecf064" />

Checking Burp as well doesn't return anything useful :

<img width="1410" height="422" alt="image" src="https://github.com/user-attachments/assets/f68c34d7-381f-4b0c-a722-4eb94f710c7d" />

Let's go back to our code base to read the /weapons section :

<img width="1154" height="604" alt="image" src="https://github.com/user-attachments/assets/f0f82baa-39ff-4b2b-a35a-15171da862a1" />

We find this :

```js
  try {
    const parsedNote = parseMarkdown(note);
```

Let's check the codebase for this function :

<img width="1213" height="377" alt="image" src="https://github.com/user-attachments/assets/ab6739af-7863-482e-8e4f-170c23eaab10" />

Let's read the challenge/markdown.js file :

<img width="1134" height="700" alt="image" src="https://github.com/user-attachments/assets/d6232d28-4266-4806-b68a-f71bad4ffe78" />

This is a huge indicator of a command injection :)

The url comes straight from a markdown image tag ![alt](url) that a user controls, and it's being dropped directly into a shell command string with no sanitization or escaping. execSync runs this through /bin/sh, so anything that breaks out of the "URL" context gets executed as a shell command.

This is pretty simple notation tho : ![alt](url) :

- ! : First this tells it that this is an image not a text .
- alt : this is the message that will get show if the image doesn't exist .
- url : this is where it will fetch the resource from .

In our case we will use the file:// protocol to read internal files ( just like a nomral ssrf) .

So back to our Burp Request , inside the field that accepts Markdown which is the Note :

<img width="929" height="663" alt="image" src="https://github.com/user-attachments/assets/e810554a-ea86-4548-8027-1ee7a610ae27" />

Now we just inject : 

```xml
![Non_existing](file:///etc/passwd)
```

<img width="1300" height="509" alt="image" src="https://github.com/user-attachments/assets/acda98b3-d2c2-4b69-af66-9912c8c51e04" />

Now if we check the Weapons endpoint :

<img width="1906" height="703" alt="image" src="https://github.com/user-attachments/assets/8bc02e2d-d4e1-4579-883f-650e899961c9" />

We see that we have an image that was imbedded , let's check the source code , we should get it in a Base64 format like the code indicates : 

<img width="1801" height="240" alt="image" src="https://github.com/user-attachments/assets/58e9fc16-51ea-4691-8769-efe08e675e3d" />

Let's decode it :

<img width="1610" height="887" alt="image" src="https://github.com/user-attachments/assets/8d738e51-ff00-4b21-aa44-a735a0cec8f1" />

Perfect we got the content of the /etc/passwd file , now let's try to read the flag : 

it is usually located at /flag.txt so the payload this time should be :

```xml
![Non_existing](file:///etc/passwd)
```

<img width="1334" height="593" alt="image" src="https://github.com/user-attachments/assets/b369c81b-bc1c-4722-85f4-ad4a4c308b10" />

Now if we go back to Dashboard we will see a new weapon that got dispached :

<img width="1343" height="461" alt="image" src="https://github.com/user-attachments/assets/e6d50e7c-9f6c-4f2f-9f40-bcb596206c68" />

Now same thing , we just read the source code , decode the base64 value and we should get the flag :

<img width="1764" height="596" alt="image" src="https://github.com/user-attachments/assets/ebb2b5e1-2f82-4682-9e1f-e02953b40cdc" />

Now back to Cyberchef :

<img width="1326" height="687" alt="image" src="https://github.com/user-attachments/assets/6ccae67a-e091-4991-bcfe-d674e7c9e9b6" />

We got our flag . 

That was all for this challenge , see you in the next one :)
