# Product and Themes Catalog Discovery

## Purpose

This discovery document captures capabilities found across Alpha Theory's major codebases to inform the products and themes architecture in tracking-review. It serves as the foundation for:
- Cross-team knowledge sharing
- MCP integration planning
- Documentation funnel design

---

## Source Projects Analyzed

| Project | Type | Status | Key Role |
|---------|------|--------|----------|
| **dme (IDM)** | Data Integration | Active | Job orchestration, Natural Keys, MCP servers |
| **openAdapter** | ETL Framework | Production | Ticker resolution, FIGI, Instruction Containers |
| **dptv2** | Portfolio Platform | Production | Calculations, Portfolio Management, Market Data |
| **internal-alpha-theory-mcp** | Knowledge Layer | Active | Gateway, Collector, Client Knowledge |
| **pyRAPI** | Research API | Production | Portfolio analytics |
| **securitymaster** | Reference Data | Production | Master data |

---

## PRODUCTS DISCOVERED

### Category 1: Core Portfolio Platform (DPTv2)

#### P1: Portfolio Calculations Engine

**Source**: DPTv2 `/calcs/`, `/src/main/webapp/js/`
**Status**: Production (mature)

**Capabilities**:
- Multi-scenario modeling (Bear/Base/Bull + custom, up to 10 per asset)
- Probability-weighted returns
- Real-time calculation modes (Browser vs RAPI/Server)
- SIMD JSON parsing, worker threads for performance
- 7 custom logic injection points

**Key Components**:
- `calcs/server.js` - Node.js Fastify server
- `calcs/calculations.cjs` - Auto-generated calculation logic
- `at_asset_calculations.js` - Core calculation source
- `at_field_utils.js` - Field definitions

**MCP Opportunity**: `portfolio-calcs-mcp`
- Tools: run_calculation, get_scenario_results, explain_field_formula

---

#### P2: Estimates Feature

**Source**: DPTv2 (Active Development - Phases 1-2)
**Status**: In Progress

**Capabilities**:
- Analyst financial forecasts (Revenue, EPS, EBITDA, 14 standard + custom KPIs)
- Consensus comparison (variance to Street)
- Multi-scenario estimates (up to 10 per asset)
- Variance trajectory tracking (gap widening/narrowing)
- Historical estimate evolution graphs

**Key Documentation**: `docs/ESTIMATES_FEATURE_DESIGN.md` (1950+ lines)

**Roadmap**:
- Phase 1-2: Database schema + REST APIs (current)
- Phase 3: Angular widget in Asset Dashboard
- Phase 4: Full-screen view + portfolio variance
- Phase 5: Excel Connect, RAPI integration
- Phase 6: React migration

**MCP Opportunity**: `estimates-mcp`
- Tools: get_estimate, compare_to_consensus, get_variance_trajectory

---

#### P3: Market Data Integration

**Source**: DPTv2 + DME
**Status**: Production

**Capabilities**:
- FactSet automated data (prices, estimates, corporate actions)
- Three workflows: INIT (seed), TOPOFF (current), BACKFILL (extend)
- TickerHistory (end-of-day) + TickerHistoryLive (real-time)
- Corporate action adjustment with REFRESH mode
- Priority-based processing (active equities first)

**Key Documentation**: `docs/FDS_PRICE_PIPELINE_GUIDE.md`

**MCP Coverage**: `factset-estimates-mcp` (partial)

---

#### P4: Custom Logic & Extensibility

**Source**: DPTv2
**Status**: Production

**Capabilities**:
- Custom fields with JavaScript injection
- Custom formulas (formula builder UI)
- Layout overrides (customer-specific forms)
- Configuration overrides (dept/user level)
- 7 injection points in calculation pipeline:
  1. Function (Setup) - Pre-calculation data adjustment
  2. Function (Pre-Scenarios) - After current calculations
  3. Function (Pre-Optimal) - Before scenario/optimal
  4. Function (Post-Optimal) - After scenario/optimal
  5. Function (Pre-Exposures) - Before constraint processing
  6. Function (Post-Exposures) - After constraint processing
  7. (Additional injection point)

**Key Documentation**: `docs/CX_CUSTOM_LOGIC_DOCUMENTATION.md`

**MCP Opportunity**: Document custom logic patterns for CX team

---

### Category 2: Data Integration (IDM/DME)

#### P5: Ticker Resolution & Symbology

**Source**: OpenAdapter (primary), DME
**Status**: Production

**Capabilities**:
- Resolve 60+ security types (equities, options, futures, bonds, baskets)
- Multi-source validation (FIGI, FactSet, Bloomberg)
- Scoring-based selection with tie-breaking
- 4-phase atomic workflow:
  1. Data Extraction
  2. Symbology Extraction
  3. FIGI Batch Assignment
  4. Ticker Resolution

**Key Components**:
- `TickerResolverServices.java` (2,781 lines)
- `OpenFigiFactSetSymbologyExtractionServices.java` (1,786 lines)
- `ClientSourceDataFileInstanceOpenFigiRequestManager.java` (959 lines)

**MCP Opportunity**: `ticker-resolution-mcp`
- Tools: resolve_ticker, validate_symbology, get_figi_mapping, explain_resolution

---

#### P6: Adapter Instructions (Instruction Containers)

**Source**: OpenAdapter
**Status**: Production

**Capabilities**:
- Configuration-driven parsing (JSON rules, no code deployment)
- 10 instruction types: field mapping, conditionals, computed fields, scenarios
- Dynamic field resolution (FX rates, currency lookup)
- Replaces 200+ legacy client-specific parsers
- Supplemental data enrichment (join external datasets)

**Key Components**:
- `ExternalAppInstructionsContainer.java` (456 lines)
- `GenericParserExternalAppInstructionsContainer.java`
- `ATDBExternalAppSettingsCustom` (database configuration)

**MCP Coverage**: Partial in `idm-knowledge-mcp`

---

#### P7: External App Data Integration

**Source**: DME + DPTv2
**Status**: Production

**Capabilities**:
- External-to-internal data model translation
- Natural key resolution (bidirectional)
- approvedDataResults JSON injection pattern
- StateBean field processing
- FactSet analytics injection via External App system

**Key Components**:
- `ExternalAppDataMappingServices.java` (DME)
- `NaturalKeyEntityBuilderServices.java` (DME)
- `populateFundAssetAttributesExternalApps()` (DPTv2)

**MCP Opportunity**: `external-data-mcp`

---

#### P8: Portfolio Mapping

**Source**: DME + OpenAdapter
**Status**: Production

**Capabilities**:
- Client mapping analysis with ticker quality
- Department-level QC workflows
- Stored procedure orchestration (3-step pipeline)
- Support user provisioning

**Key Components**:
- `PortfolioMappingQCController.java` (DME)
- `PortfolioMappingQCByDepartmentController.java` (DME)

**MCP Coverage**: Extend `idm-knowledge-mcp`

---

### Category 3: Research & Analytics

#### P9: RAPI (Research API)

**Source**: DME + pyRAPI + DPTv2
**Status**: Production

**Capabilities**:
- Portfolio calculations and analytics
- Fast fields for performance
- Timeseries data management
- Delta-adjusted exposure analysis
- Two calculation modes: Browser (interactive) vs RAPI (batch/API)

**Key Components**:
- `RapiRequestController.java` (DME)
- `RapiTimeseriesController.java` (DME)
- pyRAPI Python client
- `jav8/alphatheory/` (DPTv2 - RAPI mode JavaScript)

**MCP Coverage**: `rapi-mcp` (referenced in gateway)

---

#### P10: Fund Asset Aggregation

**Source**: DME
**Status**: Active Development

**Capabilities**:
- Bulk aggregation for departments/funds
- Underlier resolution
- Property transfer between entities
- Scenario and sector aggregation

**Key Components**:
- `FundAssetAggregationController.java` (DME)
- `FundAssetAggregationJobController.java` (DME)

**MCP Opportunity**: Extend `idm-knowledge-mcp`

---

### Category 4: Client & Knowledge

#### P11: Client Knowledge & CX Servicing

**Source**: internal-alpha-theory-mcp + DME
**Status**: Active Development

**Capabilities**:
- Client fulfillment view ("For Whom, To Give What")
- Health scores and risk assessment
- CX workflow automation
- Salesforce bidirectional sync
- 12 real DPT clients seeded

**Key Components**:
- `alpha-knowledge-mcp` (27+ tools)
- `CXServicingWorkflowService.java` (DME)

**MCP Coverage**: Full (`alpha-knowledge-mcp`)

---

#### P12: Historical Data Management

**Source**: DME + DPTv2
**Status**: Active Development

**Capabilities**:
- Department snapshots (SQLite)
- Timeseries storage
- Data cube for analytics
- H-prefixed history tables (HAssetEstimate, HAssetScenario)

**Key Components**:
- `DepartmentSnapshotDatabaseService.java` (DME)
- `TimeseriesStorageService.java` (DME)

**MCP Opportunity**: `historical-data-mcp`

---

## THEMES DISCOVERED

### T1: MCPs (Model Context Protocol)

**Purpose**: Standardized AI tool integration across all products

**Current MCP Inventory**:

| MCP Server | Project | Tools | Language | Status |
|------------|---------|-------|----------|--------|
| idm-knowledge-mcp | DME | 19 | Java | Production |
| factset-estimates-mcp | DME | ~10 | Java | Production |
| alpha-knowledge-mcp | internal-mcp | 27 | Python | Production |
| alpha-gateway-mcp | internal-mcp | 5 | Python | Production |
| alpha-collector-mcp | internal-mcp | 9 | Python | Production |
| yahoo-finance-mcp | Standalone | TBD | Python | Development |
| rapi-mcp | Referenced | TBD | TBD | Planned |

**Planned MCPs**:
- `ticker-resolution-mcp` (OpenAdapter knowledge)
- `adapter-instructions-mcp` (configuration patterns)
- `portfolio-calcs-mcp` (calculation explanation)
- `estimates-mcp` (analyst forecasts)
- `external-data-mcp` (data mappings)
- `historical-data-mcp` (timeseries)

**Architecture Pattern**:
```
User Query → alpha-gateway-mcp (intent detection)
                    ↓
            Route to appropriate MCP
                    ↓
    ┌───────────────┼───────────────┐
    ↓               ↓               ↓
alpha-knowledge  idm-knowledge  factset-estimates
(clients)        (integration)  (market data)
```

---

### T2: Data Flow Architecture

**Purpose**: Consistent data movement patterns across systems

**Flow Types Discovered**:

| Flow | Pattern | Projects |
|------|---------|----------|
| **Standard CRUD** | DB → BO → DTO → DataSource → Frontend | DPTv2 |
| **CDC/Cache** | DB Change → Debezium → Cache → Calcs | DPTv2 |
| **External Data Streaming** | Provider → SecMaster → TickerHistoryLive → Cache | DPTv2 |
| **External App Injection** | API → JSON → approvedDataResults → Record | DPTv2, DME |
| **Atomic Workflow** | Phase 1 → DB → Phase 2 → DB → Phase 3 | OpenAdapter |

**Key Insight**: Each phase writes to database tables read by next phase (idempotency)

---

### T3: Job Orchestration

**Purpose**: Consistent job management across systems

**Patterns by Project**:

| Project | Pattern | Key Features |
|---------|---------|--------------|
| **DME** | 8-endpoint REST | Locks, cooldowns, UnifiedJobManagementFacade |
| **DPTv2** | Quartz Scheduler | Background jobs, Jenkins integration |
| **OpenAdapter** | Atomic Workflow | 4-phase processing, service delegation |

**DME 8-Endpoint Pattern**:
1. `POST /jobs/execute` - Submit
2. `GET /jobs/{id}` - Status
3. `POST /jobs/{id}/cancel` - Cancel
4. `GET /jobs/active` - List active
5. `GET /jobs/recent` - List recent
6. `GET /storage/locks` - View locks
7. `GET /storage/statistics` - Stats
8. `GET /storage/cooldowns` - Cooldowns

**Cooldown Strategy** (DME):
- Short history (≤30 days): 30-minute cooldown
- Medium history (31-252 days): 2-hour cooldown
- Long history (>252 days): 8-hour cooldown

---

### T4: Entity Identity Management

**Purpose**: Consistent entity resolution across systems

**Patterns**:

| Pattern | Description | Projects |
|---------|-------------|----------|
| Natural Keys | Human-readable (`ticker:AAPL`, `client:tarheel`) | DME, internal-mcp |
| Surrogate Keys | Internal IDs (tickerId, fundId) | All |
| EntityKeyMapping | Bidirectional resolution cache | DME |
| Symbology Hash | Deduplication by identifier hash | OpenAdapter |

**Resolution Flow** (DME):
1. Check EntityKeyMapping cache
2. If miss, query database via RowMapper
3. Convert ATDB entity to External entity
4. Create EntityKeyMapping entry

---

### T5: Configuration-Driven Processing

**Purpose**: Eliminate custom code through declarative rules

**Implementations**:

| System | Mechanism | Storage |
|--------|-----------|---------|
| **OpenAdapter** | Instruction Containers | JSON in `ATDBExternalAppSettingsCustom` |
| **DPTv2** | Configuration Overrides | JavaScript via `eval()` |
| **DME** | Batch Definitions | Database templates |

**OpenAdapter Instruction Types**:
1. Field mapping
2. Conditionals (AND/OR expressions)
3. Computed fields (multiply, divide, add)
4. Scenarios (probability nodes)
5. Fund aggregation
6. Supplemental data enrichment
7. Intermediate field dependencies
8. Custom tags
9. Method resolution (FX rates, currency lookup)
10. Override system

**Benefit**: New client onboarding without code deployment

---

### T6: Calculation Pipeline

**Purpose**: Financial calculation patterns

**DPTv2 Calculation Modes**:

| Mode | Location | Setting | Use Case |
|------|----------|---------|----------|
| Browser | User's browser | calculationEngine=BROWSER | Interactive |
| RAPI | Server (Node.js/Java V8) | calculationEngine='rapi' | Batch/API |

**Injection Points** (7 total):
1. Setup - Pre-calculation data adjustment
2. Pre-Scenarios - After current calculations
3. Pre-Optimal - Before scenario/optimal
4. Post-Optimal - After scenario/optimal
5. Pre-Exposures - Before constraint processing
6. Post-Exposures - After constraint processing

**DME RAPI Patterns**:
- Delta-adjusted exposures
- Black-Scholes calculations
- Option contract handling
- Fast fields for performance

---

### T7: Vendor API Integration

**Purpose**: Consistent external API consumption

**Patterns**:

| Vendor | Rate Limiting | Batching | Projects |
|--------|---------------|----------|----------|
| OpenFIGI | 26 token pool | 75/request | OpenAdapter |
| FactSet | API key | Configurable | DME, DPTv2, OpenAdapter |
| Yahoo Finance | Rate limits | TBD | yahoo-finance-mcp |
| Salesforce | OAuth | Per-object | internal-mcp |

**OpenFIGI Pattern** (Gold Standard):
- Token pool management (26 concurrent)
- Batch grouping (75 identifiers per HTTP call)
- Iterative supplementary requests
- Response deduplication by hash
- Full audit trail

---

### T8: Data Storage

**Purpose**: Consistent persistence patterns

**Storage Types**:

| Type | Use Case | Projects |
|------|----------|----------|
| SQL Server | Transactional data | All |
| PostgreSQL | Documentation DB, migration target | OpenAdapter, DPTv2 |
| SQLite | Knowledge DBs, snapshots | DME, internal-mcp |
| Redis | Caching, CDC | DME, DPTv2 |
| MinIO/S3 | Object storage | internal-mcp |

**SQLite Pattern** (Knowledge DBs):
- FTS5 for full-text search
- Portable, single-file databases
- Schema versioning
- Export to SQL for distribution

---

### T9: Documentation Funnel

**Purpose**: Knowledge capture and cross-team communication

**Layers**:

```
┌─────────────────────────────────────────────────────────┐
│ Layer 4: Enterprise (Jira/Confluence/Salesforce)        │
├─────────────────────────────────────────────────────────┤
│ Layer 3: MCP-Accessible (queryable by AI)               │
├─────────────────────────────────────────────────────────┤
│ Layer 2: Products & Themes (tracking-review)            │
├─────────────────────────────────────────────────────────┤
│ Layer 1: Task-Level (weekly reviews, session summaries) │
└─────────────────────────────────────────────────────────┘
```

**Current Documentation**:
- DME: 76+ markdown files, MCP knowledge DB
- OpenAdapter: 76+ markdown files, PostgreSQL doc DB
- DPTv2: 100+ markdown files, Confluence auto-publish
- internal-mcp: Collector for universal storage

---

### T10: Multi-Scenario Modeling

**Purpose**: Financial scenario analysis patterns

**DPTv2 Scenarios**:
- Standard: Bear / Base / Bull
- Custom: Up to 10 per asset
- Probability weighting per scenario
- Price targets with node-based probabilities

**DME Estimates**:
- Multi-metric (14 standard + custom KPIs)
- Multi-scenario support
- Variance to consensus tracking

---

## PRODUCT-THEME MATRIX

|  | T1 MCPs | T2 Data Flow | T3 Jobs | T4 Identity | T5 Config | T6 Calcs | T7 Vendors | T8 Storage | T9 Docs | T10 Scenarios |
|--|---------|--------------|---------|-------------|-----------|----------|------------|------------|---------|---------------|
| **P1 Calcs** | Planned | Yes | - | - | Yes | **Primary** | - | - | Yes | **Primary** |
| **P2 Estimates** | Planned | Yes | - | - | - | Yes | Yes | Yes | Yes | **Primary** |
| **P3 Market Data** | Partial | **Primary** | Yes | - | - | - | **Primary** | Yes | Yes | - |
| **P4 Custom Logic** | - | - | - | - | **Primary** | Yes | - | - | Yes | - |
| **P5 Ticker** | Planned | Yes | Yes | **Primary** | - | - | **Primary** | Yes | Yes | - |
| **P6 Instructions** | Partial | Yes | - | - | **Primary** | - | - | Yes | Yes | - |
| **P7 External App** | Planned | **Primary** | Yes | **Primary** | Yes | - | Yes | Yes | Yes | - |
| **P8 Portfolio Map** | Partial | Yes | Yes | Yes | - | - | - | Yes | Yes | - |
| **P9 RAPI** | Partial | Yes | Yes | - | - | **Primary** | - | Yes | Yes | Yes |
| **P10 Aggregation** | Planned | Yes | **Primary** | Yes | - | Yes | - | Yes | Yes | - |
| **P11 CX/Client** | **Full** | - | - | Yes | - | - | Yes | Yes | Yes | - |
| **P12 Historical** | Planned | Yes | Yes | - | - | - | - | **Primary** | Yes | - |

---

## MCP INTEGRATION ROADMAP

### Phase 1: Foundation (Current)
- [x] alpha-knowledge-mcp (clients, glossary)
- [x] alpha-gateway-mcp (routing)
- [x] alpha-collector-mcp (storage)
- [x] idm-knowledge-mcp (integration patterns)
- [x] factset-estimates-mcp (market data)

### Phase 2: Core Products
- [ ] ticker-resolution-mcp (P5 - symbology)
- [ ] adapter-instructions-mcp (P6 - configuration)
- [ ] external-data-mcp (P7 - mappings)

### Phase 3: Analytics
- [ ] portfolio-calcs-mcp (P1 - calculations)
- [ ] estimates-mcp (P2 - forecasts)
- [ ] rapi-mcp (P9 - research API)

### Phase 4: Operations
- [ ] historical-data-mcp (P12 - timeseries)
- [ ] job-orchestration-mcp (T3 - job management)

---

## CROSS-TEAM COMMUNICATION MODEL

### How This Catalog Enables Collaboration

1. **Shared Vocabulary**
   - Products define what we build
   - Themes define how we build
   - Both teams use same terms

2. **Transparent Discovery**
   - All findings documented here
   - PRs to tracking-review for changes
   - Weekly reviews reference this catalog

3. **MCP Integration Points**
   - MCPs expose knowledge to AI
   - Gateway routes queries to right MCP
   - Collector stores shared artifacts

4. **Storage for Artifacts**
   - SQLite for portable knowledge DBs
   - MinIO/S3 for larger artifacts
   - GitHub for code and documentation

### Contributing

Each team contributes by:
1. Documenting capabilities in `/products/`
2. Identifying patterns for `/themes/`
3. Proposing MCP tools for AI accessibility
4. Participating in weekly reviews

---

## JIRA & DOCUMENTATION INFRASTRUCTURE DISCOVERY

### Overview Statistics

| Metric | Count |
|--------|-------|
| Total markdown files across IdeaProjects | 1,627 |
| JIRA-specific markdown files | 6 |
| IDM-prefixed documentation files | 16 |
| Roadmap documents | 7 |
| Projects with documentation | 33 |

---

### JIRA Integration Documents

#### Primary JIRA Documentation

| File | Project | Purpose |
|------|---------|---------|
| `CLAUDE_JIRA_INTEGRATION_GUIDE.md` | dme | Claude Code + JIRA API integration |
| `JIRA_SETUP_INSTRUCTIONS.md` | dme | JIRA setup and configuration |
| `JIRA_WORK_TAXONOMY_SPEC.md` | dptv2 | Work categorization system |
| `JIRA_GITHUB_INTEGRATION_OPTIONS.md` | securitymaster | GitHub Actions + JIRA automation |
| `.claude/commands/create-jira-task.md` | dptv2 | Claude skill for JIRA tasks |
| `.claude/commands/create-frontend-jira-task.md` | dptv2 | Frontend-specific JIRA tasks |

#### Key Capabilities from JIRA Documentation

**Work Taxonomy (DPTv2)** - Universal classification system:
- Client-Driven: Support, Onboarding, SOW
- Roadmap: Foundational, IT, Product
- Unplanned: Ad-hoc, urgent work
- Target: 45% Client, 45% Roadmap, 10% Unplanned

**JIRA API Integration (DME)** - 20 IDM Components defined:
- Portfolio Mapping, Adapter Orchestration, FactSet Estimates
- Ticker Resolver, OpenFIGI, Natural Key Systems
- CX-Servicing, External-App-Data, Infrastructure

**GitHub Actions Integration** - Automated workflows:
- PR comments to JIRA tickets
- Status transitions on merge
- Branch-based ticket detection

---

### IDM Documentation Inventory (dme project)

#### Root-Level IDM Files

| File | Size | Purpose |
|------|------|---------|
| `IDM_SYSTEM_STATUS_AND_ROADMAP.md` | 95KB | Comprehensive status (LARGEST) |
| `IDM_SYSTEM_CAPABILITIES_OVERVIEW.md` | 45KB | System capabilities |
| `IDM_COMPLETE_PROJECT_ROADMAP.md` | 44KB | Full project roadmap |
| `GUIDEPOST_FOR_IDM_IMPLEMENTATION_PLAN.md` | 42KB | Implementation guide |
| `IDM_MEDALLION_ARCHITECTURE_AND_FINANCIAL_CALCS.md` | 41KB | Architecture patterns |
| `IDM_ROADMAP.md` | 14KB | Condensed roadmap |
| `IDM-39_IMPLEMENTATION_GUIDE.md` | 9KB | Specific ticket guide |
| `IDM_COMPONENT_ALIGNMENT.md` | 9KB | Component mapping |

#### Subdirectory IDM Documentation

| Path | File | Purpose |
|------|------|---------|
| `database/` | `IDM_KNOWLEDGE_INVENTORY.md` | Knowledge DB inventory |
| `dme-mcp-servers/docs/` | `IDM_KNOWLEDGE_IMPLEMENTATION.md` | MCP implementation |
| `docs/AtomicAdapter/` | `IDM_OPENADAPTER_MAPPING.md` | OpenAdapter integration |
| `docs/architecture/` | `IDM_IDENTITY_GOVERNANCE_UNIFIED_ARCHITECTURE.md` | Identity patterns |
| `docs/architecture/` | `IDM_TAXONOMY_AND_ANNOTATION_ALIGNMENT.md` | Taxonomy alignment |
| `docs/planning/` | `IDM_DEVELOPMENT_BOARD.md` | Active work tracking |
| `docs/planning/` | `IDM_GLOSSARY.md` | Term definitions |
| `docs/planning/` | `IDM_ORCHESTRATION_AND_DATA_GOVERNANCE_ANALYSIS.md` | Governance |
| `docs/planning/` | `IDM_TAXONOMY_AND_NAMING_SYSTEM.md` | Naming conventions |

---

### Ticket-Referenced Documentation

#### By Ticket Pattern

| Pattern | Files Found | Example |
|---------|-------------|---------|
| `IDM-*` | 1 explicit | `IDM-39_IMPLEMENTATION_GUIDE.md` |
| `IN-*` | 2 | `IN-3587.md`, `BRANCH_IN-3513_v2_README.md` |
| `SM-*` | Referenced in code | securitymaster branches |
| `FAC-*` | 0 explicit | (referenced in git branches) |
| `DPT-*` | 0 explicit | (referenced in git branches) |

#### Git Branches with Ticket Numbers (dme)

Active/Recent IDM tickets found in git logs:
- IDM-21, IDM-24, IDM-27, IDM-28, IDM-29, IDM-30
- IDM-31, IDM-32, IDM-33, IDM-34, IDM-35
- IDM-39, IDM-40, IDM-42, IDM-43, IDM-45
- IDM-47, IDM-48 (current branch)

---

### Standalone Ticket Projects

#### IDM-33 Project

**Location**: `/mnt/c/Users/dzalika/IdeaProjects/IDM-33`

**Documentation**:
| File | Size | Topic |
|------|------|-------|
| `EXTERNAL_OBJECT_BUILDER_PROPOSAL.md` | 14KB | External object design |
| `PAPI_INTEGRATION_GUIDE.md` | 21KB | PAPI integration |
| `SUBORDINATED_ASSET_NATURAL_KEY_DESIGN.md` | 16KB | Natural key design |

---

### Roadmap Documents Across Projects

| Project | File | Focus |
|---------|------|-------|
| dme | `IDM_ROADMAP.md` | IDM project roadmap |
| dme | `IDM_COMPLETE_PROJECT_ROADMAP.md` | Comprehensive roadmap |
| dme | `IDM_SYSTEM_STATUS_AND_ROADMAP.md` | Status + roadmap combined |
| dme | `CX_SERVICING_COMPLETE_IMPLEMENTATION_ROADMAP.md` | CX feature roadmap |
| dme | `docs/aws-implementation/14-MIGRATION-ROADMAP.md` | AWS migration |
| dptv2 | `docs/PERFORMANCE_OPTIMIZATION_ROADMAP.md` | Performance improvements |
| securitymaster | `SECMASTER_ROADMAP_INTEGRATION.md` | SecMaster integration |

---

### Documentation by Project Size

| Project | MD Files | Key Documentation Areas |
|---------|----------|------------------------|
| **dptv2** | 861 | Estimates, custom logic, calculations, deployment |
| **dme** | 354 | IDM system, architecture, planning, MCPs |
| **openAdapter** | 182 | Ticker resolution, FIGI, client support |
| **securitymaster** | 42 | JIRA integration, roadmap |
| **HistoricalAnalysis** | 32 | Analysis documentation |
| **esapi-jpa** | 25 | Legacy system docs |
| **uptime-kuma-wiki** | 24 | Monitoring wiki |
| **esapi-services** | 20 | Legacy services |

---

### Related Documentation Clusters

**Cluster 1: JIRA/Project Management**
- Work taxonomy (dptv2)
- Claude JIRA integration (dme)
- GitHub Actions integration (securitymaster, dme)
- Weekly task reviews (tasking-review)

**Cluster 2: IDM System Documentation**
- System status and roadmap (95KB comprehensive)
- Component alignment and taxonomy
- Architecture patterns
- MCP implementation guides

**Cluster 3: Ticket-Specific Implementation**
- IDM-39 (Portfolio mapping TRUNCATE fix)
- IDM-33 (External object builder)
- IN-3587 (Futures ticker treatment)
- IN-3513 (OpenAdapter branch)

**Cluster 4: Roadmaps**
- IDM project roadmap
- CX servicing roadmap
- AWS migration roadmap
- Performance optimization roadmap

---

### MCP Opportunity: JIRA/Documentation Integration

**Proposed**: `atlassian-mcp` or `jira-mcp`

**Tools**:
- `search_tickets` - JQL queries
- `get_ticket_details` - Full ticket info
- `create_ticket` - New ticket creation
- `add_comment` - Add comments
- `transition_ticket` - Change status
- `link_tickets` - Create relationships

**Integration Points**:
- alpha-gateway can route JIRA queries
- alpha-collector can store ticket metadata
- Claude skills can use JIRA tools

---

## NEXT STEPS

### Immediate
1. Review this discovery with stakeholders
2. Prioritize products for detailed documentation
3. Identify first MCP to build from this catalog

### Short-Term
4. Populate `/products/` directories
5. Populate `/themes/mcps/` with architecture
6. Create product-to-MCP mapping specs

### Medium-Term
7. Implement prioritized MCPs
8. Integrate with alpha-gateway for routing
9. Establish cross-team review cadence

---

*Discovery Date: January 2026*
*Sources: dme, openAdapter, dptv2, internal-alpha-theory-mcp*
*Status: DRAFT - Initial Discovery Complete*
