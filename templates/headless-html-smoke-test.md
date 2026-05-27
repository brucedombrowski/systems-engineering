# Pattern: Headless-HTML smoke test for tools that emit a browser report

## Problem

A development tool generates an HTML report as its primary user-facing artifact (KuIP_Analyzer, ORBIT-RIFD-Status, dashboards, audit reports, …). The dev workstation is one OS (macOS, Linux), the production runtime is another (Windows + Edge). Bugs that surface only on the runtime — JS console errors, JSON parse failures, security-policy violations (`file://` origin restrictions), uncaught exceptions — slip past Python-side string checks and waste a full dev → push → pull → run round trip to detect.

Concrete instance that motivated this template (KuIP_Analyzer, 2026-05-27):
- Two bugs shipped in the same session — `<script type="application/json">` contents being HTML-escaped (broke `JSON.parse` in any browser), and `history.replaceState` throwing under Edge `file://` (file URLs are unique security origins). Both were invisible to the Python-side smoke test, which only ran `grep` against the generated HTML. Bruce caught both on Windows and reported "could not parse source-lines JSON" + "Unsafe attempt to load URL" — costing one full pull cycle per bug.

## Solution

Add a small **headless-Chromium** check that loads the generated HTML in a real browser engine, captures `console.error` / `console.warning` / page-level uncaught exceptions, and exits non-zero if any are present.

**Constraints honored:**
- Runtime stays free of new dependencies. Playwright is dev-only on the workstation, never enters the production install path.
- Check is opt-in via an env var so it doesn't slow down hot-loop iterations or stall CI environments without Chromium.
- Graceful degradation: if Playwright isn't installed, the check prints `SKIP` and continues — never blocks the main pipeline.

## Implementation

### One-time per workstation

```sh
pip3 install --user playwright
python3 -m playwright install chromium
```

Cost: ~120 MB Chromium download, one-time.

### Smoke-test script (Python, ~110 lines)

`tools/headless_check.py`. Single-purpose:

```python
"""Headless-browser smoke test for the generated HTML report."""
from __future__ import annotations
import argparse, sys
from pathlib import Path

def main(argv=None):
    p = argparse.ArgumentParser()
    p.add_argument("html")
    p.add_argument("--timeout", type=int, default=15000)
    args = p.parse_args(argv)

    html_path = Path(args.html).resolve()
    if not html_path.exists():
        print(f"[headless-check] FAIL: {html_path} missing")
        return 3

    try:
        from playwright.sync_api import sync_playwright, Error as PlaywrightError
    except ImportError:
        print("[headless-check] SKIP: install playwright + chromium to enable")
        return 2

    errors, warnings, page_errors = [], [], []
    with sync_playwright() as pw:
        browser = pw.chromium.launch(headless=True)
        page = browser.new_context().new_page()
        page.on("console", lambda m:
            (errors if m.type == "error" else
             warnings if m.type == "warning" else []).append(m.text))
        page.on("pageerror", lambda exc: page_errors.append(str(exc)))
        try:
            page.goto(html_path.as_uri(), timeout=args.timeout, wait_until="load")
            page.wait_for_timeout(500)
        finally:
            browser.close()

    issues = len(page_errors) + len(errors) + len(warnings)
    for e in page_errors: print(f"   pageerror • {e}")
    for e in errors:      print(f"   error     • {e}")
    for w in warnings:    print(f"   warning   • {w}")
    if issues:
        print(f"[headless-check] FAIL: {issues} issue(s)")
        return 1
    print(f"[headless-check] OK")
    return 0

if __name__ == "__main__":
    raise SystemExit(main())
```

### Wire into the main pipeline

In the entry-point script, gated by env var:

```python
def _run_headless_check_if_enabled(html_path):
    if os.environ.get("KUIP_HEADLESS_CHECK", "").lower() not in ("1", "true"):
        return
    subprocess.run([sys.executable, str(REPO_ROOT / "tools" / "headless_check.py"),
                    str(html_path)], capture_output=False, timeout=60)
```

### Dev workflow

```sh
# Standard dev iteration on Mac:
KUIP_HEADLESS_CHECK=1 KUIP_NO_OPEN=1 python3 -m src.main

# Output appends:
#  [headless-check] OK: findings.html loaded cleanly in headless Chromium.
```

Failure mode (real example):

```
[headless-check] 1 console.warning(s):
   warning   • Could not parse source-lines JSON: SyntaxError ...
[headless-check] FAIL: 1 issue(s) in findings.html
```

## Limits

- **Doesn't fully simulate the runtime browser.** Headless Chromium ≠ Edge + actual Windows. Edge-specific origin policy is replicated under headless, but other Windows-specific UX (focus stealing, native dialogs, SmartScreen) is not.
- **Catches runtime errors, not visual regressions.** If JS executes silently but renders the wrong layout, this check passes. Pair with manual visual review for layout-sensitive work, or extend with `page.screenshot()` diffs if needed.
- **CSP-strict pages need a browser context tweak.** If the page sets `Content-Security-Policy` that blocks inline scripts, the check needs to disable that — usually `bypassCSP=True` on the context.
- **Timeout sensitivity.** Default 15 s page-load timeout — bump for large reports or slow CI.

## Why it fits the framework

Maps cleanly to Phase 4 (Verification) — a verification method by **automated inspection**. For each "report renders without JS errors" implicit requirement, this is the evidence:

| Requirement | Evidence | Method |
|---|---|---|
| Generated HTML loads without console errors | `headless_check.py output/findings.html` exits 0 | Test (automated browser-engine inspection) |

If the project tracks verification artifacts (VER-YYYY-NNN), the headless-check exit code + output goes in as evidence for "report integrity" requirements.

## Variations seen / worth trying

- **Warnings-non-fatal mode** (`--no-warn-as-error`) for noisier libraries; keep default strict.
- **Multiple-URL run** for projects that emit several HTML artifacts (e.g. dashboard + per-payload pages).
- **`page.evaluate("() => window.someAssertions()")`** to embed application-level assertions inside the report and surface them via the same channel.
- **CI integration** by failing the build when this exit-code is non-zero.

## Files in this template

- `headless-html-smoke-test.md` (this doc)

When adopting, copy `headless_check.py` into the consumer project's `tools/` directory verbatim — it has no project-specific behavior. Wire it into whatever entry-point the project already has, behind whatever env-var naming convention that project uses.
