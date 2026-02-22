# Software & Access Catalog

## Knowledge Base for Access & Software Provisioner Agent

---

### Agent Behavior Model

This agent operates as a **fully generative Connected Agent** with no predefined Topics. It uses the software catalog, access procedures, and provisioning workflows below as its knowledge foundation, but conducts a natural, conversational interaction with the user.

**Conversation approach:**
- For software requests: identify the software, check the catalog (Section 1), confirm eligibility and approval requirements, then create and track the request in Dataverse (`lop_tickets` table) via MCP
- For access requests: identify the resource, check procedures (Section 4), gather required fields conversationally, create the request
- For self-service guidance (password reset, MFA setup): walk the user through the steps one at a time from Section 6
- If the requested software is in the Prohibited list (Section 2), explain why and suggest alternatives or the exception process
- Use Adaptive Cards (Software Request card) to present software details and request confirmations
- For new employee setup: use Section 5 checklist as a guide, create linked tickets for each item

**Context received from Orchestrator:**
- User's request (always)
- POLICY ADVISORY SUMMARY from IT Policy Advisor (when checking if software/access is permitted before requesting)

**Execution model:** Access & Software Provisioner is a full connected operational agent with direct Dataverse MCP access to `lop_tickets`.

**Schema mapping rule (mandatory):** Before the first create in a session, call
`describe_table` via MCP for `lop_tickets` to retrieve the actual logical names.
Map all inputs to those logical names; if required fields are missing, stop and
report a configuration mismatch instead of attempting create.

**User-experience rule:** Never expose technical implementation details (MCP operations, payload schema, internal metadata) in user-visible chat. Always present plain-language outcomes and next steps. Do not repeat the same guidance or send duplicate messages unless the user asks.
**Teams rule (mandatory):** The agent runs in Teams chat. Avoid duplicate replies. For Adaptive Card submits, process `value.action` and ignore plain `text`.

**Context returned to Orchestrator:** PROVISIONING SUMMARY only when a cross-agent continuation/handoff is needed (see Section 9)

---

## 1. Approved Software Catalog

### 1.1 Productivity & Office

| Software | Version | License Type | Self-Service | Approval Required | Notes |
|----------|---------|-------------|-------------|-------------------|-------|
| Microsoft 365 Apps | Latest | E3/E5 subscription | ✅ Pre-installed | No | Word, Excel, PowerPoint, OneNote |
| Microsoft Visio | Plan 2 | Add-on license | ❌ | Manager approval | Diagramming, flowcharts |
| Microsoft Project | Plan 3 | Add-on license | ❌ | Manager approval | Project management |
| Microsoft Power BI Desktop | Latest | Free / Pro | ✅ Company Portal | No (Desktop free) | Pro license needs approval |
| Adobe Acrobat Pro | 2024 | Named license | ❌ | Manager approval | PDF editing, limited licenses |
| Adobe Acrobat Reader | Latest | Free | ✅ Company Portal | No | PDF viewing |
| LibreOffice | 7.x | Free/Open Source | ✅ Company Portal | No | Alternative office suite |
| Notepad++ | Latest | Free/Open Source | ✅ Company Portal | No | Text editor |

### 1.2 Communication & Collaboration

| Software | Version | License Type | Self-Service | Approval Required | Notes |
|----------|---------|-------------|-------------|-------------------|-------|
| Microsoft Teams | Latest | E3/E5 subscription | ✅ Pre-installed | No | Chat, meetings, calls |
| Microsoft Outlook | Latest | E3/E5 subscription | ✅ Pre-installed | No | Email client |
| Slack | Enterprise | Named license | ❌ | Dept head approval | Select departments only |
| Zoom | Business | Named license | ❌ | Manager approval | External meeting needs |
| Webex | Business | Named license | ❌ | Manager approval | Client requirements |

### 1.3 Development Tools

| Software | Version | License Type | Self-Service | Approval Required | Notes |
|----------|---------|-------------|-------------|-------------------|-------|
| Visual Studio Code | Latest | Free | ✅ Company Portal | No | Code editor |
| Visual Studio 2022 | Professional | Named license | ❌ | Manager approval | Dev team only |
| Git | Latest | Free/Open Source | ✅ Company Portal | No | Version control |
| GitHub Desktop | Latest | Free | ✅ Company Portal | No | Git GUI |
| Node.js (LTS) | Latest LTS | Free/Open Source | ✅ Company Portal | No | JavaScript runtime |
| Python | 3.12+ | Free/Open Source | ✅ Company Portal | No | Programming language |
| Docker Desktop | Latest | Business license | ❌ | IT Security + Manager | Requires security review |
| Postman | Latest | Free/Team | ✅ Company Portal | No (free tier) | API testing |
| Azure Data Studio | Latest | Free | ✅ Company Portal | No | Database management |
| SQL Server Management Studio | Latest | Free | ✅ Company Portal | No | SQL Server admin |
| PowerShell 7 | Latest | Free/Open Source | ✅ Company Portal | No | Scripting |

### 1.4 Data & Analytics

| Software | Version | License Type | Self-Service | Approval Required | Notes |
|----------|---------|-------------|-------------|-------------------|-------|
| Power BI Pro | Latest | Add-on license | ❌ | Manager approval | Advanced analytics |
| Tableau Desktop | 2024 | Named license | ❌ | Dept head approval | Data visualization, limited licenses |
| R / RStudio | Latest | Free/Open Source | ✅ Company Portal | No | Statistical computing |
| Azure Storage Explorer | Latest | Free | ✅ Company Portal | No | Cloud storage management |

### 1.5 Creative & Design

| Software | Version | License Type | Self-Service | Approval Required | Notes |
|----------|---------|-------------|-------------|-------------------|-------|
| Adobe Creative Cloud | 2024 | Named license | ❌ | Dept head + IT | Design/Marketing only |
| Figma | Business | Named license | ❌ | Manager approval | UX/UI design |
| Canva | Enterprise | Named license | ❌ | Manager approval | Marketing/Comms |
| Snagit | Latest | Named license | ❌ | Manager approval | Screen capture/recording |
| ShareX | Latest | Free/Open Source | ✅ Company Portal | No | Screenshot tool |
| Paint.NET | Latest | Free | ✅ Company Portal | No | Image editing |

### 1.6 Security & Utilities

| Software | Version | License Type | Self-Service | Approval Required | Notes |
|----------|---------|-------------|-------------|-------------------|-------|
| Microsoft Defender | Latest | E3/E5 included | ✅ Pre-installed | No | Antivirus (mandatory) |
| 7-Zip | Latest | Free/Open Source | ✅ Company Portal | No | File compression |
| KeePass / Bitwarden | Latest | Free/Enterprise | ✅ Company Portal | No | Password manager |
| TreeSize Free | Latest | Free | ✅ Company Portal | No | Disk usage analysis |
| Windows Terminal | Latest | Free | ✅ Company Portal | No | Modern terminal |
| VLC Player | Latest | Free/Open Source | ✅ Company Portal | No | Media player |
| Greenshot | Latest | Free/Open Source | ✅ Company Portal | No | Screenshot tool |

---

## 2. Software NOT Allowed (Prohibited)

The following software categories are **not permitted** on company devices:

| Category | Examples | Reason |
|----------|---------|--------|
| Torrent clients | BitTorrent, uTorrent, qBittorrent | Security risk, legal liability |
| Personal VPNs | NordVPN, ExpressVPN, Surfshark | Bypass security controls |
| Unauthorized remote access | TeamViewer, AnyDesk (personal) | Security risk |
| Personal cloud sync | Dropbox, Google Drive (personal) | Data leakage risk |
| Crypto mining software | NiceHash, PhoenixMiner | Policy violation, resource abuse |
| Hacking / pen-test tools | Kali tools, Metasploit (without authorization) | Policy violation |
| Unlicensed commercial software | Pirated software of any kind | Legal liability |
| Browser extensions (unapproved) | Must be from approved list | Security risk |

**Exception process:** If you need software from the prohibited list for a legitimate business reason, submit a request with a business justification. IT Security will review and may grant an exception.

---

## 3. Microsoft 365 License Tiers

### E3 License (Standard)
**Included applications:**
- Microsoft 365 Apps (Word, Excel, PowerPoint, OneNote, Outlook, Access, Publisher)
- Exchange Online (50 GB mailbox)
- SharePoint Online
- OneDrive for Business (1 TB)
- Microsoft Teams
- Microsoft Defender for Office 365
- Azure AD P1
- Intune
- Information Protection (P1)
- Windows 11 Enterprise

### E5 License (Premium)
**Everything in E3, plus:**
- Exchange Online (100 GB mailbox)
- Microsoft Defender for Endpoint P2
- Azure AD P2
- Phone System (Teams calling)
- Audio Conferencing
- Power BI Pro
- eDiscovery (Premium)
- Information Protection (P2)
- Advanced Compliance

### License Upgrade Process
1. Manager submits business justification
2. IT reviews current license utilization
3. Cost center approval
4. License assigned (typically within 1 business day)

---

## 4. Access Request Procedures

### 4.1 SharePoint Site Access
**Request Information Needed:**
- SharePoint site name/URL
- Access level: Read, Contribute (Edit), Full Control
- Business justification
- Duration (permanent or temporary)

**Process:**
1. Submit request via SmartDesk AI
2. Ticket created with site owner as approver
3. Site owner reviews and approves/rejects
4. Upon approval, IT adds the user
5. User receives confirmation email

**Typical turnaround:** 1-2 business days

### 4.2 Shared Mailbox Access
**Request Information Needed:**
- Shared mailbox address
- Access type: Send As, Send on Behalf, Full Access
- Business justification

**Process:**
1. Submit request
2. Mailbox owner/manager approves
3. IT configures access in Exchange
4. Mailbox appears in Outlook within 1 hour (may need restart)

### 4.3 Application / System Access
**Request Information Needed:**
- Application/system name
- Role/access level needed
- Business justification
- Start date (and end date if temporary)

**Common systems and their owners:**
| System | Owner/Approver | Access Levels |
|--------|---------------|---------------|
| SAP ERP | Finance Director | Various roles |
| Salesforce CRM | Sales Director | User/Admin/Reports |
| Jira / Azure DevOps | Engineering Manager | Team Member/Lead/Admin |
| Power Platform | IT Admin | Maker/Admin |
| Azure Portal | Cloud Team Lead | Reader/Contributor/Owner |
| HR System (Workday) | HR Director | Employee/Manager/Admin |
| GitHub Enterprise | Engineering Manager | Member/Maintainer/Admin |

### 4.4 Distribution List / M365 Group Membership
**Request Information Needed:**
- Group name
- Add or Remove

**Process:**
1. Submit request
2. Group owner approves (or IT if no owner)
3. Added within 1 business day

### 4.5 Network Drive / File Share Access
**Request Information Needed:**
- Share path (e.g., \\server\share\folder)
- Access level: Read or Read/Write
- Business justification

---

## 5. New Employee IT Setup

### Standard IT Package
Every new employee receives:
- [ ] Laptop (Dell Latitude or MacBook based on role)
- [ ] Microsoft 365 E3 license
- [ ] Email account (firstname.lastname@company.com)
- [ ] Teams + OneDrive setup
- [ ] VPN client installed
- [ ] Intune enrollment (MDM)
- [ ] Security awareness training link
- [ ] IT HelpDesk contact info

### Role-Specific Add-Ons
| Role | Additional Software/Access |
|------|---------------------------|
| Developer | VS Code, Git, Docker (upon review), GitHub, Azure DevOps |
| Designer | Adobe Creative Cloud, Figma |
| Analyst | Power BI Pro, Tableau (if needed) |
| Manager | HR system access, reporting tools |
| Sales | Salesforce CRM, LinkedIn Sales Nav |
| Finance | SAP ERP, Excel advanced add-ins |
| Executive | E5 license, Teams Phone System |

### Setup Timeline
- **Day -5:** Laptop prepared, accounts created
- **Day 1:** Laptop delivered, basic orientation
- **Day 1-3:** All access provisioned
- **Day 7:** Follow-up check if anything missing

---

## 6. Password & Account Self-Service

### Password Reset
1. **Web portal:** https://passwordreset.microsoftonline.com
2. **Requirements:** Must have SSPR (Self-Service Password Reset) configured
3. **Verification methods:** Microsoft Authenticator, SMS, alternate email
4. **After reset:** New password must meet policy (12+ chars, complexity)

### MFA Setup
1. Go to https://myaccount.microsoft.com → Security info
2. Click "Add sign-in method"
3. **Recommended:** Microsoft Authenticator app
   - Install from App Store / Google Play
   - Click "Add method" → Authenticator app → Follow QR code setup
4. **Backup method:** Phone number for SMS codes

### Account Recovery (Can't Access Anything)
- **If you have ANY working login** → Use self-service portal
- **If completely locked out** → Contact IT HelpDesk at ext. 4000 or email helpdesk@company.com from a personal device
- **Verification required:** Employee ID, manager confirmation

---

## 7. License Management

### How to Request a License
1. Tell SmartDesk AI what software you need
2. Agent checks if a license is available
3. If available → auto-provisioned (some) or ticket created
4. If not available → request goes to procurement queue

### License Return / Reclamation
- When leaving a project or changing roles, unused licenses should be returned
- IT runs quarterly license audits
- Unused licenses (no login for 90 days) may be reclaimed with notice

### Cost Centers
- Software licenses are charged to the employee's department cost center
- Manager receives monthly usage report
- Significant purchases require budget approval

---

## 8. Software Installation Methods

### Method 1: Company Portal (Intune) — Preferred
1. Open **Company Portal** app on your laptop
2. Browse or search for the software
3. Click **Install**
4. Wait for download and installation to complete
5. No admin rights needed — Intune handles elevation

### Method 2: Request via SmartDesk AI
1. Tell the AI agent what you need
2. A ticket is created with software name, justification, and your device info
3. IT pushes the software to your device remotely (typically within 1-2 business days)

### Method 3: Web-Based (SaaS)
Some software doesn't need installation:
- Power BI → app.powerbi.com
- Figma → figma.com
- GitHub → github.com
- Azure Portal → portal.azure.com
- Microsoft 365 Web → office.com

### You Don't Have Admin Rights?
- Standard users do NOT have local admin rights (by security policy)
- Software must be installed through Company Portal or IT ticket
- If you need temporary admin access for a specific task → submit justification → IT Security reviews (24-hour approval)

---

## 9. Context Passing & Structured Summary

### Receiving Context from Other Agents

When the Orchestrator passes a **POLICY ADVISORY SUMMARY** from the IT Policy Advisor:
- Use the policy confirmation to skip the "Is this allowed?" check
- Reference the policy in the request ticket (e.g., "Per IT Policy — software approved for this role")
- If the Policy Advisor routed here with cross-agent routing = "Access Provisioner", the user has already been told the software/access is permitted

### PROVISIONING SUMMARY Format

Only when transitioning context back to the Orchestrator for cross-agent continuation, compile the following structured summary:

```
===== PROVISIONING SUMMARY =====
Operation: {Software Request|Access Request|License Change|Password Reset Guidance|MFA Setup Guidance|New Employee Setup|Account Recovery}
Request ID: LOP-{number} (if ticket created)
Item Requested: {software name / resource name / access type}
Status: {Submitted|Auto-Provisioned|Pending Approval|Completed|Denied|Guidance Provided}
Approval Required: {None|Manager|Dept Head|IT Security|Manager + IT Security}
Estimated Processing: {Immediate|1 business day|2-3 business days|etc.}
Next Steps: {what happens next — approval pending / install via Company Portal / ticket assigned / etc.}
===== END SUMMARY =====
```

**Important:** Do not post this as a system-style summary after every response. Use it only for handoff context when routing/continuation is required.

**Display rule:** This summary is internal handoff metadata. Do NOT display raw summary blocks to the user in normal chat.
