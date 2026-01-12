# FAC Project Closure Report

## Executive Summary

> The FAC (FactSet Alpha Theory Client) project aimed to replace FIS with FactSet and FIGI as the primary sources for Ticker Resolution. This goal was met through a production deployment that saw minimal client disruption—achieving a 99.82% seamless transition rate across over 11,000 positions.
>
> The release closes out the original Adapter board and establishes a baseline for further modernizing Alpha Theory's data management.
>
> While delivery was slower and communications were less frequent than envisioned, the project's core objectives were completed, and significant legacy risks were addressed.

---

## Major Accomplishments

- **FIS for Equity fully removed from Adapter; FactSet and FIGI are now the sources for Equity Ticker Resolution.**
- **SecMaster dependency eliminated for 95% of securities** (with US Options pending further work).
- **99.82% client transition:** Only 19 of 11,116 assets needed customer service intervention.
- **Adapter fully testable locally** (except for US Options), improving reliability and development experience.
- **Performance and operational reliability improved** via direct Ticker table integration.

---

## Impact Highlights

- Approximately **3,500 unique equity tickers** currently managed.
- Over **3,300 entity enhancements** (parents/ADRs), and nearly **3,000 FactSet-standard descriptions adopted**.
- **Data linkage and normalization upgraded** (FactSet IDs, Bloomberg tickers, etc.).
- Manual, error-prone processes and server dependence **replaced by predictable, local workflows**.

---

## Challenges and Weaknesses

- Timeline exceeded original expectations; several emergent technical problems caused delays.
- Much of the validation and refactoring was dependent on single-person expertise, **increasing the risk profile**.
- Epic tracking and project communications **were not always clear or frequent**.
- Some intended features (e.g., **full US Options coverage**, broader client-specific adaptations) remain incomplete.
- **Knowledge transfer and documentation need improvement**, especially around legacy systems.

---

## Lessons Learned

- **Conservative timelines and regular communication** are necessary, especially for legacy migrations.
- **Technical debt and unclear dependencies** often become more apparent only during execution.
- **Validation and local testability should be prioritized** to avoid bottlenecks and risky releases.
- **Expertise needs to be replicated**; distributing system knowledge reduces risk and fosters team resilience.
- **Flexibility in planning is vital,** as many essential changes are discovered mid-project.

---

## Next Steps

- **Complete FIS replacement for US Options** and finalize all SecMaster dependencies.
- **Implement a more regular stakeholder update cadence** and clearer documentation practices.
- **Expand validation routines**, especially as new workflows and data sources are brought online.
- Continue **modernizing data pipelines and improving analytics outputs** for Data Science and clients.

---

## Conclusion

> While not without issues, the FAC project met its main targets: modernizing Ticker Resolution and reducing legacy risk, all with minimal business impact. These changes provide a reliable foundation for future enhancements to Alpha Theory's data management and analytics capabilities.
>
> **FAC is closed; FactSet/FIGI integration is operational.** The focus now moves to finishing remaining gaps and driving communication, reliability, and team knowledge forward.

---

