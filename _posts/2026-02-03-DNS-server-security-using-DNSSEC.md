---
title: Configuring DNSSEC on Windows Server
date: 2026-02-03
categories: [windows server,active directory,DNS,server hardening]
tags: [AD,WS,DNS,DNSSEC,KDS,NRPT,Cache Locking, Socket Pool,writeups,learning,notes]
toc: true
---

# Introduction

DNS is a foundational service in any Windows-based infrastructure. However, **by design it provides no authentication or integrity protection**. This makes it vulnerable to serious attacks such as:

- DNS spoofing
- Cache poisoning
- Man-in-the-middle manipulation

**DNS Security Extensions (DNSSEC)** address these weaknesses by adding **cryptographic validation** to DNS responses. This ensures clients receive authentic, untampered DNS data.

In this guide we cover:

- What DNSSEC is and how it works
- How DNSSEC is implemented on Windows Server
- Configuring DNSSEC zone signing
- Enforcing DNSSEC validation on clients using NRPT
- Additional hardening with DNS Socket Pool and Cache Locking
- Common pitfalls (especially in Active Directory environments)

## What Is DNSSEC?

DNSSEC is a set of extensions to the DNS protocol that allows **cryptographic validation** of DNS responses.

It ensures that:

- Only authorized DNS servers can provide trusted answers
- DNS data has not been altered in transit

### Key DNSSEC Resource Records

When a zone is signed, the following records are added:

- **RRSIG** – digital signatures for each RRset
- **DNSKEY** – contains the public keys used to verify signatures
- **DS** – Delegation Signer record (used in the parent zone)

### Validation Process (simplified)

1. Resolver retrieves the **DNSKEY** record
2. Hashes the received DNS response data
3. Verifies the hash against the **RRSIG** signature using the public key
4. If valid → response is trusted

**Important**: DNSSEC does **not** encrypt DNS traffic — it only guarantees authenticity and integrity.

## Why DNSSEC Matters

DNSSEC protects against:

- DNS spoofing
- Cache poisoning
- Malicious redirection to phishing / malware sites

It provides **data origin authentication** and **integrity**, forming a critical layer in modern DNS security.

## DNSSEC on Windows Server – Per-Zone Signing

Windows Server uses a **per-zone signing model**.

You can:

- Use **default signing settings** (recommended for most environments)
- Customize key rollover periods, validity intervals, key master, etc.

Zone signing does **not** change the standard DNS query/response flow — it only adds a validation layer.

### High-Level Hardening Workflow

Recommended order:

1. Enable & sign DNSSEC on the zone
2. Configure client validation policies (NRPT via Group Policy)
3. Harden the DNS server itself:
   - DNS Socket Pool
   - DNS Cache Locking

## Step 1 – Signing a DNS Zone

### Using DNS Manager (GUI)

1. Open **DNS Manager**
2. Right-click the zone → **DNSSEC** → **Sign the Zone…**
3. Choose **Customize zone signing parameters** (or use defaults)
4. Complete the wizard

### Using PowerShell (preferred for automation / repeatability)

```powershell
# Sign the zone with default settings
Invoke-DnsServerZoneSign -ZoneName "example.com" -ComputerName "dc01"

# Or with custom parameters
Set-DnsServerDnsSecZoneSetting -ZoneName "example.com" `
    -DenialOfExistenceProofType NSEC3 `
    -DSRecordGenerationAlgorithm RSASHA256 `
    -KeyMasterServer "dc01.example.com"
```

### Verifying Zone Signing

```powershell
# Check for RRSIG records
Resolve-DnsName -Name example.com -Type RRSIG

# Check DNSKEY records
Resolve-DnsName -Name example.com -Type DNSKEY | Format-Table -AutoSize

# View signing settings
Get-DnsServerDnsSecZoneSetting -ZoneName "example.com"
```

## Step 2 – Configuring DNSSEC Validation on Clients (NRPT)

Clients use the **Name Resolution Policy Table (NRPT)** to decide when to require DNSSEC validation.

### Configure via Group Policy

**Path**:

```
Computer Configuration
└ Policies
  └ Windows Settings
    └ Name Resolution Policy
```

1. Create a new rule
2. Choose **Suffix** (e.g. `example.com`) or **FQDN**
3. Enable **Require DNSSEC validation**
4. Optionally require **IPsec** or other settings
5. Apply the GPO to the desired computers / OUs

### Verify Validation on a Client

```powershell
Resolve-DnsName -Name dc01.example.com -Type All
```

Look for the line:

```
DNSSEC validation: Success
```

## Step 3 – Additional DNS Server Hardening

### DNS Socket Pool

Randomizes source ports for outbound queries → makes transaction ID prediction attacks much harder.

```powershell
# Check current size
Get-DnsServerSetting -All | Select-Object SocketPoolSize

# Set to 5000 (good balance)
dnscmd /config /socketpoolsize 5000
```

Valid range: **0 – 10000** (higher = stronger protection)

### DNS Cache Locking

Prevents cached records from being overwritten before TTL expires.

```powershell
# Check current setting
Get-DnsServerCache | Select-Object LockingPercent

# Set to full protection (recommended)
Set-DnsServerCache -LockingPercent 100
```

## Important Notes & Common Pitfalls

### KDS Root Key & Domain Controllers OU Issue

DNSSEC signing often fails when:

- Domain Controllers have been moved **out** of the default **Domain Controllers** OU
- Microsoft **Key Distribution Service (KDS)** cannot start
- `Add-KdsRootKey` fails

**Fix**:

1. Move the affected Domain Controller back to the **Domain Controllers** OU
2. Wait for Group Policy refresh
3. Create the KDS root key:
   ```powershell
   Add-KdsRootKey -EffectiveImmediately
   ```
4. Start the KDS service if needed
5. Retry zone signing

### Multi-DC / Multi-Site Environments

Make sure:

- All DCs are online during initial signing
- AD replication is healthy (`repadmin /replsummary`)
- DNS zones are **AD-integrated**
- DNSSEC metadata replicates via AD (not per-DC)

## Conclusion

DNSSEC significantly strengthens DNS security in Windows Server environments by cryptographically validating responses and preventing tampering.

A complete modern DNS hardening strategy usually includes:

- DNSSEC zone signing
- NRPT-enforced client validation
- DNS Socket Pool (randomized ports)
- DNS Cache Locking (100%)

Pay special attention to:

- Correct placement of Domain Controllers
- Healthy Active Directory replication
- Availability of the Key Distribution Service (KDS)

When these pieces are correctly configured, you achieve strong **defense-in-depth** for one of the most critical services in your infrastructure — DNS.
```

This format should work well in most Jekyll blogs. You can adjust the `date`, `categories`, `tags`, or layout name according to your site's conventions.

Let me know if you'd like to add images (diagrams), code highlighting adjustments, or any other refinements!

Thanks for reading – more technical blogs around Windows Server and Active Directory coming soon. 

**Inshallah** this becomes a long series. The more I break, the more I’ll share.

Stick around if you want to learn together.

```bash{.dua-tawfik}
رَبِّ زِدْنِي عِلْمًا ۞ وَفَهْمًا وَحِكْمَةً
رَبِّ يَسِّرْ وَلَا تُعَسِّرْ ۞ وَتَمِّمْ بِالْخَيْرِ
```

#### ~ Darknight