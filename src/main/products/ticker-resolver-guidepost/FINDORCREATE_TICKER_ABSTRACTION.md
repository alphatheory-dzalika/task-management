# FindOrCreateTicker Abstraction Plan

**Document Type**: Future Architecture Planning
**Source**: GUIDEPOST_ABSTRACTION_FINDORCREATE_TICKER_PLAN.md
**Status**: Planning Phase

---

## Vision

Create a unified **FindOrCreateTicker** service that abstracts the entire ticker resolution pipeline into a clean, reusable interface. This enables:

1. **Single Entry Point**: One service for all ticker resolution needs
2. **Strategy Pattern**: Pluggable resolution strategies (Bloomberg, FactSet, FIGI, etc.)
3. **Full Cycle Coverage**: Ticker → Asset → FundAsset creation
4. **Testability**: Mock-friendly interfaces for unit testing

---

## Current State vs. Target State

### Current State (Guidepost)
```
┌─────────────────────────────────────────────────────────┐
│                  CURRENT ARCHITECTURE                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Parser ──► StateBean ──► TickerFactoryGuidepost ──►    │
│                               │                          │
│                               ├──► TickerFinder          │
│                               ├──► TickerCreator         │
│                               ├──► OptionProcessor       │
│                               ├──► SecMasterBridge       │
│                               └──► HistoryProcessor      │
│                                                          │
│  [9 classes, tightly coupled, 6,096 lines]              │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Target State (FindOrCreateTicker)
```
┌─────────────────────────────────────────────────────────┐
│                  TARGET ARCHITECTURE                     │
├─────────────────────────────────────────────────────────┤
│                                                          │
│           ┌────────────────────────────┐                │
│           │   FindOrCreateTicker       │                │
│           │      (Interface)           │                │
│           └────────────┬───────────────┘                │
│                        │                                 │
│    ┌───────────────────┼───────────────────┐            │
│    ▼                   ▼                   ▼            │
│  Bloomberg          FactSet             FIGI            │
│  Strategy           Strategy           Strategy         │
│                                                          │
│  [Clean interfaces, pluggable strategies]               │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## Three-Phase Implementation

### Phase A: Ticker Resolution

**Input**: Raw symbol string
**Output**: Resolved SecMasterTicker

```java
public interface TickerResolutionStrategy {

    /**
     * Resolve a ticker symbol to a SecMasterTicker.
     *
     * @param symbol Raw ticker symbol (e.g., "AAPL", "AAPL 250117C00150000")
     * @param context Resolution context (security type hint, exchange, etc.)
     * @return Resolved ticker or empty if not found
     */
    Optional<SecMasterTicker> resolve(String symbol, ResolutionContext context);

    /**
     * Batch resolution for performance.
     */
    List<TickerResolutionResult> resolveBatch(List<String> symbols, ResolutionContext context);

    /**
     * Indicates which symbol formats this strategy supports.
     */
    Set<SymbolFormat> supportedFormats();
}
```

**Implementations**:
- `BloombergTickerResolutionStrategy` - Bloomberg symbology
- `FactSetTickerResolutionStrategy` - FactSet entity IDs
- `FigiResolutionStrategy` - OpenFIGI lookups
- `CompositeResolutionStrategy` - Chain of responsibility

### Phase B: Fund Resolution

**Input**: SecMasterTicker + Department context
**Output**: Fund assignment for ticker

```java
public interface FundResolutionStrategy {

    /**
     * Determine which fund(s) should hold this ticker.
     *
     * @param ticker Resolved ticker
     * @param department Client department context
     * @return Fund assignment(s)
     */
    List<FundAssignment> resolveFunds(SecMasterTicker ticker, Department department);
}
```

**Implementations**:
- `DefaultFundResolutionStrategy` - Standard fund mapping rules
- `OptionUnderlierFundStrategy` - Options follow underlier's fund
- `SectorBasedFundStrategy` - Assign by sector/industry

### Phase C: Aggregation Services

**Input**: Resolved ticker + Fund assignment
**Output**: Complete Asset + FundAsset creation

```java
public interface AggregationService {

    /**
     * Create or update Asset and FundAsset records.
     *
     * @param ticker Resolved ticker
     * @param fundAssignment Fund assignment from Phase B
     * @return Aggregation result with created/updated entities
     */
    AggregationResult aggregate(SecMasterTicker ticker, FundAssignment fundAssignment);
}
```

---

## Unified Interface

```java
/**
 * FindOrCreateTicker - The unified entry point for all ticker resolution.
 *
 * This service encapsulates the entire flow from raw symbol to complete
 * Asset/FundAsset creation.
 */
public interface FindOrCreateTicker {

    /**
     * Full cycle: Symbol → Ticker → Asset → FundAsset
     */
    TickerCreationResult findOrCreate(
        String symbol,
        Department department,
        FindOrCreateOptions options
    );

    /**
     * Batch processing with progress tracking
     */
    BatchTickerCreationResult findOrCreateBatch(
        List<TickerRequest> requests,
        BatchOptions options,
        ProgressCallback callback
    );

    /**
     * Resolution only (no asset creation)
     */
    Optional<SecMasterTicker> resolveOnly(String symbol, ResolutionContext context);
}
```

### Options and Configuration

```java
public class FindOrCreateOptions {
    // Resolution preferences
    private ResolutionStrategy preferredStrategy;  // BLOOMBERG, FACTSET, FIGI, AUTO
    private boolean allowFallback;                 // Try other strategies if preferred fails

    // Asset creation preferences
    private boolean createAsset;                   // Create Asset if not exists
    private boolean createFundAsset;               // Create FundAsset linkage

    // CDC preferences
    private boolean waitForCdc;                    // Wait for CDC propagation
    private Duration cdcTimeout;                   // Max wait time

    // Tracking
    private String correlationId;                  // For distributed tracing
}
```

---

## MVP: Simple Bloomberg Lookup Service

The minimum viable product focuses on Bloomberg symbology only:

```java
/**
 * MVP Implementation - Bloomberg only
 */
public class SimpleBloombergLookupService implements FindOrCreateTicker {

    private final BloombergTickerResolutionStrategy bloombergStrategy;
    private final TickerCreatorGuidepost tickerCreator;
    private final AssetCreationService assetService;

    @Override
    public TickerCreationResult findOrCreate(
            String symbol,
            Department department,
            FindOrCreateOptions options) {

        // Phase A: Resolution
        Optional<SecMasterTicker> existing = bloombergStrategy.resolve(symbol, context);
        if (existing.isPresent()) {
            return TickerCreationResult.existingTicker(existing.get());
        }

        // Phase A (continued): Create new ticker
        ATDBTicker newTicker = tickerCreator.create(symbol);
        SecMasterTicker resolved = transform(newTicker);

        // Phase B & C: Asset creation if requested
        if (options.isCreateAsset()) {
            Asset asset = assetService.findOrCreateAsset(resolved, department);
            if (options.isCreateFundAsset()) {
                FundAsset fundAsset = assetService.linkToFund(asset, department);
                return TickerCreationResult.created(resolved, asset, fundAsset);
            }
            return TickerCreationResult.created(resolved, asset, null);
        }

        return TickerCreationResult.createdTickerOnly(resolved);
    }
}
```

---

## Migration Path

### Step 1: Extract Interface
- Define `FindOrCreateTicker` interface
- Existing Guidepost code continues unchanged
- New code uses interface

### Step 2: Implement Bloomberg MVP
- `SimpleBloombergLookupService` implements interface
- Delegates to existing Guidepost classes internally
- Provides clean API for new callers

### Step 3: Add FactSet Strategy
- `FactSetTickerResolutionStrategy` implementation
- Integrate with existing FactSet services
- Add to composite resolution chain

### Step 4: Add FIGI Strategy
- `FigiResolutionStrategy` implementation
- OpenFIGI API integration
- Universal symbol resolution

### Step 5: Decouple from Guidepost
- Gradually replace Guidepost internals
- Maintain interface stability
- Complete when all strategies migrated

---

## Benefits of Abstraction

| Aspect | Current (Guidepost) | Target (FindOrCreate) |
|--------|---------------------|----------------------|
| Entry Points | Multiple classes | Single interface |
| Testing | Requires full setup | Mock strategies |
| New Symbology | Major code changes | Add strategy |
| Maintenance | 6,096 lines to understand | Strategy isolation |
| Reusability | OpenAdapter specific | Cross-project |

---

## Integration with MCP

The `ticker-resolution-mcp` would wrap `FindOrCreateTicker`:

```java
@Tool("resolve_ticker")
public TickerResolutionMcpResult resolveTicker(
        @Param("symbol") String symbol,
        @Param("strategy") String strategy,
        @Param("create_if_missing") boolean createIfMissing) {

    FindOrCreateOptions options = FindOrCreateOptions.builder()
        .preferredStrategy(ResolutionStrategy.valueOf(strategy))
        .createAsset(createIfMissing)
        .build();

    return findOrCreateTicker.findOrCreate(symbol, currentDepartment(), options);
}
```

---

*Document derived from GUIDEPOST_ABSTRACTION_FINDORCREATE_TICKER_PLAN.md*
