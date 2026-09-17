# Red Team Assessment — Portfolio Case Study

![[Pasted image 20260916204532.png]]







**Client:** EvilCorp *(Fictionalized)*  
**Assessment Type:** Red Team Assessment  
**Environment:** Active Directory

> [!NOTE]
> ## Sanitization Notice
>
> This document is a sanitized portfolio case study derived from work performed within an authorized red team environment.
>
> Organization names, infrastructure, network addresses, identities, hostnames, application components, certificate templates, assessment objectives, and other identifying or exercise-specific information have been modified, generalized, or omitted.
>
> Generic placeholders are intentionally used throughout the report. Supporting evidence has been similarly redacted or modified to prevent disclosure of credentials, authentication material, infrastructure details, assessment answers, or information that could be used to reconstruct the original exercise.
>
> The purpose of this case study is to demonstrate red team methodology, technical analysis, risk communication, and client-facing security reporting.

---

# Table of Contents

1. [Executive Summary](#executive-summary)
2. [Engagement Overview](#engagement-overview)
3. [Attack Path Summary](#attack-path-summary)
4. [Findings Summary](#findings-summary)
5. [Security Findings](#security-findings)
   - [Finding 01 — Unauthenticated Software Update Mechanism Allows Remote Code Execution](#finding-01--unauthenticated-software-update-mechanism-allows-remote-code-execution)
   - [Finding 02 — AD CS Certificate Template Permits Privileged Account Impersonation](#finding-02--ad-cs-certificate-template-permits-privileged-account-impersonation)
   - [Finding 03 — Stored Browser Credentials Expose Reusable Domain Credentials](#finding-03--stored-browser-credentials-expose-reusable-domain-credentials)
6. [Technical Attack Narrative](#technical-attack-narrative)
   - [Phase 1 — External Reconnaissance](#phase-1--external-reconnaissance)
   - [Phase 2 — Initial Access](#phase-2--initial-access)
   - [Phase 3 — Internal Enumeration](#phase-3--internal-enumeration)
   - [Phase 4 — Credential Access](#phase-4--credential-access)
   - [Phase 5 — Privilege Escalation](#phase-5--privilege-escalation)
   - [Phase 6 — Domain Compromise](#phase-6--domain-compromise)
7. [Cleanup](#cleanup)
8. [Remediation Roadmap](#remediation-roadmap)
9. [Conclusion](#conclusion)
10. [Appendix A — Tooling](#appendix-a--tooling)
11. [Appendix B — Custom Proxy Library](#appendix-b--custom-proxy-library)
12. [Appendix C — Evidence Index](#appendix-c--evidence-index)

---

# Executive Summary

The assessment demonstrated that an unauthenticated external attacker could progress from a public-facing software update service to full compromise of the EvilCorp Active Directory environment.

Initial access was achieved through an unauthenticated file-upload function exposed by a custom software update service. The application accepted attacker-controlled executable components without meaningful content validation. This behavior allowed arbitrary code execution on the internal Windows server `EC-FS-01` in the context of `<Domain>\<Initial-User>`.

Following initial compromise, Active Directory and endpoint enumeration identified additional attack paths. Browser-stored credentials belonging to `<Domain>\<Compromised-User>` were recovered from the compromised system, providing access to an additional domain identity.

The compromised account possessed enrollment permissions over an Active Directory Certificate Services (AD CS) template vulnerable to ESC1 abuse. The affected template permitted an enrollee to supply identity information while supporting authentication-capable certificate usage.

These conditions allowed the compromised account to obtain a certificate representing a privileged domain identity. The resulting certificate was used to obtain privileged Kerberos authentication and administrative access to the Domain Controller.

The assessment ultimately established SYSTEM-level execution on `EC-DC-01`, demonstrating complete compromise of the EvilCorp Active Directory environment.

Three security weaknesses materially contributed to the demonstrated attack path:

1. An unauthenticated software update mechanism permitted arbitrary code execution.
2. An AD CS certificate template permitted privileged account impersonation.
3. Browser-stored credentials exposed a reusable domain credential following endpoint compromise.

Remediation efforts should prioritize the externally accessible update mechanism and vulnerable AD CS configuration because these weaknesses provided the initial access and privilege-escalation paths demonstrated during the assessment.

---

# Engagement Overview

The assessment evaluated EvilCorp's externally accessible attack surface and internal Active Directory environment to identify security weaknesses that could be chained to obtain privileged access.

Testing followed an attacker-driven methodology beginning with external reconnaissance and progressing through initial access, internal enumeration, credential access, privilege escalation, and lateral movement.

The assessment began without credentials or an established internal foothold.

## Scope

| Component | Value |
|---|---|
| External Target | `<External-IP>` |
| Initial Service | TCP/8080 |
| Active Directory Domain | `<Domain>` |
| Initial Internal Host | `EC-FS-01` |
| Domain Controller | `EC-DC-01` |

---

# Attack Path Summary

```text
Unauthenticated External Attacker
            |
            v
Public Software Update Service
            |
            v
Insufficient Update Validation
            |
            v
Runtime Dependency Replacement
            |
            v
Arbitrary Code Execution
         EC-FS-01
  <Domain>\<Initial-User>
            |
            v
Active Directory Enumeration
            |
            v
Stored Browser Credentials
            |
            v
<Domain>\<Compromised-User>
            |
            v
AD CS Enumeration
            |
            v
Vulnerable Certificate Template
           ESC1
            |
            v
Privileged Certificate Enrollment
            |
            v
Privileged Kerberos Authentication
            |
            v
Domain Controller Access
            |
            v
SYSTEM Execution
         EC-DC-01
```

The demonstrated attack path required multiple weaknesses to be chained together.

The internet-facing update mechanism provided the initial entry point. Credential exposure on the compromised host subsequently provided access to another domain identity, whose certificate enrollment permissions exposed a path to privileged authentication through AD CS.

The combination of these weaknesses ultimately resulted in administrative access and SYSTEM-level execution on the Domain Controller.

---

# Findings Summary

| ID | Finding | Severity | Remediation Priority |
|---|---|---|---|
| 01 | Unauthenticated Software Update Mechanism Allows Remote Code Execution | **Critical** | Immediate |
| 02 | AD CS Certificate Template Permits Privileged Account Impersonation | **Critical** | Immediate |
| 03 | Stored Browser Credentials Expose Reusable Domain Credentials | **High** | High |

---

# Security Findings

## Finding 01 — Unauthenticated Software Update Mechanism Allows Remote Code Execution

**Severity:** Critical  
**Affected Component:** Public Software Update Service  
**Affected Host:** `EC-FS-01`  
**Remediation Priority:** Immediate

### Description

The externally accessible software update service exposed an unauthenticated file-upload function that accepted executable components used by the application's update process.

Testing demonstrated that the server did not meaningfully validate the contents of uploaded executable components. Files containing arbitrary data were successfully accepted when submitted using the expected file extensions and upload fields.

The following security weaknesses were identified:

- No authentication was required to access the upload functionality.
- Uploaded executable content was not meaningfully validated.
- Arbitrary content could be supplied in fields intended for executable components.
- File validation relied primarily on filename and extension requirements.
- A runtime dependency required by the application could be replaced with attacker-controlled content.

Because the application subsequently loaded an attacker-controlled runtime component, the upload functionality could be abused to obtain arbitrary code execution.

### Security Impact

Successful exploitation allowed an unauthenticated external attacker to execute arbitrary code on a domain-joined Windows server.

During the assessment, exploitation resulted in:

| Field | Value |
|---|---|
| User Context | `<Domain>\<Initial-User>` |
| IP Address | `<Internal-IP>` |
| Hostname | `EC-FS-01` |

This foothold provided access to the internal Active Directory environment and enabled subsequent domain enumeration, credential access, privilege escalation, and lateral movement.

### Business Risk

An unauthenticated remote code execution condition exposed to the Internet provides a direct path from an untrusted network into the internal environment.

Successful exploitation could allow an attacker to execute arbitrary code, establish unauthorized access, access information available to the compromised identity, harvest credentials, enumerate internal systems, move laterally, and exploit additional internal weaknesses.

During this assessment, the weakness served as the initial entry point for an attack chain that ultimately resulted in complete Active Directory compromise.

### Evidence

Testing confirmed that arbitrary non-executable content could be submitted through fields intended for executable update components.

![[Pasted image 20260916203857.png]]

Successful exploitation subsequently established execution on `EC-FS-01`.

![[Pasted image 20260916201013.png]]

### Remediation

EvilCorp should:

- Require strong authentication and authorization for all software update functionality.
- Require cryptographically signed update packages.
- Verify package signatures before processing or executing uploaded components.
- Reject unsigned, modified, or otherwise untrusted executable components.
- Validate uploaded content against an explicit allowlist of expected packages and formats.
- Prevent arbitrary replacement of executable runtime dependencies.
- Run the update process using the minimum privileges required.
- Restrict unnecessary internal network access from the update service.
- Generate security alerts for unauthorized or failed update attempts.

---

## Finding 02 — AD CS Certificate Template Permits Privileged Account Impersonation

**Severity:** Critical  
**Affected Component:** `<Certificate-Template>`  
**Affected Environment:** `<Domain>`  
**Remediation Priority:** Immediate

### Description

Active Directory Certificate Services enumeration identified an authentication-capable certificate template vulnerable to ESC1 abuse.

The affected template permitted an enrollee to supply identity information while the compromised `<Domain>\<Compromised-User>` account possessed enrollment permissions.

This combination allowed an account with enrollment rights to request a certificate representing another Active Directory identity.

### Security Impact

During the assessment, the compromised account successfully obtained a certificate representing a privileged domain identity.

The resulting certificate was used to obtain privileged Kerberos authentication, after which administrative access to `EC-DC-01` was verified.

This demonstrated a direct privilege-escalation path from compromise of a standard domain account to domain-level administrative access.

### Business Risk

An attacker who compromises an account capable of enrolling in the affected certificate template may be able to impersonate privileged Active Directory identities.

Successful exploitation could result in:

- Impersonation of privileged domain identities.
- Administrative access to Domain Controllers.
- Unauthorized access to sensitive organizational systems.
- Modification of privileged identities or security policies.
- Persistence within Active Directory.
- Complete compromise of the Windows domain.

During the assessment, exploitation of the affected certificate configuration directly enabled progression from a compromised standard domain account to privileged Domain Controller access.

### Evidence

AD CS enumeration identified an authentication-capable certificate template available to the compromised domain identity.

> **Evidence:** `evidence-07-adcs-enumeration.png`

A certificate representing a privileged domain identity was successfully issued.

![[Pasted image 20260916201346.png]]

Certificate-based authentication subsequently resulted in privileged Kerberos authentication.

![[Pasted image 20260916201548.png]]

Administrative access to the Domain Controller was then verified.

![[Pasted image 20260916203541.png]]

### Remediation

EvilCorp should:

- Remove unnecessary enrollee-supplied identity functionality from affected certificate templates.
- Prevent nonprivileged users from supplying arbitrary identities in certificate requests.
- Restrict certificate enrollment permissions to explicitly authorized users and groups.
- Review whether authentication-capable Extended Key Usages are required.
- Require certificate manager approval or authorized signatures for sensitive enrollment workflows where appropriate.
- Audit all published certificate templates for similar privilege-escalation conditions.
- Monitor certificate issuance involving privileged identities.
- Review previously issued certificates for unexpected privileged enrollment.
- Review AD CS event logs for anomalous certificate requests.

---

## Finding 03 — Stored Browser Credentials Expose Reusable Domain Credentials

**Severity:** High  
**Affected Host:** `EC-FS-01`  
**Affected Identity:** `<Domain>\<Compromised-User>`  
**Remediation Priority:** High

### Description

Reusable domain credentials belonging to `<Domain>\<Compromised-User>` were recoverable from browser credential storage on the compromised `EC-FS-01` system.

Following compromise of the endpoint, browser credential enumeration exposed credentials associated with an additional domain identity.

The recovered credentials were subsequently validated and provided access in the context of the additional account.

### Security Impact

Credential exposure expanded access beyond the initially compromised `<Initial-User>` identity.

The newly compromised account possessed certificate enrollment permissions that exposed the AD CS privilege-escalation path described in **Finding 02**.

The browser-stored credential therefore served as an important link in the overall attack chain.

### Business Risk

Storage of reusable organizational credentials on endpoints increases the potential impact of endpoint compromise.

An attacker with access to an endpoint may recover credentials belonging to additional users and use them to access internal resources, move laterally, or identify privilege-escalation paths associated with newly compromised identities.

The risk is increased when exposed accounts possess permissions over sensitive identity infrastructure.

### Evidence

Browser credential enumeration identified credentials associated with an additional domain identity.

![[Pasted image 20260916201823.png]]

The recovered credentials were subsequently validated and used to establish access in the affected user's context.

### Remediation

EvilCorp should:

- Review organizational policies governing browser password storage.
- Disable or restrict storage of organizational credentials in browsers where appropriate.
- Prefer managed authentication mechanisms that minimize exposure of reusable passwords.
- Apply least privilege to domain accounts.
- Restrict sensitive identities from unnecessary use on shared or server systems.
- Monitor anomalous authentication following endpoint compromise.
- Reset credentials identified as exposed during security incidents.

---

# Technical Attack Narrative

The following section documents how the identified security weaknesses were chained during the assessment.

The purpose of this section is to provide the technical sequence supporting the findings while separating the chronological attack narrative from the individual security weaknesses and their associated business risk.

---

## Phase 1 — External Reconnaissance

### Public Service Enumeration

The external target was enumerated to identify exposed services and potential initial access vectors.

```bash
nmap -sT -Pn --top-ports 100 <External-IP>
```

TCP/8080 was identified as open.

A full TCP scan was subsequently performed to determine whether additional services were exposed:

```bash
sudo nmap -T4 -p- -vv <External-IP>
```

The scan confirmed TCP/8080 as the only exposed TCP service identified during testing.

![[Pasted image 20260916202012.png]]

### Infrastructure Enumeration

Publicly available network information was reviewed to establish basic context around the exposed service.

Identifying infrastructure information has been omitted from this public case study.

### Web Content Discovery

Content discovery was performed against the exposed application to identify additional accessible routes.

```bash
ffuf -u http://<External-IP>:8080/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc all -fc 404
```

### Application Enumeration

Browsing to the exposed service revealed a custom software update application.

![[Pasted image 20260916202141.png]]

Technology fingerprinting identified a Python-based web application.

The application's software update functionality was selected for further testing because it accepted executable components supplied by the user.

---

## Phase 2 — Initial Access

### Upload Validation Testing

The software update functionality accepted multiple uploaded components, including executable runtime dependencies.

Testing demonstrated that arbitrary content could be submitted using the expected upload fields and file extensions.

![[Pasted image 20260916202326.png]]

The following conditions were identified:

```text
No authentication required
Insufficient executable-content validation
Arbitrary content accepted
Extension-based validation
```

These conditions formed the basis of **Finding 01**.

### Initial Execution Testing

Initial testing evaluated multiple application components to determine whether replacement resulted in execution.

Although one replacement component was accepted by the application, no execution was observed through that path.

Testing subsequently focused on a required runtime dependency loaded during application startup.

### Identifying the Execution Path

A custom proxy library was developed to demonstrate the security impact of replacing the required runtime dependency.

The proof of concept preserved the runtime exports required by the legitimate application, forwarded legitimate API calls to the original runtime, and executed the authorized assessment payload when loaded.

The resulting execution path was:

```text
Unauthenticated Upload
        |
        v
Attacker-Controlled Update Accepted
        |
        v
Runtime Dependency Replaced
        |
        v
Application Loads Replacement Library
        |
        v
Legitimate Runtime Calls Forwarded
        |
        v
Assessment Payload Executes
        |
        v
Initial Foothold Established
```

### Initial Foothold

Successful exploitation established execution with the following context:

| Field | Value |
|---|---|
| User Context | `<Domain>\<Initial-User>` |
| IP Address | `<Internal-IP>` |
| Hostname | `EC-FS-01` |

![[Pasted image 20260916202453.png]]

At this stage, an unauthenticated external attacker had obtained access to an internal domain-joined system.

---

## Phase 3 — Internal Enumeration

Following compromise of `EC-FS-01`, host and Active Directory enumeration was performed to identify the compromised environment, current privileges, and potential paths for expanding access.

### Host and Domain Context

Network configuration, current-user privileges, domain information, domain computers, privileged groups, Organizational Units, and Group Policy were reviewed.

![[Pasted image 20260916202649.png]]
![[Pasted image 20260916202754.png]]
![[Pasted image 20260916202904.png]]
![[Pasted image 20260916202946.png]]

Enumeration identified standard domain identities, service accounts, privileged identities, and the identity associated with the initial foothold.

### Service Principal Names

Domain accounts with Service Principal Names were enumerated to identify potential service-account attack paths.

A service account configured with an SPN was identified and evaluated as a potential Kerberoasting target.

A Kerberos service ticket associated with the account was obtained and subjected to offline password cracking.

The password was not recovered during the assessment, and this path was not used to progress the compromise.

### Delegation

Active Directory delegation configurations were reviewed for potential privilege-escalation or lateral-movement opportunities.

No useful delegation path was required for the demonstrated compromise.

---

## Phase 4 — Credential Access

### Browser Credential Exposure

Credential storage on `EC-FS-01` was examined following initial compromise.

Browser credential enumeration identified reusable credentials associated with another domain identity:

```text
<Domain>\<Compromised-User>
```

![[Pasted image 20260916203046.png]]

This condition formed **Finding 03 — Stored Browser Credentials Expose Reusable Domain Credentials**.

### Additional User Access

The recovered credentials were validated and access was established in the context of the additional domain account.

The permissions associated with the compromised identity exposed the next stage of the attack path.

---

## Phase 5 — Privilege Escalation

### Active Directory Certificate Services Enumeration

Active Directory Certificate Services was enumerated to identify certificate templates available to the compromised domain identity.

Enumeration identified an authentication-capable certificate template that permitted enrollee-supplied identity information.

The compromised identity possessed enrollment permissions over the affected template.

The configuration met the conditions for ESC1 abuse and formed **Finding 02 — AD CS Certificate Template Permits Privileged Account Impersonation**.

> **Evidence:** `evidence-07-adcs-enumeration.png`

### Privileged Certificate Enrollment

The affected template was used to determine whether the compromised account could obtain a certificate representing a privileged domain identity.

A privileged certificate was successfully issued.

![[Pasted image 20260916203206.png]]

### Privileged Authentication

The resulting certificate was prepared for authentication and used to obtain Kerberos authentication associated with the privileged identity.

![[Pasted image 20260916203448.png]]

### Domain Controller Access

Administrative access to `EC-DC-01` was subsequently verified.

![[Pasted image 20260916203529.png]]

At this stage, the assessment demonstrated escalation from compromise of a standard domain identity to privileged Domain Controller access.

---

## Phase 6 — Domain Compromise

### Lateral Movement

Following successful privileged authentication, lateral movement was performed to demonstrate execution on `EC-DC-01`.

Several execution attempts occurred before a successful session was established.

The primary cause was an incorrectly configured C2 listener that resulted in assessment payloads attempting to communicate using an unavailable listener. Endpoint security also detected artifacts associated with some execution attempts.

Following correction of the C2 configuration, authorized administrative access was used to establish execution on the Domain Controller.

### Domain Controller Session

A successful session was established with the following context:

| Field | Value |
|---|---|
| User Context | `<Domain>\EC-DC-01$ (NT AUTHORITY\SYSTEM)` |
| IP Address | `<DC-IP>` |
| Hostname | `EC-DC-01` |
| OS | Windows Server 2016 |

![[Pasted image 20260916205513.png]]

SYSTEM-level execution on `EC-DC-01` demonstrated complete compromise of the EvilCorp Active Directory environment.

---

# Cleanup

Following completion of testing, cleanup was performed to remove artifacts introduced during the assessment and restore security controls modified during testing.

Cleanup activities included:

- Removal of temporary assessment files from `EC-DC-01`.
- Removal of temporary service artifacts.
- Removal of scheduled execution artifacts created during testing.
- Restoration of endpoint security controls modified during testing.
- Restoration of system services modified during the assessment to their previous configuration.

Cleanup was limited to artifacts introduced specifically by the authorized assessment to avoid unnecessary modification of the underlying environment.

---

# Remediation Roadmap

The demonstrated attack chain provides a clear order in which remediation should be prioritized.

## Immediate — Secure the Public Software Update Mechanism

The public software update service represents the highest external exposure because exploitation required no initial credentials.

EvilCorp should:

- Require authentication and authorization.
- Require cryptographically signed update packages.
- Validate package integrity before installation.
- Prevent arbitrary replacement of executable dependencies.
- Apply least privilege to the update process.
- Restrict unnecessary internal network access from the application.
- Monitor update activity for unexpected or unauthorized changes.

Remediation of this finding would eliminate the initial access vector demonstrated during the assessment.

## Immediate — Remediate the Vulnerable AD CS Template

The vulnerable certificate template provided a direct path from compromise of a standard domain account to privileged domain authentication.

EvilCorp should:

- Remove unnecessary enrollee-supplied identity functionality.
- Restrict certificate enrollment permissions.
- Review authentication-capable Extended Key Usages.
- Review all published certificate templates for similar configurations.
- Monitor certificate enrollment involving privileged identities.
- Review previously issued certificates for anomalous privileged enrollment.

Remediation of this finding would disrupt the demonstrated privilege-escalation path.

## High — Reduce Reusable Credential Exposure

Browser-stored credentials allowed compromise of an additional domain identity.

EvilCorp should:

- Review enterprise browser password-storage policies.
- Reduce storage of reusable organizational credentials.
- Apply least privilege to domain identities.
- Restrict sensitive identities from unnecessary endpoint use.
- Monitor for credential reuse following endpoint compromise.

## Improve Detection Coverage

The demonstrated attack path provides several opportunities for defensive monitoring.

Detection opportunities include:

- Unauthorized interaction with software update functionality.
- Unexpected executable-component replacement.
- Credential access from endpoints.
- Suspicious certificate enrollment involving privileged identities.
- Abnormal privileged Kerberos authentication.
- Remote service creation.
- Scheduled execution on sensitive systems.
- Changes to endpoint security configuration.

---

# Conclusion

The EvilCorp red team assessment demonstrated how multiple security weaknesses could be chained to transform an unauthenticated external attack into complete Active Directory compromise.

The most significant weaknesses identified during testing were the unauthenticated software update mechanism and the vulnerable Active Directory Certificate Services configuration.

The software update service provided the initial foothold on `EC-FS-01`. Browser-stored credentials subsequently exposed an additional domain identity whose certificate enrollment permissions provided access to a vulnerable authentication-capable certificate template.

Abuse of this template allowed privileged certificate enrollment and Kerberos authentication. Administrative access to `EC-DC-01` was then established, culminating in SYSTEM-level execution on the Domain Controller.

The assessment demonstrates why vulnerabilities should not be evaluated solely as isolated technical issues. Individual weaknesses can substantially increase organizational risk when combined with credential exposure, excessive permissions, and identity infrastructure misconfigurations.

EvilCorp should prioritize remediation of the public update mechanism and AD CS configuration, followed by improvements to credential handling and detection coverage.

---

# Appendix A — Tooling

The following tooling was used during the authorized assessment.

| Tool | Purpose | Modifications |
|---|---|---|
| Sliver C2 | Command-and-control and assessment session management | None |
| Rubeus | Kerberos security testing | None |
| Certify | Active Directory Certificate Services enumeration | None |
| SharpView | Active Directory enumeration | None |
| ffuf | Web content discovery | None |
| Hashcat | Offline password-cracking attempt | None |
| Custom Proxy Library | Initial-access proof of concept | Purpose-built for assessment |

---

# Appendix B — Custom Proxy Library

A custom proxy library was developed to demonstrate the security impact of the insufficient upload validation identified in **Finding 01**.

The application relied on an executable runtime dependency during startup.

The proof-of-concept proxy preserved the exports required by the legitimate application and forwarded legitimate API calls to the original runtime.

The proxy additionally executed the authorized assessment payload when loaded by the application.

The resulting execution flow was:

```text
Application Process
        |
        v
Attacker-Controlled Runtime Library
        |
        +----> Assessment Payload
        |
        v
Legitimate Runtime
```

This demonstrated that allowing an unauthenticated user to replace a required executable dependency could result in arbitrary code execution while preserving the application's expected runtime functionality.

Successful exploitation resulted in the initial foothold on `EC-FS-01`.

Implementation details specific to the original assessment have intentionally been omitted from this public case study.

---

# Appendix C — Evidence Index

All supporting screenshots included in this public case study have been sanitized to remove or replace original organization names, domain names, hostnames, IP addresses, usernames, credentials, hashes, authentication material, command-and-control infrastructure, assessment instructions, flags, and exercise-specific answers.

| Evidence                             | Description                                 |
| ------------------------------------ | ------------------------------------------- |
| ![[Pasted image 20260916203746.png]] | External TCP enumeration                    |
| ![[Pasted image 20260916203851.png]] | Public software update application          |
| `evidence-03-upload-validation.png`  | Upload validation testing                   |
| ![[Pasted image 20260916203916.png]] | Initial foothold on `EC-FS-01`              |
| ![[Pasted image 20260916203933.png]] | Active Directory enumeration                |
| ![[Pasted image 20260916203954.png]] | Browser credential exposure                 |
| `evidence-07-adcs-enumeration.png`   | AD CS certificate-template enumeration      |
| ![[Pasted image 20260916204004.png]] | Privileged certificate enrollment           |
| ![[Pasted image 20260916204019.png]] | Privileged certificate-based authentication |
| ![[Pasted image 20260916204023.png]] | Administrative Domain Controller access     |
| ![[Pasted image 20260916210447.png]] | SYSTEM-level execution on `EC-DC-01`        |

---

> [!NOTE]
> Assessment-specific objectives, flags, instructions, credentials, authentication material, command-and-control infrastructure, original infrastructure identifiers, and other exercise-specific information have intentionally been excluded or sanitized.
