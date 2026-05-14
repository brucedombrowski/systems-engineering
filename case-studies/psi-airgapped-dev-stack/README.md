# PSI Airgapped Dev Stack — case study

A worked example of applying this systems-engineering framework to a specific real-world project: bringing AI-assisted development capability to a CUI-bearing PSI workflow on Boeing-issued Tier 3 Windows hardware, with a bounded escalation path to a frontier model via anonymizers.

## Documents

- **[PLAN.md](PLAN.md)** — Full plan and design. Source of truth for the project framing, constraints, and locked decisions. Uses **Flight Node** (Tier 3 + CUI) and **Equivalent Node** (MacBook + frontier model) vocabulary mapped onto the ISS payload Flight Unit / Equivalent paradigm.
- **[PHASE-0-WINDOWS-BOOTSTRAP.md](PHASE-0-WINDOWS-BOOTSTRAP.md)** — Concrete bootstrap runbook for the Tier 3 Windows Flight Node, using a bootstrap USB (installers + model) and an external work drive (repos + outputs + audit). Draft pending validation on actual hardware.

## How this case study uses the parent framework

- **Phase 1 — Requirements.** `PLAN.md` captures binding constraints (CUI locality, Tier 3 hardware envelope, MIT-license posture, NPR 2210.1E compliance, two-human workflow) as the structured-requirements capture for this project.
- **Phase 2 — Implementation.** `PHASE-0-WINDOWS-BOOTSTRAP.md` is the first implementation runbook. Deterministic Python scripts and anonymizers (per `PLAN.md` §5.3, §5.6) land in later phases.
- **Phase 3 — Decision documentation.** Key decision records are embedded in `PLAN.md`: §4.3 (model choice), §4.5 (compute requirements), §6 (deployment tracks A and B), §2.6 (open-source posture).
- **Phase 4 — Verification.** Deferred to Phase 0 validation on actual Tier 3 hardware.
- **Phase 5 — Version control & traceability.** This commit is the entry point.

## Status

Active design. Plan content is stable on the locked decisions (Flight Node / Equivalent Node frame, Tier 3 hardware fixed, MIT licensing, anonymizer escalation). Bootstrap runbook is a draft pending validation.
