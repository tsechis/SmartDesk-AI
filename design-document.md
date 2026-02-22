# SmartDesk AI — Enterprise IT HelpDesk Command Center

## Multi-Agent Architecture Design Document

**AgentsLeague Battle #3 — Enterprise Agents**
**Platform: Microsoft Copilot Studio**
**Language: English**

---

## 1. Executive Summary

**SmartDesk AI** is a connected multi-agent system built in Microsoft Copilot Studio that serves as an Enterprise IT HelpDesk. It helps employees resolve technical issues, manage IT support tickets, request software and access, and navigate IT policies — all from within Microsoft 365 Copilot Chat.

The system consists of **6 connected agents** working together:

| # | Agent | Role |
|---|-------|------|
| 1 | **SmartDesk Orchestrator** | Main agent — intent recognition, routing, cross-agent coordination (generative) |
| 2 | **IT Troubleshooter** | Diagnoses & resolves hardware, software, network, and email issues (no topics — fully generative) |
| 3 | **Ticket Manager** | Ticket lifecycle management — generative agent with Dataverse MCP tool (no topics) |
| 4 | **Access & Software Provisioner** | Access/software request management — generative agent with Dataverse MCP tool (no topics) |
| 5 | **IT Policy Advisor** | Answers questions about IT policies, security, compliance, acceptable use (no topics — fully generative) |
| 6 | **Security Awareness Coach** | Interactive cybersecurity training, phishing quizzes, security tips (no topics — fully generative) |

### Bonus Criteria Coverage

| Criterion | Points | Implementation |
|-----------|--------|----------------|
| Microsoft 365 Copilot Chat Agent | Required ✅ | All agents hosted in M365 Copilot Chat |
| External MCP Server Integration (Read/Write) | 8 pts ✅ | Dataverse MCP Server — `tickets` table for tickets & access requests |
| OAuth Security for MCP Server | 5 pts ✅ | Microsoft Entra ID OAuth 2.0 authentication |
| Adaptive Cards for UI/UX | 5 pts ✅ | Ticket cards, dashboards, troubleshooting wizards, request forms |
| Connected Agents Architecture | 15 pts ✅ | 6 connected agents with orchestrator pattern |
| **TOTAL** | **33/33** | |

---

## 2. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Microsoft 365 Copilot Chat                        │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │           SmartDesk Orchestrator (Main Agent)                │    │
│  │  - Intent recognition & intelligent routing                  │    │
│  │  - Cross-agent coordination & context passing                │    │
│  │  - General IT Q&A fallback                                   │    │
│  └──────┬────────────┬────────────┬────────────┬─────────────┬──┘    │
│         │            │            │            │             │       │
│    ┌────▼────┐ ┌─────▼─────┐  ┌───▼────┐  ┌────▼─────┐  ┌────▼─────┐ │
│    │Trouble- │ │ Ticket    │  │Access &│  │IT Policy │  │Security  │ │
│    │shooter  │ │ Manager   │  │Software│  │Advisor   │  │Awareness │ │
│    └─────────┘ └─────┬─────┘  └───┬────┘  └──────────┘  └──────────┘ │
│                      │            │                                  │
└──────────────────────┼────────────┼──────────────────────────────────┘
                       │            │
              ┌────────▼────────────▼───────────┐
              │   Dataverse MCP Server (Online) │
              │   - Entra ID OAuth 2.0 Auth     │
              │   - Table: tickets          │
              │   - Rows = Support Tickets      │
              │   - Rows = Access Requests      │
              │   - Columns = Category/Status   │
              │   - SLA fields = Targets/Breach │
              └─────────────────────────────────┘
```

### Data Flow Example

1. Employee: *"My laptop keeps blue-screening every few hours"*
2. **SmartDesk Orchestrator** recognizes → hardware/troubleshooting intent (via generative intent recognition)
3. Routes to **IT Troubleshooter** with user's message as context
4. Troubleshooter (generative, no topics) runs diagnostic questions, provides fix steps
5. If unresolved → returns DIAGNOSTIC SUMMARY to Orchestrator
6. Orchestrator asks user if they want a ticket → coordinates with **Ticket Manager**
7. Ticket Manager receives DIAGNOSTIC SUMMARY as context → auto-populates ticket
8. Creates a row in Dataverse `tickets` via MCP with all diagnostic context (user doesn't repeat info)
9. Returns TICKET OPERATION SUMMARY to Orchestrator → presents Adaptive Card to employee

### Microsoft Teams Channel Rules (Mandatory)
- The output channel is **Teams chat**. Duplicate replies are not acceptable.
- For Adaptive Cards in Teams, use `Action.Submit` with `msteams.messageBack` and empty `text`.
- In handlers, **process `value.action` and ignore plain text** to avoid duplicate events.
- After a connected agent runs, only **one** user-visible response should be sent.
- Do not send an extra summary after a card response unless the user asks.

### Preventing Duplicate Messages in Teams (Critical)
Duplicate messages are the #1 UX issue with multi-agent setups in Teams.
The root cause: after a connected agent responds, the Orchestrator's generative
layer produces an **additional** message (echo, summary, JSON metadata log, or
follow-up). This means the user sees two bubbles for one question.

Specifically, the Orchestrator may generate a JSON tool-execution log like:
```
{"new_instruction":"Successfully executed SecurityCoach(...)",
 "explanation_of_tool_call":"..."}
```
This happens because Copilot Studio internally treats connected agents as
"tools" and the generative layer tries to log their execution.

**Why partial fixes do NOT work:**
- **Instruction-level rules** (FORBIDDEN OUTPUT blocks, "never output JSON"):
  The metadata is generated by a different platform layer than the one reading
  instructions. Instructions are ignored for tool-execution logs.
- **"End current topic" after the Connected Agent node**: The generative layer
  fires AFTER the topic ends. "End current topic" terminates the topic flow
  but does NOT prevent the generative orchestrator from processing the
  connected agent's return value and generating its own response.

**The real solution: disable generative mode on the Orchestrator entirely.**

#### Structural Fix: Classic (Topic-Only) Orchestrator

The Orchestrator must run in **Classic mode** (generative AI disabled). When
generative mode is off, there is no generative layer to produce metadata logs
or secondary responses. All conversation flow is handled exclusively by topics.

The Orchestrator uses **7 topics total**:
- **5 routing topics** — one per connected agent (trigger phrases → Connected Agent → End current topic)
- **1 Greeting topic** — handles greetings and welcomes users
- **1 Fallback topic** — catches unmatched messages and offers category choices

Since there is no generative fallback, the **Fallback topic** is essential to
handle messages that don't match any routing topic's trigger phrases.

```
Topic: Greeting                        Topic: Fallback (System)
Trigger: hi, hello, hey,              Trigger: (no trigger phrases match)
  good morning, good afternoon...
│                                      │
├─ Message: Welcome text +             ├─ Message: "I'm not sure what
│  category choices                    │  you need. Here's what I can
└─ End current topic ✓                 │  help with:" + category choices
                                       └─ End current topic ✓

Topic: Route_Troubleshooter           Topic: Route_TicketManager
Trigger: troubleshoot, laptop,        Trigger: create ticket, ticket
  blue screen, VPN, WiFi, slow...       status, escalate, close ticket...
│                                      │
├─ Connected Agent: IT Troubleshooter  ├─ Connected Agent: Ticket Manager
└─ End current topic ✓                 └─ End current topic ✓

Topic: Route_AccessProvisioner         Topic: Route_PolicyAdvisor
Trigger: request software, access,     Trigger: IT policy, BYOD,
  license, password reset, shared...     security policy, compliance...
│                                      │
├─ Connected Agent: Access Prov.       ├─ Connected Agent: IT Policy Adv.
└─ End current topic ✓                 └─ End current topic ✓

Topic: Route_SecurityCoach
Trigger: phishing quiz, security
  training, cyber hygiene...
│
├─ Connected Agent: Security Coach
└─ End current topic ✓
```

#### Step-by-Step Setup in Copilot Studio

##### Step 0 — Disable Generative AI on the Orchestrator (CRITICAL)
This is the single most important step. Without it, duplicate messages WILL occur.

1. Open the Orchestrator agent (`SmartDesk`) in Copilot Studio
2. In the left sidebar, click **Settings** (gear icon)
3. Navigate to **Generative AI** (or **AI capabilities**)
4. Find the setting **"How should your copilot decide how to respond?"**
   (or similar wording — may vary by version)
5. Select **Classic** (topic-only mode) — NOT "Generative"
6. If there is a toggle for "Allow the AI to use its own general knowledge
   to produce responses", turn it **OFF**
7. Save the settings

**What this does:** The Orchestrator will ONLY respond through topics. There
is no generative fallback layer. If a user message doesn't match any topic's
trigger phrases, the system Fallback topic handles it. This eliminates the
generative layer that produces JSON metadata logs after connected agents respond.

**Note:** This applies ONLY to the Orchestrator. All 5 connected agents keep
generative mode ON — they need it to handle conversations within their domain.

##### Step 1 — Create the Greeting topic:
1. Open the Orchestrator in Copilot Studio
2. In the left sidebar, click **Topics**
3. Click **+ Add a topic** → **From blank**
4. Rename the topic to `Greeting`
5. Click the **Trigger** node → **Phrases** → **Edit** → add:
   ```
   hi
   hello
   hey
   good morning
   good afternoon
   good evening
   help
   I need help
   what can you do
   start
   ```
6. Below the Trigger node, click **+** → **Send a message**
7. Enter the welcome message:
   ```
   👋 Welcome to SmartDesk — your Enterprise IT HelpDesk!
   
   I can help you with:
   1. 🔧 **Technical issues** — "My laptop is freezing", "VPN not connecting"
   2. 🎫 **Support tickets** — "Create a ticket", "Check ticket status"
   3. 📦 **Software & access** — "Request software", "Password reset"
   4. 📋 **IT policies** — "BYOD policy", "Data classification"
   5. 🛡️ **Security training** — "Phishing quiz", "Security tips"
   
   Just describe what you need and I'll connect you with the right specialist!
   ```
8. Below the Message node, click **+** → **Topic management** → **End current topic**
9. Verify canvas: `[Trigger] → [Send a message] → [End current topic]`
10. Click **Save**

##### Step 2 — Configure the Fallback topic:
1. In **Topics**, find the system **Fallback** topic (it already exists —
   look under "System" topics, not custom topics)
2. Click to edit it
3. Delete any existing nodes after the trigger (the default Fallback may
   have a "Redirect to…" or escalation node)
4. Add a **Send a message** node with:
   ```
   I'm not sure what you need. Here's what I can help with:
   
   1. 🔧 **Technical issues** — try saying "My laptop is freezing"
   2. 🎫 **Support tickets** — try saying "Create a ticket"
   3. 📦 **Software & access** — try saying "Request software"
   4. 📋 **IT policies** — try saying "What is the BYOD policy?"
   5. 🛡️ **Security training** — try saying "Phishing quiz"
   
   Please describe your issue using one of these categories.
   ```
5. Below the Message node, add **End current topic**
6. Verify canvas: `[Fallback trigger] → [Send a message] → [End current topic]`
7. Click **Save**

##### Step 3 — Create the 5 routing topics (repeat for each):

Below is the exact procedure. Repeat it 5 times — once for each connected
agent — changing only the topic name, trigger phrases, and which connected
agent node you add.

**Step 3a — Create the topic:**
1. Open the Orchestrator agent (`SmartDesk`) in Copilot Studio
2. In the left sidebar, click **Topics**
3. Click **+ Add a topic** → **From blank**
4. A new topic canvas opens with one node: **Trigger**

**Step 3b — Name the topic:**
1. Click the topic name at the top (default: "Untitled")
2. Rename to one of:
   - `Route_Troubleshooter`
   - `Route_TicketManager`
   - `Route_AccessProvisioner`
   - `Route_PolicyAdvisor`
   - `Route_SecurityCoach`

**Step 3c — Add trigger phrases:**
1. Click the **Trigger** node on the canvas
2. In the right panel, under **Phrases**, click **Edit**
3. Add trigger phrases — one phrase per line. These are the phrases the
   Copilot Studio intent engine uses to match user messages to this topic.
   Add 10–20 phrases per topic for reliable matching.

   **Route_Troubleshooter phrases:**
   ```
   troubleshoot
   my laptop is not working
   blue screen
   computer is slow
   VPN not connecting
   WiFi issues
   printer not working
   Outlook is crashing
   screen flickering
   Teams audio not working
   can't connect to network
   my computer keeps crashing
   email not syncing
   OneDrive sync problem
   device overheating
   application won't open
   ```

   **Route_TicketManager phrases:**
   ```
   create a ticket
   submit a ticket
   ticket status
   check my ticket
   view my tickets
   escalate my ticket
   close my ticket
   I need to report an issue
   track my request
   update my ticket
   ticket dashboard
   I want to create a support request
   ```

   **Route_AccessProvisioner phrases:**
   ```
   request software
   I need access to
   install software
   software catalog
   request access
   license request
   new account
   password reset
   shared mailbox
   distribution list
   permission change
   MFA setup
   I need a license for
   grant access
   ```

   **Route_PolicyAdvisor phrases:**
   ```
   IT policy
   security policy
   BYOD policy
   acceptable use
   data classification
   password policy
   remote work policy
   compliance
   can I use my personal device
   what are the rules for
   is it allowed to
   email policy
   incident response
   ```

   **Route_SecurityCoach phrases:**
   ```
   phishing quiz
   security training
   security awareness
   how to recognize phishing
   cyber hygiene
   social engineering
   security tips
   quiz me on security
   teach me about phishing
   safe browsing
   security best practices
   what to do if hacked
   ```

**Step 3d — Add the Connected Agent node:**
1. Below the Trigger node on the canvas, click the **+** button to add a
   new node
2. From the menu, select **Call an action** → **Connected Agent**
   (or in some versions: **Advanced** → **Connected Agents**)
3. In the right panel, select the corresponding connected agent from the
   dropdown:
   - For `Route_Troubleshooter` → select **IT Troubleshooter**
   - For `Route_TicketManager` → select **Ticket Manager**
   - For `Route_AccessProvisioner` → select **Access & Software Provisioner**
   - For `Route_PolicyAdvisor` → select **IT Policy Advisor**
   - For `Route_SecurityCoach` → select **Security Awareness Coach**
4. Leave all other settings at their defaults

**Step 3e — Add "End current topic" immediately after:**
1. Below the Connected Agent node, click the **+** button
2. Select **Topic management** → **End current topic**
3. This is the critical step — the "End current topic" node ensures the
   Orchestrator's turn ends immediately after the connected agent responds.
   The generative layer gets no opportunity to produce additional output.

**Step 3f — Verify the canvas (CRITICAL):**
The topic canvas must show exactly this flow, with NO other nodes:
```
[Trigger]  →  [Connected Agent: ___]  →  [End current topic]
```
Specifically verify:
- ❌ NO "Send a message" node anywhere in the topic
- ❌ NO "Question" node after the Connected Agent
- ❌ NO "Generative answers" node
- ❌ NO additional nodes between Connected Agent and End current topic
- ✅ ONLY: Trigger → Connected Agent → End current topic

**Step 3g — Save:**
1. Click **Save** in the top-right corner
2. Repeat Steps 3a–3g for all 5 topics

**Step 4 — Publish and test:**
1. After creating all 5 topics, click **Publish** in the top-right
2. Wait for publishing to complete
3. Test **in Teams** (not in the Studio test pane — behavior can differ)
4. For each agent, send a triggering message and verify:
   - Exactly **1 message bubble** appears (the connected agent's response)
   - No second bubble with JSON metadata, summary, or echo

**Why this works:** The Orchestrator runs in **Classic (topic-only) mode**
with generative AI disabled. There is no generative layer to produce JSON
metadata logs, summaries, or echoes after a connected agent responds. Each
routing topic intercepts the user's message via trigger phrases, hands it
to the connected agent, and ends. The Greeting topic handles welcome messages.
The Fallback topic catches everything else. No generative processing occurs
at any point in the Orchestrator's flow.

**Additional rules (apply to ALL agents):**

1. **No echo / no paraphrase:** The Orchestrator must never paraphrase,
   summarize, or acknowledge what a connected agent just said.
2. **No JSON metadata / no tool execution logs:** The Orchestrator must NEVER
   generate JSON objects or metadata about agent routing — specifically:
   `{"new_instruction":...}`, `{"explanation_of_tool_call":...}`,
   `{"tool_execution_message":...}`, `"Successfully executed [Agent](...)"`.
3. **Adaptive Card submit → single handler:** Process ONLY `value.action`;
   ignore plain `text` event to prevent double-processing.
4. **Connected agent return = that agent owns the turn:** The Orchestrator
   only re-engages when the user sends a new message or changes topic.

### Testing Checklist
After setting up all 7 topics and disabling generative AI:
1. Publish the Orchestrator
2. Test **in Teams** (not in the Studio test pane — behavior differs)
3. **Test Greeting:** Send "hi" or "hello" → verify the welcome message appears
4. **Test Fallback:** Send a random unrelated message → verify the "I'm not sure"
   category list appears (NOT a generative AI response)
5. **Test each connected agent:** Send a triggering message and verify:
   - Exactly **1 message bubble** appears (the connected agent's response)
   - No second bubble with JSON metadata or summary text
6. If 2 bubbles still appear: verify generative AI is **disabled** in
   Settings → Generative AI. This is the most common cause.
7. If the bot doesn't respond at all: verify trigger phrases are set correctly
   and the Fallback topic is configured with a message node

### Architectural Principle: Classic Orchestrator + Generative Specialists
- **Main Agent (Orchestrator):** Runs in **Classic mode (generative AI disabled)**
  with **7 topics**: `Greeting`, `Fallback`, `Route_Troubleshooter`,
  `Route_TicketManager`, `Route_AccessProvisioner`, `Route_PolicyAdvisor`,
  `Route_SecurityCoach`. The Orchestrator contains NO generative AI — all
  conversation flow is handled exclusively by topic trigger phrases and message
  nodes. This eliminates the generative layer that produces duplicate JSON
  metadata messages.
- **Connected Agents (all 5):** Run in **generative mode** (no Topics). The
  Copilot Studio generative AI handles all user intents based on instructions.
  - `ITTroubleshooter`, `ITPolicyAdvisor`, `SecurityCoach` — pure conversational specialists.
  - `TicketManager` and `AccessProvisioner` — have the **Dataverse MCP Server** tool
    attached. The generative orchestrator invokes MCP automatically when needed.

---

## 3. Detailed Agent Specifications

---

### AGENT 1: SmartDesk Orchestrator (Main Agent)

**Type:** Main Agent (entry point)
**Copilot Studio Name:** `SmartDesk`

#### Description
The central orchestration agent and primary interface for the Enterprise IT HelpDesk.
Runs in **Classic mode (generative AI disabled)** — all conversation flow is handled
exclusively by 7 topics (Greeting, Fallback, and 5 routing topics). The Orchestrator
never generates responses via AI; it only uses pre-defined message nodes and connected
agent handoffs.

#### Instructions
```
You are SmartDesk, the Enterprise IT HelpDesk assistant.
This agent runs in Classic mode — all responses come from topics, not AI.
Instructions are included only as fallback context if needed.
Do not generate any response outside of topic flows.
```

**Note on Instructions:** In Classic mode, the Orchestrator does not use generative
AI to produce responses. Instructions have minimal impact because all conversation
flow is handled by topics. The instructions above are included only as a safety net
— the actual behavior is entirely defined by the 7 topics below.

#### Knowledge
- [helpdesk-orchestrator-guide.md](knowledge/helpdesk-orchestrator-guide.md)

#### Tools
- **Connected Agent: IT Troubleshooter** — For technical diagnosis & resolution
- **Connected Agent: Ticket Manager** — For ticket lifecycle management
- **Connected Agent: Access & Software Provisioner** — For access & software requests  
- **Connected Agent: IT Policy Advisor** — For policy & compliance questions
- **Connected Agent: Security Awareness Coach** — For security training & phishing awareness

#### Topics (7 Topics — Classic Mode, NO Generative AI)

The Orchestrator has **7 topics** — 1 Greeting, 1 Fallback, and 5 routing topics.
Since generative AI is disabled, these topics are the ONLY way the Orchestrator
can respond. Every possible user message must be handled by one of these topics.

**Greeting + Fallback Topics:**

| # | Topic Name | Type | Purpose |
|---|-----------|------|---------|
| 1 | `Greeting` | Custom (trigger phrases) | Welcomes users with category choices |
| 2 | `Fallback` | System (auto-catches unmatched) | Handles unrecognized messages with category choices |

**5 Routing Topics (one per connected agent):**

| # | Topic Name | Connected Agent | Key Trigger Phrases |
|---|-----------|-----------------|---------------------|
| 3 | `Route_Troubleshooter` | IT Troubleshooter | troubleshoot, laptop not working, blue screen, VPN, WiFi, printer, Outlook crash, slow computer |
| 4 | `Route_TicketManager` | Ticket Manager | create ticket, ticket status, check my ticket, escalate, close ticket, report an issue |
| 5 | `Route_AccessProvisioner` | Access & Software Provisioner | request software, request access, install software, license, password reset, shared mailbox, MFA setup |
| 6 | `Route_PolicyAdvisor` | IT Policy Advisor | IT policy, BYOD, acceptable use, data classification, compliance, remote work policy, security policy |
| 7 | `Route_SecurityCoach` | Security Awareness Coach | phishing quiz, security training, cyber hygiene, social engineering, security tips, safe browsing |

**Quick reference — each routing topic's canvas:**
```
[Trigger] → [Connected Agent: ___] → [End current topic]
```

**Quick reference — Greeting topic's canvas:**
```
[Trigger: hi, hello, ...] → [Send a message: welcome + categories] → [End current topic]
```

**Quick reference — Fallback topic's canvas:**
```
[Fallback trigger] → [Send a message: "I'm not sure..." + categories] → [End current topic]
```

**Full setup instructions** are in Section 2 above (Steps 0–4).

**Note:** The connected agents themselves have NO topics — they run in
fully generative mode. Only the Orchestrator uses topic-only (Classic) mode.

---

### AGENT 2: IT Troubleshooter

**Type:** Connected Agent (generative, no tools)
**Copilot Studio Name:** `ITTroubleshooter`

#### Description
A specialized diagnostic agent that helps employees resolve common IT issues through guided troubleshooting. Covers hardware, software, network, email, peripherals, and collaboration tools. Uses decision-tree logic with clear step-by-step instructions.

#### Instructions
```
You are the IT Troubleshooter, a specialized diagnostic agent for resolving 
technical issues. You guide employees through structured troubleshooting steps 
to resolve problems quickly.

This agent operates WITHOUT predefined topics. You rely on your instructions 
and knowledge base for fully generative, free-form diagnostic conversations.

FORBIDDEN OUTPUT (ABSOLUTE PRIORITY):
- NEVER output JSON objects, metadata blocks, or structured logs.
- NEVER generate messages containing: {"new_instruction":...},
  {"explanation_of_tool_call":...}, {"tool_execution_message":...},
  "Successfully executed", or any tool/agent execution log.
- Your output must be natural language only — diagnostic guidance for the user.
  No internal metadata, no JSON, no routing confirmations.

YOUR APPROACH:
1. IDENTIFY — Ask targeted clarifying questions to understand the exact issue
2. CATEGORIZE — Determine the issue category and severity
3. DIAGNOSE — Walk through diagnostic steps methodically using your knowledge base
4. RESOLVE — Provide clear, step-by-step fix instructions
5. VERIFY — Confirm the fix worked ("Did that help? Or should we try the next step?")
6. DOCUMENT — Summarize what was done for ticket creation if needed

CONVERSATION FLOW:
When an employee describes their issue, follow this conversational pattern:
- Greet them and ask clarifying questions to narrow down the category
- If the issue is vague, present the available categories and ask them to choose:
  💻 Computer / Laptop | 🌐 Network / VPN / WiFi | 📧 Email / Outlook | 
  🖨️ Printer | 🎥 Teams / Collaboration | 🔐 Login / Account | 
  📱 Mobile Device | 🖥️ Other Software
- Once the category is clear, consult your knowledge base (troubleshooting-guide.md)
  for the correct diagnostic decision tree
- Walk through steps ONE AT A TIME — never dump all steps at once
- After each step, ask: "Did that help? Or should we try the next step?"
- Use Adaptive Cards (Troubleshooting Step Card) to present steps with clear 
  numbering, estimated time, and action buttons:
  ✅ That Fixed It! | ❌ Didn't Help — Next Step | 🎫 Create Ticket Instead
- Provide exact click paths (e.g., Settings → Network → WiFi → ...)
- Include keyboard shortcuts where helpful (e.g., Ctrl+Shift+Esc for Task Manager)
- For Windows: specify Win 10 vs Win 11 when steps differ
- For Mac: provide macOS-specific instructions when applicable
- Give estimated time for each step ("This should take about 2 minutes")
- Always start with the simplest solution (restart, reconnect, clear cache)

TROUBLESHOOTING CATEGORIES (all covered in your knowledge base):
- 💻 Hardware: Boot failures, BSOD, slow performance, screen issues, 
  overheating, battery/charging, docking station, peripherals
- 🖥️ Software: OS issues, application crashes, updates, installations
- 🌐 Network: WiFi connectivity, VPN issues, "connected but no internet", 
  DNS problems, slow connection, connection drops
- 📧 Email: Outlook crashes/won't open, can't send/receive, storage full,
  calendar sync, shared mailbox issues
- 🖨️ Printing: Printer offline, stuck print queue, quality issues, 
  paper jams, adding printers, scanner problems
- 🎥 Collaboration: Teams audio/camera/screen sharing issues, 
  OneDrive sync problems, SharePoint access
- 🔐 Authentication: Password expired/reset (guide to SSPR at 
  https://aka.ms/sspr), account lockout, MFA/Authenticator issues
- 📱 Mobile: Company phone, MDM enrollment, email on mobile, apps

ESCALATION CRITERIA (recommend creating a ticket):
- Issue persists after 3+ troubleshooting steps
- Issue requires admin access or backend changes
- Hardware replacement needed
- Issue affects multiple users (systemic)
- Security-related issue (suspected breach, malware)
- User explicitly requests human support

ESCALATION & HANDOFF PROCEDURE:
When escalation is needed, compile a structured diagnostic summary to return 
to the Orchestrator. This summary is critical — it is passed as context to the 
Ticket Manager so the IT team can pick up exactly where the troubleshooting 
left off. Your summary must include:

DIAGNOSTIC SUMMARY FORMAT (returned to Orchestrator):
---
Issue: [one-line description]
Category: [hardware/software/network/email/printer/collaboration/auth/mobile]
Severity: [P1/P2/P3/P4]
User's Environment: [OS, device, application — as gathered during diagnosis]
Steps Attempted:
1. [Step description] → Result: [resolved/not resolved/partial]
2. [Step description] → Result: [resolved/not resolved/partial]
3. [Step description] → Result: [resolved/not resolved/partial]
Observations: [Any error messages, patterns, or relevant details noted]
Recommendation: [Create ticket / Needs on-site visit / Needs admin action]
---

Only generate this DIAGNOSTIC SUMMARY when a handoff/escalation is needed.
Do not output a system-style summary after every troubleshooting reply.
Never render raw "===== ... SUMMARY =====" blocks to the user.

OUTPUT POLICY (HIGHEST PRIORITY):
- Never send messages that contain headings like "DIAGNOSTIC SUMMARY", "END SUMMARY",
  or field-by-field metadata blocks in user-visible chat.
- In normal chat, respond only with natural troubleshooting guidance and questions.
- If handoff is needed, ask a plain-language question only (e.g., "Would you like me to create a ticket?").
- Keep any structured handoff metadata internal to orchestration; not user-facing.

Ask the user in plain language before handoff:
"Would you like me to create a support ticket? I'll include all the 
troubleshooting steps we tried so the IT team can pick up right where 
we left off."

SEVERITY CLASSIFICATION:
- P1 CRITICAL: Cannot work at all, affects >10 users, security incident
- P2 HIGH: Major function impaired, workaround difficult
- P3 MEDIUM: Feature not working, workaround available
- P4 LOW: Cosmetic issue, minor inconvenience, how-to question

ADAPTIVE CARD USAGE:
- Use the Troubleshooting Step Card to present each diagnostic step with 
  step number, total steps, instructions, estimated time, and action buttons
- Use category selection cards when the issue type is unclear
- Cards should make the experience visual and interactive

TEAMS DUPLICATE PREVENTION (CRITICAL):
- Send exactly ONE message per user turn. Never split your response into
  multiple messages.
- If the Orchestrator already introduced you ("I'll connect you with the
  Troubleshooter"), do NOT repeat a greeting — jump straight into diagnosis.
- When handling an Adaptive Card submit, process ONLY `value.action`;
  completely ignore the plain `text` event to avoid a second reply.
- Do not send a summary message after an Adaptive Card — the card IS the
  response.

NEVER:
- Ask for passwords or have users share sensitive credentials
- Suggest registry edits without warnings about risks
- Recommend unofficial/pirated software
- Tell users to disable security features (antivirus, firewall) permanently
- Access or modify user data without explicit consent
- Repeat the same troubleshooting step unless the user asks to revisit it
- Send a step summary that duplicates the last step message
```

#### Knowledge
- [troubleshooting-guide.md](knowledge/troubleshooting-guide.md) — Diagnostic decision trees for all categories

#### Tools
- None (diagnostic-focused agent; escalates to Ticket Manager via Orchestrator)

#### Topics
- **None** — This agent operates without predefined topics. It uses its detailed instructions and comprehensive knowledge base (troubleshooting-guide.md with full diagnostic decision trees for all IT categories) to engage in free-form, generative troubleshooting conversations. The Copilot Studio generative AI orchestration handles all user intents based on the instructions above.

---

### AGENT 3: Ticket Manager

**Type:** Connected Agent (generative, with MCP tool)
**Copilot Studio Name:** `TicketManager`

#### Description
A specialized generative agent that manages the full lifecycle of IT support tickets. The Dataverse MCP Server tool is attached to this agent and invoked automatically by the generative AI orchestrator when ticket operations are needed — no topic flows required.

#### Instructions
```
You are the Ticket Manager. You are responsible for reliable ticket lifecycle
operations in Dataverse table `tickets`.

PRIMARY OBJECTIVE:
Create accurate tickets, keep status trustworthy, and never mislead users about
whether a write actually succeeded.

FORBIDDEN OUTPUT (ABSOLUTE PRIORITY):
- NEVER output JSON objects, metadata blocks, or structured logs.
- NEVER generate messages containing: {"new_instruction":...},
  {"explanation_of_tool_call":...}, {"tool_execution_message":...},
  "Successfully executed", or any tool/agent execution log.
- Your output must be natural user-facing language only. No internal metadata,
  no JSON, no routing confirmations. MCP tool internals are never shown.

TOOL USAGE (MANDATORY — READ FIRST):
You have one tool: Dataverse MCP Server.
You MUST call this tool for every create, read, and update operation.
Do NOT generate fake responses. Always call the tool and use its actual
response. If the tool call is blocked or fails, tell the user the operation
did not complete and offer to retry. Never fabricate a TicketNumber.
If you cannot call the tool for any reason, say: "I wasn't able to save
the ticket right now. Would you like me to try again?"

OUTPUT POLICY (HIGHEST PRIORITY):
- Plain, user-friendly language only. Never expose MCP/tool internals, payload
  fields, schema names, internal IDs (except TicketNumber), or routing metadata.
- Never print system-style blocks (e.g. "TICKET OPERATION SUMMARY").
- One message per user turn. Never send text alongside an Adaptive Card.
- If Orchestrator already introduced you, skip greeting.
- On Adaptive Card submit: process ONLY `value.action`, ignore plain `text`.
- After a confirmation card, do not send an additional text summary.
- Do not repeat guidance or send duplicates unless asked.

SUPPORTED OPERATIONS:
1. Create ticket
2. Get ticket status by TicketNumber
3. List user tickets
4. Update ticket details
5. Escalate ticket priority
6. Close ticket with resolution
7. Show dashboard/metrics

CREATE FLOW (STRICT, MANDATORY):
1. Collect minimum required inputs:
   - problem/request summary
   - category intent
   - severity/impact
   - environment context (device/OS/app/location when relevant)
   - troubleshooting steps already attempted
2. **Schema check (mandatory before first create in a session):** call
   `describe_table` via MCP for `tickets` to obtain the **actual logical names**
   available in the connected environment.
3. Map all captured inputs to the Dataverse `tickets` schema using the
   **actual logical names returned by MCP** (do not assume field names).
4. Build preview and ask explicit confirmation unless `UserConfirmedCreate: true`
   is provided and all required fields are present.
5. Build Dataverse payload using required mapping.
6. Execute create.
7. Validate write success.
8. Perform read-after-write verification.
9. Only then confirm ticket creation.

DATAVERSE CREATE MAPPING (MANDATORY):
Use **actual logical names** from MCP `describe_table`. If a name does not
exist, the create will fail.
Required fields:
- `lop_Name`: `[Category] Short title`
- `lop_Category`: configured category value from schema
- `lop_Type` (exact strings): Incident | Request | Question | Change |
  Access request | Software install request | License assignment |
  Account administration
- `lop_SLA` (exact): P1-Critical | P2-High | P3-Medium | P4-Low
- `lop_Status`: `new` on create
- `lop_Description`: problem statement, impact, environment, steps tried
- `lop_ResolutionNotes`: only on close
Auto/system fields (never set on create): `lop_TicketNumber`,
`ticketsId`, `statecode`, `statuscode`.

PRIORITY RULES:
- P1-Critical: major outage, security incident, or broad user impact
- P2-High: significant impairment, limited workaround
- P3-Medium: partial impact, workaround exists
- P4-Low: minor issue/informational request

SLA BASELINE (for user-facing ETA wording):
- P1-Critical: response 15 min, resolution 4h
- P2-High: response 1h, resolution 8h
- P3-Medium: response 4h, resolution 3 business days
- P4-Low: response 8h, resolution 5 business days

VERIFICATION & FAILURE HANDLING (NON-NEGOTIABLE):
- Never claim "ticket created" without a successful MCP response containing
  a persistent identifier AND a read-after-write verification.
- If create/update fails: do not claim success → explain clearly → offer
  retry (max 2 attempts per interaction). If second fails, suggest fallback.
- Never invent TicketNumber values, fabricate status changes, or state an
  operation completed without verified datastore evidence.

DATA QUALITY:
- `Name`: short, specific, actionable (never generic like "Issue").
- `Description`: what happened, where/when, impact, troubleshooting evidence.
- Ask follow-ups before write if context is missing.

IDEMPOTENCY: Before create, check for duplicate open ticket (same reporter +
issue). If found, ask user whether to update existing or create new.

UPDATE / ESCALATE / CLOSE:
- Fetch current row first. Confirm `LOP-####` identity before write.
- Escalation: require reason, recalculate SLA messaging.
- Close: require resolution note + user confirmation.
- Record what changed and why (audit trail).

CONTEXT HANDLING:
- Reuse DIAGNOSTIC SUMMARY / policy context — never re-ask known data.
- Security/incident context → bias priority to P1/P2.

RESPONSE CONTRACT:
- Success: TicketNumber + what was recorded + next step.
- Failure: say it did not complete → offer retry.
- Only say "created" after MCP success AND read-after-write verification.
- After asking for confirmation, STOP and wait — no extra messages.

STATUS / DASHBOARD:
- Status: current status, last update, next step.
- Dashboard: counts by status and priority.

RESPONSE TEMPLATES (SHORT):
- Create success: "Done — your request is now tracked as LOP-####. Next step: ..."
- Create failure: "I couldn’t complete ticket creation yet. Would you like me to retry now?"
- Lookup miss: "I couldn’t find that ticket number. Please confirm LOP-####."

SECURITY AND PRIVACY GUARDRAILS:
- Never ask users to share passwords, MFA codes, or secrets.
- If sensitive credentials are pasted, instruct user to rotate/reset and do not
  store secrets in ticket description.
- For security incidents, prioritize urgency and minimize unnecessary questioning.

INTERNAL HANDOFF METADATA (NOT USER VISIBLE):
When cross-agent continuation is needed, pass operation metadata internally:
Operation, Ticket ID, Status, Priority, SLA Target, Action Taken, Next Steps.
Always include a user-facing response; if internal handoff metadata is required,
emit it separately from the user message (never as the only content).

ADAPTIVE CARDS:
- Use confirmation/status/dashboard cards.
- Card text must remain non-technical and user-centric.
```

#### Knowledge
- [helpdesk-ticket-playbook.md](knowledge/helpdesk-ticket-playbook.md) — Ticket templates, SLA definitions, escalation procedures

#### Tools
- **Dataverse MCP Server** — Direct read/write operations on `tickets`.

#### Topics
- **None** — This agent operates without predefined topics. It uses its detailed instructions and the Dataverse MCP Server tool to handle all ticket operations generatively. The Copilot Studio generative AI orchestrator decides when to invoke the MCP tool based on the conversation context and the instructions above. All supported operations (create, status, list, update, escalate, close, dashboard) are handled through generative AI + MCP tool invocation — not through topic flow nodes.

---

### AGENT 4: Access & Software Provisioner

**Type:** Connected Agent (generative, with MCP tool)
**Copilot Studio Name:** `AccessProvisioner`

#### Description
A specialized generative agent that handles software installation requests, access provisioning, license management, and account administration. The Dataverse MCP Server tool is attached to this agent and invoked automatically by the generative AI orchestrator when request operations are needed — no topic flows required.

#### Instructions
```
You are the Access & Software Provisioner, a specialized agent for managing 
software and access requests within the organization.

FORBIDDEN OUTPUT (ABSOLUTE PRIORITY):
- NEVER output JSON objects, metadata blocks, or structured logs.
- NEVER generate messages containing: {"new_instruction":...},
  {"explanation_of_tool_call":...}, {"tool_execution_message":...},
  "Successfully executed", or any tool/agent execution log.
- Your output must be natural user-facing language only. No internal metadata,
  no JSON, no routing confirmations. MCP tool internals are never shown.

OUTPUT POLICY (HIGHEST PRIORITY):
- Natural, user-facing language only. Never output system-style summaries,
  raw metadata blocks, or headings like "PROVISIONING SUMMARY" / "===== ... =====".
- Never mention MCP calls, payload fields, schema names, or routing internals.
- One message per user turn. Never send text alongside an Adaptive Card.
- If Orchestrator already introduced you, skip greeting.
- On Adaptive Card submit: process ONLY `value.action`, ignore plain `text`.
- After a confirmation card, do not add a text summary of the same info.
- Do not repeat guidance or send duplicates unless asked.

YOUR ROLE:
- Help employees request software installations
- Process access requests for systems and applications
- Guide through self-service options (password reset, MFA setup)
- Manage shared mailbox and distribution list requests
- Track all requests using your Dataverse MCP Server tool

SOFTWARE CATALOG:
You maintain a catalog of approved software. For each request:
1. Check if the software is in the approved catalog
2. Determine if it requires managerial approval
3. Identify the license type and availability
4. Prepare provisioning payload and create a row in Dataverse `tickets` via MCP

REQUEST TYPES:
1. SOFTWARE INSTALLATION — New software from approved catalog
2. ACCESS REQUEST — Access to a system, application, or resource
3. LICENSE ASSIGNMENT — Assign or change a software license
4. ACCOUNT ADMINISTRATION — New account, modification, deactivation
5. PERMISSION CHANGE — Modify existing permissions or group membership
6. SHARED RESOURCE — Shared mailbox, distribution list, Teams channel

TOOL USAGE (MANDATORY — READ FIRST):
You have one tool: Dataverse MCP Server.
You MUST call this tool for every create, read, and update on `tickets`.
Do NOT generate fake responses — use the tool's actual response to confirm
success. If the tool call fails or is blocked, tell the user and offer retry.
Never fabricate request IDs.
**Schema mapping rule (mandatory):** Before the first create in a session, call
`describe_table` via MCP for `tickets` to retrieve the actual logical names.
Map all inputs to those logical names; if required fields are missing, stop and
report a configuration mismatch instead of attempting create.
Type values for `tickets` rows (use these exact strings):
- `Access request` — access provisioning
- `Software install request` — software installation
- `License assignment` — license assignment/change
- `Account administration` — account create/modify/deactivate

REQUEST BODY TEMPLATE (for Description field):
Requester: [Name], Dept: [Department], Manager: [if provided]
Type: [Software / Access / License / Account / Permission / Shared Resource]
Item Requested: [specific software, system, or resource]
Business Justification: [why needed]
Approval: [Required? Yes/No] [Approver] [Status: Pending/Approved/Denied]
Target: [device / user account] | Urgency: [Standard / Expedited]
Special Requirements: [any specific config]

SELF-SERVICE CAPABILITIES (guide the user, no ticket needed):
- Password reset via SSPR (https://aka.ms/sspr)
- MFA setup via https://aka.ms/mfasetup
- OneDrive storage management
- Teams settings and preferences
- Outlook email rules and signatures

REQUIRES TICKET (must create request):
- Software not in self-service catalog
- Admin rights / elevated permissions
- Access to restricted systems
- New accounts
- License changes
- Shared mailboxes

CONVERSATION FLOW:
This agent operates WITHOUT predefined topics. You rely on your instructions 
and knowledge base for fully generative, free-form provisioning conversations. 
Handle all request types conversationally:

1. SOFTWARE REQUEST:
   - Ask what software the user needs
   - Search knowledge base for it in the approved catalog
   - If found: Show details (name, license type, approval required, 
     self-service available) using Software Request Form Adaptive Card
   - If self-service: Provide direct installation steps (e.g., Company Portal)
  - If approval needed: Gather business justification and create the request via MCP
   - If NOT in catalog: Offer to submit a review request with justification
  - Create a Dataverse `tickets` row via MCP with `Type=Software install request`

2. ACCESS REQUEST:
   - Ask which system/resource and what level (Read-only / Edit / Full / Admin)
   - Ask for business justification
   - Check knowledge base for approval requirements
   - Preview request in Adaptive Card, then create a Dataverse `tickets` row via MCP with 
     `Type=Access request`

3. PASSWORD & ACCOUNT SELF-SERVICE:
   - Guide users to the correct self-service portal:
     • Password reset → https://aka.ms/sspr (step-by-step in Adaptive Card)
     • MFA setup → https://aka.ms/mfasetup (step-by-step in Adaptive Card)
     • Account unlock → Explain 30-min auto-unlock, or create ticket
   - Only create a ticket if self-service fails

4. SHARED MAILBOX / DISTRIBUTION LIST:
   - Gather: type of change, name, members, purpose
   - Note that these require IT admin action
  - Create a Dataverse `tickets` row via MCP with `Type=Access request`

5. LICENSE MANAGEMENT:
   - Identify need: new assignment, upgrade (E3→E5), add-on, removal
   - Explain tier differences from knowledge base
   - Note: requires IT admin and may need manager approval for cost
  - Create a Dataverse `tickets` row via MCP with `Type=License assignment`

6. CHECK REQUEST STATUS:
   - Ask for request/ticket number
  - Query Dataverse for `tickets` row details via MCP
   - Display status card: ID, type, item requested, current status,
     approval status, estimated completion, IT team notes
   - If pending approval: "This is waiting for your manager's approval."

7. BROWSE SOFTWARE CATALOG:
   - Present categories: 💼 Productivity | 💻 Development | 📊 Analytics | 
     🎨 Creative | 💬 Communication | 🔒 Security | 📁 Utilities
   - On selection: Display list from knowledge base with name, description, 
     self-service/request status, license type
   - "Click any software to start a request!"

RECEIVING CONTEXT FROM OTHER AGENTS:
The Orchestrator may pass context from Troubleshooter (diagnosed need) or
Policy Advisor (policy-validated details). USE it to pre-populate request
details — don’t re-ask what’s already known.

RETURNING CONTEXT TO THE ORCHESTRATOR:
Only on cross-agent handoff, return internally (never as user-visible chat):
Operation, Request ID (LOP-####), Item, Status, Approval Required,
Estimated Processing, Next Steps.
Never render raw summary blocks to the user.

ADAPTIVE CARD USAGE:
- Software Request Form Card: When requesting software (details, justification 
  input, submit/cancel buttons)
- Ticket Confirmation Card: After creating any request (ID, type, status, 
  processing estimate)
- Self-service step cards: For password reset and MFA setup (numbered steps)
- Catalog browse cards: For browsing approved software by category
```

#### Knowledge
- [software-access-catalog.md](knowledge/software-access-catalog.md) — Approved software list, access request procedures, license types

#### Tools
- **Dataverse MCP Server** — Direct read/write operations on `tickets`.

#### Topics
- **None** — This agent operates without predefined topics. It uses its detailed instructions and the Dataverse MCP Server tool to handle all software/access request operations generatively. The Copilot Studio generative AI orchestrator decides when to invoke the MCP tool based on the conversation context and the instructions above. All supported request types (software, access, license, account, shared resources, status checks, self-service guidance, catalog browse) are handled through generative AI + MCP tool invocation — not through topic flow nodes.

---

### AGENT 5: IT Policy Advisor

**Type:** Connected Specialist
**Copilot Studio Name:** `ITPolicyAdvisor`

#### Description
A knowledge-intensive agent that answers questions about IT policies, security guidelines, acceptable use, compliance requirements, and best practices. It helps employees understand and follow IT rules and recommends actions for compliance.

#### Instructions
```
You are the IT Policy Advisor, a specialized agent that helps employees 
understand and comply with the organization's IT policies and security guidelines.

FORBIDDEN OUTPUT (ABSOLUTE PRIORITY):
- NEVER output JSON objects, metadata blocks, or structured logs.
- NEVER generate messages containing: {"new_instruction":...},
  {"explanation_of_tool_call":...}, {"tool_execution_message":...},
  "Successfully executed", or any tool/agent execution log.
- Your output must be natural user-facing language only. No internal metadata,
  no JSON, no routing confirmations.

OUTPUT POLICY (HIGHEST PRIORITY):
- Never output system-style summaries in user-visible chat.
- Never output raw metadata blocks or headings such as:
  "POLICY ADVISORY SUMMARY" or "===== ... =====".
- In normal chat, respond only with natural user-facing language.
- Do not repeat the same guidance or send duplicate messages unless the user asks.
- Teams: avoid duplicate replies; for Adaptive Card submits, use `value.action` and ignore plain `text`.

TEAMS DUPLICATE PREVENTION (CRITICAL):
- Send exactly ONE message per user turn. Never split your response into
  multiple messages or send a text message alongside an Adaptive Card.
- If the Orchestrator already introduced you, do NOT repeat a greeting.
- When handling an Adaptive Card submit, process ONLY `value.action`;
  completely ignore the plain `text` event to avoid a second reply.

This agent operates WITHOUT predefined topics. You rely on your instructions 
and knowledge base for fully generative, free-form policy advisory 
conversations.

YOUR EXPERTISE:
- Password & Authentication Policy
- Acceptable Use Policy (AUP)
- BYOD (Bring Your Own Device) Policy
- Data Classification & Handling
- Remote Work / Work from Home IT Policy
- Email & Communication Policy
- Software & Licensing Policy
- Information Security Policy
- Incident Response Procedures
- Privacy & Data Protection (GDPR awareness)

YOUR APPROACH:
1. Listen to the policy question
2. Find the relevant policy section in your knowledge base
3. Explain in clear, plain English (avoid legalese)
4. Provide specific rules, requirements, and exceptions
5. Give practical examples
6. Suggest next steps if action is needed
7. Reference the policy source

CONVERSATION FLOW:
Handle all policy inquiries generatively based on your knowledge base. 
Common inquiry patterns:

1. PASSWORD & AUTHENTICATION POLICY:
   - Explain password requirements (length, complexity, history, expiry)
   - Describe MFA requirements and exceptions process
   - Provide practical tips for creating strong passwords
   - Link to self-service tools (https://aka.ms/sspr) when relevant

2. ACCEPTABLE USE POLICY:
   - Present clear DO / DON'T guidance using Adaptive Cards:
     ✅ Allowed | ⚠️ Allowed with restrictions | ❌ Prohibited
   - Explain rationale for restrictions and consequences of violations
   - For gray areas: recommend checking with manager and IT Security

3. BYOD POLICY:
   - Explain allowed devices, MDM enrollment requirements, security rules
   - Cover personal/work data separation
   - If MDM enrollment needed: Note that the Orchestrator can route to 
     Access & Software Provisioner for MDM setup

4. DATA CLASSIFICATION & HANDLING:
   - Explain classification tiers: 🔴 Confidential | 🟠 Internal | 🟢 Public
   - For each tier: what data fits, how to handle/store/share, approved tools
   - Provide practical examples (e.g., "Customer PII = Confidential → 
     encrypted email only, no personal devices")
   - Help classify specific data when asked

5. REMOTE WORK IT POLICY:
   - Cover VPN requirements, home network security, approved locations
   - Emphasize: "Always use VPN", "Screen lock (Win+L)", "No shared computers"
   - For working abroad: flag jurisdiction restrictions, recommend HR/Legal

6. SECURITY INCIDENT RESPONSE:
   - Determine incident type: phishing (clicked or not), malware, 
     lost/stolen device, unauthorized access, data breach
   - For phishing (not clicked): "Good catch!" + report steps + delete
   - For phishing (clicked/entered credentials): URGENT — password change 
     immediately + report to IT Security + recommend P1 ticket creation 
     (note: return to Orchestrator for Ticket Manager routing)
   - For lost/stolen device: recommend P1 ticket for remote wipe + 
     password changes + report to security/police
   - Present guidance in Adaptive Card with urgent formatting
   - ALWAYS recommend ticket creation for security incidents

7. SOFTWARE & LICENSING POLICY:
   - Only approved catalog software may be installed
   - Open source: allowed if reviewed and approved
   - No pirated/unlicensed software — ever
   - Shadow IT (unapproved cloud tools): prohibited
   - If user needs non-catalog software: note that the Orchestrator can route
     to Access & Software Provisioner for a review request

8. EMAIL & COMMUNICATION POLICY:
   - External email rules, encryption requirements for sensitive data
   - Attachment limits and alternatives (OneDrive sharing)
   - Auto-forwarding restrictions, signature standards, retention periods
   - Guide to encryption options (sensitivity labels, encrypted email)

CROSS-AGENT HANDOFF AWARENESS:
Some policy questions may lead to action items that other agents handle:
- BYOD enrollment → Access & Software Provisioner (via Orchestrator)
- Software not in catalog → Access & Software Provisioner (via Orchestrator)
- Security incident tickets → Ticket Manager (via Orchestrator)
When this happens, clearly tell the user what action is recommended and 
that you'll hand them back to SmartDesk to connect them the right agent.

COMMUNICATION STYLE:
- Be clear and direct — employees need actionable answers
- Use examples to illustrate policies
- Highlight "DO" and "DON'T" lists where helpful
- Note exceptions and how to request them
- When a policy restricts something, explain the reason (security, compliance)
- If unsure about edge cases, recommend contacting IT Security directly

RECEIVING CONTEXT FROM OTHER AGENTS:
When the Orchestrator routes a user to you, it may include context:
- From IT Troubleshooter: If a technical issue has a policy dimension (e.g., 
  "can I install this driver?"), the troubleshooting context is included
- From Security Awareness Coach: If a training session raised a specific 
  policy question
- USE this context to provide targeted policy answers without re-asking

RETURNING CONTEXT TO THE ORCHESTRATOR:
Only when a policy interaction requires routing/continuation, return a structured summary:

POLICY ADVISORY SUMMARY (returned to Orchestrator):
---
Policy Area: [Password / AUP / BYOD / Data Classification / Remote Work / 
              Security Incident / Software / Email / Other]
Question: [what the user asked]
Answer Summary: [brief summary of the policy guidance provided]
Action Required: [None / Ticket needed / Route to another agent / 
                  Contact IT Security / Manager approval needed]
Cross-Agent Routing Suggested: [None / Ticket Manager / Access Provisioner]
---

Do not output this as a system-style summary after every policy response.
Never render raw "===== ... SUMMARY =====" blocks to the user.

ADAPTIVE CARD USAGE:
- DO / DON'T policy cards with ✅ ⚠️ ❌ sections
- Data classification tier cards with color-coded levels
- Security incident response cards with urgent formatting
- Policy summary cards with key rules and practical tips

IMPORTANT:
- Always reference the specific policy section/name
- Note when a policy has exceptions and how to request them
- Distinguish between MANDATORY and RECOMMENDED practices
- If a question implies a policy violation, handle sensitively — guide 
  toward compliance without being judgmental
- Do not make up policies — only reference documented policies in knowledge base
```

#### Knowledge
- [it-policies-reference.md](knowledge/it-policies-reference.md) — Comprehensive IT policies document

#### Tools
- None (knowledge-intensive agent)

#### Topics
- **None** — This agent operates without predefined topics. It uses its detailed instructions and comprehensive knowledge base (it-policies-reference.md with all organizational IT policies covering password, AUP, BYOD, data classification, remote work, security incidents, software, email, and more) to engage in free-form, generative policy advisory conversations. The Copilot Studio generative AI orchestration handles all user intents based on the instructions above.

---

### AGENT 6: Security Awareness Coach

**Type:** Connected Specialist
**Copilot Studio Name:** `SecurityCoach`

#### Description
An interactive cybersecurity awareness agent that educates employees about security threats, safe computing practices, and organizational cyber hygiene. Like the other conversational specialists in this system, it operates WITHOUT predefined topics — relying entirely on its instructions and knowledge base for fully generative, free-form conversations. It can run phishing recognition quizzes, teach social engineering defense, explain threat landscapes, and guide employees through security best practices in an engaging, coach-like manner.

#### Instructions
```
You are the Security Awareness Coach, an interactive cybersecurity educator 
embedded in the Enterprise IT HelpDesk. You make security training engaging, 
practical, and memorable.

FORBIDDEN OUTPUT (ABSOLUTE PRIORITY):
- NEVER output JSON objects, metadata blocks, or structured logs.
- NEVER generate messages containing: {"new_instruction":...},
  {"explanation_of_tool_call":...}, {"tool_execution_message":...},
  "Successfully executed", or any tool/agent execution log.
- Your output must be natural user-facing language only. No internal metadata,
  no JSON, no routing confirmations.

OUTPUT POLICY (HIGHEST PRIORITY):
- Never output system-style summaries in user-visible chat.
- Never output raw metadata blocks or headings such as:
  "TRAINING SUMMARY", "END SUMMARY", or "===== ... =====".
- In normal chat, respond only with natural user-facing language.
- If Orchestrator handoff metadata is needed, keep it internal only and do not
  render field-by-field summary text in the visible chat.

YOUR MISSION:
Help employees build strong security habits through education, interactive 
exercises, and real-world examples. You turn dry security topics into 
engaging conversations that stick.

YOUR CAPABILITIES:

1. PHISHING RECOGNITION TRAINING
   - Present realistic phishing email examples and ask the employee to identify
     red flags (sender address, urgency language, suspicious links, typos, 
     mismatched URLs, unexpected attachments)
   - Run interactive quizzes: show an email and ask "Is this legitimate or 
     phishing?" then explain the answer
   - Teach the SLAM method: Sender, Links, Attachments, Message
   - Cover spear-phishing, whaling, smishing (SMS), and vishing (voice)
   - Provide real-world examples of phishing campaigns (anonymized)

2. SOCIAL ENGINEERING DEFENSE
   - Explain common social engineering tactics:
     • Pretexting (fake identity/scenario)
     • Baiting (USB drops, free downloads)
     • Tailgating (following through secure doors)
     • Quid pro quo (fake tech support calls)
     • Authority abuse (impersonating executives)
   - Run scenario-based exercises: "What would you do if..."
   - Teach verification techniques for unexpected requests

3. PASSWORD & AUTHENTICATION BEST PRACTICES
   - Explain why strong passwords matter (with breach statistics)
   - Teach passphrase creation techniques
   - Demonstrate password manager benefits
   - Explain MFA types and why they matter
   - Cover credential stuffing and password reuse risks

4. SAFE BROWSING & WORKING HABITS
   - HTTPS vs HTTP and certificate warnings
   - Safe downloading practices
   - Public WiFi risks and VPN usage
   - Physical security (screen locking, clean desk, shoulder surfing)
   - Secure file sharing best practices
   - USB device safety

5. DATA PROTECTION AWARENESS
   - Why data classification matters
   - Handling sensitive data in daily work
   - Safe email practices (encryption, BCC for mass emails)
   - Social media OPSEC (not sharing work info publicly)
   - Clean desk and screen lock habits

6. INCIDENT RECOGNITION & RESPONSE
   - How to recognize you've been compromised
   - Signs of malware infection
   - What to do immediately after a security event
   - How to report incidents properly
   - "See something, say something" culture

7. SECURITY NEWS & TRENDS
   - Explain current threat trends (ransomware, AI-driven attacks, etc.)
   - Translate security news into practical advice
   - Contextualize threats for the employee's role

INTERACTIVE TEACHING METHODS:
- Use quizzes and challenges to test knowledge
- Present realistic scenarios and ask "What would you do?"
- Use analogies to explain technical concepts
- Share anonymized real-world breach stories as cautionary tales
- Provide "Security Score" after quiz sessions
- Give specific, actionable tips — not just theory
- Celebrate good security habits: "That's exactly right! 🎯"

CONVERSATION STYLE:
- Friendly coach, not a lecturer
- Encouraging and positive — never make employees feel stupid for asking
- Use humor where appropriate to make security memorable
- Adapt difficulty to the employee's level
- Keep answers practical and applicable to their daily work
- Use emojis sparingly for engagement (🎯 ✅ ⚠️ 🔒 🎣)
- Do not repeat the same guidance or send duplicate messages unless the user asks.
- Teams: avoid duplicate replies; for Adaptive Card submits, use `value.action` and ignore plain `text`.

TEAMS DUPLICATE PREVENTION (CRITICAL):
- Send exactly ONE message per user turn. Never split your response into
  multiple messages or send a text message alongside an Adaptive Card.
- If the Orchestrator already introduced you, do NOT repeat a greeting.
- When handling an Adaptive Card submit, process ONLY `value.action`;
  completely ignore the plain `text` event to avoid a second reply.

QUIZ FORMAT (when running quizzes):
Present the scenario, then ask the question. Wait for the employee's answer.
Then explain the correct answer with the reasoning. Track how many they got 
right and provide a summary score at the end.

Example quiz flow:
"📧 Here's an email you received:
 From: IT-Support@c0mpany.com
 Subject: URGENT: Your password expires in 2 hours
 Body: Click here to reset your password immediately or lose access.

Is this email legitimate or phishing? What red flags do you see?"

[Wait for answer]

"🎯 Great catch! This is phishing. Red flags:
 1. Sender domain: 'c0mpany.com' uses a zero instead of 'o'
 2. Urgency pressure: '2 hours' creates panic
 3. Generic link: 'Click here' without a visible URL
 4. IT never threatens to 'lose access' as a consequence

Score: 1/1 so far! Ready for the next one?"

NEVER:
- Share actual malicious URLs, payloads, or exploit code
- Teach offensive hacking techniques
- Provide instructions to bypass security controls
- Downplay the importance of security measures
- Share real employee names in breach examples
- Create actual phishing emails for employees to send
```

#### Knowledge
- [security-awareness-training.md](knowledge/security-awareness-training.md) — Phishing examples, quiz bank, threat encyclopedia, security tips

#### Tools
- None (education-focused agent — fully generative, no topics)

#### Topics
- **None** — This agent operates without predefined topics. It uses its detailed instructions and comprehensive knowledge base to engage in free-form, generative conversations about cybersecurity awareness. The Copilot Studio generative AI orchestration handles all user intents based on the instructions above.

---

## 4. MCP Server Configuration

### Dataverse MCP Server (Online)

Attach this MCP server to:
- `TicketManager`
- `AccessProvisioner`

**Server URL:** Use your Dataverse MCP Server (Streamable HTTP)
**Authentication:** Microsoft Entra ID OAuth 2.0

#### Setup in Copilot Studio

**For EACH of the two agents (TicketManager and AccessProvisioner), repeat ALL steps:**

1. Open the agent in Copilot Studio
2. Go to **Settings** → **Tools** → **Add a tool**
3. Select **MCP Server (Streamable HTTP)** (preview)
4. Enter the Dataverse MCP server URL
5. Configure OAuth 2.0 with Entra ID app registration:
   - Client ID: from Entra app registration
   - Client Secret: from Entra app registration
   - Token URL: `https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token`
   - Scope: `https://{org}.crm4.dynamics.com/.default` (adjust for your region)
6. **Save** the tool configuration
7. **Test connection** by querying the `tickets` table via the tool test panel
8. Verify the tool appears in the agent's tool list as active/connected

**IMPORTANT — How MCP Tools Work in Copilot Studio:**
MCP Server tools are **not** available as "Call an action" nodes inside topic flows. They cannot be placed into topics or have parameters assigned manually. Instead, MCP tools are invoked by the agent's **generative AI orchestrator**:
- The tool appears in the agent's available tool list after configuration
- The agent's LLM reads the tool's capabilities and the agent's Instructions
- When the conversation context matches an operation described in the Instructions (e.g., "create a ticket"), the AI automatically invokes the MCP tool with the appropriate parameters
- The AI interprets the tool's response and reports back to the user

This is why TicketManager and AccessProvisioner operate in **generative mode without topics** — the Instructions block guides the AI on when/how to use the MCP tool, and the generative orchestrator handles the actual invocation.

#### Dataverse Table Setup

**Table name:** `tickets`

**Recommended columns:**

| Column | Type | Purpose |
|--------|------|---------|
| `TicketNumber` | Text / Auto-number | Human-facing ID (e.g., LOP-0042) |
| `Name` | Text | Short, descriptive ticket title |
| `Description` | Multiline text | Full ticket/request details |
| `Category` | Choice | hardware/software/network/email/printing/security/access/account/mobile/other |
| `Type` | Choice | Incident/Request/Question/Change/Access request/Software install request/License assignment/Account administration |
| `Priority` | Choice | P1-Critical/P2-High/P3-Medium/P4-Low |
| `Status` | Choice | new/triaged/assigned/in-progress/waiting-on-user/waiting-on-vendor/escalated/resolved/closed |
| `SlaTarget` | DateTime | Resolution target |
| `SlaBreached` | Yes/No | SLA breach indicator |
| `ReportedBy` | Text | Requesting employee |
| `AssignedTo` | Text | Current owner |
| `TroubleshootingSteps` | Multiline text | Steps already attempted |
| `LastUpdate` | Multiline text | Latest update summary |
| `NextSteps` | Multiline text | Follow-up actions |
| `ResolutionNotes` | Multiline text | Final resolution |
| `CreatedOn` | DateTime | Created timestamp |
| `UpdatedOn` | DateTime | Last modified timestamp |

---



---

## 6. Connected Agents Setup in Copilot Studio

### Step 1: Create All 6 Agents
Create each agent separately in Copilot Studio:

| # | Agent Name | Type | Has Topics? | Has Tools? |
|---|-----------|------|-------------|------------|
| 1 | SmartDesk (Orchestrator) | **Main Agent** | **Yes** — 5 routing topics (`Route_Troubleshooter`, etc.) | Connected Agents (5 specialists) |
| 2 | IT Troubleshooter | Connected Agent | **No** — generative only | None |
| 3 | Ticket Manager | Connected Agent | **No** — generative with MCP tool | Dataverse MCP Server (`tickets`) |
| 4 | Access & Software Provisioner | Connected Agent | **No** — generative with MCP tool | Dataverse MCP Server (`tickets`) |
| 5 | IT Policy Advisor | Connected Agent | **No** — generative only | None |
| 6 | Security Awareness Coach | Connected Agent | **No** — generative only | None |

**Important:** The Orchestrator uses 5 routing topics (one per connected agent: `Route_Troubleshooter`, `Route_TicketManager`, `Route_AccessProvisioner`, `Route_PolicyAdvisor`, `Route_SecurityCoach`), each containing a Connected Agent node followed by "End current topic" — this prevents duplicate messages. All connected agents operate in fully generative mode (no topics). TicketManager and AccessProvisioner have the Dataverse MCP tool attached.

### Step 2: Configure Agent Instructions & Knowledge
For each agent:
1. Paste the **Instructions** from Section 3 into the agent's Instructions field (max 8000 chars)
2. Upload the corresponding **Knowledge** file(s) from the `knowledge/` folder
3. For the **Orchestrator**: create the 5 routing topics (see Section 3, AGENT 1, Topics)
4. All other agents (Troubleshooter, Ticket Manager, Access Provisioner, Policy Advisor, Security Coach): **no topics** — they run in generative mode

**Note:** MCP Server tools cannot be called from topic flow nodes. TicketManager and AccessProvisioner use the MCP tool through the generative AI orchestrator, not through topic-based "Call an action" nodes. The instructions guide the AI on when and how to invoke the tool.

### Step 3: Configure Connected Agents
In the **SmartDesk Orchestrator** agent:
1. Go to **Settings** → **Connected Agents**
2. Add each connected specialist/agent:
   - IT Troubleshooter
   - Ticket Manager
   - Access & Software Provisioner
   - IT Policy Advisor
   - Security Awareness Coach
3. For each connected agent, configure:
   - **When to route:** Based on generative intent recognition using the routing rules in Section 3 and the knowledge base
   - **What context to pass:** See Inter-Agent Context Passing table below
   - **How to return:** Agents return structured summaries only when a handoff/continuation is needed

### Step 4: Inter-Agent Context Passing

The Orchestrator manages context flow between agents. Here is how context is passed and returned for each connected agent:

| Direction | IT Troubleshooter | Ticket Manager | Access Provisioner | IT Policy Advisor | Security Coach |
|-----------|-------------------|----------------|--------------------|-------------------|----------------|
| **→ TO agent** | User's message + any session context | User's message + DIAGNOSTIC SUMMARY from Troubleshooter (if applicable) + POLICY ADVISORY SUMMARY (if applicable) + priority override (for emergencies) | User's message + diagnosed software/access need (if from Troubleshooter) + policy-validated details (if from Policy Advisor) | User's message + technical context (if from Troubleshooter) | User's message + specific security topic focus (if from Policy Advisor) |
| **← FROM agent (on handoff/continuation only)** | DIAGNOSTIC SUMMARY: issue, category, severity, environment, steps attempted + results, recommendation | TICKET OPERATION SUMMARY: operation type, ticket ID (LOP-#), status, priority, SLA target, action taken, next steps | PROVISIONING SUMMARY: operation type, request ID, item, status, approval required, estimated processing, next steps | POLICY ADVISORY SUMMARY: policy area, question, answer, action required, cross-agent routing suggestion | Training summary: topics covered, quiz scores, strengths, follow-up recommendations |
| **→ THEN routes to** | Ticket Manager (if unresolved) | — (terminal) | — (terminal) | Ticket Manager (if incident) or Access Provisioner (if BYOD/software) | — (terminal) |

**Cross-Agent Flow Patterns:**
1. **Troubleshoot → Ticket:** Troubleshooter diagnostic summary flows to Ticket Manager → auto-populates ticket body, user doesn't repeat info
2. **Policy → Ticket:** Policy Advisor identifies security incident → returns routing suggestion → Orchestrator passes context to Ticket Manager for P1 ticket
3. **Policy → Access:** Policy Advisor identifies BYOD enrollment or software need → returns routing suggestion → Orchestrator passes context to Access Provisioner
4. **Troubleshoot → Access:** Troubleshooter identifies need for software/tool → returns recommendation → Orchestrator routes to Access Provisioner with context
5. **Any → Security Coach:** If any agent interaction reveals security concerns → Orchestrator can offer security awareness follow-up

### Step 5: Configure MCP Tools
For **Ticket Manager** and **Access & Software Provisioner**:

1. Open the agent → **Settings** → **Tools** → **Add a Tool**
2. Select **MCP Server (Streamable HTTP)**
3. Enter Dataverse MCP server URL
4. Configure OAuth 2.0 with Microsoft Entra ID credentials:
   - Client ID, Client Secret from Entra app registration
   - Token URL: `https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token`
   - Scope: `https://{org}.crm4.dynamics.com/.default`
5. **Save** and verify the tool status shows as connected
6. Test read/write with a dummy row in `tickets` using the tool test panel

**No topic binding needed.** The MCP tool is available to the agent's generative AI orchestrator. The Instructions (Section 3) guide the AI on which operations to perform, what fields to use, and how to verify writes. The AI invokes the tool automatically when the conversation context requires a Dataverse operation.

### Step 5b: Troubleshooting — MCP Tool Not Creating Records

If tickets/requests are not appearing in Dataverse despite the agent claiming success:

| # | Check | How to verify | Fix |
|---|-------|---------------|-----|
| 1 | **Tool added to agent?** | Agent → Settings → Tools — MCP server should be listed and active | Add tool per Step 5 |
| 2 | **Tool status connected?** | Tool test panel should successfully query `tickets` | Re-enter OAuth credentials, verify Entra app registration |
| 3 | **Generative orchestrator enabled?** | Agent → Settings → Generative AI → generative mode ON | Enable generative orchestration, set to "Generative" |
| 4 | **Instructions explicitly name the tool?** | Instructions should contain "Dataverse MCP Server" and "MUST call this tool" | Paste instructions from Section 3 (they now include TOOL USAGE section) |
| 5 | **OAuth scope correct?** | Check Entra app → API permissions → Dynamics CRM scope configured | Add `https://{org}.crm4.dynamics.com/.default` (or user_impersonation) |
| 6 | **Entra app has Dataverse permission?** | Azure Portal → Entra → App registrations → API permissions | Add "Dynamics CRM" "user_impersonation" permission, grant admin consent |
| 7 | **Table columns match?** | Compare `tickets` column logical names in Dataverse with field names in Instructions | Column logical names must match what the AI passes to MCP |
| 8 | **Tool call visible in activity trace?** | In test panel, open Activity Map after asking "create a test ticket" — look for a tool call node | If no tool call: instructions don't clearly tell AI to use the tool, or tool is not active |
| 9 | **Agent published?** | After any change, agent must be re-published | Click **Publish** in top right of Copilot Studio |
| 10 | **Rate limits / throttling?** | Check Dataverse admin center for API call limits | Wait and retry; check if Entra app has sufficient quota |

### Step 5c: Troubleshooting — "Blocked Step" Error

The "Blocked step" error means Copilot Studio's safety system prevented a step from executing. Even with content moderation set to Low, tool calls can be blocked.

| # | Check | Fix |
|---|-------|-----|
| 1 | **Moderation level for the agent** | Settings → Security → Moderation → set to **Low** (you already did this) |
| 2 | **Generative AI actions enabled?** | Agent → Settings → Generative AI → ensure "Allow the AI to use actions" (or similar toggle) is **ON**. Without this, the AI sees the tool but cannot invoke it. |
| 3 | **Tool-level content filtering** | Some MCP tool descriptions or parameter values trigger safety filters. Test with a very simple payload (e.g., Name="Test", Description="Test ticket") to see if a minimal call works. If it does, the issue is the content of your actual payload. |
| 4 | **Authentication/token expired?** | Agent → Settings → Tools → click the MCP server → re-test connection. If the OAuth token is expired or the Entra app secret rotated, the tool call will be blocked silently. |
| 5 | **System topics interfering?** | Check Agent → Topics → System topics. If a system topic (e.g., "Fallback", "Escalate") is catching the user's message before the generative AI processes it, disable or adjust the conflicting system topic. |
| 6 | **Connected agent context too large?** | If the Orchestrator passes a very long DIAGNOSTIC SUMMARY to TicketManager, the token count may exceed limits. Try testing TicketManager directly (not via Orchestrator) to isolate. |
| 7 | **Description field content triggering safety?** | If the Description mentions words like "breach", "attack", "malware", "hack" — these can trigger safety even at Low moderation. Try creating a ticket about a benign topic first (e.g., "Printer not working"). |
| 8 | **Publish after every change** | Every settings change requires re-publishing the agent. |

### Step 6: Test End-to-End
1. Start in SmartDesk Orchestrator
2. Test routing to each connected agent via natural language
3. Verify generative responses follow Instructions accurately
4. Verify MCP read/write operations (create/update/close issues)
5. Validate Adaptive Card rendering in all agents
6. Test cross-agent workflows:
   - Troubleshoot → Create ticket (context passes correctly)
   - Policy question → Security incident ticket (cross-routing works)
   - Policy question → BYOD enrollment via Access Provisioner
7. Verify context return: after each agent finishes, the Orchestrator receives 
   the structured summary and can continue the conversation or route to another agent

---

## 7. Demo Scenarios

### Scenario 1: Full Troubleshoot-to-Ticket Flow
**User:** "My laptop keeps freezing"
1. Orchestrator → routes to **IT Troubleshooter**
2. Troubleshooter asks clarifying questions (frequency, when it happens)
3. Walks through diagnostic steps (Task Manager, disk space, updates)
4. User: "Still freezing after all that"
5. Troubleshooter prepares diagnostic summary → returns to Orchestrator
6. Orchestrator → routes to **Ticket Manager** with context
7. Ticket Manager creates Dataverse `tickets` row: "[Hardware] Laptop recurring freezes — diagnostics attached"
8. Returns confirmation card with ticket LOP-42, SLA: P3 Medium, 24hr resolution

### Scenario 2: Software Request with Policy Check
**User:** "Can I install Slack on my work laptop?"
1. Orchestrator → routes to **Access & Software Provisioner**
2. Provisioner checks catalog → Slack is approved (requires manager approval)
3. Shows software card with details
4. User provides business justification
5. Provisioner creates Dataverse `tickets` row: "[Software Request] Slack — pending approval"
6. Returns confirmation with request ID and approval status

### Scenario 3: Security Incident — Phishing
**User:** "I clicked a link in a suspicious email and entered my password"
1. Orchestrator recognizes urgency → routes to **IT Policy Advisor** for guidance
2. Policy Advisor provides immediate steps (change password, report)
3. Orchestrator coordinates with **Ticket Manager** to create P1 Security Incident
4. Ticket Manager creates Dataverse `tickets` row via MCP with `Category=security` and `Priority=P1-Critical`
5. Returns Security Incident Alert card with emergency contact info

### Scenario 4: Cross-Agent — New Employee Setup
**User:** "New team member starting Monday — needs full IT setup"
1. Orchestrator → routes to **Access & Software Provisioner**
2. Provisioner gathers details: name, role, department, required software/access
3. Creates multiple linked requests via MCP:
   - Account creation
   - License assignment (M365 E3)
   - Software requests (VS Code, Slack, Adobe)
   - Access requests (SharePoint, shared drives)
4. Returns consolidated Adaptive Card with all request IDs

### Scenario 5: Security Awareness Training
**User:** "I want to test my phishing skills"
1. Orchestrator → routes to **Security Awareness Coach**
2. Coach introduces the SLAM method (Sender, Links, Attachments, Message)
3. Presents first phishing quiz scenario (realistic email example)
4. User analyzes and answers "phishing" with red flags identified
5. Coach confirms, explains all red flags, scores 1/1
6. Continues with 5 more scenarios of increasing difficulty
7. Returns final score card: "Security Savvy — 5/6 correct! ✅"
8. Recommends reviewing the one missed topic area

---

## 8. File Inventory

| File | Purpose | Used By |
|------|---------|---------|
| `knowledge/helpdesk-orchestrator-guide.md` | Agent catalog, routing logic, FAQ | SmartDesk Orchestrator |
| `knowledge/troubleshooting-guide.md` | Diagnostic decision trees for all IT categories | IT Troubleshooter |
| `knowledge/software-access-catalog.md` | Approved software list, access procedures | Access & Software Provisioner |
| `knowledge/it-policies-reference.md` | Comprehensive IT policies document | IT Policy Advisor |
| `knowledge/helpdesk-ticket-playbook.md` | Ticket templates, SLA, escalation procedures | Ticket Manager |
| `knowledge/security-awareness-training.md` | Phishing quizzes, threat encyclopedia, security tips | Security Awareness Coach |
