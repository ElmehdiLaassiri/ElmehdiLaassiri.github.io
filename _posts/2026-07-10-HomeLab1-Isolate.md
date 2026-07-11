---
title: " Home Lab 1 : Isolate "
date: 2026-07-10 00:00:00 +0000
categories: [Home Lab ]
tags: [Active Directory, Windows, Linux, CVEs, Privilege Escalation, Pivoting, ADCS, Homelab]
---



## Overview : 

This lab contains 6 boxes. Each one requires a foothold exploit to gain initial access, followed by a separate privilege escalation exploit to reach root or SYSTEM  the technique depends on the OS.

We'll start by building the lab using VMware to run the different boxes. The network is split into two segments  an internal network and a public-facing one, much like a real-world DMZ. This means tunneling and pivoting are required to reach anything inside once you've landed on the edge.

We'll first build the lab in VMware, and once everything is set up, we'll move to the attack phase using our Kali machine to compromise the entire network. Once every box is built, we'll break down the network architecture connecting them including how traffic actually gets from Kali into the internal segment.

The chain mixes classic and recent CVEs  from Drupalgeddon2 to fresh 2026 kernel bugs like CopyFail, DirtyFrag, and RoguePlanet. Then an AD box, where we focus on a specific ADCS attack: ESC8.

Quick version of the full attack path: 

- Foothold on **WEB01** via a React Server Components deserialization bug (CVE-2025-55182), then root via a kernel LPE, CopyFail, before pivoting into the internal network with Ligolo-ng, exposing four more machines. From there.

- **Box 2** falls to Drupalgeddon2 (CVE-2018-7600) for a www-data shell, escalated to root via another kernel LPE, DirtyFrag. **Box 3** goes down next through a WordPress plugin LFI-to-RCE chain (CVE-2023-6553, PHP filter chains) for www-data, followed by a misconfigured SUID find binary for root.

- **Box 4** is a pure misconfig box, where a no_root_squash NFS share leaks Windows credentials outright, while Samba's usermap script (CVE-2007-2447), an unauthenticated Redis instance, and a passwordless sudo rule allowing the leaked user to run nano as root (GTFOBins) all offer independent routes straight to root anyway.

- Those leaked credentials get reused on **Box 5**, a Windows workstation, where SMB access via NetExec enables RDP remotely, and RoguePlanet, a Windows Defender race condition, escalates to SYSTEM.

- The same credentials are reused once more on the Domain Controller **(Box 6)** as a valid domain account, triggering an ADCS ESC8 chain, where PetitPotam coerces the DC to authenticate, ntlmrelayx relays that auth to the undefended Web Enrollment endpoint, Certipy turns the resulting certificate into a TGT via PKINIT, and a final DCSync dumps every credential in the domain, including krbtgt.


<img width="1600" height="1000" alt="svgviewer-png-output" src="https://github.com/user-attachments/assets/9c9890b8-dc6f-42c1-b105-5e65c0f35cf1" />


## Scenarios : 


### Box 1 : Fail2Copy : 

WEB01 runs a Next.js application a full-stack framework built on top of React. React handles UI components, while Next.js adds server-side capabilities on top: file-based routing, API endpoints, and critically React Server Components (RSC), which let React components execute directly on the server, inside the Node.js process.

*(Quick primer if you're new to the JS ecosystem: JavaScript is the language, TypeScript is JavaScript with type annotations that gets compiled away before runtime, and Node.js is the runtime that lets JS execute outside a browser. None of that matters for the exploit itself what matters is that RSC means React code runs server-side, inside a real process, with real privileges.)*

This server-side execution is exactly what CVE-2025-55182 targets. By sending a crafted POST request with a malicious serialized payload, an attacker can abuse the RSC deserialization mechanism to inject and execute arbitrary JavaScript inside the Node.js process Remote Code Execution as the application user.

From that low-privileged shell, we chain into CVE-2026-31431 (CopyFail), a Linux kernel Local Privilege Escalation, to get root.


### Box 2 : DirtyGeddon2 : 

This box runs Drupal a popular open-source CMS on a classic LAMP stack: Linux, Apache, MySQL, and PHP. Apache receives the request, hands it to PHP, PHP runs Drupal, Drupal talks to MySQL, and PHP renders the response.

The foothold is CVE-2018-7600, nicknamed Drupalgeddon2, affecting Drupal 7.x before 7.58 (our target runs 7.57). The root cause sits in Drupal's Form API: internal array keys prefixed with # like #post_render are used to call PHP functions during rendering. Drupal accepted these keys directly from raw HTTP input without sanitizing them. An unauthenticated attacker can send a crafted POST request to the public registration page, inject a #post_render key pointing at PHP's exec(), and pass any OS command as the argument giving a shell as www-data, no authentication required.

From there, we chain into CVE-2026-43284 (DirtyFrag), another Linux kernel Local Privilege Escalation, to reach root.


### Box 3 : Backup : 

This box runs WordPress the most widely used CMS on the internet on the same LAMP stack as Box 2. WordPress's real attack surface, though, is its plugin ecosystem: thousands of third party extensions, any one of which can compromise the entire server if vulnerable.

The plugin here is Backup Migration v1.3.7, a legitimate backup/migration tool with 80,000+ real installs. The bug, CVE-2023-6553 (CVSS 9.8), lives in backup-heart.php, which reads a path straight from the Content-Dir HTTP header and passes it unsanitized into PHP's include() a classic Local File Inclusion primitive.

On its own, LFI is already dangerous, but this one escalates to full RCE through a PHP filter chain: include() supports stream wrappers like php://filter, which apply encoding transformations to a data stream before it's read. Chaining dozens of these filters in a specific sequence forces PHP to generate and execute arbitrary code on the fly no upload, no second stage, no authentication. One crafted request is enough for a www-data shell.

Root comes from a deliberately misconfigured SUID binary: find has been given the SUID bit, so it runs as root regardless of caller. A single GTFOBins one-liner — find . -exec /bin/sh -p \; -quit — spawns a root shell.


### Box 4 : ORNN :

Box 4 has no web application it's a Linux box exposing three misconfigured network services, and it's the bridge into the rest of the internal network (and eventually the Windows machines).

NFS shares are defined in /etc/exports. The critical flaw here is no_root_squash: normally NFS demotes a connecting client's root user to an unprivileged anonymous account ("root squashing"). With that protection disabled, root on the attacking machine stays root on the share. An attacker mounts the share, drops a SUID binary on it, then executes it from the target to get a root shell. The share also contains a password protected ZIP archive holding the Windows credentials cracked offline with John the Ripper, no exploitation needed beyond that.

Samba 3.0.20 is vulnerable to CVE-2007-2447, the "usermap script" bug. Samba's username map script feature passes the connecting client's username to a shell script unsanitized. An attacker embeds shell commands in backticks inside the username field, and Samba executes them as root, with zero authentication.

Redis is bound to 0.0.0.0 with no password. Its CONFIG SET dir / CONFIG SET dbfilename commands let an attacker redirect where Redis writes its database file. By pointing that write path at /root/.ssh/authorized_keys and saving an attacker-controlled SSH public key as a value, the attacker plants their own key in root's authorized_keys then SSHs in as root with no password and no exploit.

Finally, a sudo -l check on the user recovered from the cracked credentials reveals a passwordless sudo rule on nano  a documented GTFOBins escalation (drop to a shell from inside the editor) that gives a fourth, independent path to root.


### Box 5 : Rogue : 

Box 5 is a standard Windows 10 workstation with an intentionally minimal attack surface no web app, no exotic service. What makes it exploitable is that SMB is enabled (the default on virtually every Windows box) and a local account belongs to the Administrators group. The credentials for that account are the ones leaked from the NFS share on Box 4.

SMB is the protocol behind Windows file sharing, printer sharing, and remote administration. Using those credentials, NetExec (nxc) authenticates over SMB and remotely flips a registry key to enable RDP  no prior graphical access needed. From there, we connect over RDP as the local admin and land on the desktop.

Escalation to SYSTEM comes via RoguePlanet, a public exploit targeting a race condition in Windows Defender. The technique mounts a crafted ISO and abuses the timing window between when Defender writes a file and when it finishes scanning it  hijacking execution in that gap and inheriting Defender's SYSTEM-level privileges. No kernel exploit, no UAC bypass, no extra tooling  just the race.


### Box 6 : DC01 : 

The final box is a Windows Server 2019 Domain Controller the single point of trust for the whole domain, handling Kerberos authentication and hosting Active Directory. It also runs ADCS (Active Directory Certificate Services) with the Web Enrollment role, which exposes an HTTP certificate-request interface at http://DC/certsrv.

A low-privileged domain user exists on the DC, deliberately reusing the same credentials as the local admin on Box 5 a realistic case of password reuse across systems.

The attack abuses ESC8, one of the ADCS escalation paths from SpecterOps' "Certified Pre-Owned" research. Web Enrollment accepts NTLM authentication without Extended Protection for Authentication (EPA) enabled the default configuration  making it vulnerable to NTLM relay.

The chain runs in four steps:

- 1/ Coercion using the domain user's credentials, PetitPotam abuses MS-EFSRPC to force the DC to authenticate outbound to the attacker's machine using its own machine account.
  
- 2/ Relay  ntlmrelayx intercepts that NTLM authentication and relays it to the undefended Web Enrollment endpoint, requesting a certificate on behalf of the DC's machine account using the (enabled-by-default) Domain Controller certificate template.

- 3/ Authentication Certipy uses that certificate to authenticate via PKINIT, obtaining a TGT for the DC machine account.

- 4/ DCSync with that TGT, the attacker impersonates the DC and invokes MS-DRSR replication to dump every credential in the domain, including the krbtgt hash (Golden Ticket material) and the Domain Administrator hash.

The domain is fully compromised.


## Lab Build : 

### TLDR : 

All six boxes were built in VMware. Each machine is deliberately pinned to a specific OS, package, and where relevant kernel version, to guarantee the vulnerability is present and reproducible, and to make sure no automatic update silently patches it mid-lab. Below is the exact build for each box, in the order they're attacked.

**Box 1**
(Box 1 build Debian 12, kernel pin to 6.0.39, CopyFail verification already drafted)

**Box 2**
(Drupal 7.57 install, LAMP stack config)

**Box 3**
(WordPress install, Backup Migration v1.3.7 plugin, misconfigured SUID find)

**Box 4**
(NFS /etc/exports config with no_root_squash, Samba 3.0.20 usermap script config, Redis bound to 0.0.0.0, sudoers NOPASSWD rule on nano, credential ZIP setup)

**Box 5**
(Windows 10/11 workstation, local admin account, SMB enabled)

**Domain Controller**
(Windows Server 2019, ADCS + Web Enrollment install, domain user with reused creds, Certificate template config)


### Box 1 : Fail2Copy : 

#### ISO Installation : 

WEB01 runs on Debian 12 (Bookworm), with the kernel downgraded and pinned to a version vulnerable to CopyFail. On top of the OS, we install Node.js and npm, and deploy a Next.js application vulnerable to CVE-2025-55182, exposed on port 3000.

Now first we need the Debian Image , i downloaded it from this link : 

```bash
https://cdimage.debian.org/cdimage/archive/12.12.0/amd64/iso-dvd/
```

Once we Download it , we create a new VM : 

<img width="1159" height="513" alt="image" src="https://github.com/user-attachments/assets/7b2dc78b-2e15-46ab-8c64-4af7d55c5a01" />

Now once we start our VM , we can use Normal install instead of the graphical one to save time : 

<img width="1033" height="583" alt="image" src="https://github.com/user-attachments/assets/28520276-c15b-40bb-ac7a-9cbfdc46fd53" />

The installation is Pretty straight forward , Few things we will note : 

Hostname : 

<img width="1020" height="329" alt="image" src="https://github.com/user-attachments/assets/2bf732d5-049e-4fbf-85c9-f8f34b32a2f3" />

For the Domain name , we will keep it empty for now : 

<img width="960" height="461" alt="image" src="https://github.com/user-attachments/assets/d2e95c28-c694-423f-ad81-7b3fed7ec39f" />

Set up your root Password ;

<img width="1034" height="415" alt="image" src="https://github.com/user-attachments/assets/a12d1518-e882-401b-ac88-c5d218f15307" />

For the user we use : 

```bash
azerty : azerty
```

<img width="913" height="290" alt="image" src="https://github.com/user-attachments/assets/90a96a8a-e53f-4779-816d-c51e7c704f9d" />

Timezone in our case is Eastern . 

For the Disk : 

<img width="965" height="373" alt="image" src="https://github.com/user-attachments/assets/5cab424c-2bd6-4299-b9a6-4523269e57e7" />

For the Patitioning scheme : 

<img width="918" height="366" alt="image" src="https://github.com/user-attachments/assets/4ab7969a-cc57-4c60-a2a7-1eb98d5e9a65" />

Then we just confirm everything: 

<img width="1021" height="367" alt="image" src="https://github.com/user-attachments/assets/e3243468-5c6e-46ff-888f-28f68a7c1ed7" />

For the Extra Media Installations : 

<img width="925" height="423" alt="image" src="https://github.com/user-attachments/assets/13e4d802-833c-414c-bf1e-35a4e55293eb" />

We don't really need them for this lab . 

Since we need to install other dependencies and packages we need to enable the Network mirroring : 

<img width="951" height="358" alt="image" src="https://github.com/user-attachments/assets/3615902d-5fc3-4f58-83bb-67a73d189b84" />

For the mirror it can be anything you choose really : 

<img width="940" height="523" alt="image" src="https://github.com/user-attachments/assets/f0839ab4-5712-45f6-b59d-58fbf55277fa" />

Also just go with the default one : 

<img width="885" height="366" alt="image" src="https://github.com/user-attachments/assets/83a1de25-0376-4ec8-882f-7d61ad2760e4" />

We won't be needing any Proxies for this one :

<img width="971" height="274" alt="image" src="https://github.com/user-attachments/assets/9112f0e0-07fa-476a-abb9-ffdab86c05a9" />

Of course we need to add SSH so that we don't have to intall it manually later . 

<img width="910" height="372" alt="image" src="https://github.com/user-attachments/assets/13cc89f8-2762-4deb-8849-4b3e099d38bc" />

We should also install the Grub Boot Loader as well : 

<img width="995" height="338" alt="image" src="https://github.com/user-attachments/assets/a552261c-243c-42e6-9934-e88bcd3bda08" />

Then we specify the Device , we already put everything is 1 partition during Insallation so we should only see 1 : 

<img width="984" height="263" alt="image" src="https://github.com/user-attachments/assets/b9a0d028-35d8-4c01-b63c-62f632d83a38" />

Finally we select the Reboot option and we should be set : 

<img width="995" height="306" alt="image" src="https://github.com/user-attachments/assets/9fb5742c-49be-42be-852b-7b44863e66c9" />

#### Scenario  : 

##### Priv Esc : 

Now that everything is set , first thing we should setup is the path for the privilege escalation.

First we should check the version of the Kernel that was installed : 

<img width="1484" height="379" alt="image" src="https://github.com/user-attachments/assets/8fdc0252-9b46-4a61-9086-5bf4120e52fc" />

This version is patched and not vulnerable to CopyFail . 

So we will first start by installing a kernel version that is vulnerable . We first list all the available kernels that are installed . 

```bash
dpkg --list | grep linux-image
```

<img width="856" height="316" alt="image" src="https://github.com/user-attachments/assets/45797576-3b94-4d25-99af-1899e9c6e5c9" />

Perfect we see that we already have a kernel version that is vulnerable , already installed we just need to Boot from it . 

```bash
linux-image-6.1.0-39-amd64
```

In case you don't find it , you can easily install it . First update the packages , then check list all available kernels that we can install :

```bash
apt update
apt search linux-image-6.1
```

<img width="1237" height="704" alt="image" src="https://github.com/user-attachments/assets/89fc81d5-5690-4685-8ac5-5ed8b757906a" />

We're looking for Old versions for this one : 

```bash
apt install linux-image-6.1.0-39-amd64 
```

you should see this error : 

```bash
Media change: please insert the disc labeled                                                                                                                                                                      
 'Debian GNU/Linux 12.12.0 _Bookworm_ - Official amd64 DVD Binary-1 with firmware 20250906-15:05'
in the drive '/media/cdrom/' and press [Enter]
````

This means that APT is trying to fetch some dependencies from the Debian installation DVD, it means that that /etc/apt/sources.list still contained a cdrom: entry at that point.

You fix it by disabling the DVD repository and switching back to the official Debian mirrors:

```bash
nano /etc/apt/sources.list

===> Then you comment the First Repo :
# deb cdrom:[Debian GNU/Linux 12.12.0 _Bookworm_ ...]
```

Make sure you keep the other regular repos : 

<img width="1383" height="381" alt="image" src="https://github.com/user-attachments/assets/aba980d7-ed9f-4425-83db-1bbc61b1ce5a" />

From there we update the package cache and retry installing it : 

```bash
apt update
apt install linux-image-6.1.0-39-amd64 
```

<img width="981" height="445" alt="image" src="https://github.com/user-attachments/assets/d53b67f9-eb7b-4151-9cf4-e2c6b978ec53" />

Once we have the vulnerable kernel , we need to Boot it from Grub  . By default the bootloader will use the latest Kernel (updated one) so we first need to make sure we show the Grub Loader Menu and make sure we boot using the old Kernel : 

First check if Grub Menu is hidden and how long it takes before it boots : 

```bash
grep GRUB_TIMEOUT_STYLE /etc/default/grub
grep GRUB_TIMEOUT /etc/default/grub
```

<img width="678" height="215" alt="image" src="https://github.com/user-attachments/assets/8a4c34ec-82df-4fec-aed2-1bb85035327c" />

It's not hidden , and we have 5 Sec , before it boots with default entry : 

Now we just reboot it and Select Advanced Options in the Grub Loader menu : 

<img width="902" height="294" alt="image" src="https://github.com/user-attachments/assets/fa3fa1f1-96da-481e-89cc-024ab2463196" />

From there we slect the Vulnerable Kernel (.39) and boot with it : 

Now if we check the Kernel version : 

<img width="1227" height="301" alt="image" src="https://github.com/user-attachments/assets/4b7c1a8a-9b1e-4b51-9075-0b1bcd65042a" />

We see that it is the one vulnerable to CopyFail . 

##### Foothold : 

Now moving on to building the vulnerable Next.js application.

First, we need to install the JavaScript development environment.

The main components we need are:

- JavaScript (JS): The programming language used to write the application logic.
- Node.js: The runtime environment that allows JavaScript code to execute outside of the browser.
- npm (Node Package Manager): The package manager included with Node.js, used to install and manage application dependencies.
- Next.js: The framework used to build our application, based on React.
- React: The library responsible for creating the user interface components.

Before creating the vulnerable application, we need to install Node.js and npm on our Debian machine.

```bash
apt update
apt install nodejs npm -y
```

To verify : 

```bash
node --version
npm --version
```

<img width="1161" height="358" alt="image" src="https://github.com/user-attachments/assets/b9c1f91d-f247-4956-be8f-c03d312a4c6b" />

Now that Node.js and npm are installed, we can create the application that will simulate the vulnerable web service.

For realism, the application should not run as root. In a real-world deployment, web applications are usually executed under a dedicated service account with limited permissions.

First, we create a dedicated user for the application:

```bash
useradd -m -s /bin/bash reactlab
passwd reactlab # To give the user a password
su - reactlab
id # Verify 
```

<img width="1056" height="383" alt="image" src="https://github.com/user-attachments/assets/082441d4-4890-4cef-912e-b2661e56e838" />

Now we create the app : 

First we initialize the Node.js project :

```bash
npm init -y
```

npm init creates the initial configuration file for a Node.js project.

After running this command, a file called: *package.json* is created.

<img width="1193" height="514" alt="image" src="https://github.com/user-attachments/assets/a4dddd82-914d-4b70-b1fb-054d28dc69d7" />

package.json is the central configuration file of a Node.js application. It contains:

Project information (name, version, description)
Available commands (scripts)
Installed dependencies and their versions

For example, later it will tell npm how to start our application:

```js
"scripts": {
  "dev": "next dev"
}
```

Now we install the components required to run our vulnerable application:

```bash
npm install next@15.0.4 react@19.0.0 react-dom@19.0.0
```

This installs three packages:

**Next.js** (next@15.0.4)

Next.js is the framework responsible for running the web application.

It provides:

- The web server
- Routing
- Server-side rendering
- React Server Components support

This is the component exposing the vulnerable server-side functionality used in this lab.

**React** (react@19.0.0)

React is the UI library used by Next.js.

It allows developers to build applications using reusable components.

Example:

```js
<h1>Forged.corp</h1>
```

is a React component output.

**React DOM** (react-dom@19.0.0)

React DOM provides the integration between React and the web environment.

In a browser application, it is responsible for rendering React components into HTML.

After installation, npm creates:

- node_modules/
- package.json
- package-lock.json

<img width="1134" height="599" alt="image" src="https://github.com/user-attachments/assets/5c8dd7a7-a94f-4243-b0a9-aa7e98604d23" />

**node_modules :**

This directory contains the actual source code of all installed dependencies.

For example:

```bash
node_modules/
 ├── next/
 ├── react/
 └── react-dom/
````

<img width="926" height="276" alt="image" src="https://github.com/user-attachments/assets/39002ccd-35e7-45f0-aa3f-d43f5da5b975" />

**Package-lock.json :** 

This file records the exact versions of every installed dependency.

It ensures that if someone else downloads the project later, they install the same versions and reproduce the same environment.

To confirm the vulnerable versions were installed : 

```bash
npm list next react react-dom
```

Expected Result : 

```bash
reactlab@1.0.0 /home/reactlab
├─┬ next@15.0.4
│ ├── react-dom@19.0.0 deduped
│ ├── react@19.0.0 deduped
│ └─┬ styled-jsx@5.1.6
│   └── react@19.0.0 deduped
├─┬ react-dom@19.0.0
│ └── react@19.0.0 deduped
└── react@19.0.0
```

<img width="995" height="398" alt="image" src="https://github.com/user-attachments/assets/24e9ba05-b8e1-4c31-b2e8-b3a38871d83c" />

This step is important because npm can sometimes install newer compatible versions if versions are not explicitly pinned.

Now back to our app directory , the structure should look like this :

```bash
reactlab@Fail2Copy:~$ pwd
/home/reactlab
reactlab@Fail2Copy:~$ ls
node_modules  package.json  package-lock.json
reactlab@Fail2Copy:~$ 
```

**Next.js** provides two routing systems:

- Pages Router (older)
- App Router (newer)

For this lab, we use the App Router because it uses React Server Components.

First we create the application directory:

```bash
mkdir app
```

The project now looks like:

```bash
React-app/
├── app/
├── package.json
├── package-lock.json
└── node_modules/
```

Now let's Create the main page :

**Create:**

```bash
nano app/page.jsx
```

The file *app/page.jsx* represents the main page of the application.

In Next.js App Router:

- The folder name defines the route.
- The file page.jsx defines what is displayed.

For example:

```bash
app/page.jsx
==> becomes:
http://localhost:3000/
```

Feel free to customize your UI as you want , i won't be pasting the entire Front end code since it won't be necessary , but anything will be good for this demo really .

Now moving on to the Layout , Creating the root layout

The layout file defines the HTML structure shared by all pages.

**Create:**

```bash
nano app/layout.jsx
```

Inside the file Add:

```js
export const metadata = {
  title: "ISOLATE Cybersecurity",
  description: "Advanced cybersecurity research and defense platform",
};


export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        {children}
      </body>
    </html>
  );
}
```

Now for some styling (Not needed but in case you wanted to add it ) :

Create the global CSS file:

```bash
nano app/globals.css
```

Add :

```css
* {
  box-sizing: border-box;
}

body {
  margin: 0;
  background: #050816;
.....

```

**Importing the CSS file :**

```bash
nano app/layout.jsx
```

Add this at the top:

```js
import "./globals.css";
```

The file should now start with:

```js
import "./globals.css";

export const metadata = {
  title: "ISOLATE Cybersecurity",
  description: "Advanced cybersecurity research and defense platform",
};
....
```

<img width="999" height="830" alt="image" src="https://github.com/user-attachments/assets/7acc653b-6bf2-4eaa-a2da-f7346a085ad0" />

Add the npm scripts

Edit the **Package.json** file :

```bash
nano package.json
```

Make sure the scripts section contains:

```bash
"scripts": {
  "dev": "next dev",
  "build": "next build",
  "start": "next start"
}
```

These commands control how Next.js runs:

npm run dev → development server
npm run build → creates production build
npm run start → starts production server
Start the vulnerable application

Now checking the structure of Folders , it should look like this : 

```bash
React-app/
├── app/
│   ├── page.jsx
│   ├── layout.jsx
│   └── globals.css
├── package.json
├── package-lock.json
└── node_modules/
```

<img width="886" height="351" alt="image" src="https://github.com/user-attachments/assets/1eed94f4-21d0-4775-9498-7c2aff386c1b" />

Run as the low-privileged user:

```bash
npm run dev
```

<img width="689" height="332" alt="image" src="https://github.com/user-attachments/assets/d5d9ea98-dcfe-43d2-babb-f6115cfb9179" />

Now if we visit the localhost port 3000 : 

<img width="1830" height="858" alt="image" src="https://github.com/user-attachments/assets/51fb8562-d9a4-42c7-af9c-1b637e10d228" />

Our app is running perfectly . And we can access it from our Kali machine (But that's for the upcoming section) .

Now Box1 is done , let's move on to Box2 . 


### Box 2 : DirtyGeddon2 : 

#### ISO Installation : 

For this machine , we'll need one of these machine for the Dirty Frag exploit : 

Ubuntu 24.04.4: 6.17.0-23-generic
RHEL 10.1: 6.12.0-124.49.1.el10_1.x86_64
openSUSE Tumbleweed: 7.0.2-1-default
CentOS Stream 10: 6.12.0-224.el10.x86_64
AlmaLinux 10: 6.12.0-124.52.3.el10_1.x86_64
Fedora 44: 6.19.14-300.fc44.x86_64
....

In our case we will use the Ubuntu 24.04.4 . 

Here is the Link to Download the Image : 

```bash
https://releases.ubuntu.com/noble/
```

