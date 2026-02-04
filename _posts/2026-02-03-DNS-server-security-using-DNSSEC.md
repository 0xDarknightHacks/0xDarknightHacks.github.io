---
title: Configuring DNSSEC on Windows Server
date: 2026-02-03
categories: [windows server,active directory,DNS,server hardening]
tags: [AD,WS,DNS,DNSSEC,KDS,NRPT,Cache Locking, Socket Pool,writeups,learning,notes]
toc: true
---

# Introduction

DNS is a foundational service in any Windows-based infrastructure, but by design it does not provide authentication or integrity protection. This makes it vulnerable to attacks such as DNS spoofing, cache poisoning, and man-in-the-middle manipulation.

DNS Security Extensions (DNSSEC) address these weaknesses by allowing DNS responses to be cryptographically validated, ensuring that clients receive authentic and untampered DNS data.

In this guide, we will walk through:

- What DNSSEC is and how it works
- How DNSSEC is implemented on Windows Server
- How to validate DNSSEC on clients
- How to harden DNS further using DNS Socket Pool and DNS Cache Locking
- Common pitfalls in Active Directory environments (including KDS-related issues)

## What Is DNSSEC?

DNSSEC is a security standard that protects DNS by ensuring that only authorized DNS servers can respond to DNS queries with data that clients trust.

It is implemented as a collection of extensions to the DNS protocol and is commonly deployed alongside other DNS hardening mechanisms such as:

- DNS Cache Locking
- DNS Socket Pool

When DNSSEC is enabled, the DNS zone is extended with additional resource records, including:

- **RRSIG records**  
  These contain digital signatures for DNS data.

- **DNSKEY records**  
  These contain the public keys used to validate DNS responses.

When a DNS resolver receives a response from a DNS server, it:

1. Retrieves the DNSKEY record
2. Hashes the DNS response
3. Verifies the hash against the RRSIG signature

If validation succeeds, the response is considered authentic and untampered.

## Why DNSSEC Matters

By using cryptographic signatures, DNSSEC ensures:

- Integrity of DNS data
- Authenticity of the responding DNS server
- Protection against:
  - DNS spoofing
  - Cache poisoning
  - Tampered responses that redirect users to malicious resources

DNSSEC does not encrypt DNS traffic, but it guarantees that the data received is trustworthy.

## DNSSEC Zone Signing on Windows Server

### Per-Zone Signing Model

DNSSEC signing on Windows Server is performed per DNS zone.

You can:

- Use the default signing configuration (recommended for most environments)
- Customize advanced parameters, such as:
  - Key master
  - Key rollover intervals
  - DNSKEY and RRSIG validity periods

In most enterprise environments, the default signing parameters are sufficient and align with Microsoft best practices.

Signing a zone does not alter the standard DNS query/response mechanism. Instead, it adds a cryptographic validation layer that operates transparently to clients.

## High-Level Hardening Workflow

To properly secure a Windows DNS server, the recommended sequence is:

1. Configure DNSSEC on the DNS zone
2. Configure DNSSEC validation policies via Group Policy (NRPT)
3. Configure DNS Socket Pool
4. Configure DNS Cache Locking

## Signing a DNS Zone

Zone signing can be done using:

- The DNS Manager graphical wizard
- PowerShell

## Verifying Zone Signing

To confirm that a zone is signed, query the DNSSEC records.

**Query RRSIG records**

```powershell
Resolve-DnsName -Name domain.name -Type RRSIG
```

**Query DNSKEY records**

```powershell
Resolve-DnsName -Name domain.name -Type DNSKEY | Format-Table -AutoSize
```

**Retrieve DNSSEC signing parameters**

```powershell
Get-DnsServerDnsSecZoneSetting -ZoneName domain.name -ComputerName computer.name
```

**Modify DNSSEC signing parameters**

```powershell
Set-DnsServerDnsSecZoneSetting
```

## Configuring DNSSEC Validation on Clients (NRPT)

Before sending a DNS query, a Windows client checks the Name Resolution Policy Table (NRPT) to determine whether DNSSEC validation rules apply.

If a match is found, the client enforces the configured policy, such as requiring authenticated DNS responses.

### Configuring NRPT via Group Policy

NRPT settings are configured under:

```
Computer Configuration
└ Policies
  └ Windows Settings
    └ Name Resolution Policy
```

Within the Create Rules section, DNSSEC rules can target:

- Prefix – applies to subdomains
- Suffix – applies to the domain itself
- FQDN – applies to a specific host

Scope – rules can apply to:

- Specific IP subnets
- All computers affected by the GPO

### Verifying DNSSEC Validation on Clients

After Group Policy is applied, verify DNSSEC validation with:

```powershell
Resolve-DnsName -Name computer.domain.name -Type All
```

Authenticated responses indicate successful DNSSEC validation.

## Additional DNS Hardening

Once DNSSEC and NRPT are in place, the DNS server itself should be further hardened.

### DNS Socket Pool

DNS Socket Pool increases security by randomizing the source ports used for outbound DNS queries, making transaction ID prediction attacks significantly harder.

**Check current socket pool size**

```powershell
Get-DnsServerSetting -All | Select-Object -Property SocketPoolSize
```

**Increase socket pool size**

```powershell
dnscmd /config /socketpoolsize 5000
```

Valid values range from 0 to 10000. Larger values provide stronger protection.

### DNS Cache Locking

DNS Cache Locking prevents cached DNS records from being overwritten before their TTL expires, protecting against cache poisoning attacks.

**Check cache locking percentage**

```powershell
Get-DnsServerCache | Select-Object -Property LockingPercent
```

The recommended value is: **100**

**Enable full cache locking**

```powershell
Set-DnsServerCache -LockingPercent 100
```

## Important Notes & Common Pitfalls

### Domain Controllers Moved from Default OU

In my own setup, DNSSEC zone signing initially failed due to an unexpected dependency:

- Domain Controllers had been moved out of the default “Domain Controllers” OU
- The Microsoft Key Distribution Service (KDS) failed to start
- Creating a KDS Root Key also failed

The root cause was that the DC was no longer receiving critical DC-specific policies.

**Resolution Steps**

1. Move the Domain Controller back to the Default Domain Controllers OU
2. Allow Group Policy to apply
3. Create the KDS root key
4. Start the KDS service
5. Sign the DNS zone successfully

### Multi-DC / Multi-Site Environments

If you are running:

- Redundant Domain Controllers
- Multi-site Active Directory with replication

Ensure that:

- All DCs are online during zone signing
- Active Directory replication is healthy
- DNS zones are AD-integrated

DNSSEC keys and signing metadata are replicated through Active Directory and should never be created per DC.

## Conclusion

DNSSEC significantly strengthens DNS security in Windows Server environments by ensuring the authenticity and integrity of DNS data. When combined with:

- Proper Group Policy enforcement
- DNS Socket Pool
- DNS Cache Locking

…it forms a robust, defense-in-depth approach to DNS hardening.

Correct placement of Domain Controllers, healthy AD replication, and awareness of dependencies like KDS are critical for a successful deployment.


Thanks for reading – more technical blogs around Windows Server and Active Directory coming soon. 

**Inshallah** this becomes a long series. The more I break, the more I’ll share.

Stick around if you want to learn together.

```bash{.dua-tawfik}
رَبِّ زِدْنِي عِلْمًا ۞ وَفَهْمًا وَحِكْمَةً
رَبِّ يَسِّرْ وَلَا تُعَسِّرْ ۞ وَتَمِّمْ بِالْخَيْرِ
```

#### ~ Darknight