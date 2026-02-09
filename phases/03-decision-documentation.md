# Phase 3: Decision Documentation

## Purpose

Formally document design and policy decisions with traceable rationale. Government compliance requires documenting *why* approaches were chosen, not just *what* was implemented.

## Inputs

- Design choice or policy decision requiring documentation
- Options considered
- Applicable standards and constraints

## Process

1. **Identify decision** — recognize when a design choice warrants formal documentation
2. **Enumerate options** — minimum two alternatives considered
3. **Evaluate against criteria** — regulatory compliance, technical feasibility, licensing, risk
4. **Select and document** — generate formal decision memo
5. **Human approval** — engineer reviews and approves the decision
6. **Commit and log** — version-controlled artifact + GitHub issue

## Output Format

Decision Memoranda (DM-YYYY-NNN) using template-wrapper pattern:

```
DM-YYYY-NNN
├── Purpose: Why the decision was needed
├── Options Considered: Alternatives evaluated (minimum 2)
├── Decision: Selected approach
└── Rationale: Why, with regulatory references
```

## When to Create a Decision Memo

- Selecting an algorithm, library, or framework
- Choosing between implementation approaches
- Deviating from a standard's default recommendation
- Making a trade-off between competing requirements
- Any choice an auditor might question

## Quality Criteria

- At least two options considered
- Rationale references applicable standards
- Decision is unambiguous
- Document ID follows DM-YYYY-NNN convention

## Agent Role

**Documentation Agent** (medium reasoning demand — structured template following)

## Standards

- MIL-STD-498: Software Development and Documentation
