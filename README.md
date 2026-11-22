# Cybersecurity Case Study: Internal Network Penetration Testing & Lateral Movement

## 1. Project Overview

  -----------------------------------------------------------------------------------------------
  Item                                Details
  ----------------------------------- -----------------------------------------------------------
  **Project Title**                   Internal Network Penetration Testing & Lateral Movement
                                      Case Study

  **Lab Platform**                    TryHackMe

  **Room**                            [Basic
                                      Pentesting](https://tryhackme.com/room/basicpentestingjt)

  **Assessment Type**                 Internal Network Security Assessment

  **Objective**                       Identify exposed services, enumerate potential attack
                                      vectors, exploit credential weaknesses, perform lateral
                                      movement, and retrieve sensitive information in a
                                      controlled lab environment.

  **Target IP**                       `10.x.x.x` (anonymized)

  **Operating System**                Linux (Ubuntu; version not independently verified)

  **Domain / Workgroup**              `WORKGROUP`

  **Scope**                           The assigned TryHackMe lab target only
  -----------------------------------------------------------------------------------------------

### Tools Used

-   Nmap
-   Gobuster
-   Enum4linux
-   Hydra
-   SSH
-   LinPEAS
-   John the Ripper
-   `ssh2john.py`
-   SCP

------------------------------------------------------------------------

## 2. Reconnaissance and Enumeration

The reconnaissance phase focused on identifying network services,
exposed web directories, and potentially valid user accounts on the
target.

### 2.1 Port and Service Enumeration

A service-version scan was performed using Nmap to identify exposed
ports and running services.

**Command:**

``` bash
nmap -sV 10.x.x.x
```

**Identified services:**

  Port         Service   Version / Details
  ------------ --------- ---------------------
  `22/tcp`     SSH       OpenSSH 8.2p1
  `80/tcp`     HTTP      Apache 2.4.41
  `139/tcp`    SMB       Samba
  `445/tcp`    SMB       Samba
  `8009/tcp`   AJP       AJP13
  `8080/tcp`   HTTP      Apache Tomcat 9.0.7

**Finding:** Several services were exposed, providing multiple potential
enumeration and attack surfaces. SSH and SMB were selected for further
investigation, alongside the web service.

### 2.2 Web Directory Enumeration

Gobuster was used to discover accessible directories and potentially
hidden web content.

**Command:**

``` bash
gobuster dir -u http://10.x.x.x -w /usr/share/wordlists/dirb/common.txt
```

**Finding:**

-   Discovered `/development/`
-   HTTP status code: `301` (redirect)

The discovered directory was investigated for information that could
assist subsequent enumeration.

### 2.3 SMB Enumeration

Enum4linux was used to gather information from the SMB service,
including workgroup information and user enumeration.

**Command:**

``` bash
enum4linux -a 10.x.x.x
```

**Finding:**

The enumeration identified the following user accounts:

-   `jan`
-   `kay`
-   `ubuntu`

These usernames were used to guide subsequent credential testing.

------------------------------------------------------------------------

## 3. Initial Access

### 3.1 Vulnerability Identified

The SSH service allowed authentication using a weak password associated
with the account `jan`.

**Attack Vector:** Weak account credentials\
**Affected Service:** SSH (`22/tcp`)\
**Affected Account:** `jan`

### 3.2 Credential Testing

In the authorized lab environment, Hydra was used to test the identified
account against a password wordlist.

**Command:**

``` bash
hydra -l jan -P /usr/share/wordlists/rockyou.txt ssh://10.x.x.x
```

**Result:**

  Username   Password
  ---------- -----------
  `jan`      `armando`

The password was successfully identified during the lab exercise.

### 3.3 SSH Authentication

The credentials were then used to authenticate to the target over SSH.

**Command:**

``` bash
ssh jan@10.x.x.x
```

**Result:**

Successful authentication provided a shell under the `jan` account.

``` text
jan@ip-10-x-x-x:~$
```

**Security Impact:** An attacker able to reach the SSH service could
gain unauthorized access to a local user account through password
guessing if weak credentials were used.

------------------------------------------------------------------------

## 4. Post-Exploitation and Lateral Movement

After obtaining initial access, local enumeration was performed to
identify additional accounts, accessible files, and potential paths to
other users.

### 4.1 Local Account and File Enumeration

Local account information was reviewed to identify user accounts and
their home directories.

**Areas examined:**

-   `/etc/passwd`
-   `/home/jan`
-   `/home/kay`

The enumeration confirmed the presence of additional user profiles,
including `kay`.

### 4.2 LinPEAS Enumeration

LinPEAS was transferred to a temporary directory on the target for local
security enumeration.

**Transfer command:**

``` bash
scp linpeas.sh jan@10.x.x.x:/dev/shm/
```

The script was used to inspect local configuration, file permissions,
credentials, and potential privilege-escalation paths.

### 4.3 SSH Private Key Discovery

During local enumeration, an SSH private key belonging to the `kay`
account was discovered at:

``` text
/home/kay/.ssh/id_rsa
```

The key was readable from the compromised account, indicating inadequate
access restrictions on sensitive SSH authentication material.

**Finding:** The private key was accessible to an unauthorized local
user.

**Security Impact:** Exposure of a private SSH key can enable
unauthorized authentication to the associated account, particularly if
the key's passphrase is weak or can be recovered.

### 4.4 Private Key Passphrase Recovery

The SSH private key was converted into a format suitable for offline
password cracking using `ssh2john.py`.

**Command:**

``` bash
/usr/share/john/ssh2john.py kay_id_rsa > forjohn.txt
```

John the Ripper was then used to test candidate passphrases against the
converted hash.

**Command:**

``` bash
john forjohn.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Result:**

  Item                   Result
  ---------------------- ----------------------------
  Target account         `kay`
  Credential type        SSH private key passphrase
  Recovered passphrase   `beeswax`

The passphrase was successfully recovered during the lab exercise.

### 4.5 Lateral Movement to the `kay` Account

The recovered private key and passphrase were used to authenticate as
`kay`.

**Command:**

``` bash
ssh -i kay_id_rsa kay@10.x.x.x
```

**Result:**

Authentication as the `kay` account was successful, demonstrating
lateral movement from the initially compromised `jan` account to another
local user account.

### 4.6 Sensitive File Retrieval

After accessing the `kay` account, a backup file named `pass.bak` was
located and read.

**Command:**

``` bash
cat pass.bak
```

**Recovered content:**

``` text
heresareallystrongpasswordthatfollowsthepasswordpolicy$$
```

**Finding:** A sensitive credential was stored in a backup file in a
readable form.

**Security Impact:** Plain-text credential storage can expose passwords
to anyone who obtains access to the file or the account that can read
it.

------------------------------------------------------------------------

## 5. Findings and Risk Assessment

  -----------------------------------------------------------------------
  ID                Finding           Evidence          Risk
  ----------------- ----------------- ----------------- -----------------
  F-01              Weak SSH password Password for      High
                                      `jan` recovered   
                                      through wordlist  
                                      testing           

  F-02              Excessive access  `jan` could       High
                    to SSH private    access `kay`'s    
                    key               private key       

  F-03              Weak SSH key      Passphrase        High
                    passphrase        recovered through 
                                      offline cracking  

  F-04              Sensitive         `pass.bak`        High
                    credential stored contained a       
                    in backup file    password in       
                                      readable form     

  F-05              Multiple exposed  SSH, HTTP, SMB,   Medium; depends
                    network services  AJP, and Tomcat   on exposure and
                                      services          configuration
                                      identified        
  -----------------------------------------------------------------------

### Overall Risk

**Overall Assessment: High**

The combined weaknesses enabled an attacker to obtain initial access to
a user account, access another user's SSH authentication material, move
laterally to that account, and retrieve sensitive credentials.

The severity of individual findings may vary depending on the network
exposure, account privileges, credential reuse, and the sensitivity of
the recovered information.

------------------------------------------------------------------------

## 6. Remediation Recommendations

### 6.1 Enforce Strong Authentication

-   Enforce strong, unique passwords for all local accounts.
-   Block common and dictionary-based passwords.
-   Configure SSH to use key-based authentication where appropriate.
-   Disable password-based SSH authentication when operationally
    feasible and after validating key-based access.
-   Implement rate limiting and monitoring for repeated authentication
    failures.

### 6.2 Secure SSH Private Keys

-   Restrict private key permissions to the owning user (typically
    `600`).
-   Ensure SSH directories have appropriate permissions (typically
    `700`).
-   Prevent other local users from reading private keys.
-   Protect private keys with strong, unique passphrases.
-   Rotate any key that may have been exposed.

### 6.3 Apply the Principle of Least Privilege

-   Restrict access to user home directories and sensitive files.
-   Review ownership and permissions across `/home`.
-   Prevent users from accessing files belonging to other accounts
    unless explicitly required.
-   Audit sudo privileges and other account-level permissions.

### 6.4 Secure Credential Storage

-   Do not store passwords in plain-text backup files.
-   Remove exposed credentials from files, backups, and repositories.
-   Use an approved secrets-management solution where applicable.
-   Rotate credentials that were exposed or potentially compromised.
-   Ensure backup files have appropriate ownership, permissions, and
    encryption.

### 6.5 Reduce Network Attack Surface

-   Disable unused network services.
-   Restrict service access using firewall rules and network
    segmentation.
-   Review the security configuration of SMB, AJP, Tomcat, HTTP, and
    SSH.
-   Keep exposed services and operating system packages patched.
-   Monitor network traffic and authentication logs for suspicious
    activity.

------------------------------------------------------------------------

## 7. Lessons Learned

This assessment demonstrated how several seemingly separate weaknesses
can be combined into a successful attack path.

1.  **Reconnaissance is essential:** Service discovery and user
    enumeration provided information that narrowed the scope of
    subsequent testing.
2.  **Weak credentials create an initial foothold:** A weak SSH password
    enabled access to the target.
3.  **Local file permissions matter:** Improper access to another user's
    private key enabled credential compromise.
4.  **Offline cracking can expose weak passphrases:** A recovered
    private key is especially dangerous when its passphrase is weak.
5.  **Lateral movement increases the impact:** Access to a second
    account expanded the attacker's access to sensitive information.
6.  **Credential hygiene is critical:** Plain-text passwords in backup
    files can undermine otherwise strong authentication controls.

------------------------------------------------------------------------

## 8. Conclusion

The internal network penetration testing lab identified multiple
weaknesses involving exposed services, weak authentication, improper
file permissions, SSH key security, and sensitive credential storage.

Initial access was obtained through the compromised SSH credentials of
the `jan` account. Further enumeration uncovered an SSH private key
associated with `kay`, and its weak passphrase was recovered through
offline cracking. This enabled lateral movement to the `kay` account,
where a sensitive backup credential was found.

The assessment highlights the importance of strong authentication,
secure handling of private keys, proper file permissions,
least-privilege access, and secure credential storage. Addressing these
weaknesses would reduce the likelihood of initial compromise and limit
an attacker's ability to move laterally within an internal environment.

------------------------------------------------------------------------

## 9. Evidence Checklist

Attach screenshots or terminal output for the following evidence where
available:

-   [ ] Nmap service scan results
-   [ ] Gobuster directory enumeration results
-   [ ] Enum4linux user enumeration results
-   [ ] Hydra result showing the recovered lab credential
-   [ ] Successful SSH login as `jan`
-   [ ] LinPEAS output identifying relevant findings
-   [ ] Discovery of the `kay` SSH private key
-   [ ] `ssh2john.py` conversion output
-   [ ] John the Ripper passphrase recovery output
-   [ ] Successful SSH login as `kay`
-   [ ] `pass.bak` file discovery and contents

> **Disclaimer:** This assessment was conducted within the authorized
> TryHackMe Basic Pentesting lab environment. The target IP and
> identifying information have been anonymized. The techniques and
> results documented here are for educational and authorized security
> testing purposes only.
