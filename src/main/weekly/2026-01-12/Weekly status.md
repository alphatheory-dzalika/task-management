Last week:

Completed:
* Meeting to discuss Integration Tasking Management system, with presentation about the process and its progress this Thursday.
* https://github.com/alphatheory/yahoo-finance-mcp.  First commit. Used to help testing / discovery of FactSet Corporate Actions processing.
* Intermediate step. "FDS STAGES" for use with FactSet Global Prices. https://github.com/alphatheory/dptv2/blob/feature/AP-6048-th-calc-as-a-service/sql/fds_stages/FDS_MEDALLION_ARCHITECTURE.md
* Post-mortem write up 

In Progress:
* Task Management workflows, including individual and collective organization of details
* Prices Orchestration. For FGP.
* Find or Create Ticker as a service
* Prices & Scoring using the IDM.
* Intraday pricing for Factset. Receive documentation and trial. https://alphatheory.atlassian.net/browse/IDM-49.
**  This step will involve: 1) Retrieve Tickers to price, per region, with priority;
****  2) Build block_1_every_10_min and block_2_every_30_min, according to the time of day and (potentially), region
****  Note that we can also manage TickerHistoryLive to promote prices based on Region and in coordination of the FGP "Top-Off" Job
* Start of Ticker as Product conversation, with CBP 
Upcoming:
* THCalcServices to test ... once the Prices orchestration step is complete



Document: "C:\Users\dzalika\Downloads\FAC Project Ticker Resolver Closure Report.md"   