# IT HelpDesk Ticket Playbook

## Knowledge Base for Ticket Manager Agent

---

### Agent Behavior Model

This agent operates as a **fully generative Connected Agent** with no predefined Topics. It uses the ticket lifecycle, templates, and SLA procedures below as its knowledge foundation, but conducts a natural, conversational interaction with the user.

**Conversation approach:**
- For ticket creation: gather required fields conversationally (ask one or two questions at a time, not a long form)
- If a DIAGNOSTIC SUMMARY is received from the IT Troubleshooter via the Orchestrator, auto-populate the ticket body — do NOT re-ask the user for information already captured
- Use Adaptive Cards (Ticket Confirmation card, Ticket Status card) to present ticket details
- For status checks: retrieve ticket info from Dataverse MCP (`lop_tickets` table) and present clearly
- For escalations: confirm the reason, update priority/status fields, add escalation note
- For closures: confirm resolution with the user, prompt for satisfaction rating

**Context received from Orchestrator:**
- User's request (always)
- DIAGNOSTIC SUMMARY from IT Troubleshooter (when issue was troubleshot first)
- POLICY ADVISORY SUMMARY from IT Policy Advisor (when a security incident is reported)

**Execution model:** Ticket Manager is a full connected operational agent with direct Dataverse MCP access to `lop_tickets`.

**Create-operation rule:** Ticket Manager validates input and maps all captured fields to the Dataverse `lop_tickets` schema before executing create/update/query via Dataverse MCP.
**Schema mapping rule (mandatory):** Before the first create in a session, Ticket Manager must
call `describe_table` via MCP for `lop_tickets` to retrieve the **actual logical names** in the
connected environment. Mapping must be based on that schema output. If expected fields
are missing (e.g., `lop_Name` not found), stop and report a configuration mismatch
instead of retrying the create.

**Write verification gate (mandatory):** Confirm "ticket created" only after Dataverse MCP returns success with `TicketNumber` and/or row ID. If the write fails, times out, or returns no identifier, do not claim success; tell the user creation did not complete and offer an immediate retry.

**Persistence check (mandatory):** After create, run an immediate read-back lookup by returned row ID (or `TicketNumber`) to confirm the row exists in `lop_tickets`. If lookup fails, treat creation as failed and offer retry.

**User-experience rule:** Never expose technical implementation details (MCP operations, payload schema, internal metadata) in user-visible chat. Always present plain-language outcomes and next steps.
**Teams rule (mandatory):** The agent runs in Teams chat. Avoid duplicate replies. For Adaptive Card submits, process `value.action` and ignore plain `text`.
**Confirmation gate (mandatory):** If waiting for user confirmation or missing required fields, do NOT claim any create/update occurred. Only say "created" after MCP create success AND read-after-write verification.

**Dataverse create payload mapping (logical names):**
- Use the **actual logical names** from Dataverse. If a logical name does not exist,
  the create will fail.
- `lop_Name` (Display name: Name) → generate a descriptive title (`[Category] concise issue title`)
- `lop_Category` (Display name: Category) → one of the configured category values
- `lop_Type` (Display name: Type) → one of: `Incident`, `Request`, `Question`, `Change`, `Access request`, `Software install request`, `License assignment`, `Account administration`
- `lop_SLA` (Display name: Priority) → one of: `P1-Critical`, `P2-High`, `P3-Medium`, `P4-Low`
- `lop_Status` (Display name: Status) → set to `new` on create
- `lop_Description` (Display name: Description) → generated full narrative including already performed troubleshooting steps and their outcomes
- `lop_ResolutionNotes` (Display name: ResolutionNotes) → only on close
- `lop_TicketNumber` (Display name: TicketNumber) → auto-numbered (do not set on create)
- `lop_TicketsId` (Display name: Tickets) → system ID (do not set on create)
- `statecode` (Display name: Status) → system field (do not set on create unless required)
- `statuscode` (Display name: Status Reason) → system field (do not set on create unless required)
If `SlaTarget` exists, set it based on priority (per Section 2).

**Context returned to Orchestrator:** TICKET OPERATION SUMMARY only when a cross-agent continuation/handoff is needed (see Section 11)
Always provide a user-facing reply; if a handoff summary is required, emit it separately from the user message (never as the only content).

---

## 1. Ticket Lifecycle

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  CREATE   │────▶│  TRIAGE   │────▶│  ASSIGN   │────▶│ WORKING  │────▶│ RESOLVED │
│  (New)    │     │(Classify) │     │(Route)    │     │(In Prog) │     │(Pending) │
└──────────┘     └──────────┘     └──────────┘     └──────────┘     └──────────┘
                                                         │                │
                                                         ▼                ▼
                                                   ┌──────────┐     ┌──────────┐
                                                   │ ESCALATE  │     │  CLOSED  │
                                                   │(Higher Lvl)│     │(Verified)│
                                                   └──────────┘     └──────────┘
```

### Lifecycle Stages

| Stage | Ticket `Status` Value | Description |
|-------|-------------|-------------|
| New | `new` | Ticket just created, awaiting triage |
| Triaged | `triaged` | Classified and prioritized |
| Assigned | `assigned` | Assigned to IT team member |
| In Progress | `in-progress` | Actively being worked on |
| Waiting on User | `waiting-on-user` | Agent asked follow-up, waiting for response |
| Waiting on Vendor | `waiting-on-vendor` | Depends on external vendor action |
| Escalated | `escalated` | Moved to higher support tier |
| Resolved | `resolved` | Fix applied, pending user confirmation |
| Closed | `closed` | User confirmed resolution, ticket closed |

---

## 2. Priority Classification

### P1 — Critical 🔴
**Definition:** Complete business service outage or security incident affecting multiple users or critical systems.

**Examples:**
- Email system down for entire organization
- Active security breach / ransomware
- Critical server/application offline
- Data loss in progress
- Network outage affecting building/floor

**SLA:**
| Metric | Target |
|--------|--------|
| Initial Response | 15 minutes |
| Status Updates | Every 30 minutes |
| Resolution Target | 4 hours |
| Escalation | Automatic after 1 hour without progress |

**Dataverse fields:** `Priority = P1-Critical`, `SlaTarget` set to 4-hour resolution window

---

### P2 — High 🟠
**Definition:** Significant impact on a department or individual with no workaround, or degraded critical service.

**Examples:**
- Individual user's laptop completely non-functional
- VPN down for multiple users
- Printer serving an entire floor offline
- Software crash preventing critical work
- Employee locked out of all systems

**SLA:**
| Metric | Target |
|--------|--------|
| Initial Response | 1 hour |
| Status Updates | Every 4 hours |
| Resolution Target | 8 business hours |
| Escalation | Automatic after 4 hours without progress |

**Dataverse fields:** `Priority = P2-High`, `SlaTarget` set to 8-business-hour resolution window

---

### P3 — Medium 🟡
**Definition:** Issue affecting work productivity with a workaround available, or non-critical service request.

**Examples:**
- Outlook slow but functional
- Non-critical application error
- Printer quality issues (still printing)
- Software installation request
- Access request to a system
- General hardware issue with workaround

**SLA:**
| Metric | Target |
|--------|--------|
| Initial Response | 4 hours |
| Status Updates | Daily |
| Resolution Target | 3 business days |
| Escalation | User-requested or after 2 days without progress |

**Dataverse fields:** `Priority = P3-Medium`, `SlaTarget` set to 3-business-day resolution window

---

### P4 — Low 🟢
**Definition:** Minor issue, informational request, nice-to-have, or long-term improvement.

**Examples:**
- General IT question
- Feature request
- Non-urgent documentation update request
- Minor UI glitch that doesn't affect work
- Future software request (no urgency)
- Policy clarification

**SLA:**
| Metric | Target |
|--------|--------|
| Initial Response | 1 business day |
| Status Updates | Weekly |
| Resolution Target | 5 business days |
| Escalation | User-requested |

**Dataverse fields:** `Priority = P4-Low`, `SlaTarget` set to 5-business-day resolution window

---

## 3. Ticket Field Scheme (Dataverse `lop_tickets`)

### Category (`Category` choice field)
| Value | Description |
|-------|-------------|
| `hardware` | Hardware issues (laptop, desktop, peripherals) |
| `software` | Software issues (apps, OS, updates) |
| `network` | Network/connectivity (WiFi, VPN, internet) |
| `email` | Email and Outlook issues |
| `printing` | Printer issues |
| `security` | Security incidents |
| `access` | Access requests and permissions |
| `account` | Account, password, MFA issues |
| `mobile` | Mobile device issues (MDM, phones, tablets) |
| `other` | Other / uncategorized |

### Type (`Type` choice field)
| Value | Description |
|-------|-------------|
| `Incident` | Something is broken |
| `Request` | User requesting something |
| `Question` | User asking for information |
| `Change` | Change to existing setup |
| `Access request` | Access or permission provisioning |
| `Software install request` | Software install/provisioning request |
| `License assignment` | License assignment/update |
| `Account administration` | Account lifecycle/admin changes |

### Priority (`Priority` choice field)
| Value | Description |
|-------|-------------|
| `P1-Critical` | Critical — outage or security incident |
| `P2-High` | High — significant impact, no workaround |
| `P3-Medium` | Medium — impact with workaround |
| `P4-Low` | Low — minor or informational |

### Status (`Status` choice field)
| Value | Description |
|-------|-------------|
| `new` | New, awaiting triage |
| `triaged` | Classified and prioritized |
| `assigned` | Assigned to technician |
| `in-progress` | Actively being worked |
| `waiting-on-user` | Waiting for user info |
| `waiting-on-vendor` | Waiting for vendor |
| `escalated` | Escalated to higher tier |
| `resolved` | Fix applied, pending confirmation |
| `closed` | Verified closed |

### SLA Fields
| Field | Meaning |
|-------|---------|
| `SlaTarget` | Resolution target date/time |
| `SlaBreached` | Indicates SLA breach (`true/false`) |

---

## 4. Ticket Templates

### Template 1: General Incident
```markdown
## IT Support Ticket

**Reporter:** {user_name}
**Date:** {date}
**Priority:** {P1/P2/P3/P4}
**Category:** {hardware/software/network/email/printing/security/account}

### Description
{User's description of the issue}

### Environment
- **Device:** {device_type} — {model if known}
- **OS:** {operating_system}
- **Location:** {office/remote}
- **Application:** {affected application}

### Steps to Reproduce
1. {step 1}
2. {step 2}
3. {step 3}

### Expected Behavior
{What should happen}

### Actual Behavior
{What actually happens}

### Error Messages
{Exact error text or "None"}

### Troubleshooting Already Performed
- [ ] {Step tried and result}

### Impact
- **Users affected:** {number}
- **Business impact:** {description}
- **Workaround available:** {yes/no — describe}

### Attachments
{Screenshots, logs, etc.}
```

### Template 2: Software Request
```markdown
## Software Request

**Requester:** {user_name}
**Date:** {date}
**Priority:** P3-Medium

### Software Details
- **Software name:** {name}
- **Version:** {specific version or "Latest"}
- **License type:** {if known}

### Business Justification
{Why this software is needed for your role}

### Target Device
- **Device:** {device_name/model}
- **OS:** {operating system}
- **Disk space available:** {if known}

### Approval
- **Manager:** {manager_name}
- **Manager approved:** ⬜ Pending

### Timeline
- **Needed by:** {date}
- **Urgency:** {description}
```

### Template 3: Access Request
```markdown
## Access Request

**Requester:** {user_name}
**Date:** {date}
**Priority:** P3-Medium

### Access Details
- **System/Application:** {name}
- **Access level:** {Read/Write/Admin/specific role}
- **Access type:** {Permanent/Temporary — until date}

### Business Justification
{Why this access is needed}

### Current Access
{What access the user currently has, if any}

### Approval
- **Manager:** {manager_name}
- **System owner:** {owner_name}
- **Approved:** ⬜ Pending

### New Employee?
- **Yes/No:** {answer}
- **Start date:** {if applicable}
- **Role:** {job title}
```

### Template 4: Security Incident
```markdown
## 🚨 Security Incident Report

**Reporter:** {user_name}
**Date/Time:** {date_time}
**Priority:** P1-Critical / P2-High

### Incident Type
{phishing/malware/lost_device/unauthorized_access/data_breach/suspicious_activity}

### Description
{Detailed description of what happened}

### Timeline
- **When noticed:** {date/time}
- **When it may have started:** {date/time if different}
- **Actions taken so far:** {list}

### Affected Assets
- **Device(s):** {device details}
- **Account(s):** {email/usernames}
- **Data involved:** {type and classification}
- **Systems affected:** {list}

### Containment Status
- [ ] Device disconnected from network
- [ ] Password changed
- [ ] Incident reported to IT Security
- [ ] Manager notified

### Evidence
{Screenshots, email headers, log entries}

⚠️ **URGENCY:** This incident has been auto-flagged for immediate IT Security review.
```

### Template 5: New Employee Setup
```markdown
## New Employee IT Setup

**Requested by:** {manager_name}
**Date:** {date}
**Priority:** P3-Medium

### New Employee Details
- **Name:** {employee_name}
- **Start date:** {date}
- **Department:** {department}
- **Role:** {job_title}
- **Location:** {office/remote/hybrid}
- **Reports to:** {manager_name}

### Required Setup
#### Standard Package
- [ ] Laptop (Dell Latitude / MacBook Pro)
- [ ] Microsoft 365 E3/E5 license
- [ ] Email account
- [ ] Teams & OneDrive
- [ ] VPN client
- [ ] Intune enrollment
- [ ] Security training link

#### Role-Specific
- [ ] {additional software 1}
- [ ] {additional software 2}
- [ ] {system access 1}
- [ ] {system access 2}

### Additional Notes
{Any special requirements}

### Target: All setup complete by start date - 1 day
```

---

## 5. Escalation Procedures

### Tier 1: SmartDesk AI (Self-Service)
- Automated troubleshooting
- Ticket creation and tracking
- Knowledge base answers
- Self-service password reset guidance
- Software catalog browsing

### Tier 2: IT HelpDesk Technicians
**Escalation criteria:**
- SmartDesk AI couldn't resolve the issue
- Issue requires remote access to user's machine
- Issue requires admin-level access or configuration changes
- Hardware replacement needed

**What to include in escalation:**
- All troubleshooting steps already attempted
- Error messages and screenshots
- User's availability for remote session

### Tier 3: Specialist Teams
| Team | Handles |
|------|---------|
| Network Team | Infrastructure, switches, access points, VPN server |
| Server Team | Server issues, Active Directory, Group Policy |
| Security Team | Incidents, vulnerability response, access reviews |
| Cloud Team | Azure, M365 admin, licensing bulk changes |
| Desktop Engineering | OS imaging, driver issues, hardware refresh |

**Escalation to Tier 3 criteria:**
- Tier 2 identified root cause requiring specialist
- Infrastructure-level problem
- Security incident confirmed
- Multi-user/multi-system impact

### Tier 4: Vendor / External Support
**When to escalate externally:**
- Hardware under warranty → Manufacturer support
- SaaS application issues → Vendor support
- Microsoft 365 service issues → Microsoft Premier Support
- Network infrastructure → Managed service provider

---

## 6. SLA Breach Handling

### Breach Detection
A ticket's SLA is considered breached when resolution is not achieved within the target time:

| Priority | Resolution Target | Breach Label Applied |
|----------|------------------|---------------------|
| P1 Critical | 4 hours | `SlaBreached = true` |
| P2 High | 8 business hours | `SlaBreached = true` |
| P3 Medium | 3 business days | `SlaBreached = true` |
| P4 Low | 5 business days | `SlaBreached = true` |

### Breach Response
1. Set `SlaBreached = true` on the ticket row
2. Update `LastUpdate` with breach note and elapsed time
3. Auto-escalate:
   - P1 breach → IT Manager + CTO notification
   - P2 breach → IT Manager notification
   - P3/P4 breach → Team Lead notification
4. Assign to senior technician if not already assigned
5. Update user on status and revised timeline

---

## 7. Ticket Metrics & Dashboard

### Key Metrics
| Metric | Description | Target |
|--------|-------------|--------|
| MTTR (Mean Time to Resolve) | Average resolution time | P1: <4h, P2: <8h, P3: <3d, P4: <5d |
| First Contact Resolution | % resolved at first interaction | >60% |
| SLA Compliance | % tickets resolved within SLA | >95% |
| Customer Satisfaction | Post-resolution survey score | >4.0/5.0 |
| Open Ticket Backlog | Total currently open tickets | <50 |
| Escalation Rate | % tickets escalated from Tier 1 | <30% |

### Dashboard Queries (Dataverse `lop_tickets`)

**Open tickets by priority (examples):**
- P1: `Status ne 'closed' and Priority eq 'P1-Critical'`
- P2: `Status ne 'closed' and Priority eq 'P2-High'`
- P3: `Status ne 'closed' and Priority eq 'P3-Medium'`
- P4: `Status ne 'closed' and Priority eq 'P4-Low'`

**My open tickets:**
- `Status ne 'closed' and ReportedBy eq '{username}'`
- `Status ne 'closed' and AssignedTo eq '{username}'`

**Breached SLA tickets:**
- `Status ne 'closed' and SlaBreached eq true`

**Recently resolved:**
- `Status eq 'resolved' order by UpdatedOn desc`

**Tickets by category:**
- `Status ne 'closed' and Category eq 'hardware'`
- `Status ne 'closed' and Category eq 'network'`
- etc.

---

## 8. Communication Templates

### Initial Response
```
Hi {name},

Thank you for contacting IT Support. Your request has been logged as ticket **LOP-{number}**.

**Priority:** {priority}
**Category:** {category}
**Expected resolution:** {SLA target}

{If troubleshooting steps provided: "I've started with some troubleshooting steps. Please try the following:"}
{If assigned: "Your ticket has been assigned to {technician_name}."}
{If needs info: "To help resolve this faster, could you please provide: {info_needed}"}

You can check your ticket status anytime by asking me "What's the status of LOP-{number}?"
```

### Status Update
```
**Ticket Update — LOP-{number}**

Current status: {status}
Assigned to: {technician}
Last action: {what was done}
Next step: {what happens next}
Updated ETA: {time estimate}
```

### Resolution Notification
```
Hi {name},

Great news! Your ticket **LOP-{number}** has been resolved.

**Resolution:** {description of fix}

Please verify the fix on your end. If everything is working, I'll close the ticket. If the issue persists, let me know and I'll reopen it.

How would you rate your support experience? (1-5 stars)
```

### SLA Breach Notification
```
⚠️ **SLA Alert — LOP-{number}**

This ticket has exceeded its {priority} SLA target of {target_time}.
- **Opened:** {open_time}
- **Elapsed:** {elapsed_time}
- **Status:** {current_status}

This ticket has been auto-escalated to {escalation_target}. We apologize for the delay and are prioritizing resolution.
```

---

## 9. Ticket Numbering Convention

- **Format:** `LOP-{sequential_number}`
- **LOP** = Lukas Oplt Platform
- Number generated by Dataverse auto-number field (`TicketNumber`)
- Referenced in all communications as LOP-{number}
- Ticket title format in `Title`: `[LOP-{number}] {brief_description}`

**Example titles:**
- `[LOP-42] Laptop won't connect to VPN after Windows Update`
- `[LOP-43] Software request: Adobe Acrobat Pro for Marketing`
- `[LOP-44] Access request: SharePoint Finance Reports site`
- `[LOP-45] 🚨 SECURITY: Phishing email — credentials entered`

---

## 10. Ticket Best Practices

### For Creating Tickets
1. **Clear title:** Include the core issue in 10 words or less
2. **Detailed description:** Use the appropriate template
3. **Correct labels:** Apply category + priority + type at creation
4. **One issue per ticket:** Don't combine multiple unrelated issues
5. **Link related tickets:** If issues are connected, reference other ticket numbers

### For Updating Tickets
1. **Comment, don't edit:** Add comments for updates, keep original description intact
2. **Update labels:** Move status labels as the ticket progresses
3. **Timestamp actions:** Note when significant actions were taken
4. **Mention people:** Use @mentions for assigned technicians

### For Closing Tickets
1. **Resolution documented:** Always describe how it was resolved
2. **User confirmation:** Wait for user to confirm the fix works
3. **Auto-close:** If no response for 5 business days after resolution, auto-close with notice
4. **Satisfaction survey:** Prompt for rating on close
5. **Knowledge base update:** If the resolution addresses a common issue, update troubleshooting guide

---

## 11. Context Passing & Structured Summary

### Receiving Context from Other Agents

When the Orchestrator passes a **DIAGNOSTIC SUMMARY** from the IT Troubleshooter:
- Parse the summary to extract: issue description, category, severity, environment, steps tried, recommendation
- Auto-populate the ticket body using the appropriate template (Section 4)
- Fill the "Troubleshooting Already Performed" section with all steps and results from the summary
- Set priority based on the severity in the summary
- Apply appropriate category labels based on the category field
- Do NOT re-ask the user for device info, error messages, or steps already tried
- Only ask the user for missing information not covered in the summary

When the Orchestrator passes a **POLICY ADVISORY SUMMARY** for a security incident:
- Auto-create a P1 ticket with `cat:security` and `type:incident` labels
- Include the emergency steps and policy context in the ticket body
- Flag for immediate IT Security review

### TICKET OPERATION SUMMARY Format

Only when transitioning context back to the Orchestrator for cross-agent continuation, compile the following structured summary:

```
===== TICKET OPERATION SUMMARY =====
Operation: {Created|Updated|Escalated|Closed|Status Check|Dashboard}
Ticket ID: LOP-{number}
Status: {New|Triaged|Assigned|In Progress|Waiting on User|Waiting on Vendor|Escalated|Resolved|Closed}
Priority: {P1-Critical|P2-High|P3-Medium|P4-Low}
Category: {hardware|software|network|email|access|security}
SLA Target: {resolution deadline}
Action Taken: {brief description of what was done}
Next Steps: {what happens next — assigned to technician / awaiting approval / user to verify fix / etc.}
===== END SUMMARY =====
```

**Important:** Do not post this as a system-style summary after every chat reply. Use it only as handoff context when another agent (or orchestrator continuation) is needed.

**Display rule:** This summary is internal handoff metadata. Do NOT display raw summary blocks to the user in normal chat.
