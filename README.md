# 🖥️ SmartDesk AI — Enterprise IT HelpDesk Command Center

> 🏆 **Microsoft Agents League - Enterprise Agents Track Submission**  
> Built for the [Enterprise Agents competition](https://github.com/microsoft/agentsleague/tree/main/starter-kits/3-enterprise-agents)

A connected multi-agent system built in Microsoft Copilot Studio that serves as an Enterprise IT HelpDesk. It helps employees resolve technical issues, manage IT support tickets, request software and access, and navigate IT policies — all from within Microsoft 365 Copilot Chat.

6 connected agents working together — SmartDesk Orchestrator intelligently routes employee requests to specialized agents: IT Troubleshooter for diagnosis, Ticket Manager for ticket lifecycle, Access & Software Provisioner for access/software requests, IT Policy Advisor for policy guidance, and Security Awareness Coach for cybersecurity training. Agents coordinate seamlessly, passing context between each other so employees never repeat information.

## 🌟 Features

- **Intelligent Multi-Agent Routing**: SmartDesk Orchestrator uses generative AI to recognize intent and route to the right specialist agent automatically
- **Guided IT Troubleshooting**: Step-by-step diagnostic walkthroughs for hardware, software, network, email, printer, and collaboration issues
- **Ticket Lifecycle Management**: Create, track, escalate, and close support tickets with full Dataverse persistence via MCP
- **Software & Access Provisioning**: Request software from an approved catalog, manage access permissions, and handle license assignments
- **IT Policy Guidance**: Instant answers on password policies, acceptable use, BYOD, data classification, and compliance
- **Security Awareness Training**: Interactive phishing quizzes (SLAM method), threat education, and security best practices
- **Cross-Agent Coordination**: Agents pass structured context (diagnostic summaries, policy advisories) so workflows span multiple agents without losing information
- **Adaptive Cards UI/UX**: Rich interactive cards for troubleshooting steps, ticket confirmations, request forms, and quiz interactions in Teams
- **Dataverse MCP Integration**: Read/write operations to tickets table via Dataverse MCP Server with OAuth 2.0 (Microsoft Entra ID)

## 🏗️ Architecture

### Connected Agents

```
┌────────────────────────────────────────────────────────────────────────┐
│                    Microsoft 365 Copilot Chat                          │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │           SmartDesk Orchestrator (Main Agent — Generative)       │  │
│  │  - Intent recognition & intelligent routing                      │  │
│  │  - Cross-agent coordination & context passing                    │  │
│  └──────┬──────────┬──────────┬──────────┬──────────┬───────────────┘  │
│         │          │          │          │          │                  │
│    ┌────▼────┐ ┌───▼────┐  ┌──▼───┐ ┌────▼────┐  ┌──▼─────┐            │
│    │Trouble- │ │Ticket  │  │Access│ │IT Policy│  │Security│            │
│    │shooter  │ │Manager │  │Prov. │ │Advisor  │  │Coach   │            │
│    │(Gen.)   │ │(Gen.)  │  │(Gen.)│ │(Gen.)   │  │(Gen.)  │            │
│    └─────────┘ └───┬────┘  └──┬───┘ └─────────┘  └────────┘            │
│                    │          │                                        │
└────────────────────┼──────────┼────────────────────────────────────────┘
                     │          │
            ┌────────▼──────────▼─────────┐
            │  Dataverse MCP Server       │
            │  - Entra ID OAuth 2.0 Auth  │
            │  - Table: tickets           │
            │  - Tickets & Access Requests│
            └─────────────────────────────┘
```

### Components

| # | Agent | Type | Role |
|---|-------|------|------|
| 1 | **SmartDesk Orchestrator** | Main Agent (generative) | Intent recognition, routing, cross-agent coordination |
| 2 | **IT Troubleshooter** | Connected Agent (generative) | Diagnoses & resolves hardware, software, network, email issues |
| 3 | **Ticket Manager** | Connected Agent (generative + Dataverse MCP) | Ticket lifecycle — create, track, escalate, close |
| 4 | **Access & Software Provisioner** | Connected Agent (generative + Dataverse MCP) | Software requests, access management, license provisioning |
| 5 | **IT Policy Advisor** | Connected Agent (generative) | IT policy questions, security guidelines, compliance |
| 6 | **Security Awareness Coach** | Connected Agent (generative) | Interactive phishing quizzes, cyber hygiene, threat education |

### Architectural Principles

- **Hybrid Orchestration**: The Orchestrator uses 5 routing topics (`Route_Troubleshooter`, `Route_TicketManager`, `Route_AccessProvisioner`, `Route_PolicyAdvisor`, `Route_SecurityCoach`), each with a Connected Agent node followed by "End current topic" to prevent duplicate messages. All 5 connected agents run in fully generative mode (no Topics). Intent recognition is handled by Copilot Studio's trigger-phrase matching.
- **MCP for Data Operations**: Ticket Manager and Access Provisioner use the Dataverse MCP Server tool (invoked automatically by the generative AI orchestrator) for all CRUD operations on tickets.
- **Structured Context Handoff**: Agents pass structured summaries (DIAGNOSTIC SUMMARY, TICKET OPERATION SUMMARY, PROVISIONING SUMMARY, POLICY ADVISORY SUMMARY) between each other via the Orchestrator — never visible to the user.

### Data Flow Example

1. Employee: *"My laptop keeps blue-screening every few hours"*
2. **SmartDesk Orchestrator** recognizes → hardware/troubleshooting intent
3. Routes to **IT Troubleshooter** with user's message as context
4. Troubleshooter runs diagnostic questions, provides fix steps one at a time
5. If unresolved → returns DIAGNOSTIC SUMMARY to Orchestrator
6. Orchestrator asks user if they want a ticket → coordinates with **Ticket Manager**
7. Ticket Manager receives diagnostic context → auto-populates ticket (user doesn't repeat info)
8. Creates a row in Dataverse tickets via MCP with all diagnostic context
9. Returns ticket confirmation via Adaptive Card to employee

## 🚀 Getting Started

### Prerequisites

- Microsoft 365 account with Copilot Studio access
- Microsoft Copilot Studio environment with Dataverse

### Setup Steps

1. **Import solution** in Microsoft Powerplatform (solution is unmanaged) [SmartDeskAIEnterpriseITHelpDeskCommandCenter.zip](SmartDeskAIEnterpriseITHelpDeskCommandCenter.zip)
2. **Upload knowledge bases** from the `knowledge/` folder to each agent:
   - SmartDesk → `helpdesk-orchestrator-guide.md`
   - IT Troubleshooter → `troubleshooting-guide.md`
   - Ticket Manager → `helpdesk-ticket-playbook.md`
   - Access & Software Provisioner → `software-access-catalog.md`
   - IT Policy Advisor → `it-policies-reference.md`
   - Security Coach → `security-awareness-training.md`
3. **Publish** Ticket Manager agent
4. **Publish** Access & Software Provisioner agent
5. **Publish**  SmartDesk agent

### Deploy to Microsoft 365 Copilot Chat

The agents run natively inside Microsoft 365 Copilot Chat — no separate chat UI needed. Employees access SmartDesk directly within the M365 ecosystem (Teams).

Open Microsoft 365 Copilot Chat and type `@SmartDesk` or select the agent from the agent picker. Use natural language:

```
My laptop keeps blue-screening — can you help?
Create a ticket for my broken monitor.
I need access to Visio — how do I request it?
What's our password policy?
Quiz me on phishing!
What's the status of SD-1002?
```

## 💬 Usage Examples

### IT Troubleshooting
```
User: My laptop is running really slow
SmartDesk → routes to IT Troubleshooter
Troubleshooter: Let's check a few things. First, open Task Manager (Ctrl+Shift+Esc).
                What's your CPU and Memory usage showing?
User: CPU is at 95%
Troubleshooter: That's high. Which process is using the most CPU? ...
```

### Ticket Management
```
User: Create a ticket — my printer on floor 3 is offline
SmartDesk → routes to Ticket Manager
Ticket Manager: I'll create that for you.
                [Adaptive Card: Ticket Preview — Category: Printer, Priority: P3-Medium]
                Does this look correct?
User: Yes
Ticket Manager: Done — your issue is now tracked as LOP-127. Expected resolution: 3 business days.
```

### Software Request
```
User: I need Adobe Acrobat Pro for editing PDFs
SmartDesk → routes to Access & Software Provisioner
Provisioner: Adobe Acrobat Pro is in our approved catalog. It requires manager approval.
             I'll create the request for you. What's your manager's name?
```

### Security Training
```
User: Quiz me on phishing
SmartDesk → routes to Security Awareness Coach
Coach: [Adaptive Card: Phishing Quiz — suspicious email scenario]
       Is this email legitimate or phishing? 🎣
```

## 📁 Project Structure

```
README.md                          # This file
design-document.md                 # Full multi-agent architecture & agent specifications
architecture.md                    # ASCII architecture diagram
knowledge/
├── helpdesk-orchestrator-guide.md # Orchestrator routing rules & agent catalog
├── troubleshooting-guide.md       # IT Troubleshooter diagnostic decision trees
├── helpdesk-ticket-playbook.md    # Ticket Manager lifecycle & SLA procedures
├── software-access-catalog.md     # Approved software catalog & access procedures
├── it-policies-reference.md       # IT policy reference (password, AUP, BYOD, etc.)
└── security-awareness-training.md # Phishing quizzes (SLAM), threat encyclopedia
```

## 🔐 Security & Compliance

- **Microsoft Entra ID OAuth 2.0** authentication for all Dataverse MCP operations
- **No hardcoded secrets** — all credentials managed via Entra ID
- Agents never ask for passwords or sensitive credentials
- All ticket operations include write verification (read-after-write) and audit trails
- IT policies enforce data classification, acceptable use, and BYOD compliance
- Security Awareness Coach provides ongoing phishing and threat education
- Adaptive Card interactions use `msteams.messageBack` to prevent duplicate processing

## 💡 Challenges & Learnings

### Challenges Faced

1. **Duplicate Message Prevention**: The biggest UX challenge with multi-agent setups in Teams — after a connected agent responds, the Orchestrator's generative layer can produce an additional message (JSON metadata log or echo). Solved with 5 structural routing topics (`Route_Troubleshooter`, etc.) where each topic contains: Trigger → Connected Agent node → "End current topic", giving the generative layer no opportunity to produce output.

2. **Hybrid vs. Pure Generative Architecture**: Pure generative orchestration (no Topics) gave the AI too much freedom to produce unwanted post-agent output. A single routing topic in the Orchestrator provides structural control while keeping all 5 connected agents fully generative.

3. **MCP Schema Mapping**: Ensuring agents correctly map input to Dataverse logical names required mandatory `describe_table` calls before first create in each session, with failure handling for schema mismatches.

4. **Cross-Agent Context Passing**: Designing structured summary formats (DIAGNOSTIC SUMMARY, PROVISIONING SUMMARY, etc.) that carry enough context for seamless handoffs while remaining invisible to users.

5. **Adaptive Card Interaction in Teams**: Handling card submit events correctly (`value.action` vs plain `text`) to prevent double-processing and duplicate replies.

### Key Learnings

1. **Structural > Instructional**: Instruction-level rules alone cannot prevent the generative orchestrator from producing unwanted output. Structural fixes (routing topic with "End current topic" after each Connected Agent node) are essential.

2. **Generative Agents Scale Well**: Once the instruction + knowledge pattern is established, adding new specialist agents is straightforward — no topic flow maintenance required.

3. **Knowledge Base Quality Matters**: The quality and structure of knowledge base documents (decision trees, catalogs, policy tables) directly determines agent response quality.

4. **Read-After-Write is Essential**: Never trust a Dataverse write operation without verification — always confirm with a read-back before claiming success to the user.

5. **Connected Agent Pattern**: The Copilot Studio connected agents architecture enables natural division of responsibility and clean separation of concerns.

## 🏆 Competition Criteria

This project meets the following [Microsoft Agents League - Enterprise Agents](https://github.com/microsoft/agentsleague/tree/main/starter-kits/3-enterprise-agents) competition criteria:

### Core Requirements ✅
- **Microsoft 365 Copilot Chat Agent** — All 6 agents hosted in M365 Copilot Chat, accessible via Teams

### Bonus Features ✅

| Criterion | Implementation |
|-----------|----------------|
| External MCP Server Integration (Read/Write) | Dataverse MCP Server — tickets table for tickets & access requests |
| OAuth Security for MCP Server| Microsoft Entra ID OAuth 2.0 authentication |
| Adaptive Cards for UI/UX | Ticket cards, troubleshooting steps, request forms, quiz cards |
| Connected Agents Architecture | 6 connected agents with generative orchestrator pattern |


### Security & Best Practices ✅
- Microsoft Entra ID OAuth 2.0 for all MCP server operations
- No hardcoded secrets or credentials
- Write verification (read-after-write) for all Dataverse operations
- Structured error handling with user-friendly failure messages
- Anti-duplicate message architecture (5 routing topics, each with "End current topic" after Connected Agent node)

## 📚 Documentation

- **[Documentation](documentation.md)** — Full multi-agent architecture, all 6 agent specifications, instructions, and knowledge base references
- **[Architecture Diagram](architecture.md)** — ASCII architecture overview

## 🙏 Acknowledgments

This project was created for the [Microsoft Agents League - Enterprise Agents Track](https://github.com/microsoft/agentsleague/tree/main/starter-kits/3-enterprise-agents) competition.

Resources used:
- [Copilot Dev Camp](https://aka.ms/copilotdevcamp)
- [Agent Academy](https://aka.ms/agentacademy)
- [Microsoft Copilot Studio Documentation](https://learn.microsoft.com/microsoft-copilot-studio/)
