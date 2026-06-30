# Chat: Create High-Level Architecture Diagram on Miro

## User

create a high level diagramm for caldiy-masterclass application using /miro

## Assistant

I'll explore the codebase structure and then create a high-level architecture diagram on Miro. Let me start by understanding the project layout and checking available Miro tools.

### Tools Used

1. **Task (explore subagent)** — Explored Cal.diy codebase structure to understand high-level architecture
2. **Shell** — Listed available Miro MCP tools (`ls mcps/user-miro-mcp/tools/`)
3. **Read** — Read `diagram_create.json` and `diagram_get_dsl.json` tool schemas
4. **Read** — Read `board_create.json` tool schema
5. **CallMcpTool (`board_create`)** — Attempted to create a new board → failed (plan limitation)
6. **CallMcpTool (`board_search_boards`)** — Searched for existing boards → found 3 boards
7. **CallMcpTool (`diagram_get_dsl`)** — Retrieved flowchart DSL specification for the "Ideas" board
8. **CallMcpTool (`diagram_create`)** — Created the architecture diagram on the "Ideas" board

### Key Decisions

- Could not create a new board due to Miro plan limitations
- Used existing "Ideas" board (`https://miro.com/app/board/uXjVPgpcSOM=/`)
- Chose `flowchart` diagram type to show layered architecture with data flow
- Used clusters to group components by architectural layer
- Color-coded by layer (cyan=clients, blue=apps, purple=API, green=domain, yellow=supporting, red=data stores, pink=integrations)

### Result

The diagram was created successfully:

**[Cal.diy - High Level Architecture](https://miro.com/app/board/uXjVPgpcSOM=/?moveToWidget=3458764676324555621)**

The diagram shows the complete application architecture organized into these layers:

| Layer | Components |
|-------|-----------|
| **Clients** | Web Browser, Embed/Atoms SDK, External API Clients |
| **Applications** | Next.js Web App (apps/web), NestJS Platform API v2 (apps/api/v2) |
| **API Layer** | tRPC Routers, REST Routes, Platform REST Controllers, NextAuth.js |
| **Domain Layer** | Feature Services (packages/features), Shared Utilities (packages/lib) |
| **Data Layer** | Prisma ORM, PostgreSQL, Redis/Upstash |
| **Integrations** | App Store with 150+ plugins |
| **External Services** | Google Calendar, Zoom/Daily.co, Stripe |
| **Supporting** | UI Components, i18n Translations, Email/SMS, Background Jobs (Trigger.dev) |

The connectors show how data flows through the system - from clients through the application layer, into the API routing layer, delegating to domain services which then interact with databases, integrations, and external services.
