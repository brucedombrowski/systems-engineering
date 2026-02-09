# Phase 2: Implementation

## Purpose

Write compliant code and configuration that satisfies the requirements baseline, using only approved libraries and patterns.

## Inputs

- Requirements document (REQ-YYYY-NNN)
- Project instruction file (CLAUDE.md or equivalent)
- Applicable coding standards and constraints

## Process

1. **Load compliance context** — agent reads project instructions encoding standards and constraints
2. **Implement requirements** — write code that satisfies each REQ item
3. **Document compliance values** — annotate security-relevant constants with standard citations
4. **Write tests** — automated verification for testable requirements
5. **Human review** — engineer reviews implementation for correctness and compliance
6. **Commit** — atomic commit per logical unit of work

## Constraints

- Use platform-provided, validated libraries for security-critical operations
- No third-party dependencies for cryptographic or security functions
- Compliance-relevant configuration values must cite governing standard in comments
- Code must be compatible with target platform requirements

## Quality Criteria

- Every requirement from REQ document has corresponding implementation
- No security-critical operations use non-validated libraries
- Automated tests cover testable requirements
- Code compiles/runs without errors

## Agent Role

**Implementation Agent** (medium reasoning demand — balanced speed/quality)

## Standards

- NIST SP 800-53 SA-11: Developer Testing and Evaluation
- Applicable domain standards (per project)
