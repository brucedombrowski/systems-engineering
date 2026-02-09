# Systems Engineering Process

A reusable systems engineering process framework for AI-assisted development projects. Defines *how* work is done — phases, standards, traceability, and process artifacts.

## Relationship to Other Repos

| Repo | Defines | Scope |
|------|---------|-------|
| **systems-engineering** (this) | *How* — process, phases, standards, ceremonies | Process framework |
| [ai-agents](https://github.com/brucedombrowski/ai-agents) | *Who* — agent roles and templates | Agent definitions |
| [Scrum](https://github.com/brucedombrowski/Scrum) | *When* — sprint cadence and orchestration | Project management |
| [WhitePaper](https://github.com/brucedombrowski/WhitePaper) | *What* — academic documentation | Research paper |

## Process Phases

### Phase 1: Requirements Capture
- Identify governing standards (NIST, FIPS, IEEE, domain-specific)
- Generate structured requirements in machine-readable format (JSON)
- Use RFC 2119 keywords (SHALL, SHOULD, MAY)
- Human reviews AI-generated requirements for accuracy and completeness

### Phase 2: Implementation
- Code within constraints defined by requirements
- Use platform-provided libraries for security-critical operations
- Agent operates within CLAUDE.md / project instruction context
- Compliance-relevant values documented with standard citations

### Phase 3: Decision Documentation
- Formal decision memoranda for design choices
- Template-wrapper pattern for consistency
- Minimum two options considered per decision
- Rationale with regulatory references

### Phase 4: Verification
- Map each requirement to implementation evidence
- Verification matrix: requirement → file → line → method
- Methods: inspection, test, analysis
- Cross-reference integrity between REQ, VER, DM, and source

### Phase 5: Version Control & Interaction Traceability
- Git as audit trail (NIST SP 800-53 CM-3)
- GitHub issues as structured interaction log
- Labels: `human-prompt`, `agent-output`, `decision`
- Semantic versioning (semver.org)
- Changelog (keepachangelog.com)

## Standards Framework

### Process Standards
| Standard | Application |
|----------|------------|
| IEEE 1028 | Software Reviews and Audits |
| IEEE 29148 | Requirements Engineering |
| MIL-STD-498 | Software Development and Documentation |
| ISO/IEC 25010 | Software Quality |
| Scrum Guide | Iterative delivery framework |

### Security & Compliance Standards
| Standard | Application |
|----------|------------|
| NIST SP 800-53 | Security and Privacy Controls |
| NIST SP 800-171 | Protecting CUI in Nonfederal Systems |
| NIST SP 800-53 AC-5 | Separation of Duties |
| NIST SP 800-53 SA-11 | Developer Testing |
| NIST SP 800-53 CM-3 | Configuration Change Control |
| NIST SP 800-53 AU-3 | Audit Logging |

## Traceability Model

```
Standard → Requirement → Implementation → Test → Evidence
    ↕            ↕              ↕            ↕        ↕
  Citation    REQ JSON      Source code   Test case  VER doc
    ↕            ↕              ↕            ↕        ↕
  BibTeX     GitHub issue   Git commit   CI/scan   Attestation
```

Bidirectional traceability: auditors can start from any point and trace forward or backward to any other point.

## Artifacts

| Artifact | Format | Convention |
|----------|--------|------------|
| Requirements | JSON | REQ-YYYY-NNN |
| Decision Memoranda | LaTeX | DM-YYYY-NNN |
| Verification Documents | LaTeX/JSON | VER-YYYY-NNN |
| Traceability Matrix | JSON | mapping.json |
| Changelog | Markdown | CHANGELOG.md |
| Process Log | Markdown + GitHub Issues | PROCESS.md + issues |

## Key Principles

1. **Review over authoring**: AI drafts, human reviews. The engineer's role is reviewer, not author.
2. **Separation of duties**: Reviewers cannot modify what they audit (NIST SP 800-53 AC-5).
3. **Everything is traceable**: Every artifact traces to a standard, every change traces to a decision, every decision traces to a human directive.
4. **Process refines iteratively**: The methodology itself evolves during active development, documented in the issue trail.
5. **Cross-project learning**: Patterns from one project propagate to others via agent instruction ingestion.
6. **Model-agnostic**: The process is independent of any specific AI vendor or model.

## Applicability

This process framework applies to any domain requiring documented engineering rigor:

- Federal information security (NIST, FIPS, CUI)
- Safety-critical systems (DO-178C, IEC 61508)
- Information-critical systems (HIPAA, SOX)
- CAD and construction engineering
- Hardware/software integration
- Any project where auditors need evidence of process

## Status

Active research. This framework is being refined iteratively across multiple concurrent projects. See the [WhitePaper](https://github.com/brucedombrowski/WhitePaper) for academic documentation of the methodology.
