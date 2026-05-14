# Phase 0 — Tier 3 Windows Flight Node bootstrap

**Status:** Draft for validation. None of these steps have been executed on an actual Tier 3 Boeing laptop yet. Treat as a runbook to be executed and refined, not a finished SOP.

**Scope:** Track A from `PLAN.md` §6 — bringing up local AI development capability on a Boeing-issued Tier 3 Windows developer laptop, starting from a **baseline Windows load** (no PSI-stack tooling pre-installed). Tier 3 is permanently connected to the corporate network, internet, NASA VPN, and the enterprise Git host — Track A is a *data* airgap, not a network airgap (`PLAN.md` §1.5).

**The baseline install is delivered from a USB drive**, not from public installers fetched over the network. Every Tier 3 laptop gets the same versions, the install works regardless of network state, and IT can review one USB kit once instead of approving each download per laptop.

The four baseline artifacts on the USB:

1. **Python** (3.11+) installer
2. **Git for Windows** installer
3. **Ollama for Windows** installer
4. **The local model GGUF file** (Qwen2.5-Coder-14B Q4_K_M, ~8.5 GB)

Everything else (opencode, gh CLI, Node.js, custom slash commands, working repos) is the **agent-harness layer**, installed after the baseline is verified.

**Time estimate:** ~20–30 minutes for the baseline, another ~20–30 minutes for the agent-harness layer.

---

## Storage model — two external drives

Track A uses **two distinct external drives**, each with a clear role. Nothing about the work session is stored on Tier 3 internal storage (matching the Track B "host is disposable" principle — if a Tier 3 laptop has to be re-imaged, no work is lost).

| Drive | Purpose | Lifecycle |
|---|---|---|
| **Bootstrap USB** | Carries installers (Python, Git, Ollama) + the model GGUF + `bootstrap.ps1`. Used during initial setup. | Delivered physically once. Can be removed after Step 1 succeeds; kept on hand for re-bootstrapping or new laptops. |
| **External work drive** | Carries the working repo(s), all session outputs, audit logs, and opencode/AGENTS configuration. Always attached during a work session. | Plugged in when the engineer sits down to work, unplugged when done. Survives Tier 3 re-imaging. |

**Recommended external work drive:** USB-C SSD enclosure, ≥256 GB, USB 3.2 Gen 2. The work drive holds repos and session outputs; size depends on dataset volume, but 256 GB comfortably handles years of accumulated derived products from CSV-scale analysis work.

**Default Tier 3 internal storage usage in Track A:**

- Python, Git, Ollama **runtime binaries** — yes, internal (one-time install).
- Ollama **model files** — by default internal at `%USERPROFILE%\.ollama\models\`. Optionally redirected to the external work drive via `OLLAMA_MODELS` if the engineer wants Tier 3 fully disposable; cost is the engineer must attach the work drive before any AI usage (vs. ad-hoc local AI working without it).
- **Repos, outputs, audit, agent config** — never on internal storage. Always on the external work drive.

The remainder of this runbook assumes the work drive is mounted at `E:\` (substitute the actual drive letter on the laptop).

---

## Prerequisites — confirm with Boeing IT before starting

- Local administrator rights on the Tier 3 laptop.
- Tier 3 laptop is cleared for AI Coding Assistant (Ollama runtime). The Email Assistant project (`~/Email`, `brucedombrowski/Email`) is the existing precedent — its `launch.bat` installs Ollama via the same path. If that project ran on a given Tier 3 laptop, this runbook will too.
- A target repo with appropriate IP / CUI handling clearance for the work the engineer will do.
- A bootstrap USB drive (≥16 GB, USB 3.0+ preferred for model-file copy speed) prepared per **USB kit assembly** below.
- A non-CUI smoke-test dataset for first-run validation (no real CUI until the stack itself is verified).

If any prerequisite fails, **stop and resolve before proceeding** — do not work around IT policy.

---

## USB kit assembly (done once, on a separate trusted online machine)

Build the bootstrap USB on a non-Tier-3 machine that has internet access. The USB is delivered physically to the engineer afterward.

**USB layout:**

```
bootstrap-usb/
├── installers/
│   ├── python-3.11.x-amd64.exe        (from python.org/downloads/windows/)
│   ├── Git-2.x.x-64-bit.exe           (from git-scm.com/download/win)
│   └── OllamaSetup.exe                 (from ollama.com/download/windows)
├── models/
│   ├── manifests/qwen2.5-coder/...    (copied from assembly machine's
│   └── blobs/sha256-<hash>             ~/.ollama/models/ after `ollama pull`)
├── bootstrap.ps1                       (PowerShell installer — see Step 1)
└── README.txt                          (one-page how-to)
```

**To produce the model files for the USB:** on the assembly machine, run

```bash
ollama pull qwen2.5-coder:14b-instruct-q4_K_M
```

then copy `~/.ollama/models/manifests/` and `~/.ollama/models/blobs/` (the SHA-hash files referenced by the manifest) into the USB's `models/` directory, preserving the same subdirectory structure. **The exact file layout is a Phase 0 validation item** — Ollama's on-disk format is internal-undocumented and may change. Worst-case fallback: skip the GGUF copy and let the Tier 3 laptop `ollama pull` on first run, accepting the longer first-run time.

Hash the USB once assembled. Print the SHA-256 on a label affixed to the drive case.

---

## Step 1 — Baseline install (python, git, ollama, model) — from USB

Insert the bootstrap USB into the Tier 3 laptop. Open **PowerShell as Administrator** from the USB drive root.

Run the bootstrap script:

```powershell
.\bootstrap.ps1
```

What the script does (each substep is idempotent — safe to re-run):

1. **Install Python 3.11** from `installers\python-3.11.x-amd64.exe` if `python --version` does not report a 3.11+ install.
2. **Install Git** from `installers\Git-2.x.x-64-bit.exe` if `git --version` is absent.
3. **Install Ollama** from `installers\OllamaSetup.exe` if `ollama --version` is absent. (Most Tier 3 laptops will already have Ollama from the Email Assistant install — this step no-ops on those.)
4. **Stage the model** by copying `models\manifests\` and `models\blobs\` from the USB into `%USERPROFILE%\.ollama\models\`. Skip if the model is already present.

**Verify:**

```powershell
python --version          # should report 3.11.x
git --version             # should report 2.x
ollama --version          # should report a version
ollama list               # should list qwen2.5-coder:14b-instruct-q4_K_M
ollama run qwen2.5-coder:14b-instruct-q4_K_M "Briefly: what model are you?"
```

If `ollama list` does not show the model, the file-copy step did not land — re-check that the USB's `models/` directory structure matches `%USERPROFILE%\.ollama\models\` exactly. As a fallback, `ollama pull qwen2.5-coder:14b-instruct-q4_K_M` over the corporate network (~8.5 GB download) achieves the same end state.

---

## Step 2 — Verify connectivity and the local model

Track A is not network-airgapped; this step confirms the existing networking is in the expected state and the local model is reachable on loopback. **No firewall lockdown.**

```powershell
# Local model on loopback:
Invoke-WebRequest -Uri http://127.0.0.1:11434/api/tags
# Should return Ollama's model list JSON.

# Enterprise Git host (substitute your host):
Test-NetConnection -ComputerName <enterprise-git-host> -Port 443 -InformationLevel Quiet

# If NASA VPN is the route to internal resources, confirm the VPN client is connected.
```

If the local model does not respond on loopback, Ollama service isn't running — start it from the Start menu or restart the laptop.

---

## Step 3 — Baseline smoke test

```powershell
ollama run qwen2.5-coder:14b-instruct-q4_K_M
```

In the interactive prompt:

- Ask: *"Write a Python function that returns the sum of an array of integers."*
- Confirm the response is a coherent Python function.
- Watch Task Manager — `ollama.exe` should consume CPU and (if a GPU is present) GPU.
- Exit with `/bye`.

This is the **end of the baseline phase**. The laptop now has a working local LLM and the four core tools (python, git, ollama, model). Everything beyond this point is the agent-harness layer.

---

## Step 4 — Agent harness layer (opencode + gh + Node)

The baseline above is sufficient for ad-hoc local-AI usage. The agent-harness layer adds the structured agent workflow described in `PLAN.md` §5 — opencode (the terminal-native agent), GitHub CLI for issue management, and Node.js (opencode's runtime if installing via npm).

Add these installers to the same USB kit alongside the baseline, then run from PowerShell as Administrator:

```powershell
.\installers\node-vXX.X.X-x64.msi
.\installers\gh_x.xx.x_windows_amd64.msi
npm install -g opencode-ai
```

(Or, since Tier 3 is internet-connected, fall back to `winget install OpenJS.NodeJS.LTS` and `winget install GitHub.cli` if the USB doesn't carry these — pick one approach and document the choice in `README.txt`.)

Configure opencode to use the local model — create `%USERPROFILE%\.config\opencode\config.json`:

```json
{
  "provider": {
    "ollama": {
      "type": "openai-compatible",
      "url": "http://127.0.0.1:11434/v1",
      "models": {
        "qwen2.5-coder:14b-instruct-q4_K_M": { "name": "qwen2.5-coder-14b" }
      }
    }
  },
  "model": "ollama/qwen2.5-coder-14b"
}
```

Schema follows opencode's current docs — **verify against https://github.com/sst/opencode** before committing the file.

---

## Step 5 — Clone working repo to the external work drive

Attach the external work drive (assume `E:\`). Repos and all session work product live here, not on Tier 3 internal storage.

```powershell
mkdir E:\PSI-Work\repos
mkdir E:\PSI-Work\outputs
mkdir E:\PSI-Work\audit
cd E:\PSI-Work\repos
gh auth login              # first time only; choose enterprise hostname if applicable
gh repo clone <owner>/<repo>
cd <repo>
```

In the repo root, create or update `AGENTS.md` to give opencode project-specific context (analogous to `CLAUDE.md` in the Claude Code workflow). Start with whatever convention the repo already uses; refine as you find the agent missing context.

Optionally configure opencode's workspace and audit-log paths to point at the external drive in `%USERPROFILE%\.config\opencode\config.json`:

```json
{
  "workspace": "E:\\PSI-Work\\repos",
  "audit_log": "E:\\PSI-Work\\audit"
}
```

---

## Step 6 — Issue-as-contract slash commands

Per `PLAN.md` §5.1 and §5.2:

- `/task <description>` — calls `gh issue create` first, captures the issue number, then proceeds with code changes referencing it.
- `/needs-update <area> <reason>` — files an issue tagged `agent-blocked` and stops, returning the decision to a human.

opencode reads custom commands from `%USERPROFILE%\.config\opencode\commands\`. Author `task.md` and `needs-update.md`. Exact syntax follows opencode's slash-command docs.

---

## Step 7 — Full agent smoke test

Confirm the external work drive is attached, then:

```powershell
cd E:\PSI-Work\repos\<repo>
opencode
```

In the opencode session:

1. **Read-only sanity check:** ask the agent to list files and read a known script. Confirm `ollama.exe` consumes resources during the response.
2. **Edit + commit:** ask the agent to make a trivial whitespace fix in a comment, commit it locally, show `git log`.
3. **Issue creation:** `/task Test the bootstrap flow`. Confirm an issue is created on the enterprise Git host.
4. **Failure path:** ask the agent to do something out of scope (e.g., reach a public URL not configured for it). Confirm it does not silently succeed.

---

## Step 8 — Capture as a reproducible script (Phase 1 deliverable)

The runbook should ultimately collapse into two PowerShell entrypoints on the USB:

- `bootstrap.ps1` — does Step 1 end-to-end. Already referenced above; Phase 0 turns the inline description into a real script.
- `agent-harness.ps1` — does Step 4 end-to-end.

Both should:

- Idempotently detect already-installed components and skip them.
- Log every action with timestamps to `E:\PSI-Work\audit\bootstrap-<timestamp>.log` (or fall back to a temp dir if the work drive isn't attached during bootstrap).
- Refuse to run if local admin rights aren't present.
- Print a clear next-step prompt at completion.

The USB itself is then the reproducible artifact — **IT-reviewable once, deployable many times.**

---

## Open assumptions to validate

1. **Ollama's on-disk model layout.** Step 1.4 copies manifests + blobs from USB into `%USERPROFILE%\.ollama\models\`. Ollama's internal layout is undocumented and could change between versions. Validate on the actual Tier 3 environment; fall back to `ollama pull` if needed (slower first run, but unambiguous).
2. **Local admin rights.** Steps 1 and 4 require admin. If admin is unavailable, IT runs the install or grants an elevation window.
3. **Ollama Windows GPU support.** Ollama on Windows uses CUDA (NVIDIA) or ROCm/Vulkan (AMD) automatically. Intel iGPU-only Tier 3 is CPU-bound — validate token throughput; consider `llama.cpp` with Vulkan or DirectML as an alternative runtime if Ollama's CPU-only mode is unusable.
4. **opencode for Windows.** opencode is actively developed; install method and config schema may have changed since this draft. Verify against the current `sst/opencode` README.
5. **USB installer signing.** Boeing IT may require signed installers; the python.org, git-scm.com, and ollama.com signed installers should pass, but confirm before staging.

---

## What this runbook does **not** cover (yet)

- USB kit assembly automation (Phase 1 — `assemble-usb.ps1` on the trusted online build machine).
- Anonymizer setup and the Equivalent Node side of the workflow (separate doc).
- LoRA fine-tuning workflow (Phase 2+, separate non-CUI training machine).
- CUI / PII audit tooling for the public GitHub repo (separate workstream).
- Per-project `AGENTS.md` content patterns (project-specific).

Each gets its own sibling doc as it lands.
