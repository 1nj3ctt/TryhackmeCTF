# Mr Robot CTF

Let's make nmap scan

<img width="529" height="444" alt="Screenshot From 2026-01-01 18-58-11" src="https://github.com/user-attachments/assets/83040969-512b-46dd-b823-8687be534d23" />
<img width="1906" height="585" alt="Screenshot From 2026-01-01 18-58-35" src="https://github.com/user-attachments/assets/68530bfb-e0ad-41f0-a5e7-9b0278bc0143" />

Navigate to http://MACHINE-IP

<img width="1915" height="927" alt="Screenshot From 2026-01-01 18-56-38" src="https://github.com/user-attachments/assets/a94bdc60-1e62-460f-adb7-19e232841938" />

Let's check if the site has robots.txt file
<img width="1906" height="585" alt="Screenshot From 2026-01-01 18-59-16" src="https://github.com/user-attachments/assets/9547fad8-12e7-4fdd-81e3-96554c625d75" />

We find the first key and dictionary, let's download it!
<img width="1906" height="502" alt="Screenshot From 2026-01-01 19-01-04" src="https://github.com/user-attachments/assets/30207c94-b31b-4415-a55f-dae140385dd9" />

The dict contains many duplicates we can remove that by using this command

```bash
cat fsocity.dic | tee | sort | uniq > pass.txt
```

Let's run gobuster and see what hidden directories site has
<img width="1114" height="198" alt="Screenshot From 2026-01-01 19-05-24" src="https://github.com/user-attachments/assets/e320cf92-545f-4969-8aec-0a2dac056b20" />
<img width="552" height="366" alt="Screenshot From 2026-01-01 19-06-02" src="https://github.com/user-attachments/assets/21350ab1-7219-4a3b-9379-1ab7bf5949d7" />

The wp-login is very interesting path, by navigating you can see that this is wordpress login page
<img width="1919" height="895" alt="Screenshot From 2026-01-01 19-07-21" src="https://github.com/user-attachments/assets/2b93ac37-52b3-4d3e-9b88-cd59a97ef156" />

Now because this room is called "Mr robot CTF" i tried to guess the username based on the characters' names,
by inserting a username and invalid password and "elliot" is valid, otherwise we could to use such tools like hydra or wpscan to brute force the username.
<img width="1919" height="895" alt="Screenshot From 2026-01-01 19-12-09" src="https://github.com/user-attachments/assets/4e493534-8211-425b-a626-3dc8e081d927" />

Now we can use wpscan to brute force the password with username "elliot" with dict that we downloaded.
<img width="1919" height="343" alt="Screenshot From 2026-01-01 19-14-58" src="https://github.com/user-attachments/assets/504b3437-a899-421d-8601-1e03e71ad43a" />

We've successfuly found the valid combination!Let's login.
<img width="1919" height="896" alt="Screenshot From 2026-01-01 19-19-01" src="https://github.com/user-attachments/assets/8190c8e6-f404-426a-aa86-d22d22bd0de1" />

Now to achieve the RCE here go to appearance > Editor
<img width="1919" height="896" alt="Screenshot From 2026-01-01 19-22-49" src="https://github.com/user-attachments/assets/8a7718c6-9ccc-4a19-aa9e-bd84d84b3dfd" />

I will add my template in 404.php and this gonna be the php reverse shell.
If you are using kali it's available in /usr/share/webshells/php path, otherwise you can download this repository https://github.com/pentestmonkey/php-reverse-shell.

<img width="320" height="524" alt="Screenshot From 2026-01-01 19-27-31" src="https://github.com/user-attachments/assets/e47f3c96-22b9-40b7-b2b7-52c87a93e235" />

Change the ip variable to your ip, and port change to 4444 or you can leave so, then create a netcat listener.
<img width="346" height="81" alt="Screenshot From 2026-01-01 19-30-05" src="https://github.com/user-attachments/assets/08699f92-6a44-4b42-bcb1-d7ce7512967c" />

Because we added our exploit in 404.php, we need to trigger the webapp to return us 404 status code to achieve RCE, simply go to the path in app that do not exist.
<img width="1051" height="203" alt="Screenshot From 2026-01-01 19-33-21" src="https://github.com/user-attachments/assets/09880e87-1636-415c-95e6-d6fe863fef43" />

The second flag is in /home/robot, but we don't have permission to read it, but we have some interesting file that is containing a md5 hash and it's releated to robot, let's crack it by using Crackstation
<img width="1051" height="677" alt="Screenshot From 2026-01-01 19-35-05" src="https://github.com/user-attachments/assets/6b061843-7993-4641-b885-2912410bb07a" />
<img width="1063" height="102" alt="Screenshot From 2026-01-01 19-38-03" src="https://github.com/user-attachments/assets/8bc4c88a-f87e-4d48-96cd-484511e3202b" />

We can change our user to robot by using cracked password and now we can read our file that is containing our second key.
<img width="1488" height="160" alt="Screenshot From 2026-01-01 19-39-32" src="https://github.com/user-attachments/assets/b257766c-227d-4e2d-ba91-04172cd2e526" />

The third key is in /root directory, so we need to escalate our priveleges.
The basic vector is checking the binaries that have SUID set.
The following command will help us

```bash
find / -perm -4000 -type f 2>/dev/null
```
<img width="1488" height="373" alt="Screenshot From 2026-01-01 19-43-40" src="https://github.com/user-attachments/assets/582634c7-01e2-47af-846d-2197128c8219" />

Then we need to check each binary in GTFObins, and we can use nmap binary to escalate our priveleges

```bash
nmap --interactive
nmap> !sh
```

<img width="1488" height="137" alt="Screenshot From 2026-01-01 19-46-44" src="https://github.com/user-attachments/assets/d719eb30-0a54-430e-9182-26ae8680bdf3" />


## MITRE ATT&CK TTPs

### Reconnaissance
T1595.001 - Active Scanning: Scanning IP Blocks
Nmap scan to discover open ports and services.

T1592.002 - Gather Victim Host Information: Software
Identifying WordPress CMS via gobuster and directory enumeration.

### Discovery
T1083 - File and Directory Discovery
Checking robots.txt to discover hidden files (key, dictionary).

T1518.001 - Software Discovery: Security Software Discovery
Identifying WordPress version and login panel via gobuster.

### Credential Access
T1110.001 - Brute Force: Password Guessing
Guessing the username "elliot" based on context (Mr. Robot characters).

T1110.003 - Brute Force: Password Spraying
Using WPScan with fsocity.dic wordlist to brute-force the WordPress password.

T1110.002 - Brute Force: Password Cracking
Cracking the MD5 hash of robot user's password via CrackStation.

### Initial Access
T1078 - Valid Accounts
Logging into WordPress admin panel with discovered credentials (elliot).

### Execution
T1059.004 - Command and Scripting Interpreter: Unix Shell
PHP reverse shell injected into 404.php to gain remote code execution.

### Persistence
T1505.003 - Server Software Component: Web Shell
Injecting PHP reverse shell into WordPress theme file (404.php).

### Command & Control
T1071.001 - Application Layer Protocol: Web Protocols
Triggering the reverse shell over HTTP by navigating to a non-existent page.

### Privilege Escalation
T1548.001 - Abuse Elevation Control Mechanism: Setuid and Setgid
Discovering SUID binaries via: find / -perm -4000 -type f 2>/dev/null
Using nmap --interactive and !sh to spawn a root shell (GTFOBins).

### Lateral Movement
T1078 - Valid Accounts
Switching to the robot user using the cracked password via su robot.

