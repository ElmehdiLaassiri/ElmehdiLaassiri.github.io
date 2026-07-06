---
title: " PortSwiggerlabs: Response queue poisoning via H2.TE request smuggling"
date: 2026-07-07 00:00:00 +0000
categories: [PortSwiggerLabs , HTTP Request Smuggling ]
tags: [PortSwiggerLabs , HTTP Request Smuggling , Challenge , Web_Attacks ]
---


## Information : 

This lab is vulnerable to request smuggling because the front-end server downgrades HTTP/2 requests even if they have an ambiguous length.

To solve the lab, delete the user carlos by using response queue poisoning to break into the admin panel at /admin. An admin user will log in approximately every 15 seconds.

The connection to the back-end is reset every 10 requests, so don't worry if you get it into a bad state - just send a few normal requests to get a fresh connection.


## Solution : 

