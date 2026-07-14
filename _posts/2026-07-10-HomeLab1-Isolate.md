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

- Those leaked credentials get reused on **Box 5**, a Windows workstation, where the svc_backup account provides WinRM access. A misconfigured backup service running with elevated privileges exposes an unquoted service path vulnerability, allowing the service binary to be replaced and executed under the Administrator context. From there, RDP provides a full graphical session, where RoguePlanet, a Windows Defender race-condition exploit, escalates the Administrator shell to SYSTEM.

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
/srv/share *(rw,sync,no_subtree_check)
```

Explanation:

- /srv/share → directory being shared
- * → allows any client to connect (use a specific IP in real env)
- rw → read/write access
- sync → ensures data is written safely before confirming
- no_subtree_check → improves performance and avoids path checking issues

We save the file then we apply the changes 

```bash
sudo exportfs -a
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
Password: N7!qV4#zL9@xP2$k

[Backup-Service]
Username: svc_backup
Password: backup123

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

Create the directories Redis needs :

```bash
sudo mkdir -p /usr/local/redis/etc
sudo mkdir -p /home/azerty/.ssh
```

Create the config file to expose Redis and point to the SSH directory :

```bash
sudo nano /usr/local/redis/etc/redis.conf
```

Inside of it we add these lines :

```bash
bind 0.0.0.0
protected-mode no
port 6379
dir /home/azerty/.ssh
dbfilename authorized_keys
```

- bind 0.0.0.0 exposes Redis on all interfaces.
- protected-mode no disables the safety guard.
- dir and dbfilename pre-configure where the database will be saved.

Kill any old Redis processes and start the new vulnerable version :

```bash
sudo pkill -f redis-server
sudo /usr/local/bin/redis-server /usr/local/redis/etc/redis.conf &
```

Then finally verify it's listenning :

```bash
redis-cli ping
```

It should respond with a PONG . 

<img width="1478" height="627" alt="image" src="https://github.com/user-attachments/assets/c0c84487-12e2-4331-a852-bf2d4e85aef1" />

Redis is now Set up . 

##### PrivEsc : 

**Path 1: Samba CVE-2007-2447**

Direct unauthenticated RCE as root. No escalation needed ,you're already root.

**Path 2: NFS + svc_backup credentials**

Create the svc_backup user with a home directory (this user will be found in the NFS credentials) :

```bash
sudo useradd -m svc_backup
```

Set the password to backup123 (this credential will be stored in the /srv/share/windows-creds.zip on the NFS share) :

```bash
sudo passwd svc_backup
```

Password can be anything , we're using : backup123

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

