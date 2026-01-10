# Guidepost Phase-by-Phase Processing

**Document Type**: Technical Deep Dive
**Source**: OpenAdapter Guidepost System

---

## Phase Processing Flow

```
StateBean[] ──► PHASE 1 ──► PHASE 2 ──► PHASE 3 ──► PHASE 4 ──► PHASE 5 ──► SecMasterTicker[]
                  │            │            │            │            │
                  │            │            │            │            │
               NO UDP       NO UDP      UDP CALLS     NO UDP       NO UDP
             (Cache Only)  (DB Write)  (Isolated)  (Transform)  (History)
```

---

## PHASE 1: Process StateBeans (NO SecMaster)

### Purpose
Extract ticker symbols from StateBeans and resolve against existing cache WITHOUT any network calls.

### Input
```java
List<StateBean> stateBeans  // Raw parsed data from adapter
```

### Processing Steps
1. **UUID Assignment**: Each StateBean gets unique tracking ID
2. **Symbol Extraction**: Pull ticker symbol from StateBean fields
3. **Cache Lookup**: Check ATDBTicker cache for existing match
4. **Classification**: Determine security type (EQUITY, OPTION, ETF, etc.)
5. **Routing Decision**: Mark as EXISTING (skip Phase 2) or NEW (proceed)

### Output
```java
List<TickerResolutionContext> contexts
  - stateBean: original input
  - uuid: tracking identifier
  - symbol: extracted ticker symbol
  - securityType: classified type
  - cacheHit: true/false
  - existingTicker: ATDBTicker or null
```

### Key Class
`TickerFinderGuidepost` - Handles cache lookup and classification

### Performance Target
- 10,000 StateBeans in < 500ms (cache-only, no network)

---

## PHASE 2: Batch INSERT via CDC

### Purpose
Insert NEW tickers into database, letting CDC propagate to cache. No direct cache manipulation.

### Input
```java
List<TickerResolutionContext> newTickers  // contexts where cacheHit = false
```

### Processing Steps
1. **Batch Preparation**: Group new tickers for bulk INSERT
2. **Database INSERT**: Write to ATDBTicker table
3. **CDC Capture**: Debezium captures INSERT events automatically
4. **Event Propagation**: Events flow to TickerCache consumer
5. **Cache Update**: TickerCache receives new tickers (async)

### Output
```java
List<ATDBTicker> insertedTickers  // With assigned TickerId
```

### Key Class
`TickerCreatorGuidepost` - Handles INSERT operations

### CDC Flow
```
ATDBTicker Table
      │
      ▼ (Debezium)
Kafka Topic: atdb.ticker.changes
      │
      ▼ (Consumer)
TickerCache.update(event)
```

### Important
- NO direct cache writes
- NO synchronous UDP calls
- Database is source of truth
- Cache eventually consistent (typically < 100ms)

---

## PHASE 3: TickerMaster Isolation Container

### Purpose
Handle ALL SecMaster/TickerMaster network calls in a single isolated phase.

### Why Isolated?
- UDP calls are slow (50-200ms each)
- Network calls can fail/timeout
- Rate limiting required for SecMaster
- Centralized retry and circuit breaker logic

### When Phase 3 is Needed

#### 1. OPTION Tickers
Options require underlier resolution:
```
AAPL 250117C00150000 → needs underlier AAPL → SecMaster lookup
```

#### 2. Fallback Lookups
Tickers not found in cache AND not in database:
```
NEW_TICKER_XYZ → not in ATDBTicker → SecMaster lookup required
```

### Processing Steps
1. **Queue Collection**: Gather all tickers requiring SecMaster
2. **Batch Request**: Send batched UDP requests to TickerMaster
3. **Response Processing**: Parse SecMaster responses
4. **Option Enrichment**: Link underliers to option tickers
5. **Cache Population**: Update cache with SecMaster data

### Key Classes
- `SecMasterBridgeGuidepost` - UDP communication
- `OptionTickerProcessorGuidepost` - Option-specific logic

### Rate Limiting
```java
// SecMaster rate limits
MAX_CONCURRENT_REQUESTS = 10
REQUEST_DELAY_MS = 50
TIMEOUT_MS = 5000
MAX_RETRIES = 3
```

### Circuit Breaker
```java
// If SecMaster unavailable
FAILURE_THRESHOLD = 5
RESET_TIMEOUT_MS = 30000
FALLBACK_MODE = QUEUE_FOR_RETRY
```

---

## PHASE 4: Build SecMasterTicker from ATDBTicker

### Purpose
Transform ATDBTicker (database entity) into SecMasterTicker (domain object) for downstream processing.

### Input
```java
List<ATDBTicker> resolvedTickers  // From Phases 1-3
```

### Processing Steps
1. **Entity Loading**: Fetch full ATDBTicker with all fields
2. **SecMaster Enrichment**: Add any cached SecMaster metadata
3. **Field Mapping**: Map ATDBTicker fields → SecMasterTicker fields
4. **Validation**: Verify required fields present
5. **Object Construction**: Build immutable SecMasterTicker

### Output
```java
List<SecMasterTicker> secMasterTickers
  - tickerId: from ATDBTicker
  - symbol: canonical symbol
  - securityType: classified type
  - exchange: trading venue
  - cusip/sedol/isin: identifiers
  - underlier: for options
  - strikePrice/expiration: for options
```

### Key Class
`TickerFactoryForIdmServicesGuidepost` (transformation methods)

### Field Mapping Example
```
ATDBTicker.tickerSymbol    → SecMasterTicker.symbol
ATDBTicker.securityTypeId  → SecMasterTicker.securityType (via lookup)
ATDBTicker.cusip           → SecMasterTicker.cusip
ATDBTicker.underlierId     → SecMasterTicker.underlier (via join)
```

---

## PHASE 5: Process Ticker Histories

### Purpose
Link historical data and process corporate actions for resolved tickers.

### Input
```java
List<SecMasterTicker> tickers
```

### Processing Steps
1. **History Lookup**: Find existing price history records
2. **History Linking**: Associate ticker with history tables
3. **Corporate Actions**: Process splits, dividends, spinoffs
4. **Adjustment Calculation**: Compute adjusted prices if needed
5. **History Gap Detection**: Identify missing historical periods

### Output
```java
List<SecMasterTicker> tickersWithHistory
  - historicalPrices: linked price records
  - corporateActions: processed events
  - adjustmentFactors: split/dividend factors
```

### Key Class
`TickerHistoryProcessorGuidepost`

### Corporate Action Types
- STOCK_SPLIT
- REVERSE_SPLIT
- DIVIDEND
- SPINOFF
- MERGER
- NAME_CHANGE

---

## Phase Execution Summary

| Phase | Network Calls | Database Ops | Cache Ops | Duration Target |
|-------|---------------|--------------|-----------|-----------------|
| 1 | None | Read only | Read only | < 500ms |
| 2 | None | Batch INSERT | None (CDC) | < 200ms |
| 3 | SecMaster UDP | UPDATE | Write | Variable (network) |
| 4 | None | Read joins | Read | < 100ms |
| 5 | None | Read/Write | Read | < 300ms |

### Total Target (10,000 tickers)
- **Cache hits (90%)**: ~1 second total
- **New tickers with cache**: ~3 seconds total
- **New tickers needing SecMaster**: ~30+ seconds (network dependent)

---

## Error Handling by Phase

### Phase 1 Errors
- Invalid StateBean format → Skip and log
- Cache connection failure → Fail entire batch

### Phase 2 Errors
- INSERT constraint violation → Mark as existing, retry lookup
- Database connection failure → Fail with retry

### Phase 3 Errors
- SecMaster timeout → Queue for retry
- UDP failure → Circuit breaker activation
- Rate limit exceeded → Backoff and retry

### Phase 4 Errors
- Missing required fields → Skip ticker with warning
- Join failures → Return partial data

### Phase 5 Errors
- History not found → Continue without history
- Corporate action parse error → Log and continue

---

*Document derived from GUIDEPOST_TICKER_RESOLUTION_SYSTEM_COMPLETE_DOCUMENTATION.md*
