---
description: Reverse Engineering an Insane Ransomware |  HTB DFIR
icon: lock
---

# LockPick4.0

Author : sn0xsharma

<figure><img src="../../../.gitbook/assets/image (281).png" alt=""><figcaption></figcaption></figure>

### Introduction

LockPick4.0 is a formidable multi-stage ransomware targeting Windows environments, showcasing a blend of social engineering, privilege escalation, and anti-analysis techniques. As a member of [Forela.org](http://forela.org)’s Incident Response (IR) team, I was tasked with dissecting this malicious payload to uncover its operational mechanics, extract Indicators of Compromise (IOCs), and formulate robust defensive strategies. This writeup provides an in-depth analysis of LockPick4.0’s infection chain, leveraging reverse engineering tools like IDA Pro, Sysmon, and Splunk, while addressing the challenge questions with precision.

The ransomware employs a virtual hard disk (VHDX) delivery mechanism, JScript and PowerShell payloads, DLL side-loading, and sophisticated anti-debugging measures. By meticulously analyzing each stage, we can expose its inner workings and provide actionable insights for detection and prevention. Let’s dive into the technical intricacies of this cyberthreat.

***

### Scenario: A Rapidly Escalating Crisis

[Forela.org](http://forela.org)’s IT Helpdesk reported a surge in system anomalies, with employees unable to access files and ominous ransom notes appearing within 30 minutes of the initial reports. The scale and speed of the attack pointed to a sophisticated ransomware campaign. My objective was to reverse engineer the malware sample, map its behavior, and extract IOCs to enhance [Forela.org](http://forela.org)’s detection capabilities and prevent further compromise.

***

### Stage 1: Delivery via Social Engineering and VHDX

LockPick4.0’s infection begins with a cunning delivery mechanism: a self-contained virtual hard disk (VHDX) file. This format is often overlooked by email attachment filters, allowing attackers to bypass traditional security controls. When a user double-clicks the VHDX, it mounts as a Windows disk drive, revealing two files in an Explorer window:

1.  **A text file** masquerading as an urgent directive from [Forela.org](http://forela.org)’s IT Security Team. It employs social engineering, exploiting authority and urgency to coax users into running a script named `defenderscan.js` for “security verification.” The message reads:

    ```
    Subject: Run defenderscan.js for Security Verification
    Dear Forela User,
    Our IT security team has identified a critical need to verify your workstation’s security integrity. Please run the defenderscan.js script immediately to scan for vulnerabilities and ensure network safety.
    ...
    How to Run: Double-click defenderscan.js in this folder.

    ```
2. **A JScript file (`defenderscan.js`)**, the initial malicious payload, designed to kickstart the infection chain.

This social engineering tactic preys on user trust, highlighting the need for robust phishing awareness training.

***

### Stage 2: JScript Execution (`defenderscan.js`)

The `defenderscan.js` file, executed via Windows Script Host (WSH) through `wscript.exe` or double-clicking, serves as the entry point for the ransomware’s multi-stage attack. Its key functions are:

1. **Decoding a Base64-Encoded Image**: Saves a decoy file, `redbadger.webp` (MD5: `2c92c3905a8b978787e83632739f7071`), likely used to maintain the pretext of legitimacy.
2. **Decoding a Base64-Encoded PowerShell Script**: Saves it as a randomly named `.tmp` file (e.g., `rad88127.tmp`).
3. **Dynamic Execution**: Uses `WScript.Shell` to invoke `powershell.exe` with `Invoke-Expression` to execute the decoded script.

Here’s a sanitized excerpt of the JScript code, illustrating its core logic:

```jsx
var fso = new ActiveXObject("Scripting.FileSystemObject");
var b64_image = "<base64-encoded image>";
var b64_ps = "<base64-encoded PowerShell>";

// Convert base64 to binary
function base64ToBytes(base64) {
    var dom = new ActiveXObject("MSXml2.DOMDocument");
    var elem = dom.createElement("Base64Data");
    elem.dataType = "bin.base64";
    elem.text = base64;
    return elem.nodeTypedValue;
}

// Write binary data to file
function writeBinary(binData, fileName) {
    var stream = new ActiveXObject("ADODB.Stream");
    stream.Type = 1; // Binary
    stream.Open();
    stream.Write(binData);
    stream.SaveToFile(fileName, 2);
    stream.Close();
    return true;
}

// Save decoy image
writeBinary(base64ToBytes(b64_image), "redbadger.webp");

// Decode and save PowerShell script
var psBin = base64ToBytes(b64_ps);
var psText = (new ActiveXObject("ADODB.Stream")).Type = 2, .Charset = "UTF-8", .Open(), .Write(psBin), .Position = 0, .ReadText();
var tmpPath = fso.GetAbsolutePathName(".") + "\\\\" + fso.GetTempName();
var tmpFile = fso.CreateTextFile(tmpPath, true);
tmpFile.Write(psText);
tmpFile.Close();

// Execute PowerShell
var shell = new ActiveXObject("WScript.Shell");
var cmd = `powershell.exe -ExecutionPolicy Bypass -Command "Invoke-Expression (Get-Content '${tmpPath}')"`;
shell.Run(cmd, 0, true);

```

To analyze safely, comment out the execution (`shell.Run`) and file deletion (`fso.DeleteFile`) lines to preserve artifacts for further inspection.

***

### Stage 3: PowerShell Payload (`rad88127.tmp`)

The PowerShell script, obfuscated with Lord of the Rings-themed function names (e.g., `Test-HobbitInCouncil`, `Test-MordorElevation`), orchestrates privilege escalation and payload deployment. Its key components are:

#### 1. Administrator Privilege Check

The script verifies if the user belongs to the local Administrators group (SID: `S-1-5-32-544`) or Domain Admins group using `whoami /groups`:

```powershell
function Test-HobbitInCouncil {
    try {
        $groups = whoami /groups
        $isLocalAdmin = $groups -match "S-1-5-32-544"
        $isDomainAdmin = $groups -match "Domain Admins"
        return $isLocalAdmin -or $isDomainAdmin
    } catch {
        Write-Error $_.Exception.Message
        return $false
    }
}

```

#### 2. Elevation Check

It checks for high-integrity context using the .NET `WindowsPrincipal` class:

```powershell
function Test-MordorElevation {
    try {
        $principal = [Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()
        return $principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
    } catch {
        return $false
    }
}

```

#### 3. UAC Bypass via CMSTP (MITRE ATT\&CK ID: T1218.003)

If not elevated, the script employs a UAC bypass using `cmstp.exe`, a signed Microsoft binary (`C:\\Windows\\System32\\cmstp.exe`, signer: **Microsoft Corporation**) for installing Connection Manager profiles. The bypass involves:

* Creating an INF file (`ScrollOfEru.inf`) with a `RunPreSetupCommandsSection` to re-execute `wscript.exe defenderscan.js`.
* Running `cmstp.exe /au ScrollOfEru.inf` to execute in a high-integrity context.
* Using Windows Forms to automate the Enter key press for the CMSTP prompt.

```
[RunPreSetupCommandsSection]
wscript.exe "%currentDirectory%\\%currentFilename%"
taskkill /IM cmstp.exe /F

```

This technique grants administrative privileges without triggering UAC prompts.

#### 4. Windows Defender Evasion

In high-integrity mode, the script adds exclusions to Windows Defender for:

* The process `msmpeng.exe`.
* The current directory.

#### 5. Alternate Data Streams (MITRE ATT\&CK ID: T1564.004)

The script extracts two files from NTFS Alternate Data Streams (ADS) embedded in `defenderscan.js`:

* `defenderscan.js:lolbin` → `msmpeng.exe`
* `defenderscan.js:payload` → `mpsvc.dll`

#### 6. DLL Side-Loading (MITRE ATT\&CK ID: T1574.002)

The script launches `msmpeng.exe`, a signed Windows Defender binary, which side-loads the malicious `mpsvc.dll`. This Living-Off-the-Land Binary (LOLBIN) technique evades detection by leveraging a trusted process.

***

### Stage 4: DLL Analysis (`mpsvc.dll`)

The `mpsvc.dll` file, with its entry point `ServiceCrtMain`, contains the ransomware’s core logic. Analysis in IDA Pro reveals a stripped binary (no debugging symbols) with advanced anti-analysis measures.

#### Anti-Debugging Techniques (Total: 3)

1.  **IsDebuggerPresent**:

    Checks for a debugger using the Windows API. If detected, the malware exits:

    ```
    call cs:IsDebuggerPresent
    jnz loc_7FF96A6E5ECC

    ```
2.  **Parent Process Verification**:

    Enumerates the parent process using Windows APIs, ensuring it is `powershell.exe`. If not, the malware terminates.
3.  **Custom Exception Handler (Windows API: `SetUnhandledExceptionFilter`)**:

    Sets a custom exception filter to detect debugging tools that intercept exceptions. Unexpected exception codes trigger termination.

#### Ransomware Configuration

A decrypted resource in the DLL specifies:

* **Target Directories**: User profile folders (Desktop, Documents, Downloads, Music, Pictures, Videos).
* **Target Extensions** (in order): `.doc`, `.docx`, `.xls`, `.xlsx`, `.ppt`, `.pptx`, `.pdf`, `.txt`, `.csv`, `.rtf`.
*   **Ransom Note**: A base64-encoded HTML file displayed to victims, directing them to a TOR-based customer service portal:

    ```
    RANSOMWARE DEMAND NOTICE
    We have encrypted your data and exfiltrated critical files. Please browse to yrwm7tdvrtejpx7ogfhax2xuxkqej2gjb634qwyoyabkt2eydssrad.onion:9001 using a TOR browser to communicate with the RedBadger customer service department.

    ```

#### Network Communication

The ransomware sends HTTP POST requests to a C2 server’s `/connect` endpoint. Breakpoints on `WinHTTPConnect` reveal the server’s FQDN (pending update). The POST data includes:

* Hostname (e.g., `DESKTOP-HQFUI02`).
* A passphrase.

The server responds with:

* A client identifier (e.g., `eb6bc`).
* An AES key and IV (e.g., `59b243640ec7fb078fb4180d66c85c591e3d6a28850a751b941`, `b4fa5d6c9a4a5`).

These are dynamically generated per execution, complicating file recovery.

#### File Encryption

The ransomware targets files in specified directories with the listed extensions, encrypting them with `BCryptEncrypt` and appending the `.evil` extension. Dynamic analysis with a test file (`ATestFile.txt`) confirmed the encryption process by setting a breakpoint on `BCryptEncrypt`. The input data (RDX register) matched the test file’s contents, and the output was saved as `ATestFile.evil`.

***

### Advanced Reverse Engineering Techniques

To dissect LockPick4.0, I employed:

* **Static Analysis**: Used IDA Pro to analyze `mpsvc.dll`, identifying `ServiceCrtMain` and anti-debugging logic.
* **Dynamic Analysis**: Ran the malware in a sandbox, setting breakpoints on `WinHTTPConnect`, `WinHTTPSendRequest`, and `BCryptEncrypt` to capture network traffic and encryption behavior.
* **Sysmon Logs**: Configured Sysmon to log file creations (EventCode=11) and registry modifications (EventCode=13) for detecting ADS and rapid file renaming.
* **Splunk Queries**: Developed queries to hunt for suspicious activities (see Detection Engineering section).

To bypass anti-debugging, I:

* Patched `IsDebuggerPresent` to return `False`.
* Spoofed the parent process to `powershell.exe`.
* Disabled the custom exception handler by modifying `SetUnhandledExceptionFilter`.

***

### Indicators of Compromise (IOCs)

* **MD5 Hash (`redbadger.webp`)**: `2c92c3905a8b978787e83632739f7071`
* **Local Administrator SID**: `S-1-5-32-544`
* **Ransomware Server FQDN**: (Pending update)
* **TOR Portal**: `yrwm7tdvrtejpx7ogfhax2xuxkqej2gjb634qwyoyabkt2eydssrad.onion:9001`
* **Encrypted File Extension**: `.evil`
* **Signed Binary Signer**: `Microsoft Corporation`
* **DLL Entry Point**: `ServiceCrtMain`
* **Target Extensions**: `.doc`, `.docx`, `.xls`, `.xlsx`, `.ppt`, `.pptx`, `.pdf`, `.txt`, `.csv`, `.rtf`

***

### Hardening and Detection Strategies

To mitigate LockPick4.0 and similar threats, implement the following:

* **Advanced Filtering**: Block VHDX attachments and scan for phishing signatures.
* **User Training**: Conduct regular workshops on recognizing social engineering.
* **Phishing Simulations**: Test employee responses with simulated attacks.

#### Script Execution Controls

*   **PowerShell Restrictions**: Enforce signed scripts and Constrained Language Mode:

    ```powershell
    Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy AllSigned
    $env:__PSLockdownPolicy = 4

    ```
* **WSH Hardening**: Disable or restrict `wscript.exe` and `cscript.exe` via Group Policy.

#### Endpoint Security

* **Antivirus/EDR**: Deploy solutions like CrowdStrike or Microsoft Defender with real-time monitoring.
*   **Application Whitelisting**: Use AppLocker to allow only trusted binaries:

    ```powershell
    New-AppLockerPolicy -FilePath "C:\\Program Files\\*" -RuleType Publisher -User Everyone

    ```
*   **Sysmon Configuration**: Log ADS and process creation events:

    ```xml
    <Sysmon schemaversion="4.81">
      <EventFiltering>
        <FileCreateStreamHash onmatch="include">
          <Rule name="ADS" condition="contains">:</Rule>
        </FileCreateStreamHash>
        <ProcessCreate onmatch="include">
          <Image condition="contains">wscript.exe</Image>
          <Image condition="contains">powershell.exe</Image>
        </ProcessCreate>
      </EventFiltering>
    </Sysmon>

    ```

#### Backup Strategy

* **Automated Backups**: Schedule daily backups of critical data.
* **Offsite Storage**: Use cloud solutions like Azure Blob Storage.
* **Testing**: Regularly validate restoration processes.
* **Volume Shadow Copies**: Enable for quick recovery.

#### Detection Engineering

Leverage Splunk for proactive threat hunting:

*   **Monitor Script Execution**:

    ```
    index=your_index sourcetype=your_sourcetype (process_name="wscript.exe" OR process_name="cscript.exe" OR process_name="powershell.exe")
    | stats count by host, user, process_name, parent_process_name, process_path, command_line, _time
    | sort -_time

    ```
*   **Hunt for ADS**:

    ```
    index=your_index sourcetype=xmlwineventlog:Microsoft-Windows-Sysmon/Operational EventCode=11
    | eval ADS=if(match(TargetFilename, ":.+$"), "true", "false")
    | search ADS="true"
    | stats count by host, user, TargetFilename, ProcessId, ProcessGuid, Image, CommandLine, _time
    | sort -_time

    ```
*   **Detect Rapid File Renaming**:

    ```
    index=your_index sourcetype=xmlwineventlog:Microsoft-Windows-Sysmon/Operational EventCode=13
    | bin _time span=1m
    | stats count by host, user, _time, SourceImage, TargetFilename, ProcessGuid, Image, CommandLine
    | where count > 50
    | sort -_time

    ```

***

### Challenge Questions and Answers

1. **MD5 hash of the first file written to disk?**
   * `2c92c3905a8b978787e83632739f7071` (`redbadger.webp`)
2. **String used to check for local administrator privileges?**
   * `S-1-5-32-544`
3. **MITRE ATT\&CK ID for privilege elevation?**
   * \<REDUCTED\_GO\_FIND\_BR0\_:)> (CMSTP UAC bypass)
4. **MITRE ATT\&CK ID for hiding the final payload?**
   * `T1564.004` (NTFS Alternate Data Streams)
5. **Full name of the signer of the signed binary?**
   * `Microsoft Corporation`
6. **Final payload’s entry point function?**
   * `ServiceCrtMain`
7. **Number of anti-debugging techniques and the Windows API function for the final technique?**
   * `3,` `SetUnhandledExceptionFilter`
8. **List of targeted file extensions in order?**
   * `.doc`, `.docx`, `.xls`, `.xlsx`, `.ppt`, `.pptx`, `.pdf`, `.txt`, `.csv`, `.rtf`
9. **FQDN of the ransomware server?**
   * \<REDUCTED\_GO\_FIND\_BR0\_:)>
10. **MITRE ATT\&CK ID for running the final payload?**
    * `T1574.002` (DLL Side-Loading)
11. **Full URL of the ransomware group’s customer service portal?**
    * `yrwm7tdvrtejpx7ogf{<REDUCTED_GO_FIND_BR0_:)>}bkt2eydssrad.onion:9001`
12. **File extension for encrypted files?**
    * `.evil`

***

### Conclusion

LockPick4.0 exemplifies the sophistication of modern ransomware, combining social engineering, UAC bypass, DLL side-loading, and anti-debugging to devastating effect. Through meticulous reverse engineering, we uncovered its infection chain, extracted critical IOCs, and proposed comprehensive mitigation strategies. Organizations must adopt layered defenses—email filtering, script restrictions, endpoint protection, and robust backups—while leveraging advanced detection tools like Sysmon and Splunk. This analysis highlights the importance of proactive threat hunting and user education in combating evolving cyberthreats.

<figure><img src="../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
