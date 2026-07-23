# Multi-IdP Identity Governance Lab (Okta, Active Directory, Entra ID)

A hands-on identity lab simulating a hybrid enterprise environment where on-premises Active Directory acts as the identity source of truth. This project is being built in phases; completed phases so far cover Okta AD Agent integration, delegated authentication, and SAML-based SSO federation with AWS IAM Identity Center. Planned phases will extend this into Microsoft Entra ID coexistence, lifecycle propagation testing, access governance, and cross-system security monitoring.

---

## Table of Contents

- [Overview](#overview)
- [Technologies Used](#technologies-used)
- [Architecture](#architecture)
- [Project Phases](#project-phases)
  - [Phase 1 — Okta AD Agent Integration](#phase-1--okta-ad-agent-integration)
  - [Phase 2 — Okta SSO Application Integrations](#phase-2--okta-sso-application-integrations)
  - [Phase 3 — Entra ID Coexistence](#phase-3--entra-id-coexistence)
  - [Phase 4 — Lifecycle Propagation Testing](#phase-4--lifecycle-propagation-testing)
  - [Phase 5 — Access Governance & Orphaned Account Detection](#phase-5--access-governance--orphaned-account-detection)
  - [Phase 6 — Monitoring & Reporting](#phase-6--monitoring--reporting)
- [Key Skills Demonstrated](#key-skills-demonstrated)

---

## Overview

Many enterprises run more than one identity provider, whether due to mergers and acquisitions, incumbent tooling from before a Microsoft 365 migration, or a deliberate choice to keep a vendor-neutral SSO layer for non-Microsoft SaaS apps while relying on Entra ID for Microsoft workloads. This lab is being built to eventually replicate that reality, with Okta handling SaaS SSO and Microsoft Entra ID handling Microsoft 365/Azure access, both fed by the same on-premises Active Directory. So far, the project has established the on-prem AD to Okta connection: delegated authentication and SAML-based SSO federation with AWS IAM Identity Center, using group-based access assignment and a dedicated Sign-On Policy. Entra ID coexistence, lifecycle propagation testing, access governance, and cross-system monitoring are planned in later phases.

<img width="793" height="542" alt="Screenshot 2026-07-10 at 9 34 40 AM" src="https://github.com/user-attachments/assets/b3e8e5b2-0f2f-407e-be02-1e0069e43b22" />

---

## Technologies Used

| Technology | Purpose |
|---|---|
| Windows Server 2025 | Domain Controller / AD DS |
| Active Directory Domain Services | On-premises identity store / source of truth |
| Okta (Integrator Free Plan) | Cloud IdP for SaaS SSO |
| Okta AD Agent | Delegated authentication and directory import |
| AWS IAM Identity Center | SAML service provider connected to Okta for SSO |

*Additional technologies (Microsoft Entra ID, Entra Connect Sync, Okta API, Microsoft Graph API, Python, Azure Log Analytics, KQL) are planned for later phases and will be added here as they're implemented.*

---

## Architecture

```
                         ┌────────────────────────────┐
                         │  On-Premises Active          │
                         │  Directory (Source of Truth) │
                         │  Domain: examlabpractice.com  │
                         └──────────┬─────────────────────┘
                                    │
                     Okta AD Agent  │
                                    ▼
                   ┌─────────────────────┐
                   │        Okta          │
                   │                       │
                   │  Delegated Auth       │
                   │  SSO (SAML)           │
                   │  Sign-On Policies     │
                   └─────────────────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │  AWS IAM Identity     │
                   │  Center (SAML SP)     │
                   └─────────────────────┘
```

*The Microsoft Entra ID side of this architecture (Entra Connect Sync, Conditional Access, RBAC/PIM, Log Analytics) is planned for Phase 3 and will be added to this diagram once implemented.*

---

## Project Phases

### Phase 1 — Okta AD Agent Integration

- Created a dedicated `OktaTestUsers` OU in Active Directory to scope the AD Agent import within the Okta Integrator Free Plan's 10-user limit, rather than importing an entire production-sized OU
- Installed the Okta AD Agent on the domain-joined Windows Server 2025 controller and registered it with the Okta org
- Granted the auto-created `OktaService` account Domain Admins permissions to support write access for future user provisioning
- Scoped the Okta import to the `OktaTestUsers` OU only, for both Users and Groups sync
- Ran a full import and confirmed 9 new Okta users, activating them immediately on confirmation
- Enabled delegated authentication so Okta validates credentials directly against Active Directory rather than storing separate passwords
- Verified delegated authentication using Okta's built-in test tool, confirming successful authentication using a test user's AD credentials in User Principal Name (UPN) format (`username@examlabpractice.com`)

<img width="800" height="550" alt="Screenshot 2026-07-18 at 1 48 49 PM" src="https://github.com/user-attachments/assets/23823d2b-4943-453d-8589-a9f2f3cb1e98" />
<img width="800" height="550" alt="Screenshot 2026-07-21 at 10 20 56 AM" src="https://github.com/user-attachments/assets/d20527cd-3c5e-4413-b85b-43f3cc9ccd10" />

---

### Phase 2 — Okta SSO Application Integrations

- Created a free AWS account and enabled AWS IAM Identity Center, then changed its identity source from the built-in directory to an external identity provider (Okta), establishing SAML-based federation
- Connected AWS IAM Identity Center as a SAML 2.0 application in Okta, exchanging service provider and identity provider metadata between the two platforms
- Resolved a metadata parsing failure during initial setup (`Unable to find IDPSSODescriptor in provided idp metadata object`) caused by browser-corrupted XML; resolved by downloading the raw Okta metadata file directly via `curl` rather than copy/paste
- Created a dedicated Active Directory security group (`GRP_ITAdmins`), synced it into Okta via the AD Agent, and reassigned AWS application access from an individually assigned user to the group, matching the project's group-based access model
- Created an AWS permission set (`ReadOnlyAccess`) and assigned it to the group's member for least-privilege access to the AWS account
- Built a dedicated Okta Authentication Policy scoped to `GRP_ITAdmins`, requiring any two factor types for access, prioritized above the org's default catch-all policy so the more specific rule is evaluated first
- Verified the complete SSO flow end-to-end: signed in to Okta as an AD-imported user and federated into the AWS Console via SAML with no separate AWS credentials required

A second application using the OIDC protocol was evaluated (GitHub Organization SSO) but not implemented, as GitHub restricts organization-level SSO to its paid Enterprise Cloud tier. The SAML integration above still demonstrates the core protocol mechanics (metadata exchange, assertion consumption, service provider trust) that OIDC's authorization code flow shares conceptually, just with a token-based rather than assertion-based handshake.

<img width="1642" height="452" alt="Screenshot 2026-07-23 at 10 18 00 AM" src="https://github.com/user-attachments/assets/2ed2dc1b-1b0b-4a17-939d-5610ff4d7c50" />

## SSO Application Access Policy

| Application | Protocol | Assigned Group | Sign-On Policy |
|---|---|---|---|
| AWS IAM Identity Center | SAML | GRP_ITAdmins | Require any two factor types, prioritized above the default catch-all policy |

---

### Phase 3 — Entra ID Coexistence

- Kept Microsoft Entra Connect syncing the same on-prem AD to Entra ID for Microsoft 365 and Azure access
- Scoped Okta to non-Microsoft SaaS applications only, leaving Microsoft-ecosystem access to Entra ID
- Documented the coexistence model so the two IdPs read as an intentional architectural split, not redundant tooling

## Why Two Identity Providers

| Reason | Explanation |
|---|---|
| App catalog breadth | Okta's app catalog covers non-Microsoft SaaS more broadly than Entra ID's gallery |
| Vendor neutrality | Keeps SSO independent from the Microsoft ecosystem for orgs not fully committed to it |
| M&A / multi-forest complexity | Okta can consolidate identity across multiple AD forests or Entra tenants more easily than tenant-to-tenant migration |
| Migration-in-progress reality | Common when an org is mid-migration between platforms and hasn't fully retired the incumbent IdP |

---

### Phase 4 — Lifecycle Propagation Testing

- Created a new AD user and timed propagation into both Okta (scheduled import) and Entra ID (delta sync)
- Disabled a user in AD and timed how long deprovisioning took to reflect in each downstream system
- Documented the sync interval differences between Okta's import schedule and Entra Connect's delta sync cycle

## Propagation Timing Results

| Event | AD Action | Okta Propagation | Entra ID Propagation |
|---|---|---|---|
| New user creation | Created in target OU | Reflected after next scheduled import | Reflected after next delta sync |
| Attribute update | Group membership changed | Reflected after next scheduled import | Reflected after next delta sync |
| Account disablement | Account disabled | Delayed until next import cycle | Delayed until next delta sync |

### Risk Identified
The gap between AD action and downstream reflection creates a window where a disabled AD account can remain active in Okta and/or Entra ID, a real operational risk relevant to deprovisioning SLAs in production environments.

---

### Phase 5 — Access Governance & Orphaned Account Detection

Okta's native Access Certification feature requires a Governance-tier license not included in the Integrator Free Plan used for this lab. To demonstrate the same governance outcome, this phase was implemented as a custom script rather than the native UI feature.

- Built a Python script using the Okta API and Microsoft Graph API to pull active users and their status from both Okta and Entra ID
- Cross-referenced each account's status against Active Directory (the source of truth)
- Flagged accounts active in Okta and/or Entra ID but disabled in AD, simulating orphaned account detection
- Documented findings as a mini audit report

## Orphaned Account Detection Logic

| Check | Source of Truth | Compared Against | Flag Condition |
|---|---|---|---|
| Account status | Active Directory | Okta active status | AD disabled, Okta still active |
| Account status | Active Directory | Entra ID active status | AD disabled, Entra ID still active |
| Group membership drift | Active Directory | Okta group assignment | AD group removed, Okta assignment unchanged |

---

### Phase 6 — Monitoring & Reporting

- Pulled Okta System Log data via the Okta API for sign-in and policy evaluation events
- Correlated Okta sign-in activity with Entra ID sign-in logs in Azure Log Analytics
- Wrote KQL queries for cross-system anomaly detection

## Monitoring Coverage

| Query Purpose | Data Source | Detection Goal |
|---|---|---|
| Off-hours sign-ins | Okta System Log + Entra sign-in logs | Flag authentication outside normal business hours |
| Repeated MFA failures | Okta System Log | Identify possible credential-stuffing attempts |
| Inconsistent account state | Okta API + Microsoft Graph API | Identify accounts active in one system but disabled in AD |

---

## Key Skills Demonstrated

- Active Directory and Okta hybrid identity integration via the Okta AD Agent
- Delegated authentication configuration and verification
- SAML federation setup, including metadata exchange and troubleshooting
- Group-based application access assignment
- Sign-On Policy design for risk-based, group-scoped authentication requirements
- AWS IAM Identity Center configuration, including permission sets and external identity provider federation

*This list will expand as later phases (Entra ID coexistence, lifecycle propagation testing, access governance scripting, and security monitoring) are completed.*
