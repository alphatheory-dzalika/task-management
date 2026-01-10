❯ you explain what is the concept of Claude Skills  Skills
Repeatable, customizable instructions that Claude can follow in any chat. Learn more.

Search
Your skills
Organization skills
Example skills     and how they might relate to a huge repository such as the IDM system?   Consider
that we have the current MCPs in this and that we have more to go

● Claude Skills Explained

Claude Skills are reusable, customizable instruction sets that Claude can follow across conversations. Think of them as "saved prompts with context" that can be invoked on demand.

Core Concept

Skills are essentially:
1. Predefined workflows - Step-by-step instructions Claude follows
2. Domain-specific knowledge - Context about your codebase, conventions, patterns
3. Repeatable actions - Commands like /commit, /review-pr, /test that trigger consistent behavior

How Skills Differ from MCPs
┌─────────┬───────────────────────────────────┬─────────────────────────────────┐
│ Aspect  │            MCP Servers            │          Claude Skills          │
├─────────┼───────────────────────────────────┼─────────────────────────────────┤
│ Purpose │ Provide data access and tools     │ Provide behavioral instructions │
├─────────┼───────────────────────────────────┼─────────────────────────────────┤
│ Runtime │ External processes with APIs      │ Prompt-based, runs in Claude    │
├─────────┼───────────────────────────────────┼─────────────────────────────────┤
│ State   │ Can query live databases          │ Stateless instructions          │
├─────────┼───────────────────────────────────┼─────────────────────────────────┤
│ Example │ idm-knowledge-mcp querying SQLite │ /commit skill for git workflow  │
└─────────┴───────────────────────────────────┴─────────────────────────────────┘
How Skills Could Complement IDM's MCPs

Your IDM system has MCP servers providing knowledge access:
- idm-knowledge-mcp - Feature cards, recipes, code patterns
- factset-estimates-mcp - Domain-specific data

Skills could layer on top for workflow automation:

/new-controller     → Follows Job Controller Template Pattern from CLAUDE.md
/add-rowmapper      → Creates RowMapper following JDBC standards
/cx-feature         → Prioritizes CX servicing patterns
/document-feature   → Generates feature card + recipe using MCP data

Potential IDM Skills

Given your codebase:

1. /idm-controller - Generates 8-endpoint job controller from template
2. /idm-entity - Creates model + rowmapper + service following layer rules
3. /idm-feature-card - Queries MCP, generates documentation
4. /cx-workflow - Follows CX-first production priority patterns

Skills would encode the knowledge in .claude/*.md files as executable workflows, while MCPs provide the queryable knowledge base.

Want me to explore how to create a specific skill for your IDM patterns?

✻ Churned for 1m 7s

❯ yes.  The problem is tht IDM has grown too large, so skills specific to this code base will help

● Let me explore the IDM codebase structure to identify the best skill candidates.

● Explore(Explore IDM for skill candidates)
⎿  Done (25 tool uses · 89.1k tokens · 3m 16s)

● Excellent exploration. Based on this analysis, here's a Claude Skills Architecture for IDM:

Proposed IDM Skills Suite

Tier 1: High-Impact Workflow Skills
┌─────────────────┬──────────────────────────────┬────────────────────────────────────┐
│      Skill      │           Trigger            │              Purpose               │
├─────────────────┼──────────────────────────────┼────────────────────────────────────┤
│ /idm-controller │ Creating new job controllers │ 8-endpoint pattern + lock strategy │
├─────────────────┼──────────────────────────────┼────────────────────────────────────┤
│ /idm-entity     │ Adding new data entities     │ Model → RowMapper → Service chain  │
├─────────────────┼──────────────────────────────┼────────────────────────────────────┤
│ /cx-feature     │ CX servicing work            │ Production-priority patterns       │
├─────────────────┼──────────────────────────────┼────────────────────────────────────┤
│ /idm-session    │ Session documentation        │ Capture decisions, anti-patterns   │
└─────────────────┴──────────────────────────────┴────────────────────────────────────┘
Tier 2: Domain-Specific Skills
┌───────────────┬────────────────────────┬───────────────────────────────────────┐
│     Skill     │        Trigger         │                Purpose                │
├───────────────┼────────────────────────┼───────────────────────────────────────┤
│ /natural-keys │ Entity resolution work │ Cache-first, bidirectional resolution │
├───────────────┼────────────────────────┼───────────────────────────────────────┤
│ /aggregation  │ Fund/asset aggregation │ Chunking, property transfer           │
├───────────────┼────────────────────────┼───────────────────────────────────────┤
│ /portfolio-qc │ Portfolio mapping QC   │ Stored procedure orchestration        │
└───────────────┴────────────────────────┴───────────────────────────────────────┘
Tier 3: Knowledge Access Skills
┌──────────────┬──────────────────────────┬────────────────────────┐
│    Skill     │         Trigger          │        Purpose         │
├──────────────┼──────────────────────────┼────────────────────────┤
│ /idm-pattern │ "How do we do X in IDM?" │ Query MCP for patterns │
├──────────────┼──────────────────────────┼────────────────────────┤
│ /idm-recipe  │ Implementation guidance  │ Step-by-step recipes   │
└──────────────┴──────────────────────────┴────────────────────────┘
  ---
Skill Implementation Example: /idm-controller

Let me create a concrete skill definition:

● Write(.claude/skills/idm-controller.md)
⎿  User rejected write to .claude/skills/idm-controller.md
# IDM Job Controller Skill

     ## Trigger
     Use this skill when creating a new job controller in IDM.

     ## Prerequisites
     Before starting, gather:
     1. **Domain name** (e.g., `factset`, `ticker`, `portfolio`)
     2. **Feature name** (e.g., `pricing`, `estimates`, `mapping-qc`)
     3. **Lock strategy requirements** (singleton vs concurrent)
     … +113 lines

❯ Pause.   Please look at the project "tracking review" ... IdeaProjects\tracking-review ...   I've
started the concepts of products and themes, and in themes I have mcps, which is meant to be a more
generalized understanding of what these are ..    the larger picture of this is documented in #
Documentation Funnel & Integration System   as well as in the project
IdeaProjects\alpha-theory-internal-mcp ...   Let's start building out the "products" and "themes" by
understanding what is present in IDM, OpenAdapter, RAPI, and so on.  And to find the best of what CAN
be implemented.   We want this to be our planning project, and that each participant will contribute
to some common, curated superset, with the process being transparent, and for this to be an inviting
approach to cross-team communication ...  and we would use minIO, S3, SQLite, and MCPs for a very
rich experience.  But it starts with this foundation.

● Let me explore the tracking-review project to understand the current foundation and vision.

● 2 Explore agents finished (ctrl+o to expand)
├─ Explore tracking-review project · 38 tool uses · 50.4k tokens
│  ⎿  Done
└─ Explore alpha-theory-internal-mcp · 30 tool uses · 85.6k tokens
⎿  Done

● This is excellent context. Now I understand the vision:

The Big Picture

┌─────────────────────────────────────────────────────────────────────────────┐
│                     TRACKING-REVIEW (Planning Hub)                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  PRODUCTS   │  │   THEMES    │  │   WEEKLY    │  │  DOC FUNNEL │        │
│  │  (What)     │  │   (How)     │  │  (Progress) │  │  (Output)   │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
└─────────┼────────────────┼────────────────┼────────────────┼────────────────┘
│                │                │                │
▼                ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│              ALPHA-THEORY-INTERNAL-MCP (Knowledge Layer)                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                         │
│  │  Knowledge  │  │   Gateway   │  │  Collector  │                         │
│  │  (Terms,    │  │  (Routing,  │  │  (Storage,  │                         │
│  │   Clients)  │  │   Intent)   │  │   Memory)   │                         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                         │
└─────────┼────────────────┼────────────────┼─────────────────────────────────┘
│                │                │
▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SOURCE PROJECTS (Implementation)                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │     DME     │  │ OpenAdapter │  │    RAPI     │  │   esapi-jpa │        │
│  │  (IDM MCP)  │  │  (Adapters) │  │  (Research) │  │   (Legacy)  │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘

Let me explore what exists to populate Products & Themes

● Explore(Explore OpenAdapter project)
⎿  Done (22 tool uses · 77.6k tokens · 37m 36s)

● Bash(ls -la /mnt/c/Users/dzalika/IdeaProjects/ | head -40)
⎿  total 240
drwxrwxrwx 1 root root    512 Jan  8 21:28 .
drwxrwxrwx 1 root root    512 Jan  8 21:55 ..
… +34 lines (ctrl+o to expand)

● Excellent. Now I have a complete picture across the ecosystem. Let me synthesize this into a concrete Products & Themes Catalog for tracking-review.

Ecosystem Summary
Project: dme (IDM)
Purpose: Integration Data Management
Key Capabilities: Job orchestration, Natural Keys, CX servicing, MCPs
────────────────────────────────────────
Project: openAdapter
Purpose: Data ETL & Transformation
Key Capabilities: Ticker resolution, FIGI, Instruction Containers
────────────────────────────────────────
Project: internal-alpha-theory-mcp
Purpose: Knowledge Layer
Key Capabilities: Gateway routing, Collector storage, Client knowledge
────────────────────────────────────────
Project: pyRAPI
Purpose: Research API (Python)
Key Capabilities: Portfolio analytics, calculations
────────────────────────────────────────
Project: securitymaster
Purpose: Security reference data
Key Capabilities: Master data management
────────────────────────────────────────
Project: dptv2
Purpose: Department tooling
Key Capabilities: Department-specific workflows
  ---
Proposed Products & Themes Architecture

Let me create the foundation document:

● Write(/mnt/c/Users/dzalika/IdeaProjects/tasking-review/src/main/PRODUCTS_AND_THEMES_CATALOG.md)
⎿ Tool use rejected with user message: Can you first review the DPTv2 .MD files for product and features
as well?   And this Product and Themes Catalog.md   ...  let's call it what it is ... product and themes
catalog discovery.md

● Explore(Explore DPTv2 for products/features)
⎿  Done (15 tool uses · 93.9k tokens · 1m 6s)

● Now I have DPTv2 context. This reveals significant additional products and themes. Let me update the todo and create the discovery catalog:

● Write(/mnt/c/Users/dzalika/IdeaProjects/tasking-review/src/main/PRODUCT_AND_THEMES_CATALOG_DISCOVERY.md)
⎿  Tool use rejected with user message: PLEASE DO NOT LOSE THIS WORK.  Please write this as a DRAFT.MD file.  Then please check in for next steps.

● Write(/mnt/c/Users/dzalika/IdeaProjects/tasking-review/src/main/DRAFT_PRODUCT_AND_THEMES_CATALOG_DISCOVERY.md)
⎿  Wrote 643 lines to ../tasking-review/src/main/DRAFT_PRODUCT_AND_THEMES_CATALOG_DISCOVERY.md                            
# Product and Themes Catalog Discovery
## Purpose
This discovery document captures capabilities found across Alpha Theory's major codebases to inform the products and themes architecture in tracking-review. It serves as the
foundation for:
- Cross-team knowledge sharing
- MCP integration planning
- Documentation funnel design
---
… +633 lines (ctrl+o to expand)

● Draft saved. Here's what we discovered and captured:

Summary

Saved to: tasking-review/src/main/DRAFT_PRODUCT_AND_THEMES_CATALOG_DISCOVERY.md

Products Found: 12
┌────────────────────────────┬──────────────────────────────────────────────────────────────────────────────┐
│          Category          │                                   Products                                   │
├────────────────────────────┼──────────────────────────────────────────────────────────────────────────────┤
│ Portfolio Platform (DPTv2) │ P1-Calculations, P2-Estimates, P3-Market Data, P4-Custom Logic               │
├────────────────────────────┼──────────────────────────────────────────────────────────────────────────────┤
│ Data Integration (IDM)     │ P5-Ticker Resolution, P6-Instructions, P7-External App, P8-Portfolio Mapping │
├────────────────────────────┼──────────────────────────────────────────────────────────────────────────────┤
│ Research & Analytics       │ P9-RAPI, P10-Fund Aggregation                                                │
├────────────────────────────┼──────────────────────────────────────────────────────────────────────────────┤
│ Client & Knowledge         │ P11-CX/Client, P12-Historical Data                                           │
└────────────────────────────┴──────────────────────────────────────────────────────────────────────────────┘
Themes Found: 10
┌───────────────┬─────────────────────────┐
│     Theme     │  Cross-cutting pattern  │
├───────────────┼─────────────────────────┤
│ T1-MCPs       │ AI tool integration     │
├───────────────┼─────────────────────────┤
│ T2-Data Flow  │ Movement patterns       │
├───────────────┼─────────────────────────┤
│ T3-Jobs       │ Orchestration           │
├───────────────┼─────────────────────────┤
│ T4-Identity   │ Natural/surrogate keys  │
├───────────────┼─────────────────────────┤
│ T5-Config     │ Declarative rules       │
├───────────────┼─────────────────────────┤
│ T6-Calcs      │ Financial pipelines     │
├───────────────┼─────────────────────────┤
│ T7-Vendors    │ API integration         │
├───────────────┼─────────────────────────┤
│ T8-Storage    │ Persistence patterns    │
├───────────────┼─────────────────────────┤
│ T9-Docs       │ Documentation funnel    │
├───────────────┼─────────────────────────┤
│ T10-Scenarios │ Multi-scenario modeling │
└───────────────┴─────────────────────────┘
MCP Inventory

- 5 existing (idm-knowledge, factset-estimates, alpha-knowledge, alpha-gateway, alpha-collector)
- 7 planned (ticker-resolution, adapter-instructions, portfolio-calcs, estimates, external-data, historical-data, rapi)

  ---
Check-in: Next Steps?

What would you like to focus on next?

1. Refine the draft - Add/remove products or themes?
2. Populate product directories - Create detailed docs for specific products?
3. Build the themes/mcps structure - Document MCP architecture?
4. Prioritize MCP roadmap - Which new MCPs to build first?
5. Something else - Different direction?



**************************

