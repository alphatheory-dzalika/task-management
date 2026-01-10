FactSet FGP FactSetGlobalPricing


FX.  Entity is Currency Code ISO.   
We have the logic of INIT and TOPOFF, BackFill.

The source is the FactSet table.  FDS.Ref_v2.econ_fx_rates_usd
The intermediate table is vendors.fds.exchange_rate_history


Then we have to copy from  vendors.fds.exchange_rate_history  into atstaging/atprod.dbo.exchangeRateHistory


****

Dividend History.

This goes from vendors.fds.ca_events_flattened and related tables to the TickerDividendHistory table.

