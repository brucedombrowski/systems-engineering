# Pattern: BugReporter — auto-posted CUI-safe diagnostics on failure

## Problem

A development tool runs on an operator's workstation (Windows test laptop, JSL machine, etc.) while the dev side iterates on a different OS (Mac, Linux). When something fails on the runtime side, the dev side needs trace data — exit codes, environment, file inventory, exception type, terminal output — but can't reach in to grab it.

Two recurring failure modes the dev side keeps hitting:

1. **The operator can't easily share traces** — typing terminal output into chat is slow and error-prone; copy-paste loses formatting; uploading log files needs them to know where the file lives.
2. **CUI is everywhere in real error messages** — payload acronyms, IP addresses, internal file paths. Slapping the raw terminal output into a GitHub issue leaks sensitive content into the public repo.

Result: every diagnostic round-trip costs a manual upload, OR the dev side asks the operator to retype, OR (worst) the operator pastes raw output and now there's a CUI scrub to do.

Concrete instance that motivated this template (KuIP_Analyzer, 2026-05-28):

> Bruce: *"you should have the diagnose capability so that the next launch all the data you need gets put in a github issue."*
> 
> Followed by: *"we need to document this pattern as like BugReporter for all my projects, because i have to keep reteaching you this idea."*

## Solution

A two-piece pattern adopted across every project of this shape:

1. **`diagnostics.py` (or equivalent)** — a builder that composes a Markdown diagnostic report from **structured metadata only**, with any free-text fields passed through a CUI redactor before they're emitted. The output is safe to publish in any public repo by construction.

2. **`_main_wrapped()` (or equivalent)** — the entry point's outer try/except. Always writes the diagnostic to `output/diagnostic.txt`; optionally posts it as a `[diagnose]` GitHub issue via the platform's CLI (`gh` / `glab`). Gated by an `*_AUTO_POST_DIAGNOSE=1` env var that the GUI host sets when spawning the analyzer subprocess, so the operator never sees a prompt and never has to type anything.

### What goes into the report (CUI-safe by construction)

| Section | Content | CUI risk |
|---|---|---|
| Run metadata | timestamp, exit code, cwd, repo root | none |
| Environment | language version, OS, git HEAD, embedded-binary hashes | none |
| Input file inventory | per-file: size, sha256-short, exists?, search-path-tried | none (paths are dev machine paths, not payload data) |
| Discovery chain | how many files matched at each search location | none |
| Output file inventory | what got written, sizes, hashes | none |
| Exception | `type:` (class name), `message:` (redacted) | medium → CUI-redactor handles |
| Tail of run log | last 60 lines, redacted | medium → CUI-redactor handles |

### What CUI redaction looks like

Two regexes catch most of the surface area for ISS-payload work:

```python
_IPV4 = re.compile(r"\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b")
_PAYLOAD_NAME = re.compile(
    r"(payload(?:\s+acronym)?\s*[:='\"]+)([^'\"\s]{1,80})",
    re.IGNORECASE,
)

def redact(text: str) -> str:
    out = _IPV4.sub("<ip>", text)
    out = _PAYLOAD_NAME.sub(lambda m: m.group(1) + "<redacted>", out)
    return out
```

For other domains, swap the regexes:
- PII contexts: SSN, email, phone-number patterns
- Financial: account-number-shaped tokens
- Healthcare: MRN patterns

The principle is the same: **structured metadata passes through verbatim; free-text passes through `redact()`**.

### The wrapper

```python
def _main_wrapped() -> int:
    exit_code, exc_type, exc_msg = 0, "", ""
    try:
        exit_code = main()
    except SystemExit:
        raise
    except Exception as exc:
        import traceback
        exit_code = 1
        exc_type = type(exc).__name__
        exc_msg = "\n".join(traceback.format_exception_only(type(exc), exc)).rstrip()
        sys.stderr.write(traceback.format_exc() + "\n")
    finally:
        try:
            report = diagnostics.build_report(
                REPO_ROOT,
                exit_code=exit_code,
                exception_type=exc_type,
                exception_message=exc_msg,
            )
            (OUTPUT_DIR / "diagnostic.txt").write_text(report, encoding="utf-8")
            if os.environ.get("KUIP_AUTO_POST_DIAGNOSE", "").lower() in ("1", "true", "yes"):
                _post_diagnose_issue(report, exit_code)
        except Exception as exc:
            sys.stderr.write(f"[diagnose] could not write report: {exc}\n")
    return exit_code
```

### The auto-post

Uses the same `gh` / `glab` CLI path the existing per-launch counts-only issue uses. Soft-fails through every error so the diagnostic write never becomes a new failure mode. Title format: `[diagnose] <ProjectName> · <timestamp> · exit <N>`.

### GUI integration

When the operator runs the tool via a WinForms (or equivalent) host that spawns the analyzer as a subprocess, the host sets `*_AUTO_POST_DIAGNOSE=1` in the child's environment. Failure → issue posted automatically. Operator sees the WinForms status bar say "Analyzer failed (exit 1)" and can click on a "Console" tab; meanwhile the dev side already has the diagnostic on GitHub.

```csharp
new ProcessStartInfo {
    FileName = "python",
    Arguments = "-m src.main",
    Environment = {
        ["KUIP_NO_OPEN"] = "1",
        ["KUIP_AUTO_POST_DIAGNOSE"] = "1",
    },
    ...
}
```

## What goes wrong if you don't have this

- Dev cycle gets a ~5-minute round trip for every diagnostic: ask, wait, retype, scrub. Across a day of testing, that's hours.
- Operator pastes raw output → CUI in public issue → history-scrub cycle that breaks their clone with divergence.
- Errors that depend on environment state (file presence, hashes, version mismatches) never get caught because the dev side doesn't see what the runtime actually saw.

Saving the operator from having to type the trace is the win; saving the dev side from a manual scrub is the bonus.

## Files in this template

- `bug-reporter.md` (this doc)

When adopting in a new project:
1. Copy the `diagnostics.py` shape from `KuIP_Analyzer/src/diagnostics.py` — adjust the input-file inventory list to your project's actual search locations, and add domain-specific redactor regexes.
2. Wrap your entry point with the `_main_wrapped()` pattern.
3. In your GUI host (WinForms / Electron / whatever), set the `*_AUTO_POST_DIAGNOSE=1` env var when spawning the analyzer subprocess.
4. Verify the headless test path: trigger a deliberate failure, confirm the `[diagnose]` issue appears in your project's repo, confirm no CUI in the body.

## Maps to Phase 4 (Verification)

This is automated browser-engine-less inspection — every launch produces a verification artifact (the diagnostic report) that documents *what the system actually saw* at runtime. Pairs cleanly with the [headless-html-smoke-test](headless-html-smoke-test.md) template: that one catches JS regressions before push, this one captures runtime state when something escapes pre-push checks.
