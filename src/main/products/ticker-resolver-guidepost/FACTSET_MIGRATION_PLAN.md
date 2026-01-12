# FactSet Migration Master Plan

**Product Category**: P6 - Ticker Resolution & Symbology
**Program Status**: Active Migration
**Target Completion**: Q2 2026

---

## Executive Summary

The FactSet Migration transforms Alpha Theory's ticker infrastructure from legacy Sungard JTIC-based symbology to FactSet-powered resolution with Bloomberg Composite Ticker as the display standard. This is a foundational shift that enables:

- **Unified Symbology**: Bloomberg Composite Ticker as standard across all products
- **Enhanced Estimates**: Direct FactSet entity ID mapping for estimates/multiples
- **Modern Architecture**: CDC-based resolution (Guidepost system)
- **Ticker as Data Product**: Client-facing ticker services

---

## Phase Roadmap

| Phase | Focus | Target | Status |
|-------|-------|--------|--------|
| **0** | Ticker Stabilization & Quick Wins | Complete | Done |
| **1** | EOD Pricing (Scoring) | Staging/Prod | Code Complete |
| **2** | Prices: Intraday | Jan-Apr 2026 | Not Started |
| **3** | Prices: THCalcServices & SecMaster & TickerHistory | Jan-Apr 2026 | In Progress |
| **4** | Black-Scholes | Jan-Apr 2026 | Planning |
| **5** | Find or Create Ticker & FoC FundAsset | Jan-Apr 2026 | Planning |
| **6** | Ticker as Data Product | Q2 2026 | Future |

**Milestone**: Upon Phase 2 completion, ready to pay FactSet licensing

---

## Phase 0: Ticker Stabilization & Quick Wins (COMPLETE)

### Key Outcomes
- **11,116** Fund Asset client records processed since release
- **~3,500** unique equity tickers stabilized
- **153** assets updated to align with new standards
- **19** assets with active research (minimal client impact)

### Ticker Enhancements Delivered
| Enhancement | Count | Impact |
|-------------|-------|--------|
| Parent/ADR linkages added | 3,304 | Proper security hierarchy |
| Descriptions updated to FactSet standard | 2,835 | Consistent naming |
| FactSet IDs updated for Estimates | 688 | Better estimates mapping |
| Bloomberg tickers normalized | 226 | Regional exchange codes |

---

## Phase 1: EOD Pricing (Scoring)

### Status: Code Complete

### Path to Production

**Staging Requirements**:
- [ ] Run in staging environment
- [ ] Conform results in Ticker/FundAsset ExternalAppData
- [ ] Confirm correct use of ATSettings / DepartmentSettings

**Production Prerequisites**:
- [ ] Add new intermediate-stage data tables in prod
- [ ] Run IDM as a .jar in adapter (ATOP pattern)
- [ ] Final ATSettings / DepartmentSettings validation

### Technical Approach
SQL orchestration identified as quickest path to complete workflow.

**SQL Strategy**:
- Fast implementation
- Peer-reviewable
- Will require PostgreSQL conversion (or equivalent)

### Open Items
- SQL peer review needed
- Complete orchestration in progress
- Most layers correct, outcomes need validation

---

## Phase 2: Prices - Intraday

### Status: Ready to Start

### Immediate Actions
- Can begin now with existing infrastructure
- Robert available for capacity support

### Implementation Options

**Option A: Simple Start (No Additional Work)**
- Poll from Ticker table using `fsymIdExchangeLevel`
- Immediate value with existing data

**Option B: Dynamic Process (Additional Work)**
- Request only tickers held by clients
- Cross-reference tickers held IN THE APPLICATION
- More efficient, targeted polling

### Target
Jan-Apr 2026 completion unlocks FactSet payment readiness

---

## Phase 3: Prices - THCalcServices & SecMaster & TickerHistory

### Status: Orchestration In Progress

### Scope
- FactSet → TickerHistory data flow
- THCalcServices integration
- SecMaster synchronization

### Technical Notes
See "SQL Server as implementation choice vs. other choices" document for architecture decisions.

### Target
Jan-Apr 2026

---

## Phase 4: Black-Scholes

### Status: Planning Phase

### Primary Dependency
**Volatility data from FactSet** - Required before implementation can begin

### Scope
- Data structures for volatility storage
- No technical blockers identified
- Planning expected complete by **January 22, 2026**

### Target
Jan-Apr 2026 (after volatility availability)

---

## Phase 5: Find or Create Ticker & FoC FundAsset

### Status: Planning Phase

### What It Does
Port the Adapter's Ticker Resolver into an independent, reusable system.

### Key Capabilities

**Ticker Resolution**:
```
Incoming "Ticker Candidate"
    → Find best matching existing ticker
    → OR create new ticker
    → Return resolved SecMasterTicker
```

**FundAsset Resolution**:
```
Resolved Ticker + Client Environment
    → Search for ticker in client's department
    → Return existing FundAsset OR create new
```

### Architecture Reference
See `FINDORCREATE_TICKER_ABSTRACTION.md` for detailed design.

### Target
Jan-Apr 2026

---

## Phase 6: Ticker as Data Product

### Status: Future Planning

### Vision
Transform internal ticker infrastructure into client-facing services.

### Components

**1. JTIC Standard Decommissioning**

Current State:
- `Ticker.NAME` based on Sungard JTIC
- US: `Four_space_five_convention`
- CAD: `Ticker_asterisk`
- RestOfWorld: `S_Colon`

New Standard:
- Bloomberg Composite Ticker (widely accepted, open source)
- Backed by Bloomberg Global ID
- ISIN support
- MIC Exchange Code and details

**Migration Artifacts Needed**:
- Map: TickerID → Ticker.Name (OLD) as of cutover date

**2. Ticker Consolidation (RETIRING)**

When switching to new standard:
- Some tickers will resolve to same Bloomberg Composite Ticker
- Rule needed for determining primary ticker (usually JTIC-based one)

**3. Delisting Process (DELISTING)**

Decisions needed:
- Setting "expired" value
- Potentially adding "delisted" field
- Price removal process
- Handling clients with YTD portfolio views (delisted ticker references)
- Identify test clients for validation

**4. Ticker Lifecycle Orchestration**

Build orchestration for:
- Capital changes
- Corporate actions events
- Identifier changes

**5. External Client Communication**

**CBP & Masterfile Support**:
- Alpha Theory provides "validated Identifier explanation" with each ticker
- Work with CBP as contractors (CBP licensing, CBP AWS instances)
- Build Bloomberg SAPI/LSEG Masterfile connectors
- Exact entity mappings visible (ADR/non-primary, Chinese Foreign vs. National)

**Alpha Theory Ticker Lookup Service**:
- Explain to clients exactly how incoming data is treated
- Clients know coverage gaps upfront
- Prepare supplementary data sources accordingly

### Target
Q2 2026

---

## Key Statistics Summary

```
┌────────────────────────────────────────────────────────────┐
│           FACTSET MIGRATION KEY METRICS                    │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Fund Assets Processed:     11,116                         │
│  Unique Equity Tickers:     ~3,500                         │
│  Assets Updated:            153                            │
│  Active Research Impact:    19 (minimal)                   │
│                                                            │
│  TICKER ENHANCEMENTS                                       │
│  ─────────────────────                                     │
│  Parent/ADR Linkages:       +3,304                         │
│  Descriptions Updated:      2,835                          │
│  FactSet IDs Updated:       688                            │
│  Bloomberg Tickers Norm:    226                            │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Strategic Outcome

**Significant net benefit and important step toward making FactSet our primary vendor**

The migration establishes:
1. **Modern Symbology**: Bloomberg Composite Ticker standard
2. **Vendor Alignment**: FactSet as primary data provider
3. **Client Services**: Ticker as a data product offering
4. **Architecture**: CDC-based, scalable resolution system

---

## Related Documents

| Document | Location | Purpose |
|----------|----------|---------|
| Guidepost Overview | `TICKER_RESOLVER_GUIDEPOST_OVERVIEW.md` | CDC architecture |
| Phase Processing | `PHASE_BY_PHASE_PROCESSING.md` | Technical phases |
| FindOrCreate Abstraction | `FINDORCREATE_TICKER_ABSTRACTION.md` | Service design |
| EOD Pricing Plan | TBD | Path to production |
| SQL vs Other Choices | TBD | Architecture rationale |

---

*Last Updated: January 2026*
*Program Owner: FactSet Migration Team*
