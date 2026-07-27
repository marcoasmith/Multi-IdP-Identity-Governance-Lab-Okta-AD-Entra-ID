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
      ┌─────────────────────────────┐
      │     On-Premises Active      │
      │                             │
      │ Directory (Source of Truth) │
      │ Domain: examlabpractice.com │
      └─────────────────────────────┘
                     │
                     │  Okta AD Agent
                     ▼
           ┌──────────────────┐
           │       Okta       │
           │                  │
           │  Delegated Auth  │
           │    SSO (SAML)    │
           │ Sign-On Policies │
           └──────────────────┘
                     │
                     ▼
        ┌─────────────────────────┐
        │ AWS IAM Identity Center │
        │                         │
        │ (SAML Service Provider) │
        └─────────────────────────┘
```

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

<img width="800" height="550" alt="Screenshot 2026-07-18 at 1 48 49 PM" src="https://github.com/user-attachments/assets/23823d2b-4943-453d-8589-a9f2f3cb1e98" />
<img width="800" height="550" alt="Screenshot 2026-07-21 at 10 20 56 AM" src="https://github.com/user-attachments/assets/d20527cd-3c5e-4413-b85b-43f3cc9ccd10" />


---

### Phase 2 — Okta SSO Application Integrations

- Federated AWS IAM Identity Center to Okta via SAML 2.0, switching its identity source to Okta as an external IdP
- Fixed a metadata parsing failure by pulling Okta's metadata via curl instead of copy/paste
- Assigned access through a dedicated AD-synced group (GRP_ITAdmins) with least-privilege AWS permissions (ReadOnlyAccess)
- Built a Sign-On Policy requiring MFA for GRP_ITAdmins, prioritized above the default catch-all
- Verified end-to-end SSO: signed in via Okta with AD credentials, landed in AWS with no separate login
- <img width="1642" height="452" alt="Screenshot 2026-07-23 at 10 18 00 AM" src="https://github.com/user-attachments/assets/2ed2dc1b-1b0b-4a17-939d-5610ff4d7c50" />


## SSO Application Access Policy

| Application | Protocol | Assigned Group | Sign-On Policy |
|---|---|---|---|
| AWS IAM Identity Center | SAML | GRP_ITAdmins | Require any two factor types, prioritized above default policy |

---

### Phase 3 — Entra ID Coexistence

- Verified Microsoft Entra Connect is still actively syncing the on-prem AD domain to Entra ID, confirmed via continuous successful sync runs in Synchronization Service Manager
- Found Entra Connect scoped to sync the entire domain rather than excluding OktaTestUsers, while Okta's AD Agent remains separately scoped to that OU only
- Documented the resulting model: AD as the single source of truth, with Okta and Entra ID independently syncing for their respective app ecosystems despite some directory-level overlap
- <img width="932" height="596" alt="Screenshot 2026-07-24 at 9 21 14 AM" src="https://github.com/user-attachments/assets/07129750-4f2b-4883-8f4f-12b3b58ff7ec" />


## Why Two Identity Providers

| Reason | Explanation |
|---|---|
| App catalog breadth | Okta's app catalog covers non-Microsoft SaaS more broadly than Entra ID's gallery |
| Vendor neutrality | Keeps SSO independent from the Microsoft ecosystem for orgs not fully committed to it |
| M&A / multi-forest complexity | Okta can consolidate identity across multiple AD forests or Entra tenants more easily than tenant-to-tenant migration |
| Migration-in-progress reality | Common when an org is mid-migration between platforms and hasn't fully retired the incumbent IdP |

---

### Phase 4 — Lifecycle Propagation Testing

- Created a new AD user in the OktaTestUsers OU and measured propagation time into both Okta (manually triggered import) and Entra ID (automatic delta sync)
- Disabled the same user in AD and measured deprovisioning propagation into each system
- Found that Okta's import schedule was set to "Never" (manual-only) in this lab, so all Okta timings reflect manually triggered imports rather than a scheduled interval

## Propagation Timing Results

| Event | AD Action Time | Okta Propagation | Entra ID Propagation |
|---|---|---|---|
| New user creation | 3:06 PM | ~2 min (manual import) | ~6 min (automatic delta sync) |
| Account disablement | 9:24 AM | ~1 min (manual import) | Delayed until a delta sync was manually triggered the next day, following an overnight gap where the domain controller VM was offline |

### Risk Identified

Okta's manually-triggered imports reflected changes within 1-2 minutes, while Entra Connect's automatic delta sync took roughly 6 minutes under normal conditions. During deprovisioning testing, Entra Connect's sync scheduler appeared to pause while the domain controller VM was offline overnight, delaying propagation of the disabled account until a delta sync was manually triggered the next day. This demonstrates that propagation delay isn't bounded only by the scheduled sync interval, it's also dependent on the sync server's uptime, a real operational risk in environments where the sync infrastructure itself isn't highly available.
<img width="792" height="800" alt="Screenshot 2026-07-25 at 9 42 43 AM" src="https://github.com/user-attachments/assets/aefbaf11-768a-48c0-8115-690f3dd9a188" />

<img width="1067" height="510" alt="Screenshot 2026-07-25 at 9 39 52 AM" src="https://github.com/user-attachments/assets/7476c3c5-e4a0-47a8-bb78-e52ddeb6f4b8" />



---

### Phase 5 — Access Governance & Orphaned Account Detection

Okta's native Access Certification feature requires a Governance-tier license not included in the Integrator Free Plan used for this lab. To demonstrate the same governance outcome, this phase was implemented as a custom script rather than the native UI feature.

- Registered an app in Entra ID with Microsoft Graph's User.Read.All application permission, and created an Okta API token, to enable programmatic access to both platforms
- Built a Python script that reads Active Directory account status from a PowerShell-exported CSV (the source of truth), then pulls user status from both the Okta API and Microsoft Graph API
- Cross-referenced each account's AD status against its Okta and Entra ID status, flagging any account disabled in AD but still active downstream
- Validated the detection logic by disabling a test user in AD without re-syncing Okta, confirming the script correctly flagged the account as orphaned (disabled in AD, still Active in Okta)
- Printed findings as a console-based audit report

<img width="1111" height="544" alt="Screenshot 2026-07-25 at 3 05 04 PM" src="https://github.com/user-attachments/assets/0da82e93-b7d2-4dc9-bdc8-ad5f4aee7f30" />


## Orphaned Account Detection Logic

| Check | Source of Truth | Compared Against | Flag Condition |
|---|---|---|---|
| Account status | Active Directory | Okta active status | AD disabled, Okta still active |
| Account status | Active Directory | Entra ID active status | AD disabled, Entra ID still active |
---


### Phase 6 — Monitoring & Reporting

- Built a Python script to pull Okta System Log data via the Okta API, covering sign-in events, MFA authentication events, and policy evaluation events, with cursor-based pagination handling
- Built a repeated MFA failure detection script that scans System Log data and flags any user with 3+ MFA failures within a 15-minute window, a common signal for credential-stuffing or MFA-fatigue attacks
- Entra ID sign-in log correlation is planned but currently blocked: diagnostic logging is configured correctly (tenant, workspace, and category all confirmed) but SignInLogs is not yet populating in Log Analytics, under active troubleshooting

<img width="1261" height="546" alt="Screenshot 2026-07-27 at 1 04 18 PM" src="https://github.com/user-attachments/assets/4517a67a-1b21-4f18-8dd1-a4144ca03561" />


## Monitoring Coverage

| Query Purpose | Data Source | Detection Goal | Status |
|---|---|---|---|
| Repeated MFA failures | Okta System Log | Identify possible credential-stuffing attempts | Implemented and tested |
| Off-hours sign-ins | Okta System Log + Entra sign-in logs | Flag authentication outside normal business hours | Planned, pending Entra log ingestion |
| Inconsistent account state | Okta API + Microsoft Graph API | Identify accounts active in one system but disabled in AD | Planned |

---

## Key Skills Demonstrated

- Hybrid multi-IdP architecture design across Active Directory, Okta, and Entra ID
- Delegated authentication and directory import via the Okta AD Agent
- Federation protocols (SAML, OIDC) and group-based application access assignment
- Sign-On Policy and Conditional Access design for risk-based authentication
- Custom access review scripting using the Okta API and Microsoft Graph API
- Lifecycle propagation analysis and orphaned account risk identification
- Security monitoring and anomaly detection using Python and the Okta System Log API
