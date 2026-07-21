# Multi-IdP Identity Governance Lab (Okta, Active Directory, Entra ID)

A hands-on identity governance lab simulating a hybrid enterprise environment where on-premises Active Directory acts as the identity source of truth, feeding into two cloud identity providers, Okta and Microsoft Entra ID, each serving a distinct purpose. This project covers the full multi-IdP lifecycle, from federation and delegated authentication to access governance, lifecycle propagation testing, and cross-system security monitoring.

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

Many enterprises run more than one identity provider, whether due to mergers and acquisitions, incumbent tooling from before a Microsoft 365 migration, or a deliberate choice to keep a vendor-neutral SSO layer for non-Microsoft SaaS apps while relying on Entra ID for Microsoft workloads. This lab replicates that reality: Okta handles broad SaaS SSO, Entra ID handles Microsoft 365 and Azure-native access, and both are fed by the same on-premises Active Directory. The project demonstrates federation, RBAC and group-based access assignment, access governance, and cross-system reconciliation, using the Okta API and Microsoft Graph API for reporting.






<img width="793" height="542" alt="Screenshot 2026-07-10 at 9 34 40 AM" src="https://github.com/user-attachments/assets/b3e8e5b2-0f2f-407e-be02-1e0069e43b22" />


---

## Technologies Used

| Technology | Purpose |
|---|---|
| Windows Server 2025 | Domain Controller / AD DS |
| Active Directory Domain Services | On-premises identity store / source of truth |
| Okta (Integrator Free Plan) | Cloud IdP for SaaS SSO |
| Okta AD Agent | Delegated authentication and directory import |
| Microsoft Entra ID | Cloud IdP for Microsoft 365 / Azure access |
| Microsoft Entra Connect Sync | Hybrid identity synchronization |
| Okta API | Sign-in log retrieval and account status reporting |
| Microsoft Graph API | Entra ID reporting and cross-referencing |
| Python | Custom access review / orphaned account detection script |
| Azure Log Analytics | Security monitoring |
| KQL (Kusto Query Language) | Log correlation and anomaly detection |

---

## Architecture

```
                         ┌────────────────────────────┐
                         │  On-Premises Active          │
                         │  Directory (Source of Truth) │
                         │  Domain: examlabpractice.com  │
                         └──────────┬──────────┬─────────┘
                                    │           │
                     Okta AD Agent  │           │  Entra Connect Sync
                                    ▼           ▼
                   ┌─────────────────────┐  ┌──────────────────────────┐
                   │        Okta          │  │     Microsoft Entra ID    │
                   │                       │  │                           │
                   │  Delegated Auth       │  │  Microsoft 365 / Azure    │
                   │  SSO (SAML/OIDC)      │  │  Conditional Access       │
                   │  Sign-On Policies     │  │  RBAC / PIM               │
                   │  System Log (API)     │  │  Log Analytics            │
                   └─────────────────────┘  │  Sign-in Logs (Graph API) │
                                             └──────────────────────────┘
```

---

## Project Phases

### Phase 1 — Okta AD Agent Integration

- Installed the Okta AD Agent on the domain-joined Windows Server 2025 controller
- Imported existing AD OUs, users, and groups into Okta
- Configured delegated authentication so Okta validates credentials directly against Active Directory rather than storing separate passwords
- Documented import matching rules used to reconcile AD users against existing Okta profiles
- Verified agent health and tested end-to-end login using AD credentials

---

### Phase 2 — Okta SSO Application Integrations

- Connected multiple SAML and OIDC applications in Okta, including AWS IAM Identity Center
- Assigned application access based on AD-synced security groups rather than individual users
- Configured a distinct Sign-On Policy requiring step-up MFA for a higher-privilege application group
- Documented the SAML vs. OIDC protocol choice per application

## SSO Application Access Policy

| Application | Protocol | Assigned Group | Sign-On Policy |
|---|---|---|---|
| AWS IAM Identity Center | SAML | ITAdmins | Require MFA, no additional restrictions |
| Sample SAML App | SAML | AllEmployees | Standard sign-on policy |
| GitHub | OIDC | ITAdmins | Require MFA, step-up on new device |
| Salesforce Dev Org | OIDC | Sales | Standard sign-on policy |

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

- Hybrid multi-IdP architecture design across Active Directory, Okta, and Entra ID
- Delegated authentication and directory import via the Okta AD Agent
- Federation protocols (SAML, OIDC) and group-based application access assignment
- Sign-On Policy and Conditional Access design for risk-based authentication
- Custom access review scripting using the Okta API and Microsoft Graph API
- Lifecycle propagation analysis and orphaned account risk identification
- Security monitoring and anomaly detection using KQL and Azure Log Analytics
