---
title: "Mentoring Four Microsoft Security Engineering Projects"
date: 2026-07-10
categories: [Microsoft Security, Entra ID, Defender XDR, Internship]
tags: [Microsoft Graph, Entra ID, Defender XDR, PowerShell, OAuth, Conditional Access, KQL, Identity Security]
---

# Mentoring Four Microsoft Security Engineering Projects

As part of a Microsoft Security internship program, I am mentoring four interns working on focused security engineering projects around **Microsoft Entra ID, Microsoft Graph, Conditional Access, and Defender XDR**.

The goal was deliberately not to build generic Graph scripts. Each project addresses a real consulting problem where Microsoft already exposes most of the required data, but that data remains fragmented across portals, APIs, permissions, relationships, and logs. The objective is to transform that raw information into **explainable security intelligence, prioritized findings, and consultant-ready reporting**. :contentReference[oaicite:0]{index=0}

The projects cover four complementary areas:

```text
Application Identity Exposure
        ↓
Privilege Escalation Paths
        ↓
Conditional Access Exposure
        ↓
Detection & Incident Investigation
````


## 1. OAuth & Application Identity Exposure Analyzer

### Problem

Microsoft 365 tenants continuously accumulate:

* App registrations
* Enterprise applications
* Service principals
* OAuth consent grants
* Application permissions
* Secrets and certificates
* Third-party integrations

The difficult question is not *what applications exist*, but:

> **Which application identities represent the greatest security exposure, and why?**

### Solution

The intern developed a PowerShell/Microsoft Graph assessment engine that correlates:

```text
Service Principal
├── Application + Delegated Permissions
├── OAuth Consent
├── Ownership
├── Credential Lifecycle
├── Sign-in Activity
├── Publisher / Provenance
├── Audit Events
└── Historical Configuration Drift
```

It then produces risk-ranked HTML, JSON, and CSV assessment artifacts.

A major part of my mentoring work on this project was challenging the **security semantics behind the findings**, rather than only validating that the code executed successfully.

Early versions, for example, confused:

```text
No evidence available
```

with:

```text
Evidence confirms absence
```

This caused Microsoft or external service principals to be incorrectly penalized for credentials that the tenant could not inspect, and sign-in gaps to be interpreted too aggressively as inactivity.

We progressively introduced:

* local vs third-party vs Microsoft first-party classification;
* explicit evidence states;
* improved permission-risk taxonomy;
* ownership enrichment;
* credential lifecycle analysis;
* activity correlation;
* audit-log correlation;
* historical drift;
* business-impact explanations.

The latest tenant-wide assessment successfully processes hundreds of service principals and produces materially more useful prioritization than the initial prototype.

The remaining engineering work is mainly **correctness hardening**: completing the resource-aware permission catalog, refining first-party activity semantics, and ensuring that historical baselines and recommendation logic remain technically defensible.

---

## 2. Entra Application Ownership Privilege Path Analyzer

### Problem

Traditional Entra administration focuses heavily on direct directory roles:

```text
User → Global Administrator
```

But privilege can also exist indirectly.

A normal user might own an application whose service principal already holds powerful application permissions:

```text
Standard User
      ↓ OWNS
Application
      ↓
Service Principal
      ↓
RoleManagement.ReadWrite.Directory
```

The user may not appear privileged in a normal role review while still controlling a privileged workload identity.

### Solution

This project builds a focused graph of relationships between:

* Users
* Groups
* Applications
* Service principals
* Owners
* Directory roles
* Microsoft Graph permissions

The analyzer discovers ownership-based paths and explains how control over an identity can translate into effective privilege.

One important design correction was preventing the project from becoming a simplified **BloodHound clone**. Its value is deliberately narrower: understandable, application-centric Entra privilege paths that can be included directly in a consulting assessment. 

During review, another important distinction emerged:

```text
Attack Path ≠ Finding
```

Multiple graph paths can support the same underlying security exposure.

Similarly:

```text
Directory.Read.All
```

is sensitive reconnaissance, but it is not equivalent to:

```text
RoleManagement.ReadWrite.Directory
```

which can represent actual privilege escalation capability.

The next maturity step is therefore a stronger finding model separating:

* Verified privilege escalation
* Privileged workload ownership
* Identity/application control
* Sensitive reconnaissance
* Potential privilege-enabling capability

and validating the engine against a deliberately created **non-admin → privileged application** test path.

---

## 3. Conditional Access Effective Exposure Analyzer

### Problem

A tenant may contain many apparently strong Conditional Access policies:

```text
Require MFA
Require compliant devices
Block legacy authentication
Protect administrators
Block risky authentication
```

Yet this does not answer the most useful consulting question:

> **Who is actually protected?**

Policy exclusions, nested groups, role targeting, report-only configurations, exceptions, and overlapping policies can leave important identities exposed despite the presence of apparently strong policies.

### Solution

The project evaluates Conditional Access from the **identity's perspective** instead of simply exporting policy configuration.

The intended model is:

```text
Identity
   ↓
Applicable CA Policies
   ↓
Assignments / Exclusions
   ↓
Grant Controls
   ↓
Effective Protection
   ↓
Exposure
```

The analyzer combines:

* Conditional Access policies;
* users and groups;
* directory roles;
* inclusions and exclusions;
* policy state;
* control objectives;
* sign-in evidence where appropriate.

The target output is something closer to:

```text
Privileged identities assessed: 42
Protected by enforced MFA: 39
Excluded from all applicable MFA controls: 3
Recently observed signing in: 2

Result: HIGH EXPOSURE
```

rather than:

```text
Policy X excludes Group Y
```

The main architectural requirement for this project is to ensure it remains an **effective exposure analyzer** and does not regress into another Conditional Access inventory/export tool.

---

## 4. Defender XDR Incident Case Brief Generator

### Problem

Microsoft Defender XDR already provides rich incident telemetry, alerts, entities, and Advanced Hunting data.

The operational problem is converting all of that into a repeatable investigation package.

Junior analysts can otherwise spend significant time manually reconstructing:

```text
What happened?
Which identities/devices were affected?
What evidence supports the conclusion?
What happened first?
What should be contained?
```

### Solution

The intern built a modular investigation engine that retrieves Defender incidents and alerts, extracts entities, classifies investigation scenarios, and launches evidence-driven KQL pivots.

The architecture currently separates:

```text
Authentication
      ↓
Incident Collection
      ↓
Evidence Extraction
      ↓
Investigation Classification
      ↓
KQL / Advanced Hunting
      ↓
Analysis
```

Supported investigation logic already covers areas such as malware, suspicious execution, identity compromise, OAuth abuse, ransomware, persistence, credential theft, and defense evasion.

This was technically good progress—but it also introduced the most obvious **scope drift** of the four projects.

The original objective was:

> **Defender XDR Incident Case Brief Generator**

while development increasingly moved toward:

> **General Defender Investigation Assistant**

The investigation engine is valuable, but it should remain supporting infrastructure.

The final product must still turn the evidence into:

```text
Executive Summary
Incident Metadata
Affected Entities
Chronological Timeline
Evidence Table
Analyst Interpretation
MITRE ATT&CK Mapping
Containment Actions
Eradication / Recovery Actions
KQL Appendix
Open Questions
```

The key remaining milestone is therefore a full controlled:

```text
Defender Incident
     ↓
Alert / Evidence Collection
     ↓
Advanced Hunting
     ↓
Evidence Normalization
     ↓
Timeline Reconstruction
     ↓
Final Case Brief
```

validation.

---

# Shared Security Engineering Approach

The four projects deliberately share several engineering principles.

### Microsoft Graph and least privilege

Each collector should request only the permissions necessary for its assessment function. The privilege of the **assessment application itself** must remain separate from the privilege being analyzed.

### PowerShell-first implementation

The tools are primarily PowerShell-based and follow common conventions around authentication, Graph connectivity, output directories, modular functions, and repeatable execution.

### OpenBao

OpenBao was introduced later as shared internship infrastructure for **secrets and PKI management**.

Rather than leaving client secrets or certificates on intern workstations, project identities can retrieve controlled credentials from a centralized secrets platform.

Its role is intentionally infrastructural:

```text
Intern
   ↓
OpenBao
   ↓
Project Credential
   ↓
Microsoft Graph / Defender XDR
   ↓
Assessment Tool
```

It should support the projects—not redefine them.

### Evidence before conclusions

One of the most important design principles across all four projects has become:

> **Never turn missing evidence into a definitive security conclusion.**

`Unknown`, `NotCollected`, `NotAuthorized`, `NotApplicable`, and `ConfirmedAbsent` are materially different states in a security assessment.

This became especially important when analyzing service-principal activity, credentials, ownership, and Defender evidence.

---

# What I Am Focusing on as Mentor

My role has gone beyond assigning project topics or reviewing whether Graph calls work.

A significant part of the mentoring process is reviewing whether the **security conclusions themselves are defensible**.

That includes challenging questions such as:

```text
Does this permission really enable privilege escalation?

Is this one finding or multiple supporting paths?

Can the tenant actually remediate this object?

Does "no sign-in observed" really mean "unused"?

Is this Microsoft-managed identity being incorrectly scored?

Does the recommendation match the actual OAuth consent model?

Can the assessment reproduce the evidence behind this conclusion?
```

This distinction is important.

A security tool can be technically functional while still producing misleading security conclusions.

The goal of the internship is therefore not simply to produce four working repositories, but to teach the interns how to move from:

```text
API Data
```

to:

```text
Defensible Security Finding
      ↓
Business Impact
      ↓
Evidence
      ↓
Actionable Remediation
```

---

# Current Status

All four projects remain works in progress, but they now form a coherent Microsoft Security engineering portfolio:

| Project                                        | Current Focus                                                              |
| ---------------------------------------------- | -------------------------------------------------------------------------- |
| OAuth & Application Identity Exposure Analyzer | Final correctness and risk-model hardening                                 |
| Entra Ownership Privilege Path Analyzer        | Finding semantics and verified non-admin escalation testing                |
| Conditional Access Effective Exposure Analyzer | Effective identity-level coverage evaluation                               |
| Defender XDR Incident Case Brief Generator     | Returning focus to timeline reconstruction and final case-brief generation |

Together, the projects cover **application identity governance, privilege escalation, preventive access controls, and security operations**.

More importantly, they provide the interns with practical exposure to Microsoft Graph, Entra identity models, OAuth, service principals, Conditional Access, Defender XDR, KQL, PowerShell, PKI, secrets management, testing, evidence handling, and security assessment methodology.

The solutions are intentionally evolving through repeated review, failed assumptions, controlled testing, and architectural corrections—the same process required when moving security tooling from a working prototype toward something that can eventually support a real consulting engagement.

