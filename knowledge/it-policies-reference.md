# IT Policies Reference

## Knowledge Base for IT Policy Advisor Agent

---

### Agent Behavior Model

This agent operates as a **fully generative Connected Agent** with no predefined Topics. It uses the comprehensive IT policies below as its knowledge foundation, but conducts a natural, conversational interaction with the user.

**Conversation approach:**
- Answer policy questions directly and clearly, citing the specific policy section
- When users ask "Can I...?" or "Is it allowed to...?" — give a definitive YES/NO with the policy rationale
- For security incidents (phishing clicked, lost device, etc.): provide IMMEDIATE action steps first from Section 8, THEN suggest creating a P1 ticket
- When a policy answer naturally leads to an action (e.g., "Yes, you can request Visio" → "Would you like to request it?"), prepare routing context only if a handoff is needed
- For data classification questions: help the user determine the correct classification level and explain handling rules
- Do not lecture — be concise and practical
- Do not repeat the same guidance or send duplicate messages unless the user asks
**Teams rule (mandatory):** The agent runs in Teams chat. Avoid duplicate replies. For Adaptive Card submits, process `value.action` and ignore plain `text`.

**Context received from Orchestrator:** User's policy question or security concern

**Context returned to Orchestrator:** POLICY ADVISORY SUMMARY only when a cross-agent continuation/handoff is needed (see Section 12)

---

## 1. Password & Authentication Policy

### 1.1 Password Requirements
| Parameter | Requirement |
|-----------|-------------|
| Minimum length | 12 characters |
| Complexity | At least 3 of: uppercase, lowercase, numbers, special characters |
| Maximum age | 90 days (prompted to change) |
| Minimum age | 1 day (no immediate re-change) |
| History | Cannot reuse last 12 passwords |
| Lockout threshold | 5 failed attempts |
| Lockout duration | 30 minutes (auto-unlock) |
| Self-service reset | Enabled via SSPR portal |

### 1.2 Prohibited Password Patterns
- Company name or variations (e.g., "Company2024!")
- Personal info (name, birthday, employee ID)
- Sequential characters ("123456", "abcdef")
- Common dictionary words ("Password", "Welcome")
- Previously breached passwords (checked against known breach lists)

### 1.3 Multi-Factor Authentication (MFA)
- **Mandatory** for all employees, no exceptions
- **Required for:** All Microsoft 365 services, VPN, all cloud applications
- **Preferred method:** Microsoft Authenticator app (push notification)
- **Backup methods:** SMS, phone call, FIDO2 security key
- **Remember device:** Allowed for 30 days on compliant, managed devices only
- **Guest accounts:** MFA required for all external guests accessing company resources

### 1.4 Single Sign-On (SSO)
- All company applications should use Azure AD SSO where available
- Employees must NOT create separate accounts for SSO-enabled services
- Personal accounts must not be linked to company SSO

---

## 2. Acceptable Use Policy (AUP)

### 2.1 General Principles
- Company IT resources are provided **primarily for business purposes**
- Limited personal use is permitted during breaks, provided it does not:
  - Interfere with work duties
  - Consume excessive bandwidth
  - Create legal liability or security risks
  - Violate any other company policy

### 2.2 Acceptable Activities
- ✅ Business email and communication
- ✅ Research related to job duties
- ✅ Professional development and online learning
- ✅ Brief personal web browsing during breaks
- ✅ Reasonable personal email access (webmail)
- ✅ Using approved collaboration tools
- ✅ Cloud storage for business files (OneDrive, SharePoint)

### 2.3 Prohibited Activities
- ❌ Accessing or distributing illegal content
- ❌ Unauthorized file sharing or torrenting
- ❌ Cryptocurrency mining
- ❌ Running personal businesses on company equipment
- ❌ Gambling websites
- ❌ Excessive streaming (Netflix, YouTube binge-watching during work)
- ❌ Unauthorized remote access tools
- ❌ Bypassing security controls (proxy tools, personal VPNs)
- ❌ Storing personal files on company OneDrive/SharePoint
- ❌ Sharing company credentials with anyone
- ❌ Connecting unauthorized devices to the corporate network
- ❌ Disabling or uninstalling security software

### 2.4 Monitoring Notice
- Company reserves the right to monitor all usage of company IT resources
- This includes: email, web browsing, file access, application usage
- Monitoring is performed to protect company assets and ensure compliance
- Employees have no expectation of privacy when using company resources
- All data on company devices is company property

---

## 3. BYOD (Bring Your Own Device) Policy

### 3.1 Scope
This policy applies to personal devices used to access company resources, including:
- Personal smartphones and tablets
- Personal laptops (where enabled)
- Personal desktop computers (remote access only)

### 3.2 Requirements for Personal Devices
| Requirement | Detail |
|-------------|--------|
| OS Version | iOS 16+ / Android 12+ / Windows 11 / macOS 13+ |
| Screen Lock | Required (PIN 6+, password, biometric) |
| Encryption | Must be enabled (auto on iOS; verify Android/Windows) |
| Antivirus | Required on Windows/macOS devices |
| MDM Enrollment | Required via Microsoft Intune Company Portal |
| OS Updates | Must be installed within 14 days of release |
| Jailbreak/Root | Absolutely prohibited |

### 3.3 What Company Can See on Your Personal Device
- ✅ Device type and OS version
- ✅ Compliance status (meets policy requirements)
- ✅ App inventory (managed apps only)
- ✅ Device name

### 3.4 What Company CANNOT See on Your Personal Device
- ❌ Personal photos, videos, messages
- ❌ Browsing history
- ❌ Personal email
- ❌ Personal app data
- ❌ Location (unless device reported lost)
- ❌ Call history or SMS

### 3.5 Remote Wipe
- Company can perform **selective wipe** (company data only) on BYOD devices
- Company data includes: Outlook email, Teams data, managed apps, OneDrive (work)
- Personal data is NOT affected by selective wipe
- Full wipe is ONLY performed on company-owned devices

### 3.6 Support
- IT HelpDesk supports company apps/access on BYOD devices
- IT does NOT provide support for personal apps, personal OS issues, or hardware repair
- If a personal device is lost/stolen with company data: report immediately to IT Security

---

## 4. Data Classification & Handling Policy

### 4.1 Classification Levels

#### 🔴 CONFIDENTIAL
**Definition:** Information that, if disclosed, could cause significant harm to the company, its employees, or its customers.

**Examples:**
- Customer personal data (PII)
- Financial reports (pre-publication)
- Strategic plans and M&A activities
- Employee salary and HR records
- Security credentials and keys
- Legal documents under privilege
- Source code of core products

**Handling rules:**
- Must be encrypted at rest and in transit
- Access on need-to-know basis only
- Do NOT store on personal devices
- Do NOT share via personal email or unapproved cloud services
- Do NOT print unless absolutely necessary — secure printing only
- Must be labeled "CONFIDENTIAL" in document headers/footers
- Retention and deletion per Records Retention Policy

#### 🟡 INTERNAL
**Definition:** Information intended for internal company use that should not be shared externally without authorization.

**Examples:**
- Internal procedures and policies
- Internal presentations and meeting notes
- Organizational charts
- Internal project documentation
- Internal communications
- Non-sensitive business data

**Handling rules:**
- Can be shared freely within the company
- Do NOT share externally without manager approval
- Can be stored on company devices and OneDrive
- Should be labeled "INTERNAL" when practical
- Follow standard retention policies

#### 🟢 PUBLIC
**Definition:** Information that is approved for public distribution.

**Examples:**
- Published marketing materials
- Public website content
- Press releases
- Published annual reports
- Job postings
- Open-source project documentation

**Handling rules:**
- Can be shared freely
- Must be approved by Communications/Marketing before publication
- No special handling required once approved

### 4.2 Data Handling Quick Reference

| Action | Confidential | Internal | Public |
|--------|-------------|----------|--------|
| Email internally | ✅ (encrypted) | ✅ | ✅ |
| Email externally | ❌ (unless encrypted + approved) | ⚠️ Manager approval | ✅ |
| Store on OneDrive | ✅ (company only) | ✅ | ✅ |
| Store on personal device | ❌ | ❌ | ✅ |
| Print | ⚠️ Secure print only | ✅ | ✅ |
| Share via Teams | ✅ (company Teams) | ✅ | ✅ |
| Post on SharePoint | ✅ (restricted site) | ✅ | ✅ |
| Upload to personal cloud | ❌ | ❌ | ✅ |
| Discuss by phone in public | ❌ | ⚠️ Be cautious | ✅ |

---

## 5. Remote Work / Work from Home Policy

### 5.1 Eligibility
- All employees whose role permits remote work, as approved by their manager
- Standard arrangement: Hybrid (2-3 days office, 2-3 days remote)
- Full remote: requires VP approval and HR confirmation

### 5.2 IT Requirements for Remote Work

| Requirement | Detail |
|-------------|--------|
| Internet | Minimum 25 Mbps download / 5 Mbps upload |
| VPN | Must connect via company VPN for internal resources |
| Device | Company-provided laptop (preferred) or approved BYOD |
| WiFi Security | WPA2 or WPA3 encryption (NOT open WiFi) |
| Physical Security | Screen lock when away; no unauthorized viewers |
| Software Updates | Must keep all software up to date |
| Backup | Business files on OneDrive (auto-sync) |

### 5.3 Prohibited Remote Work Practices
- ❌ Using public / open WiFi without VPN
- ❌ Working from shared / public computers (internet cafés, libraries)
- ❌ Leaving laptop unattended in public places
- ❌ Allowing family members to use work device
- ❌ Printing confidential documents at home (unless approved)
- ❌ Using personal USB drives with company data

### 5.4 International Remote Work
- Working from another country requires advance approval (HR + IT Security + Legal)
- Data residency laws may restrict access to certain systems
- Tax implications must be reviewed by HR
- VPN may not work in certain countries (regional restrictions)

---

## 6. Email & Communication Policy

### 6.1 Appropriate Use
- Business email is an official communication channel
- All business email is company property and may be audited
- Use professional tone and language in all communications
- Include appropriate disclaimers in external communications (auto-appended)

### 6.2 Email Security Rules
- **DO NOT** open attachments from unknown senders
- **DO NOT** click links in suspicious emails
- **DO NOT** forward chain emails, hoaxes, or unverified warnings
- **DO NOT** auto-forward company email to personal email accounts
- **DO NOT** send confidential data via unencrypted email
- **DO** report suspicious emails using the "Report Phishing" button in Outlook
- **DO** verify unexpected requests from executives (CEO fraud/whaling protection)

### 6.3 Email Retention
- Standard retention: 7 years for business email
- Legal hold: Some emails may be preserved for legal proceedings
- Deleted emails: Recoverable for 30 days, then permanently deleted
- Archive: Available for E5 users (unlimited archive storage)

### 6.4 Email Size Limits
| Limit | Value |
|-------|-------|
| Max message size (internal) | 150 MB |
| Max message size (external) | 25 MB |
| Max attachment size | 25 MB per attachment |
| Mailbox size (E3) | 50 GB |
| Mailbox size (E5) | 100 GB |
| Archive mailbox | 50 GB (auto-expanding for E5) |

**For large files:** Use OneDrive links instead of attachments

---

## 7. Software & Licensing Policy

### 7.1 General Rules
- Only **approved software** may be installed on company devices (see Software Catalog)
- Users do NOT have local admin rights by default
- All software must be installed via Company Portal (Intune) or IT request
- **No pirated, cracked, or unlicensed software** — zero tolerance
- Open-source software from the approved list is permitted
- All SaaS/cloud services must be vetted by IT Security before use

### 7.2 Shadow IT
"Shadow IT" refers to the use of unapproved applications, services, or devices for company work.

**Not allowed:**
- Signing up for SaaS services (Notion, Trello, etc.) without IT approval
- Using personal accounts for business data (personal Google Drive, Dropbox)
- Setting up unauthorized integrations or automations

**Why it matters:**
- Data may be exposed or unrecoverable
- Violates compliance requirements
- Creates security vulnerabilities
- Leads to duplicate, inconsistent tools

**What to do instead:** Request the software through SmartDesk AI. If approved, IT will provision it properly.

### 7.3 Open Source Software
- Approved open-source tools are available in the Company Portal
- New open-source tools require IT Security review (license compatibility, vulnerability assessment)
- Copyleft licenses (GPL) may have restrictions — check with Legal before using in products

---

## 8. Security Incident Response

### 8.1 What Is a Security Incident?
Any event that threatens the confidentiality, integrity, or availability of company data or systems.

**Examples:**
- Phishing email received (even if not clicked)
- Clicking a suspicious link
- Malware/virus detected
- Lost or stolen device
- Unauthorized access to data
- Data accidentally shared with wrong person
- Suspicious account activity
- Physical security breach (unauthorized office access)

### 8.2 Immediate Response Steps

#### 🎣 Phishing / Suspicious Email
1. **DO NOT** click any links or open any attachments
2. **DO NOT** reply to the email
3. **DO NOT** forward it to others (except IT Security)
4. Click the **"Report Phishing"** button in Outlook
5. If you already clicked a link: **IMMEDIATELY** change your password and contact IT Security
6. Report to SmartDesk AI → creates P1 security incident ticket

#### 💻 Malware / Virus Alert
1. **Disconnect from network** (unplug Ethernet, disable WiFi)
2. Do NOT turn off the computer (preserves evidence)
3. Note what you were doing when the alert appeared
4. Contact IT Security immediately: security@company.com / ext. 5555
5. Report via SmartDesk AI → P1 incident ticket

#### 📱 Lost or Stolen Device
1. Report **immediately** — do not wait
2. Contact IT Security: security@company.com / ext. 5555
3. Report via SmartDesk AI → P1 incident ticket
4. IT will remotely wipe the device
5. File a police report if stolen
6. Change all passwords that were saved on the device

#### 🔓 Unauthorized Access
1. Change your password immediately
2. Review recent account activity in https://myaccount.microsoft.com
3. Contact IT Security
4. Report via SmartDesk AI

#### 📤 Accidental Data Disclosure
1. Contact the unintended recipient and ask them to delete the data
2. Report to your manager
3. Report to IT Security / Data Protection Officer
4. Document what data was shared and with whom

### 8.3 Incident Severity Levels

| Severity | Description | Response Time | Examples |
|----------|-------------|---------------|---------|
| P1 Critical | Active breach, data loss in progress | 15 minutes | Ransomware, active intrusion, mass data leak |
| P2 High | Confirmed security event, contained | 1 hour | Phishing clicked + credentials entered, malware found on device |
| P3 Medium | Potential security event, needs investigation | 4 hours | Suspicious email reported, unusual login attempt |
| P4 Low | Informational, no immediate threat | Next business day | Phishing email received (not acted on), policy question |

---

## 9. Privacy & Data Protection

### 9.1 Principles
- Collect only the minimum personal data necessary
- Use personal data only for its stated purpose
- Store personal data securely (encrypted, access-controlled)
- Retain personal data only as long as needed
- Delete personal data when no longer required
- Report any breach of personal data within 24 hours

### 9.2 Employee Responsibilities
- Handle customer/employee data with care appropriate to its classification
- Do not access personal data you don't need for your role
- Report any suspected data breaches immediately
- Complete annual Data Protection training
- Follow data retention schedules

### 9.3 Data Subject Rights
Employees and customers may exercise these rights:
- Right to access their personal data
- Right to rectification (correction)
- Right to erasure (in certain circumstances)
- Right to restrict processing
- Right to data portability
- Right to object to processing

**For data subject requests:** Forward to privacy@company.com

---

## 10. Physical Security Policy

### 10.1 Workspace Security
- Lock your computer when leaving your desk (`Win + L`)
- Enable screen lock timeout (max 5 minutes)
- Do not leave sensitive documents on your desk (clean desk policy)
- Shred confidential paper documents
- Do not allow tailgating (unauthorized persons following you through secure doors)

### 10.2 Visitor Policy
- All visitors must be registered and escorted
- Visitors must wear visitor badges at all times
- Visitors do NOT have access to company WiFi (guest network only)
- Do not leave visitors unattended in secure areas

### 10.3 Device Security
- Laptops must be secured with cable lock when left in the office
- Do not leave devices visible in vehicles
- Use a privacy screen when working in public
- Report lost or stolen devices immediately (see Section 8.2)

---

## 11. Compliance Summary

| Regulation | Applicability | Key Requirement |
|------------|--------------|-----------------|
| GDPR | EU employees and customers | Data protection, consent, breach reporting (72h) |
| ISO 27001 | Company-wide | Information Security Management |
| SOC 2 | Customer-facing services | Security, availability, confidentiality |
| PCI DSS | Payment processing | Card data security |
| NIS2 | EU operations | Cybersecurity measures, incident reporting |

**Annual training:** All employees must complete Security Awareness and Data Protection training annually. Completion is tracked and reported to management.

---

## 12. Context Passing & Structured Summary

### Cross-Agent Routing Guidance

The IT Policy Advisor often answers questions that lead to actions handled by other agents. Use the POLICY ADVISORY SUMMARY to signal when routing is appropriate:

| User Question Pattern | Policy Answer | Suggested Routing |
|----------------------|---------------|-------------------|
| "Can I install X?" → Yes, it's approved | Confirm policy allows it | Access Provisioner (to request installation) |
| "Can I use my personal device?" → Yes, with conditions | Explain BYOD requirements | Access Provisioner (for Intune enrollment) |
| "I clicked a phishing link" | Provide emergency steps (Section 8.2) | Ticket Manager (P1 security incident) |
| "How do I classify this document?" | Explain classification levels | None (informational only) |
| "I lost my laptop" | Provide emergency steps | Ticket Manager (P1 security incident) |
| "Can I work from another country?" | Explain international remote work policy | None (suggest HR + IT Security approval) |

### POLICY ADVISORY SUMMARY Format

Only when transitioning context back to the Orchestrator for routing/continuation, compile the following structured summary:

```
===== POLICY ADVISORY SUMMARY =====
Policy Area: {Password & Auth|Acceptable Use|BYOD|Data Classification|Remote Work|Email & Communication|Software & Licensing|Security Incident|Privacy & Data Protection|Physical Security|Compliance}
Question: {user's original question, paraphrased}
Answer Summary: {concise answer — 1-3 sentences}
Action Required: {None|Ticket needed|Route to another agent|Contact IT Security|Contact HR|Contact Legal}
Urgency: {Normal|Urgent — security incident}
Cross-Agent Routing: {None|Ticket Manager|Access Provisioner|Security Coach}
Routing Context: {if routing suggested, brief context for the next agent to use}
===== END SUMMARY =====
```

**Important:** Do not post this as a system-style summary after every response. Use it only when policy advice must be handed off to another agent/workflow.

**Display rule:** This summary is internal handoff metadata. Do NOT display raw summary blocks to the user in normal chat.
