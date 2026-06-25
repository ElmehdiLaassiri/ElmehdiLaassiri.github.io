---
title: " PortSwiggerlabs: DOM XSS in jQuery anchor href attribute sink using location.search source "
date: 2026-06-25 00:00:00 +0000
categories: [PortSwiggerLabs , XSS ]
tags: [PortSwiggerLabs , XSS , Challenge , Web_Attacks ]
---


## Information : 


This lab contains a DOM-based cross-site scripting vulnerability in the submit feedback page. It uses the jQuery library's $ selector function to find an anchor element, and changes its href attribute using data from location.search.

To solve this lab, make the "back" link alert document.cookie.



## Solution : 

First we must find the /back link : 

We see that we can submit a feedback : 

<img width="1453" height="861" alt="image" src="https://github.com/user-attachments/assets/09038112-4f8a-4f55-85cb-80d69b7738f1" />

Upon Clicking on it : 

<img width="1559" height="971" alt="image" src="https://github.com/user-attachments/assets/11a27444-0607-4007-9499-8d3e0bf3913c" />

We find the Return Path parameter where we can inject : 

I first started by injecting a simple payload to see how the server would respond : 

```js
<script>alert(window.origin)</script>
```

<img width="1391" height="965" alt="image" src="https://github.com/user-attachments/assets/ca9c6b83-1b66-40f0-88c6-e57ff099b3d6" />

Nothing , if we check the source code : 

<img width="1173" height="535" alt="image" src="https://github.com/user-attachments/assets/892dcc5d-0602-47d2-bb80-f6fdf5d7577a" />

We see that whatever we enter there it gets passed inside the href , href is usually used for links and redirections , it expects https:// and http:// ftp:// ... BUT we can instead specify another URL scheme which is 'javascript:' This will tell it , instead of fetching the URL , execute this code instead : 

Since we control the parameter being passed to the href  : '.get('returnPath')'

We can just inject this payload instead : 

```js
javascript:alert(document.cookie)
```

This will execute our JS code instead of fetching any external URL . 

We're just abusing how href works :

<img width="1549" height="996" alt="image" src="https://github.com/user-attachments/assets/db326a13-ca06-4589-b55f-aaa88953c761" />

The lab is solved once the XSS is triggered . 

That was all for this lab , see you in the next one :)
