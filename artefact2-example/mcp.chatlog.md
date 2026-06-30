create a high level diagramm for 
caldiy-masterclass
 application using /miro 

I'll explore the codebase structure and then create a high-level architecture diagram on Miro. Let me start by understanding the project layout and checking available Miro tools.

Composer 2.5 Fast

Now let me check the diagram_create tool schema to understand the parameters.

I need to first get the DSL specification for a flowchart, then create the diagram. Since the tool requires a miro_url for diagram_get_dsl, I'll need to create a new board or use an existing one. Let me first get the DSL spec and create the diagram. I'll create a new board since no URL was provided.

Let me also check the board_create tool to see if I should create a board first.

I'll create a new Miro board for this architecture diagram, then get the DSL specification and create the diagram. Let me proceed with creating the board first.

The plan doesn't allow creating new boards. Let me search for an existing board I can use.

I have 3 existing boards available but can't create a new one due to plan limitations. I'll use the "Ideas" board since it seems like a general-purpose board. Let me get the flowchart DSL specification and then create the architecture diagram there.

Now I have the DSL specification. Let me create a comprehensive high-level architecture diagram for Cal.diy.

The diagram has been created successfully. Here's what I built on your Miro "Ideas" board:

Cal.diy - High Level Architecture

The diagram shows the complete application architecture organized into these layers:

Layer	Components
Clients
Web Browser, Embed/Atoms SDK, External API Clients
Applications
Next.js Web App (apps/web), NestJS Platform API v2 (apps/api/v2)
API Layer
tRPC Routers, REST Routes, Platform REST Controllers, NextAuth.js
Domain Layer
Feature Services (packages/features), Shared Utilities (packages/lib)
Data Layer
Prisma ORM, PostgreSQL, Redis/Upstash
Integrations
App Store with 150+ plugins
External Services
Google Calendar, Zoom/Daily.co, Stripe
Supporting
UI Components, i18n Translations, Email/SMS, Background Jobs (Trigger.dev)
The connectors show how data flows through the system - from clients through the application layer, into the API routing layer, delegating to domain services which then interact with databases, integrations, and external services.