# Guidepost Known Issues & Implementation Guidance

**Document Type**: Bug Fixes & Missing Features
**Source**: GUIDEPOST_BEHAVIOR_ADDITIONS_FOR_MERGE_IN_PARITY.md
**Status**: Implementation Backlog

---

## Priority Matrix

| Priority | Description | Target |
|----------|-------------|--------|
| **P0** | Critical bugs blocking production | Immediate |
| **P1** | Important missing features | Current sprint |
| **P2** | Enhancements for parity | Next sprint |
| **P3** | Nice-to-have improvements | Backlog |

---

## P0: Critical Bug Fixes

### 1. Option Expiration Date Handling

**Issue**: Option expiration dates not properly handled during resolution
**Impact**: Options may have incorrect expiration, causing pricing/position errors
**Location**: `OptionTickerProcessorGuidepost`

**Fix Required**:
```java
// Current: Expiration may be null or incorrectly parsed
// Fix: Validate and normalize expiration during Phase 3

private LocalDate normalizeExpiration(String optionSymbol) {
    // Parse OCC format: AAPL 250117C00150000
    // Extract: 250117 = 2025-01-17
    String dateStr = extractDateFromOcc(optionSymbol);
    return parseOccDate(dateStr);
}
```

**Test Cases**:
- Standard OCC format options
- Weekly options
- Quarterly options
- LEAPS

---

### 2. Missing Option Resolution Data

**Issue**: Some option fields not populated during SecMaster lookup
**Impact**: Incomplete option records in database
**Location**: `SecMasterBridgeGuidepost`

**Missing Fields**:
- `exerciseStyle` (American/European)
- `multiplier` (usually 100)
- `deliverableType` (Cash/Physical)

**Fix Required**:
```java
private void enrichOptionFromSecMaster(ATDBTicker option, SecMasterResponse response) {
    option.setExerciseStyle(response.getExerciseStyle());
    option.setMultiplier(response.getMultiplier() != null ? response.getMultiplier() : 100);
    option.setDeliverableType(response.getDeliverableType());
}
```

---

### 3. SecMaster Persistence Gap

**Issue**: SecMaster responses not always persisted before CDC event fires
**Impact**: Race condition where cache updates before database commit
**Location**: `TickerCreatorGuidepost`

**Fix Required**:
```java
@Transactional
public ATDBTicker createWithPersistence(TickerResolutionContext context) {
    // 1. Insert to database
    ATDBTicker ticker = repository.save(buildTicker(context));

    // 2. Force flush to ensure CDC captures committed data
    repository.flush();

    // 3. Return with database-assigned ID
    return ticker;
}
```

---

## P1: Important Missing Features

### 4. Batch SecMaster Optimization

**Issue**: SecMaster calls made one-at-a-time in Phase 3
**Impact**: Slow processing for large option batches
**Location**: `SecMasterBridgeGuidepost`

**Current Flow**:
```
Option1 → SecMaster → Wait → Response
Option2 → SecMaster → Wait → Response
Option3 → SecMaster → Wait → Response
```

**Optimized Flow**:
```
[Option1, Option2, Option3] → Batch SecMaster → Single Response
```

**Implementation**:
```java
public List<SecMasterResponse> batchLookup(List<String> symbols) {
    // Build single UDP packet with multiple symbols
    byte[] request = buildBatchRequest(symbols);

    // Single network call
    byte[] response = udpClient.send(request);

    // Parse multiple responses
    return parseBatchResponse(response);
}
```

---

### 5. UUID Correlation Logging

**Issue**: UUID tracking exists but not consistently logged
**Impact**: Difficult to trace StateBean through all 5 phases
**Location**: `StateBeanTrackerGuidepost`

**Required Enhancement**:
```java
// Add MDC context for all log statements
private void processWithTracking(StateBean bean, String uuid) {
    try (MDC.MDCCloseable ignored = MDC.putCloseable("tickerUuid", uuid)) {
        log.info("Phase 1 start: symbol={}", bean.getSymbol());
        // ... processing
        log.info("Phase 5 complete: tickerId={}", result.getTickerId());
    }
}
```

---

### 6. Cache Miss Metrics

**Issue**: No visibility into cache hit/miss ratios
**Impact**: Cannot optimize cache strategy or detect issues
**Location**: `TickerCacheManagerGuidepost`

**Required Enhancement**:
```java
@Component
public class CacheMetrics {
    private final MeterRegistry meterRegistry;

    public void recordCacheHit(String securityType) {
        meterRegistry.counter("ticker.cache.hits", "type", securityType).increment();
    }

    public void recordCacheMiss(String securityType) {
        meterRegistry.counter("ticker.cache.misses", "type", securityType).increment();
    }
}
```

---

## P2: Parity Enhancements

### 7. ETF Classification Enhancement

**Issue**: ETFs sometimes misclassified as equities
**Impact**: Incorrect processing for ETF-specific logic
**Location**: `TickerClassifierGuidepost`

**Enhancement**:
```java
private SecurityType classifySymbol(String symbol, SecMasterData data) {
    // Check explicit ETF markers
    if (data.getSecuritySubType() != null &&
        data.getSecuritySubType().contains("ETF")) {
        return SecurityType.ETF;
    }

    // Check known ETF issuers
    if (KNOWN_ETF_ISSUERS.contains(data.getIssuer())) {
        return SecurityType.ETF;
    }

    // Fallback to equity
    return SecurityType.EQUITY;
}
```

---

### 8. Index Ticker Support

**Issue**: Index tickers (^SPX, ^VIX) not properly handled
**Impact**: Cannot resolve index-based tickers
**Location**: `TickerFinderGuidepost`

**Enhancement**:
```java
private boolean isIndexTicker(String symbol) {
    return symbol.startsWith("^") ||
           symbol.startsWith(".") ||
           KNOWN_INDEX_PREFIXES.stream().anyMatch(symbol::startsWith);
}

private Optional<ATDBTicker> resolveIndex(String symbol) {
    // Strip prefix for lookup
    String baseSymbol = symbol.replaceFirst("^[.^]", "");
    return findBySymbolAndType(baseSymbol, SecurityType.INDEX);
}
```

---

### 9. ADR/Ordinary Share Linking

**Issue**: ADRs not consistently linked to underlying ordinary shares
**Impact**: Position aggregation across share classes incomplete
**Location**: `TickerFactoryForIdmServicesGuidepost`

**Enhancement**:
```java
private void linkAdrToOrdinary(ATDBTicker adr) {
    if (adr.getAdrRatio() != null) {
        // Find corresponding ordinary share
        Optional<ATDBTicker> ordinary = findOrdinaryForAdr(adr);
        ordinary.ifPresent(ord -> {
            adr.setUnderlyingTickerId(ord.getTickerId());
            adr.setAdrToOrdinaryRatio(adr.getAdrRatio());
        });
    }
}
```

---

## P3: Nice-to-Have Improvements

### 10. Parallel Phase Processing

**Issue**: Phases 1, 2, 4, 5 run sequentially even though they don't require Phase 3
**Impact**: Suboptimal throughput for batches with no options/fallbacks

**Potential Optimization**:
```
Standard Flow:          Parallel Flow:
P1 → P2 → P3 → P4 → P5    P1 → P2 → P4 → P5  (no SecMaster needed)
                              ↘ P3 ↗          (only if needed)
```

---

### 11. Warm Cache on Startup

**Issue**: Cold cache after service restart causes slow first batch
**Impact**: First batch after deployment processes slowly

**Enhancement**:
```java
@EventListener(ApplicationReadyEvent.class)
public void warmCache() {
    // Load most frequently used tickers
    List<ATDBTicker> frequentTickers = repository.findMostFrequentlyAccessed(10000);
    frequentTickers.forEach(cache::put);
    log.info("Cache warmed with {} tickers", frequentTickers.size());
}
```

---

### 12. Async CDC Confirmation

**Issue**: No confirmation that CDC event was processed
**Impact**: Cannot guarantee cache consistency without polling

**Enhancement**:
```java
public CompletableFuture<Boolean> waitForCdcConfirmation(Integer tickerId, Duration timeout) {
    return CompletableFuture.supplyAsync(() -> {
        Instant deadline = Instant.now().plus(timeout);
        while (Instant.now().isBefore(deadline)) {
            if (cache.contains(tickerId)) {
                return true;
            }
            Thread.sleep(10);
        }
        return false;
    });
}
```

---

## Implementation Checklist

### Phase 1 Fixes (Current Sprint)
- [ ] P0-1: Option Expiration Date Handling
- [ ] P0-2: Missing Option Resolution Data
- [ ] P0-3: SecMaster Persistence Gap
- [ ] P1-4: Batch SecMaster Optimization

### Phase 2 Fixes (Next Sprint)
- [ ] P1-5: UUID Correlation Logging
- [ ] P1-6: Cache Miss Metrics
- [ ] P2-7: ETF Classification Enhancement
- [ ] P2-8: Index Ticker Support

### Backlog
- [ ] P2-9: ADR/Ordinary Share Linking
- [ ] P3-10: Parallel Phase Processing
- [ ] P3-11: Warm Cache on Startup
- [ ] P3-12: Async CDC Confirmation

---

## Testing Requirements

Each fix requires:
1. Unit test for the specific bug/feature
2. Integration test with mock SecMaster
3. Regression test suite execution
4. Performance benchmark (for P1-4 especially)

---

*Document derived from GUIDEPOST_BEHAVIOR_ADDITIONS_FOR_MERGE_IN_PARITY.md*
