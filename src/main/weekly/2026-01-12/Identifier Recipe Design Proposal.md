
  ---
Identifier Recipe System: Design Proposal

1. THE RECIPE CONCEPT

A recipe answers: "Given THIS trusted identifier, what can I retrieve?"

┌─────────────────────────────────────────────────────────────────────────────┐
│                         IDENTIFIER RECIPE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  INPUT:    Bloomberg Ticker (e.g., "AAPL US")                               │
│  TRUST:    HIGH (client-provided, validated)                                │
│                                                                             │
│  OUTPUT:   fsym_id, fsym_regional_id, fsym_primary_listing_id               │
│            bbg_id (FIGI), currency, fref_listing_exchange                   │
│            ticker_region (e.g., "AAPL-US"), ISIN, SEDOL, CUSIP              │
│            entity_id, entity_proper_name, GICS classification               │
│            active_flag, delisting_date, historical ticker changes           │
└─────────────────────────────────────────────────────────────────────────────┘

  ---
2. AVAILABLE FACTSET LOOKUP PATHS (The Recipes)

RECIPE 1: Bloomberg Ticker → Everything

-- Step 1: Bloomberg Ticker → fsym_id + bbg_id
SELECT fsym_id, bbg_id, bbg_ticker, most_recent
FROM fds.sym_v1.sym_bbg
WHERE bbg_ticker = REPLACE(:bloombergTicker, ' EQUITY', '')
AND most_recent = 1

-- Step 2: fsym_id → Full Coverage
SELECT fsym_id, currency, proper_name, fsym_primary_equity_id,
fsym_primary_listing_id, fsym_regional_id, fsym_security_id,
active_flag, fref_security_type, fref_listing_exchange
FROM fds.sym_v1.sym_coverage
WHERE fsym_id = :fsym_id

-- Step 3: fsym_regional_id → Ticker Region (AAPL-US format)
SELECT fsym_id, ticker_region, start_date, end_date
FROM fds.sym_v1.sym_ticker_region
WHERE fsym_id = :fsym_regional_id

-- Step 4: fsym_id → ISIN
SELECT fsym_id, isin
FROM fds.sym_v1.sym_isin
WHERE fsym_id = :fsym_id

-- Step 5: fsym_security_id → Entity + GICS
SELECT factset_entity_id, fsym_id, iso_country, entity_proper_name,
entity_type, start_date, end_date
FROM fds.ent_v1.ent_entity_coverage
WHERE fsym_id = :fsym_security_id

RECIPE 2: ISIN → Everything

-- Step 1: ISIN → fsym_id
SELECT fsym_id, isin
FROM fds.sym_v1.sym_isin
WHERE isin = :isin

-- Then follow Recipe 1 from fsym_id

RECIPE 3: FIGI (bbg_id) → Everything

-- Step 1: FIGI → fsym_id + bbg_ticker
SELECT fsym_id, bbg_id, bbg_ticker, most_recent
FROM fds.sym_v1.sym_bbg
WHERE bbg_id = :bbg_id

-- Then follow Recipe 1 from fsym_id

RECIPE 4: SEDOL → Everything

-- Step 1: SEDOL → fsym_id (via sym_sedol if available, or via sym_coverage)
-- Note: FactSet stores SEDOL in sym_sedol table (if available in your dataset)

RECIPE 5: CUSIP → Everything

-- Step 1: CUSIP → fsym_id (via sym_cusip if available)

  ---
3. UNIFIED RECIPE OUTPUT: FactSetIdentifierResolutionResult

Proposed Entity Structure

/**
* FactSetIdentifierResolutionResult
*
* Unified output from any identifier recipe execution.
* Stores the complete resolution result for SecMaster access.
  */
  public class FactSetIdentifierResolutionResult implements ATDBQueryObject {

  // ==================== Input Tracking ====================
  private Long resolutionResultId;           // PK - auto-generated
  private String inputIdentifierType;        // BLOOMBERG_TICKER, ISIN, FIGI, SEDOL, CUSIP
  private String inputIdentifierValue;       // The actual input value
  private Integer inputIdentifierHashCode;   // Hash for quick lookups
  private Timestamp resolutionTimestamp;     // When this was resolved

  // ==================== Core FactSet IDs ====================
  private String fsym_id;                    // Primary fsym_id found
  private String fsym_regional_id;           // Regional ID (ends with -R)
  private String fsym_primary_listing_id;    // Listing ID (ends with -L)
  private String fsym_security_id;           // Security ID (ends with -S)
  private String fsym_primary_equity_id;     // Primary equity ID

  // ==================== Bloomberg Data ====================
  private String bbg_id;                     // FIGI code
  private String bbg_ticker;                 // Bloomberg ticker (e.g., "AAPL US")
  private Integer bbg_most_recent;           // 1 = current, 0 = historical

  // ==================== Ticker Symbols ====================
  private String ticker_region;              // Regional ticker (e.g., "AAPL-US")
  private String ticker_exchange;            // Exchange ticker (e.g., "AAPL-NAS")
  private Date ticker_region_start_date;     // When this ticker became active
  private Date ticker_region_end_date;       // When this ticker was replaced (null = current)

  // ==================== Standard Identifiers ====================
  private String isin;                       // ISIN code
  private String sedol;                      // SEDOL code
  private String cusip;                      // CUSIP code

  // ==================== Security Details ====================
  private String currency;                   // Trading currency (ISO 4217)
  private String proper_name;                // Security proper name
  private String fref_security_type;         // FactSet security type
  private String fref_listing_exchange;      // Exchange code (e.g., "NAS", "NYS")
  private Integer active_flag;               // 1 = active, 0 = inactive

  // ==================== Entity Details ====================
  private String factset_entity_id;          // Entity ID for company
  private String entity_proper_name;         // Company name
  private String iso_country;                // Country code
  private String entity_type;                // Entity type

  // ==================== GICS Classification ====================
  private String gics_sector_code;
  private String gics_industry_group_code;
  private String gics_industry_code;
  private String gics_sub_industry_code;

  // ==================== Resolution Metadata ====================
  private String resolution_path;            // Which recipe was used
  private Integer resolution_steps;          // How many steps were executed
  private Boolean resolution_complete;       // Did all steps succeed?
  private String resolution_notes;           // Any warnings/notes

  // ==================== SecMaster Link ====================
  private Long secmaster_ticker_id;          // Link to SecMaster Ticker.tickerId
  private String secmaster_ticker_name;      // SecMaster ticker name for comparison
  private Boolean secmaster_match_found;     // Was a SecMaster match found?
  private String secmaster_match_type;       // EXACT, FUZZY, NONE
  }

  ---
4. DATABASE SCHEMA (DDL)

-- =====================================================
-- FactSet Identifier Resolution Results
-- =====================================================
-- Purpose: Persist identifier lookup results for:
--   1. Fast subsequent lookups (avoid re-querying FactSet)
--   2. SecMaster access point (join on identifiers)
--   3. Audit trail of resolution decisions
-- =====================================================

CREATE TABLE FactSetResolution.dbo.IdentifierResolutionResult (

      -- Primary Key
      resolutionResultId          BIGINT IDENTITY PRIMARY KEY,

      -- Input Tracking
      inputIdentifierType         VARCHAR(50) NOT NULL,    -- BLOOMBERG_TICKER, ISIN, FIGI, SEDOL, CUSIP
      inputIdentifierValue        VARCHAR(100) NOT NULL,   -- The actual input value
      inputIdentifierHashCode     INT NOT NULL,            -- Hash for quick lookups
      resolutionTimestamp         DATETIME NOT NULL DEFAULT GETDATE(),

      -- Core FactSet IDs
      fsym_id                     CHAR(8),
      fsym_regional_id            CHAR(8),
      fsym_primary_listing_id     CHAR(8),
      fsym_security_id            CHAR(8),
      fsym_primary_equity_id      CHAR(8),

      -- Bloomberg Data
      bbg_id                      CHAR(12),                -- FIGI code
      bbg_ticker                  VARCHAR(30),
      bbg_most_recent             SMALLINT,

      -- Ticker Symbols
      ticker_region               VARCHAR(50),             -- e.g., "AAPL-US"
      ticker_exchange             VARCHAR(50),             -- e.g., "AAPL-NAS"
      ticker_region_start_date    DATE,
      ticker_region_end_date      DATE,

      -- Standard Identifiers
      isin                        CHAR(12),
      sedol                       CHAR(7),
      cusip                       CHAR(9),

      -- Security Details
      currency                    CHAR(3),
      proper_name                 VARCHAR(200),
      fref_security_type          VARCHAR(40),
      fref_listing_exchange       VARCHAR(4),
      active_flag                 SMALLINT,

      -- Entity Details
      factset_entity_id           VARCHAR(20),
      entity_proper_name          VARCHAR(200),
      iso_country                 CHAR(2),
      entity_type                 VARCHAR(50),

      -- GICS Classification
      gics_sector_code            VARCHAR(10),
      gics_industry_group_code    VARCHAR(10),
      gics_industry_code          VARCHAR(10),
      gics_sub_industry_code      VARCHAR(10),

      -- Resolution Metadata
      resolution_path             VARCHAR(100),
      resolution_steps            INT,
      resolution_complete         BIT,
      resolution_notes            VARCHAR(MAX),

      -- SecMaster Link
      secmaster_ticker_id         BIGINT,
      secmaster_ticker_name       VARCHAR(100),
      secmaster_match_found       BIT,
      secmaster_match_type        VARCHAR(20),

      -- Audit
      created                     DATETIME NOT NULL DEFAULT GETDATE(),
      modified                    DATETIME NOT NULL DEFAULT GETDATE()
);

-- =====================================================
-- INDEXES for Fast Lookups
-- =====================================================

-- Primary lookup: by input identifier (what SecMaster will use)
CREATE INDEX idx_input_identifier
ON FactSetResolution.dbo.IdentifierResolutionResult
(inputIdentifierType, inputIdentifierHashCode);

CREATE INDEX idx_input_value
ON FactSetResolution.dbo.IdentifierResolutionResult
(inputIdentifierValue);

-- FactSet ID lookups
CREATE INDEX idx_fsym_id
ON FactSetResolution.dbo.IdentifierResolutionResult
(fsym_id);

CREATE INDEX idx_fsym_regional_id
ON FactSetResolution.dbo.IdentifierResolutionResult
(fsym_regional_id);

-- Bloomberg lookups
CREATE INDEX idx_bbg_id
ON FactSetResolution.dbo.IdentifierResolutionResult
(bbg_id);

CREATE INDEX idx_bbg_ticker
ON FactSetResolution.dbo.IdentifierResolutionResult
(bbg_ticker);

-- Standard identifier lookups
CREATE INDEX idx_isin
ON FactSetResolution.dbo.IdentifierResolutionResult
(isin);

CREATE INDEX idx_sedol
ON FactSetResolution.dbo.IdentifierResolutionResult
(sedol);

CREATE INDEX idx_cusip
ON FactSetResolution.dbo.IdentifierResolutionResult
(cusip);

-- Ticker region lookup (for SecMaster alphaTheoryTickerName matching)
CREATE INDEX idx_ticker_region
ON FactSetResolution.dbo.IdentifierResolutionResult
(ticker_region);

-- SecMaster link
CREATE INDEX idx_secmaster_ticker_id
ON FactSetResolution.dbo.IdentifierResolutionResult
(secmaster_ticker_id);

  ---
5. EXPOSING ALL AVAILABLE BLOOMBERG CODES

Simple Query (Fast - Just Lists Available Tickers)

-- All active Bloomberg tickers in FactSet
SELECT DISTINCT bbg_ticker, bbg_id, fsym_id, most_recent
FROM fds.sym_v1.sym_bbg
WHERE bbg_ticker IS NOT NULL
AND most_recent = 1
ORDER BY bbg_ticker;

-- Count: ~500,000+ records

Materialized View for Fast Access

-- =====================================================
-- Materialized View: All Available Bloomberg Codes
-- =====================================================
-- Purpose: Fast lookup of ALL Bloomberg codes FactSet knows about
-- Refresh: Daily (or on-demand)
-- =====================================================

CREATE TABLE FactSetResolution.dbo.AvailableBloombergCodes (
bbg_ticker          VARCHAR(30) PRIMARY KEY,
bbg_id              CHAR(12),           -- FIGI
fsym_id             CHAR(8),
fsym_regional_id    CHAR(8),
currency            CHAR(3),
fref_listing_exchange VARCHAR(4),
proper_name         VARCHAR(200),
active_flag         SMALLINT,
last_refreshed      DATETIME DEFAULT GETDATE()
);

-- Populate/Refresh Query
INSERT INTO FactSetResolution.dbo.AvailableBloombergCodes
SELECT DISTINCT
b.bbg_ticker,
b.bbg_id,
b.fsym_id,
c.fsym_regional_id,
c.currency,
c.fref_listing_exchange,
c.proper_name,
c.active_flag,
GETDATE() as last_refreshed
FROM fds.sym_v1.sym_bbg b
JOIN fds.sym_v1.sym_coverage c ON b.fsym_id = c.fsym_id
WHERE b.bbg_ticker IS NOT NULL
AND b.most_recent = 1
AND c.active_flag = 1;

  ---
6. SECMASTER ACCESS PATTERNS

Pattern 1: SecMaster Queries by Bloomberg Ticker

-- SecMaster can now query: "Do we know about AAPL US?"
SELECT *
FROM FactSetResolution.dbo.IdentifierResolutionResult
WHERE inputIdentifierType = 'BLOOMBERG_TICKER'
AND inputIdentifierValue = 'AAPL US';

Pattern 2: SecMaster Queries by Any Identifier

-- SecMaster can query by ISIN
SELECT *
FROM FactSetResolution.dbo.IdentifierResolutionResult
WHERE isin = 'US0378331005';

-- SecMaster can query by FIGI
SELECT *
FROM FactSetResolution.dbo.IdentifierResolutionResult
WHERE bbg_id = 'BBG000B9XRY4';

-- SecMaster can query by ticker_region (matches AlphaTheory ticker name format)
SELECT *
FROM FactSetResolution.dbo.IdentifierResolutionResult
WHERE ticker_region = 'AAPL-US';

Pattern 3: Bulk Validation for SecMaster

-- SecMaster wants to validate all its tickers against FactSet
SELECT
t.tickerId,
t.name as secmaster_name,
t.bloombergTicker,
t.isin,
r.ticker_region as factset_ticker_region,
r.active_flag,
r.currency,
CASE
WHEN r.resolutionResultId IS NOT NULL THEN 'MATCHED'
ELSE 'NO_FACTSET_MATCH'
END as validation_status
FROM SecMaster.dbo.Ticker t
LEFT JOIN FactSetResolution.dbo.IdentifierResolutionResult r
ON t.bloombergTicker = r.bbg_ticker
OR t.isin = r.isin;

  ---
7. RECIPE EXECUTION SERVICE (Java)

/**
* FactSetIdentifierRecipeService
*
* Executes identifier recipes and persists results.
  */
  public class FactSetIdentifierRecipeService {

  /**
    * Execute a recipe starting from Bloomberg Ticker
      */
      public static FactSetIdentifierResolutionResult executeBloombergTickerRecipe(
      String bloombergTicker) {

      FactSetIdentifierResolutionResult result = new FactSetIdentifierResolutionResult();
      result.setInputIdentifierType("BLOOMBERG_TICKER");
      result.setInputIdentifierValue(bloombergTicker);
      result.setInputIdentifierHashCode(bloombergTicker.hashCode());
      result.setResolutionTimestamp(Timestamp.valueOf(LocalDateTime.now()));
      result.setResolution_path("BLOOMBERG_TICKER → sym_bbg → sym_coverage → sym_ticker_region → sym_isin");

      int steps = 0;

      // Step 1: Bloomberg Ticker → fsym_id + bbg_id
      ATDBFactSetSymbologyBloomberg bbgResult =
      ATDBFactSetSymbologyBloombergRowMapper.getByBloombergTicker(bloombergTicker);

      if (bbgResult != null) {
      steps++;
      result.setFsym_id(bbgResult.getFsym_id());
      result.setBbg_id(bbgResult.getBbg_id());
      result.setBbg_ticker(bbgResult.getBbg_ticker());
      result.setBbg_most_recent(bbgResult.getMost_recent());

           // Step 2: fsym_id → Full Coverage
           ATDBFactSetSymbologyCoverage coverage =
               ATDBFactSetSymbologyCoverageRowMapper.getByFsymId(bbgResult.getFsym_id());

           if (coverage != null) {
               steps++;
               result.setFsym_regional_id(coverage.getFsym_regional_id());
               result.setFsym_primary_listing_id(coverage.getFsym_primary_listing_id());
               result.setFsym_security_id(coverage.getFsym_security_id());
               result.setCurrency(coverage.getCurrency());
               result.setProper_name(coverage.getProper_name());
               result.setFref_listing_exchange(coverage.getFref_listing_exchange());
               result.setActive_flag(coverage.getActive_flag());

               // Step 3: fsym_regional_id → ticker_region
               // Step 4: fsym_id → ISIN
               // Step 5: Entity + GICS
               // ... continue cascading lookups
           }
      }

      result.setResolution_steps(steps);
      result.setResolution_complete(steps >= 5);

      return result;
      }

  /**
    * Batch execute recipes and persist results
      */
      public static void executeBatchBloombergTickerRecipes(
      List<String> bloombergTickers) {

      List<FactSetIdentifierResolutionResult> results = new ArrayList<>();

      for (String ticker : bloombergTickers) {
      results.add(executeBloombergTickerRecipe(ticker));
      }

      // Batch insert results
      FactSetIdentifierResolutionResultRowMapper.batchInsert(results);
      }
      }

  ---
8. RECIPE SUMMARY MATRIX
   ┌──────────┬──────────────────┬─────────────┬───────────────────┬────────────────────────────────────────────────────────────┐
   │ Recipe # │ Input Identifier │ Trust Level │   Primary Table   │                        What You Get                        │
   ├──────────┼──────────────────┼─────────────┼───────────────────┼────────────────────────────────────────────────────────────┤
   │ 1        │ Bloomberg Ticker │ HIGH        │ sym_bbg           │ fsym_id, FIGI, coverage, ticker_region, ISIN, entity, GICS │
   ├──────────┼──────────────────┼─────────────┼───────────────────┼────────────────────────────────────────────────────────────┤
   │ 2        │ ISIN             │ HIGH        │ sym_isin          │ fsym_id → then same as Recipe 1                            │
   ├──────────┼──────────────────┼─────────────┼───────────────────┼────────────────────────────────────────────────────────────┤
   │ 3        │ FIGI (bbg_id)    │ HIGH        │ sym_bbg           │ fsym_id, bbg_ticker → then same as Recipe 1                │
   ├──────────┼──────────────────┼─────────────┼───────────────────┼────────────────────────────────────────────────────────────┤
   │ 4        │ SEDOL            │ MEDIUM      │ sym_sedol         │ fsym_id → then same as Recipe 1                            │
   ├──────────┼──────────────────┼─────────────┼───────────────────┼────────────────────────────────────────────────────────────┤
   │ 5        │ CUSIP            │ MEDIUM      │ sym_cusip         │ fsym_id → then same as Recipe 1                            │
   ├──────────┼──────────────────┼─────────────┼───────────────────┼────────────────────────────────────────────────────────────┤
   │ 6        │ Ticker Region    │ HIGH        │ sym_ticker_region │ fsym_regional_id → then reverse to get Bloomberg           │
   ├──────────┼──────────────────┼─────────────┼───────────────────┼────────────────────────────────────────────────────────────┤
   │ 7        │ fsym_id (any)    │ HIGH        │ sym_coverage      │ Full coverage + all linked identifiers                     │
   └──────────┴──────────────────┴─────────────┴───────────────────┴────────────────────────────────────────────────────────────┘
  ---
9. NEXT STEPS

1. Create the DDL - FactSetResolution.dbo.IdentifierResolutionResult
2. Create the entity - FactSetIdentifierResolutionResult.java
3. Create the RowMapper - FactSetIdentifierResolutionResultRowMapper.java
4. Create the Recipe Service - FactSetIdentifierRecipeService.java
5. Create the materialized view - AvailableBloombergCodes (daily refresh)
6. Add SecMaster integration - View/stored procedures for SecMaster to query

