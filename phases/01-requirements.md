# Phase 1: Requirements Capture

## Purpose

Extract, structure, and validate requirements from governing standards. Transform regulatory text into machine-readable, traceable artifacts.

## Inputs

- Governing standards (NIST SP, FIPS, IEEE, CFR, domain-specific)
- Project scope and applicability statement
- Organizational compliance posture

## Process

1. **Identify applicable standards** — determine which publications, regulations, and frameworks govern the project
2. **Extract requirements** — AI agent reads standards and generates structured requirements
3. **Classify requirements** — mandatory (SHALL) vs recommended (SHOULD) vs optional (MAY) per RFC 2119
4. **Assign verification methods** — inspection, test, or analysis for each requirement
5. **Human review** — engineer verifies interpretation accuracy and completeness
6. **Baseline** — commit reviewed requirements as versioned artifact

## Output Format

```json
{
  "document_id": "REQ-YYYY-NNN",
  "title": "Project Requirements Specification",
  "version": "1.0",
  "date": "YYYY-MM-DD",
  "categories": [
    {
      "name": "Category Name",
      "standard": "NIST SP 800-132",
      "requirements": [
        {
          "id": "REQ-X.Y",
          "standard": "NIST SP 800-132",
          "section": "Section 5.1",
          "text": "The system SHALL...",
          "priority": "mandatory",
          "verification": "inspection"
        }
      ]
    }
  ]
}
```

## Quality Criteria

- Every requirement has a unique, stable ID (IEEE 29148)
- Every requirement cites a specific standard and section
- No requirements from cited standards are omitted
- RFC 2119 language used correctly
- Verification method is appropriate for the requirement type

## Agent Role

**Requirements Agent** (high reasoning demand — use strongest available model)

## Standards

- IEEE 29148: Requirements Engineering
- RFC 2119: Requirement Level Keywords
