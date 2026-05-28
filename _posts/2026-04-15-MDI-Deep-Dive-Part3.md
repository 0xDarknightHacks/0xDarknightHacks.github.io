---
title: "Microsoft Defender for Identity Deep Dive — Part 3: XDR Investigation, Hunting, and Response"
date: 2026-04-15
categories: [microsoft-defender-xdr, identity-security]
tags: [mdi, defender-xdr, active-directory, incident-response, advanced-hunting, kql, soc, identity-security]
toc: true
---

# Microsoft Defender for Identity Deep Dive — Part 3: XDR Investigation, Hunting, and Response

In Part 1, we treated Microsoft Defender for Identity as an identity telemetry engine.

In Part 2, we looked at how MDI detections actually work: deterministic detections, behavioral baselines, thresholds, false positives, and detection limits.

Now we move to the part that matters most in production:

> What does the SOC actually do with MDI?

This is where many deployments fail.

Not because the sensor is broken.

Not because the alerts are useless.

But because MDI is handled like a standalone alerting tool instead of an identity signal source inside Microsoft Defender XDR.

That is the wrong operating model.

MDI alerts should not be triaged in isolation.

An MDI alert is rarely the full story. It is usually one identity-centered slice of a wider attack path involving endpoint activity, authentication flows, cloud sign-ins, SaaS access, privilege changes, or lateral movement.

The right workflow is:

```text
MDI alert
   ↓
Defender XDR incident
   ↓
Alert Story and Alert Graph
   ↓
Identity Timeline
   ↓
Raw evidence and entity context
   ↓
Advanced Hunting
   ↓
Response actions
   ↓
Posture remediation
````

That last step matters.

A mature MDI program does not stop at closing alerts. It uses identity incidents to reduce attack paths before the same behavior happens again.

---

# 1. MDI alerts should not be triaged alone

A common SOC mistake is to open an MDI alert, read the title, classify it, and move on.

That may work for simple low-impact alerts, but it fails for serious identity attacks.

Identity attacks are usually chain-based.

An attacker may:

1. compromise an endpoint,
2. dump or reuse credentials,
3. query Active Directory,
4. request Kerberos tickets,
5. move laterally,
6. modify privileged groups,
7. abuse AD CS,
8. compromise a domain controller,
9. pivot to Entra ID,
10. access cloud applications.

MDI may only own part of that story.

Microsoft Defender for Endpoint may show the initial process execution.

Microsoft Defender for Identity may show LDAP reconnaissance, Kerberos abuse, lateral movement, or suspicious directory changes.

Microsoft Entra ID Protection may show cloud identity risk.

Microsoft Defender for Cloud Apps may show suspicious SaaS activity.

Microsoft Defender XDR is where these signals should come together.

## Field insight

If the SOC investigates MDI alerts only inside the identity alert view, they will often miss root cause.

They may answer:

> “What identity behavior was suspicious?”

But they may not answer:

> “Where did the compromise start, how far did it spread, and what should be contained first?”

That is why the incident page should be the default starting point.

## Architectural clarification: MDI signal vs XDR orchestration

MDI is a signal source and identity detection engine.

Defender XDR is the correlation and orchestration layer.

That distinction matters during response conversations.

MDI can raise high-value identity alerts, enrich incidents with identity context, and support built-in identity response actions. But MDI alone is not a SOAR platform, not a workflow engine, and not the place where you design complex custom incident automation.

A practical way to explain the boundary is:

> MDI tells Defender XDR what happened in the identity plane.
> Defender XDR correlates that identity signal with endpoint, cloud, email, and app signals.
> Sentinel, Logic Apps, or an external SOAR can extend automation and integration beyond built-in actions.

This prevents a common customer expectation problem: assuming that because MDI detects identity threats, it should also run any custom response workflow by itself.

---

# 2. Security alerts vs health issues

Before going deeper, separate two concepts:

* **MDI security alerts**
* **MDI health issues**

Security alerts are investigation signals. They can be correlated into Defender XDR incidents.

Health issues are operational signals. They tell you whether MDI is collecting and processing identity telemetry correctly.

Examples of health issues include:

* sensor connectivity problems,
* DSA or gMSA failures,
* missing audit policies,
* resource constraints,
* cloud communication issues,
* NNR problems,
* or sensor service failures.

These health issues are not usually handled as security incidents, but they directly affect detection confidence.

## What to watch for

Do not let the SOC ignore health issues because they are “not alerts.”

A health issue on a domain controller sensor can mean the SOC is investigating identity incidents with partial visibility.

That is not just an operations problem.

That is a detection quality problem.

---

# 3. The correct investigation entry point: Defender XDR incident

When an MDI alert appears, the first question should be:

> “Is this part of a Defender XDR incident?”

If yes, investigate the incident first.

The incident gives you:

* related alerts,
* impacted users,
* impacted devices,
* evidence,
* timeline,
* MITRE techniques,
* affected assets,
* automated investigation status,
* and response actions.

This is where MDI becomes more than an AD alert.

MDI identity context can be correlated with endpoint, email, SaaS, cloud identity, and cloud workload signals.

## Example

A standalone MDI alert might say:

> Suspicious LDAP reconnaissance from WIN11-CLIENT.

That is useful.

But inside an XDR incident, that same alert may appear beside:

* suspicious PowerShell on the same endpoint,
* credential dumping evidence from MDE,
* Kerberos ticket anomalies from MDI,
* risky sign-in from Entra ID Protection,
* unusual SharePoint download from Defender for Cloud Apps.

Now the story changes.

The LDAP alert is no longer “some reconnaissance.”

It may be the discovery phase of a larger compromise.

## Common failure mode

SOC teams sometimes close the identity alert as benign because they recognize the user or device.

That is dangerous.

A known admin account performing suspicious identity behavior may be worse, not better.

The right question is not:

> “Is this a real user?”

The right question is:

> “Was this behavior expected from this user, from this device, at this time, in this sequence?”

---

# 4. Alert Story: the chronological view

The Alert Story is one of the most useful investigation views for MDI.

It helps analysts understand what happened before and after the triggering event.

Think of it as the narrative layer.

It can show:

* the sequence of identity activities,
* involved users,
* involved devices,
* domain controllers,
* suspicious actions,
* related evidence,
* and how the alert developed.

## How to use it

When opening the Alert Story, do not only read the alert title.

Look for sequence.

Ask:

* What happened before the alert?
* Was there reconnaissance first?
* Was there credential access?
* Did the same identity touch multiple systems?
* Did the same source device interact with sensitive assets?
* Did the behavior escalate over time?
* Did another Defender workload observe related activity?

## Field insight

Alert Story is especially useful for explaining identity attacks to customers.

Many customers understand endpoint alerts because they can see a process tree.

Identity alerts are more abstract.

Alert Story gives identity activity a timeline that consultants can walk through:

> “First this device queried privileged users. Then this account requested Kerberos service tickets. Then the same identity authenticated to another host. Then a sensitive group was modified.”

That turns identity telemetry into an attack narrative.

---

# 5. Alert Graph: the relationship view

The Alert Graph helps visualize relationships between entities.

It can map connections between:

* users,
* devices,
* domain controllers,
* IP addresses,
* sensitive assets,
* and related identity objects.

The graph is useful because identity incidents are relationship-heavy.

The question is often not only:

> “What happened?”

But also:

> “Which entities are connected, and how?”

## How to use it

Use Alert Graph to answer:

* Which user was involved?
* Which device initiated the activity?
* Which domain controller observed it?
* Which sensitive account or group was touched?
* Is there a path from the source to a privileged target?
* Are there multiple devices using the same identity?
* Are there multiple identities active from the same device?
* Does the relationship graph suggest lateral movement?

## Common failure mode

Analysts sometimes treat the graph as a visual decoration.

It is not.

The graph is often the fastest way to understand blast radius.

If one compromised device is connected to several identities, or one identity is connected to several sensitive systems, that changes severity.

## Consultant callout

Teach analysts to use the graph as a scoping tool.

A good question is:

> “What does the graph show that the alert title does not?”

If the answer is nothing, the analyst probably did not inspect it deeply enough.

---

# 6. Identity Timeline: the investigation backbone

The Identity Timeline is where MDI becomes operationally powerful.

It lets analysts view identity activity over time and filter by useful dimensions such as:

* time range,
* activity type,
* protocol,
* location,
* source,
* and related entities.

This is where you reconstruct what an identity did before, during, and after an alert.

## Why the timeline matters

Attackers rarely perform one action.

They perform sequences.

The timeline helps you find those sequences.

For example:

```text
09:12 — User logs on to workstation
09:19 — Workstation performs LDAP reconnaissance
09:23 — Kerberos service tickets requested
09:31 — User authenticates to file server
09:36 — Remote logon to admin workstation
09:44 — Sensitive group membership modified
10:02 — Cloud session established
```

That sequence is much more important than any single event.

## How to investigate with the timeline

When pivoting to the impacted identity:

1. Set the time range around the alert.
2. Expand backward before the alert.
3. Expand forward after the alert.
4. Filter by protocol.
5. Filter by activity type.
6. Compare source devices.
7. Look for authentication changes.
8. Look for directory changes.
9. Look for cloud activity.
10. Export records when needed for reporting or deeper analysis.

## Field insight

The timeline is not only for confirmed incidents.

It is also useful for validating whether an alert is isolated.

If an identity has one suspicious event but no related activity before or after, the severity may be lower.

If the timeline shows reconnaissance, credential access, lateral movement, and privilege changes in sequence, severity increases quickly.

## Why customers get surprised: disabled users may still appear in timelines

A disabled account can still appear in an identity timeline or incident after containment.

That does not automatically mean the account is still successfully authenticating.

It may appear because:

* historical evidence remains visible,
* delayed telemetry is still being processed,
* the account had Kerberos tickets issued before disablement,
* endpoint sessions still reference the account,
* cached credentials exist on a device,
* cloud sessions were not revoked yet,
* or failed authentication attempts are still being recorded.

This is an important distinction:

> Disabling an account changes future authentication eligibility.
> It does not erase historical evidence or instantly invalidate every session, ticket, and cached reference.

SOC analysts should validate whether post-disable activity is:

* a new successful authentication,
* a failed authentication attempt,
* Kerberos ticket usage,
* cached endpoint context,
* historical evidence,
* delayed ingestion,
* or cloud session activity that requires revocation.

This prevents unnecessary panic and also prevents under-response.

---

# 7. Raw evidence: where confidence lives

After reviewing the story, graph, and timeline, inspect the raw evidence.

This is where analysts should validate confidence.

Important evidence areas include:

* source device,
* source IP,
* user identity,
* domain controller,
* protocol,
* timestamp,
* NNR certainty,
* entity evidence,
* alert evidence,
* directory object details,
* and related activity.

## NNR certainty matters

Network Name Resolution affects attribution.

If MDI is confident that an IP maps to a specific computer, the evidence is stronger.

If attribution is weak, the analyst needs to be careful.

Poor NNR can create confusion such as:

* wrong source device,
* unknown device,
* NAT ambiguity,
* VPN ambiguity,
* stale DNS mapping,
* or incomplete source identity.

## What to watch for

Before escalating an incident, ask:

* How certain is the source device?
* Is this a NAT or VPN address?
* Is the same IP shared by many users?
* Does MDE confirm the device identity?
* Does the device timeline support the MDI evidence?
* Is the domain controller sensor healthy?
* Were there health issues during the event window?

This is the difference between “alert handling” and real investigation.

---

# 8. Advanced Hunting: from alert review to identity threat hunting

Advanced Hunting is where detection engineers and senior SOC analysts can move beyond portal views.

MDI data appears in identity-focused hunting tables that can be used to investigate:

* authentication behavior,
* LDAP queries,
* directory changes,
* account metadata,
* alert evidence,
* incident relationships,
* and cross-domain activity.

Core tables include:

* `IdentityLogonEvents`
* `IdentityQueryEvents`
* `IdentityDirectoryEvents`
* `IdentityInfo`
* `AlertInfo`
* `AlertEvidence`

Depending on tenant integrations, additional tables may help correlate with endpoint, cloud, email, or SaaS activity.

## Table overview

| Table                     | Main use                                                         |
| ------------------------- | ---------------------------------------------------------------- |
| `IdentityLogonEvents`     | Kerberos, NTLM, and identity authentication activity             |
| `IdentityQueryEvents`     | LDAP and directory query behavior                                |
| `IdentityDirectoryEvents` | Directory object changes, group changes, sensitive modifications |
| `IdentityInfo`            | Identity metadata and enrichment                                 |
| `AlertInfo`               | Alert metadata                                                   |
| `AlertEvidence`           | Entities and evidence connected to alerts                        |

## Consultant callout

Do not teach Advanced Hunting as “run some random KQL.”

Teach it as an investigation extension.

The hunting question should come from the incident.

Examples:

* Did this identity authenticate elsewhere?
* Did this device query other sensitive accounts?
* Did other accounts perform the same LDAP pattern?
* Did this source request unusual Kerberos tickets?
* Was a sensitive group modified recently?
* Are there other alerts involving the same entity?
* Did this on-premises identity later appear in cloud activity?

---

# 9. Hunting use case: suspicious LDAP reconnaissance

Use this when an MDI alert suggests LDAP reconnaissance and you want to see whether the behavior is isolated.

```kql
IdentityQueryEvents
| where Timestamp > ago(7d)
| where QueryType has_any ("LDAP", "SAMR")
| summarize QueryCount=count(),
            FirstSeen=min(Timestamp),
            LastSeen=max(Timestamp),
            TargetCount=dcount(TargetAccountDisplayName)
    by AccountName, DeviceName, QueryType
| where QueryCount > 50 or TargetCount > 20
| order by QueryCount desc
```

## How to use it

This query helps identify accounts or devices performing unusually broad directory queries.

It is not a final detection by itself.

It is a triage query.

After finding a suspicious source, pivot into:

* the device page,
* identity timeline,
* related alerts,
* logon activity,
* endpoint process evidence,
* and whether the source is an approved admin or scanner system.

## Field insight

Do not immediately assume high query count equals malicious.

Inventory tools, IAM platforms, backup systems, scanners, and admin tools may query AD heavily.

The investigation question is:

> “Is this source expected to perform this kind of query?”

---

# 10. Hunting use case: Kerberos activity from a suspicious source

Use this when you suspect Kerberoasting, lateral movement, or abnormal Kerberos behavior.

```kql
IdentityLogonEvents
| where Timestamp > ago(7d)
| where Protocol == "Kerberos"
| summarize LogonCount=count(),
            TargetCount=dcount(TargetDeviceName),
            AccountCount=dcount(AccountName),
            FirstSeen=min(Timestamp),
            LastSeen=max(Timestamp)
    by DeviceName, AccountName, LogonType
| order by TargetCount desc, LogonCount desc
```

## How to use it

This helps identify identities or devices touching many targets through Kerberos.

It can support investigation of:

* lateral movement,
* unusual service access,
* broad ticket usage,
* or compromised admin accounts.

## What to watch for

High target count may be normal for:

* service accounts,
* management servers,
* monitoring systems,
* backup platforms,
* deployment servers,
* or privileged admin workflows.

The goal is not to flag everything.

The goal is to identify behavior that does not match the role of the account or device.

---

# 11. Hunting use case: sensitive group modifications

Use this when investigating privilege escalation or persistence.

```kql
IdentityDirectoryEvents
| where Timestamp > ago(30d)
| where ActionType has_any ("Group Membership changed", "Member added to group", "Member removed from group")
| where TargetAccountDisplayName has_any ("Domain Admins", "Enterprise Admins", "Administrators", "Account Operators", "Backup Operators")
| project Timestamp,
          ActionType,
          AccountName,
          AccountDomain,
          TargetAccountDisplayName,
          DestinationDeviceName,
          AdditionalFields
| order by Timestamp desc
```

## How to use it

This query helps identify recent changes to privileged groups.

For every result, ask:

* Who made the change?
* From which device?
* Was it approved?
* Was the source a PAW?
* Was the target account expected?
* Did the change happen near another alert?
* Was the account later used for authentication?

## Field insight

A privileged group change by a real admin is not automatically benign.

Compromised admin accounts still generate legitimate-looking directory changes.

The investigation should validate intent, not just permissions.

---

# 12. Hunting use case: identity alerts connected to a user or device

Use this when pivoting from an incident to all related identity evidence.

```kql
let Lookback = 14d;
let EntityToInvestigate = "user@domain.com";
AlertEvidence
| where Timestamp > ago(Lookback)
| where EntityType in ("User", "Machine", "Ip")
| where EvidenceRole in ("Impacted", "Related")
| where AccountUpn =~ EntityToInvestigate
   or DeviceName =~ EntityToInvestigate
   or RemoteIP =~ EntityToInvestigate
| join kind=inner (
    AlertInfo
    | where Timestamp > ago(Lookback)
    | project AlertId, Title, Severity, Category, ServiceSource, DetectionSource, Timestamp
) on AlertId
| project Timestamp, Title, Severity, Category, ServiceSource, DetectionSource, EntityType, EvidenceRole, AccountUpn, DeviceName, RemoteIP
| order by Timestamp desc
```

## How to use it

This helps analysts see whether an entity appears in other alerts.

It is useful for:

* scoping,
* prioritization,
* case management,
* and escalation.

## Consultant callout

One identity alert may not look severe alone.

But if the same user appears in endpoint, identity, and cloud app alerts, the priority changes.

That is the value of XDR.

---

# 13. Hunting use case: suspicious directory changes by non-PAW sources

Use this when assessing whether privileged changes came from expected administrative workstations.

```kql
IdentityDirectoryEvents
| where Timestamp > ago(30d)
| where ActionType has_any ("Account modified", "Group Membership changed", "User Account Changed", "Computer Account Changed")
| where isnotempty(DeviceName)
| summarize ChangeCount=count(),
            FirstSeen=min(Timestamp),
            LastSeen=max(Timestamp),
            ChangedObjects=dcount(TargetAccountDisplayName)
    by AccountName, DeviceName, ActionType
| where ChangeCount > 5 or ChangedObjects > 3
| order by ChangeCount desc
```

## How to use it

This is a starting point for identifying admin-like behavior from unexpected devices.

To make it production-ready, enrich it with:

* a PAW device list,
* privileged admin account list,
* known automation accounts,
* approved management servers,
* maintenance windows.

## Field insight

If privileged directory changes are regularly performed from normal user workstations, do not only tune the alert.

Fix the admin model.

MDI can detect suspicious behavior, but it cannot compensate for poor Tier 0 discipline.

---

# 14. Response actions: containment must be designed before the incident

MDI supports response actions through the Defender portal, but response effectiveness depends on permissions, roles, approvals, and operating model.

Possible actions include:

* disable user accounts,
* enable accounts when appropriate,
* force password reset,
* revoke Entra ID sessions,
* mark user as compromised,
* deactivate Okta accounts where integrated,
* set Okta account risk level where integrated,
* trigger PAM-integrated password reset actions,
* track actions in Action Center.

These actions are powerful because identity compromise spreads quickly.

But they also require governance.

## What usually goes wrong

The SOC sees a high-confidence identity alert but cannot act.

Reasons include:

* SOC only has read-only permissions,
* action account is not configured,
* Entra roles are missing,
* AD write permissions are missing,
* change approval is unclear,
* account disablement requires identity team approval,
* executives are excluded from response,
* or response actions were never tested.

## Architectural clarification: MDI built-in response vs custom response

A common customer question is:

> “Can we create custom response actions using MDI alone?”

The answer is no, not in the SOAR sense.

MDI alone supports limited built-in response actions, such as identity containment actions exposed through the Microsoft Defender portal, depending on permissions and configuration.

It does not provide a full custom automation engine where you design arbitrary workflows, conditional playbooks, third-party ticketing flows, enrichment chains, or multi-system remediation logic.

For custom response, you typically need one of the following:

* Microsoft Defender XDR capabilities,
* Microsoft Sentinel automation,
* Azure Logic Apps,
* Microsoft Graph / Defender APIs,
* or a third-party SOAR platform.

A useful way to explain it:

> MDI provides identity detections and limited built-in identity response actions.
> Defender XDR correlates and orchestrates security operations across Defender workloads.
> Sentinel, Logic Apps, APIs, or external SOAR platforms provide broader custom automation.

## Control-plane vs data-plane reality

Identity response is usually control-plane response.

Disabling an account, forcing a reset, or revoking sessions changes whether the identity should be allowed to authenticate or continue cloud sessions.

It does not automatically erase endpoint processes, close every network connection, remove every cached credential, invalidate every Kerberos ticket instantly, or isolate a device.

That is why identity containment is not the same as endpoint isolation.

For endpoint containment, use Microsoft Defender for Endpoint actions such as device isolation, live response, antivirus remediation, or investigation package collection.

For identity containment, use disable account, reset password, revoke sessions, mark compromised, and related identity controls.

For cross-domain orchestration, use Defender XDR, Sentinel, Logic Apps, APIs, or SOAR.

## Field insight

Do not wait for a real incident to discover that nobody can disable a compromised account.

During deployment or assessment, test response readiness with safe accounts.

Validate:

* who can perform the action,
* which systems are affected,
* where the action is logged,
* how rollback works,
* how Action Center tracks it,
* and how identity engineering is notified.

---

# 15. Automated Investigation and Response

MDI can participate in Automated Investigation and Response through Defender XDR.

AIR helps analyze and remediate threats across domains.

In practice, this means identity alerts can contribute to automated investigation chains that include endpoint and other Defender signals.

## Field insight

AIR is useful, but it should not replace human understanding.

For identity incidents, analysts still need to answer:

* Was this a real compromise?
* Which credential material may be exposed?
* Which accounts need rotation?
* Which systems were touched?
* Was privilege escalated?
* Was persistence created?
* Did the attacker pivot to cloud?
* Which posture issue enabled the path?

Automation can help contain.

It does not replace root cause analysis.

## What MDI does / does not do

MDI contributes identity evidence and built-in identity response options.

Defender XDR provides the broader automated investigation and cross-workload correlation experience.

Sentinel and Logic Apps are commonly used when the organization needs custom automation such as:

* creating ITSM tickets,
* notifying Teams or Slack channels,
* enriching incidents from CMDBs,
* calling third-party APIs,
* triggering firewall or NAC changes,
* sending alerts to non-Microsoft SOC platforms,
* or coordinating response across tools outside Defender.

The practical boundary is:

> MDI detects identity-plane abuse.
> Defender XDR correlates and investigates across Defender workloads.
> Sentinel and SOAR tools extend automation across the wider security ecosystem.

This keeps expectations realistic and avoids positioning MDI as a workflow platform.

---

# 16. Automatic Attack Disruption

Automatic Attack Disruption is one of the strongest examples of MDI’s value inside Defender XDR.

In high-confidence scenarios, Defender XDR can automatically interrupt an active attack chain.

For identity compromise, this may include disabling a compromised user account in Active Directory, which then syncs to Microsoft Entra ID.

That is powerful because it cuts across the hybrid identity boundary.

## Why this matters

Attackers move fast.

Manual response often moves slowly.

If XDR has enough confidence, automatic disruption can reduce attacker dwell time and stop lateral movement before the SOC completes manual triage.

## What to watch for

Automatic disruption should be understood, tested, and governed.

Security architects should know:

* which scenarios can trigger disruption,
* which identities are eligible,
* how exclusions are handled,
* how actions appear in Action Center,
* how identity teams are notified,
* how rollback works,
* and how business-critical accounts are protected without creating blind spots.

## Why customers get surprised: “disable user” is not instant erasure

Customers sometimes expect that disabling a user should instantly remove all activity everywhere.

That is not how identity systems work.

Disabling a user is an important containment action, but it does not instantly invalidate every existing authentication artifact.

It may not immediately remove:

* Kerberos tickets already issued before disablement,
* cached credentials on endpoints,
* already-established logon sessions,
* existing cloud tokens unless sessions are revoked,
* application sessions outside Microsoft control,
* historical evidence in Defender XDR,
* delayed telemetry still being ingested,
* or previous identity context on devices.

This is why identity response is asynchronous.

There can be a delay between:

```text
Control action taken
        ↓
Directory state updated
        ↓
Authentication systems reflect the change
        ↓
Cloud synchronization completes
        ↓
Sessions/tickets expire or are revoked
        ↓
Telemetry reflects the new state
```

So when an analyst sees activity after disablement, they should not assume one of two extremes:

* “The control failed.”
* “Everything is fine.”

They should investigate the type of activity.

Ask:

* Was it a successful new authentication?
* Was it failed authentication after disablement?
* Was it Kerberos ticket usage?
* Was it cached logon context?
* Was it delayed telemetry?
* Was it a still-active cloud session?
* Was session revocation performed?
* Was the endpoint isolated if compromise was suspected?

## Consultant callout

Automatic Attack Disruption is not a reason to weaken detection review.

It is a reason to improve response governance.

The worst outcome is not automation.

The worst outcome is automation nobody understands.

---

# 17. Scenario response: credential compromise

Credential compromise is one of the most common identity incident types.

MDI may detect signals such as:

* brute force,
* password spray,
* suspicious Kerberos activity,
* Pass-the-Hash,
* Pass-the-Ticket,
* unusual logon behavior,
* or credential-based lateral movement.

## Investigation priorities

Start with:

* impacted identity,
* source device,
* authentication protocol,
* successful vs failed logons,
* target systems,
* privilege level,
* cloud sign-in risk,
* recent endpoint alerts,
* recent password changes,
* MFA events,
* and timeline before and after compromise.

## Response priorities

Typical actions include:

* disable the account,
* force password reset,
* revoke cloud sessions,
* mark user as compromised,
* review MFA registration,
* check recent group changes,
* identify systems accessed,
* rotate exposed service credentials if relevant,
* and allow Automatic Attack Disruption where triggered.

## What to watch for

Do not reset the password too early if the attacker still has active sessions or tokens.

For hybrid compromise, password reset alone may not be enough.

Use session revocation where applicable.

Also check whether the account is used by services. Disabling a service account without planning may cause outages.

## Why disabling is only one part of containment

Credential compromise response should treat identity state, session state, endpoint state, and application state as separate but related layers.

For example:

* disabling the AD account stops normal future authentication,
* password reset invalidates the known password,
* session revocation targets cloud tokens,
* endpoint isolation stops activity from a compromised device,
* service account rotation prevents application reuse,
* and hunting confirms whether the attacker moved elsewhere.

A single button rarely completes containment.

This is why governance matters more than buttons.

## Field insight

The key credential compromise question is:

> “What did the attacker do with the credential before we contained it?”

The answer comes from the identity timeline, logon events, endpoint data, and cloud activity.

---

# 18. Scenario response: lateral movement

Lateral movement often appears as suspicious authentication or remote access behavior.

MDI may detect:

* Pass-the-Hash,
* Pass-the-Ticket,
* Overpass-the-Hash,
* NTLM relay,
* suspicious SMB enumeration,
* remote execution attempts,
* or abnormal authentication paths.

## Investigation priorities

Ask:

* What was the initial source device?
* Which account moved?
* Which authentication protocol was used?
* Which targets were accessed?
* Was the identity privileged?
* Was this account normally used from that source?
* Did MDE see remote execution?
* Did the attacker reach Tier 0 systems?
* Was lateral movement blocked or successful?
* Are there multiple target devices?

## Response priorities

Actions may include:

* isolate or investigate the source endpoint through MDE,
* disable or reset the compromised identity,
* revoke sessions,
* remove active SMB/RDP exposure where needed,
* reset credentials used on touched systems,
* review local admin relationships,
* review lateral movement paths,
* restrict access to domain controllers,
* and validate whether privileged systems were reached.

## Field insight

Lateral movement is where MDE and MDI must be used together.

MDI can show identity movement.

MDE can show process and device activity.

If you only look at one side, you may miss either the identity path or the execution mechanism.

---

# 19. Scenario response: domain controller compromise

Domain controller compromise is a Tier 0 incident.

Treat it differently from normal user compromise.

MDI may surface DC compromise through alerts related to:

* DCSync,
* DCShadow,
* suspicious service creation,
* Skeleton Key,
* Golden Ticket,
* abnormal replication,
* suspicious remote execution,
* or privileged directory modifications.

## Investigation priorities

Immediately determine:

* which DC was involved,
* which identity performed the action,
* source device,
* replication activity,
* privileged group changes,
* GPO modifications,
* service creation,
* suspicious logons to DCs,
* AD CS or AD FS involvement,
* KRBTGT exposure possibility,
* and whether persistence exists.

## Response priorities

Possible actions include:

* contain compromised accounts,
* restrict remote access to DCs,
* investigate source hosts,
* remove unauthorized DCSync rights,
* revert malicious GPO changes,
* remove suspicious services,
* patch vulnerable DCs,
* rotate exposed privileged credentials,
* review local admin exposure,
* validate AD replication health,
* and involve identity engineering immediately.

## Common failure mode

Customers treat DC compromise as a normal SOC ticket.

That is wrong.

A domain controller is not just another server.

If the attacker controls a DC or DC-equivalent privileges, they may control the domain.

## Consultant callout

For DC compromise, the response should be led as a Tier 0 incident with identity engineering, infrastructure, SOC, and security architecture involved.

The question is no longer:

> “Which user was compromised?”

The question becomes:

> “Can we still trust the domain?”

---

# 20. Scenario response: Golden Ticket

Golden Ticket detection is one of the most serious MDI scenarios.

It suggests forged Kerberos Ticket Granting Tickets and possible KRBTGT compromise.

## Investigation priorities

Ask:

* Which account used the suspicious ticket?
* Which device presented it?
* Which domain controller observed it?
* What anomaly was detected?
* Was KRBTGT material potentially exposed?
* Was there prior DCSync activity?
* Was there Domain Admin compromise?
* Were there suspicious logons to DCs?
* Were privileged groups modified?
* Are there other persistence mechanisms?

## Response priorities

For confirmed Golden Ticket activity:

* investigate how KRBTGT compromise occurred,
* identify Domain Admin or replication-rights compromise,
* check for DCSync,
* check for DC compromise,
* check for persistence,
* reset KRBTGT password twice,
* wait at least 10 hours between resets,
* rotate exposed privileged credentials,
* validate Kerberos ticket lifetimes,
* and monitor for continued forged ticket activity.

## Why the double KRBTGT reset matters

Resetting KRBTGT once is not enough to fully invalidate all forged ticket scenarios.

The supported response requires resetting it twice, with appropriate waiting time between resets so existing Kerberos tickets can expire or be invalidated.

## Field insight

A Golden Ticket alert is not the beginning of the incident.

It is usually evidence that the attacker already reached deep privilege.

The real investigation is backward:

> “How did they get enough access to forge this?”

---

# 21. MDI as posture intelligence

MDI is not only detection technology.

It also contributes to identity posture management.

This is where security architects should pay close attention.

MDI can help identify systemic risks such as:

* stale accounts,
* unsecure Kerberos delegation,
* unmonitored domain controllers,
* risky lateral movement paths,
* local admin exposure on sensitive systems,
* weak password exposure,
* sensitive group changes,
* GPO modification risks,
* AD CS exposure,
* and Tier 0 monitoring gaps.

## Lateral Movement Paths

Lateral Movement Path analysis helps visualize routes an attacker could take from a lower-privileged account or system toward a sensitive asset.

This matters because many incidents are not caused by one catastrophic misconfiguration.

They are caused by chains.

Example:

```text
User workstation
   ↓ local admin relationship
Management server
   ↓ cached privileged session
Admin account
   ↓ group membership
Sensitive server
   ↓ domain privilege
Domain Admin path
```

MDI can help expose these paths so teams can reduce them before compromise.

## Consultant callout

Lateral Movement Paths should not only be reviewed after incidents.

They should be part of recurring identity posture review.

The best time to remove an attack path is before an attacker uses it.

---

# 22. Secure Score and Identity Security Posture Management

MDI contributes identity-related posture recommendations into Microsoft Secure Score and identity security posture views.

Examples include:

* resolving unsecure Kerberos delegation,
* removing unnecessary local admins on identity assets,
* addressing unmonitored identity servers,
* removing risky DCSync permissions,
* hardening domain controller exposure,
* improving password hygiene,
* and reducing insecure configurations.

## Field insight

Security leadership often wants scores.

Engineers often want evidence.

MDI posture data can serve both.

For leadership:

> “Identity exposure is trending down.”

For engineers:

> “These specific accounts, servers, permissions, and paths need remediation.”

The consultant’s job is to connect both views without turning Secure Score into a vanity metric.

---

# 23. Identity risk and prioritization

MDI can enrich identities with risk context.

Risk is influenced by factors such as:

* privileged role,
* sensitivity,
* alert history,
* exposure,
* lateral movement path involvement,
* and cloud identity signals where available.

This helps analysts prioritize.

A suspicious event involving a normal user is one thing.

The same event involving:

* a Domain Admin,
* Entra Connect account,
* AD FS service account,
* AD CS admin,
* Tier 0 operator,
* or sensitive service account

is very different.

## What to watch for

Sensitive identities must be correctly tagged and maintained.

If the environment does not identify sensitive users, service accounts, PAWs, and Tier 0 assets, MDI and XDR correlation lose important prioritization context.

## Consultant callout

Entity tagging is not cosmetic.

It affects investigation quality.

A well-tagged environment helps analysts understand impact faster.

---

# 24. Reporting and leadership value

MDI also provides reporting value for different audiences.

## Auditors

Relevant outputs include:

* sensitive group modification reports,
* clear-text password exposure reports,
* summary reports,
* health issue history,
* Advanced Hunting data,
* RBAC scoping review.

## Security leadership

Relevant views include:

* Identity Security Dashboard,
* Coverage and Maturity views,
* identity risk trends,
* identity security initiatives,
* deployment status,
* posture trends.

## Risk assessors

Relevant capabilities include:

* Identity Security Posture Management,
* Lateral Movement Path analysis,
* Identity Risk Score,
* Domain Investigation views,
* Password Protection views.

## Field insight

Do not present MDI reporting as “more dashboards.”

Present it as identity risk evidence.

The strongest executive message is:

> “Here are the identity paths that could lead to domain compromise, and here is how we are reducing them.”

---

# 25. Integration reality: SIEM, Sentinel, KUMA, and third-party SOAR

A common customer question is:

> “Can we forward MDI telemetry directly to a third-party SIEM like Kaspersky KUMA?”

The clean answer is:

> MDI telemetry is not typically forwarded directly from MDI to third-party SIEMs as a native MDI forwarding pipeline.

MDI is integrated into Microsoft Defender XDR. From there, organizations can use supported integration paths to export or consume alerts, incidents, and security data.

For third-party SIEM or SOAR integration, common options include:

* Microsoft Sentinel,
* Microsoft Graph Security API,
* Microsoft Defender XDR APIs,
* streaming/export mechanisms supported by the Microsoft security stack,
* Logic Apps,
* or a third-party SOAR connector that consumes Microsoft security APIs.

The important point:

> MDI is not the SIEM forwarding layer.
> Defender XDR and Sentinel are the more appropriate integration layers.

## Is Sentinel required?

Sentinel is not required for MDI to detect identity threats or for Defender XDR to correlate incidents.

MDI and Defender XDR can operate without Sentinel.

But Sentinel becomes important when the customer needs broader SIEM/SOAR capabilities, such as:

* ingesting logs from many non-Microsoft sources,
* long-term retention,
* custom analytics across Microsoft and non-Microsoft data,
* playbook automation through Logic Apps,
* integration with third-party ticketing systems,
* forwarding to external platforms,
* centralized SOC workflows,
* or custom cross-platform response.

So the practical answer is:

> Sentinel is not required for MDI detection.
> Sentinel is often the right layer for broader integration, automation, retention, and SIEM/SOAR workflows.

## What belongs where?

| Capability                                                        | Best-fit layer                           |
| ----------------------------------------------------------------- | ---------------------------------------- |
| Identity signal collection from AD/hybrid identity infrastructure | MDI                                      |
| Identity detections and posture findings                          | MDI                                      |
| Cross-workload incident correlation                               | Defender XDR                             |
| Built-in investigation experience                                 | Defender XDR                             |
| Built-in identity response actions                                | MDI / Defender XDR portal                |
| Endpoint isolation and endpoint response                          | Microsoft Defender for Endpoint          |
| Custom playbooks and automation                                   | Sentinel / Logic Apps / SOAR             |
| Long-term SIEM retention and cross-source analytics               | Sentinel or third-party SIEM             |
| Third-party SIEM/SOAR integration                                 | Sentinel, Graph/API, connectors, or SOAR |
| Data exfiltration / content inspection                            | Purview DLP / MDCA / storage/app logs    |

## Consultant framing

For a customer using Kaspersky KUMA or another SIEM, the discussion should not be:

> “Can MDI send syslog directly?”

The better discussion is:

> “Which Microsoft security signals do you need outside Defender XDR, which supported export path will provide them, and what security outcome do you expect in the third-party SIEM?”

If the goal is alert visibility, API-based alert ingestion may be enough.

If the goal is correlation across Microsoft and non-Microsoft sources, Sentinel or the third-party SIEM architecture needs to be designed carefully.

If the goal is automated containment, use Defender XDR built-in actions where possible and Sentinel/Logic Apps/SOAR for custom workflows.

Do not force MDI to act like a syslog appliance.

That is not its architectural role.

---

# 26. Case management and classification

MDI alerts should feed into a disciplined SOC classification process.

Recommended classifications include:

* True Positive,
* Benign True Positive,
* False Positive.

These categories matter.

## True Positive

The alert represents malicious or unauthorized activity.

Action:

* contain,
* investigate scope,
* remediate root cause,
* document lessons learned.

## Benign True Positive

The alert correctly identified suspicious-looking behavior, but the activity was authorized.

Example:

* approved scanner,
* approved admin script,
* planned maintenance,
* red team activity.

Action:

* document evidence,
* consider narrow tuning,
* avoid broad exclusion.

## False Positive

The alert does not represent the behavior it claims or is technically incorrect in context.

Action:

* validate evidence,
* tune carefully,
* improve telemetry,
* review NNR or baseline issues.

## Common failure mode

Many teams classify everything non-malicious as False Positive.

That is wrong.

A scanner triggering reconnaissance detection is often a Benign True Positive.

The detection worked.

The activity was just authorized.

That distinction helps preserve confidence in MDI.

## Architectural clarification: case management vs response automation

Case management is not the same as response automation.

Defender XDR can manage incidents and evidence across Defender workloads.

Sentinel can extend case workflows, analytics, automation, and integrations across Microsoft and non-Microsoft sources.

A third-party SOAR can orchestrate actions across many external tools.

MDI should not be expected to replace any of those.

MDI’s job is to provide identity detections, identity context, and identity response hooks.

The SOC operating model decides where the case is owned:

* Defender XDR for Microsoft-first incident operations,
* Sentinel for SIEM/SOAR-centric operations,
* third-party SIEM/SOAR for external SOC platforms,
* or a hybrid model with clear ownership.

The dangerous model is having alerts in multiple places with no single owner.

---

# 27. Playbook structure for SOC teams

A practical MDI playbook should include the following phases.

## Phase 1: Incident intake

* Start from the Defender XDR incident.
* Review severity.
* Identify impacted entities.
* Review all correlated alerts.
* Confirm whether MDI is one signal or the primary signal.
* Confirm where the case is owned: Defender XDR, Sentinel, or another SOC platform.

## Phase 2: Identity story reconstruction

* Open Alert Story.
* Review Alert Graph.
* Pivot to Identity Timeline.
* Expand the time range.
* Filter by protocol and activity type.
* Look for reconnaissance, credential access, lateral movement, and modifications.

## Phase 3: Evidence validation

* Validate source device.
* Check NNR certainty.
* Review domain controller evidence.
* Confirm user context.
* Review related endpoint evidence.
* Check cloud sign-in risk.
* Identify sensitive asset involvement.
* Distinguish historical evidence from new post-containment activity.

## Phase 4: Hunting and scoping

* Query `IdentityLogonEvents`.
* Query `IdentityQueryEvents`.
* Query `IdentityDirectoryEvents`.
* Join with `AlertInfo` and `AlertEvidence`.
* Search for related users, devices, IPs, groups, and domains.
* Identify whether the behavior occurred elsewhere.

## Phase 5: Containment

* Disable compromised accounts where appropriate.
* Revoke sessions.
* Force password reset.
* Isolate endpoint through MDE if needed.
* Restrict access to sensitive systems.
* Allow or review Automatic Attack Disruption actions.
* Confirm that identity containment, endpoint containment, and cloud session containment are all addressed separately.

## Phase 6: Eradication and recovery

* Remove persistence.
* Revert unauthorized directory changes.
* Reset exposed credentials.
* Remove risky permissions.
* Patch affected systems.
* Harden protocols.
* Review Tier 0 access.
* Validate whether Kerberos ticket lifetime, cached credentials, or cloud sessions still create residual risk.

## Phase 7: Posture remediation

* Review lateral movement paths.
* Review Secure Score recommendations.
* Fix risky delegation.
* Remove unnecessary local admins.
* Monitor sensitive group changes.
* Improve NNR and sensor health.
* Update tuning rules.

## Phase 8: Integration and automation review

* Decide whether the incident should remain in Defender XDR or be synchronized to Sentinel/SOAR.
* Validate whether external SIEM ingestion is required.
* Confirm whether Graph/API, Sentinel, Logic Apps, or a SOAR connector is the correct integration path.
* Document which system owns the case lifecycle.
* Document which system owns automation.
* Document rollback and approval requirements for response actions.

## Consultant warning

A playbook that says “disable user” is incomplete.

A mature playbook must specify:

* who approves the action,
* who performs it,
* whether sessions are revoked,
* whether endpoint isolation is required,
* whether service accounts are impacted,
* whether Kerberos tickets remain relevant,
* whether cloud sync delay matters,
* how the action is verified,
* and where the action is tracked.

Governance matters more than buttons.

---

# 28. What mature MDI operations look like

A mature MDI operating model has clear ownership.

## Identity engineers own

* sensor coverage,
* sensor health,
* DSA/gMSA configuration,
* audit policy,
* NNR dependencies,
* AD FS coverage,
* AD CS coverage,
* Entra Connect coverage,
* response-action permissions,
* Tier 0 infrastructure readiness.

## SOC analysts own

* incident triage,
* Alert Story review,
* Alert Graph review,
* Identity Timeline investigation,
* Advanced Hunting,
* alert classification,
* evidence validation,
* response execution,
* case notes,
* escalation.

## Detection engineers own

* hunting content,
* detection validation,
* tuning rules,
* false-positive analysis,
* test mode planning,
* baseline-aware testing,
* identity threat queries,
* cross-domain correlation.

## Security architects own

* Zero Trust alignment,
* Tier 0 design,
* privileged access model,
* lateral movement reduction,
* Secure Score prioritization,
* identity risk strategy,
* XDR integration model,
* response governance.

## SIEM / SOAR engineers own

* Sentinel or third-party SIEM ingestion design,
* API-based integrations,
* Logic App or SOAR playbooks,
* ticketing integration,
* alert normalization,
* retention strategy,
* cross-platform automation,
* and external notification workflows.

## Field insight

MDI fails operationally when every team thinks another team owns it.

It is an identity product, an XDR signal source, a SOC investigation tool, and a posture engine.

That means ownership must be shared but clearly defined.

---

# 29. Final consultant framing

When explaining MDI operations to a customer, I would avoid saying:

> “When MDI raises an alert, the SOC should investigate it.”

That is too generic.

A stronger framing is:

> “MDI alerts should be investigated as identity evidence inside Defender XDR incidents. Analysts should start from the incident, reconstruct the attack sequence through Alert Story, Alert Graph, and Identity Timeline, validate raw evidence and attribution, use Advanced Hunting to scope related activity, execute response actions where appropriate, and feed lessons learned into identity posture remediation.”

I would also avoid saying:

> “MDI can automate our response.”

That creates the wrong expectation.

A more accurate response architecture is:

> “MDI provides identity detections, posture findings, and limited built-in identity response actions. Defender XDR correlates and orchestrates across Microsoft security workloads. Sentinel, Logic Apps, APIs, or external SOAR platforms provide broader custom automation and third-party integration.”

That is the operating model.

MDI’s value is not only that it detects identity attacks.

Its value is that it helps answer five production questions:

1. **What happened?**
2. **Which identity and device were involved?**
3. **How far did the attacker move?**
4. **What can we contain now?**
5. **Which identity path must be removed so this does not happen again?**

And when response begins, the team must remember:

> Identity containment is asynchronous.
> Disabling an account is not the same as isolating a device.
> Revoking a session is not the same as invalidating every Kerberos ticket.
> MDI detection is not SOAR automation.
> Defender XDR is the orchestration layer, and Sentinel or SOAR extends integration beyond it.

That is why MDI belongs inside the Defender XDR learning journey.

It is not just a sensor on a domain controller.

It is the identity layer that helps turn authentication, directory, protocol, and privilege activity into investigation, response, and risk reduction.

---

# Series conclusion

Across this three-part series, the main message is consistent:

> Microsoft Defender for Identity is the on-premises and hybrid identity signal engine inside Microsoft Defender XDR.

In Part 1, we looked at architecture and telemetry.

The key lesson was:

> Detection quality starts with sensor coverage, event collection, ETW, directory enrichment, NNR, and Tier 0 visibility.

In Part 2, we looked at detection logic.

The key lesson was:

> MDI detections are a mix of deterministic patterns and behavioral baselines. Testing and tuning must respect visibility, maturity, and context.

In Part 3, we looked at operations.

The key lesson is:

> MDI alerts become valuable when they are investigated as part of Defender XDR incidents, enriched with identity timelines and hunting, connected to response actions, and used to reduce attack paths.

The operational clarification is just as important:

> MDI is not a SOAR platform and not a third-party SIEM forwarding engine. It is the identity signal and detection layer. Defender XDR correlates and orchestrates Microsoft security operations. Sentinel, Logic Apps, APIs, and external SOAR platforms extend automation, retention, and third-party integration.

That is the consultant-grade view.

Not “MDI detects AD attacks.”

But:

> MDI turns hybrid identity behavior into security intelligence that Defender XDR can use to detect, investigate, disrupt, and reduce identity-driven attack paths.