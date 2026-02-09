# Phase 4: Verification

## Purpose

Produce verification documents that map each requirement to its implementation evidence, creating a complete traceability chain for auditors.

## Inputs

- Requirements document (REQ-YYYY-NNN)
- Source code and configuration
- Test results

## Process

1. **Read requirements baseline** — load all REQ items
2. **Locate implementations** — for each requirement, find the source file, function, and line
3. **Record evidence** — file path, code excerpt, configuration value
4. **Assign verification method** — confirm it matches the REQ-specified method
5. **Generate verification matrix** — VER-YYYY-NNN document
6. **Human review** — engineer verifies the tracing is accurate and current
7. **Commit** — versioned artifact

## Output Format

Verification Document (VER-YYYY-NNN):

| Requirement | Evidence | File:Line | Method | Status |
|-------------|----------|-----------|--------|--------|
| REQ-1.1 | `[Aes]::Create()` call | Encrypt.ps1:42 | Inspection | PASS |
| REQ-2.3 | `$ITERATIONS = 100000` | Encrypt.ps1:15 | Inspection | PASS |

## Traceability Chain

```
Standard → REQ-X.Y → Source:Line → Test/Inspection → VER entry
```

Every requirement must have exactly one verification entry. Orphaned requirements (in REQ but not VER) or orphaned evidence (in VER but not REQ) are defects.

## Quality Criteria

- Every requirement from REQ has a corresponding VER entry (MIL-STD-498)
- File paths and line numbers are current, not stale (NIST SP 800-53 SA-11)
- Code excerpts match actual source
- Verification method matches REQ specification

## Agent Role

**Documentation Agent** (generates) + **Review Agent** (validates)

## Standards

- MIL-STD-498 A.5.19: Traceability
- NIST SP 800-53 SA-11: Developer Testing and Evaluation
- IEEE 29148: Requirements traceability verification
