# Attack Chain Playbook Cheatsheet

> Pick the matching playbook by target type; each playbook defines the standard path from initial access to objective achievement.

---

## Playbook 1: External Web Application → Domain Controller

```
1. Subdomain enumeration + port scanning
2. Web fingerprinting → find components with known vulnerabilities
3. Exploit to get a Webshell / RCE
4. Internal network info gathering (ipconfig/ifconfig, arp, net user)
5. Set up tunneling (frp/chisel/ssh)
6. Internal network scanning (live hosts, open ports)
7. Credential harvesting (mimikatz/hashdump/config files)
8. Lateral movement (PTH/WMI/PsExec)
9. Domain info gathering (BloodHound)
10. Domain privilege escalation (Kerberoasting/DCSync/constrained delegation)
11. Obtain domain controller access
```

**Key toolchain**: subfinder → httpx → nuclei → sqlmap/sstimap → frp → nmap → mimikatz → crackmapexec → bloodhound → certipy

---

## Playbook 2: Phishing → Internal Network Penetration

```
1. Target employee info gathering (LinkedIn/脉脉)
2. Craft the phishing email (spoofed sender/legitimate subject)
3. Build the payload (macro document/LNK/ISO/HTML smuggling)
4. Send the phishing email
5. Wait for callback (C2 beacon)
6. Local info gathering + privilege escalation
7. Credential extraction
8. Lateral movement
9. Persistence
10. Objective achieved
```

**Key toolchain**: theHarvester → gophish → msfvenom/cobalt-strike → mimikatz → bloodhound

---

## Playbook 3: Proximity Penetration (近源渗透) → Internal Network

```
1. Physical reconnaissance (WiFi signal, access control types, USB ports)
2. WiFi attack (Fluxion rogue AP / WPA cracking)
   or BadUSB implant (Rubber Ducky keystroke injection)
   or network implant (Raspberry Pi / LAN Turtle)
3. Obtain an internal network entry point
4. Internal network scanning
5. Continue with Playbook 1 steps 5-11
```

**Key toolchain**: fluxion/aircrack-ng → rubber-ducky → frp → nmap → crackmapexec

---

## Playbook 4: Cloud Environment Penetration

```
1. Cloud asset discovery (subdomain → CNAME → cloud provider)
2. Storage bucket enumeration (S3/OSS/Blob public access)
3. SSRF → cloud metadata (169.254.169.254)
4. Obtain temporary credentials (AK/SK/Token)
5. Cloud API enumeration (IAM/EC2/Lambda/RDS)
6. Privilege escalation (PassRole/AssumeRole)
7. Lateral movement (cross-account/cross-region)
8. Data acquisition
```

**Key toolchain**: subfinder → nuclei(ssrf) → aws-cli → pacu → ScoutSuite

---

## Playbook 5: Bug Bounty / SRC Quick Assessment

```
1. Asset collection (subdomains + ports + JS files)
2. Fingerprinting → quick validation of known vulnerabilities (nuclei)
3. Parameter discovery (arjun/paramspider)
4. Test category by category:
   - IDOR/privilege escalation (change ID/change role)
   - SSRF (internal probing/cloud metadata)
   - SQL injection (sqlmap)
   - XSS (xsstrike)
   - File upload (bypass detection)
   - Logic flaws (payment/verification code/password reset)
5. Write PoC + submit report
```

**Key toolchain**: subfinder → httpx → nuclei → arjun → sqlmap → xsstrike → burpsuite

---

## Playbook 6: AD CS Certificate Attacks

```
1. Discover AD CS services (certipy find)
2. Identify vulnerable templates (ESC1-ESC8)
3. Request a malicious certificate
4. Authenticate as the target user with the certificate
5. Obtain NTLM hashes or a TGT
6. DCSync to dump all credentials
```

**Key toolchain**: certipy → rubeus → mimikatz → secretsdump

---

## Generic Decision Matrix

| Current state | Next-step priority |
|---------|-------------|
| Domain name only | Subdomain enumeration → port scanning → Web fingerprinting |
| Web vulnerability found | Get a shell → internal info gathering |
| Low-privilege shell | Privilege escalation → credential extraction |
| One internal machine | Tunnel → internal scanning → lateral movement |
| Domain user credentials | BloodHound → find an attack path |
| Domain admin hash | DCSync → Golden Ticket |
| Cloud AK/SK | Enumerate permissions → escalate → data acquisition |
| Phishing callback | Local escalation → credentials → lateral movement |
| Proximity access | Internal scanning → same as above |
