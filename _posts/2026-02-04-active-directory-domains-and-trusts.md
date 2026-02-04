---
title: Active Directory Domains and Trusts (Concepts, Types, and Secure Configuration) 
date: 2026-02-04
categories: [windows server,active directory,domains and trusts,DNS]
tags: [AD,WS,DNS,domains,trusts,writeups,learning,notes]
toc: true
---

# AD Domains and Trusts

Active Directory (AD) domains are logical groupings of users, computers, and other network resources that simplify management and improve security within an organization. These domains act as containers, organizing resources based on policies, permissions, and administrative boundaries.

Trusts are connections established between different domains, enabling them to communicate securely and share resources. Trusts play a critical role in environments with multiple domains, allowing users in one domain to access resources in another without compromising security or requiring repetitive authentication.

AD Domains and Trusts enable seamless resource sharing while maintaining security across multiple domains, making it easier for organizations to collaborate and access shared assets. However, if not properly configured, this can result in operational inefficiencies, unauthorized access, and significant security vulnerabilities, potentially exposing sensitive data and critical systems to cyber threats.

## Properties of AD Trust Relationships

AD Trust Relationships come with specific properties that define how domains interact:

### Directional vs. Bidirectional Trusts

- **One-way Trust**: Occurs when only one domain trusts another.
- **Two-way Trust**: Exists when both domains trust each other, allowing mutual access to resources between the two domains.

### Transitivity

- **Transitive Trust**: Automatically extends to other domains. For example, if Domain A trusts Domain B, and Domain B trusts Domain C, then Domain A also trusts Domain C through transitivity.
- **Non-Transitive Trust**: Restricted to the two directly connected domains; it does not extend beyond the immediate connection.

### Authentication Types

- **Kerberos Authentication**: Provides a secure method for validating identity and access within a trust, ensuring that only authorized users can interact with resources, maintaining the integrity and security of the system.
- **Selective Authentication**: Offers granular control over resource access across a trust, meaning only specified users or groups are permitted to access resources, enhancing security by limiting unnecessary access.

### Domain-Wide Authentication

All users in the trusted domain have access to resources within the trusting domain unless explicitly restricted.

## Types of Trust Relationships in AD

Trust relationships in AD span several types, each designed for different scenarios:

- **Parent-Child Trust**: Automatically created when a child domain is added to a parent domain in the same tree. Always two-way and transitive. Commonly used in hierarchical domain environments where child domains inherit parent policies and resources.
- **Tree-Root Trust**: Automatically established between the roots of two domain trees in the same forest. Always two-way and transitive. Used to connect multiple domain trees within the same forest for unified resource sharing.
- **External Trust**: Manually created between domains in different forests or between an AD domain and a non-AD domain. Non-transitive, can be one-way or two-way. Ideal for resource-sharing between AD and older Windows NT 4.0 domains or AD domains in separate forests.
- **Forest Trust**: Manually created between the root domains of two forests. Transitive within each forest but non-transitive across multiple forests. Used to establish extensive trust between corporate environments with separate forests.
- **Shortcut Trust**: Manually created within a forest to reduce authentication time between two domains. Transitive, can be one-way or two-way. Suitable for large forests where users frequently access resources in different domains.
- **Realm Trust**: Establishes a connection between a Windows domain and a Kerberos realm. Can be transitive or non-transitive, and may be one-way or two-way. Enables secure integration between AD and UNIX/Linux environments.
- **Cross-Link Trust**: Created manually to improve authentication time between domains in different trees within the same forest. Always transitive. Used when authentication needs to bypass standard parent-child trust paths for efficiency.

## Common Issues with Domain Trust Configurations

Common issues with Domain Trust configurations include the following:

- **Time Synchronization**: Trust relationships rely on time synchronization between Domain Controllers; even a slight time drift can cause authentication failures.
- **DNS Resolution**: DNS is critical for trust relationships; domain name resolution should be functioning correctly between domains.
- **Network Firewalls**: Misconfigured network firewalls—ports like 445 (SMB) and 135 (RPC) must be open for trust communication.
- **Entra Connect Synchronization Issues**: Common issues include missing users, groups, or attributes in Entra ID due to misconfigured synchronization settings.

## Security Considerations for Trust Relationships

Ensuring the security of trust relationships minimizes the risk of data breaches or unauthorized access. Some key security considerations include:

### Enforce the Principle of Least Privilege

Restrict access to only the users and groups that truly need it for their roles. For instance, instead of allowing domain-wide authentication, selective authentication can be used to limit access to only the necessary resources. This minimizes the potential attack surface and reduces the risk of misuse.

### Regular Trust Relationships Auditing

Regularly monitor trust relationship configurations using robust auditing tools to detect unauthorized changes or errors. Look for outdated trusts, excessive permissions, or unusual activity, and resolve any issues promptly to maintain a secure environment.

### SID Filtering

To prevent improper privilege escalation, enable Active SID Filtering. This feature blocks unknown or unauthorized SIDs during authentication, ensuring that only verified credentials are used for accessing resources. This is important in environments with multiple domains or external trusts, where rogue SIDs could be exploited.

### IPsec

Protect sensitive inter-domain communications by encrypting data exchanges with IPsec. This protocol secures communication channels against eavesdropping, tampering, and unauthorized access. Data shared between domains should remain private and protected from potential breaches or interceptions by leveraging IPsec.

## Overview of Trust Relationships

Briefly said, Trust Relationships between AD domains allow users from one domain to authenticate to another domain. They are most often configured when merging or migrating multiple organizations. Trust Relationships can only be configured between AD forest root domains.

## Configuring Trust Relationships

When configuring Trust Relationships, a first key step before establishing the trust relationship is to make sure that the domain controllers on both sides can see each other and perform name resolution in the forest. This is done via configuring DNS conditional forwarders in both AD domains.

This can be done using the DNS Manager on each domain controller/DNS server:
- In the DNS console, right-click Conditional Forwarders.
- Select New Conditional Forwarder.
- Specify the name of the second domain to establish a trust relationship with, the IP address of the Domain Controller in that domain.
- Enable the option "Store this conditional forwarder in Active Directory" and in the drop-down menu choose "All DNS servers in this forest."
- Replicate the same setup on the other domain controller/DNS server with the relevant domain name and IP address.

Once both rules are configured across both servers, check the properties and make sure they both correctly resolved.

The following steps enable the trust relationship:
- Open AD Domains and Trusts.
- Open domain properties.
- Go to the Trusts tab and click "New Trust."
- Specify the name of the forest to establish a trust relationship with.
- Choose the type of trust relationship (e.g., External Trust or Forest Trust).
- Select the direction of trust (two-way, or one-way: incoming or outgoing).
- Choose the domain in which the trust will be created (this domain only, or both this domain and the specified domain).
- Set the authentication scope (forest-wide or selective).

To properly configure the trust relationship for your environment, refer to the guide above to understand how you want to set up the relationship, and then follow this technical step-by-step guide for the actual configuration.

*Note: This blog doesn't cover the configuration using PowerShell (might be updated in the future if I had to set it up).*

## Conclusion

Mastering AD Domains and Trusts is essential for efficient and secure network management in multi-domain environments. By understanding the properties, types, common issues, and security best practices, administrators can ensure robust configurations that support organizational needs while mitigating risks.

Thanks for reading – more technical blogs around Windows Server and Active Directory coming soon. 

**Inshallah** this becomes a long series. The more I break, the more I’ll share.

Stick around if you want to learn together.

```bash{.dua-tawfik}
رَبِّ زِدْنِي عِلْمًا ۞ وَفَهْمًا وَحِكْمَةً
رَبِّ يَسِّرْ وَلَا تُعَسِّرْ ۞ وَتَمِّمْ بِالْخَيْرِ
```

#### ~ Darknight