---
layout: post
title: "Building a Production-Grade Microsoft 365 Security Assessment Orchestration Platform"
date: 2026-06-01
categories: [Microsoft 365, Security, Automation, PowerShell]
tags: [Microsoft Entra ID, Microsoft Graph, Maester, Monkey365, ScubaGear, ORCA, Microsoft365DSC, PowerShell, Security Assessment]
toc: true
---

# Building a Production-Grade Microsoft 365 Security Assessment Orchestration Platform

Most Microsoft 365 security assessment tools are useful on their own.

The harder problem is making several of them behave like **one trustworthy assessment platform**.

This project started from a relatively simple idea: orchestrate multiple Microsoft 365 security engines, normalize their findings, correlate them, preserve historical context, and generate a client-ready report.

It eventually became a much deeper engineering exercise around:

- authentication isolation,
- evidence integrity,
- deterministic execution,
- failure handling,
- capability reconciliation,
- immutable releases,
- source provenance,
- and deciding when an assessment result can actually be trusted.

The platform is now reaching a production-grade state, although it remains **a work in progress**.


## The Problem

A Microsoft 365 tenant can be assessed from many different perspectives:

- Microsoft Entra ID configuration
- Microsoft Graph-exposed directory relationships
- Conditional Access
- Exchange Online
- Microsoft Defender-related controls
- Microsoft 365 configuration baselines
- privilege and identity exposure
- configuration drift
- historical remediation effectiveness

No single assessment engine covers all of these areas equally well.

Instead of building another scanner from scratch, I designed an orchestration layer around several specialized engines:

| Engine | Purpose |
|---|---|
| **EntraFalcon** | Entra ID security assessment |
| **Maester** | Microsoft 365 / Entra configuration tests |
| **Monkey365** | Broad Microsoft 365 security reconnaissance and assessment |
| **ScubaGear** | CISA-aligned Microsoft 365 security baselines |
| **M365Assess** | Additional Microsoft 365 security assessment coverage |
| **ORCA** | Exchange Online protection configuration assessment |
| **Microsoft365DSC** | Configuration-state evidence and export |
| **Custom Graph collectors** | First-party directory, identity and relationship evidence |

The goal was not simply:

> Run six tools and concatenate their reports.

The real goal became:

> Produce a deterministic, explainable and auditable security assessment from heterogeneous engines while knowing exactly what ran, what did not run, which evidence supports each conclusion, and whether the final result is trustworthy.

---

# Architecture

At a high level, the platform follows this pipeline:

```text
                    Microsoft 365 Tenant
                            |
                            v
                 Certificate-Based Identities
                            |
                            v
+---------------------------------------------------------+
|                Assessment Orchestrator                  |
+---------------------------------------------------------+
     |        |        |        |        |        |
     v        v        v        v        v        v
 Entra     Maester   Monkey   Scuba    M365      ORCA
 Falcon              365      Gear    Assess
     \        |        |        |        |        /
      \_______|________|________|________|_______/
                      |
                      v
             First-Party Evidence
          Graph + Microsoft365DSC
                      |
                      v
                Normalization
                      |
                      v
                 Correlation
                      |
              +-------+-------+
              |               |
              v               v
       Privilege Paths      History
              |               |
              +-------+-------+
                      |
                      v
               Risk Intelligence
                      |
                      v
               Client Reporting
                      |
                      v
              Execution Ledger
````

Each layer has its own contract and can fail independently.

That separation became one of the most important architectural decisions in the project.

---

# Identity Isolation and Least Privilege

One of the earliest lessons was that using one highly privileged application for everything would make the platform easier to build but significantly harder to trust.

The platform therefore uses dedicated certificate-based application identities for different responsibilities.

Examples include:

```text
SharedAssessment
ORCADedicated
ActivityEvidenceDedicated
Microsoft365DSCDedicated
```

Each identity has its own:

* Entra application registration,
* service principal,
* certificate,
* permission set,
* runtime binding,
* desired state,
* and validation contract.

No client secrets are used for the production assessment path.

This separation also gave me a useful security property: **permission drift became detectable**.

For example, late in the project the validator detected that `SharedAssessment` unexpectedly contained:

```text
Agreement.Read.All
```

The permission existed both in the application's `requiredResourceAccess` and as an active service-principal app-role assignment.

It was technically valid in Microsoft Graph, but it was **not part of the platform's frozen least-privilege design**.

Instead of modifying the validator to tolerate it, the platform failed closed.

The permission was removed from the tenant and the identity returned to its intended state.

That incident reinforced an important principle:

> A security assessment platform should apply the same configuration-drift discipline to itself that it applies to the environments it assesses.

---

# Normalizing Different Security Engines

Another difficult part was that every tool describes security differently.

One engine might produce:

```text
Control Failed
```

while another exposes:

```text
Finding
Observation
Result
Check
Policy
Recommendation
```

Severity scales differ.

Object identifiers differ.

Some engines return configuration evidence while others return opinions about that evidence.

The platform therefore introduced a **canonical finding model**.

Conceptually:

```text
Engine Finding
      |
      v
Canonical Observation
      |
      +--> Affected Resource
      +--> Evidence
      +--> Security Domain
      +--> Severity
      +--> Framework Mapping
      +--> Remediation
      +--> Historical Identity
```

This allowed findings from different engines to be correlated rather than simply displayed next to each other.

A single risky configuration can therefore be supported by multiple independent evidence sources.

---

# From Findings to Security Intelligence

Once normalization worked, I wanted the system to answer more useful questions than:

> How many failed controls do we have?

The intelligence layer was expanded to include:

* cross-engine correlation,
* Conditional Access exposure,
* privilege chains,
* security choke points,
* historical state,
* remediation effectiveness,
* accepted risk,
* compensating controls,
* deferred remediation,
* regression detection.

Historical observations are classified as:

```text
New
Persistent
Resolved
Regressed
Changed
```

One subtle bug appeared here.

Initially, historical identity accidentally contained assessment-specific identifiers. The same finding from two different assessments could therefore appear to be two unrelated findings.

The real problem was not comparison logic.

It was **identity design**.

Once persistent tenant-level identity was separated from assessment identity, history became reliable.

This was a recurring pattern throughout the project:

> Several difficult-looking runtime problems were ultimately architecture problems one layer earlier.

---

# Reporting Was Harder Than Expected

Generating HTML was easy.

Generating a report that a security consultant could actually give to a client was not.

An early report contained hundreds of canonical findings, but only a small subset was meaningfully reachable through the interface.

I found issues such as:

* findings hidden by presentation logic,
* broken internal anchors,
* JavaScript errors,
* weak remediation navigation,
* insufficient explanation of security metrics,
* privilege relationships that were technically present but difficult to interpret.

The reporting layer was redesigned around a hybrid model influenced by:

* Microsoft Fluent 2,
* security consulting reports,
* OWASP-style technical clarity,
* attack-path visualization,
* evidence-oriented assessment reporting.

The final report is designed to answer questions such as:

* What was actually assessed?
* What could not be assessed?
* What are the most material risks?
* Which controls are already strong?
* Which findings reinforce each other?
* What should be fixed first?
* What evidence supports the finding?
* Has the issue existed before?
* Was remediation effective?
* How complete is the assessment evidence?

I also introduced metrics such as:

* **Observed Control Posture**
* **Evidence Coverage**
* **Coverage-Adjusted Assurance**

These are deliberately not represented as probabilities of compromise.

---

# The Long Debugging Phase

The project took considerably longer than expected.

A large part of the engineering work happened only after the platform already appeared to be "working".

That was probably the most valuable part of the project.

## Directory Evidence Race Condition

One Source assessment failed while enumerating Microsoft Graph groups.

A group existed during initial enumeration but disappeared before its members were queried.

The naive interpretation would have been:

> Graph failed.

The actual problem was a legitimate directory race.

The solution became an object-scoped revalidation strategy:

```text
enumerate object
      |
      v
query relationship
      |
      +--> object still exists -> continue
      |
      +--> object disappeared -> record DeletedObjectRace
```

Only this specific condition is tolerated.

Unexpected pagination errors, transport failures or unexplained Graph errors still fail closed.

---

## Transport Failures Without HTTP Status

Another Graph failure returned effectively:

```text
status = 0
```

There was no HTTP response at all.

This was not an authorization problem.

It was a transport-level failure.

The retry architecture had to distinguish:

```text
HTTP error
Graph semantic error
deleted-object race
transport failure
authentication failure
```

rather than applying one generic retry loop to everything.

---

# Maester: Several Bugs Hiding Behind One Failure

Maester produced some of the most difficult debugging sessions.

At one point Exchange-backed tests produced large numbers of failures and skipped controls.

The first apparent problem involved:

```powershell
$ErrorActionPreference
```

inside temporary Exchange Online proxy modules.

The key discovery was that generated `tmpEXO_*` modules had their own module-local script scope.

Changing the calling process preference was therefore insufficient.

The fix required resolving the actual owning `PSModuleInfo` and setting execution preferences inside that module's scope.

Later, another apparently identical error turned out to have a different cause:

```powershell
$ProgressPreference
```

was being restored from an invalid value during an Exchange retry path.

Same visible symptom.

Different root cause.

That experience changed the way I approached the rest of the project:

> Stop patching symptoms that look similar. Trace the exact execution boundary first.

---

# Native Artifact Integrity

Maester also exposed another architectural weakness.

The engine successfully produced results, but normalization later failed because the consumer still expected the old flat artifact path.

The producer had changed.

The consumer contract had not.

Instead of adding recursive file discovery, I implemented deterministic artifact resolution with precedence:

```text
Engine manifest
      ↓
Canonical inventory
      ↓
Explicit legacy path
```

Artifacts are validated for:

* containment,
* expected identity,
* hash,
* JSON integrity.

No recursive "find something that looks right" behavior is allowed in the certification path.

---

# Empty Data Is Not Always Failure

Another subtle issue came from Exchange Online.

Commands such as:

```powershell
Get-ReportSubmissionPolicy
Get-ReportSubmissionRule
```

can legitimately return zero objects when the feature has never been configured.

The platform originally interpreted:

```text
0 objects
```

as:

```text
collection failed
```

That was incorrect.

I introduced explicit collection states:

```text
Collected
CollectedEmpty
Unavailable
```

This small change matters because security tooling must distinguish:

> I checked and nothing exists.

from:

> I failed to check.

---

# ORCA and the Wrong Kind of Retry

ORCA later failed because Exchange Online returned a transient server-side error while reading Safe Attachment policy.

My first retry implementation wrapped the individual Exchange command.

That was a mistake.

It interfered with PowerShell command resolution and eventually caused recursive behavior.

The correct recovery boundary was much higher:

```text
ORCA Attempt 1
      |
      +--> exact authenticated Exchange transient
              |
              v
         Fresh Process
         ORCA Attempt 2
```

Each attempt is isolated.

Only the successful attempt becomes authoritative.

This became another architectural rule:

> Retry at the boundary where state can safely be reconstructed.

---

# Microsoft365DSC and Certificate Store Semantics

Microsoft365DSC exposed a particularly interesting integration problem.

The dedicated certificate existed and was valid, but it lived under:

```text
Cert:\CurrentUser\My
```

while the Microsoft365DSC authentication flow expected the certificate in:

```text
Cert:\LocalMachine\My
```

The wrong fix would have been to patch third-party authentication behavior.

Instead, I rotated the dedicated certificate into the correct store:

```text
RSA 3072
SHA-256
Client Authentication EKU
Non-exportable private key
```

The application received the new public certificate using Microsoft Graph's certificate credential flow while retaining the previous credential during validation.

The runtime then successfully authenticated using:

```text
CertificateAppOnly
```

and Microsoft365DSC successfully exported:

```text
AADNamedLocationPolicy
```

with no tenant writes.

---

# Tests Can Drift Too

After the runtime was fixed, regression still failed.

Not because Microsoft365DSC was broken.

A test simply did not supply the newly mandatory:

```text
CertificateStore
```

parameter.

Later, another test still expected the obsolete ORCA runner:

```text
Invoke-ORCA.ps1
```

instead of:

```text
Invoke-ORCAWithAttempts.ps1
```

Both were test-contract drift.

The important decision was **not to weaken production code to satisfy stale tests**.

The tests were corrected to represent the current production contract.

Eventually the complete suite reached:

```text
634 passed
0 failed
0 skipped
```

---

# When the Assessment Process Itself Died

One of the final Source assessments did not fail because of an engine.

The execution environment terminated the parent process while Monkey365 was running.

The assessment remained:

```text
Running
```

even though no assessment process was alive.

That exposed another missing lifecycle condition: **orphaned executions**.

I added reconciliation for externally interrupted assessments.

Instead of fabricating engine failures or pretending execution could resume, the run becomes explicitly:

```text
Failed — ExternalExecutionInterruption
```

Completed artifacts are preserved.

The run is excluded from:

* historical baseline selection,
* source certification,
* release freezes,
* production certification.

A completely fresh Source assessment was then executed successfully.

---

# The Final Architectural Problem: Trusting the Source Itself

By this stage:

```text
6 / 6 engines succeeded
17 / 17 capabilities assessed
Directory assessed
Microsoft365DSC assessed
Maester 418 unique tests
Execution ledger passed
Report generated
```

It looked finished.

It still was not.

The source workspace was dirty.

The assessment recorded:

```text
Git commit
dirty = true
```

but that does not cryptographically identify the actual modified bytes that were executed.

That meant I could not prove:

```text
validated source
      ==
release candidate source
```

Creating an immutable release directly from the current workspace would therefore undermine the entire release model.

This became the final significant architectural correction.

The lifecycle now implements an exact-byte **Source Freeze**.

Conceptually:

```text
Mutable Workspace
      |
      v
Exact-Byte Source Freeze
      |
      +--> per-file SHA-256
      +--> deterministic manifest
      +--> aggregate source hash
      +--> race detection
      +--> containment validation
      |
      v
Freeze-Bound Source Assessment
      |
      v
Immutable Candidate
      |
      v
Independent Validation
      |
      v
Activation
      |
      v
Frozen Production Certification
```

This was probably the most important lesson from the entire project.

A security platform is not trustworthy simply because its tests pass.

You also need to prove **what exact code those tests validated**.

---

# Current State

The platform currently provides:

* six independently orchestrated Microsoft 365 security engines;
* certificate-only application authentication;
* dedicated least-privilege identities;
* identity and permission drift validation;
* Microsoft Graph directory evidence collection;
* Microsoft365DSC configuration evidence;
* normalized canonical findings;
* cross-engine correlation;
* Conditional Access intelligence;
* privilege-path analysis;
* historical findings and remediation tracking;
* capability-level execution reconciliation;
* deterministic artifact resolution;
* transient-aware engine isolation;
* orphaned-execution reconciliation;
* client-ready HTML reporting;
* regression and certification gates;
* immutable release lifecycle;
* cryptographic source-freeze provenance.

A successful end-to-end assessment has reached:

```text
6 / 6 engines           Succeeded
17 / 17 capabilities    Assessed
Maester                  418 unique tests
Duplicate results        0
Missing capabilities     0
Silent skips             0
Execution ledger         Pass
Directory evidence       Assessed
M365DSC evidence         Assessed
Report                   Generated
```

---

# What I Learned

The most difficult part of this project was not PowerShell, Microsoft Graph, or any individual assessment engine.

It was building **trust boundaries between components**.

Most long-running problems eventually came down to one of these:

```text
Wrong execution boundary
Wrong identity boundary
Wrong artifact boundary
Wrong retry boundary
Wrong state model
Wrong provenance boundary
```

The largest improvements came when I stopped asking:

> How do I make this failing command pass?

and started asking:

> What architectural assumption allowed this failure to become ambiguous in the first place?

That shift transformed the platform from a collection of assessment scripts into something much closer to a security assessment system.

---

# Work in Progress

This project is still evolving.

There are areas I want to continue improving, particularly around:

* richer assessment intelligence,
* stronger evidence provenance,
* more advanced privilege-path modelling,
* remediation prioritization,
* historical analytics,
* report usability,
* operational certification,
* and further hardening of the release lifecycle.

But the project has already achieved the goal that mattered most to me:

> building a Microsoft 365 assessment platform where a successful result means more than simply "the scripts finished running."

It means the identities were correct, the expected capabilities executed, the evidence was reconciled, the failures were understood, the source was identifiable, and the final assessment could be defended technically.

That was considerably harder than I initially expected.

It was also the most valuable part of building it.

```
```
