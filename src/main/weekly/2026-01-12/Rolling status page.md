Working on MCPs  

Your claude_desktop_config.json already has 8 MCPs configured:

| MCP               | Status                             |
  |-------------------|------------------------------------|
| alpha-knowledge   | ✅ Ready                           |
| alpha-gateway     | ✅ Ready                           |
| alpha-collector   | ✅ Ready                           |
| rapi-mcp          | ✅ Ready (in modelcontextprotocol) |
| idm-knowledge     | ✅ Ready (Java)                    |
| factset-estimates | ✅ Ready (Java)                    |
| yahoo-finance     | ✅ Ready                           |
| sqlite-yf         | ✅ Ready              




Working on Project Management system

Excel Connect 

Working on FDS FGP orchestration

Roadmap for CX / Integration
ongoing maintenance?   
Quality stats on current departments
IDM -->  Path to production for ...
Scoring -->  the start towards a larger "monitoring and enrichment" system
Pricing (a start towards Sec Master Market Data Provider fulfillment)
Using ExternalAppData
vs. using IDM-34 orchestration
FactSet Estimates and Multiples
Atomic Adapter into the IDM
Find or Create Ticker into the IDM ... plus Aggregation
Standard Adapter into the IDM ...  external app history and leaving the bash.sh system
External API into External App History for Adapter servicing (ATOP / Arcana / ???)
CX Servicing ... all steps to onboard a client as easily as possible
Historical Data Upload ... revisiting the "External" object types
RAPI and everything related
Adapter Instructions / Client Data Feed and related Adapter processing MCP
CX servicing
Pre-sales
Incentivizing Sales cooperation --> our best / most turn-key
More features ...  including Optimal-position-size as part of adapter run time




Adapter Ticker Resolver --> Find or Create Ticker
Recent Adapter release:  TickerFactory upgrade:  Updated 10K unique tickers, 7K of which have FSYM_IDs
Uses the FIGI/FactSet validated symbol; Upgraded Bloomberg Tickers, defining a standard
Summarize number of records added: 3500 Entities defined, for de-duplication;  
The IDM --> From master to main
This is how we create the hardened API ready for use by other clients
Robert mentioned mTLS for consideration for use with internal-only endpoints
Steve can help us stand up the API
Scoring and EOD prices in ExternalAppData
Ready, but delayed, Antoine is leading, but has been OOO with Covid
Remaining steps: Run in Staging and generate Proof of Results
Pricing (Corporate Actions / Dividend History; Adjusted Prices ; FX)
Lookup (Find or Create Ticker)
Internal Answer Engine
Client Lifecycle system 


Upgraded Bloomberg Tickers, defining a standard competence 






Roadmap for  FactSet migration
Phase	Focus	Target
0	Ticker stabilization & wins	✅ Done
1	EOD Pricing (Scoring)
TBD (Antoine) --> Code complete; See "EOD Pricing (Scoring)" Plan for path to prod.


2

Prices: Intraday

Jan–Apr -->  Upon completion, ready to pay FactSet

3
Prices: THCalcServices & SecMaster &TickerHistory

Jan–Apr









4
Black-Scholes
Jan–Apr --> Requires Volatility


5

Find or Create Ticker & FoC FundAsset

Jan–Apr


6

Ticker as Data Product

Q2


EOD Pricing (Scoring):
Needed:
Run in staging, conform results -->
In Ticker/FundAsset ExternalAppData; Confirm the correct use of ATSettings / DepartmentSettings
Pre-req for prod:
Add new tables in prod for intermediate-stage data
Run IDM as a .jar in adapter, as we do ATOP
Confirm correct use of ATSettings / DepartmentSettings
FactSet → TickerHistory
Status:  Orchestration is in progress
Conclusion up front: SQL orchestration is the quickest path to a complete workflow
SQL does need peer review. It is fast, we can check the work.
It will need to be converted to Postgres, or equivalent.  
See supporting "SQL Server as an implementation choice vs. other choices"
Still in progress ... the complete orchestration; most layers are correct, but the system is detailed and the outcomes need to be complete.
FactSet --> Intraday;
We can and should start now
Robert could help if he has any capacity
Without any additional work, we could begin polling from the Ticker table, using the fsymIdExchangeLevel
With additional work, we can make this a dynamic process that only requests tickers held by clients, and we could even cross reference ticker held by clients IN THE APPLICATION
Find or Create Ticker & FundAsset
This is the porting of the Adapter's Ticker Resolver into an independent system
An incoming "Ticker Candidate" will find the best matching choice that exists or it will create that ticker
It will search for that ticker in the given client's environment and will return that FundAsset
Black Scholes
This mainly depends on having Volatilities available from FactSet
There are corresponding Data structures to be built, and there are no blockers
We will begin planning and expect that the planning can be completed in the next two weeks, by January 22nd

*** **********

Related:
Ticker as a Product discussions and next steps:
Status: We have stats for the ticker coverage:
✅ Key Outcomes
Significant net benefit, and an important step toward making FactSet our primary vendor
11,116 Fund Asset client records processed since the release
~3,500 unique equity tickers
153 total assets updated to align with the new standards
Only 19 had active research, resulting in minimal client impact and straightforward triage
🏷️ Ticker Enhancements
3,304 additions to the Ticker entity to properly link Parent securities and ADRs
2,835 ticker descriptions updated to match FactSet standards
688 ticker FactSet IDs updated to improve Estimates mapping
226 Bloomberg tickers updated to normalize on regional exchange codes
Decommissioning of the JTIC standard
Currently, Ticker.NAME is based on the Sungard JTIC
US = Four_space_five_convention; CAD = Ticker_asterisk;  RestOfWorld = S_Colon
The new standard will be the Bloomberg Composite Ticker
This is widely accepted and is open source
We can back this up with a Bloomberg Global ID
There is some support for an ISIN
We should have the MIC Exchange Code and other supporting details
We will need to have a map of TickerID to Ticker.Name (OLD) as of a certain date
RETIRING tickers
When we switch to the new standard, we will have some tickers that resolve to the same Bloomberg Composite Ticker value
We will have a rule for deciding which of the tickers is the primary; usually it will be the one that was using the JTIC convention
DELISTING tickers
We'll have to decide how to begin delisting tickers, including setting the "expired" value, possibly adding a "delisted" field, removing a price, and more.
We have clients sending us delisted tickers, because they provide Year-to-Date views of their portfolio
We would want to use them as test clients
Ticker lifecycle processing ...
We will have to build orchestration for the many capital changes and corporate actions events and any relating identifier changes
External Client Communication
CBP & Masterfile support.
Alpha Theory can provide a "validated Identifier explanation" with each ticker, and can work with CBP in their environment (AT as CBP contractors with CBP licensing on CBP AWS instances) to build the corresponding connectors to Bloomberg SAPI/equivalent and/or LSEG's Masterfile.
CBP would be able to see exact entity mappings, including ADR/non-primary holdings and Chinese Foreign vs. National entities
Alpha Theory Ticker Lookup as a service
AT will be able to explain to clients exactly how we treat their incoming data
Clients would be able to know up front if there are any coverage gaps and prepare supplementary data sources accordingly