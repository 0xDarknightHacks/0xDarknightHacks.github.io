---
title: "Portfolio | From Infrastructure to Microsoft Security Engineering"
date: 2026-08-16 10:52:00 +0100
categories: [Portfolio, Career]
tags: [microsoft-365, microsoft-security, entra-id, defender-xdr, intune, powershell, microsoft-graph, zero-trust, infrastructure, pki]
description: "My journey from networking, infrastructure and cybersecurity labs to Microsoft 365 security consulting, engineering, automation, presales and technical leadership."
pin: true
toc: true
comments: false
published: false
---------------

# Portfolio | From Infrastructure to Microsoft Security Engineering

I am **Alaa Eddine Ayedi**, a Microsoft 365 and Cloud Security Consultant working across the Microsoft security ecosystem, with a particular focus on **Microsoft Entra ID, Defender XDR, Intune, Microsoft Purview, Microsoft Graph, PowerShell automation, Zero Trust, security assessments and solution engineering**.

Today, my work sits at the intersection of three areas I enjoy most:

* **Security consulting** — assessing, designing, implementing and troubleshooting Microsoft security solutions.
* **Security engineering** — building automation, assessment platforms and security tooling with PowerShell and Microsoft Graph.
* **Solution engineering and presales** — translating requirements into architectures, technical proposals, workshops, licensing scenarios and implementation approaches.

That is where I am today.

Getting here was less of a straight line.

---

## The Foundations

My technical journey began with **Information and Communication Technologies**, where I developed the networking, systems and telecommunications foundations that would later become important in cybersecurity.

My early internships gave me exposure to several sides of engineering rather than immediately locking me into one specialization.

In **2021**, I completed an early internship around **machine learning**. In **2022**, I gained exposure to engineering in the automotive industry. By **2023**, an R&D internship pushed me further toward experimentation, infrastructure and building things rather than simply studying them.

At the same time, I became increasingly interested in cybersecurity.

That interest quickly moved beyond coursework.

I started building labs, testing security technologies, breaking configurations, rebuilding them and documenting what I learned. I also became involved in the **Google Developer Student Club at ENET'Com**, where I eventually led cybersecurity activities and delivered technical sessions and workshops.

What started as curiosity around networking and security gradually became a much clearer direction.

---

## Building Security Labs Instead of Only Reading About Them

One of the most important stages in my development was building increasingly realistic security environments.

An early project evolved into a **segmented virtual cybersecurity lab** with multiple security zones and technologies spanning:

* Active Directory and Windows endpoints
* pfSense
* IDS/IPS technologies
* SIEM
* malware-analysis environments
* penetration-testing systems
* DFIR tooling
* network segmentation and monitoring

The objective was not simply to install tools.

I wanted to understand how **identity, endpoints, networks, telemetry and security controls interact as a system**.

That same mindset later led me into virtualization and self-hosted infrastructure.

I built and operated environments around **Proxmox VE, Docker, Linux, monitoring, DNS, reverse proxies and internal services**, including projects such as:

* [Proxmox monitoring with InfluxDB and Grafana](https://github.com/nattyCoder/Proxmox-monitoring-InfluxDB-Grafana)
* Elastic Stack and Suricata telemetry pipelines
* SOAR/EDR experimentation with Tines and LimaCharlie
* segmented offensive/defensive cybersecurity labs

Some of those repositories are old now, but they represent an important part of the progression: learning infrastructure deeply enough to understand what security controls are actually protecting.

---

## 2024 — Network & Infrastructure Security

In 2024, I joined **Spark-IT** as a Network & Infrastructure Security Intern.

This was an important transition from personal and academic environments toward structured security engineering.

My work included deploying and integrating technologies around:

* Security Onion
* Elastic Stack
* Suricata
* pfSense
* Filebeat / Elastic Agent
* Prometheus
* Telegraf
* InfluxDB
* Grafana
* Python-based telemetry automation

I worked on centralized security monitoring, network telemetry and threat-detection infrastructure.

This experience reinforced something that still shapes how I approach Microsoft security today:

> A security product is only useful when you understand the architecture, telemetry, dependencies and operational workflow behind it.

---

## Moving Into Microsoft Security

My focus shifted strongly toward the Microsoft ecosystem during my Telecommunications Engineering studies, where I specialized further in cybersecurity.

I began experimenting with **Azure, Active Directory, hybrid identity and Microsoft security technologies**, including building a domain controller environment, synchronizing identities through Microsoft Entra Connect and integrating Microsoft security capabilities.

In early **2025**, I joined **Consultim-IT**, initially working in a junior/internship capacity around a Microsoft Zero Trust project.

The assignment became a major turning point.

Rather than working with a single Microsoft product, I had to understand how Microsoft's security ecosystem operates as a whole:

* Microsoft Entra ID
* Conditional Access
* MFA and authentication
* Privileged Identity Management
* Microsoft Intune
* Defender for Endpoint
* Defender for Identity
* Defender for Office 365
* Defender for Cloud Apps
* Defender XDR
* Microsoft Purview
* Microsoft 365 workloads
* Microsoft Graph
* PowerShell
* hybrid Active Directory environments

That initial work evolved into broader responsibilities, and eventually into my current position in **Microsoft Security consulting**, with increasing involvement in **presales, solution design and engineering**.

---

# What I Work On Today

## Microsoft 365 Security & Zero Trust

A significant part of my work involves designing, configuring and validating security controls across Microsoft environments.

This includes areas such as:

* identity and privileged-access security;
* Conditional Access and authentication;
* endpoint security and compliance;
* Defender XDR integration;
* email and collaboration security;
* SaaS and cloud application governance;
* data security and Microsoft Purview;
* hybrid identity;
* Zero Trust architecture;
* security posture assessment and remediation.

I approach these technologies as connected security layers rather than isolated products.

---

## Consulting, Presales & Solution Design

My role has progressively expanded beyond implementation.

I now contribute to customer-facing activities including:

* technical discovery and requirements analysis;
* solution architecture;
* licensing analysis;
* implementation planning;
* effort estimation;
* technical proposals;
* RFP/RFI responses;
* workshops and demonstrations;
* migration scenarios;
* troubleshooting and technical support;
* documentation and operational handover.

I particularly enjoy the point where a customer requirement has to be converted into something technically realistic:

> **What is the actual problem? What Microsoft capabilities solve it? What are the licensing and architectural dependencies? How should it be implemented and validated?**

---

# Engineering Microsoft Security Assessments

One of my largest ongoing engineering projects is a **Microsoft 365 Security Assessment Orchestration Platform**.

The objective is to move away from fragmented assessment outputs and toward a unified assessment model spanning almost the entire Microsoft 365 security estate.

The platform orchestrates multiple security-assessment engines and processes their outputs through a common pipeline involving:

```text
Assessment engines
        │
        ▼
Collection & orchestration
        │
        ▼
Normalization
        │
        ▼
Canonical control mapping
        │
        ▼
Deduplication & evidence correlation
        │
        ▼
Risk & confidence analysis
        │
        ▼
Remediation prioritization
        │
        ▼
Reporting & historical comparison
```

The project combines several things I am particularly interested in:

**Microsoft security architecture + PowerShell engineering + Microsoft Graph + security assessment methodology + data normalization + reporting.**

Rather than writing another collection of independent scripts, the goal is to engineer a reusable assessment platform capable of bringing different Microsoft 365 security signals into a coherent model.

---

# Building Security Solutions & Mentoring

Another important part of my current work has been technically mentoring and contributing to the development of **four Microsoft Security solutions**.

### OAuth & Application Identity Exposure Analyzer

Designed to answer a deceptively simple question:

> Which applications represent the greatest identity and Microsoft 365 security exposure?

The solution analyzes application registrations, service principals, permissions, credentials, ownership and activity to translate raw Microsoft Graph information into prioritized security findings.

### Entra Application Ownership Privilege Path Analyzer

A graph-oriented identity security project focused on identifying indirect privilege paths between users, groups, application ownership and privileged application permissions.

The objective is to expose situations where a user may not appear privileged through traditional role assignments but still controls a path toward a highly privileged application identity.

### Conditional Access Effective Exposure Analyzer

Rather than simply documenting Conditional Access policies, this project focuses on **effective protection**.

It evaluates identities, roles, groups, assignments, exclusions and sign-in evidence to identify users who are not receiving the protections an organization expects.

### Real-Time Zero Trust Simulation & Validation Platform

The fourth project extends beyond static assessment.

It explores how Microsoft security controls can be **simulated and continuously validated**, providing a controlled environment for testing Zero Trust behavior and security-control effectiveness across Microsoft technologies.

Across these projects, my involvement spans technical mentorship, architecture, security methodology, PowerShell/Microsoft Graph integration, testing, troubleshooting and design reviews.

---

# Architecting the Platform Behind the Projects

Supporting engineering work properly required more than handing developers a Microsoft 365 tenant.

I designed and currently maintain a **private internal engineering platform** supporting development, collaboration, identity, certificates and secrets.

The platform runs across **Proxmox VE, Ubuntu Server and Docker** and includes services such as:

```text
                    Private Network
                          │
                     Proxmox VE
                          │
                    Ubuntu Server
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
     Forgejo            Nginx            Pi-hole
        │              Internal TLS         │
   PostgreSQL                              DNS
        │
        └──── Microsoft Entra ID OIDC
                          │
                       OpenBao
                    Secrets + PKI
```

The environment provides:

* private source control;
* isolated project repositories;
* Microsoft Entra ID OIDC authentication;
* internal DNS;
* internal TLS;
* reverse proxying;
* centralized secrets management;
* PKI and certificate issuance;
* audit logging;
* project-aware authorization;
* restricted administrative access;
* backup and maintenance workflows.

---

## OpenBao, PKI & Microsoft Graph Authentication

OpenBao became the trust and secrets layer for the environment.

I integrated it as a locally hosted **secrets-management and PKI service**, using:

* integrated Raft storage;
* Shamir unseal;
* internal TLS;
* audit logging;
* Microsoft Entra ID OIDC;
* identity groups and policies;
* project-aware authorization;
* KV v2 secrets storage;
* Root and Intermediate CAs;
* certificate issuance roles.

One of the practical use cases is securing **Microsoft Graph application authentication**.

Instead of placing application credentials inside repositories or configuration files, projects can use OpenBao-backed workflows for:

### Client-secret authentication

```text
Entra application
      │
      ▼
Client secret
      │
      ▼
OpenBao KV
      │
      ▼
PowerShell runtime retrieval
      │
      ▼
Microsoft Graph
```

### Certificate-based authentication

```text
Local private key
      │
      ▼
CSR
      │
      ▼
OpenBao Intermediate CA
      │
      ▼
Signed client certificate
      │
      ├── Private key remains local
      │
      └── Public certificate → Entra ID
```

This infrastructure project allowed me to work beyond Microsoft admin portals and into **identity federation, PKI, secrets management, Linux infrastructure, containerization and secure application authentication**.

---

# Technical Writing & Knowledge Sharing

Writing has always been part of how I learn.

My older Medium articles documented experiments around areas such as **Proxmox, hybrid Active Directory/Entra environments and Zero Trust**.

Over time, I wanted the writing to become more technical, reproducible and closely connected to the work I actually perform.

That became **DarknightHacks**.

Some of the work I have documented includes:

* [Microsoft Defender for Identity Deep Dive — Architecture, Sensors and Identity Telemetry](https://0xdarknighthacks.ayedialaa.org/posts/MDI-Deep-Dive-Part1/)
* [Designing a Production-Representative Microsoft Defender for Identity Lab](https://0xdarknighthacks.ayedialaa.org/posts/MDI-Lab/)
* [Building a Local Microsoft Learn RAG Assistant with Ollama, FastAPI and Open WebUI](https://0xdarknighthacks.ayedialaa.org/posts/microsoft-documentation-ai-assistant/)

The RAG project in particular reflects another area I am exploring: how local AI systems can consume **official Microsoft documentation first**, instead of relying blindly on model memory when dealing with security configurations, Microsoft Graph permissions and fast-changing product behavior.

---

# The Technologies I Work With

### Microsoft Security

`Microsoft Entra ID` · `Defender XDR` · `MDE` · `MDI` · `MDO` · `MDCA` · `Microsoft Intune` · `Microsoft Purview` · `Microsoft 365`

### Identity & Zero Trust

`Conditional Access` · `MFA` · `PIM` · `RBAC` · `Identity Protection` · `Access Reviews` · `Hybrid Identity` · `OAuth 2.0` · `OIDC` · `SAML`

### Engineering & Automation

`PowerShell` · `Microsoft Graph API` · `Microsoft Graph PowerShell` · `REST APIs` · `KQL` · `Python`

### Infrastructure & Security Engineering

`Windows Server` · `Active Directory` · `PKI` · `Certificates` · `OpenBao` · `Proxmox VE` · `Hyper-V` · `Ubuntu` · `Docker` · `Nginx` · `DNS`

### Consulting & Presales

`Security Assessments` · `Solution Design` · `Zero Trust` · `RFP/RFI` · `Technical Proposals` · `Microsoft Licensing` · `Workshops` · `PoCs` · `Migration` · `Troubleshooting`

---

# Where I Am Now

Looking back, the progression makes more sense than it did while I was going through it.

Networking taught me how systems communicate.

Infrastructure taught me how services depend on each other.

Security labs taught me how controls fail.

Active Directory introduced me to identity as a security boundary.

Microsoft 365 showed me what happens when identity, endpoints, applications, email, data and security operations have to work together.

PowerShell and Microsoft Graph gave me a way to automate that ecosystem.

Consulting taught me that technical correctness alone is not enough — a solution also has to address the customer's requirements, licensing, operational reality and business constraints.

And presales is teaching me how to turn all of that into a solution before implementation even begins.

Today, the area I want to continue developing is exactly where those experiences overlap:

> **Microsoft Security, Identity, Security Engineering, Automation, Zero Trust, Solution Architecture and Technical Consulting.**

I am still building, still experimenting and still documenting what I learn — but the objective is much clearer now.

---

## Elsewhere

* **LinkedIn:** [Alaa Eddine Ayedi](https://www.linkedin.com/in/alaaeddineayedi/)
* **GitHub:** [nattyCoder](https://github.com/nattyCoder)
* **Technical Writing:** [DarknightHacks](https://0xdarknighthacks.ayedialaa.org/)
