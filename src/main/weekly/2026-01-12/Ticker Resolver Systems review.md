All the Atomic Adapter components for ticker

Legacy Adapter (Ticker Identification Container)

ATAdapterStations

Guidepost system


Direct lookup tables to and from the FSYM_ID:
* fds.sym_v1.sym_*    coverage, bbg, ticker_region, ticker_exchange, isin

TickerMapping.mapped.symbologyResolvedTicker -->  Join to sourceSymbol 
= Ticker.Name and also a join to ResolvedAlphaTheoryTickerName = BloombergCompositeTicker

Handling of non-Equity, per Guidepost

Handling of Black Scholes, expanding what is in Guidepost



------------



Ticker Resolution System: Complete Table Reference

1. FACTSET SYMBOLOGY TABLES (fds.sym_v1.*)

These are the direct lookup tables for FSYM_ID resolution:
Table: sym_coverage
Schema.Table: fds.sym_v1.sym_coverage
Key Fields: fsym_id (PK), currency, proper_name, fsym_primary_equity_id, fsym_primary_listing_id,
fref_security_type, fref_listing_exchange, fsym_regional_id, fsym_security_id, universe_type
Purpose: Master symbology coverage - links all FSYM_ID types
────────────────────────────────────────
Table: sym_bbg
Schema.Table: fds.sym_v1.sym_bbg
Key Fields: fsym_id, bbg_id, bbg_ticker, most_recent
Purpose: Bloomberg ↔ FSYM_ID mapping
────────────────────────────────────────
Table: sym_isin
Schema.Table: fds.sym_v1.sym_isin
Key Fields: fsym_id, isin
Purpose: ISIN ↔ FSYM_ID mapping
────────────────────────────────────────
Table: sym_ticker_region
Schema.Table: fds.sym_v1.sym_ticker_region
Key Fields: fsym_id, ticker_region
Purpose: Regional ticker symbols (e.g., "AAPL-US")
────────────────────────────────────────
Table: sym_ticker_region_hist
Schema.Table: fds.sym_v1.sym_ticker_region_hist
Key Fields: fsym_id, ticker_region, start_date, end_date
Purpose: Historical ticker region with date ranges
────────────────────────────────────────
Table: sym_ticker_exchange
Schema.Table: fds.sym_v1.sym_ticker_exchange
Key Fields: fsym_id, ticker_exchange
Purpose: Exchange-specific ticker symbols
  ---
2. TICKERMAPPING DATABASE TABLES

A. Source Data Layer
┌───────────────┬────────────────────────────────────┬────────────────────────────────────────┐
│     Table     │            Schema.Table            │                Purpose                 │
├───────────────┼────────────────────────────────────┼────────────────────────────────────────┤
│ SymbologyData │ TickerMapping.source.SymbologyData │ Raw client symbology before resolution │
└───────────────┴────────────────────────────────────┴────────────────────────────────────────┘
Key Fields:
- symbologyHashCode - Hash linking to resolution
- symbol, bloombergTicker, description, currencyCode
- cusip, sedol, isin
- Underlying versions of all identifiers
- Source security types (1-4)

B. Mapped/Resolved Layer
Table: SymbologyResolvedTicker
Schema.Table: TickerMapping.mapped.SymbologyResolvedTicker
Purpose: Resolved ticker after validation
Key Fields:
- Resolution Results:
    - resolvedAlphaTheoryTickerName - Final AT ticker name
    - resolvedUnderlyingAlphaTheoryTickerName - For derivatives
    - resolvedTickerRegion - e.g., "AAPL-US"
    - resolvedFsymIdRegional, resolvedFsymIdListing
    - resolvedFigi, compositeFigi
    - resolvedBbgTickerComposite
- Linking Fields:
    - symbologyHashCode - Links to source
    - externalAppId, externalAppSettingsId, externalAppHistoryId
- Enterprise Classification:
    - enterpriseSecurityType - Modern security classification
    - tickerNameQualifiers - JSON array of qualifiers

  ---
3. PORTFOLIOMAPPING DATABASE TABLES
   Table: SymbologyResolvedTicker
   Schema.Table: PortfolioMapping.mapped.SymbologyResolvedTicker
   Purpose: Same schema as TickerMapping but for portfolio context
  ---
4. LEGACY ADAPTER TABLES (StateBeans)

A. Ticker Identification Container (Stage 1)
┌────────────────────────────────────────┬───────────────────────────────────────────────────┐
│                 Table                  │                      Purpose                      │
├────────────────────────────────────────┼───────────────────────────────────────────────────┤
│ StateBeanTickerIdentificationContainer │ Captures raw ticker data from portfolio positions │
└────────────────────────────────────────┴───────────────────────────────────────────────────┘
Key Fields:
- adapterUuid (PK) - UUID linking to source row
- externalAppHistoryID - Adapter run identifier
- symbologyHashCode - Hash from Atomic Adapter validation
- 40+ identification fields: Bloomberg Ticker, ISIN, SEDOL, CUSIP, FIGI, Currency, Country, Security Type, Underlying identifiers

B. Resolved Ticker Details (Stage 2)
┌────────────────────────────────┬──────────────────────────────────────────────────┐
│             Table              │                     Purpose                      │
├────────────────────────────────┼──────────────────────────────────────────────────┤
│ StateBeanResolvedTickerDetails │ Stores resolved ticker names and resolution path │
└────────────────────────────────┴──────────────────────────────────────────────────┘
Key Fields:
- resolvedAlphaTheoryTickerName - From Mode 1 (Atomic)
- legacyTickerName - Final resolved name
- legacyTickerNameType - Resolution indicator:
    - US_CURRENCY_MATCH (auto-approved)
    - ROTW_SEDOL_CURRENCY_MISMATCH (needs review)
    - MATCHED_FROM_RESOLVED_TO_VALIDATED_HOLISTIC (Mode 2)

  ---
5. PROVENANCE TABLES (6-Step Audit Trail)
   Table: StateBeanTickerResolutionEnrichmentSources
   Step: 1
   Records per UUID: 1
   Purpose: Enrichment attempts (SelectFeed, FIGI, Symbology validation) with full JSON responses
   ────────────────────────────────────────
   Table: StateBeanTickerResolutionIntermediateValues
   Step: 2
   Records per UUID: 1
   Purpose: Workflow decisions, transformations, predicted actions
   ────────────────────────────────────────
   Table: StateBeanTickerResolutionCandidates
   Step: 2.5
   Records per UUID: 5-20
   Purpose: Ticker candidates (Legacy Cascade only)
   ────────────────────────────────────────
   Table: StateBeanTickerResolutionAttribution
   Step: 3-4
   Records per UUID: 1-2
   Purpose: BEFORE/AFTER ticker snapshots
   ────────────────────────────────────────
   Table: StateBeanTickerResolutionOtherIdentifierMatch
   Step: 4
   Records per UUID: 0-1
   Purpose: OtherIdentifier match results
   ────────────────────────────────────────
   Table: StateBeanTickerResolutionPredictedChanges
   Step: 5
   Records per UUID: N
   Purpose: Field-by-field change predictions
   ────────────────────────────────────────
   Table: StateBeanTickerResolutionSelectfeedSummary
   Step: -
   Records per UUID: 0-1
   Purpose: SelectFeed query summary (if attempted)
  ---
6. GUIDEPOST SYSTEM (UUID-Based Maps, Not Tables)

The Guidepost system uses in-memory UUID maps rather than database tables:

// Core UUID Tracking Maps
Map<String, ATAdapterStateBean> uuidToStateBean;
Map<String, TickerIdentificationContainer> uuidToTic;
Map<String, ATDBTicker> uuidToResolvedTicker;
Map<String, OptionTickerResolutionGuidepost> uuidToOptionResolution;

// Outcome Tracking Sets
Set<String> uuidsNeedingSave;           // New tickers
Set<String> uuidsNeedingUpdate;         // Existing tickers with changes
Set<String> uuidsNeedingSecMasterResolution; // OPTIONS only
Set<String> uuidsInvalid;               // Failed resolution

Guidepost uses CDC (Change Data Capture): Writes to the Ticker table, and Debezium CDC automatically syncs to TickerCache.

  ---
7. TICKER TABLE (Core SecMaster)
   ┌────────┬─────────────────────────────────┐
   │ Table  │             Purpose             │
   ├────────┼─────────────────────────────────┤
   │ Ticker │ Core ticker entity in SecMaster │
   └────────┴─────────────────────────────────┘
   Key Fields:
- tickerId (PK), name, atTicker
- bloombergTicker, isin, sedol, cusip, figi
- description, currencyCode, exchangeId, securityTypeId
- underlierId (for options/derivatives)
- GICS classification fields
- active, deleted, version

  ---
8. KEY RELATIONSHIPS

                          Your Question Components:

┌─────────────────────────────────────────────────────────────────┐
│ TickerMapping.mapped.SymbologyResolvedTicker                    │
│   → sourceSymbol = Ticker.Name                                  │
│   → resolvedAlphaTheoryTickerName = BloombergCompositeTicker    │
│   → symbologyHashCode → Links to StateBeanTIC                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ FactSet sym_v1 tables                                           │
│   → fsym_id = Universal key                                     │
│   → sym_bbg.bbg_ticker → Bloomberg ticker                       │
│   → sym_isin.isin → ISIN                                        │
│   → sym_ticker_region.ticker_region → Regional ticker           │
│   → sym_coverage.currency → Currency                            │
└─────────────────────────────────────────────────────────────────┘

  ---
9. NON-EQUITY HANDLING (Per Guidepost)
   TickerTreatmentType: primaryExchangeInstrument
   Routing Path: ATDBTickerRowMapper lookups
   TickerMaster Required: NO
   ────────────────────────────────────────
   TickerTreatmentType: primaryOtcInstrument
   Routing Path: OTC lookups (name→CUSIP→ISIN→SEDOL)
   TickerMaster Required: NO
   ────────────────────────────────────────
   TickerTreatmentType: option (Black-Scholes)
   Routing Path: ATDBTicker + CDC
   TickerMaster Required: NO
   ────────────────────────────────────────
   TickerTreatmentType: option (non-Black-Scholes)
   Routing Path: TickerMaster resolution
   TickerMaster Required: YES
   ────────────────────────────────────────
   TickerTreatmentType: subordinatedOtcUnderExchangeInstrument
   Routing Path: ATDBTicker + underlying
   TickerMaster Required: NO
   ────────────────────────────────────────
   TickerTreatmentType: subordinatedOtcUnderOtcInstrument
   Routing Path: OTC lookups + underlying
   TickerMaster Required: NO
   ────────────────────────────────────────
   TickerTreatmentType: subordinatedExchangeUnderExchangeInstrument
   Routing Path: ATDBTicker + underlying
   TickerMaster Required: NO
  ---
10. BLACK-SCHOLES ELIGIBILITY

Options are Black-Scholes eligible (no TickerMaster needed) if:
1. Has parseableEquityOption
2. Has underlyingBloombergTicker
3. Has underlyingBloombergExchangeCode
4. Exchange code is NOT US or Canadian (i.e., is foreign)

Foreign options are handled via local Black-Scholes calculation, avoiding the TickerMaster dependency.

  ---
Would you like me to dive deeper into any specific table or relationship?





● Now I have a complete picture. Let me provide you with a comprehensive high-level overview of the AtomicAdapterWorkflowServices pipeline.

  ---
AtomicAdapterWorkflowServices: Complete Pipeline Overview

High-Level Architecture

┌─────────────────────────────────────────────────────────────────────────────────┐
│                        AtomicAdapterWorkflowServices                            │
│                         (Master Orchestrator - 397 lines)                       │
└─────────────────────────────────────────────────────────────────────────────────┘
│
┌────────────────────────────────┼────────────────────────────────┐
│                                │                                │
▼                                ▼                                ▼
┌─────────────┐              ┌─────────────────┐              ┌─────────────────┐
│  PHASE 1    │              │    PHASE 2      │              │    PHASE 3      │
│  EXTRACTION │     ───►     │   FIGI/FACTSET  │     ───►     │ TICKER RESOLVER │
│             │              │   REQUEST+EXEC  │              │    SERVICES     │
└─────────────┘              └─────────────────┘              └─────────────────┘

  ---
PHASE 1: Data Extraction (Get Symbology to Query)

Entry Point

StreamlinedClientDataFeedExtractionServices.processSourceDataRowMapList()
// OR for legacy StateBeans:
StreamlinedClientDataFeedExtractionServices_forStateBean.processStateBeansForTransformedRawAndSymbologyRecords()

What It Does

1. Parse client portfolio files → Extract raw data rows
2. Transform to normalized format → TransformedClientDataRowInstance
3. Extract unique symbology → SymbologyDataInstance (deduplicated by symbologyHashCode)

Output Tables
┌──────────────────────────────────┬────────────────────────────────┬───────────────────────────────┐
│              Table               │            Database            │            Purpose            │
├──────────────────────────────────┼────────────────────────────────┼───────────────────────────────┤
│ TransformedClientDataRowInstance │ Portfolio/TickerMapping.source │ Normalized exposure data      │
├──────────────────────────────────┼────────────────────────────────┼───────────────────────────────┤
│ RawClientDataRowMapInstance      │ Portfolio/TickerMapping.source │ Raw JSON audit trail          │
├──────────────────────────────────┼────────────────────────────────┼───────────────────────────────┤
│ SymbologyDataInstance            │ Portfolio/TickerMapping.source │ Unique symbology per hashcode │
└──────────────────────────────────┴────────────────────────────────┴───────────────────────────────┘
Key Fields Extracted (Per SymbologyDataInstance)

- symbol, bloombergTicker, description, currencyCode
- isin, sedol, cusip
- underlyingBloombergTicker, underlyingIsin, underlyingSedol, underlyingCusip
- sourceSecurityType (1-4 fields)
- symbologyHashCode - THE PRIMARY KEY FOR ALL DOWNSTREAM JOINS

  ---
PHASE 2: FIGI & FactSet Extraction + Execution

Sub-Phase 2A: Symbology Extraction Service

OpenFigiFactSetSymbologyExtractionServices.processSymbologyRecordListForIdentifierRequest(
externalAppSettingsId, externalAppHistoryId, referenceDate, passNumber, symbologyHashCodesToIsolate
);

What It Does:
1. Read SymbologyDataInstance records from Phase 1
2. Transform to FIGI identifier request format (multiple requests per symbology record)
3. Generate FactSet-ready entries for secondary validation
4. Persist requests to staging tables

Output Tables:
┌────────────────────────────────────────┬─────────────────────────────────────────────────────┐
│                 Table                  │                       Purpose                       │
├────────────────────────────────────────┼─────────────────────────────────────────────────────┤
│ ClientOpenFigiRequest                  │ One record per unique identifier to query OpenFIGI  │
├────────────────────────────────────────┼─────────────────────────────────────────────────────┤
│ ClientSymbologyRecordFactSetReadyEntry │ One record per symbology ready for FactSet matching │
└────────────────────────────────────────┴─────────────────────────────────────────────────────┘
Sub-Phase 2B: FIGI Batch Assignment & API Execution

processFigiBatchAssignmentAndRequestResolution(...)

Step-by-Step:
Step: 1
Action: Clear previous batch records
Output: Clean slate
────────────────────────────────────────
Step: 2
Action: ClientSourceDataFileInstanceOpenFigiRequestManager.batchAssignmentPreProcessOpenFigiRequestsForClientFile()
Output: FigiRequestBatchSequenceRecord (batch numbers, request ordering)
────────────────────────────────────────
Step: 3
Action: Determine job type (TICKER vs DAILY_ADAPTER)
Output: Token pool selection
────────────────────────────────────────
Step: 4
Action: OpenFigiTokenManagementServices.getAvailableTokenManagersForTheEnvironment()
Output: List<ATDBOpenFigiTokenManager>
────────────────────────────────────────
Step: 5
Action: Mark tokens in-use
Output: Concurrency lock
────────────────────────────────────────
Step: 6
Action: Extract API key strings
Output: Set<String> figiAPITokenSet
────────────────────────────────────────
Step: 7
Action: ClientSourceDataFileInstanceOpenFigiRequestManager.makeBatchRequestsForOpenFigiRequestsFromClientFile()
Output: Actual OpenFIGI API calls
────────────────────────────────────────
Step: 8
Action: Release tokens
Output: Cleanup
FIGI API Execution Output Tables:
┌─────────────────────────────────┬─────────────────────────────────────────┐
│              Table              │                 Purpose                 │
├─────────────────────────────────┼─────────────────────────────────────────┤
│ OpenFigiRequest                 │ vendors.figi - Request records          │
├─────────────────────────────────┼─────────────────────────────────────────┤
│ OpenFigiResponsePerHashCode     │ vendors.figi - Raw API responses        │
├─────────────────────────────────┼─────────────────────────────────────────┤
│ FigiResponseInterpretationEntry │ vendors.figi - INTERPRETED FIGI results │
└─────────────────────────────────┴─────────────────────────────────────────┘
  ---
PHASE 2C: FIGI & FactSet INTERPRETATION Services

This is where raw responses become scored, interpreted results:

FIGI Interpretation

The FigiResponseInterpretationEntry table stores:
- figi - The FIGI code
- compositeFigi - The composite FIGI
- securityType, securityType2 - OpenFIGI security classifications
- exchCode - Exchange code from FIGI
- ticker - Bloomberg ticker from FIGI
- name - Security name from FIGI
- Scoring fields for ranking multiple results

FactSet Extraction (via FIGI → FactSet join)

The ATDBFactSetTickerRegionBloomberg entry captures:
- fsym_id, fsym_regional_id, fsym_primary_listing_id
- bbg_ticker - Bloomberg ticker from FactSet
- currency, fref_listing_exchange
- Delisting information from FactSet

  ---
PHASE 3: TickerResolverServices (GRADING ALL RESULTS)

Entry Point

TickerResolverServices.processAllSymbologyHashCodesPerExternalAppHistoryID(
externalAppHistoryId, externalAppSettingsId
);

What It Does (Per SymbologyHashCode)

FOR EACH SymbologyDataInstance:
│
├─► 1. Initialize supporting data via TickerResolverInitializationServices
│
├─► 2. Define Enterprise Security Type (from source fields)
│       → defineEnterpriseSecurityTypeEnumSetPair()
│       → Returns Pair<EnterpriseSecurityTypeEnum, Set<EnterpriseSecurityTypeEnum>>
│
├─► 3. Process Security Type Qualifiers
│       → processSecurityTypeQualifiers()
│       → Determines tickerNameQualifiers (ADR, CFD, SWAP flags)
│
├─► 4. Grade FIGI Results (for main holding)
│       → Score and select best FIGI from multiple responses
│       → Resolve to alphaTheoryTickerName
│
├─► 5. Grade FIGI Results (for underlying, if derivative)
│       → Score and select best underlying FIGI
│       → Resolve to underlyingAlphaTheoryTickerName
│
├─► 6. Join FactSet validation data
│       → Currency, exchange, delisting validation
│       → fsym_id resolution
│
└─► 7. Output: ValidatedSymbologyTickerResolverEntry

Output Table
Table: SymbologyResolvedTicker
Database: PortfolioMapping.mapped / TickerMapping.mapped
Purpose: Final graded ticker resolution
  ---
TickerResolverInitializationServices: The Many Layers

This service pre-loads and caches all supporting data before per-record processing. Here are the 15+ cached data sets:

Cache Architecture

┌────────────────────────────────────────────────────────────────────────────────┐
│                    TickerResolverInitializationServices                        │
│                           (Cache Manager - 993 lines)                          │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│   initializeSupportingFactSetDataFromExternalAppHistory()                     │
│   ├─► Query: ATDBFigiRequestInterpretationPerExternalAppHistory (Standard)    │
│   ├─► Query: ATDBFigiRequestInterpretationPerExternalAppHistory (Composite)   │
│   ├─► Query: ATDBFigiRequestInterpretationPerExternalAppHistory (Underlying)  │
│   │                                                                            │
│   │   Extract distinct FIGIs from interpretations                              │
│   │            ▼                                                               │
│   ├─► Query: ATDBFdsBloombergCoverage (FIGI → FactSet sym_bbg + sym_coverage) │
│   │                                                                            │
│   │   Extract distinct fsym_ids                                                │
│   │            ▼                                                               │
│   ├─► Query: ATDBFdsResolvedHistoricalSymbol (sym_ticker_region_hist)         │
│   │                                                                            │
│   │   Extract distinct fsym_security_ids                                       │
│   │            ▼                                                               │
│   ├─► Query: ATDBFdsEntityHistoryLatest (entity history)                      │
│   │                                                                            │
│   │   Extract distinct factset_entity_ids                                      │
│   │            ▼                                                               │
│   ├─► Query: ATDBFdsEntitySectorDetails (GICS sector info)                    │
│   │                                                                            │
│   │   Extract distinct fref_listing_exchange codes                             │
│   │            ▼                                                               │
│   ├─► Query: ATDBFdsListingExchange (exchange reference)                      │
│   │                                                                            │
│   │   Extract distinct iso_country codes                                       │
│   │            ▼                                                               │
│   ├─► Query: ATDBFdsListingCountry (country reference)                        │
│   │                                                                            │
│   │   Extract distinct region_codes                                            │
│   │            ▼                                                               │
│   └─► Query: ATDBFdsListingRegion (region reference)                          │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘

The 15+ Cached Data Sets
#: 1
Cache Variable: figiRequestInterpretationEntries
Source Query/Table: FIGI interpretation (Standard + Composite)
Purpose: FIGI results for main holding
────────────────────────────────────────
#: 2
Cache Variable: figiRequestInterpretationUnderlyingEntries
Source Query/Table: FIGI interpretation (Underlying)
Purpose: FIGI results for underlying
────────────────────────────────────────
#: 3
Cache Variable: fdsBloombergCoverageEntries
Source Query/Table: fds.sym_v1.sym_bbg + sym_coverage
Purpose: FIGI → FactSet mapping
────────────────────────────────────────
#: 4
Cache Variable: fdsResolvedTickerHistoryEntries
Source Query/Table: fds.sym_v1.sym_ticker_region_hist
Purpose: Historical ticker symbols
────────────────────────────────────────
#: 5
Cache Variable: fdsEntityHistoryLatestEntries
Source Query/Table: Entity history
Purpose: Entity info per fsym_id
────────────────────────────────────────
#: 6
Cache Variable: fdsEntitySectorDetailsEntries
Source Query/Table: Entity sector details
Purpose: GICS classification
────────────────────────────────────────
#: 7
Cache Variable: fdsListingExchangeEntries
Source Query/Table: Listing exchange reference
Purpose: Exchange info
────────────────────────────────────────
#: 8
Cache Variable: fdsListingCountryEntries
Source Query/Table: Listing country reference
Purpose: Country info
────────────────────────────────────────
#: 9
Cache Variable: fdsListingRegionEntries
Source Query/Table: Listing region reference
Purpose: Region info
────────────────────────────────────────
#: 10
Cache Variable: clientSymbologyFactSetTickerFdsBloombergValidatedEntries
Source Query/Table: FactSet-validated entries
Purpose: Bloomberg/FactSet cross-validation
────────────────────────────────────────
#: 11
Cache Variable: clientSymbologyExternalAppHistoryRequestResultsView
Source Query/Table: Request results view
Purpose: FIGI request results
────────────────────────────────────────
#: 12
Cache Variable: clientSymbologyExternalAppHistoryRequestResultsViewUnderlying
Source Query/Table: Underlying request results
Purpose: Underlying FIGI results
────────────────────────────────────────
#: 13
Cache Variable: validateFigiWithFactSetEntries
Source Query/Table: FIGI ↔ FactSet validation
Purpose: Cross-validation
────────────────────────────────────────
#: 14
Cache Variable: validateSourceIsinWithFactSetEntries
Source Query/Table: Source ISIN ↔ FactSet
Purpose: ISIN validation
────────────────────────────────────────
#: 15
Cache Variable: validateSourceIsinWithFactSetEntries_undly
Source Query/Table: Underlying ISIN ↔ FactSet
Purpose: Underlying ISIN validation
Cache Population Flow (Cascading Queries)

FIGI Interpretations
│
│ Extract distinct FIGIs
▼
FDS Bloomberg Coverage (sym_bbg + sym_coverage)
│
├─► Extract fsym_ids ──────────► FDS Resolved Ticker History
│
├─► Extract fsym_security_ids ─► FDS Entity History Latest
│                                       │
│                                       ▼
│                               FDS Entity Sector Details
│
└─► Extract fref_listing_exchange ──► FDS Listing Exchange
│
▼
FDS Listing Country
│
▼
FDS Listing Region

  ---
Summary: The Complete Pipeline
Phase: 1
Service: StreamlinedClientDataFeedExtractionServices
Input: Client file
Output: SymbologyDataInstance
Key Operation: Extract symbology to query
────────────────────────────────────────
Phase: 2A
Service: OpenFigiFactSetSymbologyExtractionServices
Input: SymbologyDataInstance
Output: ClientOpenFigiRequest
Key Operation: Figure out FIGI requests
────────────────────────────────────────
Phase: 2B
Service: ClientSourceDataFileInstanceOpenFigiRequestManager
Input: ClientOpenFigiRequest
Output: FigiResponseInterpretationEntry
Key Operation: Execute FIGI API calls
────────────────────────────────────────
Phase: 2C
Service: (Implicit via rowmappers)
Input: FIGI responses
Output: Interpretation entries
Key Operation: FIGI interpretation
────────────────────────────────────────
Phase: 3-Init
Service: TickerResolverInitializationServices
Input: externalAppHistoryId
Output: 15+ caches
Key Operation: Pre-load FactSet reference data
────────────────────────────────────────
Phase: 3
Service: TickerResolverServices
Input: SymbologyDataInstance + caches
Output: ValidatedSymbologyTickerResolverEntry
Key Operation: Grade all results → Final ticker
  ---
Would you like me to dive deeper into any specific phase, the scoring/grading algorithms, or the FactSet table joins?






