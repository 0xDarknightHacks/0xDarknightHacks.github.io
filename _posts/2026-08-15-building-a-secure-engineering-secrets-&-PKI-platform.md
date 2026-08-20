---
title: "Building a Secure Engineering, Secrets & PKI Platform with OpenBao, Microsoft Entra ID, and Proxmox"
date: 2026-08-20 15:45:00 +0100
categories: [Security Engineering, Homelab]
tags: [openbao, microsoft-entra-id, microsoft-graph, pki, x509, oidc, proxmox, docker, powershell, secrets-management, zero-trust]
description: "How I designed a private security-engineering platform that combines Proxmox, Microsoft Entra ID, OpenBao, PKI, project-scoped RBAC, and certificate-based Microsoft Graph authentication."
toc: true
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Building a Secure Engineering, Secrets & PKI Platform with OpenBao, Microsoft Entra ID, and Proxmox

I spend a significant part of my work around **Microsoft 365 security, Microsoft Entra ID, Microsoft Graph, PowerShell automation, and security assessments**.

As the number of internal security-engineering projects I was working on and supervising increased, one infrastructure problem became increasingly obvious:

> How do I give different security projects the credentials and certificates they need without distributing privileged secrets across developer machines, configuration files, repositories, and scripts?

The answer eventually became a small but fairly complete **private security-engineering platform** built around:

* Proxmox VE;
* Ubuntu Linux;
* Docker;
* internal DNS and TLS;
* Microsoft Entra ID;
* OpenBao;
* X.509 PKI;
* project-scoped authorization;
* Microsoft Graph app-only authentication.

The goal was not simply to deploy OpenBao.

The goal was to build a security boundary around the development environment itself.

---

# Project at a Glance

| Area                        | Implementation                                                                                   |
| --------------------------- | ------------------------------------------------------------------------------------------------ |
| **Objective**               | Centralize secrets and certificate issuance for internal Microsoft Security engineering projects |
| **Infrastructure**          | Proxmox VE, Ubuntu Server, Docker Compose                                                        |
| **Human authentication**    | Microsoft Entra ID via OIDC                                                                      |
| **Authorization**           | OpenBao Identity + project-scoped templated policies                                             |
| **Secrets**                 | OpenBao KV v2                                                                                    |
| **PKI**                     | Dedicated Root CA + operational Intermediate CA                                                  |
| **Workload authentication** | Client secret and X.509 certificate                                                              |
| **Microsoft integration**   | Entra App Registrations + Microsoft Graph                                                        |
| **Private-key model**       | Prefer locally generated, non-exportable keys + CSR signing                                      |
| **Auditing**                | OpenBao file audit logging                                                                       |
| **Storage**                 | OpenBao Integrated Storage / Raft                                                                |
| **Recovery model**          | Shamir unseal + Raft snapshots                                                                   |
| **Security principles**     | Least privilege, separation of duties, project isolation, credential lifecycle management        |

The initial OpenBao integration was intentionally scoped to the projects that actually required centralized Microsoft Graph credentials and PKI rather than forcing every project to depend on OpenBao.

---

# The Problem

A security tool querying Microsoft Graph often needs an Entra application identity.

That identity may require:

```text
Tenant ID
Application / Client ID
Client Secret
or
Certificate + Private Key
```

The simplest development pattern is also one of the worst:

```text
Developer workstation
├── .env
├── config.json
├── certificate.pfx
├── password.txt
└── PowerShell variables
```

It works.

It also creates several problems.

## Credential sprawl

Secrets start existing in multiple places:

* local files;
* PowerShell scripts;
* shell history;
* repositories;
* exported PFX files;
* chat messages;
* temporary configuration files.

## Weak isolation

If multiple projects share credentials, compromising one project can expose another.

## Poor attribution

When several tools authenticate with the same application identity, it becomes much harder to determine which project performed a particular operation.

## Certificate lifecycle problems

Self-signed certificates generated independently on developer machines quickly become difficult to manage.

Who issued them?

When do they expire?

Where is the private key?

Can the private key be exported?

Was the certificate ever replaced?

## Excessive administrative access

A common shortcut in labs is to let everyone use the administrative credential because implementing proper authorization takes longer.

That was explicitly something I wanted to avoid.

The target model became:

```text
Human identity
      ↓
Authenticated centrally
      ↓
Receives only project-specific access
      ↓
Uses only its project's workload credentials
      ↓
Workload receives only required Microsoft Graph permissions
```

---

# Design Goals

Before configuring OpenBao, I defined several design principles.

## 1. Human identity must be separate from workload identity

An engineer authenticating to the secrets platform is not the same identity as the application authenticating to Microsoft Graph.

```text
Human
  ↓
Microsoft Entra ID
  ↓
OIDC
  ↓
OpenBao
```

is separate from:

```text
Security Tool
  ↓
Entra App Registration
  ↓
Certificate / Client Secret
  ↓
Microsoft Graph
```

This separation sounds obvious, but collapsing those two identity planes is an easy mistake to make in small environments.

---

## 2. Projects must not share application identities by default

Each project should have its own:

```text
Entra App Registration
OpenBao secret path
PKI authorization role
Certificate
Private key
Microsoft Graph permissions
```

This provides a much cleaner security boundary.

It also improves:

* auditing;
* revocation;
* credential rotation;
* troubleshooting;
* permission reviews;
* accountability.

---

## 3. Authentication must not imply authorization

Microsoft Entra ID answers:

> Who is this person?

OpenBao answers:

> What is this person allowed to retrieve or issue?

Those are intentionally separate decisions.

---

## 4. Protecting the credential does not reduce its Graph privilege

This became one of the most important architectural principles.

OpenBao controls:

> Who can access the credential?

Microsoft Entra ID controls:

> What can the application do with that credential?

The complete authorization model therefore looks like this:

```text
Engineer
   │
   ▼
Microsoft Entra ID
   │
   │ OIDC
   ▼
OpenBao
   │
   │ OpenBao Policy
   ▼
Project Credential
   │
   ▼
Entra App Registration
   │
   │ Application Permissions
   ▼
Microsoft Graph
```

A perfectly protected certificate attached to an excessively privileged application is still an excessively privileged application.

Secret management and cloud authorization are separate security problems.

---

# High-Level Architecture

The platform sits inside my private lab infrastructure.

```text
                         Microsoft Entra ID
                               │
                               │ OIDC
                               ▼
                       ┌─────────────────┐
                       │     OpenBao     │
                       ├─────────────────┤
                       │ Identity        │
                       │ Policies        │
                       │ Audit           │
                       ├─────────────────┤
                       │ KV v2           │
                       │ PKI Root        │
                       │ PKI Intermediate│
                       └────────┬────────┘
                                │
              ┌─────────────────┴─────────────────┐
              │                                   │
              ▼                                   ▼
       Project A Credential                Project B Credential
              │                                   │
              ▼                                   ▼
       Entra App Registration              Entra App Registration
              │                                   │
              └──────────────┬────────────────────┘
                             │
                             ▼
                     Microsoft Graph
```

Underneath that security layer:

```text
Proxmox VE
    │
    ▼
Ubuntu Server
    │
    ├── Docker / Docker Compose
    ├── OpenBao
    ├── Internal DNS
    ├── Reverse proxy services
    ├── Private source control
    └── Secure remote connectivity
```

The infrastructure is intentionally small, but the security boundaries are designed to resemble patterns used in larger environments.

---

# Infrastructure Layer

The underlying platform runs on **Proxmox VE**.

A dedicated Ubuntu Server virtual machine hosts containerized infrastructure services through Docker Compose.

The wider environment includes:

| Component          | Purpose                                       |
| ------------------ | --------------------------------------------- |
| **Proxmox VE**     | Virtualization                                |
| **Ubuntu Server**  | Container host                                |
| **Docker Compose** | Service deployment                            |
| **Pi-hole**        | Internal DNS                                  |
| **Nginx**          | Reverse proxying                              |
| **Forgejo**        | Private source control                        |
| **Tailscale**      | Private remote connectivity                   |
| **OpenBao**        | Secrets, identity-aware authorization and PKI |

An internal `home.arpa` DNS namespace is used for lab services, while internally issued certificates provide TLS trust between managed systems.

The intention was to avoid using raw IP addresses throughout the environment and instead operate services through predictable DNS identities.

---

# Deploying OpenBao

OpenBao became the central security component of the platform.

The implementation uses:

```text
OpenBao 2.6.x
Integrated Storage / Raft
TLS
Shamir unseal
File audit logging
OpenBao UI
Microsoft Entra ID OIDC
```

OpenBao's data is persisted through its integrated Raft storage backend.

Conceptually:

```hcl
storage "raft" {
  path    = "/openbao/data"
  node_id = "openbao-01"
}
```

The real deployment is containerized with persistent storage mounted from the Ubuntu host.

I selected Raft because it keeps the OpenBao data layer self-contained rather than adding another external database dependency to a relatively small environment.

---

# Seal and Recovery Model

The initial deployment uses **Shamir key splitting**.

```text
Key shares: 5
Threshold:  3
```

This means three valid shares are required to unseal the platform.

The important part was not simply enabling Shamir.

It was understanding what the recovery material represents.

The following should not be stored together casually:

```text
OpenBao data
+
unseal shares
+
initial root token
+
TLS private keys
+
backup material
```

Doing so would largely defeat the security boundary.

The initial root token is therefore treated as bootstrap/emergency material rather than a routine login mechanism.

Day-to-day human access happens through Microsoft Entra ID.

---

# Microsoft Entra ID as the Human Identity Provider

I configured OpenBao's OIDC authentication method against Microsoft Entra ID.

The resulting authentication path is:

```text
Engineer
   │
   ▼
Microsoft Entra ID
   │
   │ MFA / Conditional Access capable identity plane
   ▼
OIDC Token
   │
   ▼
OpenBao Identity
```

This provides several advantages over locally managed OpenBao accounts or routine root-token usage:

* centralized identity lifecycle;
* MFA support;
* Conditional Access support;
* account disablement from Entra;
* group-based assignment;
* better attribution.

---

# An OIDC Problem That Actually Mattered

One of the useful implementation problems appeared immediately after configuring OIDC.

OpenBao returned:

```text
failed to fetch groups:
"groups" claim not found in token
```

The OpenBao OIDC role expected a claim similar to:

```text
groups
```

but Microsoft Entra ID was not emitting the required group membership information.

The fix was not an OpenBao policy change.

The token itself needed to contain the expected claim.

In Entra, I configured the application token settings to emit group claims and restricted them to groups relevant to the application instead of blindly returning all of the user's memberships.

The flow became:

```text
Entra User
   │
   ▼
Assigned Entra Group
   │
   ▼
OIDC groups claim
   │
   ▼
OpenBao identity mapping
   │
   ▼
OpenBao policy
```

This reinforced an important identity lesson:

> Authorization logic can only be as reliable as the claims entering the authorization system.

---

# Authentication vs Authorization

OIDC authenticates the user.

It does **not** directly give that user permission to secrets.

Once the user is authenticated, OpenBao Identity maps the authenticated identity into internal entities and groups.

Policies then determine what the identity can access.

That distinction makes the design significantly cleaner:

```text
Entra ID
   │
   └── proves identity

OpenBao Identity
   │
   └── maps identity context

OpenBao Policy
   │
   └── authorizes resource access
```

Administrators and engineers therefore do not share the same authorization model.

---

# Project Isolation with Templated Policies

I wanted project isolation without creating an entirely separate copy of every policy.

OpenBao's identity-aware policy templating solved this cleanly.

Each relevant entity contains project metadata.

For example:

```text
Project A:
project = ca

Project B:
project = defender
```

A shared policy can then reference:

```hcl
{{identity.entity.metadata.project}}
```

A simplified KV v2 policy looks like this:

```hcl
path "intern-secrets/data/{{identity.entity.metadata.project}}/*" {
  capabilities = ["create", "read", "update"]
}

path "intern-secrets/metadata/{{identity.entity.metadata.project}}/*" {
  capabilities = ["read", "list"]
}
```

The same policy therefore becomes different effective access depending on the authenticated entity.

For the `ca` project:

```text
intern-secrets/ca/*
```

For the `defender` project:

```text
intern-secrets/defender/*
```

This creates horizontal isolation:

```text
CA project      X      Defender secrets
Defender project X     CA secrets
```

without duplicating the policy architecture.

---

# Vertical Separation

Horizontal project isolation is only half the model.

The platform also separates users from infrastructure administration.

A normal project identity does **not** receive access to:

```text
root policy
sudo capabilities
sys/auth/*
sys/policies/*
sys/mounts/*
pki-root/*
unseal keys
initial root token
```

The model is effectively:

```text
Platform Administrator
├── Authentication configuration
├── Identity mapping
├── Policies
├── Secrets engines
├── PKI issuers
├── Audit
└── Recovery

Project Engineer
├── Own project secrets
├── Own project certificate role
└── Normal self-service operations
```

This is both **horizontal separation** and **vertical separation**.

---

# KV v2 Secrets Architecture

A dedicated KV v2 secrets engine stores application credentials.

The logical structure looks like:

```text
intern-secrets/
├── ca/
│   └── graph/
│       ├── client-secret
│       └── certificate
│
└── defender/
    └── graph/
        ├── client-secret
        └── certificate
```

A client-secret record can contain:

```text
tenant_id
client_id
client_secret
expires_on
```

Certificate metadata can contain:

```text
tenant_id
client_id
thumbprint
subject
serial_number
expires_on
```

The important change is architectural.

Instead of:

```powershell
$ClientSecret = "something-sensitive"
```

inside a script or configuration file, the intended model becomes:

```text
Authenticate to OpenBao
        ↓
Authorize project path
        ↓
Retrieve required credential
        ↓
Use credential in process memory
        ↓
Authenticate workload
```

The repository does not need to contain the credential.

---

# Why I Did Not Treat OpenBao as a Password Database

Centralization is useful, but simply moving every secret into OpenBao does not automatically make an architecture good.

The more important questions are:

```text
Who can retrieve it?
Why can they retrieve it?
For which project?
For how long?
What can the credential do after retrieval?
Can its use be audited?
Can it be rotated?
```

That is why the OpenBao implementation combines:

```text
Identity
+
Policy
+
KV
+
PKI
+
Audit
```

rather than using KV alone.

---

# PKI Architecture

The second major OpenBao function is internal PKI.

Instead of allowing every project to independently create and manage arbitrary self-signed authentication certificates, I created a simple two-tier CA hierarchy.

```text
Graph Authentication Root CA
             │
             │ signs
             ▼
Graph Authentication Intermediate CA
             │
             │ issues
             ▼
Application Authentication Certificates
```

The corresponding OpenBao PKI mounts are conceptually:

```text
pki-root/
pki-graph/
```

The Root CA exists primarily as the trust anchor.

Routine application certificate issuance happens through the Intermediate CA.

---

# Why Separate Root and Intermediate CAs?

Technically, the Root CA could issue every application certificate.

I intentionally avoided that model.

The better hierarchy is:

```text
Root CA
   │
   └── signs Intermediate

Intermediate CA
   │
   ├── signs Project Certificate A
   ├── signs Project Certificate B
   └── signs future workload certificates
```

This limits routine exposure of the root signing capability.

The Root CA is therefore an administrative resource, not a self-service resource.

---

# PKI Roles as an Authorization Boundary

Having an Intermediate CA is not enough.

If every project can request any certificate identity it wants, the PKI still has a weak authorization model.

I therefore created project-specific certificate roles.

Conceptually:

```text
graph-ca
graph-defender
```

The roles restrict issuance characteristics such as:

* allowed certificate identity;
* key algorithm;
* key size;
* validity;
* key usage;
* certificate usage;
* wildcard behavior.

Example project identities are structured similarly to:

```text
ca.graph-auth.home.arpa
defender.graph-auth.home.arpa
```

The policy can then use the same project metadata pattern:

```hcl
path "pki-graph/sign/graph-{{identity.entity.metadata.project}}" {
  capabilities = ["create", "update"]
}

path "pki-graph/issue/graph-{{identity.entity.metadata.project}}" {
  capabilities = ["create", "update"]
}
```

This means the same shared policy dynamically resolves to the correct certificate role.

```text
project=ca
    ↓
graph-ca

project=defender
    ↓
graph-defender
```

A user from one project cannot simply use the other project's role.

---

# Two Certificate Issuance Models

OpenBao supports two relevant workflows.

## Model 1 — OpenBao Generates the Key

```text
Engineer
   │
   ▼
OpenBao PKI issue endpoint
   │
   ├── generates private key
   └── generates certificate
```

This is operationally convenient.

But the private key is returned during issuance.

For long-lived workload authentication, that was not my preferred security model.

---

# Model 2 — Local Key + CSR Signing

The preferred workflow is:

```text
Windows Host
   │
   ├── Generate non-exportable private key
   │
   └── Generate PKCS#10 CSR
   │
   ▼
OpenBao
   │
   └── Sign CSR through authorized PKI role
   │
   ▼
Signed Certificate
   │
   ▼
Windows Certificate Store
```

The critical property is:

> The private key never has to leave the endpoint that uses it.

OpenBao acts as the certificate issuer rather than the private-key repository.

That difference matters.

---

# End-to-End Certificate Authentication Flow

The resulting Microsoft Graph certificate workflow is:

```text
1. Generate private key locally
          ↓
2. Generate CSR
          ↓
3. Submit CSR to OpenBao
          ↓
4. OpenBao validates assigned PKI role
          ↓
5. Intermediate CA signs certificate
          ↓
6. Install certificate chain
          ↓
7. Bind signed leaf certificate to local private key
          ↓
8. Upload PUBLIC leaf certificate to Entra App Registration
          ↓
9. Keep private key local
          ↓
10. Authenticate application to Microsoft Graph
```

Only the public certificate is registered with Microsoft Entra ID.

The private key remains on the machine performing the workload authentication.

---

# Windows Certificate Store

For the certificate-based workflow, the certificate chain can be installed into the normal Windows certificate stores.

Conceptually:

```text
Trusted Root Certification Authorities
└── Graph Authentication Root CA

Intermediate Certification Authorities
└── Graph Authentication Intermediate CA

Personal / My
└── Application Authentication Certificate
       └── Private key
```

The application authentication private key can be created as non-exportable.

This provides a much stronger model than distributing `.pfx` files between machines.

---

# Microsoft Graph App-Only Authentication

Once the public certificate is registered on the Entra application, Microsoft Graph PowerShell can authenticate using the application identity.

A simplified example looks like:

```powershell
$TenantId  = "<tenant-id-from-authorized-metadata>"
$ClientId  = "<client-id-from-authorized-metadata>"
$Thumbprint = "<certificate-thumbprint>"

Connect-MgGraph `
    -TenantId $TenantId `
    -ClientId $ClientId `
    -CertificateThumbprint $Thumbprint

Get-MgContext
```

A successful context should indicate:

```text
AuthType = AppOnly
```

At this point, OpenBao is no longer deciding what Microsoft Graph allows.

The token's effective Microsoft Graph privilege comes from the **application permissions granted and consented to on the Entra application registration**.

That distinction is essential.

---

# Least-Privilege Microsoft Graph Applications

Each project receives its own application registration rather than sharing one generic privileged application.

Conceptually:

```text
Conditional Access Analyzer
        │
        └── only required Graph permissions

Defender Investigation Tool
        │
        └── only required Defender / Graph permissions
```

This gives the architecture two independent least-privilege boundaries.

## Credential boundary

```text
OpenBao
→ Who can retrieve or use the credential?
```

## Cloud authorization boundary

```text
Microsoft Entra ID
→ What can the application actually do?
```

Both have to be correct.

---

# Client Secrets Still Have a Place — Temporarily

The platform also supports client-secret authentication.

The flow is:

```text
Entra App Registration
        │
        └── Client Secret
                │
                ▼
         OpenBao KV v2
                │
                ▼
         Authorized Project
                │
                ▼
         App-only Graph Auth
```

This is useful for compatibility and transition.

But for persistent workloads, I prefer certificate-based authentication over maintaining long-lived password-style application credentials where certificate or stronger workload identity options are practical.

---

# Human Identity != Workload Identity

One architectural decision deserves repeating.

An engineer may authenticate to OpenBao through:

```text
Microsoft Entra ID
+
OIDC
```

but the engineering application itself should not depend on a human OIDC session forever.

Automated workloads need their own workload authentication model.

```text
Human authentication
             ≠
Workload authentication
```

This makes lifecycle, revocation and auditing much cleaner.

---

# Auditability

OpenBao audit logging is enabled.

The file audit device captures request/response activity for audited OpenBao operations.

That provides visibility into questions such as:

```text
Who authenticated?
Which secret path was accessed?
Which PKI role was used?
Was a certificate requested?
Was access denied?
Was a secret read?
```

The audit data itself is security-sensitive and is protected accordingly.

I also treat audit reliability as part of the platform rather than as optional logging added afterward.

---

# Backup and Disaster Recovery

A secrets platform without a recovery strategy is a future outage.

Because the platform uses Raft integrated storage, the recovery approach is based around Raft snapshots rather than copying live container directories and assuming the result is consistent.

The lifecycle is conceptually:

```text
OpenBao
   │
   ▼
Raft snapshot
   │
   ▼
Encrypted backup
   │
   ▼
Off-host protected storage
   │
   ▼
Periodic restore validation
```

Recovery planning also has to account for:

* OpenBao configuration;
* TLS material;
* DNS configuration;
* OIDC configuration documentation;
* unseal shares;
* CA lifecycle information.

Recovery secrets should not simply be bundled together with the data they protect.

---

# Certificate Lifecycle

Certificate authentication moves the problem from:

```text
"Where is the password?"
```

to:

```text
"How do I govern the certificate lifecycle?"
```

That is a better problem, but it is still a problem.

For every workload certificate, I want to be able to identify:

```text
Serial number
Thumbprint
Subject
Issue date
Expiration
Associated Entra application
Project owner
OpenBao issuer
OpenBao role
```

The renewal process should eventually become:

```text
Issue replacement certificate
        ↓
Register public certificate in Entra
        ↓
Validate new authentication
        ↓
Move workload to new certificate
        ↓
Remove old Entra credential
        ↓
Revoke / retire previous certificate
```

Credential rollover should happen before expiration, not after an application stops working.

---

# Validation

I did not consider the platform complete merely because OpenBao displayed:

```text
Initialized: true
Sealed: false
```

The security boundaries needed to be tested.

The validation included several layers.

## OIDC Authentication

Validated that intended users could authenticate using Microsoft Entra ID.

```text
Entra Login
    ↓
OIDC
    ↓
OpenBao Identity
    ↓
Expected policy
```

---

## Negative Authorization Testing

A project should not only be able to access what it needs.

It must also fail to access what it does not need.

I validated separation around:

```text
Project A → Project A secrets     PASS
Project A → Project B secrets     DENIED

Project A → Project A PKI role    PASS
Project A → Project B PKI role    DENIED
```

Negative tests are often more valuable than successful login tests when validating least privilege.

---

## Client-Secret Authentication

Validated:

```text
OpenBao authentication
        ↓
Authorized KV read
        ↓
Credential retrieval
        ↓
Microsoft Graph app-only authentication
```

---

## Certificate Authentication

Validated:

```text
Local key generation
        ↓
CSR
        ↓
OpenBao signing
        ↓
Certificate installation
        ↓
Public certificate registration in Entra
        ↓
Microsoft Graph certificate authentication
        ↓
App-only context
```

Both client-secret and certificate-based Microsoft Graph authentication were therefore exercised end to end.

<!-- Recommended screenshot: redacted Get-MgContext output showing AuthType = AppOnly -->

---

# Security Controls Summary

The final architecture applies several controls at different layers.

| Security Objective        | Control                                          |
| ------------------------- | ------------------------------------------------ |
| Human authentication      | Microsoft Entra ID OIDC                          |
| MFA / CA capability       | Microsoft Entra identity plane                   |
| Project isolation         | OpenBao Identity metadata + policies             |
| Secret isolation          | KV v2 project paths                              |
| Certificate isolation     | Project-specific PKI roles                       |
| Private-key protection    | Local non-exportable key + CSR                   |
| CA protection             | Root / Intermediate separation                   |
| Graph privilege           | Per-application least-privilege permissions      |
| Traceability              | OpenBao audit logs + Entra logs                  |
| Recovery                  | Raft snapshots + protected recovery material     |
| Credential lifecycle      | Expiring secrets/certificates + planned rotation |
| Administrative separation | Admin and project policies kept separate         |

The important part is that no single control is treated as sufficient.

---

# A Security Boundary I Nearly Underestimated

One of the most valuable conclusions from this project was that:

> Secure credential storage and least-privilege authorization are not the same thing.

It is possible to build an extremely secure secret store around a dangerously privileged Entra application.

It is also possible to have a perfectly least-privileged application whose private key is sitting unprotected on a workstation.

The security model therefore has to evaluate both:

```text
Credential Security
        +
Authorization Security
```

This project forced me to work across both sides of that boundary.

---

# What I Would Change for a Larger Production Deployment

This platform is designed for a controlled internal engineering environment.

If I were scaling the same architecture into a larger production environment, I would extend several areas.

## Multi-node OpenBao

The current environment values simplicity.

A production deployment requiring higher availability should use multiple OpenBao nodes with Raft replication and an appropriately designed failure domain.

---

## Stronger Unseal Strategy

Manual Shamir unseal is appropriate for learning and controlled environments.

Depending on production requirements, I would evaluate HSM/KMS-backed or other automated unseal designs with stronger operational controls.

---

## Redundant Audit Destinations

The current file audit model provides local auditability.

For production I would also evaluate:

```text
OpenBao Audit
      ↓
Multiple audit devices
      ↓
Central security monitoring / SIEM
```

This reduces reliance on one local audit destination.

---

## Automated Certificate Rotation

The current workflow proves the entire lifecycle.

The next engineering step would be automation around:

* expiry discovery;
* CSR creation;
* renewal;
* Entra credential registration;
* validation;
* old-certificate retirement.

---

## Eliminate Client Secrets Where Practical

Client secrets remain useful for compatibility.

The stronger target is:

```text
Managed Identity
or
Federated Workload Identity
or
Certificate
```

depending on where the workload actually runs.

Because this environment is locally hosted rather than Azure-native, certificate-based authentication is a practical fit.

---

## Formal Recovery Testing

Having a backup is not the same as having a recovery capability.

A more mature operational model would include scheduled restore exercises with documented recovery objectives.

---

# What This Project Taught Me

This project started as:

> "I don't want application secrets sitting on developer machines."

It became much more interesting than that.

I had to work through:

* OIDC claims;
* Entra application configuration;
* identity mapping;
* RBAC;
* templated authorization;
* secrets-engine design;
* PKI hierarchy design;
* certificate issuance;
* CSR workflows;
* Windows certificate stores;
* app-only OAuth;
* Microsoft Graph permissions;
* container networking;
* internal DNS;
* TLS;
* audit logging;
* secret recovery;
* credential lifecycle management.

More importantly, it connected those technologies into one security model.

The main lesson is:

> Security architecture is usually less about choosing one secure product and more about defining clear trust boundaries between several systems.

---

# Engineering Skills Demonstrated

From a technical perspective, this project touches several areas I work with regularly.

### Microsoft Identity

```text
Microsoft Entra ID
OIDC
App Registrations
Service Principals
Application Permissions
App-only Authentication
```

### Security Engineering

```text
Least Privilege
Separation of Duties
Credential Isolation
Trust Boundaries
RBAC
Auditability
Credential Lifecycle
```

### PKI

```text
X.509
Root CA
Intermediate CA
CSR
Certificate Signing
Certificate Stores
Private-Key Protection
Certificate Rotation
```

### Automation

```text
PowerShell
Microsoft Graph PowerShell SDK
Microsoft Graph API
Certificate Authentication
Secret Retrieval
```

### Infrastructure

```text
Proxmox VE
Ubuntu Linux
Docker
Docker Compose
DNS
TLS
Reverse Proxying
Private Networking
```

### Secrets Management

```text
OpenBao
Integrated Storage / Raft
KV v2
OIDC
Identity
Templated Policies
PKI Secrets Engine
Audit Devices
```

---

# Final Architecture

The final design can be summarized as:

```text
                           HUMAN PLANE

                      Microsoft Entra ID
                             │
                             │ OIDC
                             ▼
                         OpenBao
                 ┌───────────┼───────────┐
                 │           │           │
              Identity      KV v2       PKI
                 │           │           │
                 │      Project Secrets │
                 │                       │
                 └──── Project RBAC ─────┘


                         WORKLOAD PLANE

                     Security Project
                            │
                            │
                    Certificate / Secret
                            │
                            ▼
                  Entra App Registration
                            │
                   Application Permissions
                            │
                            ▼
                     Microsoft Graph


                      INFRASTRUCTURE PLANE

                       Proxmox VE
                            │
                      Ubuntu Server
                            │
                          Docker
                            │
             DNS / TLS / OpenBao / Git / Network
```

Each plane has a different responsibility.

```text
Entra OIDC
→ Who is the human?

OpenBao
→ Which credential may they access?

PKI
→ Which certificate may they obtain?

Entra App Registration
→ Which workload identity are they using?

Microsoft Graph permissions
→ What can that workload identity do?
```

That separation is the core of the architecture.

---

# Conclusion

What started as a way to avoid storing application secrets on individual machines became a practical exercise in **identity, PKI, secrets management, authorization, infrastructure and Microsoft Graph security**.

The final platform provides:

* centralized secrets management;
* internal certificate issuance;
* Microsoft Entra OIDC authentication;
* project-scoped authorization;
* Root/Intermediate CA separation;
* non-exportable private-key workflows;
* certificate-based Microsoft Graph authentication;
* auditable secret and PKI operations;
* per-project application identities;
* a defined credential lifecycle.

The part I value most is not that OpenBao is running.

It is that the environment now has explicit answers to questions such as:

> Who can access this credential?

> Which project owns it?

> Which certificate may that identity request?

> Where does the private key live?

> What Microsoft Graph permissions does the workload actually have?

> Can the operation be audited?

> How is the credential rotated or revoked?

Those are the questions that turn a collection of infrastructure services into a security architecture.

---

# References

The implementation and design were validated against official documentation covering:

* OpenBao Integrated Storage / Raft;
* OpenBao Identity and templated policies;
* OpenBao PKI secrets engine;
* OpenBao audit devices;
* Microsoft Entra ID OIDC and group claims;
* Microsoft Entra application credentials;
* Microsoft Graph app-only authentication;
* Microsoft Graph PowerShell certificate authentication;
* Microsoft identity platform application credential security guidance.
