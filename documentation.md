# SmartDesk AI — Solution Documentation

## Enterprise IT HelpDesk Command Center

**Platform:** Microsoft Copilot Studio  
**Deployment:** Microsoft 365 Copilot Chat (Teams)  
**Data Layer:** Microsoft Dataverse (MCP Server)  
**Authentication:** Microsoft Entra ID OAuth 2.0

---

## Table of Contents

1. [Solution Overview](#1-solution-overview)
2. [Architecture](#2-architecture)
3. [Agent Catalog](#3-agent-catalog)
   - 3.1 SmartDesk Orchestrator
   - 3.2 IT Troubleshooter
   - 3.3 Ticket Manager
   - 3.4 Access & Software Provisioner
   - 3.5 IT Policy Advisor
   - 3.6 Security Awareness Coach
4. [MCP Server & Dataverse Integration](#4-mcp-server--dataverse-integration)
5. [Cross-Agent Coordination](#5-cross-agent-coordination)
6. [Adaptive Cards](#6-adaptive-cards)
7. [Knowledge Base Reference](#7-knowledge-base-reference)
8. [Security & Authentication](#8-security--authentication)
9. [Deployment Guide](#9-deployment-guide)
10. [Anti-Duplicate Message Architecture](#10-anti-duplicate-message-architecture)
11. [Demo Scenarios](#11-demo-scenarios)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Solution Overview

### What is SmartDesk AI?

SmartDesk AI is a connected multi-agent system built in Microsoft Copilot Studio that functions as an Enterprise IT HelpDesk. It allows employees to resolve technical issues, manage IT support tickets, request software and access, navigate IT policies, and receive cybersecurity training — all through natural language conversation within Microsoft 365 Copilot Chat (Teams).

### Key Capabilities

| Capability | Description |
|-----------|-------------|
| **IT Troubleshooting** | Guided step-by-step diagnostics for hardware, software, network, email, printer, and collaboration issues |
| **Ticket Management** | Full ticket lifecycle — create, track, escalate, close — with Dataverse persistence via MCP |
| **Software & Access Provisioning** | Request software from an approved catalog, manage access permissions, handle license assignments |
| **IT Policy Guidance** | Instant answers on password policies, acceptable use, BYOD, data classification, compliance |
| **Security Training** | Interactive phishing quizzes (SLAM method), threat education, security best practices |
| **Multi-Agent Coordination** | Seamless context passing between agents — employees never repeat information |

### Design Principles

1. **Hybrid Orchestration**: The Orchestrator uses 5 routing topics (`Route_Troubleshooter`, `Route_TicketManager`, `Route_AccessProvisioner`, `Route_PolicyAdvisor`, `Route_SecurityCoach`), each with a Connected Agent node followed by "End current topic" to structurally prevent duplicate messages. All 5 connected agents run in fully generative mode (no Topics). Intent recognition is handled by Copilot Studio's trigger-phrase matching.

2. **Single Conversation Entry Point**: Employees interact only with the SmartDesk Orchestrator. It transparently routes to the right specialist agent.

3. **Context Preservation**: When workflows span multiple agents (e.g., troubleshooting → ticket creation), structured context summaries flow between agents so users never repeat information.

4. **Write Verification**: Every Dataverse write operation is verified with a read-after-write check before confirming success to the user.

5. **No Duplicate Messages**: Structural and instructional safeguards prevent the Orchestrator from producing additional output after a connected agent responds.

---

## 2. Architecture

### System Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Microsoft 365 Copilot Chat                        │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │           SmartDesk Orchestrator (Main Agent)                │    │
│  │  - Generative intent recognition & intelligent routing       │    │
│  │  - Cross-agent coordination & context passing                │    │
│  │  - 5 routing topics + generative AI                              │    │
│  └──────┬────────────┬────────────┬────────────┬─────────────┬──┘    │
│         │            │            │            │             │       │
│    ┌────▼────┐ ┌─────▼─────┐  ┌───▼────┐  ┌────▼─────┐  ┌────▼─────┐ │
│    │IT       │ │ Ticket    │  │Access &│  │IT Policy │  │Security  │ │
│    │Trouble- │ │ Manager   │  │Software│  │Advisor   │  │Awareness │ │
│    │shooter  │ │           │  │Provis. │  │          │  │Coach     │ │
│    │(Gen.)   │ │(Gen.+MCP) │  │(Gen.+  │  │(Gen.)    │  │(Gen.)    │ │
│    │         │ │           │  │ MCP)   │  │          │  │          │ │
│    └─────────┘ └─────┬─────┘  └───┬────┘  └──────────┘  └──────────┘ │
│                      │            │                                  │
└──────────────────────┼────────────┼──────────────────────────────────┘
                       │            │
              ┌────────▼────────────▼───────────┐
              │   Dataverse MCP Server (Online) │
              │                                 │
              │   Transport: Streamable HTTP    │
              │   Auth: Entra ID OAuth 2.0      │
              │   Table: tickets                │
              │                                 │
              │   Operations:                   │
              │   - describe_table              │
              │   - create row                  │
              │   - read / query rows           │
              │   - update row                  │
              │   - list rows                   │
              └─────────────────────────────────┘
```

### Component Summary

| Component | Technology | Role |
|-----------|-----------|------|
| SmartDesk Orchestrator | Copilot Studio (generative) | Entry point, routing, coordination |
| 5 Connected Agents | Copilot Studio (generative) | Specialized domain expertise |
| Dataverse MCP Server | MCP (Streamable HTTP) | CRUD operations on `tickets` table |
| Microsoft Entra ID | OAuth 2.0 | Authentication for MCP server access |
| Microsoft Teams | M365 Copilot Chat | User interface / delivery channel |
| Adaptive Cards | Teams Card Framework | Interactive UI elements |

### How MCP Tools Work in Copilot Studio

MCP Server tools are **not** available as "Call an action" nodes inside topic flows. They are invoked by the agent's **generative AI orchestrator**:

1. The tool is added to the agent's configuration (Settings → Tools)
2. The agent's LLM reads the tool's capabilities and the agent's Instructions
3. When the conversation context matches an operation described in the Instructions (e.g., "create a ticket"), the AI automatically invokes the MCP tool with the appropriate parameters
4. The AI interprets the tool's response and reports back to the user

This is why Ticket Manager and Access Provisioner operate in generative mode without Topics — the Instructions guide the AI on when/how to use the MCP tool.

---

## 3. Agent Catalog

### 3.1 SmartDesk Orchestrator (Main Agent)

| Property | Value |
|----------|-------|
| **Copilot Studio Name** | `SmartDesk` |
| **Type** | Main Agent (entry point) |
| **Mode** | Generative + 5 routing topics (`Route_Troubleshooter`, `Route_TicketManager`, `Route_AccessProvisioner`, `Route_PolicyAdvisor`, `Route_SecurityCoach`) |
| **Tools** | 5 Connected Agents |
| **Knowledge Base** | `helpdesk-orchestrator-guide.md` |

**Purpose**: The central hub. Greets employees, recognizes intent via generative AI, and routes to the appropriate specialist agent. Never answers specialized questions itself — always delegates.

**Routing Rules**:

| User Intent | Routes To |
|-------------|-----------|
| Hardware/software issues, errors, connectivity, printing, email problems | IT Troubleshooter |
| Create/track/escalate/close tickets | Ticket Manager |
| Software requests, access requests, password resets, licenses | Access & Software Provisioner |
| Policy questions, BYOD, data classification, acceptable use | IT Policy Advisor |
| Security training, phishing quizzes, cyber hygiene | Security Awareness Coach |

**Cross-Agent Coordination**: The Orchestrator manages context flow between agents:
- Troubleshooter → Ticket Manager (diagnostic summary → auto-populate ticket)
- Policy Advisor → Ticket Manager (security incident → P1 ticket)
- Policy Advisor → Access Provisioner (BYOD enrollment, software need)
- Any agent → Security Coach (security concerns → awareness follow-up)

**5 Routing Topics**: Each connected agent has a dedicated routing topic (`Route_Troubleshooter`, etc.) with trigger phrases matching that agent's domain. Each topic contains exactly: Trigger → Connected Agent node → "End current topic" node (no Message nodes). Copilot Studio's trigger-phrase matching handles intent recognition; the "End current topic" node prevents the generative layer from producing any additional output (JSON logs, echoes, summaries) after the connected agent responds.

---

### 3.2 IT Troubleshooter

| Property | Value |
|----------|-------|
| **Copilot Studio Name** | `ITTroubleshooter` |
| **Type** | Connected Agent |
| **Mode** | Generative (no Topics, no tools) |
| **Knowledge Base** | `troubleshooting-guide.md` |

**Purpose**: Guides employees through structured troubleshooting for common IT issues. Uses decision-tree logic from its knowledge base to walk through diagnostic steps one at a time.

**Diagnostic Approach**:
1. **IDENTIFY** — Ask targeted clarifying questions
2. **CATEGORIZE** — Determine issue category and severity
3. **DIAGNOSE** — Walk through diagnostic steps one at a time
4. **RESOLVE** — Provide clear fix instructions
5. **VERIFY** — Confirm the fix worked
6. **DOCUMENT** — Summarize for ticket creation if needed

**Troubleshooting Categories**:

| Category | Covers |
|----------|--------|
| 💻 Hardware | Boot failures, BSOD, slow performance, screen issues, overheating, battery, docking station |
| 🖥️ Software | OS issues, application crashes, updates, installations |
| 🌐 Network | WiFi, VPN, "connected but no internet", DNS, slow connection |
| 📧 Email | Outlook crashes, can't send/receive, storage full, calendar sync, shared mailbox |
| 🖨️ Printing | Printer offline, stuck queue, quality issues, paper jams, adding printers |
| 🎥 Collaboration | Teams audio/camera/screen sharing, OneDrive sync, SharePoint access |
| 🔐 Authentication | Password expired/reset (SSPR), account lockout, MFA issues |
| 📱 Mobile | Company phone, MDM enrollment, email on mobile |

**Escalation Criteria** (recommend creating a ticket):
- Issue persists after 3+ troubleshooting steps
- Issue requires admin access or backend changes
- Hardware replacement needed
- Issue affects multiple users (systemic)
- Security-related issue
- User explicitly requests human support

**Handoff Output**: When escalation is needed, produces a DIAGNOSTIC SUMMARY containing: issue description, category, severity (P1–P4), user's environment, steps attempted with results, observations, and recommendation.

---

### 3.3 Ticket Manager

| Property | Value |
|----------|-------|
| **Copilot Studio Name** | `TicketManager` |
| **Type** | Connected Agent |
| **Mode** | Generative (no Topics, with MCP tool) |
| **Tools** | Dataverse MCP Server (`tickets` table) |
| **Knowledge Base** | `helpdesk-ticket-playbook.md` |

**Purpose**: Manages the full lifecycle of IT support tickets with direct Dataverse read/write operations via MCP.

**Supported Operations**:
1. **Create ticket** — gather details, schema check, build payload, execute create, verify
2. **Get ticket status** — query by ticket number (e.g., SD-1002)
3. **List user tickets** — show all open tickets for the requesting user
4. **Update ticket** — add information, change fields
5. **Escalate ticket** — increase priority with reason, recalculate SLA
6. **Close ticket** — require resolution note + user confirmation
7. **Dashboard** — counts by status and priority

**Create Flow (Mandatory)**:
1. Collect required inputs (summary, category, severity, environment, steps attempted)
2. Call `describe_table` via MCP to get actual Dataverse logical names
3. Map inputs to schema logical names
4. Build preview and ask explicit confirmation
5. Execute create via MCP
6. Validate write success
7. Perform read-after-write verification
8. Only then confirm ticket creation to user

**Dataverse Field Mapping**:

| Field | Purpose | Example Value |
|-------|---------|---------------|
| `Name` | Short ticket title | `[Hardware] Laptop recurring freezes` |
| `Category` | Issue category | `hardware` |
| `Type` | Ticket type | `Incident` / `Request` / `Access request` / `Software install request` |
| `SLA` | Priority | `P1-Critical` / `P2-High` / `P3-Medium` / `P4-Low` |
| `Status` | Current status | `new` (on create) |
| `Description` | Full details | Problem statement + impact + environment + steps tried |
| `ResolutionNotes` | Resolution | Only on close |

**SLA Baselines**:

| Priority | Response Time | Resolution Target |
|----------|--------------|-------------------|
| P1-Critical | 15 minutes | 4 hours |
| P2-High | 1 hour | 8 hours |
| P3-Medium | 4 hours | 3 business days |
| P4-Low | 8 hours | 5 business days |

---

### 3.4 Access & Software Provisioner

| Property | Value |
|----------|-------|
| **Copilot Studio Name** | `AccessProvisioner` |
| **Type** | Connected Agent |
| **Mode** | Generative (no Topics, with MCP tool) |
| **Tools** | Dataverse MCP Server (`tickets` table) |
| **Knowledge Base** | `software-access-catalog.md` |

**Purpose**: Handles software installation requests, access provisioning, license management, and account administration. Creates tracking entries in Dataverse via MCP.

**Request Types**:

| Type | Dataverse `Type` Value | Description |
|------|---------------------------|-------------|
| Software Installation | `Software install request` | New software from approved catalog |
| Access Request | `Access request` | Access to systems, SharePoint sites, applications |
| License Assignment | `License assignment` | Assign, upgrade, or change licenses |
| Account Administration | `Account administration` | New account, modification, deactivation |

**Self-Service Capabilities** (no ticket needed):
- Password reset via SSPR (https://aka.ms/sspr)
- MFA setup via https://aka.ms/mfasetup
- OneDrive storage management
- Teams settings and preferences

**Software Catalog Categories**:
- 💼 Productivity & Office (M365, Visio, Project, Adobe, Power BI)
- 💻 Development (VS Code, Visual Studio, Git, Docker, Node.js, Python)
- 📊 Data & Analytics (Power BI Pro, SQL Server Management Studio, R Studio)
- 🎨 Creative & Design (Adobe Creative Cloud, Figma, Canva, Snagit)
- 💬 Communication (Teams, Slack, Zoom, Webex)
- 🔒 Security (CrowdStrike, 1Password, Cisco AnyConnect)
- 📁 Utilities (7-Zip, WinRAR, TreeSize, Greenshot)

Each software item has defined: license type, self-service availability, approval requirements, and installation method.

---

### 3.5 IT Policy Advisor

| Property | Value |
|----------|-------|
| **Copilot Studio Name** | `ITPolicyAdvisor` |
| **Type** | Connected Agent |
| **Mode** | Generative (no Topics, no tools) |
| **Knowledge Base** | `it-policies-reference.md` |

**Purpose**: Answers questions about IT policies, security guidelines, acceptable use, compliance, and best practices. Provides clear guidance with policy references.

**Policy Domains**:

| Domain | Key Topics |
|--------|-----------|
| Password & Authentication | 12-char minimum, 90-day expiry, MFA mandatory, SSPR, lockout rules |
| Acceptable Use (AUP) | Business use primary, limited personal use, prohibited activities |
| BYOD Policy | Allowed devices, MDM enrollment required, security requirements, data separation |
| Data Classification | 🔴 Confidential / 🟠 Internal / 🟢 Public — handling rules per tier |
| Remote Work | VPN required, approved locations, home network security, screen lock |
| Email & Communication | External email rules, encryption for sensitive data, attachment limits |
| Software & Licensing | Only approved catalog, no pirated software, shadow IT prohibited |
| Security Incidents | Immediate action steps, incident reporting, P1 ticket creation |
| Privacy & Data Protection | GDPR awareness, data retention, breach notification |

**Cross-Agent Handoff**: When a policy question leads to an action:
- BYOD enrollment needed → suggests routing to Access Provisioner
- Software request validated → suggests routing to Access Provisioner
- Security incident identified → suggests routing to Ticket Manager (P1)

---

### 3.6 Security Awareness Coach

| Property | Value |
|----------|-------|
| **Copilot Studio Name** | `SecurityCoach` |
| **Type** | Connected Agent |
| **Mode** | Generative (no Topics, no tools) |
| **Knowledge Base** | `security-awareness-training.md` |

**Purpose**: Interactive cybersecurity educator that makes security training engaging through quizzes, scenarios, and practical tips.

**Training Modules**:

| Module | Content |
|--------|---------|
| Phishing Recognition | SLAM method (Sender, Links, Attachments, Message), realistic email quizzes, spear-phishing, whaling, smishing, vishing |
| Social Engineering | Pretexting, baiting, tailgating, quid pro quo, authority abuse, scenario exercises |
| Password Security | Strong passwords, passphrases, password managers, MFA, credential stuffing risks |
| Safe Browsing | HTTPS, certificate warnings, safe downloads, public WiFi, VPN usage |
| Data Protection | Classification awareness, safe email, social media OPSEC, clean desk |
| Incident Recognition | Compromise indicators, malware signs, immediate response steps |

**Quiz Format**: The coach presents realistic email or scenario examples one at a time, asks the employee to identify whether it's legitimate or a threat, waits for the answer, then provides detailed explanation with scoring.

**SLAM Method**:

| Letter | Check | What to Look For |
|--------|-------|-------------------|
| **S** — Sender | Who sent this? | Misspelled domains, free email providers, unusual sender |
| **L** — Links | Where do links go? | Misspelled URLs, extra subdomains, HTTP instead of HTTPS |
| **A** — Attachments | Expected? | .exe, .scr, .zip, .docm, .html — unexpected = dangerous |
| **M** — Message | Makes sense? | Urgency, threats, too-good-to-be-true, generic greetings |

---

## 4. MCP Server & Dataverse Integration

### Overview

The Dataverse MCP Server provides the data persistence layer for ticket and request management. It is attached to two agents (Ticket Manager and Access Provisioner) and accessed via the MCP (Model Context Protocol) over Streamable HTTP transport.

### Configuration

| Setting | Value |
|---------|-------|
| **Server Type** | MCP Server (Streamable HTTP) |
| **Authentication** | Microsoft Entra ID OAuth 2.0 |
| **Token URL** | `https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token` |
| **Scope** | `https://{org}.crm4.dynamics.com/.default` |
| **Target Table** | `tickets` |

### Dataverse Table Schema (`tickets`)

| Column | Display Name | Type | Purpose |
|--------|-------------|------|---------|
| `TicketNumber` | TicketNumber | Auto-number | Human-facing ID (e.g., SD-1002) |
| `Name` | Name | Text | Short, descriptive ticket title |
| `Description` | Description | Multiline text | Full ticket/request details |
| `Category` | Category | Choice | hardware / software / network / email / printing / security / access / account / mobile / other |
| `Type` | Type | Choice | Incident / Request / Question / Change / Access request / Software install request / License assignment / Account administration |
| `SLA` | Priority | Choice | P1-Critical / P2-High / P3-Medium / P4-Low |
| `Status` | Status | Choice | new / triaged / assigned / in-progress / waiting-on-user / waiting-on-vendor / escalated / resolved / closed |
| `SlaTarget` | SLA Target | DateTime | Resolution target timestamp |
| `SlaBreached` | SLA Breached | Yes/No | SLA breach indicator |
| `ReportedBy` | Reported By | Text | Requesting employee |
| `AssignedTo` | Assigned To | Text | Current owner |
| `TroubleshootingSteps` | Troubleshooting Steps | Multiline text | Steps already attempted |
| `LastUpdate` | Last Update | Multiline text | Latest update summary |
| `NextSteps` | Next Steps | Multiline text | Follow-up actions |
| `ResolutionNotes` | Resolution Notes | Multiline text | Final resolution (on close) |
| `statecode` | Status | System | System status field |
| `statuscode` | Status Reason | System | System status reason |

### MCP Operations Used by Agents

| Operation | Used By | Purpose |
|-----------|---------|---------|
| `describe_table` | Ticket Manager, Access Provisioner | Schema discovery before first write in session |
| Create row | Ticket Manager, Access Provisioner | Create new tickets/requests |
| Read / Query rows | Ticket Manager, Access Provisioner | Status checks, dashboard, read-after-write verification |
| Update row | Ticket Manager | Update ticket fields, escalate, close |
| List rows | Ticket Manager, Access Provisioner | List user's tickets, filtered queries |

### Write Verification Protocol

Every create/update operation follows a strict verification process:

1. **Execute write** via MCP
2. **Check response** — must contain a persistent identifier (TicketNumber or row ID)
3. **Read-after-write** — immediately query the created/updated row to confirm it exists
4. **Only then** confirm success to the user
5. **On failure** — do not claim success; explain the issue and offer retry (max 2 attempts)

---

## 5. Cross-Agent Coordination

### Context Passing Model

All agents return structured summaries to the Orchestrator only when a handoff or continuation is needed. These summaries are **internal metadata** — never rendered in user-visible chat.

### Summary Formats

**DIAGNOSTIC SUMMARY** (from IT Troubleshooter):
```
Issue: [one-line description]
Category: [hardware/software/network/email/printer/collaboration/auth/mobile]
Severity: [P1/P2/P3/P4]
User's Environment: [OS, device, application]
Steps Attempted:
1. [Step] → Result: [resolved/not resolved/partial]
2. [Step] → Result: [resolved/not resolved/partial]
Observations: [error messages, patterns]
Recommendation: [create ticket / on-site visit / admin action]
```

**TICKET OPERATION SUMMARY** (from Ticket Manager):
```
Operation: [Created/Updated/Escalated/Closed/Status Check]
Ticket ID: [SD-####]
Status: [current status]
Priority: [P1-P4]
SLA Target: [DateTime]
Action Taken: [what was done]
Next Steps: [follow-up actions]
```

**PROVISIONING SUMMARY** (from Access Provisioner):
```
Operation: [Software Request/Access Request/License Change/etc.]
Request ID: [SD-####]
Item: [specific software or access]
Status: [created/pending approval/approved/denied]
Approval Required: [Yes/No — Approver]
Estimated Processing: [timeframe]
Next Steps: [what happens next]
```

**POLICY ADVISORY SUMMARY** (from IT Policy Advisor):
```
Policy Area: [Password/AUP/BYOD/Data Classification/Remote Work/Security Incident/etc.]
Question: [what the user asked]
Answer Summary: [brief guidance provided]
Action Required: [None/Ticket needed/Route to agent/Contact IT Security]
Cross-Agent Routing Suggested: [None/Ticket Manager/Access Provisioner]
```

### Cross-Agent Flow Patterns

```
Pattern 1: Troubleshoot → Ticket
─────────────────────────────────
User → Orchestrator → IT Troubleshooter
                      (diagnoses, cannot resolve)
                      → DIAGNOSTIC SUMMARY → Orchestrator
                      → "Create a ticket?" → User confirms
                      → Ticket Manager (with full diagnostic context)
                      → Auto-populated ticket created in Dataverse

Pattern 2: Policy → Access Request
───────────────────────────────────
User → Orchestrator → IT Policy Advisor
                      (confirms policy allows the action)
                      → POLICY ADVISORY SUMMARY → Orchestrator
                      → "Would you like to request it?" → User confirms
                      → Access Provisioner (with policy context)
                      → Request created in Dataverse

Pattern 3: Security Incident → Emergency Ticket
────────────────────────────────────────────────
User → Orchestrator → IT Policy Advisor
                      (provides immediate action steps)
                      → POLICY ADVISORY SUMMARY (action: P1 ticket) → Orchestrator
                      → Ticket Manager (Priority: P1-Critical, Category: security)
                      → Emergency ticket created in Dataverse

Pattern 4: New Employee Onboarding
───────────────────────────────────
User → Orchestrator → Access Provisioner
                      (gathers requirements: name, role, software, access)
                      → Creates multiple linked requests in Dataverse:
                        - Account creation
                        - License assignment
                        - Software installs
                        - Access requests
                      → Consolidated confirmation card

Pattern 5: Phishing Incident → Training + Ticket
──────────────────────────────────────────────────
User (didn't click) → Orchestrator → Security Coach
                                     (recognition training + tips)

User (clicked link)  → Orchestrator → IT Policy Advisor
                                      (emergency steps)
                                      → Ticket Manager (P1 security incident)
                                      → Later: Security Coach (follow-up training)
```

---

## 6. Adaptive Cards

Adaptive Cards provide rich, interactive UI elements within Microsoft Teams. Each agent uses cards for specific purposes:

### Card Types by Agent

| Agent | Card Type | Purpose |
|-------|-----------|---------|
| IT Troubleshooter | Troubleshooting Step Card | Step number, instructions, estimated time, action buttons (✅ Fixed / ❌ Next Step / 🎫 Create Ticket) |
| IT Troubleshooter | Category Selection Card | When issue type is unclear — presents category options |
| Ticket Manager | Ticket Confirmation Card | Ticket ID, category, priority, SLA target, status |
| Ticket Manager | Ticket Status Card | Current status, last update, next steps, SLA status |
| Ticket Manager | Dashboard Card | Counts by status and priority |
| Access Provisioner | Software Request Form Card | Software details, justification input, submit/cancel |
| Access Provisioner | Self-Service Step Card | Numbered steps for password reset, MFA setup |
| Access Provisioner | Catalog Browse Card | Software categories and details |
| IT Policy Advisor | Policy DO/DON'T Card | ✅ Allowed / ⚠️ Restricted / ❌ Prohibited |
| IT Policy Advisor | Data Classification Card | Color-coded tiers with handling rules |
| IT Policy Advisor | Security Incident Card | Urgent formatting, immediate action steps |
| Security Coach | Phishing Quiz Card | Email scenario, answer options |
| Security Coach | Score Card | Session score, strengths, improvement areas |

### Teams Card Rules

- All Adaptive Cards use `Action.Submit` with `msteams.messageBack` and empty `text` property
- Agents process `value.action` only; plain `text` events are ignored to prevent duplicate replies
- Cards are the response — no additional text message is sent alongside a card

---

## 7. Knowledge Base Reference

Each agent has a dedicated knowledge base document uploaded to Copilot Studio:

| Knowledge File | Agent | Content | Size |
|---------------|-------|---------|------|
| `helpdesk-orchestrator-guide.md` | SmartDesk Orchestrator | Agent catalog, routing decision tree, cross-agent workflow patterns, context passing model, FAQ, emergency contacts | ~374 lines |
| `troubleshooting-guide.md` | IT Troubleshooter | Diagnostic decision trees for all IT categories — computer, network, email, printer, collaboration, authentication, mobile | ~423 lines |
| `helpdesk-ticket-playbook.md` | Ticket Manager | Ticket lifecycle, templates, SLA definitions, escalation procedures, Dataverse schema mapping, status workflows | ~666 lines |
| `software-access-catalog.md` | Access & Software Provisioner | Approved software catalog (7 categories), access request procedures, prohibited software list, self-service guides, license types | ~364 lines |
| `it-policies-reference.md` | IT Policy Advisor | Full IT policy reference — password, AUP, BYOD, data classification, remote work, email, security incidents, privacy/GDPR | ~495 lines |
| `security-awareness-training.md` | Security Awareness Coach | SLAM method, phishing quiz bank (6+ scenarios), social engineering encyclopedia, security tips, scoring system | ~502 lines |

### Knowledge Base Design Principles

- **Structured for LLM consumption**: Decision trees, tables, numbered lists, and clear section headers enable accurate generative responses
- **Self-contained per agent**: Each document contains everything the agent needs — no cross-references required at runtime
- **Actionable content**: Not just rules, but step-by-step procedures, click paths, keyboard shortcuts, and practical examples
- **Consistent format**: All knowledge files follow the same pattern — Agent Behavior Model section → content sections → handoff format → Adaptive Card usage

---

## 8. Security & Authentication

### Authentication Architecture

| Component | Auth Method | Details |
|-----------|-----------|---------|
| MCP Server → Dataverse | OAuth 2.0 Client Credentials | Entra ID app registration with Dynamics CRM scope |
| Users → Copilot Chat | Microsoft 365 SSO | Standard M365 user authentication |
| Agent → Connected Agent | Copilot Studio built-in | Internal agent-to-agent routing |

### Entra ID App Registration Requirements

For the Dataverse MCP Server:
- **API Permissions**: Dynamics CRM → `user_impersonation` (or `.default`)
- **Token URL**: `https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token`
- **Scope**: `https://{org}.crm4.dynamics.com/.default`
- **Admin Consent**: Required for Dynamics CRM permissions

### Security Guardrails (All Agents)

- Agents never ask for passwords, MFA codes, or sensitive tokens
- If sensitive credentials are pasted, the agent instructs the user to rotate/reset immediately
- Sensitive data is never stored in ticket description fields
- No hardcoded secrets — all credentials managed via Entra ID
- All ticket operations include write verification and audit trails
- Security incidents are automatically prioritized as P1-Critical

### Data Handling

- IT policies enforce data classification: 🔴 Confidential / 🟠 Internal / 🟢 Public
- Agents reference classification rules when handling user data
- No customer data is exposed in agent responses beyond what the user provided
- Dataverse row-level security applies to ticket queries

---

## 9. Deployment Guide

### Prerequisites

- Microsoft 365 account with Copilot Studio access
- Microsoft Copilot Studio environment with Dataverse
- Microsoft Entra ID tenant with app registration capability
- Dataverse instance with `tickets` table configured

### Step-by-Step Deployment

#### Step 1: Import Solution

Import the unmanaged solution into your Power Platform environment:

```
SmartDeskAIEnterpriseITHelpDeskCommandCenter.zip
```

This creates all 6 agents with their instructions pre-configured.

#### Step 2: Upload Knowledge Bases

For each agent, upload the corresponding knowledge file from the `knowledge/` folder:

| Agent | Knowledge File |
|-------|---------------|
| SmartDesk (Orchestrator) | `knowledge/helpdesk-orchestrator-guide.md` |
| IT Troubleshooter | `knowledge/troubleshooting-guide.md` |
| Ticket Manager | `knowledge/helpdesk-ticket-playbook.md` |
| Access & Software Provisioner | `knowledge/software-access-catalog.md` |
| IT Policy Advisor | `knowledge/it-policies-reference.md` |
| Security Awareness Coach | `knowledge/security-awareness-training.md` |

#### Step 3: Configure MCP Server

For **Ticket Manager** and **Access & Software Provisioner**:

1. Open the agent in Copilot Studio
2. Go to **Settings → Tools → Add a tool**
3. Select **MCP Server (Streamable HTTP)**
4. Enter the Dataverse MCP server URL
5. Configure OAuth 2.0:
   - Client ID: from Entra app registration
   - Client Secret: from Entra app registration
   - Token URL: `https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token`
   - Scope: `https://{org}.crm4.dynamics.com/.default`
6. **Save** and **test connection** by querying the `tickets` table

#### Step 4: Publish Agents

Publish in this order (dependencies matter):

1. **IT Troubleshooter** (no dependencies)
2. **IT Policy Advisor** (no dependencies)
3. **Security Awareness Coach** (no dependencies)
4. **Ticket Manager** (depends on MCP connection)
5. **Access & Software Provisioner** (depends on MCP connection)
6. **SmartDesk Orchestrator** (depends on all 5 connected agents being published)

#### Step 5: Test in Teams

1. Open Microsoft 365 Copilot Chat
2. Type `@SmartDesk` or select the agent from the agent picker
3. Test each routing path:
   - `"My laptop is slow"` → should route to IT Troubleshooter
   - `"Create a ticket for my printer"` → should route to Ticket Manager
   - `"I need Visio"` → should route to Access Provisioner
   - `"What's the BYOD policy?"` → should route to IT Policy Advisor
   - `"Quiz me on phishing"` → should route to Security Coach
4. Verify exactly 1 message per user input (no duplicates)
5. Test a cross-agent flow (e.g., troubleshoot → ticket creation)

---

## 10. Anti-Duplicate Message Architecture

Duplicate messages are the #1 UX issue with multi-agent setups in Teams. SmartDesk uses a layered approach to prevent them:

### Root Cause

After a connected agent responds, the Orchestrator's generative layer gets a turn to produce output. Without suppression, it generates an echo, summary, or follow-up — resulting in 2 message bubbles for 1 user input.

### Prevention Layers

| Layer | Mechanism | Impact |
|-------|----------|--------|
| **Structural** | 5 routing topics in Orchestrator (`Route_Troubleshooter`, etc.) — each topic: Trigger → Connected Agent → "End current topic" (no Message nodes) | **Highest** — terminates the Orchestrator's turn before the generative layer can produce output |
| **Instructional** | Orchestrator instructions: "After routing, PRODUCE ABSOLUTELY NO TEXT. Zero tokens. NEVER generate JSON metadata." | Medium — guides LLM behavior as additional safeguard |
| **Card Handling** | All agents: process `value.action` only, ignore plain `text` from card submits | Prevents card-triggered duplicates |

### Implementation Priority

If duplicate messages appear, apply fixes in this order:

1. Create the 5 routing topics with "End current topic" after each Connected Agent node (highest impact)
2. Verify instruction blocks contain anti-duplicate rules and forbidden JSON patterns
3. Re-publish the Orchestrator and all connected agents
4. Test in Teams (not just the Copilot Studio test pane — behavior differs)

---

## 11. Demo Scenarios

### Scenario 1: Full Troubleshoot-to-Ticket Flow

```
User: "My laptop keeps freezing"

→ Orchestrator routes to IT Troubleshooter
Troubleshooter: "That sounds frustrating. Let's figure this out. When does it 
                 freeze — during specific tasks or randomly?"
User: "Randomly, about 3 times a day"
Troubleshooter: [Adaptive Card: Step 1 — Check Task Manager (Ctrl+Shift+Esc)]
User: "CPU is at 95%"
Troubleshooter: [Adaptive Card: Step 2 — Identify top process, restart it]
User: "Still freezing"
Troubleshooter: [Adaptive Card: Step 3 — Check disk space]
User: "Tried that too, still happening"
Troubleshooter: "I've tried several steps without success. Would you like me to 
                 create a support ticket? I'll include all the troubleshooting 
                 steps we tried."
User: "Yes please"

→ Orchestrator passes DIAGNOSTIC SUMMARY to Ticket Manager
Ticket Manager: [Adaptive Card: Ticket Confirmation]
                "Done — your issue is tracked as SD-1042.
                 Priority: P3-Medium | Expected resolution: 3 business days.
                 All diagnostic steps are included for the IT team."
```

### Scenario 2: Software Request with Policy Check

```
User: "Can I install Slack on my work laptop?"

→ Orchestrator routes to Access & Software Provisioner
Provisioner: "Slack is in our approved catalog! It requires department head approval 
              since it's an Enterprise license. 
              [Adaptive Card: Slack — Enterprise license, Dept head approval required]
              Would you like to submit a request? I'll need a brief business 
              justification."
User: "We need it for our external partner collaboration"
Provisioner: "Got it. Creating your request now..."
             [Adaptive Card: Request Confirmation — SD-1043, Software Install, 
              Pending Approval]
             "Your request is tracked as SD-1043. Your department head will 
              receive an approval notification."
```

### Scenario 3: Security Incident — Phishing Click

```
User: "I clicked a link in a suspicious email and entered my password"

→ Orchestrator recognizes urgency, routes to IT Policy Advisor
Policy Advisor: "⚠️ This is a potential security incident. Take these steps 
                 IMMEDIATELY:
                 1. Change your password NOW at https://aka.ms/sspr
                 2. Do NOT click any more links in that email
                 3. Report the email: Right-click → Report → Phishing
                 4. Check your Sent folder for any emails you didn't send
                 I'm creating an emergency ticket for the IT Security team."

→ Orchestrator routes to Ticket Manager with P1 priority
Ticket Manager: [Adaptive Card: Security Incident Ticket — SD-1044, P1-Critical]
                "Emergency ticket SD-1044 created. IT Security will respond 
                 within 15 minutes. If you haven't changed your password yet, 
                 do it now: https://aka.ms/sspr"
```

### Scenario 4: Security Awareness Quiz

```
User: "Quiz me on phishing"

→ Orchestrator routes to Security Awareness Coach
Coach: "Let's test your phishing detection skills! I'll show you emails and 
        you tell me if they're legitimate or phishing. Ready?

        📧 Email #1:
        From: IT-HelpDesk@c0mpany-support.com
        Subject: ⚠️ URGENT: Password Expires in 2 Hours
        'Click the link below immediately to reset your password 
         or your account will be locked.'
        
        [Adaptive Card: Legitimate ✅ | Phishing 🎣]"

User: "Phishing"
Coach: "🎯 Correct! Red flags:
        1. Sender: c0mpany-support.com — zero instead of 'o', fake domain
        2. Urgency: '2 hours' creates panic
        3. Threat: 'account will be locked' — scare tactic
        4. Generic greeting: no personalization
        5. IT never sends password reset links — you go to SSPR yourself
        
        Score: 1/1 — Ready for the next one?"
```

### Scenario 5: New Employee Setup

```
User: "New team member Sarah Chen starting Monday — she's a Data Analyst"

→ Orchestrator routes to Access & Software Provisioner
Provisioner: "I'll set up everything Sarah needs. Let me create the requests:
             
             Based on the Data Analyst role, I recommend:
             - M365 E3 license (standard)
             - Power BI Pro (for analytics)
             - Python + R Studio (from approved dev tools)
             - Access to Analytics SharePoint site
             
             [Adaptive Card: New Employee Setup — 4 linked requests]
             
             Shall I create all of these? Any additional software or access needed?"
User: "Yes, and add Visio too"
Provisioner: "Done! Created 5 linked requests:
              SD-1045: Account creation + M365 E3
              SD-1046: Power BI Pro license
              SD-1047: Python + R Studio install
              SD-1048: Analytics SharePoint access
              SD-1049: Visio (requires manager approval)
              
              All should be ready by Monday morning. Visio approval is pending."
```

---

## 12. Troubleshooting

### MCP Tool Not Creating Records

| # | Check | How to Verify | Fix |
|---|-------|---------------|-----|
| 1 | Tool added to agent? | Agent → Settings → Tools — MCP server listed and active | Add tool per deployment guide |
| 2 | Tool status connected? | Tool test panel queries `tickets` successfully | Re-enter OAuth credentials |
| 3 | Generative orchestrator enabled? | Agent → Settings → Generative AI → generative mode ON | Enable generative orchestration |
| 4 | Instructions reference the tool? | Instructions contain "Dataverse MCP Server" and "MUST call this tool" | Update instructions |
| 5 | OAuth scope correct? | Entra app → API permissions → Dynamics CRM scope configured | Add correct scope, grant admin consent |
| 6 | Table columns match? | Compare `tickets` column logical names with Instructions | Align field names |
| 7 | Activity trace shows tool call? | Test panel → Activity Map → look for tool call node | If missing: instructions unclear or tool inactive |
| 8 | Agent published? | Changes require re-publishing | Click **Publish** in Copilot Studio |

### "Blocked Step" Error

| # | Check | Fix |
|---|-------|-----|
| 1 | Moderation level | Settings → Security → Moderation → set to **Low** |
| 2 | AI actions enabled? | Settings → Generative AI → "Allow the AI to use actions" → **ON** |
| 3 | Content filtering | Test with minimal payload (Name="Test", Description="Test ticket") |
| 4 | Token expired? | Re-test MCP connection in tool settings |
| 5 | System topics? | Check if a system topic (Fallback, Escalate) is catching messages |
| 6 | Context too large? | Test agent directly (not via Orchestrator) to isolate |
| 7 | Safety-triggering words? | Words like "breach", "attack", "hack" can trigger safety — test with benign content first |

### Duplicate Messages in Teams

See [Section 10](#10-anti-duplicate-message-architecture) for the full prevention architecture and fix priority list.

### Common Issues

| Symptom | Likely Cause | Solution |
|---------|-------------|----------|
| Agent responds but no ticket in Dataverse | MCP tool not invoked or write failed silently | Check activity trace for tool call; verify OAuth |
| Agent claims success without ticket number | Fabricated response (LLM hallucination) | Strengthen "TOOL USAGE (MANDATORY)" in instructions |
| Orchestrator answers instead of routing | Intent not clearly matched to routing rules | Review Instructions routing rules; check knowledge base |
| Connected agent re-asks information | Context not passed from Orchestrator | Verify DIAGNOSTIC SUMMARY or other summary format is being generated |
| Cards not rendering | Adaptive Card JSON issues or Teams version | Test in Teams desktop (not web); validate card JSON |

---

*Documentation generated for SmartDesk AI — Enterprise IT HelpDesk Command Center*  
*Microsoft Agents League — Enterprise Agents Track*
