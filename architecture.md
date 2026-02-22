# SmartDesk Architecture (ASCII)

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

Legend:
- Orchestrator routes to connected agents based on intent.
- TicketManager and AccessProvisioner call MCP (describe_table -> map -> create).
- Other agents are generative, no tools.
```
