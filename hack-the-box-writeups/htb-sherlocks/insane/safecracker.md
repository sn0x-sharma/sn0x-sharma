---
icon: shield-plus
---

# SAFECRACKER

<figure><img src="../../../.gitbook/assets/image (280).png" alt=""><figcaption></figcaption></figure>

**Difficulty**: Insane\
**Type**: Forensics, DFIR, Malware Reverse Engineering\
**Platform**: Windows + Linux ELF (packer + payload)\
**Author(s)**: Blitztide, Sebh24\
**Write-up by**: sn0x sharma

***

### Task Scenario

We recently hired some contractors to continue the development of our backup services hosted on a Windows server. These contractors were provided with domain accounts.

However, when the system administrator logged in recently, several critical files were found encrypted, alongside a ransom note left by attackers.

We suspect a ransomware attack. Our existing tooling didn’t detect any of the attacker’s actions, suggesting an advanced threat.

Our mission is to investigate the attack and reverse-engineer the malware involved. The archive **safecracker.zip** is provided as forensic evidence, and the password to open it is `hacktheblue`.

***

### 1. Initial Access & Preparation

After extracting the contents of `safecracker.zip`, we find a file named `DANGER.txt`, which warns about **live ransomware** binaries within. We proceed with analysis inside a **controlled VM environment**.

#### File Summary:

* File: `safecracker.zip`
* Hash: `61EA7F259763C0937BEE773A16F4CE84`
* Contains: Windows log files, memory artifacts, and the malicious binary

***

### 2. Filesystem & MFT Analysis

We begin our analysis with `$MFT` using **MFTECmd**:

```bash
MFTECmd.exe -f $MFT --csv Safecracker
```

Opened the generated CSV in **Timeline Explorer**. Two user profiles were identified:

* `contractor01`
* `Administrator`

A directory named `Backups` was found under the `Administrator` profile, containing files with `.31337` and `.note` extensions.

#### Ransom Note Discovered:

```
You have been hacked by Cybergang31337
Please deposit $200,000 in BTC to: 16ftSEQ4ctQFDtVZiUBusQUjRrGhM3JYwe
Email: decryption@cybergang31337.hacker
```

We then filtered the MFT to find all `.31337` encrypted files:\
**Total Encrypted Files**: 33

***

### 3. Windows Event Log Analysis

Using **EvtxECmd**:

```bash
EvtxECmd.exe -d $LOG_FOLDER --csv $EVENTS
```

From the console history of `contractor01`, the attacker executed:

```powershell
.\PsExec64.exe -s -i cmd.exe
```

This escalated the user to `NT\SYSTEM`.

We pivot to Event ID `4624` for Remote Desktop logins (Logon Type 10). A successful RDP login occurred at:

* `DateTime`: 2023-06-21 13:01:00

***

### 4. WSL (Windows Subsystem for Linux) Detection

Hyper-V Compute logs show evidence of WSL being started at:

```xml
TimeCreated = 2023-06-21T13:01:07
Path = C:\Users\Administrator\AppData\Local\Packages\CanonicalGroupLimited.Ubuntu20.04onWindows\...\ext4.vhdx
```

The attacker mounted a virtual disk with an Ubuntu environment, likely used to stage and deploy the payload.

***

### 5. Discovery of Suspicious File: `MsMpEng.exe`

This file was found under:

```
C:\Users\Administrator\Downloads\MsMpEng.exe
```

Normally, `MsMpEng.exe` belongs to Windows Defender and should reside in `C:\Program Files\...`, not in Downloads.

Uploaded to VirusTotal — detected as a **Linux ELF binary**.

***

### 6. Static Analysis (Strings)

Command used:

```bash
strings MsMpEng.exe | awk 'length($0) > 9' | sort | uniq > /tmp/orig.strings
```

**Interesting Strings Identified:**

* `/home/blitztide/Projects/Payloads/LockPick3.0/lib/ssl`
* `inflate`, `deflate`, `zlib compression`
* `crypto/evp/e_aes.c`, `SHA512 block`, OpenSSL indicators

We suspect a **packer** written in C with OpenSSL and Zlib used for compression and encryption.

***

### 7. Behavior via strace

The binary performs:

* `memfd_create()`: creates in-memory file descriptor
* Writes a new **ELF binary**
* Executes it using `execl()`

The unpacked binary is stored in-memory, indicating **fileless execution**.

***

### 8. Extracting the Inner Binary

From reversing, the unpacked buffer begins at: `DAT_003893a0`

Relevant code:

```c
malloc_1 = malloc(size1);
malloc_2 = malloc(size2);
file_size = process(malloc_1, malloc_2);
memfd = memfd_create("test", 0x1);
write(memfd, malloc_2, file_size);
execl("/proc/self/fd/memfd", "PROGRAM", 0);
```

After dumping, we obtain a second ELF binary named `inner.elf`.

***

### 9. Decrypting the Packed Payload

From embedded constants and obfuscated data on Pastebin:

* **Key**: `a5f41376d435dc6c61ef9ddf2c4a9543c7d68ec746e690fe391bf1604362742f`
* **IV**: `95e61ead02c32dab646478048203fd0b`
* **Cipher**: AES-256-CBC
* **Compression**: zlib

Decryption & decompression done in **CyberChef** results in:

* Output File: `inner.elf`
* MD5: `57c4a3c06a461f96f1ed3a986f4e6016`

***

### 10. Reverse Engineering `inner.elf`

Filtered new strings:

```bash
strings inner.elf | awk 'length($0) > 9' > /tmp/new.strings
comm -1 -3 /tmp/orig.strings /tmp/new.strings
```

Found:

* VPN connection hints
* Libcurl debug messages
* Debugger string: `*******DEBUGGED********`

***

### 11. Anti-Debugging Check

`inner.elf` checks `/proc/self/status` for `TracerPid`:

* If `TracerPid != 0`, binary triggers `raise(SIGSEGV)`

Patch applied to bypass check:

```bash
echo -en "\xEB" | dd of=./inner.elf bs=1 seek=306664 conv=notrunc
```

***

### 12. strace Results & Syscalls

After patching:

* `getdents64`: lists directory contents
* `unlink`: deletes original files
* Drops `.note` files and appends `.31337` to encrypted files

***

### 13. URL Extraction & Key Hosting

Found obfuscated blob decoded via CyberChef:

* URL: `https://pastebin.com/raw/Wgnjffdi`
* Contents:

```
a5f41376...:95e61ead...
```

This hosted the **same key/IV** used in malware encryption.

***

### 14. Malware Behavior Summary

* **Infection vector**: RDP (contractor01)
* **Privilege Escalation**: PsExec to SYSTEM
* **Execution**: WSL → ELF → Packer → memfd → inner.elf
* **Encryption**: AES-256-CBC
* **Compression**: zlib
* **Persistence**: None observed
* **Evasion**: Anti-debug, fileless execution
* **Destruction**: 33 files renamed to `.31337` and notes dropped

***

### 15. Answers to Challenge Questions

| No. | Question                        | Answer                                 |
| --- | ------------------------------- | -------------------------------------- |
| 1   | Initial account used            | contractor01                           |
| 2   | SYSTEM escalation command       | .\PsExec64.exe -s -i cmd.exe           |
| 3   | Files encrypted                 | 33                                     |
| 4   | Process name of unpacked binary | PROGRAM                                |
| 5   | XOR key used                    | daV324982S3bh2                         |
| 6   | Encryption used                 | AES-256-CBC                            |
| 7   | Key and IV                      | `a5f41376... : 95e61ead...`            |
| 8   | memfd name                      | test                                   |
| 9   | Target directory                | /mnt/c/Users                           |
| 10  | Compression used                | zlib                                   |
| 11  | Debugger check file             | /proc/self/status                      |
| 12  | Exception raised                | SIGSEGV                                |
| 13  | Extension not encrypted         | .exe                                   |
| 14  | Compiler used                   | gcc                                    |
| 15  | Debugger string printed         | _**DEBUGGED**_\*                       |
| 16  | .comment section                | GCC: (Debian 10.2.1-6) 10.2.1 20210110 |
| 17  | Encrypted extension             | .31337                                 |
| 18  | Bitcoin address                 | 16ftSEQ4ctQFDtVZiUBusQUjRrGhM3JYwe     |
| 19  | Debugger string search          | TracerPid                              |
| 20  | Original hacker handle          | blitztide                              |
| 21  | Syscall used to list files      | getdents64                             |
| 22  | Syscall used to delete files    | unlink                                 |

***

### 16. Conclusion

Safecracker is a brilliantly constructed DFIR challenge combining real-world TTPs:

* Windows forensic analysis
* Privilege escalation
* WSL exploitation
* Custom Linux packer
* Memory execution
* Reversible ransomware payload

<figure><img src="../../../.gitbook/assets/complete (33).gif" alt=""><figcaption></figcaption></figure>



