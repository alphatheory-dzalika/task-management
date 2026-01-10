Notes.  This was not an easy session.

The act of coordinating so many stored procedures, tables, calling them, testing them, and more, using SQL and Claude, is quite difficult.

But, as ChatGPT has explained, this is still using standard Data Orchestration principles.

It's just that we have to design it in Claude and without the ability to use Object Oriented Programming, a benefit of Java.


Where we were ...   
First:
We built the Price orchestration. We tried to use the FactSet "already derived" version.
1. Define TickerMapping.Snapshots. Ticker Universe from Clients with Count and Priority.
2. Get FactSet details, including Listing/Regional pricing stats, Region and Exchange metadata, and 
3.   We discovered that we can't trust the derived factset adjustment factors 
Questions to Confirm

1. Is INIT = "never processed before"? Or does it mean something else?  Yes.
2. Is TOPOFF = "changed since last run"? Based on checksum/count comparison? 
   I think it can be as simple as "If we have changes that moved from UPCOMING to PROCESS NOW"
    ...  so we will have to have a way of understanding the "upcoming" entries, what does that mean?
    Ah, we will always review from the same window ... check ENTRY_ID where effective_date is _should process today_ 
   And if that ENTRY_ID has not yet been processed, then that means we have a REFRESH to perform
    REFRESH means that we rebuild the corporate actions artifacts and then re-INIT the prices as well
    IMPORTANT:   The corporate actions adjustment artifacts should go back to 2015, January 1, but INIT is typically 3 or 4 years for pricing.
        This is so that we can perform backfill safely as we want. And we will backfill no earlier than 20150101 as of this time.
3. Is BACKFILL = "force reprocess specific items"? Ignoring current status?  ...      It's a simple addition of price * adjustment for that date ...  
        And it's a tool.   It allows us to INIT the bare minimum and then to process the rest of the earlier process using Backfill.
        BUT.  Just like TOPOFF, if it's changed since the last run, then we will first perform INIT and then back fill.
4. Is there a REFRESH mode? (Reprocess ALL regardless of status - like BACKFILL but for everything?)   Yes. 
       Ok, let's have a FORCE REFRESH, which is the same as an INIT, but it ignores if there is already an INIT.
5. Who decides which mode to run? Is it a parameter to the procedure, or separate entry points?
  Every day, we will first run the stored proc that populates TickerMapping.snapshots.tickers_with_holding_counts
    then we will run the proc that populates Vendors.fds.ticker_factset_coverage_summary 
    
  Every ticker first runs INIT.  If they aren't yet in the system, we enter INIT, to the latest day.
        
   When we first run this job, we will run INIT for all tickers in  Vendors.fds.ticker_factset_coverage_summary 
    And on a go-forward basis, only new tickers will be INIT-ed
And separately, every day, we have a job that runs TOPOFF from Vendors.fds.ticker_factset_coverage_summary, per Region.   
So, we try to top-off each ticker, per region, and let's say that each region runs as many times following each region, 
     until done, let's say, for 5 hours after that region has closed.




MODES - Confirmed Understanding

1. INIT

- When: Ticker has never been processed before
- CA Artifacts: Process back to 2015-01-01
- Prices: Process only 3-4 years (latest data)
- Trigger: New ticker appears in ticker_factset_coverage_summary

2. TOPOFF

- When: Daily operations for existing tickers
- Detection: Check if any UPCOMING events are now effective (effective_date <= TODAY)
- If event_id not yet processed → Triggers REFRESH for that fsym_id
- Runs: Per region, loops until done (up to 5 hours after region close)

3. REFRESH (triggered by TOPOFF)

- What it does:
  a. Rebuild CA artifacts (translated → daily → cumulative)
  b. Re-INIT prices (apply new factors)
- When: UPCOMING event became effective and wasn't processed yet

4. FORCE REFRESH

- Same as INIT but ignores existing status
- Use case: Need to reprocess from scratch

5. BACKFILL

- A tool to add historical price adjustments beyond INIT's 3-4 year window
- Goes back to: 2015-01-01 (not earlier)
- Important: If changes detected since last run → run INIT first, then backfill
- Formula: Simple price × adjustment_factor for historical dates

  ---
DATE RANGES - Critical Distinction
┌─────────────────┬────────────────────┬────────────────────────────────┐
│      What       │     Date Range     │              Why               │
├─────────────────┼────────────────────┼────────────────────────────────┤
│ CA Artifacts    │ Back to 2015-01-01 │ Full adjustment history needed │
├─────────────────┼────────────────────┼────────────────────────────────┤
│ INIT Prices     │ Only 3-4 years     │ Minimum needed for daily use   │
├─────────────────┼────────────────────┼────────────────────────────────┤
│ BACKFILL Prices │ Back to 2015-01-01 │ Historical analysis, on-demand │
└─────────────────┴────────────────────┴────────────────────────────────┘
  ---
DAILY WORKFLOW

┌─────────────────────────────────────────────────────────────────────┐
│  DAILY JOB (runs once at start of day)                              │
├─────────────────────────────────────────────────────────────────────┤
│  1. sp_refresh_tickers_with_holding_counts (TickerMapping)          │
│  2. build_ticker_factset_coverage_summary (Vendors.fds)             │
│  3. For each NEW ticker → queue for INIT                            │
└─────────────────────────────────────────────────────────────────────┘
↓
┌─────────────────────────────────────────────────────────────────────┐
│  REGIONAL TOPOFF JOBS (run per region, loop until done)             │
├─────────────────────────────────────────────────────────────────────┤
│  Per region (A, E, F, L, M, N, Y):                                  │
│    - Check UPCOMING events where effective_date <= TODAY            │
│    - If event_id not processed → queue REFRESH for that fsym_id     │
│    - Process queue until empty (or 5 hours after region close)      │
└─────────────────────────────────────────────────────────────────────┘

  ---
UPCOMING Events - The Key Mechanism

UPCOMING tables hold future-dated events:
- ca_events_translated_upcoming
- ca_daily_factors_upcoming

Daily check:
SELECT event_id, fsym_id, effective_date
FROM ca_events_translated_upcoming
WHERE effective_date <= CAST(GETDATE() AS DATE)
AND event_id NOT IN (SELECT processed_event_id FROM fsym_ca_status)

    → These need REFRESH

  ---
Questions to Confirm

1. Region codes: Are these the FDS region codes (A=Asia, E=Europe, etc.)? And each region has its own TOPOFF job?

A,Asia
E,Europe
F,Africa
L,Latin America
M,Middle East
N,North America
Y,Pacific


2. INIT prices "3-4 years": Is this a config parameter or hardcoded? What's the exact logic?
Let's say, this is going to be a global configurable variable BUT we will start with a default value in the INIT using 4 years: Beginning of 2022.

3. BACKFILL triggers INIT first: Does this mean if I request BACKFILL for ticker X, and X has unprocessed UPCOMING events, we first do REFRESH then BACKFILL?
Ok, this is nuanced, and is incorrect.  Let's say that first, we need a process during TOPOFF and BACKFILL that first does Evaluate Corporate Actions
...  it will start with a Date:CorporateActionsLastEvaluationDate. ...  and it will find all Events in CA_EVENTS with Effective_date > Today - 3 days ago.
... and then it will see if those events have been processed.  The "UPCOMING" is really just as a reference and for reporting.
...  we always update UPCOMING, but only when we have events that are TODAY or earlier do we process them.  And EVERY NEW CA_EVENT means that we have a REFRESH
 ... Yes? Clear logic?
4. "5 hours after region close": Is this a timeout/ceiling, or does it keep retrying for 5 hours?
The things is that for each region ... we are tracking their earliest 4 pm local time.  
So from 3 pm and on, we begin reading from each region, until 9 pm their local time
 ....   we could do something like ...  Process all tickers once, every 10 minutes until all are either processed, or done ... 
5. Should I document this in a single authoritative file that becomes the new "CLAUDE_MUST_READ_FIRST.md" for V5/V6?
Let's understand what this document is.  Are we doing "Claude must read first">
No, we are writing:   TickerMapping processing through CA Events and Prices-Derived.  And Within that document we have what you just learned, and now these questions to confirm.
Which are explaining INIT/BackFill/Topoff logic.



....  next ...

Updated. The key clarification:

Every 10 minutes:
- If still running → Wait (no overlap)
- If not running → Start and run to completion

This prevents multiple instances of the same region job running simultaneously.

Anything else to correct in the specification before we proceed?

❯ no

● Good. Here's where we stand:

Current State
┌──────────────────────────────────────────┬───────────────────────────────────┐
│                   Item                   │              Status               │
├──────────────────────────────────────────┼───────────────────────────────────┤
│ sql/fds_v5/FDS_PIPELINE_SPECIFICATION.md │ ✅ Complete - authoritative spec  │
├──────────────────────────────────────────┼───────────────────────────────────┤
│ sql/fds_v5/CLAUDE_MUST_READ_FIRST.md     │ ✅ Design reference (still valid) │
├──────────────────────────────────────────┼───────────────────────────────────┤
│ sql/fds_v5/01_tickermapping/             │ ✅ Complete (tables + procedure)  │
├──────────────────────────────────────────┼───────────────────────────────────┤
│ sql/fds_v5/02_fds_tracking/              │ ❌ Not started                    │
├──────────────────────────────────────────┼───────────────────────────────────┤
│ sql/fds_v5/03_ca_output/                 │ ❌ Not started                    │
├──────────────────────────────────────────┼───────────────────────────────────┤
│ sql/fds_v5/04_ca_procedures/             │ ❌ Not started                    │
└──────────────────────────────────────────┴───────────────────────────────────┘
Next Steps

1. 02_fds_tracking/ - Create tracking schema:
   - ca_pipeline_queue (with filtered indexes)
   - fsym_ca_status
   - Helper procedures (claim/complete/error batch)
2. 03_ca_output/ - Create output tables:
   - ca_events_translated + _upcoming
   - ca_daily_factors + _upcoming
   - ca_cumulative_factors
   - factset_prices_adjusted_derived
3. 04_ca_procedures/ - Create processing procedures:
   - sp_evaluate_corporate_actions
   - sp_process_ca_level1/2/3
   - sp_process_price_derived
   - sp_run_ca_pipeline (orchestrator)

Ready to start building 02_fds_tracking?

❯ give me a plan for   │ sql/fds_v5/02_fds_tracking/              │ ❌ Not started                    │   and yes.

● Plan: sql/fds_v5/02_fds_tracking/

Folder Structure

sql/fds_v5/02_fds_tracking/
├── 01_schema.sql                    ← Create fds_tracking schema
├── 02_tables.sql                    ← ca_pipeline_queue + fsym_ca_status
├── 03_sp_claim_ca_batch.sql         ← Claim work, set INCOMPLETE
├── 04_sp_complete_ca_batch.sql      ← Mark COMPLETED, hand off to next stage
├── 05_sp_error_ca_batch.sql         ← Mark ERROR with message
├── 06_sp_reset_stuck_batches.sql    ← Reset timed-out INCOMPLETE items
├── 07_sp_refresh_fsym_ca_status.sql ← Refresh source metadata
├── 08_sp_evaluate_corporate_actions.sql ← Key: detect new events needing REFRESH
├── 09_sp_queue_ca_events.sql        ← Queue work (INIT/TOPOFF/REFRESH modes)
├── 10_sp_get_pipeline_status.sql    ← Health check / monitoring
└── README.md                        ← Documentation for this folder

File Details

01_schema.sql

-- Create fds_tracking schema in Vendors database
USE Vendors;
CREATE SCHEMA fds_tracking;

02_tables.sql

Two tables:

ca_pipeline_queue (granular work items)
┌────────────────┬──────────────────┬────────────────────────────────────────────────────┐
│     Column     │       Type       │                      Purpose                       │
├────────────────┼──────────────────┼────────────────────────────────────────────────────┤
│ queue_id       │ BIGINT IDENTITY  │ PK                                                 │
├────────────────┼──────────────────┼────────────────────────────────────────────────────┤
│ stage          │ VARCHAR(20)      │ CA_LEVEL1, CA_LEVEL2, CA_LEVEL3, PRICE_DERIVED     │
├────────────────┼──────────────────┼────────────────────────────────────────────────────┤
│ fsym_id        │ CHAR(8)          │ Always required                                    │
├────────────────┼──────────────────┼────────────────────────────────────────────────────┤
│ event_id       │ CHAR(8)          │ Required for CA_LEVEL1, NULL for others            │
├────────────────┼──────────────────┼────────────────────────────────────────────────────┤
│ effective_date │ DATE             │ Required for CA_LEVEL1/2/3, NULL for PRICE_DERIVED │
├────────────────┼──────────────────┼────────────────────────────────────────────────────┤
│ status         │ VARCHAR(20)      │ REQUESTED, INCOMPLETE, COMPLETED, ERROR            │
├────────────────┼──────────────────┼────────────────────────────────────────────────────┤
│ requested_at   │ DATETIME2(3)     │ When queued                                        │
├────────────────┼──────────────────┼────────────────────────────────────────────────────┤
│ started_at     │ DATETIME2(3)     │ When INCOMPLETE set                                │
├────────────────┼──────────────────┼────────────────────────────────────────────────────┤
│ completed_at   │ DATETIME2(3)     │ When finished                                      │
├────────────────┼──────────────────┼────────────────────────────────────────────────────┤
│ error_message  │ NVARCHAR(MAX)    │ Error details                                      │
├────────────────┼──────────────────┼────────────────────────────────────────────────────┤
│ retry_count    │ INT              │ Number of retries                                  │
├────────────────┼──────────────────┼────────────────────────────────────────────────────┤
│ batch_id       │ UNIQUEIDENTIFIER │ Groups items in same batch                         │
├────────────────┼──────────────────┼────────────────────────────────────────────────────┤
│ worker_id      │ INT              │ @@SPID for debugging                               │
└────────────────┴──────────────────┴────────────────────────────────────────────────────┘
Plus 4 filtered indexes (one per stage)

fsym_ca_status (per-fsym_id tracking)
┌────────────────────────┬──────────────┬────────────────────────────────────┐
│         Column         │     Type     │              Purpose               │
├────────────────────────┼──────────────┼────────────────────────────────────┤
│ fsym_id                │ CHAR(8)      │ PK                                 │
├────────────────────────┼──────────────┼────────────────────────────────────┤
│ last_event_id          │ CHAR(12)     │ Last processed event               │
├────────────────────────┼──────────────┼────────────────────────────────────┤
│ last_event_date        │ DATE         │ Date of last event                 │
├────────────────────────┼──────────────┼────────────────────────────────────┤
│ source_event_count     │ INT          │ Events in source                   │
├────────────────────────┼──────────────┼────────────────────────────────────┤
│ source_checksum        │ BIGINT       │ For change detection               │
├────────────────────────┼──────────────┼────────────────────────────────────┤
│ processed_event_count  │ INT          │ Events we've processed             │
├────────────────────────┼──────────────┼────────────────────────────────────┤
│ processed_checksum     │ BIGINT       │ What we've processed               │
├────────────────────────┼──────────────┼────────────────────────────────────┤
│ ca_level1_complete     │ BIT          │ Stage status                       │
├────────────────────────┼──────────────┼────────────────────────────────────┤
│ ca_level2_complete     │ BIT          │ Stage status                       │
├────────────────────────┼──────────────┼────────────────────────────────────┤
│ ca_level3_complete     │ BIT          │ Stage status                       │
├────────────────────────┼──────────────┼────────────────────────────────────┤
│ price_derived_complete │ BIT          │ Stage status                       │
├────────────────────────┼──────────────┼────────────────────────────────────┤
│ last_evaluation_date   │ DATE         │ CorporateActionsLastEvaluationDate │
├────────────────────────┼──────────────┼────────────────────────────────────┤
│ last_processed_at      │ DATETIME2(3) │ When last processed                │
└────────────────────────┴──────────────┴────────────────────────────────────┘
03-06: Batch Management Procedures

- sp_claim_ca_batch: SELECT + UPDATE in transaction, returns batch_id
- sp_complete_ca_batch: Mark done, queue next stage
- sp_error_ca_batch: Capture error, increment retry
- sp_reset_stuck_batches: Find INCOMPLETE > N minutes, reset to REQUESTED

07_sp_refresh_fsym_ca_status.sql

- Scans fgp_ca_events for our ticker universe
- Updates source_event_count, source_checksum
- Used for INIT detection (processed IS NULL)

08_sp_evaluate_corporate_actions.sql (KEY PROCEDURE)

Input: @region (optional), @fsym_id (optional)

Logic:
1. Get last_evaluation_date from fsym_ca_status
2. Find events in CA_EVENTS where effective_date > TODAY - 3 days
3. For each event:
    - If event_id not in ca_events_translated → NEW
    - If NEW and effective_date <= TODAY → needs REFRESH
4. Return list of fsym_ids needing REFRESH

09_sp_queue_ca_events.sql

Input: @mode (INIT, TOPOFF, REFRESH, FORCE_REFRESH), @fsym_id (optional)

Logic by mode:
- INIT: Queue fsym_ids where processed_event_count IS NULL
- TOPOFF: Call sp_evaluate_corporate_actions, queue results
- REFRESH: Queue specific fsym_id regardless of status
- FORCE_REFRESH: Queue all fsym_ids regardless of status

10_sp_get_pipeline_status.sql

- Queue counts by stage/status
- fsym_id completion summary
- Recent errors
- Stuck items

  ---
Build Order

1. 01_schema.sql - Schema first
2. 02_tables.sql - Tables with indexes
3. 03-06 - Batch management (can be parallel)
4. 07_sp_refresh_fsym_ca_status.sql - Depends on tables
5. 08_sp_evaluate_corporate_actions.sql - Depends on tables
6. 09_sp_queue_ca_events.sql - Depends on #8
7. 10_sp_get_pipeline_status.sql - Depends on tables
8. README.md - Documentation

how is this used?   I suppose we're calculating the checksum after each time we refresh or init or force refresh the corporate action events translated, then to stage 2
and stage 3 ...   but we would still never have the checksum for change detection.  What do you think it means?  Don't we just have the EventID ...  And what do we
retain?  We retain ALL Events that were processed.  Since 2015.  Right?     So we are going to have the following, all the time ...  ALl the CA_Events that were
processed and moved through the different stages.  And how does that look, from an orchestration perspective?  And then what else?   What are we building?  Capturing?
* What we always have:
**  All events to process ... From INIT.  And we process ONLY the ones that are effective_date = TODAY and earlier.
**  All UPCOMING events.  We know they are there.  We don't do anything with them.  MAYBE we run a stats on them.  Out of scope. 
**  And then we process the events until we are done.  All orchestration steps.
Then ..   we are done.  Until the next time we run the process.  What do we do?
We find ... #1 ... The LATEST effective_date for that FSYM_ID, processed in CA_EVENTS_that_we_tracked_in_the_tracking_table
  #2 ...  All the CA_Events for that FSYM_ID SINCE that latest CA_EVENTS_that_we_tracked_in_the_tracking_table result
   #3 ... any new events, either are in UPCOMING ... OR  they have to be processed TODAY (they are today or earlier) and thus we need to refresh







Let me acknowledge what happened:
1. We had a detailed plan for 02_fds_tracking/
2. The conversation was compressed
3. I jumped to later stages (03_ca_output, 04_ca_procedures) without confirming the earlier steps were done correctly

Let me go back and confirm my understanding of the tracking/detection logic:

  ---
My Understanding (Please Correct Me)

What We Retain (Permanently)

- ALL translated events since 2015 in ca_events_translated
- This is our source of truth for "what we've processed"

How Change Detection Works

Step 1: Find what we last processed for an fsym_id
last_processed_event_date = MAX(effective_date)
FROM ca_events_translated
WHERE fsym_id = X

Step 2: Find new events in source
SELECT * FROM fgp_ca_events
WHERE fsym_id = X
AND effective_date >= last_processed_event_date  -- inclusive
AND event_id NOT IN (SELECT event_id FROM ca_events_translated WHERE fsym_id = X)

Step 3: Classify new events
- If effective_date <= TODAY → needs REFRESH (process now)
- If effective_date > TODAY → UPCOMING (note it, don't process)

No Checksums Needed?  -->  DZ:  RIGHT?   What's the point of it, besides extra information that likely won't be used?  

Because we retain ALL processed events, we can detect new events by checking if event_id exists in ca_events_translated.

Question: What about event MODIFICATIONS? If FactSet changes data for an existing event_id (e.g., amount changes), we won't detect it with this approach. Is that:
- A) Not a concern (events don't get modified)?  -->  DZ:  As far as I know, EVENTs don't change.  PLEASE WRITE ME A QUESTION FOR FACTSET...  
  -  Dear FactSet, how are events updated? Are there errors? Are changes pushed? How can we know what these changes are?  
- B) Handled separately?  -->  As of now, no
- C) Something we need to design for?  -->  Maybe 

  ---
The fsym_ca_status Table

Is this table needed? Or can we derive everything from ca_events_translated?
 -->  Yes, it's helpful to see the stats.   As of  snapshot_date, snapshot_timestamp, FSYM_ID, latest event (from today or earlier), number upcoming events: ##,
    and other stats.  It's helpful, informational ... maybe something like ... earliest entry, latest entry (from today or earlier),
    avg adjustment ... plus we have MANY things we are checking ... so any thing that we check, we log it in ca_status

If needed, what should it track?
- last_processed_event_date (derived from ca_events_translated?)
- Stage completion flags?  -->  this is more like after we are done ... this isn't for completion ...  except if there is an Error, we should have something like LATEST ERROR
- Something else?

  ---
Please confirm or correct my understanding before I proceed.

Todos
☐ PAUSE - Address critical design questions before continuing
☐ Document how each of 9 event types flows through all 3 layers
☐ Design UPCOMING events handling (separate tables? promotion job?)
☐ Design event MODIFICATION detection (how to detect changed events?)
☐ Verify encapsulation between layers is correct
☒ Create 03_ca_output folder with table definitions
☐ Create 04_ca_procedures folder with CA procedures
☐ Create docs folder with migrated V4 CA references




  ---
Plan for sql/fds_v5/02_fds_tracking/

Purpose

This schema provides:
1. Work queue - Track items through pipeline stages (REQUESTED → INCOMPLETE → COMPLETED | ERROR)
2. Stats/Info - Per-fsym_id statistics (informational, not control)
3. Procedures - Orchestration helpers

  ---
Tables

Table 1: ca_pipeline_queue (work items)
┌────────────────┬──────────────────┬────────────────────────────────────────────────┐
│     Column     │       Type       │                    Purpose                     │
├────────────────┼──────────────────┼────────────────────────────────────────────────┤
│ queue_id       │ BIGINT IDENTITY  │ PK                                             │
├────────────────┼──────────────────┼────────────────────────────────────────────────┤
│ stage          │ VARCHAR(20)      │ CA_LEVEL1, CA_LEVEL2, CA_LEVEL3, PRICE_DERIVED │
├────────────────┼──────────────────┼────────────────────────────────────────────────┤
│ fsym_id        │ CHAR(8)          │ Always required                                │
├────────────────┼──────────────────┼────────────────────────────────────────────────┤
│ event_id       │ CHAR(12)         │ Required for CA_LEVEL1 only                    │
├────────────────┼──────────────────┼────────────────────────────────────────────────┤
│ effective_date │ DATE             │ Required for CA_LEVEL1/2/3                     │
├────────────────┼──────────────────┼────────────────────────────────────────────────┤
│ status         │ VARCHAR(20)      │ REQUESTED, INCOMPLETE, COMPLETED, ERROR        │
├────────────────┼──────────────────┼────────────────────────────────────────────────┤
│ batch_id       │ UNIQUEIDENTIFIER │ Groups items claimed together                  │
├────────────────┼──────────────────┼────────────────────────────────────────────────┤
│ requested_at   │ DATETIME2(3)     │ When queued                                    │
├────────────────┼──────────────────┼────────────────────────────────────────────────┤
│ started_at     │ DATETIME2(3)     │ When INCOMPLETE set                            │
├────────────────┼──────────────────┼────────────────────────────────────────────────┤
│ completed_at   │ DATETIME2(3)     │ When finished                                  │
├────────────────┼──────────────────┼────────────────────────────────────────────────┤
│ error_message  │ NVARCHAR(MAX)    │ Error details                                  │
├────────────────┼──────────────────┼────────────────────────────────────────────────┤
│ retry_count    │ INT              │ Retries                                        │
├────────────────┼──────────────────┼────────────────────────────────────────────────┤
│ region_code    │ CHAR(1)          │ For regional processing                        │
└────────────────┴──────────────────┴────────────────────────────────────────────────┘
Filtered indexes (different keys per stage):
- CA_LEVEL1: (stage, fsym_id, event_id, effective_date)
- CA_LEVEL2: (stage, fsym_id, effective_date)
- CA_LEVEL3: (stage, fsym_id, effective_date)
- PRICE_DERIVED: (stage, fsym_id)

  ---
Table 2: fsym_ca_status (stats/info per fsym_id)
┌───────────────────────┬───────────────┬───────────────────────┐
│        Column         │     Type      │        Purpose        │
├───────────────────────┼───────────────┼───────────────────────┤
│ fsym_id               │ CHAR(8)       │ PK                    │
├───────────────────────┼───────────────┼───────────────────────┤
│ snapshot_date         │ DATE          │ When stats captured   │
├───────────────────────┼───────────────┼───────────────────────┤
│ snapshot_timestamp    │ DATETIME2(3)  │ Precise timestamp     │
├───────────────────────┼───────────────┼───────────────────────┤
│ earliest_event_date   │ DATE          │ First event           │
├───────────────────────┼───────────────┼───────────────────────┤
│ latest_event_date     │ DATE          │ Most recent (≤ TODAY) │
├───────────────────────┼───────────────┼───────────────────────┤
│ upcoming_event_count  │ INT           │ Events > TODAY        │
├───────────────────────┼───────────────┼───────────────────────┤
│ total_event_count     │ INT           │ All events            │
├───────────────────────┼───────────────┼───────────────────────┤
│ avg_adjustment_factor │ FLOAT         │ Statistical           │
├───────────────────────┼───────────────┼───────────────────────┤
│ latest_error          │ NVARCHAR(MAX) │ Last error            │
├───────────────────────┼───────────────┼───────────────────────┤
│ latest_error_at       │ DATETIME2(3)  │ When error occurred   │
└───────────────────────┴───────────────┴───────────────────────┘
  ---
Procedures
┌─────┬───────────────────────────┬────────────────────────────────────────────────────────┐
│  #  │         Procedure         │                        Purpose                         │
├─────┼───────────────────────────┼────────────────────────────────────────────────────────┤
│ 1   │ sp_claim_ca_batch         │ Claim REQUESTED items, set INCOMPLETE, return batch_id │
├─────┼───────────────────────────┼────────────────────────────────────────────────────────┤
│ 2   │ sp_complete_ca_batch      │ Mark COMPLETED, queue next stage                       │
├─────┼───────────────────────────┼────────────────────────────────────────────────────────┤
│ 3   │ sp_error_ca_batch         │ Mark ERROR, capture message                            │
├─────┼───────────────────────────┼────────────────────────────────────────────────────────┤
│ 4   │ sp_reset_stuck_batches    │ Reset INCOMPLETE > N minutes → REQUESTED               │
├─────┼───────────────────────────┼────────────────────────────────────────────────────────┤
│ 5   │ sp_evaluate_ca            │ KEY: Detect new events needing processing              │
├─────┼───────────────────────────┼────────────────────────────────────────────────────────┤
│ 6   │ sp_queue_ca_work          │ Queue work by mode (INIT/TOPOFF/REFRESH)               │
├─────┼───────────────────────────┼────────────────────────────────────────────────────────┤
│ 7   │ sp_refresh_fsym_ca_status │ Update stats table                                     │
├─────┼───────────────────────────┼────────────────────────────────────────────────────────┤
│ 8   │ sp_get_pipeline_status    │ Monitoring/health check                                │
└─────┴───────────────────────────┴────────────────────────────────────────────────────────┘
  ---
Key Procedure Logic: sp_evaluate_ca

FOR each fsym_id in ticker universe:

    -- Step 1: What did we last process?
    last_date = MAX(effective_date) FROM ca_events_translated WHERE fsym_id = X

    -- Step 2: Find new events in source
    new_events = SELECT FROM fgp_ca_events
                 WHERE fsym_id = X
                   AND effective_date >= last_date  -- inclusive
                   AND event_id NOT IN ca_events_translated

    -- Step 3: Classify
    FOR each new_event:
      IF effective_date <= TODAY → mark as REFRESH
      IF effective_date > TODAY  → mark as UPCOMING (info only)

    RETURN fsym_ids needing REFRESH

  ---
Key Procedure Logic: sp_queue_ca_work
┌───────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────────┐
│     Mode      │                                                Logic                                                │
├───────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ INIT          │ Find fsym_ids with NO events in ca_events_translated. Queue all their events (≤ TODAY) to CA_LEVEL1 │
├───────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ TOPOFF        │ Call sp_evaluate_ca. Queue returned fsym_ids to CA_LEVEL1                                           │
├───────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ REFRESH       │ Queue specific fsym_id's events to CA_LEVEL1 (rebuilds that fsym_id)                                │
├───────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────────┤
│ FORCE_REFRESH │ Queue ALL fsym_ids regardless of status                                                             │
└───────────────┴─────────────────────────────────────────────────────────────────────────────────────────────────────┘
  ---
File Structure

sql/fds_v5/02_fds_tracking/
├── 01_schema.sql
├── 02_tables.sql
├── 03_sp_claim_ca_batch.sql
├── 04_sp_complete_ca_batch.sql
├── 05_sp_error_ca_batch.sql
├── 06_sp_reset_stuck_batches.sql
├── 07_sp_evaluate_ca.sql
├── 08_sp_queue_ca_work.sql
├── 09_sp_refresh_fsym_ca_status.sql
├── 10_sp_get_pipeline_status.sql
└── README.md

  ---
Questions Before I Proceed

1. Does this plan look correct? 
    DZ: Yes.   This is somewhat the generic tracking, but maybe we're missing the details of exactly which step?  
    - CA_LEVEL1: (stage, fsym_id, event_id, effective_date)
- CA_LEVEL2: (stage, fsym_id, effective_date)
- CA_LEVEL3: (stage, fsym_id, effective_date)
  - PRICE_DERIVED: (stage, fsym_id)  DZ  -->   So when we're done, we're in a batch, how do we make sure that each pass is logged as part of the tracking
      if there is a failure or error, or number of whatever, or any opportunity to expand this logging, here is what we want to also retain 
   ... I see status but I'm still unclear about how that looks with real examples, even if they are directionally accurate
   ... but we do have plenty of sample data to use to begin fleshing this out
2. Should fsym_ca_status have any other columns?
3. For REFRESH mode - do we DELETE existing ca_events_translated rows for that fsym_id and rebuild, or INSERT/UPDATE?
    For Refresh mode ...  we delete all steps ahead ...
   ca_events_translated, yes, but don't we have a step for stage 1, stage 2, stage 3?  In all cases, we should have a _prior version of EACH table 
    ... and we will simply 