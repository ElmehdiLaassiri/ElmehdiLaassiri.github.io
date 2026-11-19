---
title: " Home Lab 1 : Enumeration / AS-REP / Kerberoasting"
date: 2026-11-30 00:00:00 +0000
categories: [Active Directory Pentesting]
tags: [Active Directory, Homelab]
---



Hello everyone , as mentioned in the previous homelab, we will be continuing the attack series ,the idea is simple , we will have few attacks or techniques to use , explain why it happens in the first place , how to perform it , and how to remediate it . 

For this one, the main focus is enumeration with Anonymous access , Kerberoasting and AS-REProasting . 


## Enumeration via anonymous access : 

Whenever we're given access to a Network but without any creds , the first thing we should always try is to check , shares , dump users , and machine accounts using anonymous access . 

### What's causing it ? 

For compatibility reasons , sometimes the group Pre-Windows 2000 Compatible Access is enabled , this can lead to anyone who has access to the network to request from the DC a list of all users and groups without having a valid creds .

But to really see why that works , we need a quick history lesson . Back in the NT4 days , machines talked to the domain controller through a **null session** : an unauthenticated connection over IPC$ that legacy clients used to enumerate users , groups and shares . When AD replaced NT4 domains , Microsoft had to keep those old machines working , so that compatibility path survived the migration .

In Active Directory terms , it comes down to two separate settings , and both need to be on :

**1/ The LDAP gate** : by default , a DC accepts an anonymous bind but refuses to serve anything useful with it ( you can connect , but searching users returns nothing beyond the rootDSE ) . That restriction lives in an attribute called `dSHeuristics` , on `CN=Directory Service,CN=Windows NT,CN=Services,CN=Configuration` . Set its seventh character to `2` and the DC now treats anonymous sessions like authenticated ones for LDAP operations . One character , and we're good to go.

**2/ The read rights** : once anonymous sessions are allowed to search , what they actually get back is decided by the DACLs . And that's where the **Pre-Windows 2000 Compatible Access** group ( SID `S-1-5-32-554` ) comes in . This built in group carries a set of Allow ACEs on the domain root , inherited by everything under it ( list contents , read general info ... ) . On a fresh domain it only holds **Authenticated Users** and **Interactive** , which is harmless . But on domains that were upgraded from NT4 ( or that an admin "fixed" for some legacy app ) , you will find **Everyone** and **ANONYMOUS LOGON** inside it . ANONYMOUS LOGON is exactly what it sounds like : the identity Windows gives to a session that never authenticated ( SID `S-1-5-7` ) . Put it in the group and every null session inherits those read rights .

So as a summary LDAP Gate open + read rights = the full user list over LDAP with zero creds .

**Quick info** : on a default install , both settings are off , so pulling a user list this way is a real misconfiguration , not default behavior . If you followed the homelab post , you remember we flipped both of them on purpose .

Now what this allows is retrieving the users via LDAP , but not SMB or RPC . For those , there is another door with its own gate : by default , modern Windows blocks anonymous enumeration over RPC with two policies ("Network access: Do not allow anonymous enumeration of SAM accounts" and "Network access: Do not allow anonymous enumeration of SAM accounts and shares") , which is why on a default DC a null session can connect to IPC$ just fine but gets access denied the second it asks for the user list or the share list .

### Practical Exploiation : 

To exploit this it is pretty simple 
