---
icon: skull
cover: ../../../.gitbook/assets/Screenshot 2026-03-24 160435.png
coverY: -23.855141750514164
---

# NOVITAS

## Scenario What is Actually Happening Here

Before jumping into anything, let's actually understand what we're dealing with because this one has some real depth to it.

So there's this dude named Binz. He gets an email asking him to review some 3D model files for a client's family. He downloads them, opens one, notices weird behavior on his machine, panics, and deletes the files. But deleting files doesn't undo what already ran. So the SOC team captures a full memory dump (`memory.raw`) from the affected machine and hands it to us.

Our job is clear: dig through this memory dump, figure out exactly what ran, extract the malware from memory, reverse it, and pull out the IOCs (Indicators of Compromise) so the EDR team can build detections.

The twist here is the attack technique. With Microsoft locking down Office macros harder every year (MOTW, group policy blocks, etc.), attackers had to find a new "click this and you're owned" delivery mechanism. Enter **GrimResource** — a technique that weaponizes `.msc` (Microsoft Management Console Snap-in) files. These are legitimate Windows files that get opened by `mmc.exe`, a signed Windows binary. Nobody's suspicious of `mmc.exe`. That's exactly the point.

So the full chain is: phishing email → download ZIP → extract MSC file → open it → mmc.exe loads malicious .NET assembly → shellcode injected into dllhost.exe → Cobalt Strike beacon phones home. That's what we're about to tear apart piece by piece.

***

## Memory Analysis Getting the Lay of the Land

### Step 1.1: Confirm the OS and Architecture

The very first thing you do with any memory dump is figure out what you're even looking at. Is this Windows? 32-bit? 64-bit? What build? This matters because Volatility plugins, symbol paths, and offset math all depend on it.

```
┌──(sn0x㉿sn0x)-[~/HTB/Novitas]
└─$ volatility3 -f memory.raw windows.info.Info
```

```
Variable                    Value
Kernel Base                 0xf80437a00000
DTB                         0x1ad000
Is64Bit                     True
IsPAE                       False
Major/Minor                 15.19041
MachineType                 34404
KeNumberProcessors          4
SystemTime                  2024-09-05 16:01:34+00:00
NtSystemRoot                C:\Windows
NtMajorVersion              10
NtMinorVersion              0
```

`Is64Bit: True` tells us we're on a 64-bit system, which matters when you're looking at memory addresses they'll all be 64-bit pointers. `Major/Minor 15.19041` decodes to Windows 10 build 19041, which is the 2004/20H1 release. Good to know. System time is `2024-09-05 16:01:34 UTC`. We'll cross-reference this with process timestamps later.

***

### Q1: When Does the Suspicious Process Start?

#### Primary Method Process Tree via Volatility

Now we run the process tree plugin. This shows every running process and their parent-child relationships. The goal is to find something that looks out of place wrong parent, wrong path, or a legit-sounding binary doing something weird.

```
┌──(sn0x㉿sn0x)-[~/HTB/Novitas]
└─$ volatility3 -f memory.raw windows.pstree.PsTree
```

```
*** 3120   3144   mmc.exe   0xa78511394080   14   -   1   False
    2024-09-05 15:58:11.000000 UTC   N/A
    \Device\HarddiskVolume3\Windows\System32\mmc.exe
    "C:\Windows\system32\mmc.exe"
    "C:\Users\IEUser\AppData\Local\Temp\MicrosoftEdgeDownloads\
     91617dd3-f62f-4c28-ba7d-8769251040b3\family_image.msc"
    C:\Windows\system32\mmc.exe
```

There it is. `mmc.exe` (PID 3120) launched at `2024-09-05 15:58:11 UTC`, and its command line argument is a file called `family_image.msc` sitting inside Edge's temp downloads folder. That is not a normal place to be loading an MSC file from. `mmc.exe` itself is completely legitimate it's the Microsoft Management Console, used to load administrative snap-ins. But here it's being fed a malicious snap-in downloaded from the internet, and that's the entire GrimResource premise.

**Answer: `2024-09-05 15:58:11`**

#### Why This Matters

This single process tree entry tells us everything about the initial execution. The attacker didn't use powershell.exe or wscript.exe both of which have heavy detections. They used `mmc.exe`. Most SIEMs and EDRs don't flag `mmc.exe` spawns by default because sysadmins use it constantly. That's clever.

### Alternative Attack Vector How Else Could This Have Been Detected?

If you didn't have the process tree, you could also catch this through:

**Sysmon Event ID 1** (Process Create): Would log the full command line of `mmc.exe` including the path to `family_image.msc`. The presence of an MSC file in a `%TEMP%\MicrosoftEdgeDownloads\` path in the command line is an immediate red flag.

**Sysmon Event ID 11** (File Create): The moment the MSC file lands on disk from Edge, a file create event fires. If your SIEM has a rule watching for `.msc` files created in temp/download directories, this triggers before execution even happens.

**MRU Registry Keys**: Even after deletion, `HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\RecentDocs\.msc` would retain a reference to recently opened MSC files.

***

### Q2: What is the Size of the Archive File in Bytes?

#### Primary Method Browser History from Memory

We know the file came from Edge's temp download folder. So let's go look at Edge's history database. First we need to find it in memory.

```
┌──(sn0x㉿sn0x)-[~/HTB/Novitas]
└─$ volatility3 -f memory.raw windows.filescan.FileScan
```

```
0xa7850f527080   \Users\IEUser\AppData\Local\Microsoft\Edge\User Data\Default\History
```

Found the Edge history SQLite database at virtual address `0xa7850f527080`. Now we dump it out of memory.

```
┌──(sn0x㉿sn0x)-[~/HTB/Novitas]
└─$ volatility3 -f memory.raw windows.dumpfiles.DumpFiles --virtaddr 0xa7850f527080
```

```
DataSectionObject   0xa7850f527080   History
file.0xa7850f527080.0xa7850ecd0db0.DataSectionObject.History.dat

SharedCacheMap      0xa7850f527080   History
file.0xa7850f527080.0xa7850f1798a0.SharedCacheMap.History.vacb
```

Now we query it. You can use DB Browser for SQLite visually, or just python:

```
┌──(sn0x㉿sn0x)-[~/HTB/Novitas]
└─$ python3 scripts/sqlite.py
```

```python
import sqlite3

con = sqlite3.connect("file.0xa7850f527080.0xa7850ecd0db0.DataSectionObject.History.dat")
cur = con.cursor()
cur.execute("select * from downloads")
for i in cur.fetchall():
    print("File Name: " + str(i[3]))
    print("Total Bytes: " + str(i[5]))
```

```
File Name: C:\Users\IEUser\Downloads\family_image.zip
Total Bytes: 0

File Name: C:\Users\IEUser\AppData\Local\Temp\MicrosoftEdgeDownloads\
           91617dd3-f62f-4c28-ba7d-8769251040b3\family_image.zip
Total Bytes: 1971433
```

The first entry (Downloads folder) shows 0 bytes because Edge first creates a placeholder, then the actual download goes to the temp path and shows `1971433` bytes that's the real file size.

**Answer: `1971433`**

#### Why This Matters

The Edge history database is gold in browser-based infection scenarios. It not only gives us the file size but also timestamps for when the download started and completed, which helps build a precise infection timeline. The `danger_type` column in the downloads table can also tell you if Edge flagged the file as suspicious (spoiler: it probably did and the user clicked through).

#### Alternative Vector

If the history DB had been wiped, you could recover the same information from:

**Windows Prefetch** (`C:\Windows\Prefetch\`): `family_image.msc-{hash}.pf` would exist and contain a run timestamp plus file references.

**Zone.Identifier ADS**: Every file downloaded from the internet gets an Alternate Data Stream added called `Zone.Identifier` with `ZoneId=3` (Internet zone). Even if Edge history is gone, this stream on the ZIP file confirms browser-based download origin.

**USN Journal** (`$UsnJrnl`): The NTFS change journal logs file creates, deletes, and renames. The download and subsequent extraction of the ZIP would be logged here even if the files were deleted later.

***

### Q3: What Files Were Inside the Archive?

#### Primary Method | String Search in Memory

The ZIP was extracted, so those file names will exist somewhere in memory. We use Sysinternals `strings.exe` (or `strings` on Linux) to dump all readable strings and filter for our target.

```
┌──(sn0x㉿sn0x)-[~/HTB/Novitas]
└─$ strings.exe memory.raw | findstr family_image
```

```
family_image.zip
family_image.msc
[family_image.msc
family_image
C:\Users\IEUser\AppData\Local\Temp\MicrosoftEdgeDownloads\
91617dd3-f62f-4c28-ba7d-8769251040b3\family_image.obj
812D04EB0B02DF89\family_image.obj
family_image.obj
[family_image.msc
/C:\Users\IEUser\AppData\Local\Temp\MicrosoftEdgeDownloads\...family_image.msc
\IEUser\AppData\Local\Temp\MicrosoftEdgeDownloads\...\family_image.obj\
'family_image.zip
C:\Users\IEUser\AppData\Local\Temp\MicrosoftEdgeDownloads\...family_image.zip
```

Two distinct filenames show up: `family_image.msc` and `family_image.obj`. The `.msc` is the execution vector (the weaponized snap-in file), and the `.obj` is likely the payload component or embedded shellcode that the MSC file references and loads.

**Answer: `family_image.msc,family_image.obj`**

#### Why This Matters

The `.obj` file is interesting because `.obj` files are normally compiler output (object files before linking). Using that extension for a malicious payload is intentional misdirections it looks less suspicious to a casual observer than `.exe` or `.dll`.

***

## Deep Process Analysis

#### Setting Up MemProcFS

Before answering Q4 onwards, we need a better view into the `mmc.exe` process. MemProcFS mounts the memory dump as a virtual filesystem so you can browse it like a drive.

```
┌──(sn0x㉿sn0x)-[~/HTB/Novitas]
└─$ MemProcFS -f memory.raw
```

This mounts at `M:\`. We navigate to PID 3120 (mmc.exe) and grab its minidump:

```
M:\pid\3120\minidump\minidump.dmp
```

Now we have a process-level memory dump we can load into WinDbg or dotnet-dump for .NET analysis.

***

### Q4: How Many NAT (Native) Modules Are Loaded?

#### Primary Method , dotnet-dump Module Listing

The key concept here: inside a .NET process, modules fall into two categories. **NAT (Native) modules** are regular unmanaged DLLs — `ntdll.dll`, `kernel32.dll`, etc. **CLR modules** are managed .NET assemblies — things like `mscorlib.dll`, `System.dll`, and any custom assemblies loaded by the app.

We load the minidump into `dotnet-dump` and list all modules verbosely:

```
┌──(sn0x㉿sn0x)-[~/HTB/Novitas]
└─$ dotnet-dump analyze files\minidump.dmp
```

```
Loading core dump: files\minidump.dmp ...
Ready to process analysis commands.
> lm -v
```

This spits out 101 total modules. We look at the `IsManaged` field for each. Three of them have `IsManaged: True`, meaning they are CLR modules. The rest (101 - 3 = 98) are native.

**Answer: `98`**

#### Alternative via WinDbg `!peb`

You can also do this in WinDbg by loading the minidump and running `!peb` to see the full module list from the Process Environment Block, then manually counting. dotnet-dump is just faster and easier to script against.

***

### Q5: Assembly Addresses of All CLR Modules in Ascending Order

#### Primary Method `!dumpdomain` in dotnet-dump

`!dumpdomain` enumerates all .NET AppDomains in the process and shows every loaded assembly with its base address and metadata.

```
> !dumpdomain
```

The output shows the three managed (CLR) assemblies loaded in the process. We pull their assembly addresses and sort them ascending:

**Answer:**

```
0000000004E62FD0,0000000004E630F0,0000000004E63690,0000000004E638D0,0000000004E63B10
```

#### Why This Matters

AppDomain enumeration is one of the most powerful techniques for detecting in-memory .NET malware. Legitimate processes like `mmc.exe` load a known, stable set of .NET assemblies. Any extra assembly with a random GUID-style name, no PublicKeyToken, and no disk-resident file path is immediately suspicious. This is exactly what we find next.

***

### Q6: What is the Name of the Malicious Module?

#### Primary Method Anomaly in `!dumpdomain` Output

Looking through the `!dumpdomain` output, all the assemblies are standard Windows .NET system libraries except for one:

```
Assembly:   0000000004e63b10
            [Ad00bce9305554c87927205710b17699f, Version=1.0.0.0,
             Culture=neutral, PublicKeyToken=null]
Module Name:
00007ff894955b70   Ad00bce9305554c87927205710b17699f, Version=1.0.0.0,
                   Culture=neutral, PublicKeyToken=null
```

A few things immediately scream malicious here:

The name is a GUID-style hex string `Ad00bce9305554c87927205710b17699f`. Real .NET system libraries have human-readable names like `System.Xml` or `mscorlib`. Random hex names are a classic indicator of dynamically generated or obfuscated .NET payloads.

`PublicKeyToken=null` means the assembly is not strongly signed. All legitimate Microsoft .NET assemblies in the GAC are strongly signed. No strong name = custom assembly, no chain of trust.

There's no physical file path associated with it, meaning it was loaded entirely from memory never touched disk. This is reflective loading / in-memory execution.

**Answer: `Ad00bce9305554c87927205710b17699f`**

***

## Malware Extraction and Reversing

### Q7: MD5 Hash of the Dumped Malicious DLL

#### Primary Method | Raw Memory Dump via WinDbg `.writemem`

`!dlldump` in dotnet-dump can pull the module, but the resulting bytes are in PE layout which gets scrambled. The clean approach is to use WinDbg's `.writemem` to dump raw memory bytes directly from the known base address.

First, get the base address using `!dumpmodule` on the assembly address we found:

```
> !dumpmodule 0000000004e63b10
```

This tells us:

* Base address: `0x06630000`
* Size: `0x16200` bytes (calculated as `0x6646200 - 0x6630000 = 90624 bytes = 0x16200`)

Now in WinDbg, we write that memory region directly to a file:

```
0:000> .writemem output_dump.bin 0x0000000006630000 L0x16200
Writing 16200 bytes.....................................
```

Then hash it:

```
┌──(sn0x㉿sn0x)-[~/HTB/Novitas]
└─$ md5sum output_dump.bin
e67f5692a35b8e40049e30ad04c12b41 *output_dump.bin
```

**Answer: `e67f5692a35b8e40049e30ad04c12b41`**

#### Why `.writemem` Instead of `!dlldump`?

When a PE file loads into memory, Windows transforms the on-disk layout (file alignment) into the in-memory layout (section alignment). `!dlldump` tries to reconstruct the on-disk format and gets it wrong for dynamically generated assemblies that were never actually files. `.writemem` just copies raw bytes as they exist in memory, which is what you actually want when you're going to analyze execution behavior and hash the running payload.

***

### Q8: What is the XOR Key Used to Obfuscate Strings?

#### Primary Method | Static Analysis in dnSpy

We load `output_dump.bin` into dnSpy. dnSpy is a .NET decompiler/debugger that takes the raw bytecode and reconstructs readable C# source.

Inside the assembly namespace `Ac696fde6dac74a2b8d3c4bbaec8e0a74`, we find the XOR decryption function:

```csharp
namespace Ac696fde6dac74a2b8d3c4bbaec8e0a74
{
    internal class Ac696fde6dac74a2b8d3c4bbaec8e0a74
    {
        public static byte[] vdzzzjsfos(byte[] pxffgr, string qnhvkn)
        {
            byte[] bytes = Encoding.UTF8.GetBytes(qnhvkn);
            byte[] array = new byte[pxffgr.Length];
            for (int i = 0; i < pxffgr.Length; i++)
            {
                array[i] = (pxffgr[i] ^ bytes[i % bytes.Length]);
            }
            return array;
        }
    }
}
```

This is a standard repeating-key XOR. It takes a byte array (ciphertext) and a string key, XORs each byte against the key (cycling through the key if the ciphertext is longer), and returns the plaintext bytes.

Now we need to find where this function is called with actual arguments to identify the key. We look at another function `wnsdakcrsq` which handles unmanaged memory allocation for API calls, and inside its error handling we find the key in use:

```csharp
throw new NotSupportedException(
    Encoding.UTF8.GetString(
        Ac696fde6dac74a2b8d3c4bbaec8e0a74.xor_enc(
            Convert.FromBase64String("NU4RARlYWhUNRkUSREJGTFFS"),
            "a7ad965a-50b4-4846-bfb2-2282839f8d0c"
        )
    ).Replace("\\n", "\n").Replace("\\r", "\r").Replace("\\t", "\t").Replace("\\\"", "\"")
);
```

The XOR key is the second argument: `a7ad965a-50b4-4846-bfb2-2282839f8d0c`. It looks like a GUID. Hiding cryptographic keys inside what appear to be GUIDs is a nice trick it blends in visually if someone just scans for strings.

**Answer: `a7ad965a-50b4-4846-bfb2-2282839f8d0c`**

#### What Makes This Obfuscation Clever

The error message that would tell a debugger what went wrong is itself XOR-encrypted. So even if you're watching exceptions get thrown during dynamic analysis, the error strings are gibberish until you decrypt them. The attacker used obfuscation defensively, not just for payloads, but for all internal diagnostic strings. That's a sign of a well-written implant.

***

### Q9: What is the C2 IP and Port?

#### Primary Method | Payload Extraction and Cobalt Strike Config Parsing

The malware contains a function `awxrltzpes()` that orchestrates the final injection:

```csharp
private static Tuple<IntPtr, IntPtr> awxrltzpes()
{
    string systemDirectory = Environment.SystemDirectory;
    string nlegun = systemDirectory +
        Encoding.UTF8.GetString(Convert.FromBase64String("XGRsbGhvc3QuZXhl"));
    // ^ decodes to "\dllhost.exe"

    string text = Encoding.UTF8.GetString(
        Ac696fde6dac74a2b8d3c4bbaec8e0a74.xor_enc(
            Convert.FromBase64String("QRgxFlZVUBJeXFRYTw=="),
            "a7ad965a-50b4-4846-bfb2-2282839f8d0c"
        )
    ) + Af8db1ec58dc14f17b7535b23c2f5985c.tnkrvzuxf() + "}";

    IntPtr zero = IntPtr.Zero;
    Tuple<IntPtr, IntPtr> tuple = Ae555509b7d114e538171cd15b0c6bd9a.ovujqtaeoz(nlegun, text, zero);
    if (tuple == null)
    {
        tuple = Ae555509b7d114e538171cd15b0c6bd9a.ovujqtaeoz(nlegun, text);
    }
    return tuple;
}
```

This launches `dllhost.exe` from System32 and injects a payload into it. The payload is passed as a string that's XOR-decoded from a Base64 blob.

Now here's the clever part about finding the actual shellcode. While analyzing `dllhost.exe` (PID 7736) with Volatility's envars plugin, we find one of its environment variables contains a massive, obviously-not-normal string using `A$+` as a delimiter:

```
┌──(sn0x㉿sn0x)-[~/HTB/Novitas]
└─$ volatility3 -f memory.raw windows.envars.Envars
```

The `B_1`, `B_2`, etc. labeled environment variables contain the obfuscated payload. The `andxkgfxxp()` deobfuscation function from the DLL tells us exactly how to reverse it: strip `A$+`, reverse the string, add Base64 padding, decode.

```python
import base64
import re
import hashlib

def andxkgfxxp(lpxpmd: str) -> bytes:
    text = lpxpmd.replace('\n', '').replace("A$+", "")
    array = list(text)
    array.reverse()
    text2 = "".join(array)
    array2 = base64.b64decode(text2)
    return array2

def extract_encoded_strings(filename):
    encoded_dict = {}
    with open(filename, 'r', encoding='utf-8') as file:
        for line in file:
            parts = line.split()
            if len(parts) >= 5 and re.match(r'B_\d+', parts[3]):
                label = parts[3]
                encoded_string = parts[4]
                if label not in encoded_dict:
                    encoded_dict[label] = []
                encoded_dict[label].append(encoded_string)
    final = []
    for key in sorted(encoded_dict, key=lambda x: int(x.split('_')[1])):
        final.append(' '.join(encoded_dict[key]))
    return "\n".join(final)

rawcode = extract_encoded_strings('envxor.txt')
base64_decoded = andxkgfxxp(rawcode)

with open("decoded_output.bin", "wb") as f:
    f.write(base64_decoded)

print(hashlib.md5(base64_decoded).hexdigest())
```

We get a clean binary blob: `decoded_output.bin`. Now we run it through Didier Stevens' Cobalt Strike beacon config parser:

```
┌──(sn0x㉿sn0x)-[~/HTB/Novitas]
└─$ python3 1768.py decoded_output.bin
```

```
File: decoded_output.bin
xorkey b'.' 2e
0x0001 payload type          windowsbeacon_http-reverse_http
0x0002 port                  8484
0x0003 sleeptime             88535
0x0004 maxgetsize            2101637
0x0005 jitter                37
0x0007 publickey             30819f300d06092a...
0x0008 server,get-uri        '149.28.22.48,/divide/News/8HK4QT7Q,
                              0.0.0.0,/divide/News/8HK4QT7Q'
0x0043 DNS_STRATEGY          0
```

There it is. Cobalt Strike HTTP beacon, C2 at `149.28.22.48`, port `8484`, hitting URI `/divide/News/8HK4QT7Q`. Sleep time of 88535ms (\~88 seconds) with 37% jitter — designed to blend into normal web traffic and avoid beacon detection based on regular call-home intervals.

**Answer: `149.28.22.48:8484`**

#### Alternative Detection Path

If you couldn't recover the environment variable payload, you could also do network-level detection. With the C2 config extracted, defenders can now:

Block `149.28.22.48` at the firewall/proxy. Search existing proxy logs for requests to `/divide/News/8HK4QT7Q` — that specific URI pattern is a hard IOC. Look for `dllhost.exe` making outbound HTTP connections, which is abnormal behavior for that process.

***

### Q10: MD5 Hash of the Final Stage Shellcode

This one is already solved from the previous step. The decoded payload blob IS the shellcode:

```
┌──(sn0x㉿sn0x)-[~/HTB/Novitas]
└─$ md5sum decoded_output.bin
f7efce4bac431a5c703e73cce7c5f7c7 *decoded_output.bin
```

**Answer: `f7efce4bac431a5c703e73cce7c5f7c7`**

***

### Full Attack Flow

```
[Attacker Infrastructure]
        |
        | Phishing email with link to archive
        v
[Victim Downloads family_image.zip via Microsoft Edge]
        |
        | family_image.zip (1971433 bytes)
        | Contains: family_image.msc + family_image.obj
        v
[User Extracts ZIP and Opens family_image.msc]
        |
        | Windows associates .msc with mmc.exe (signed binary)
        v
[mmc.exe (PID 3120) Spawns — 2024-09-05 15:58:11 UTC]
        |
        | family_image.msc is a GrimResource weaponized snap-in
        | Triggers script/XSL execution within MMC context
        v
[Malicious .NET Assembly Loaded In-Memory]
        |
        | Assembly: Ad00bce9305554c87927205710b17699f
        | No file on disk, no PublicKeyToken, GUID-style name
        | XOR key: a7ad965a-50b4-4846-bfb2-2282839f8d0c
        v
[.NET Assembly Spawns dllhost.exe (PID 7736)]
        |
        | Injects Cobalt Strike beacon shellcode into dllhost.exe
        | Payload passed via environment variable (B_1, B_2... chunks)
        | Obfuscated: A$+ delimiters + reversed + Base64
        v
[dllhost.exe Executes Cobalt Strike Beacon]
        |
        | Protocol: HTTP reverse beacon
        | C2: 149.28.22.48:8484
        | URI: /divide/News/8HK4QT7Q
        | Jitter: 37% | Sleep: ~88s
        v
[Attacker Has Full C2 Access to Victim Machine]
```

***

### Techniques

| Technique                                    | Where Used                                                          |
| -------------------------------------------- | ------------------------------------------------------------------- |
| GrimResource (MSC weaponization)             | Initial execution via `mmc.exe` loading `family_image.msc`          |
| Living-off-the-Land Binary (LOLBin)          | `mmc.exe` and `dllhost.exe` used to blend into normal activity      |
| Memory forensics with Volatility 3           | OS identification, process tree, file scan, environment variables   |
| Browser history extraction (SQLite)          | Recovering download metadata from Edge's `History` database         |
| String carving from memory dump              | Recovering filenames with `strings \| findstr`                      |
| MemProcFS process dump                       | Extracting `mmc.exe` minidump for .NET analysis                     |
| .NET AppDomain enumeration (`!dumpdomain`)   | Identifying the anomalous in-memory .NET assembly                   |
| Raw memory extraction (`.writemem`)          | Dumping malicious DLL bytes correctly from known base address       |
| .NET decompilation (dnSpy)                   | Reverse engineering XOR encryption routine and injection code       |
| XOR string deobfuscation                     | Decrypting internal error messages and payload config               |
| Reversed Base64 + junk delimiter obfuscation | Decoding the shellcode from environment variable storage            |
| Cobalt Strike config extraction (`1768.py`)  | Parsing beacon config to recover C2 IP, port, URI, sleep/jitter     |
| Process injection (dllhost.exe)              | Shellcode execution hidden inside a legitimate system process       |
| Environment variable as shellcode transport  | Chunked B\_1/B\_2 env vars used to pass payload to injected process |

<figure><img src="../../../.gitbook/assets/image (651).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
