---
title: "Webverselabs Logcraft Command Injection "
date: 2026-06-09 00:00:00 +0000
categories: [Webverselabs]
tags: [Webverselabs , Medium , Challenge , Web_Attacks ]
---


## Scenario : 

LogCraft is a log aggregation SaaS for engineering teams. One of its paid features is the Health Report generator, which produces a formatted summary of log volume, error rates, and active sources. The report is named with a custom title supplied by the user. The developer knew double quotes were dangerous inside a shell command, so they stripped them before passing the title in. The command wraps the title in double quotes — and inside double-quoted strings, backtick substitution still executes.

## Solution : 

<img width="1025" height="620" alt="image" src="https://github.com/user-attachments/assets/0093a92b-b801-4736-8ea9-f84cbf06e04e" />

First thing first , we start by creating our account , and navigating the application just like a normal user would , then we will check Burp's history for endpoints that we can test further . 

<img width="1399" height="714" alt="image" src="https://github.com/user-attachments/assets/11c1b86a-9104-4b01-ae60-822af3c9c93c" />

Few endpoints can be identified just from looking at the Home page . 

/Search : 

<img width="1294" height="735" alt="image" src="https://github.com/user-attachments/assets/8ce5bacf-ade4-4d0b-a054-f78a38eeeb6d" />

This allows us to filter and search for specific Logs . 

/Dashboard : 

<img width="1735" height="787" alt="image" src="https://github.com/user-attachments/assets/81a4de49-4583-4c5d-a373-4b0748206265" />

This is just a Dashboard to visualize all the logs and events in a graph . 

/Reports : 

<img width="1155" height="577" alt="image" src="https://github.com/user-attachments/assets/01906678-c5b6-41b0-ac6d-85b4d20883da" />

This allows us to generate a Health check report . 

/alerts :

<img width="1801" height="888" alt="image" src="https://github.com/user-attachments/assets/7fa84ead-426d-4e29-9f91-a0e3d70b255a" />

This allows us to create new Rules . 

/Team : 

<img width="1895" height="773" alt="image" src="https://github.com/user-attachments/assets/68406c10-c735-4a68-ba20-dfa370449aec" />

This endpoint allows us to invite new user to our Projects 

Now looking at the description , it meantions something about reports generation , as well as double quote filtering . Let's go check the reports endpoint and use it to see the normal use case for it .

<img width="1527" height="426" alt="image" src="https://github.com/user-attachments/assets/bc257e11-6d67-4c6e-8545-6c64c3f1b9ac" />

We see that whatever we type in the Report title gets rendered to us next to "Health Report :" 

We already know this is a Command injection challenge so SSTI isn't the way here , we can assume that whatever we specify it gets passed as a parameter to a command that echoes " Health Report :"

I tried multiple Command injection payloads , but it just gets treated as text . 

<img width="1547" height="564" alt="image" src="https://github.com/user-attachments/assets/0b0b24ae-ef26-4e72-a2a5-c64c0db9a91e" />

If we check the description , we find that it says something about "" which means , maybe whatever we insert gets passed inside a "" which interprets it like a text .

For example : "test;" / "test||id" / "test&&id" => All of this will be printed out as a result of the echo command like normal Text . 

This means if we can add " at the beguinning maybe we can break out of the quotes and run our command ? 

<img width="1633" height="572" alt="image" src="https://github.com/user-attachments/assets/c988fc87-61e6-45be-857f-964d4c01253a" />

Now our " gets sanitized , which means there is probably a function that filters for " . 

Now we'll just assume we are inside a double quote , there is always a way to interpret a command as an executable command even inside the "" , by using `` or $() , this will be treated like commands to run even if we put them inside a double quote . 

For example echo "$(id)" will still execute id instead of printing it . 

<img width="1003" height="391" alt="image" src="https://github.com/user-attachments/assets/70ac1c64-c684-4038-934b-3e750a67075a" />

Now let's try to inject this command : $(id) : 

<img width="1660" height="548" alt="image" src="https://github.com/user-attachments/assets/6cf48352-7b9b-4ded-81fa-325c03d239ee" />

This worked perfectly . Now we can use this to get out flag .

<img width="1499" height="711" alt="image" src="https://github.com/user-attachments/assets/142a7caf-7528-4297-a80e-d113226182f4" />

We see that it is located in / as usual . 

<img width="1669" height="554" alt="image" src="https://github.com/user-attachments/assets/ad9fca2e-994a-42aa-ad9c-3a32110ca93d" />

Another thing , if we do an ls we can see the source code for our application , it's written in ruby . 

<img width="1369" height="489" alt="image" src="https://github.com/user-attachments/assets/e3f58e70-76db-4665-a132-9bae4d642c00" />

If we try and take a look at it : 

<img width="1855" height="776" alt="image" src="https://github.com/user-attachments/assets/e7dbaae0-21c7-434d-b997-87e0162d67ae" />

We'll try and check the function that does the sanitization . 

<img width="890" height="697" alt="image" src="https://github.com/user-attachments/assets/cc3fe42a-affd-4965-8b26-f8f11b363a8b" />

```ruby
==>the tr command will Replace " with nothing :) . 
safe_title = title.tr('"', '')
```

It is indeed filtering our the " inside of the title parameter whenever we try generating a report , this is done to eliminate the possibility of breaking out of the ""in the echo command . 

But they didn't filter other ones like $ () , ``` which can execute code even inside the double quote . 

That was all for this challenge , see you in the next one :) 




