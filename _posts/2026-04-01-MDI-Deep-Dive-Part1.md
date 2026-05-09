---
title: Microsoft Defender for Identity Deep Dive — Part 1: Architecture, Sensors, and Identity Telemetry
date: 2026-04-01
categories: [microsoft-defender-xdr, identity-security]
tags: [mdi, defender-xdr, active-directory, kerberos, ntlm, ldap, identity-security, soc, detection-engineering]
toc: true
---

# Microsoft Defender for Identity Deep Dive — Part 1: Architecture, Sensors, and Identity Telemetry

Microsoft Defender for Identity is often introduced as a tool that detects attacks against Active Directory.

That description is not wrong, but it is incomplete.

In real Microsoft Defender XDR environments, MDI should be understood as the **on-premises and hybrid identity signal engine**. Its job is not simply to raise alerts when something suspicious happens on a domain controller. Its real value is that it transforms Active Directory protocol activity, authentication flows, directory changes, and identity infrastructure signals into something Defender XDR can use:

- correlated incidents,
- identity investigation pivots,
- behavioral baselines,
- lateral movement context,
- posture findings,
- and response actions.

That distinction matters.

If you treat MDI as “just an AD IDS,” you will focus only on whether an alert fired.

If you treat MDI as an identity signal engine, you will ask better questions:

- Are all identity infrastructure systems covered?
- Are the sensors collecting the right signals?
- Is Network Name Resolution reliable?
- Are Windows events and ETW available?
- Are baselines mature enough?
- Are MDI alerts being correlated with endpoint, cloud app, email, and Entra ID signals?
- Can analysts pivot from an identity alert into the wider XDR story?

This first part focuses on architecture and telemetry. The goal is not to explain how to click through an installation wizard. The goal is to understand how MDI sees the environment, where that visibility breaks, and what consultants should validate before trusting the detections.

---

# 1. MDI is not “Cloud ATA rebranded”

A common mistake is to describe Microsoft Defender for Identity as the cloud version of Microsoft Advanced Threat Analytics.

That comparison helps people understand the history, but it undersells what MDI has become.

MDI is cloud-based, integrated into Microsoft Defender XDR, and designed to enrich cross-domain incidents with identity context. It does not operate like a standalone appliance where identity alerts live in a separate console and analysts manually stitch them together with endpoint or cloud activity.

In a mature Defender XDR environment, MDI contributes identity evidence into the same investigation experience used for Microsoft Defender for Endpoint, Microsoft Defender for Office 365, Microsoft Defender for Cloud Apps, Microsoft Entra ID, and Microsoft Sentinel integrations.

That is the key architectural shift:

> MDI is not the destination.  
> MDI is a source of identity intelligence for Defender XDR.

It monitors identity infrastructure, analyzes identity behavior, and feeds the wider XDR engine with signals that help explain how an attacker moved through the environment.

This is especially important in hybrid environments, where an attacker may start on an endpoint, authenticate against Active Directory, abuse Kerberos or NTLM, touch a privileged group, move laterally, and later pivot into cloud resources.

Without MDI, Defender XDR may still see endpoint and cloud activity. But it loses a deep view of the on-premises identity layer where many attacks become serious.

---

# 2. The architectural model: local collection, cloud analytics, XDR correlation

At a high level, MDI has four major architectural parts:

1. **MDI sensors**
2. **Directory enrichment**
3. **MDI cloud analytics**
4. **Defender XDR correlation**

The sensors sit close to identity infrastructure. They collect local signals from domain controllers and other identity servers. The cloud service performs analytics, stores identity security data, and exposes alerts, posture findings, and investigation views through the Microsoft Defender portal.

The important point is that MDI does not simply forward raw domain controller traffic to the cloud.

The sensor performs local capture and parsing first. It extracts the required identity signals, enriches them with directory context, and sends the relevant telemetry to the cloud service over secure outbound connectivity.

That design matters for two reasons.

First, it reduces unnecessary data transfer. Domain controllers can process huge amounts of authentication and directory traffic. Sending all raw traffic to the cloud would be inefficient and operationally unrealistic.

Second, it allows MDI to convert noisy low-level identity activity into structured security signals before correlation.

A simplified flow looks like this:

```text
Active Directory activity
        ↓
MDI sensor on identity infrastructure
        ↓
Local parsing of protocols, events, and ETW
        ↓
Directory enrichment through DSA / local context
        ↓
Secure outbound transmission to MDI cloud service
        ↓
Behavioral analytics and detection logic
        ↓
Microsoft Defender XDR incident correlation
        ↓
Analyst investigation, hunting, posture, and response
````

This is why I prefer to describe MDI as an **identity telemetry pipeline**, not just an alerting product.

The quality of the final alert depends heavily on the quality of each earlier stage.

If the sensor is missing, unhealthy, undersized, or blind to certain events, Defender XDR cannot magically reconstruct the identity story later.

---

# 3. What MDI actually collects

MDI visibility comes from several signal types.

The most important are:

* network protocol activity,
* Windows security events,
* directory information,
* Event Tracing for Windows,
* and identity metadata.

These sources complement each other. None of them is enough alone.

## 3.1 Protocol visibility

MDI analyzes identity-related protocols such as:

* Kerberos,
* NTLM,
* LDAP,
* LDAPS,
* DNS,
* SMB,
* RPC,
* SAM-R,
* Netlogon,
* and authentication-related flows against domain controllers.

This is where MDI gets much of its value.

Identity attacks often leave protocol-level traces before they leave obvious endpoint artifacts. For example:

* LDAP queries can reveal reconnaissance.
* Kerberos ticket requests can reveal Kerberoasting behavior.
* NTLM authentication patterns can reveal lateral movement or relay behavior.
* Directory replication requests can reveal DCSync-style abuse.
* SAM-R queries can reveal enumeration of users, groups, and privileged relationships.

A SIEM can ingest domain controller logs, but logs alone may not fully describe the protocol behavior. MDI is designed specifically to understand identity protocol semantics.

That is why MDI often detects things that generic log rules miss.

## 3.2 Windows events

MDI also relies on Windows events for important context.

Examples include events related to:

* logons,
* NTLM authentication,
* directory object changes,
* group membership changes,
* account modifications,
* service creation,
* AD FS activity,
* AD CS activity,
* and Entra Connect-related behavior.

This is where many real deployments fail.

Customers sometimes install the sensor and assume visibility is complete. But if required auditing is missing or inconsistent, MDI may not receive the event context needed for certain detections or investigation views.

That is why “sensor installed” does not mean “MDI operational.”

A sensor can be running, connected, and still not producing the level of security value the customer expects.

## 3.3 ETW

Event Tracing for Windows is one of the architectural reasons direct sensors are preferred over standalone sensors.

ETW gives MDI access to additional operating system-level signals that are not available through standard logs or mirrored network traffic alone.

This distinction is critical when comparing sensor deployment models.

A standalone sensor can observe mirrored traffic, but it does not have the same local access to ETW on the domain controller. That reduces detection coverage.

So when a customer says:

> “We do not want agents on domain controllers. We will use standalone sensors instead.”

The consultant response should not be:

> “That is fine, same coverage.”

The correct response is:

> “That is possible, but it changes the visibility model. Standalone sensors require network mirroring and do not support ETW, so some advanced detection coverage is reduced.”

That trade-off should be explicitly documented.

## 3.4 Directory enrichment

MDI does not only see raw activity. It enriches that activity with directory context.

That context includes things like:

* users,
* computers,
* groups,
* privileges,
* sensitivity,
* deleted objects,
* domain relationships,
* and lateral movement paths.

This enrichment depends on the Directory Service Account model, permissions, and the ability to query Active Directory correctly.

If the DSA is misconfigured, expired, missing permissions, or not usable by the sensor, MDI loses important context.

In real environments, these issues often surface as “MDI is not detecting enough,” but the root cause is not detection logic. The root cause is usually identity integration health.

---

# 4. Sensor types: the decision that shapes visibility

MDI sensor architecture is one of the most important design areas for consultants.

There are three sensor models you need to understand:

1. classic MDI sensor v2.x,
2. standalone sensor v2.x,
3. MDI sensor v3.x via Microsoft Defender for Endpoint.

Each model has different operational and detection implications.

---

# 5. Classic MDI sensor v2.x

The classic MDI sensor is installed directly on identity infrastructure.

Typical deployment targets include:

* domain controllers,
* read-only domain controllers,
* AD FS servers,
* AD CS servers,
* Microsoft Entra Connect servers.

This model provides strong coverage because the sensor runs directly on the server it monitors.

It can collect:

* local network traffic,
* Windows events,
* directory information,
* ETW-based signals where supported.

From a detection perspective, this is usually the cleanest model.

It avoids the complexity of port mirroring, gives the sensor direct access to the relevant local telemetry, and supports richer detection coverage than standalone monitoring.

## Field insight

In most production environments, direct sensor installation on domain controllers is the preferred model when the servers meet requirements and the organization allows security agents on Tier 0 infrastructure.

The resistance is usually political or operational, not technical.

Common concerns include:

* “We do not install agents on DCs.”
* “This might affect authentication performance.”
* “The domain controller team does not own Defender.”
* “We already send DC logs to Sentinel.”
* “We have network IDS monitoring the domain controllers.”

These concerns are understandable, but they should be challenged carefully.

MDI is not simply another generic agent. It is the component that gives Defender XDR deep identity visibility. If it is not installed where identity traffic actually happens, detection confidence drops.

---

# 6. Standalone sensor v2.x

The standalone sensor is installed on a dedicated server instead of directly on the domain controller.

It observes domain controller traffic through:

* port mirroring,
* SPAN,
* RSPAN,
* ERSPAN,
* or network TAP.

This model exists for environments where direct installation on domain controllers is prohibited or technically impossible.

It can be useful, but it is not equivalent to a direct sensor.

## The key limitations

The standalone sensor:

* requires reliable traffic mirroring,
* needs correct network adapter configuration,
* may require manual Windows Event Forwarding or syslog,
* must be sized for the aggregate traffic it receives,
* and does not support ETW.

The ETW limitation is the big one.

Because ETW contributes to several advanced detections, standalone sensors provide reduced coverage compared to sensors installed directly on identity infrastructure.

## Common failure mode

The most common standalone sensor failure is assuming that network mirroring is complete.

In diagrams, port mirroring looks simple.

In real environments, it often breaks because:

* only one direction of traffic is mirrored,
* traffic from some VLANs is missing,
* virtual switch traffic is not mirrored correctly,
* load-balanced DC traffic is incomplete,
* encrypted traffic limits inspection,
* or high-volume authentication traffic overwhelms the capture path.

When that happens, MDI does not necessarily fail loudly in the way a consultant expects. Instead, the customer may see partial detections, weak entity attribution, or incomplete attack stories.

That is worse than a clean failure because everyone believes the environment is monitored.

## What to watch for

Use standalone sensors only when there is a clear reason.

Good reasons include:

* direct DC installation is blocked by policy,
* temporary monitoring is needed during a transition,
* legacy infrastructure prevents direct installation,
* or the customer accepts reduced visibility as a documented risk.

Bad reasons include:

* “We want the easiest approval path.”
* “We assume it is the same as a normal sensor.”
* “The network team says SPAN is good enough.”
* “We do not want to talk to the AD team.”

Standalone sensors can be part of a valid architecture, but they should never be sold as full parity.

---

# 7. MDI sensor v3.x via Microsoft Defender for Endpoint

Sensor v3.x changes the deployment model.

Instead of installing the classic MDI package separately, the sensor is delivered through Microsoft Defender for Endpoint on supported domain controllers.

This is strategically important because Microsoft is moving toward a more integrated Defender platform model.

The v3.x sensor is activated through the Microsoft Defender portal on eligible MDE-onboarded domain controllers. Updates are handled through Windows Update rather than the older dedicated sensor updater model.

## Why v3.x matters

The v3.x model improves operational consistency because it aligns MDI deployment with MDE infrastructure.

It also introduces important differences:

* it requires Microsoft Defender for Endpoint,
* it applies to supported newer Windows Server versions,
* it uses LocalSystem identity for actions,
* it has different resource caps,
* it can simplify Network Name Resolution by using Defender device inventory,
* and it can automatically configure certain auditing requirements.

This can reduce operational friction.

But it also creates migration considerations.

## Common migration issue

One important field issue is response-action identity.

In many v2.x deployments, customers configure a gMSA for MDI-related directory access or response actions.

With v3.x, the sensor uses the LocalSystem identity for certain actions. If the previous gMSA configuration remains in place incorrectly, response actions may fail or behave differently than expected.

That is exactly the kind of detail that is easy to miss in a migration plan.

The migration conversation should include:

* operating system eligibility,
* MDE onboarding status,
* existing v2.x sensor health,
* gMSA usage,
* response-action permissions,
* auditing behavior,
* proxy/connectivity model,
* and rollback expectations.

## Field insight

v3.x is not just “newer sensor, same design.”

It changes the operational dependency model.

Identity engineers and endpoint engineers now have to coordinate more closely because MDI activation depends on the MDE foundation.

That is good for platform consistency, but it can create ownership confusion.

In some organizations, the AD team owns domain controllers, the endpoint team owns MDE, and the SOC owns Defender XDR. If nobody owns the intersection, MDI health suffers.

---

# 8. Sensor comparison

Here is the practical consultant view.

| Area                  | Classic sensor v2.x                          | Standalone sensor v2.x                        | Sensor v3.x via MDE                                            |
| --------------------- | -------------------------------------------- | --------------------------------------------- | -------------------------------------------------------------- |
| Installed on          | DCs, RODCs, AD FS, AD CS, Entra Connect      | Dedicated monitoring server                   | MDE-onboarded supported DCs                                    |
| Traffic source        | Local traffic on monitored server            | Mirrored DC traffic                           | Local identity signals via MDE-based model                     |
| Windows events        | Local                                        | Requires forwarding/syslog for some scenarios | Improved/automated auditing support depending on configuration |
| ETW                   | Supported                                    | Not supported                                 | Supported where available                                      |
| Deployment complexity | Moderate                                     | High network complexity                       | Lower if MDE is already healthy                                |
| Detection coverage    | Strong classic coverage                      | Reduced coverage                              | Modern integrated coverage                                     |
| Best use case         | Direct monitoring of identity infrastructure | When direct installation is not allowed       | Modern Defender-integrated DC monitoring                       |
| Main risk             | Agent/resource concerns on Tier 0            | Incomplete mirroring and no ETW               | MDE dependency and migration assumptions                       |

The architecture decision should be made based on detection outcomes, not only installation convenience.

---

# 9. Full Tier 0 coverage is not optional

MDI’s value depends heavily on coverage.

At minimum, production designs should account for:

* all writable domain controllers,
* all read-only domain controllers,
* AD FS servers,
* AD CS servers,
* Microsoft Entra Connect servers,
* staging Entra Connect servers,
* and other identity infrastructure that participates in authentication or trust flows.

Why?

Because attackers do not care which DC your architecture diagram shows as “primary.”

They care which DC responds.

If one domain controller is unmonitored, that DC becomes a visibility gap. If AD CS is unmonitored, certificate abuse may be missed or poorly correlated. If Entra Connect is ignored, the bridge between on-premises identity and cloud identity remains under-observed.

## Common failure mode

A customer says:

> “We installed MDI on the main domain controllers.”

That sentence should immediately trigger follow-up questions:

* What does “main” mean?
* Are all sites covered?
* Are RODCs covered?
* Are child domains covered?
* Are legacy DCs still authenticating users?
* Are AD FS and AD CS monitored?
* Is Entra Connect monitored?
* Are staging sync servers included?
* Are decommissioned or forgotten DCs still online?

Partial coverage is one of the most dangerous MDI failure modes because it creates confidence without complete visibility.

## Field insight

In real assessments, I do not start by asking whether MDI is deployed.

I start by asking:

> “Which identity systems are not covered by MDI, and why?”

That question quickly reveals whether the customer has a deployment or a defensible architecture.

---

# 10. Domain controllers are not the whole identity plane

Many MDI discussions focus only on domain controllers.

That is understandable, but incomplete.

Modern hybrid identity attacks often target adjacent identity infrastructure:

## AD FS

AD FS can be involved in federation, token issuance, and authentication flows. If compromised, it may support attacks that bypass normal password-based detection logic.

Monitoring AD FS helps MDI provide better visibility into federation-related abuse.

## AD CS

AD CS is increasingly important because certificate-based attacks can provide powerful persistence and privilege escalation paths.

A weak certificate template, vulnerable enrollment path, or abnormal certificate request may become a domain compromise path.

Ignoring AD CS because “it is not a DC” is a mistake.

## Microsoft Entra Connect

Entra Connect is the synchronization bridge between on-premises AD and Microsoft Entra ID.

That makes it strategically sensitive.

If an attacker compromises sync infrastructure or manipulates synchronization-related accounts, they may be able to influence cloud identity outcomes from the on-premises environment.

For hybrid environments, Entra Connect should be treated as Tier 0 identity infrastructure.

## Consultant position

A strong MDI design does not only ask:

> “Are all DCs covered?”

It also asks:

> “Are all systems that can influence authentication, authorization, federation, certificates, synchronization, or privileged identity state covered?”

That is the difference between a product deployment and an identity security architecture.

---

# 11. Network Name Resolution: the quiet dependency behind investigation quality

Network Name Resolution, or NNR, is one of those topics that seems boring until an investigation depends on it.

MDI often sees activity involving IP addresses. To make that useful for analysts, it needs to correlate IPs to computer identities.

If NNR is unreliable, investigation quality drops.

You may see:

* source computers not resolved correctly,
* lower confidence in entity attribution,
* confusing alert evidence,
* increased false positives,
* incomplete lateral movement paths,
* and harder hunting pivots.

## What usually goes wrong

Common NNR issues include:

* blocked RPC-related ports,
* blocked NetBIOS name resolution,
* incomplete DNS records,
* NAT devices hiding source systems,
* VPN concentrators collapsing many users behind one source,
* stale computer objects,
* and segmented networks where sensors cannot query resolution sources.

In v2.x deployments, traditional NNR may require internal connectivity such as TCP 135, UDP 137, and related flows depending on the resolution method.

In v3.x deployments, MDE device inventory can improve this model, but only if MDE onboarding and device inventory are healthy.

## Field insight

Poor NNR does not always mean “no alert.”

It often means “alert with weaker evidence.”

That distinction matters.

A weakly attributed alert may still be technically correct, but it slows down the SOC. Analysts waste time asking basic questions:

* Which machine was this?
* Which user was really active?
* Is this a NAT issue?
* Is this a scanner?
* Is this source reliable?

Good identity detection is not only about firing alerts. It is about producing evidence that analysts can trust under pressure.

---

# 12. Directory Service Account design

MDI uses directory access to enrich identity activity with AD context.

This usually involves a Directory Service Account, often implemented as a gMSA.

The DSA is used for operations such as:

* querying directory metadata,
* enriching user and computer entities,
* reading deleted objects where configured,
* supporting SAM-R-related visibility,
* and building context used for lateral movement analysis.

## gMSA vs standard account

A gMSA is generally preferred because Active Directory manages password rotation automatically.

A standard user account may be simpler in some environments, especially where trusts or multi-forest scenarios complicate gMSA usage, but it introduces password lifecycle risk.

A standard DSA with an expired password can silently damage MDI’s ability to enrich and analyze activity.

## Common failure mode

The DSA exists, but it is not actually usable.

Typical causes:

* missing “Log on as a service” rights,
* gMSA host group membership not refreshed,
* server not rebooted after gMSA host group update,
* expired standard account password,
* missing DeletedObjects permissions,
* insufficient read permissions in monitored domains,
* performance counter registry permission issues,
* or account hardening policies that unintentionally block sensor impersonation.

The result is usually not a dramatic outage. Instead, MDI becomes less useful.

That is why consultants should validate DSA health, not just DSA existence.

## What to watch for

Do not use the same account for directory reads and response actions unless there is a very deliberate reason.

The DSA should be treated as a read-oriented identity enrichment account.

Response actions, such as disabling users or resetting passwords, require a different permission model and should be designed according to least privilege.

---

# 13. Action accounts and response permissions

MDI can support response actions through the Defender portal, such as disabling accounts, forcing password resets, revoking sessions, or marking identities as compromised depending on the environment and integration.

But response requires permissions.

This is where security architecture and operations meet.

A customer may have MDI deployed and still be unable to respond effectively because:

* the required action permissions were never delegated,
* the gMSA model does not match the sensor version,
* Entra roles are missing,
* the SOC has read-only Defender access,
* Unified RBAC is misconfigured,
* or the organization has not approved automated containment.

## Field insight

Detection without response design creates a false sense of readiness.

During an assessment, do not only ask:

> “Can MDI detect credential compromise?”

Ask:

> “When MDI or XDR identifies a compromised identity, who can disable the account, revoke sessions, force reset, and validate containment?”

If the answer is “we open a ticket to another team,” that is not necessarily wrong. But it must be understood as part of the response SLA.

---

# 14. Sizing and performance: authentication volume matters

MDI sensor sizing is driven by authentication and network traffic volume.

The important metric is **busy packets per second**, usually measured during the busiest 15-minute window over a 24-hour period.

This matters because domain controllers are already critical infrastructure. A security sensor must not destabilize authentication.

MDI includes safeguards, but capacity planning still matters.

## Practical sizing considerations

A consultant should review:

* number of domain controllers,
* authentication volume per site,
* peak logon periods,
* VPN authentication patterns,
* service account activity,
* legacy application authentication,
* DC CPU and memory,
* virtualization settings,
* and whether domain controllers are already overloaded.

## v2.x and v3.x resource behavior

Classic v2.x sensors reserve resources to protect the domain controller.

v3.x sensors use stricter caps, including CPU and memory limits, to reduce the risk of impacting domain controller operations.

That is good, but there is a trade-off.

If the sensor is resource-limited and the DC receives more traffic than the sensor can analyze, some traffic or events may not be processed. MDI may raise health issues such as network traffic or Windows events not being fully analyzed.

In other words:

> Resource protection prevents the sensor from hurting the DC, but it does not guarantee full analysis under overload.

## Common failure mode

Dynamic Memory is enabled on virtualized domain controllers.

This is a classic lab and production issue.

Domain controllers running MDI sensors should have statically allocated memory. Dynamic memory can create unpredictable sensor behavior and health problems, especially during authentication spikes.

## Field insight

A customer may say:

> “The sensor is installed and healthy.”

The better question is:

> “Is the sensor able to analyze peak authentication traffic without dropping traffic or events?”

Those are not the same thing.

---

# 15. Connectivity and proxy realities

MDI sensors communicate outbound to the cloud service over HTTPS.

That sounds simple until enterprise proxy, TLS inspection, and firewall controls enter the picture.

Common issues include:

* outbound HTTPS blocked,
* proxy authentication required,
* SSL inspection enabled,
* missing trusted root certificates,
* TLS registry settings not aligned,
* workspace API URL not reachable,
* or proxy 401/403 responses misread as licensing or connectivity problems.

## What usually goes wrong

Many organizations treat MDI connectivity as a generic “allow 443” requirement.

That is not enough.

MDI relies on secure communication with the cloud backend and certificate-based mutual authentication. SSL inspection can break this model.

Authenticated proxies may be supported, but credentials and proxy behavior must be configured correctly.

## Consultant callout

When a sensor shows cloud connectivity or licensing-looking errors, do not immediately assume licensing.

Check proxy behavior.

A proxy authentication failure or SSL inspection issue can look like something else in the portal, especially during early deployment.

---

# 16. Health issues are security signals too

MDI health issues are often treated as operational noise.

That is a mistake.

Health issues tell you whether the identity detection layer is trustworthy.

Examples include:

* sensor service stopped,
* cloud connectivity issues,
* missing audit policy,
* DSA failure,
* sensor resource constraints,
* traffic not analyzed,
* events not analyzed,
* NNR issues,
* unsupported OS,
* or missing configuration.

These are not just admin tasks.

They directly affect detection confidence.

## Field insight

In a real assessment, I treat MDI health issues as part of the security posture.

A critical identity alert from a healthy, well-covered environment is very different from an alert in an environment where half the DCs are unmonitored and several sensors cannot resolve source devices.

Before analyzing detections, validate health.

---

# 17. MDI and Defender XDR: why correlation is the point

MDI’s best value appears when its identity signals are correlated with other Defender XDR workloads.

For example:

* MDE sees suspicious process execution on a workstation.
* MDI sees that same identity perform LDAP reconnaissance.
* MDI later sees abnormal Kerberos ticket behavior.
* MDE sees remote execution on another device.
* Entra ID Protection sees risky cloud sign-in behavior.
* Defender for Cloud Apps sees unusual SaaS access.

Individually, each signal may be interesting.

Together, they become an attack story.

That is the Defender XDR value.

MDI contributes identity context such as:

* who the user is,
* whether the identity is sensitive,
* what devices were involved,
* which domain controllers observed the activity,
* what protocols were used,
* whether lateral movement paths exist,
* and whether the behavior matches known identity attack patterns.

This is why MDI alerts should not be consumed only as standalone notifications.

They should be investigated as part of the wider XDR incident graph.

## Connection point to Defender XDR

Part 3 of this series will go deeper into investigation workflows, but the architectural point belongs here:

> MDI is not successful because it creates many identity alerts.
> MDI is successful when identity signals improve incident correlation, investigation speed, and response confidence across Defender XDR.

---

# 18. What customers often misunderstand

## Misconception 1: “MDI is a full auditing tool”

MDI is not a general-purpose audit platform.

It does not capture every operation on every server. It captures the signals required for identity detections, posture recommendations, and investigation context.

If the customer needs full audit retention, compliance evidence, or long-term raw event storage, MDI should be integrated with Sentinel or another SIEM strategy.

## Misconception 2: “If MDI is installed, all AD attacks are visible”

No.

MDI visibility depends on:

* sensor coverage,
* supported server versions,
* event collection,
* ETW availability,
* NNR health,
* directory enrichment,
* protocol visibility,
* baselines,
* and tuning decisions.

An installed sensor is not the same as complete detection coverage.

## Misconception 3: “Standalone sensors provide the same coverage”

They do not.

Standalone sensors can be useful, but they do not support ETW and depend heavily on correct network mirroring.

That means reduced detection coverage and higher operational complexity.

## Misconception 4: “MDI only matters for synced Entra ID users”

MDI provides value for on-premises Active Directory accounts even if they are not synchronized to Entra ID.

That matters in hybrid environments where service accounts, legacy identities, privileged admin accounts, or infrastructure accounts may remain on-premises only.

## Misconception 5: “The SOC owns MDI”

Partially.

The SOC consumes MDI alerts and investigation data, but identity engineers own many prerequisites:

* DC coverage,
* audit policy,
* DSA health,
* gMSA configuration,
* AD CS/AD FS/Entra Connect coverage,
* NNR dependencies,
* and Tier 0 design.

Security architects own the bigger questions:

* identity tiering,
* response model,
* posture management,
* Zero Trust alignment,
* and XDR integration.

MDI fails when everyone assumes another team owns the hard parts.

---

# 19. A consultant readiness checklist

When assessing an MDI deployment, I use a readiness model instead of a simple deployment checklist.

The goal is to answer one question:

> Can this environment produce reliable identity security signals for Defender XDR?

## 19.1 Coverage readiness

Validate:

* all writable DCs are monitored,
* all RODCs are monitored,
* all AD FS servers are monitored,
* all AD CS servers are monitored,
* all Entra Connect servers are monitored,
* staging sync servers are included,
* legacy DCs are identified,
* multi-forest environments are accounted for,
* Tier 0 systems are included in the design.

## 19.2 Sensor health readiness

Validate:

* sensors are connected,
* sensors are updated,
* no critical health issues exist,
* OS versions are supported,
* sensor services are stable,
* CPU and memory are sufficient,
* no traffic or event analysis drops are occurring,
* virtualized DCs use static memory.

## 19.3 Signal readiness

Validate:

* Kerberos visibility,
* NTLM visibility,
* LDAP visibility,
* DNS visibility,
* SAM-R visibility,
* Windows events,
* ETW availability,
* AD FS auditing,
* AD CS auditing,
* Entra Connect visibility.

## 19.4 Directory enrichment readiness

Validate:

* DSA/gMSA configured correctly,
* DSA password lifecycle handled,
* gMSA host group membership effective,
* DeletedObjects read access configured if required,
* cross-domain and cross-forest permissions understood,
* LocalSystem behavior understood for v3.x,
* response-action identity model documented.

## 19.5 NNR readiness

Validate:

* source IPs resolve correctly,
* required internal ports are not blocked,
* DNS records are accurate,
* NAT/VPN impact is understood,
* MDE device inventory is healthy for v3.x scenarios,
* entity attribution is reliable in alert evidence.

## 19.6 XDR readiness

Validate:

* MDI alerts appear in Defender XDR incidents,
* analysts can pivot to identity pages,
* Identity Timeline is useful,
* Alert Story and Alert Graph are populated,
* Advanced Hunting tables contain identity data,
* SOC roles have appropriate permissions,
* response actions are tested,
* Action Center is monitored.

## 19.7 Posture readiness

Validate:

* sensitive accounts are tagged,
* honeytokens are configured where appropriate,
* lateral movement paths are reviewed,
* Secure Score identity recommendations are assigned,
* risky delegation is tracked,
* Tier 0 local admin exposure is reviewed,
* privileged group changes are monitored,
* identity risk context is reviewed.

---

# 20. What good looks like

A good MDI deployment is not defined by “sensors installed.”

A good MDI deployment has:

* complete identity infrastructure coverage,
* healthy sensors,
* reliable directory enrichment,
* stable network and cloud connectivity,
* correct audit policy,
* usable NNR,
* mature baselines,
* well-defined sensitive identities,
* documented sensor architecture trade-offs,
* tested response actions,
* SOC workflows connected to Defender XDR incidents,
* and ownership across identity, SOC, endpoint, and architecture teams.

That is the difference between deploying MDI and operationalizing MDI.

---

# 21. Final consultant framing

When explaining MDI to a customer, I would avoid saying:

> “MDI detects attacks against Active Directory.”

That is too small.

A better framing is:

> “Microsoft Defender for Identity is the on-premises and hybrid identity signal engine inside Defender XDR. It turns Active Directory authentication, protocol, directory, and identity infrastructure activity into security signals that Defender XDR can correlate with endpoint, email, cloud app, and Entra ID telemetry. Its value depends on complete Tier 0 coverage, healthy sensors, mature baselines, reliable entity attribution, and operational response design.”

That framing sets the right expectations.

MDI is not magic. It cannot see what is not monitored. It cannot fix broken identity architecture. It cannot compensate for poor Tier 0 hygiene. It cannot make partial telemetry complete.

But when deployed and operated correctly, it gives Defender XDR something extremely valuable:

identity context at the point where many attacks become enterprise-impacting.

That is why MDI deserves to be treated as a pillar of Microsoft Defender XDR architecture, not just another security product installed on domain controllers.

---

# Coming next

In Part 2, we will move from architecture to detection logic.

We will look at how MDI detects identity attacks across:

* reconnaissance,
* credential access,
* lateral movement,
* privilege escalation,
* persistence,
* and domain dominance.

More importantly, we will separate deterministic detections from behavioral detections, explain why baseline maturity matters, and discuss what MDI cannot see or guarantee.

Because in identity detection, understanding why an alert fires is just as important as knowing that it fired.
