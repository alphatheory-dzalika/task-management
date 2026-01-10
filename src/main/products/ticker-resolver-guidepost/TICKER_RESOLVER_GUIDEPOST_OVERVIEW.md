# Ticker Resolver Guidepost System

**Product Category**: P6 - Ticker Resolution & Symbology
**Source Project**: OpenAdapter (IdeaProjects/openAdapter)
**Status**: Production Architecture (CDC-Based)

---

## BLUF (Bottom Line Up Front)

The Guidepost system is a **CDC-based ticker resolution architecture** that eliminates direct SecMaster calls during batch processing. Instead of synchronous UDP lookups, it uses database writes that auto-sync via Debezium to TickerCache, achieving massive performance improvements for bulk ticker operations.

**Key Innovation**: Isolate all TickerMaster/SecMaster interactions into a single "box" (Phase 3), allowing Phases 1-2 and 4-5 to operate without network latency.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    GUIDEPOST TICKER RESOLUTION SYSTEM                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         INPUT LAYER                                  │   │
│  │                                                                      │   │
│  │  StateBeans (from Parser) ─────►  UUID Assignment ────► Tracking    │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                     │                                       │
│                                     ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    PHASE 1: PROCESS STATEBEANS                       │   │
│  │                         (NO SecMaster)                               │   │
│  │                                                                      │   │
│  │  • Extract ticker symbols from StateBeans                           │   │
│  │  • Resolve against existing ATDBTicker cache                        │   │
│  │  • Mark NEW vs EXISTING tickers                                     │   │
│  │  • Classify: EQUITY, OPTION, ETF, INDEX, FUTURE                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                     │                                       │
│                                     ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    PHASE 2: BATCH INSERT via CDC                     │   │
│  │                                                                      │   │
│  │  • INSERT new tickers to ATDBTicker table                           │   │
│  │  • Debezium captures INSERT events                                  │   │
│  │  • Events propagate to TickerCache                                  │   │
│  │  • NO direct UDP calls to SecMaster                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                     │                                       │
│                                     ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │              PHASE 3: TICKERMASTER ISOLATION CONTAINER               │   │
│  │                   (ONLY SecMaster Contact Point)                     │   │
│  │                                                                      │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  OPTION Processing                                           │    │   │
│  │  │  • Underlier resolution requires SecMaster                   │    │   │
│  │  │  • Strike/Expiry validation                                  │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                                                                      │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │  FALLBACK Processing                                         │    │   │
│  │  │  • Tickers not found in cache                                │    │   │
│  │  │  • Require SecMaster lookup                                  │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                     │                                       │
│                                     ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │              PHASE 4: BUILD SecMasterTicker FROM ATDBTicker          │   │
│  │                                                                      │   │
│  │  • Convert ATDBTicker → SecMasterTicker format                      │   │
│  │  • Enrich with cached SecMaster data                                │   │
│  │  • Prepare for downstream processing                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                     │                                       │
│                                     ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                  PHASE 5: PROCESS TICKER HISTORIES                   │   │
│  │                                                                      │   │
│  │  • Link historical price data                                       │   │
│  │  • Process corporate actions (splits, dividends)                    │   │
│  │  • Update historical references                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                     │                                       │
│                                     ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         OUTPUT LAYER                                 │   │
│  │                                                                      │   │
│  │  SecMasterTicker[] ────► Asset Creation ────► FundAsset Linking     │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Class Inventory (9 Classes, 6,096 Lines)

| Class | Lines | Purpose |
|-------|-------|---------|
| **TickerFactoryForIdmServicesGuidepost** | 2,476 | Main orchestrator - all 5 phases |
| **TickerFinderGuidepost** | 816 | Database lookup and cache resolution |
| **TickerCreatorGuidepost** | 754 | INSERT operations with CDC integration |
| **OptionTickerProcessorGuidepost** | 623 | Option-specific logic (underliers, strikes) |
| **TickerCacheManagerGuidepost** | 412 | Cache synchronization and invalidation |
| **TickerClassifierGuidepost** | 356 | Security type classification |
| **SecMasterBridgeGuidepost** | 298 | Isolated SecMaster/TickerMaster calls |
| **StateBeanTrackerGuidepost** | 234 | UUID tracking throughout phases |
| **TickerHistoryProcessorGuidepost** | 127 | Historical data linkage |

---

## Key Design Principles

### 1. CDC-First Architecture
- Database writes trigger Debezium events
- TickerCache subscribes to change events
- Eliminates need for synchronous cache updates
- Provides audit trail for all ticker changes

### 2. Phase Isolation
- Phases 1, 2, 4, 5: NO SecMaster network calls
- Phase 3: ONLY point of SecMaster contact
- Enables parallel processing for non-Phase-3 work
- Dramatically reduces network latency impact

### 3. UUID Tracking
- Every StateBean assigned UUID at entry
- UUID tracks through all 5 phases
- Enables debugging and audit trails
- Links input StateBeans to output Tickers

### 4. TickerMaster Isolation Container
- Single class (`SecMasterBridgeGuidepost`) makes all UDP calls
- Rate limiting and retry logic centralized
- Circuit breaker pattern for SecMaster outages
- Mock-friendly for testing

---

## Integration Points

### Input: StateBeans from Parser
```
Parser Output → StateBean[] → Guidepost Phase 1
```

### Output: SecMasterTicker for Asset Creation
```
Phase 5 Output → SecMasterTicker[] → AssetCreationService
```

### CDC Integration (Debezium)
```
ATDBTicker INSERT → Debezium Event → TickerCache Update
```

### SecMaster Integration (Phase 3 Only)
```
OptionTicker/Fallback → SecMasterBridgeGuidepost → TickerMaster UDP
```

---

## MCP Opportunity

**Proposed MCP**: `ticker-resolution-mcp`

**Tools**:
1. `resolve_ticker` - Single ticker resolution with phase tracking
2. `batch_resolve_tickers` - Bulk resolution with CDC monitoring
3. `get_ticker_cache_status` - Cache hit/miss statistics
4. `inspect_phase_3_queue` - View pending SecMaster lookups
5. `get_resolution_audit_trail` - UUID-based tracking lookup

**Resources**:
- `ticker://cache/statistics` - Real-time cache metrics
- `ticker://cdc/lag` - Debezium lag monitoring
- `ticker://secmaster/health` - SecMaster connectivity status

---

## Source Documentation

The complete technical documentation is maintained in OpenAdapter:

| Document | Purpose |
|----------|---------|
| `GUIDEPOST_TICKER_RESOLUTION_SYSTEM_COMPLETE_DOCUMENTATION.md` | Full architecture with code examples |
| `GUIDEPOST_ABSTRACTION_FINDORCREATE_TICKER_PLAN.md` | Planning for FindOrCreateTicker service |
| `GUIDEPOST_BEHAVIOR_ADDITIONS_FOR_MERGE_IN_PARITY.md` | Bug fixes and missing features |

---

## Related Products

- **P1 - OpenAdapter Core**: Parser → StateBean flow feeds Guidepost
- **P5 - FactSet Estimates & Multiples**: Uses resolved tickers for estimates
- **P7 - Portfolio Mapping**: Links resolved tickers to client positions
- **P11 - Historical Data Systems**: Phase 5 history processing

## Related Themes

- **T2 - CDC & Event-Driven**: Core architectural pattern
- **T4 - Batch Processing**: Phase-based bulk processing
- **T8 - Entity Resolution**: Ticker = primary entity resolution domain

---

*Last Updated: January 2026*
*Source: OpenAdapter/knowledgerepo/GUIDEPOST_*.md*
