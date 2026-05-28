---
title: "Microsoft Defender for Identity Deep Dive — Part 2: Detection Logic, Baselines, and Limits"
date: 2026-04-08
categories: [microsoft-defender-xdr, identity-security]
tags: [mdi, defender-xdr, active-directory, kerberos, ntlm, ldap, detection-engineering, soc, baselining]
toc: true
---

# Microsoft Defender for Identity Deep Dive — Part 2: Detection Logic, Baselines, and Limits

In Part 1, we looked at Microsoft Defender for Identity from an architectural perspective.

The main point was simple:

> MDI is not just an AD alerting tool.  
> It is the on-premises and hybrid identity signal engine inside Microsoft Defender XDR.

That distinction matters even more when we start talking about detections.

A lot of people evaluate MDI the wrong way.

They run one command, wait for an alert, and then conclude that MDI either “works” or “doesn’t work.”

That is not how identity detection should be assessed.

MDI detections are built from a mix of:

- protocol analysis,
- Windows events,
- ETW signals,
- directory enrichment,
- behavioral baselines,
- deterministic attack-pattern matching,
- anomaly detection,
- threat intelligence,
- and Defender XDR correlation.

Some alerts can fire quickly because they match known malicious patterns.

Others need time because MDI first has to learn what “normal” looks like in that specific Active Directory environment.

That is why early testing often creates confusion.

A consultant may simulate reconnaissance, Kerberoasting, Pass-the-Hash, or suspicious admin behavior and expect immediate results. Sometimes they get alerts. Sometimes they get partial evidence. Sometimes they get nothing useful.

The reason is rarely “MDI is broken.”

More often, the reason is one of these:

- the environment is still in a learning period,
- the sensor does not have full visibility,
- required Windows events are missing,
- the activity was too isolated,
- the source device was poorly resolved,
- the account or tool already looks normal,
- thresholds are too conservative for testing,
- or the detection depends on a later phase of the attack chain.

This post is about understanding those details.

Not just what MDI detects, but **why** it detects it, **when** it detects it, and **where** its confidence breaks.

---

# 1. Identity detection is different from endpoint detection

Endpoint detections often start with process behavior.

An endpoint product may see:

- suspicious PowerShell,
- credential dumping,
- process injection,
- malicious DLL loading,
- encoded command execution,
- unusual child processes,
- LSASS access,
- or known malware behavior.

MDI sees the world differently.

MDI does not primarily think in terms of process trees. It thinks in terms of identity infrastructure behavior:

- Who authenticated?
- Which protocol was used?
- Which domain controller observed it?
- Which account requested a Kerberos ticket?
- Which computer performed LDAP enumeration?
- Which identity asked for replication data?
- Which account touched a sensitive group?
- Which device queried privileged group membership?
- Which source reused a credential in an abnormal way?
- Which identity path could lead to a sensitive target?

That is why MDI alerts may feel different from endpoint alerts.

An endpoint alert often says:

> “This process did something suspicious on this machine.”

An MDI alert often says:

> “This identity or device behaved abnormally against the identity plane.”

That difference changes how we investigate.

You cannot evaluate MDI purely by asking, “Did it detect the tool?”

You need to ask:

- Did MDI see the protocol activity?
- Did the activity reach a monitored identity server?
- Did MDI have the required event context?
- Did MDI understand the source device?
- Was the behavior abnormal for that identity or computer?
- Was the technique deterministic enough to alert without a baseline?
- Did Defender XDR correlate it with endpoint or cloud evidence?

This is the mindset shift.

MDI is not a tool detector.

MDI is an identity behavior and protocol analytics engine.

## Customer misconception: “Is this user behavior monitoring?”

No.

MDI is not designed to monitor employee productivity, browsing behavior, screen activity, file usage, or general “what did this person do all day?” activity.

It evaluates identity misuse and identity-plane behavior.

That means it looks at signals such as:

- authentication attempts,
- Kerberos and NTLM behavior,
- LDAP and SAM-R queries,
- directory object changes,
- replication behavior,
- privilege relationships,
- and suspicious interactions with identity infrastructure.

A user account may appear in an MDI alert, but the alert is about the account’s identity behavior, not personal surveillance.

A useful workshop phrase is:

> MDI is account-aware and behavior-aware, but it is not employee monitoring.

This distinction matters for SOC trust and customer communication. If MDI is framed as a surveillance tool, business stakeholders misunderstand its purpose. If it is framed correctly, it becomes what it actually is: an identity security signal source.

---

# 2. The detection model: patterns, behavior, and context

MDI detections generally come from three broad logic families.

## 2.1 Deterministic attack-pattern detections

Some behaviors are suspicious because they match known attack patterns.

Examples include:

- abnormal directory replication requests,
- known Kerberos ticket anomalies,
- suspicious protocol implementations,
- certain domain dominance behaviors,
- known malicious authentication patterns,
- or activity strongly associated with offensive tooling.

These detections do not always require a long learning period because the behavior itself is suspicious.

A simple example is DCSync-style behavior.

Domain controllers legitimately replicate directory data. A normal workstation should not behave like a replication partner. If a non-DC requests replication data in a suspicious way, that is not just “unusual.” It maps directly to a known identity attack technique.

That kind of detection can be more deterministic.

## 2.2 Behavioral baseline detections

Other behaviors are suspicious only when they are abnormal for the environment.

Examples include:

- unusual LDAP query volume,
- abnormal SAM-R reconnaissance,
- unexpected DNS mapping,
- suspicious authentication patterns,
- unfamiliar source-to-target relationships,
- unusual administrative behavior,
- abnormal VPN patterns,
- or identity activity outside a learned norm.

This requires historical context.

A query that is suspicious from a normal workstation may be normal from a management server. A large number of LDAP queries may be normal from an identity governance tool but suspicious from a user laptop.

This is why MDI needs baselines.

It is not enough to know that something happened. MDI needs to know whether it is normal for that source, identity, protocol, and environment.

## 2.3 Contextual and XDR-enriched detections

Some signals become more meaningful when correlated with other security data.

For example:

- an endpoint compromise from MDE,
- followed by LDAP reconnaissance in MDI,
- followed by lateral movement,
- followed by risky cloud activity,
- followed by Defender for Cloud Apps detecting unusual downloads.

Individually, each event may not tell the full story.

Together, they form an attack chain.

This is where Defender XDR matters.

MDI provides identity context, but XDR correlation gives that context a broader storyline.

---

# 3. Detection families across the identity attack lifecycle

MDI detections are usually easier to understand when mapped to the identity attack lifecycle.

The core families are:

1. reconnaissance,
2. credential access,
3. lateral movement,
4. privilege escalation,
5. persistence and domain dominance,
6. suspicious modifications.

Let’s walk through each one from a practitioner perspective.

---

# 4. Reconnaissance detections

Reconnaissance is where many identity attacks start.

Before attackers can move effectively, they need to understand the environment:

- valid users,
- privileged groups,
- domain controllers,
- service accounts,
- SPNs,
- admin relationships,
- DNS structure,
- reachable systems,
- and sensitive identities.

In Active Directory, reconnaissance is not always loud in the traditional sense.

Attackers often use legitimate protocols.

That is exactly why MDI matters.

A firewall may not care that a workstation queried LDAP. A SIEM may ingest the event but not understand the query pattern. MDI is built to understand whether that identity activity looks like discovery, enumeration, or preparation for later abuse.

## 4.1 LDAP reconnaissance

LDAP is one of the most important protocols for AD reconnaissance.

Attackers use LDAP queries to discover:

- users,
- groups,
- privileged accounts,
- service accounts,
- SPNs,
- delegation settings,
- domain policies,
- and interesting object attributes.

MDI can detect suspicious LDAP behavior when the query shape, volume, source, or target context deviates from normal behavior.

### Why this detection works

LDAP reconnaissance is useful to attackers because AD itself is rich with attack path information.

The attacker does not need to exploit anything to ask questions.

They can query the directory and learn:

- who is privileged,
- which users have SPNs,
- which accounts are sensitive,
- where delegation exists,
- which computers are servers,
- which accounts may be stale,
- and which paths might support escalation.

MDI looks at the way these questions are asked.

A normal business application may query AD in a predictable way. A domain admin workstation may perform broader queries. A random user laptop suddenly asking broad security-principal questions is different.

That difference is the detection opportunity.

### Field insight

A common customer misunderstanding is expecting every LDAP query to be suspicious.

That is not realistic.

LDAP is normal in AD environments.

The detection value comes from identifying LDAP behavior that looks like reconnaissance based on source, volume, target, query type, and baseline.

---

## 4.2 SAM-R reconnaissance

SAM-R can be used to enumerate users, groups, and membership information.

This matters because attackers often need to answer practical questions:

- Who is in Domain Admins?
- Which users are local admins?
- Which groups contain privileged accounts?
- Which accounts are worth attacking next?

MDI can detect suspicious user and group membership reconnaissance over SAM-R.

### Why this detection works

SAM-R enumeration often has a recognizable pattern.

The attacker is not performing a normal business login. They are asking the domain about users and group relationships.

That behavior can be legitimate from administrative tools, vulnerability scanners, or management systems. But from an unusual workstation or user context, it becomes suspicious.

### Common failure mode

SAM-R detections can require longer baseline maturity because legitimate administrative tools may use the same protocol.

This is where customers get frustrated.

They simulate SAM-R enumeration on day one and ask why the alert did not behave as expected.

The answer is usually baseline maturity.

MDI needs to learn which systems normally perform SAM-R-style activity before it can confidently separate normal administration from suspicious enumeration.

---

## 4.3 DNS network mapping

Attackers may use DNS to map internal infrastructure.

Suspicious DNS behavior can include:

- unusual query volume,
- zone transfer attempts,
- broad internal name discovery,
- or network mapping behavior from non-DNS systems.

### Why this detection works

In healthy environments, DNS behavior is usually predictable.

Most client systems perform routine name resolution. They should not usually perform DNS zone transfers or broad mapping behavior.

When a workstation behaves like it is mapping the environment, that stands out.

### Field insight

DNS reconnaissance detections are often easier for customers to understand because the behavior is conceptually simple:

> “Why is this workstation trying to discover the internal namespace?”

But false positives still happen, especially with scanners, inventory tools, monitoring systems, and misconfigured scripts.

---

## 4.4 Honeytoken queries

Honeytokens are decoy accounts or objects that should not be touched during normal operations.

MDI can alert when honeytoken accounts are queried or used.

### Why this detection works

Honeytokens are high-signal because the expected baseline is near zero.

A normal user should not authenticate as a honeytoken account.

A normal system should not query it.

A normal administrator should not interact with it casually.

That gives MDI a strong detection anchor.

### Common failure mode

Honeytokens are only useful if they are designed and documented properly.

Bad honeytokens create noise.

Examples:

- the account name is too obvious,
- the account is accidentally included in scripts,
- legacy systems still query it,
- administrators test with it,
- it is placed in a group where tools enumerate it constantly,
- or nobody remembers why it exists.

A honeytoken is not just a checkbox.

It is a detection asset.

---

# 5. Credential access detections

Credential access is where identity attacks become dangerous.

Once an attacker obtains reusable credentials, hashes, tickets, or replication access, they can often move with less noise than malware would create.

MDI has strong value here because many credential attacks must interact with domain controllers, Kerberos, NTLM, or directory services.

---

## 5.1 Brute force and password spray

MDI can detect suspicious authentication failures using Kerberos or NTLM.

A brute-force attack usually targets one account with many password attempts.

A password spray usually targets many accounts with one or a few common passwords.

### Why this detection works

Authentication failures are normal.

But patterns matter.

MDI can reason about:

- number of failures,
- number of targeted accounts,
- source device,
- authentication protocol,
- timing,
- successful login after failures,
- and whether the behavior matches the environment baseline.

### Field insight

Password spray is often more dangerous than brute force because it is designed to avoid lockouts.

A customer may say:

> “Our lockout policy protects us.”

That only helps if the attacker triggers lockouts.

Password spraying is specifically designed to stay below that threshold.

MDI helps because it can detect the broader pattern across accounts and authentication attempts.

---

## 5.2 AS-REP Roasting

AS-REP Roasting targets accounts that do not require Kerberos pre-authentication.

An attacker can request authentication data for those accounts and attempt offline cracking.

### Why this detection works

The weakness exists because of account configuration.

The suspicious behavior is the request pattern against accounts that allow this exposure.

MDI can identify activity that suggests an attacker is attempting to collect crackable material.

### Consultant callout

This is a good example of detection meeting posture.

If MDI detects AS-REP Roasting, the immediate investigation matters, but the long-term fix is account hygiene:

- identify accounts without pre-authentication,
- validate whether the setting is justified,
- rotate exposed credentials,
- and remove the configuration where possible.

---

## 5.3 Kerberoasting

Kerberoasting targets service accounts with SPNs.

The attacker requests service tickets and attempts offline cracking of the ticket material.

### Why this detection works

Kerberoasting has two important phases:

1. identifying roastable accounts,
2. requesting tickets.

MDI can see suspicious LDAP searches for SPNs and suspicious Kerberos ticket activity.

The detection is not simply “someone requested a ticket.”

Service ticket requests are normal in Kerberos.

The suspicious part is the pattern:

- who requested it,
- from where,
- for which service accounts,
- how many,
- whether the request pattern is abnormal,
- and whether it follows reconnaissance behavior.

### Common failure mode

Customers sometimes test Kerberoasting with one service ticket request and expect a full alert.

But real Kerberoasting detections often depend on pattern, context, and baseline.

A single request may not be enough.

A broader sequence involving SPN discovery, unusual ticket requests, and suspicious source behavior is more likely to produce meaningful detection.

---

## 5.4 DPAPI backup master key request

DPAPI backup master key abuse can allow attackers to decrypt protected secrets.

### Why this detection works

This is not normal behavior for standard users or workstations.

When an identity attempts to retrieve sensitive cryptographic material from a domain controller, that is highly suspicious.

### Field insight

This kind of alert should be treated seriously because it often indicates the attacker is not just guessing passwords. They are attempting to access material that can unlock stored secrets.

---

## 5.5 DCSync

DCSync is one of the most important identity compromise detections.

It abuses directory replication permissions to request password data from a domain controller.

### Why this detection works

Domain controllers replicate with each other.

Normal workstations do not.

A non-DC requesting replication data is a strong signal of domain compromise or serious privilege abuse.

DCSync usually means the attacker has already obtained high-level privileges or compromised an identity with replication rights.

### Consultant callout

DCSync should not be treated as “just another alert.”

If confirmed, assume the attacker may have accessed password hashes for sensitive accounts.

The response should include:

- identifying the source identity,
- validating replication permissions,
- reviewing recent privileged activity,
- checking for persistence,
- rotating exposed credentials,
- and considering KRBTGT implications depending on scope.

---

# 6. Lateral movement detections

Lateral movement is where endpoint and identity signals begin to overlap heavily.

An attacker may compromise one machine, steal credentials, and then use those credentials to access another system.

MDI becomes valuable because lateral movement often touches authentication infrastructure even when the endpoint artifact is weak or missing.

---

## 6.1 Pass-the-Hash

Pass-the-Hash abuses NTLM hashes instead of plaintext passwords.

### Why this detection works

MDI can detect suspicious reuse of NTLM authentication material across systems.

The suspicious part is not simply “NTLM happened.”

NTLM still exists in many environments.

The suspicious part is the pattern of authentication:

- source,
- target,
- account,
- timing,
- protocol behavior,
- and whether the activity matches normal use.

### Common failure mode

If NTLM is widely used everywhere, detection confidence becomes harder.

A flat, legacy-heavy environment produces more noise than a well-segmented environment with limited NTLM usage.

That is not an MDI problem only.

That is an identity architecture problem.

---

## 6.2 Pass-the-Ticket

Pass-the-Ticket abuses Kerberos tickets.

### Why this detection works

Kerberos tickets have structure, timing, encryption, and usage patterns.

MDI can detect suspicious ticket usage when the behavior deviates from expected Kerberos flows or known legitimate patterns.

### Field insight

Pass-the-Ticket detections can be affected by legitimate security features or administrative patterns.

For example, Remote Credential Guard over RDP may produce behavior that resembles ticket-related abuse in some scenarios.

This does not mean the detection is bad.

It means the SOC needs context.

---

## 6.3 Overpass-the-Hash

Overpass-the-Hash uses an NTLM hash to obtain a Kerberos ticket.

### Why this detection works

This bridges NTLM credential material into Kerberos authentication.

That transition can create suspicious Kerberos behavior, especially if it does not match normal Windows authentication flows.

### Important limitation

Some Overpass-the-Hash detection scenarios are affected by Kerberos Armoring.

This is exactly the kind of limitation consultants should mention early.

Not because MDI is weak, but because credible security architecture requires knowing where detection coverage changes.

---

## 6.4 NTLM relay

NTLM relay abuses captured or coerced authentication and relays it to another service.

### Why this detection works

MDI can observe suspicious NTLM authentication flows and abnormal relationships between source, target, and identity.

NTLM relay is especially dangerous in environments where legacy authentication is still heavily used and signing/channel binding protections are incomplete.

### Consultant callout

MDI detection should not be the only control for NTLM relay.

The long-term fix is hardening:

- reduce NTLM,
- enforce LDAP signing and channel binding where appropriate,
- harden AD CS web enrollment exposure,
- review SMB signing,
- reduce coercion paths,
- and monitor privileged authentication.

Detection is not a substitute for protocol hardening.

---

## 6.5 Remote execution attempts

MDI can detect suspicious remote execution attempts against sensitive systems, including behavior associated with:

- PSExec,
- WMI,
- PowerShell remoting,
- or suspicious service creation.

### Why this detection works

Remote execution against identity infrastructure is not normal from random endpoints.

Even when the mechanism is legitimate, the context may be suspicious:

- unusual source,
- sensitive target,
- abnormal user,
- unexpected service creation,
- or activity following credential access.

### Field insight

This is where XDR correlation is important.

MDI may show identity-side evidence of remote access or execution attempts, while MDE may show the actual process, command line, service creation, or script behavior on the endpoint.

Separately, each signal is useful.

Together, they are much stronger.

---

# 7. Privilege escalation detections

Privilege escalation in AD often involves abusing configuration, protocols, or privileged relationships.

MDI can detect several escalation-related behaviors, especially when they touch domain controllers or sensitive directory objects.

---

## 7.1 Netlogon privilege elevation

Netlogon abuse, such as ZeroLogon-style behavior, targets the secure channel relationship with a domain controller.

### Why this detection works

The behavior involves abnormal Netlogon protocol activity against a domain controller.

That kind of protocol abuse has recognizable characteristics.

### Consultant callout

If a customer sees this type of alert, patch status and DC exposure matter immediately.

The investigation should not only focus on the source machine. It should validate whether domain controller hardening and patching are current.

---

## 7.2 SID-History injection

SID-History can be abused to grant unauthorized access by adding powerful SIDs to an account.

### Why this detection works

SID-History is sensitive because it affects authorization.

A suspicious change to SID-History may allow an attacker to bypass normal group membership review and gain access as if they had another identity’s privileges.

### Field insight

SID-History is one of those features that exists for legitimate migration reasons but becomes dangerous when misused.

A consultant should ask:

- Is SID-History still required?
- Which accounts have it?
- Who can modify it?
- Are changes monitored?
- Is it expected in this domain?

---

## 7.3 AdminSDHolder modification

AdminSDHolder protects privileged accounts by applying security descriptors.

If an attacker modifies AdminSDHolder, they may create persistent control over privileged objects.

### Why this detection works

AdminSDHolder is a sensitive object.

Changes to it are rare and highly impactful.

MDI can identify suspicious modifications because the object has special meaning in AD privilege management.

### Consultant callout

AdminSDHolder modification is not just privilege escalation.

It can be persistence.

If confirmed, investigate privileged object permissions broadly, not only the account that triggered the alert.

---

## 7.4 AD CS abuse

AD CS abuse can allow certificate-based privilege escalation or persistence.

MDI can detect suspicious certificate-related behavior when sensors and auditing are correctly configured on AD CS servers.

### Why this detection works

Certificate Services can issue credentials that are trusted by the domain.

That means a misconfigured certificate template or abnormal certificate request can become equivalent to credential compromise.

### Common failure mode

Customers forget AD CS.

They deploy MDI on DCs and assume the identity plane is covered.

But if AD CS is issuing certificates that can authenticate to AD, then AD CS is identity infrastructure.

Unmonitored AD CS is a visibility gap.

---

# 8. Persistence and domain dominance detections

Persistence and domain dominance alerts are among the highest-impact MDI detections.

These are not usually early-stage signals.

They often indicate that the attacker has already achieved significant control.

---

## 8.1 Golden Ticket

A Golden Ticket attack involves forged Kerberos Ticket Granting Tickets.

This usually implies compromise of the KRBTGT account material.

### Why this detection works

Kerberos tickets have expected structures, lifetimes, encryption behavior, and validation patterns.

Forged tickets may contain anomalies that differ from normal domain-issued tickets.

MDI can detect suspicious ticket behavior that suggests Golden Ticket use.

### Consultant callout

Golden Ticket detection is not a “reset the user password and move on” situation.

If confirmed, the organization must consider KRBTGT compromise response, including the supported double KRBTGT password reset process with appropriate timing.

Also investigate how the attacker got enough privilege to access KRBTGT material in the first place.

---

## 8.2 DCShadow

DCShadow attempts to register or behave like a rogue domain controller to push malicious directory changes.

### Why this detection works

Domain controller registration and replication behavior are highly sensitive and should be predictable.

A non-standard source attempting domain-controller-like replication behavior is suspicious.

### Field insight

DCShadow is not just an event.

It is a statement that the attacker understands AD internals.

If this appears in production, assume the attacker is advanced enough to manipulate directory state intentionally.

---

## 8.3 Suspicious sensitive group additions

Adding a user to a privileged group is one of the simplest ways to gain persistent access.

MDI can detect suspicious additions to sensitive groups.

### Why this detection works

Privileged group membership changes are security-critical.

The detection logic can use:

- group sensitivity,
- actor identity,
- source system,
- timing,
- baseline behavior,
- and whether the change fits normal administrative patterns.

### Common failure mode

Some environments change privileged groups frequently.

That is bad for both security and detection.

If Domain Admins, Enterprise Admins, or other sensitive groups are modified casually, MDI may still detect suspicious changes, but the SOC will struggle to separate emergency admin work from malicious privilege escalation.

Operational discipline improves detection quality.

---

## 8.4 Skeleton Key

Skeleton Key malware allows attackers to authenticate as any user while keeping legitimate passwords working.

### Why this detection works

Skeleton Key behavior can create encryption or authentication anomalies on domain controllers.

MDI can detect suspicious patterns associated with this kind of domain controller compromise.

### Consultant callout

This is a domain controller compromise scenario.

Treat it as Tier 0 incident response, not normal user compromise.

---

## 8.5 Suspicious service creation

A newly created service on a domain controller can indicate persistence or remote execution.

### Why this detection works

Service creation on a domain controller is sensitive.

A service may be used to execute attacker-controlled code, maintain access, or run commands remotely.

MDI can use Windows event context and identity infrastructure telemetry to detect suspicious service creation.

### Field insight

This is another place where MDE and MDI should be combined.

MDI may explain the identity context.

MDE may explain the process context.

Defender XDR should connect both.

---

# 9. Suspicious modifications

Suspicious modifications are not always flashy, but they are often extremely important.

Attackers may not need malware if they can modify identity infrastructure.

Examples include:

- sensitive attribute changes,
- AD FS trust changes,
- AD CS permission changes,
- Group Policy changes,
- account control flag changes,
- delegation changes,
- password reset abuse,
- object permission changes,
- and modifications to synchronization-related identities.

### Why this detection works

Active Directory security is not only about authentication.

It is also about state.

If the attacker changes the right object, permission, attribute, template, or trust setting, they can create access that persists beyond the original compromise.

MDI helps detect modifications that affect identity trust, privilege, authentication, or persistence.

### Consultant callout

In real environments, suspicious modifications are often dismissed as “admin activity.”

That is dangerous.

The right question is not:

> “Was this done by an admin?”

The right question is:

> “Was this expected, approved, and consistent with normal administrative behavior?”

Compromised admins still look like admins.

That is the core problem in identity security.

---

# 10. Deterministic detections vs behavioral baselines

This is one of the most important sections in the whole series.

Not all MDI detections behave the same way.

Some detections are based on known malicious patterns.

Others depend on baselines.

If you do not understand this distinction, you will misunderstand MDI during testing and production operations.

## 10.1 Deterministic detections

Deterministic detections are based on behavior that is suspicious because of what it is.

Examples may include:

- known malicious protocol patterns,
- abnormal replication requests,
- known ticket anomalies,
- domain dominance behavior,
- or activity strongly mapped to known attack techniques.

These detections may fire quickly because they do not need weeks of historical behavior to understand that something is wrong.

## 10.2 Behavioral detections

Behavioral detections ask a different question:

> “Is this normal for this identity, device, domain controller, protocol, or environment?”

That requires learning.

For example:

- LDAP queries may be normal from one system and suspicious from another.
- SAM-R enumeration may be normal from a management server and suspicious from a laptop.
- DNS query volume may be normal for a DNS server and suspicious for a user endpoint.
- Administrative actions may be normal from a PAW and suspicious from a workstation.

This is why MDI needs baselines.

## Field insight

A lot of bad MDI testing happens because people test behavioral detections as if they were deterministic detections.

They simulate one suspicious action in a fresh lab and expect production-grade alerting.

That is not a fair test.

A better test plan separates:

- detections expected to trigger immediately,
- detections requiring baseline maturity,
- detections requiring multi-step behavior,
- detections requiring specific audit events,
- detections requiring complete sensor coverage,
- and detections that need XDR correlation for full story value.

---

# 11. Baseline maturity timelines

MDI learning periods vary depending on the detection.

A practical reference model:

| Detection or capability | Typical learning requirement |
|---|---|
| Known malicious patterns | Immediate |
| General behavioral detections | Around three weeks |
| Full organizational baseline | Up to 30 days per domain controller |
| DNS network mapping | Around 8 days |
| LDAP security principal reconnaissance | Around 15 days per computer |
| SAM-R user/group reconnaissance | Around 4 weeks per domain controller |
| Suspicious VPN connection | Around 30 days from first VPN connection and at least 5 connections |
| Brute force | Around 1 week |

Do not treat these as marketing numbers.

Treat them as operational planning inputs.

If you deploy MDI on Monday and run a workshop on Tuesday, you should not expect every behavioral alert to behave like a mature production tenant.

## Common failure mode

A customer enables MDI, immediately runs a simulation, sees inconsistent results, and says:

> “MDI missed the attack.”

Maybe.

But maybe not.

The right follow-up questions are:

- Was the detection deterministic or behavioral?
- Was the learning period complete?
- Were thresholds changed?
- Was Recommended Test Mode enabled?
- Was the activity realistic or single-step?
- Did the activity hit a monitored DC?
- Were required events available?
- Was the source device resolved correctly?
- Was the account already known for similar behavior?
- Was the sensor healthy at the time?

Without answering those questions, the conclusion is premature.

---

# 12. Recommended Test Mode

Recommended Test Mode is useful, but often misunderstood.

It lowers alert thresholds to increase visibility during evaluation.

It is especially helpful for:

- proof-of-concept deployments,
- lab environments,
- red team exercises,
- workshops,
- early validation,
- and controlled detection testing.

But it does not magically replace baseline maturity.

## What it changes

Recommended Test Mode sets alert thresholds to Low for a limited period, usually up to 60 days.

This makes MDI more sensitive and allows more detections to trigger earlier.

## What it does not change

It does not make incomplete telemetry complete.

It does not fix:

- missing sensors,
- broken NNR,
- missing audit events,
- unmonitored DCs,
- standalone sensor ETW limitations,
- DSA failures,
- or unrealistic single-step testing.

It also does not mean every alert is equally high confidence.

## Field insight

Recommended Test Mode is good for learning what MDI can surface.

It is not a production tuning strategy.

Leaving production environments at overly sensitive thresholds can create alert fatigue and reduce SOC trust.

---

# 13. Thresholds: High, Medium, and Low

MDI supports alert threshold adjustment.

The practical model is:

| Threshold | Intended use |
|---|---|
| High | Default production mode; reduces false positives |
| Medium | More sensitive; may surface more legitimate activity |
| Low | High-volume testing mode; useful for validation but noisy in production |

## High threshold

High is the default production posture.

It favors precision.

This is usually where mature environments should operate unless there is a specific reason to change it.

## Medium threshold

Medium can be useful when:

- the environment has high risk,
- the SOC can handle more alerts,
- a specific detection area needs more sensitivity,
- or a controlled monitoring period is underway.

But it should be intentional.

## Low threshold

Low is mainly for testing.

It can help during:

- workshops,
- lab simulations,
- internal red team exercises,
- PoC validation,
- or early detection coverage reviews.

But low thresholds in production can create excessive noise.

## Consultant callout

Do not confuse sensitivity with maturity.

A noisy environment is not a mature environment.

More alerts do not automatically mean better security.

The goal is not maximum alert volume.

The goal is reliable, explainable identity detection that analysts trust.

---

# 14. Tuning philosophy: prefer precision over silence

MDI supports tuning and exclusions.

These are not the same thing.

## 14.1 Alert tuning

Alert tuning is usually the preferred approach.

It allows the SOC to reduce noise using evidence-based conditions.

For example:

- a known scanner triggers a specific reconnaissance alert,
- a management server performs expected enumeration,
- a service account has a known behavior pattern,
- or a specific source-target relationship is understood.

Good tuning narrows the condition.

It does not blindly remove visibility.

## 14.2 Exclusions

Exclusions are more dangerous.

If you exclude an entity, IP, device, or domain too broadly, you may create a blind spot.

That blind spot becomes dangerous if an attacker compromises the excluded asset.

## 14.3 Global exclusions

Global exclusions are the highest-risk option.

They should be rare and heavily justified.

Avoid excluding:

- honeytoken accounts,
- domain controllers,
- sensitive administrative accounts,
- NAT or VPN devices,
- cross-domain indicators used for XDR correlation,
- PAWs,
- Entra Connect servers,
- AD FS servers,
- AD CS servers.

## Consultant warning: exclusions are risk decisions, not tuning shortcuts

A common workshop question is:

> “When should we configure exclusions in the MDI portal?”

The safest answer is:

> Only after you understand the alert, validate the benign pattern, and confirm that narrower tuning will not solve the problem.

Exclusions should be treated as last-resort risk decisions, not as the default way to make the portal quieter.

Before creating an exclusion, ask:

- What exact alert or behavior are we trying to suppress?
- Is the behavior truly benign, or only familiar?
- Is this source ever used for privileged access?
- Could an attacker compromise this excluded device or account?
- Will the exclusion hide future reconnaissance, credential access, or lateral movement?
- Can we tune by condition, scope, maintenance window, source, or known tool instead?
- Who owns the exclusion?
- When will it be reviewed?

The point is not “never use exclusions.”

The point is:

> Exclusions should reduce known noise without removing visibility into meaningful identity abuse.

If an exclusion hides a real attack path, the environment becomes quieter but less secure.

## Customer misconception: “Can exclusions be used to reduce noise safely?”

Sometimes, but only with strict scope and ownership.

A safe exclusion is narrow, documented, justified, and reviewed.

A risky exclusion is broad, permanent, unexplained, and applied because the SOC is tired of seeing an alert.

For example:

- excluding a specific known scanner from a specific reconnaissance alert may be reasonable after validation,
- excluding an entire subnet because “a lot of alerts come from there” is usually dangerous,
- excluding a privileged admin account is almost never a safe default,
- excluding a domain controller or Tier 0 asset creates a serious blind spot.

Noise reduction should improve SOC trust.

It should not train the environment to ignore the exact systems attackers want to abuse.

## Field insight

The best tuning question is:

> “Can we reduce this specific false positive without removing visibility into the attack technique?”

If the answer is no, the team may need to improve context, not suppress the alert.

---

# 15. Common false positives and how to reason about them

False positives are not all equal.

Some are caused by bad detection logic.

Many are caused by environmental ambiguity.

MDI is especially sensitive to identity context. If context is poor, detection quality drops.

Common benign or false-positive triggers include:

- security scanners using DNS or LDAP discovery,
- vulnerability management tools,
- broad administrative scripts,
- maintenance activity,
- Remote Credential Guard over RDP,
- legacy systems touching honeytokens,
- macOS or non-Windows authentication quirks,
- NAT or VPN source ambiguity,
- poor Network Name Resolution,
- incomplete learning periods,
- low thresholds in production,
- service accounts behaving like users,
- users performing admin work from non-PAW devices.

## 15.1 Security scanners

Security scanners often perform behavior that looks like reconnaissance.

They enumerate systems, query DNS, inspect exposed services, and sometimes touch directory information.

### How to handle

Do not immediately exclude the scanner globally.

First ask:

- Which detection is triggered?
- Which scanner is involved?
- Which source IP or device?
- Is the scanner expected to query AD?
- Is the behavior limited to maintenance windows?
- Can a tuning rule target only the known benign pattern?

## 15.2 Administrative tools

Admin tools often query users, groups, policies, services, and computers.

That can resemble attacker reconnaissance.

### How to handle

Separate normal admin behavior from risky admin behavior.

Ask:

- Is the source a PAW?
- Is the user privileged?
- Is the tool approved?
- Is the timing expected?
- Is the target sensitive?
- Is this behavior documented?

If admin activity happens from random workstations, the problem is not only alert tuning.

The problem is weak administrative hygiene.

## 15.3 NAT and VPN ambiguity

NAT and VPN devices can collapse many users or devices behind one source IP.

This makes attribution harder.

### How to handle

Improve identity and device correlation where possible.

Review:

- VPN integration,
- RADIUS accounting,
- NNR health,
- source IP mapping,
- device inventory,
- DHCP/DNS hygiene,
- and XDR correlation with endpoint data.

Do not blindly exclude VPN infrastructure.

That may remove visibility into real attacks.

## 15.4 Poor NNR

Network Name Resolution problems can make alerts look vague or wrong.

MDI may know that suspicious activity came from an IP address but struggle to map it confidently to a computer.

### How to handle

Fix NNR before tuning away alerts.

Poor attribution creates both false positives and false negatives.

## 15.5 Incomplete baselines

During learning periods, alerts may be noisier or less accurate.

### How to handle

Set expectations early.

For the first few weeks, classify alerts carefully, monitor health, and avoid aggressive tuning unless the benign source is well understood.

## 15.6 Disabled users still appearing active or “present in the network”

A common customer question is:

> “Why does a disabled user still appear active, present, or visible in MDI?”

This usually comes from a misunderstanding of identity lifecycle versus evidence visibility.

Disabling an account prevents new authentication in normal conditions, but it does not erase the account from historical telemetry, existing evidence, or every active context immediately.

A disabled user may still appear in MDI or Defender XDR because:

- the account appears in historical logs or alert evidence,
- the account had Kerberos tickets issued before it was disabled,
- cached credentials may exist on endpoints,
- sessions or tokens may not yet be revoked,
- directory references still exist,
- the account is still linked to past activity,
- devices may still show previous logon context,
- or delayed telemetry is still arriving and being processed.

This does not necessarily mean the disabled account is successfully authenticating right now.

It means the identity still exists in security evidence and may still be relevant to investigation.

### Why this happens

Identity systems are not erased instantly.

Authentication state, session state, ticket lifetime, cached logon context, and historical evidence are different things.

For example:

- disabling an AD account stops new domain authentication,
- existing Kerberos tickets may remain valid until expiry unless invalidated by other controls,
- cloud sessions require session revocation to terminate active tokens,
- endpoint logon sessions may persist until logoff or restart,
- historical evidence remains visible for investigation.

### Consultant framing

A better way to explain it is:

> Disabled means the account should not be allowed to perform new normal authentication.  
> It does not mean all historical evidence, tickets, sessions, or cached references disappear immediately.

This matters in investigations.

If a disabled account appears in an MDI timeline, the SOC should check whether the event is:

- historical evidence,
- delayed ingestion,
- failed authentication,
- ticket-based activity,
- cached endpoint context,
- active cloud session behavior,
- or true post-disable authentication.

That distinction prevents false escalation and improves SOC confidence.

---

# 16. What breaks detection confidence

MDI detection confidence depends on telemetry quality.

The following issues reduce confidence:

- incomplete sensor coverage,
- unmonitored DCs,
- unmonitored AD FS,
- unmonitored AD CS,
- unmonitored Entra Connect,
- standalone sensors without ETW,
- missing Windows audit policies,
- DSA or gMSA problems,
- unhealthy NNR,
- blocked internal resolution ports,
- encrypted protocol content,
- legacy operating systems,
- low-quality asset inventory,
- noisy admin behavior,
- broad exclusions,
- and immature baselines.

## Consultant callout

When investigating a missed or weak alert, do not start by blaming detection logic.

Start with telemetry quality.

Ask:

```text
Did MDI have the right sensor, on the right system, seeing the right signal,
with the right event context, at the right time, with enough baseline maturity?
````

That question solves many “MDI did not detect X” conversations.

## Consultant warning: exclusions can also break confidence

Exclusions are sometimes invisible in day-to-day triage because they reduce alert volume.

That is exactly why they are dangerous.

A broad exclusion can make the SOC believe the environment is clean when MDI is simply no longer alerting on important behavior.

Exclusions can break detection confidence by hiding:

* reconnaissance from known scanners that later become compromised,
* suspicious activity from privileged admin devices,
* identity activity through NAT or VPN infrastructure,
* behavior involving sensitive service accounts,
* or attack paths involving Tier 0 infrastructure.

Every exclusion should be treated like a detection coverage exception.

It should have:

* an owner,
* a reason,
* a scope,
* a risk statement,
* an expiration or review date,
* and evidence that narrower tuning was considered first.

If those details do not exist, the exclusion is not tuning.

It is an undocumented blind spot.

---

# 17. What MDI cannot see or guarantee

A credible MDI blog series must include limitations.

Not because we want to weaken confidence in the product.

Because serious consultants do not sell magic.

They explain detection boundaries.

## 17.1 MDI cannot detect activity on unmonitored identity infrastructure

If a domain controller is not monitored, MDI cannot fully see activity that only touches that DC.

This also applies to:

* RODCs,
* AD FS servers,
* AD CS servers,
* Entra Connect servers,
* staging sync servers,
* and other identity infrastructure.

Partial coverage creates blind spots.

## 17.2 Standalone sensors do not provide full coverage

Standalone sensors do not support ETW.

They also depend on correct network mirroring.

That means reduced detection coverage compared to direct sensors.

## 17.3 MDI is not a full audit platform

MDI is not designed to capture every operation on every server.

It captures signals required for threat detection, posture recommendations, investigation, and identity analytics.

For long-term audit retention, compliance evidence, and broad log analytics, use Sentinel or another SIEM strategy.

## 17.4 MDI is not a DLP solution

MDI is not Data Loss Prevention.

It does not inspect file content, classify sensitive documents, monitor clipboard activity, enforce data handling rules, or determine whether a user copied confidential information into an email, USB drive, browser, or SaaS platform.

That responsibility belongs to Microsoft Purview DLP and related data protection controls, not MDI.

MDI may help detect identity behavior that occurs before or around data access, such as:

* credential compromise,
* lateral movement,
* suspicious authentication,
* privilege escalation,
* or directory reconnaissance.

But it does not answer:

> “What sensitive file did the user copy?”

It answers identity-plane questions such as:

> “Which account authenticated, from where, using which protocol, against which identity infrastructure, and was that behavior suspicious?”

This distinction is critical.

Do not sell MDI as DLP.

Do not tune MDI as if it understands data content.

Do not use MDI alerts alone to prove data exfiltration.

Use MDI to understand identity compromise and identity abuse. Use Purview, Defender for Cloud Apps, MDE, Sentinel, and storage/application logs to investigate data access and exfiltration.

## 17.5 MDI cannot decrypt all encrypted protocol traffic

MDI performs deep packet inspection, but it cannot decrypt certain encrypted protocol traffic such as AtSvc and WMI.

It may still analyze traffic patterns, but not all content.

## 17.6 Kerberos Armoring affects some detections

Kerberos Armoring is generally supported, but specific detections, such as some Overpass-the-Hash scenarios, may not work when Kerberos Armoring is enabled.

## 17.7 Source-user attribution may fail

Sometimes MDI cannot correlate the source user to the activity.

In those cases, it may show only the source computer.

This usually happens when the relevant user context is not available in the captured traffic or cannot be correlated confidently.

## 17.8 Behavioral detections are weaker before baselines mature

Before learning periods complete, MDI has less context.

This can cause:

* lower confidence,
* more noise,
* missed behavioral anomalies,
* or inconsistent test results.

## 17.9 Established admin behavior can hide abuse

If an account or machine regularly performs broad administrative tasks, MDI may learn that behavior as normal.

If an attacker compromises that account or system and performs similar activity, anomaly detection becomes harder.

This is why PAWs, tiering, and least privilege matter.

## 17.10 Manual exclusions can create attacker paths

If an excluded scanner, admin workstation, server, IP, or domain is later compromised, the attacker may inherit the exclusion blind spot.

Exclusions should be treated like security exceptions.

They need owners, justification, review dates, and scope control.

## 17.11 MDI does not guarantee real-time identity disappearance after disablement

Disabling an account is a control action.

It is not an evidence deletion action.

A disabled account can still appear in:

* identity timelines,
* alert evidence,
* historical logs,
* device context,
* session-related traces,
* Kerberos ticket-related activity,
* and correlated XDR incidents.

This does not automatically mean the account is still successfully logging on.

The SOC must distinguish between:

* new successful authentication,
* failed authentication after disablement,
* ticket usage,
* cached endpoint context,
* historical evidence,
* delayed ingestion,
* and active cloud sessions that require revocation.

This is why credential compromise response should include more than password reset or disablement when applicable.

For hybrid identity, consider:

* disabling the account,
* forcing password reset,
* revoking cloud sessions,
* validating active logons,
* checking Kerberos ticket implications,
* isolating affected endpoints,
* and reviewing timeline evidence after containment.

## 17.12 Single-step simulations may not produce complete attack stories

MDI is strongest when it sees sequences.

A single isolated command in a lab may not reflect a realistic attack chain.

Good testing should simulate meaningful identity behavior across phases, not just one atomic technique.

---

# 18. How to test MDI properly

A proper MDI validation should not be a random set of commands.

It should be structured around detection assumptions.

## 18.1 Separate detection types

Group your tests into:

* deterministic detections,
* behavioral detections,
* posture findings,
* XDR correlation scenarios,
* response-action tests,
* hunting validation,
* and sensor health validation.

## 18.2 Document prerequisites

For each scenario, document:

* required sensor location,
* required protocol visibility,
* required Windows events,
* required baseline maturity,
* required entity tagging,
* required threshold setting,
* expected XDR correlation,
* expected alert evidence,
* expected hunting tables.

## 18.3 Avoid unrealistic single-step tests

A realistic identity attack chain may include:

1. endpoint compromise,
2. AD reconnaissance,
3. SPN discovery,
4. Kerberos ticket activity,
5. credential reuse,
6. lateral movement,
7. privilege escalation,
8. suspicious directory modification,
9. cloud pivot,
10. data access or exfiltration.

MDI may not fire on every single step as a separate alert.

That is not the goal.

The goal is to produce enough identity evidence for Defender XDR to tell the attack story.

## 18.4 Validate negative cases too

Testing should also confirm what MDI does not see.

For example:

* activity against an unmonitored DC,
* standalone sensor ETW-dependent gaps,
* poor NNR attribution,
* missing audit policy behavior,
* baseline-dependent detections before maturity,
* and excluded entities.

This makes the assessment honest.

## 18.5 Validate exclusion behavior intentionally

If the environment uses exclusions, include them in the test plan.

Do not assume exclusions only remove noise.

Validate:

* which alerts are suppressed,
* which entities are affected,
* whether the exclusion applies globally or narrowly,
* whether related XDR correlation is impacted,
* whether hunting still shows the underlying activity,
* and whether the SOC understands the residual risk.

A good validation exercise should answer:

> “If this excluded asset is compromised, what will MDI still show us, and what will it no longer show us?”

If nobody can answer that, the exclusion is not ready for production.

---

# 19. Detection quality is an architecture outcome

MDI detection quality is not only a product feature.

It is the result of architecture and operations.

A clean MDI deployment has:

* full sensor coverage,
* healthy identity infrastructure monitoring,
* correct audit policy,
* reliable NNR,
* working DSA/gMSA design,
* mature baselines,
* limited broad exclusions,
* well-tagged sensitive identities,
* documented test mode usage,
* and XDR investigation workflows.

A weak deployment has:

* missing DCs,
* standalone sensors used without documented trade-offs,
* unmonitored AD CS,
* poor NNR,
* broken audit policy,
* noisy admin behavior,
* low thresholds left in production,
* broad exclusions,
* and no owner for health issues.

The same detection engine will behave differently in those two environments.

That is why consultants should not only ask:

> “Which alerts does MDI support?”

They should ask:

> “Is this environment capable of producing high-confidence identity detections?”

---

# 20. Practical consultant assessment questions

When assessing MDI detection readiness, use questions like these.

## Detection logic

* Which detections are deterministic?
* Which require behavioral baselines?
* Which require specific audit events?
* Which require ETW?
* Which require XDR correlation to become meaningful?
* Which are affected by Kerberos Armoring or encrypted traffic?

## Baselines

* How long has MDI been deployed?
* Are all DCs past baseline maturity?
* Were sensors recently added?
* Were thresholds changed?
* Is Recommended Test Mode enabled?
* Are alerts being evaluated differently during learning?

## Tuning

* Are tuning rules evidence-based?
* Are broad exclusions used?
* Are scanners excluded globally?
* Are sensitive accounts excluded?
* Are NAT or VPN devices excluded?
* Are exclusions reviewed periodically?
* Does each exclusion have an owner and review date?
* Was narrower tuning attempted before exclusion?
* Could any exclusion hide lateral movement or reconnaissance?

## False positives

* Are scanners generating reconnaissance alerts?
* Are admin tools generating SAM-R or LDAP alerts?
* Are VPN/NAT devices causing ambiguity?
* Is NNR healthy?
* Are low thresholds causing production noise?
* Are legacy systems touching honeytokens?
* Are disabled accounts being interpreted correctly as historical evidence, failed activity, ticket usage, or true post-disable authentication?

## Limits

* Are any DCs unmonitored?
* Are AD FS, AD CS, and Entra Connect monitored?
* Are standalone sensors used?
* Are required events missing?
* Are legacy OS versions present?
* Is encrypted traffic limiting inspection?
* Are established admin patterns reducing anomaly sensitivity?
* Are stakeholders expecting MDI to behave like DLP?
* Are stakeholders expecting MDI to monitor user productivity or general user activity?

---

# 21. Final field framing

When explaining MDI detection logic to a customer, I would avoid saying:

> “MDI detects reconnaissance, credential theft, lateral movement, and domain dominance.”

That is true, but too shallow.

A better explanation is:

> “MDI detects identity attacks by combining AD protocol analysis, Windows events, ETW, directory enrichment, behavioral baselines, deterministic attack patterns, anomaly detection, and Microsoft threat intelligence. Some detections can trigger immediately because the behavior matches known malicious patterns. Others require MDI to learn what normal looks like for users, devices, protocols, and domain controllers. Detection quality depends heavily on sensor coverage, event availability, NNR health, baseline maturity, and careful tuning.”

I would also avoid saying:

> “MDI monitors users.”

That creates the wrong expectation.

A more accurate consultant explanation is:

> “MDI evaluates identity misuse and suspicious identity-plane behavior. It does not monitor employee productivity, inspect data content, or replace DLP. It helps Defender XDR understand authentication, authorization, directory, privilege, and protocol activity involving accounts and identity infrastructure.”

That framing sets realistic expectations.

MDI is strong when it has visibility, context, and time.

It is weaker when identity infrastructure is partially monitored, baselines are immature, administrative behavior is messy, or teams tune by excluding instead of understanding.

The best MDI deployments do not chase the maximum number of alerts.

They produce identity detections that are explainable, correlated, and useful during an investigation.

That is what makes MDI valuable inside Defender XDR.

---

# Coming next

In Part 3, we will move from detection logic to operations.

We will look at how to use MDI inside Microsoft Defender XDR investigations:

* why MDI alerts should not be triaged in isolation,
* how MDI alerts correlate into XDR incidents,
* how to use Alert Story, Alert Graph, and Identity Timeline,
* which Advanced Hunting tables matter,
* how to respond to credential compromise, lateral movement, DC compromise, and Golden Ticket scenarios,
* and how MDI posture intelligence helps reduce attack paths before compromise.

Because detection is only useful if the SOC can investigate, hunt, and respond with confidence.