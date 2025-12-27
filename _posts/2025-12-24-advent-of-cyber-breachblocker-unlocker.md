---
layout: post
title: Advent of Cyber - BreachBlocker Unlocker
date: 2025-12-24 18:37 +0000
category: [Advent Of Cyber]
tags: [tryhackme, aoc, malware-analysis]
media_subpath: /assets/img/breachblocker/
published: false
---

For Side Quest 4 of Advent of Cyber we have BreachBlocker Unlocker. We have to break into Sir BreachBlockers phone. 

Lets start with an Nmap scan:
```bash
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-24 13:39 EST
Nmap scan report for 10.82.145.104
Host is up (0.022s latency).
Not shown: 65531 closed tcp ports (reset)
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 bc:90:cb:e2:38:9a:9b:db:20:41:83:f7:7e:6c:b9:d2 (ECDSA)
|_  256 b1:cf:e9:b7:85:32:27:5d:46:34:86:e4:fa:b7:38:c6 (ED25519)
25/tcp    open  smtp     Postfix smtpd
| smtp-commands: hostname, PIPELINING, SIZE 10240000, VRFY, ETRN, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING
|_ 2.0.0 Commands: AUTH BDAT DATA EHLO ETRN HELO HELP MAIL NOOP QUIT RCPT RSET STARTTLS VRFY XCLIENT XFORWARD
8443/tcp  open  ssl/http nginx 1.29.3
| ssl-cert: Subject: organizationName=Internet Widgits Pty Ltd/stateOrProvinceName=Some-State/countryName=AU
| Not valid before: 2025-12-11T05:00:31
|_Not valid after:  2026-12-11T05:00:31
|_http-title: Mobile Portal
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|   h2
|   http/1.1
|   http/1.0
|_  http/0.9
|_http-server-header: nginx/1.29.3
21337/tcp open  http     Werkzeug httpd 3.0.1 (Python 3.12.3)
|_http-title: Unlock Hopper's Memories
|_http-server-header: Werkzeug/3.0.1 Python/3.12.3
```

On 8443 we find a webpage that simulates our phone:

![phone](phone.png)
_Phone Interface_

## Recon

Theres a good few apps to search through, Hopflix, Hopsec Bank are locked behind a login form. Authenticator requires FaceId BUT in the settings app we can turn this off if we find out the phones passcode. 

That helps us work out what to attack first, we wont be able to get into the Bank until we know the phones passcode. So lets either try Hopflix or Authenticator first. 

## Hopflix

The hopflix app opens to a login page, here you can attempt to login which calls an api endpoint of **/api/check-credentials** and returns json

At first I tried to just alter the response from false to true, this did get me to the next page but it doesnt return the data so we will need to get the password.
```js
HTTP/2 200 OK
Server: nginx/1.29.3
Date: Wed, 24 Dec 2025 19:12:31 GMT
Content-Type: application/json
Content-Length: 45

{"error":"Incorrect Password","valid":true}
```
In the messages app I found a clue, it seems the password is something relating to rabbits:

![message](messages.png)
_Password is something to do with 'rabbits'_

As the **/api/check-credentials** endpoint doesnt have a rate limit, lets hit it with a bruteforce attack. For the wordlist I've taken all the passwords from rockyou with rabbit.

```bash
grep rabbit rockyou.txt | tee rockyou-rabbit.txt
```

## Authenticator

To get into the Authenticator app we need to verify with FaceId, as we don't have BreachBlockers face or a camera attached to our phone we cant do this. We can however disabled the FaceId verification in the Settings as long as we know the phones passcode.

I decided to go looking through the **main.js** file for interesting api endpoints and look what I found. 

```js
const PHONE_PASSCODE = "210701";
```

With that I disabled FaceId.

![disable-faceid](faceid-disabled.png)
_FaceId disabled for Authenticator_