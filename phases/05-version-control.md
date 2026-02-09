# Phase 5: Version Control & Interaction Traceability

## Purpose

Maintain auditable records of all changes (git) and all human-agent interactions (GitHub issues), providing the evidence of process that compliance frameworks require.

## Git as Audit Trail

### Commit Discipline

- One atomic commit per logical unit of compliance work
- Commit messages describe the compliance-relevant change
- Reference GitHub issue numbers where applicable
- Never silently amend published commits

### Semantic Versioning

Follow [semver.org](https://semver.org/):
- **MAJOR** (X.0.0): Structural changes (new sections, reorganization)
- **MINOR** (0.X.0): New content (features, case studies, methodology additions)
- **PATCH** (0.0.X): Fixes (typos, corrections, formatting)

### Changelog

Follow [Keep a Changelog](https://keepachangelog.com/):
- Update CHANGELOG.md with every commit
- Categories: Added, Changed, Fixed, Removed, Security
- Reference issue numbers
- Keep [Unreleased] section at top

### Release Process

1. Move [Unreleased] items to new version heading
2. Update comparison links
3. Commit: "Release vX.Y.Z"
4. Tag: `git tag -a vX.Y.Z -m "description"`
5. Push: `git push origin main --tags`

## GitHub Issues as Interaction Log

### Label Scheme

| Label | Meaning | Created by |
|-------|---------|------------|
| `human-prompt` | Human directive to agent | Agent (logging human's instruction) |
| `agent-output` | Agent findings or deliverables | Agent |
| `decision` | Design/process decision with rationale | Either |
| `critical` | Severity: blocks progress | Either |
| `minor` | Severity: non-blocking | Either |

### When to Create Issues

- Every human directive to an agent
- Every agent analysis or deliverable
- Every design or process decision
- Every review finding

### Audit Queries

```bash
# All human directives
gh issue list --label human-prompt

# All agent deliverables
gh issue list --label agent-output

# All decisions
gh issue list --label decision
```

## Standards Alignment

| Control | Standard | How Addressed |
|---------|----------|--------------|
| CM-3 | NIST SP 800-53 | Git commit log as change record |
| AU-3 | NIST SP 800-53 | GitHub issues as audit log |
| CM-8 | NIST SP 800-53 | Semantic versioning for component identification |

## Agent Role

**All agents** — every agent logs interactions. **Project Setup Agent** maintains versioning infrastructure.
