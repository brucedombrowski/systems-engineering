# PSI Airgapped Dev Stack — Plan

**Status:** Draft for review (2026-05-14). No code written yet.
**Owner:** Bruce Dombrowski
**Audience:** Next PSI engineer who picks this up — start here before reading any script.

---

## 1. What this is

An airgapped, open-source equivalent of the Claude Code workflow that Bruce uses today. The intent is **not** to clone Claude Code feature-for-feature — local open models are meaningfully weaker than frontier hosted models, so the design compensates by pushing intelligence into **deterministic scripts that wrap existing documented desk instructions**, and using the local LLM for orchestration, drafting, and analysis narration.

The deployment vehicle is a **two-USB pattern**:

1. **USB 1 (tools)** — a live Ubuntu image carrying opencode, Ollama, the chosen model(s), the agent harness configuration, the script libraries, the local reference corpus, **and the target Git repo(s) pre-cloned onto the USB**. The live OS **never connects to the internet during a work session**. All online work — repo clone, model pull, wheel mirror, reference sync — happens during the **USB build phase** on a separate, trusted, internet-connected machine. The build phase is the only network event in the lifecycle.
2. **USB 2 (data)** — connected *after* the host has booted from USB 1. Carries the CUI-or-equivalent dataset to be analyzed (typically CSV files; could also be Excel, JSON, parquet, log dumps). Mounted with controlled write paths so derived products land in known locations with appropriate markings.

The canonical task shape is the same as what Bruce's existing private repos already do: **given an analysis script library and a new dataset, update or extend the scripts to answer a specific question, run them, and produce a marked output.** The local agent's job is to read the desk instruction, decide which script to run or modify, invoke it, and narrate the result — with the human triaging anything outside the scripted scope via `agent-blocked` issues.

## 1.5 Conceptual frame — Flight Unit and Payload Equivalent

The two-machine architecture in this plan maps directly onto a paradigm PSI engineers already use every day: the **Flight Unit / Payload Equivalent** model from ISS hardware development. Adopting that vocabulary is not analogy for analogy's sake — it makes the audit story and the development discipline obvious to anyone who has worked an ISS payload.

| ISS hardware practice | This airgapped dev stack |
|---|---|
| **Flight Unit** — on orbit, irreplaceable, carries real mission data; every change is audited and verified before it gets there | **Tier 3 laptop** — airgapped, carries CUI; the only machine where CUI is allowed to live, every commit and output audited |
| **Payload Equivalent** — ground-based replica, free to experiment on, no flight consequences; used for development and anomaly investigation | **Bruce's MacBook + Claude Code** — connected, runs frontier model; used for problem-shape diagnosis on anonymized data and for non-CUI development |
| **Telemetry / downlinked anomalies** — what comes off the Flight Unit for diagnosis on the Equivalent | **Anonymized edge cases** — what crosses the airgap from the Flight Node to the Equivalent Node, with CUI stripped (§5.6) |
| **Procedures / commands developed on Equivalent, uplinked to Flight** | **Code fixes developed against anonymized cases on the Equivalent Node, applied to the CUI-bearing scripts on the Flight Node** |
| **Pre-flight verification** — Equivalent runs the same procedures the Flight Unit will, so changes are de-risked | **Regression test suite** — anonymized edge cases become permanent unit tests (§5.6); every future change runs against them on the Flight Node before release |

Concrete benefits to using this framing:

1. **Familiar vocabulary.** Payload engineers don't need a new mental model — they already operate two-system development daily.
2. **Audit story is immediate.** The Equivalent-stays-separate-from-Flight discipline already exists in PSI practice; the airgapped stack inherits its conventions.
3. **Naming.** From here on, this plan uses **Flight Node** for the Tier 3 + CUI laptop and **Equivalent Node** for the MacBook + frontier-model environment. Any tooling we build should use the same names.
4. **OS asymmetry as defense-in-depth.** The Flight Node runs **Windows** (Boeing-issued Tier 3); the Equivalent Node runs **macOS**. Different platforms by construction means binary tools don't trivially move between them, and any script that crosses must be deliberately cross-platform. The asymmetry also matches Bruce's stated preference for *developing on Mac and deploying on Windows* — the discipline of writing code that works on both platforms catches bugs that a same-platform workflow would miss.

## 2. Why this exists

- Boeing PSI work often must run on networks where hosted AI services are unreachable or prohibited.
- Bruce's current workflow leans on Claude Code for: clarifying desk instructions, drafting deterministic Python tooling, filing GitHub issues per task, and elevating analysis output. Losing all of that on a closed network is a regression.
- A portable, reproducible, airgapped stack means the same workflow survives the network boundary, with an auditable trail (every action either lives in a Git commit or in a GitHub issue).
- It also redefines the human role on the loop: the PSI engineer's primary work becomes **managing the issue backlog and reviewing agent output**, not hand-coding everything or chasing developer teammates for updates.

## 2.5 Success metric — script quality is the goal

**The ultimate goal is the quality of the deterministic Python script library that this workflow produces.** Everything else — the USB, the model, the agent harness — is scaffolding around that artifact.

Concretely, the system is succeeding when:

- A PSI engineer can hand a new dataset (USB 2) to the stack and get back analysis output produced by a script that another engineer can read, audit, test, and trust.
- The scripts in `repos/<project>/scripts/` would pass code review on their own merits — clear naming, type hints, docstrings that explain *why* the logic is the way it is (per Bruce's documentation practice), unit tests where the logic is non-trivial, and no LLM-generated noise.
- A reviewer can trace any output back to a specific commit, a specific script, a specific desk-instruction section, and a specific issue.
- Agent involvement leaves no signature in the final code: no boilerplate apologies, no over-engineered abstractions, no defensive code for impossible states, no comments restating what the code already says.

This metric is the lens for every other decision in this plan. If a choice makes the agent more impressive but the resulting scripts harder to review, reject the choice.

## 2.6 Open source posture — MIT-licensed payload software

The payload software developed against this stack — deterministic scripts, anonymizers, harness configurations, anonymized test fixtures — is released as **MIT-licensed open source**. This is a stance, not an afterthought, and it shapes several decisions.

**What it means in practice:**

- Every PSI-developed artifact in the script library carries an SPDX `MIT` identifier and a top-level `LICENSE` file.
- Every dependency must be license-compatible with MIT redistribution. The recommended stack already satisfies this:
  - opencode — MIT ✓
  - Ollama — MIT ✓
  - llama.cpp — MIT ✓
  - **Qwen2.5 / Qwen2.5-Coder — Apache 2.0** (MIT-compatible) ✓
  - DeepSeek-Coder, DeepSeek-V3 — MIT ✓
  - vLLM — Apache 2.0 ✓
  - **Llama 3.x family — Llama Community License** (has restrictions; *not* straightforwardly redistributable). Avoid as the default model family for that reason; document the tradeoff at any decision point that would require it.
- **Model weights are licensed separately from code.** A LoRA adapter fine-tuned on PSI data is a derivative work of the base model and inherits its license. Code that *uses* a model is MIT-licensable even if the model itself isn't.
- **No CUI in the public repository, ever.** Only anonymized or synthetic fixtures and the anonymizers themselves are committed. The anonymizer (§5.6) is the gate; its job is to ensure nothing it emits is CUI-bearing.

**Why this matters:**

- **Reuse.** PSI is not the only organization with this problem. NASA partners, other prime contractors, academic groups, and downstream missions can build on the same scaffolding. MIT removes the legal friction.
- **Quality pressure.** Open source means external review. The "script quality is the goal" metric (§2.5) effectively gains a third reviewer: the broader community.
- **Continuity.** When PSI engineers move on, the work doesn't get locked behind a corporate boundary. The next maintainer — inside Boeing or out — has the full artifact.
- **Aligns with the broader trend.** ISS payload software is increasingly developed and released open. This plan fits that trajectory rather than fighting it.

**Policy basis — NASA's Open Data Initiative and NPR 2210.1E:**

The open-source posture aligns with NASA's authoritative policy framework, which is the right reference for any PSI-adjacent payload software:

- **NPR 2210.1E — Release of NASA Software.** Defines Open Source Software, requires an OSI-approved license, sets contribution limits, and explicitly prohibits: *"All code contributions shall not contain export-controlled data, CUI, restricted computer software, or other type of data upon which restrictions are applicable."* The anonymizer-as-gate principle in §5.6 is the operational mechanism that satisfies this prohibition continuously, not just at release.
- **NASA Software Engineering Handbook SWE-149 — Open Source Conditions.** Translates NPR 2210.1E into engineering-level practice (license selection, third-party contribution rules, release approval workflow).
- **NASA Open Data Initiative.** The broader institutional stance: NASA defaults to making code and data freely available unless a specific restriction applies. PSI payload software fits comfortably under that default.

Citations and direct URLs are kept in the reference memory `reference_nasa_open_source.md` so they stay current and don't rot inline in this plan.

**Public GitHub hosting with continuous CUI/PII audit:**

The public artifact lives on **public GitHub**, not an internal mirror. The repository contains **no CUI by construction** (anonymizer-as-gate, §5.6), and it is **routinely audited** for CUI, PII, and other restricted-data leaks as an ongoing process — not a one-time pre-publication check. The audit posture matters because:

- NPR 2210.1E's contribution prohibition is continuous, not a release gate. Any subsequent commit could in principle introduce a leak.
- Anonymizer bugs are a real risk. Periodic scanning catches what slipped through.
- Auditability of the audit itself — when scans ran, what they found, what was remediated — is part of the project's evidence trail.

Specific scanning tools and cadence (gitleaks-style pattern scanning, PII regex sweeps, manual review of fixture-file changes) are a Phase 0+ workstream documented separately.

**Implication for §4 dependency choices:** prefer Apache 2.0 / MIT / BSD-licensed model families. The default model recommendation in §4.3 (Qwen2.5-Coder) is deliberately Apache 2.0 to keep MIT redistributability clean downstream.

## 2.7 Risk model — human-in-the-loop is the feature, not the cost

The workflow described in this plan — two repos, two humans copying issue text between them, anonymizers, regression tests on every fix, agent-blocked escalation, supervised CUI handling — has a lot of moving parts. It looks like friction. It is friction, **deliberately**, and the case for accepting it rests on a specific risk framing.

**Scope of this stack is AI-assisted development of data analysis tools, not critical flight software.** Inside that scope, the blast radius of an AI agent making a wrong call is bounded:

- **Worst plausible outcome from a bad agent suggestion:** bad code lands in the script library, the developer notices it in review, the regression test suite catches it, or both. The change is reverted. Cost: minutes.
- **Worst plausible outcome from a bad agent action on Tier 3:** misconfigured environment, corrupted local state, runaway process. Cost: a Windows reboot, or at worst a Boeing IT re-image. Re-image is annoying, not expensive — the work product lives in Git, the data lives on the data USB, neither is destroyed. The whole laptop is replaceable in the time it takes to push the image again.
- **Worst plausible outcome from CUI mishandling:** this is the only failure mode that would actually hurt, which is why every CUI-touching step is human-gated by construction (§5, §7) and audited continuously (§2.6).

Compare to AI assistance in **critical flight software**: a bad suggestion that escapes review could ground a payload, damage a satellite, or cost mission-time that cannot be recovered. The blast radius is millions of dollars and irreversible. The same workflow shape would still need humans in the loop *more* tightly, not less.

The plan therefore takes two stances:

1. **Within the data-analysis-tool scope: accept the friction.** Two repos and human copy-paste look inefficient compared to a fully-automated pipe, but the apparent inefficiency *is* the audit story. The cost of friction is low (minutes per task) and the cost of getting it wrong unsupervised is large (CUI leak, regression that wastes a downstream engineer's day).
2. **Outside the data-analysis-tool scope: the workflow needs more gates, not fewer.** If this stack is ever extended to support code that runs on a flight unit or drives an operational system, every gate in this plan stays and additional ones get added. Do not generalize the friction away.

**The general principle: human-in-the-loop is the feature, not the friction to be optimized out.** Optimization targets are *quality of agent output* and *clarity of human review* — not *reduction of human touchpoints*.

## 3. Non-goals

- Not a replacement for Claude Opus on novel reasoning. Don't ask the local model to design new systems from scratch.
- Not a general-purpose corporate Linux distro. Single-purpose: drive deterministic PSI tooling under human supervision.
- Not internet-connected during work sessions. Bootstrap-only network access.
- Not a Boeing IT-blessed image yet. That review is a separate workstream.

---

## 4. Recommended stack

### 4.1 Agent harness — `sst/opencode`

- Repo: `github.com/sst/opencode`
- License: MIT
- Why: terminal-native, first-class Ollama provider, supports custom slash commands and an `AGENTS.md` convention that maps cleanly onto Claude Code's `CLAUDE.md` + skills model.
- Alternatives considered:
  - `aider` — more conservative, edit-focused; weaker as a general agent harness.
  - `continue.dev` — IDE-bound; less useful in a terminal-driven PSI workflow.

### 4.2 Local model runtime — Ollama

- Why: simplest to install, single binary, model files are portable `.gguf` blobs that move across the air gap cleanly.
- Alternatives if multi-user serving becomes necessary: `vLLM` or `llama.cpp` server. Single-engineer-per-USB doesn't need this.

### 4.3 Model — Tier 3 laptop, local only, maximum extraction

**Binding constraints (not tunable):**

- The developer node is a Boeing-issued **Tier 3 Windows developer laptop** (Intel i7 H-series, 32 GB RAM, 6–8 GB VRAM dGPU, Windows 11 Pro/Enterprise on a Boeing IT-managed image). Hardware *and OS* are fixed — this is not a generic-laptop live-Linux-USB target.
- **CUI data must never leave this machine.** It cannot be sent to a hosted frontier model (Bruce's MacBook + Claude Code is not in scope for CUI work — that is a separate, non-airgapped workflow for non-CUI tasks).
- The local model on this Tier 3 laptop is the **only** AI capability with direct access to CUI. A **bounded escalation path** exists for hard problems via anonymizers (see §5.6) — the *shape* of an edge case can be genericized and sent to frontier, while the *data* stays on Tier 3 — but the local model has to be strong enough to handle the bulk of the work unaided.

Given those constraints, the goal is to **extract maximum capability from Tier 3 hardware** using aggressive open-source inference techniques. Model selection is not "pick by leaderboard" — it is "find the largest, strongest model that runs usably on this exact hardware envelope, then push it harder with every available technique."

**Recommended baseline (Phase 0 starting point):**

- **Qwen2.5-Coder-14B-Instruct, Q4_K_M (~8.5 GB)** — fits 8 GB VRAM with KV-cache offload to system RAM. Strong coding ability, mature tool-use, well-supported by every runtime.

**Stretch techniques to push Tier 3 toward its real ceiling:**

1. **Better quantization.** IQ3_M / IQ3_XS quantization lets **Qwen2.5-Coder-32B** (~13 GB at IQ3_M) run on a Tier 3 laptop with partial CPU offload at 5–15 tok/s. Quality is better than 14B Q4. This is the single biggest accessible upgrade.
2. **Reasoning-tuned variants.** **QwQ-32B-Preview** or **DeepSeek-R1-Distill-Qwen-32B** at IQ3 — slower per token but markedly stronger on multi-step reasoning. Route hard issues to these models selectively.
3. **Speculative decoding.** Pair Qwen2.5-Coder-1.5B (draft) with the 32B target — net 1.5–3× speedup on the larger model with no quality cost.
4. **KV cache quantization (Q4_0 / Q8_0 cache).** Halves cache memory pressure. Enables longer context or larger models in the same VRAM envelope.
5. **Domain fine-tuning via LoRA.** Train a LoRA adapter on PSI-specific docs, code patterns, and historical issue/PR pairs (training happens on a separate non-CUI machine using non-CUI data; only the adapter ships to Tier 3). Typically a **10–20 point quality bump on in-domain tasks** for a few GPU-hours of training — the single largest controllable lever after model choice.
6. **Better runtime than Ollama.** Direct `llama.cpp` or `vLLM` (if VRAM permits) can produce 20–50% more throughput than Ollama on the same model. Ollama wins on ergonomics; raw runtimes win on capability extraction. Worth a Phase 0 bake-off.

**Realistic ceiling on Tier 3 with all techniques applied: ~60–65% of Opus 4.7 quality on coding tasks.** This is the honest answer to "approximate the frontier." The gap is bounded by:

- The open-weight model itself trails the closed frontier by 1–2 generations.
- The 6–8 GB VRAM envelope caps which open-weight models can run usably.
- No amount of inference cleverness fully closes either gap.

The workflow design (§5: two-human, issue-as-contract, deterministic scripts wrapping desk instructions) is what compensates for the residual gap. Final output quality is a function of **model + workflow combined**, not model alone — and the workflow is the lever that's actually free to push on.

**Year-over-year trend lines push the ceiling up without touching the hardware:**

- Llama 4, Qwen3, DeepSeek successors are each expected to add ~10–15 points of quality at the same hardware envelope.
- 2-bit / 1.58-bit quantization research may fit 70B-class models on 8 GB VRAM within 2 years.
- The PSI-domain LoRA grows with every closed issue; periodic retraining adds quality without new hardware.
- Re-evaluate model selection annually. The build script (§8) makes model swaps a one-line config change.

### 4.4 Git host

- **Preferred:** GitHub Enterprise Server on-prem (if PSI has a license).
- **Fallback:** Gitea or Forgejo self-hosted on the same airgapped LAN. `gh`-compatible CLIs exist; the `/task` and `/needs-update` commands described below need to be ported to whichever CLI is chosen.
- **Single-engineer model:** if no on-prem host is available, USB 1 ships with the repo(s) pre-cloned and commits stay local until USB 1 is moved to the trusted online machine for push.

### 4.5 Honest capability picture: Tier 3 + open model vs. Opus 4.7

The stated goal is for the airgapped stack to approximate the capability of **Claude Opus 4.7**, the frontier model Bruce uses today. This is achievable only as an asymptote — open-weight models currently trail frontier closed models by roughly **1–2 generations**, and the gap moves on both sides. The honest task is **picking a point on the cost/quality curve** that matches PSI's budget and risk tolerance, then revisiting yearly as the open frontier advances.

**Sidebar — why this requires expensive hardware that today's setup does not:**

In Bruce's current Claude Code workflow, the MacBook contributes **almost nothing** to model quality. Claude Code is a thin client — the model runs on Anthropic's GPU cluster, and the laptop only runs the CLI, holds the file system, and streams tokens over the network. A five-year-old MacBook Air would produce the same quality output as a maxed-out M4 Max, because the model isn't local.

The subscription tiers Bruce has used ($20 Pro, $100 Max 5×, $200 Max 20×) reinforce this. The observed difference between tiers is **usage caps and reset windows**, not model quality — every tier delivers Opus 4.7 with the same per-query compute. **What you pay monthly is quota, not capability.** Anthropic absorbs the per-query cost through fleet-wide amortization.

When the network goes away, two distinct things shift at once:

- **Capability** has to materialize locally. The open-weight model you can run locally is meaningfully weaker than Opus 4.7. That quality gap is a property of the *model*, not the *hardware* — no amount of GPU spend closes it fully on its own.
- **Quota** stops being a recurring cost. The local box runs as much as you want, with no resets. The recurring subscription line item is replaced by a one-time capital outlay.

So the comparison is **not** "$100/mo vs. $5K once" on a dollars-for-equivalent-service basis. It's:

| Today (online) | Airgapped Path B |
|---|---|
| $100/mo recurring | ~$5,000 one-time (Mac Studio Ultra + 70B model) |
| Full Opus 4.7 capability | ~70–80% of Opus 4.7 capability |
| Capped quota, resets every few hours | Unlimited local quota, no resets |
| Requires network | Fully offline |

Break-even on dollars alone is roughly **4 years**. But if the airgap is a hard PSI work requirement, the dollar comparison is moot — you have to materialize compute locally regardless of subscription value, because the subscription isn't available to you in that environment at any price.

This is the right intuition for the table below: cost ≈ "the price of bringing model inference inside the airgap," and quality is capped by how good the open-weight model is, not just by how much GPU you throw at it.

**Capability tiers (current as of 2026-05):**

| Tier | Best open-weight model | Quality vs. Opus 4.7 (coding) | Required hardware | Approx. cost | Fits live-USB pattern? |
|---|---|---|---|---|---|
| Floor — laptop fallback | Qwen2.5-Coder-7B Q4 | ~30–40% | 16 GB RAM, iGPU only | ~$2,000 | Yes |
| Developer laptop (§4.3 default) | Qwen2.5-Coder-14B Q4 | ~50–55% | 32 GB RAM, 8 GB VRAM dGPU | ~$3,000–$5,000 | Yes |
| High-end laptop / mobile workstation | Qwen2.5-Coder-32B Q4 | ~60–65% | 32 GB RAM, 16 GB+ VRAM dGPU | ~$5,000–$8,000 | Marginal — load time and overlay overhead get heavy |
| **Dedicated airgapped workstation** | **Llama 3.3 70B or Qwen2.5-72B Q4** | **~70–80%** | **Mac Studio M2/M3 Ultra, 64–128 GB unified memory** | **~$4,000–$7,000** | **No — drops live-USB, switches to dedicated-host model** |
| Aspirational airgapped workstation | DeepSeek-V3 671B-MoE Q4 (~200 GB) | ~80–90% | Mac Studio Ultra 192 GB+, or multi-GPU server | ~$8,000–$15,000 (Mac); $50K+ (GPU) | No |
| Server-room / frontier parity | Llama 4 (expected 2026), or DeepSeek successors | ~85–95% | 4–8× H100 80GB | $50,000–$200,000 | No |

Quality percentages are calibrated guesses from public benchmarks (MMLU, HumanEval, SWE-bench) and informal coding-task evaluations. They are directional, not precise — and they tighten every few months as open models advance.

**Recommended target if the goal is to approximate Opus 4.7:**

The inflection point on this curve is the **dedicated airgapped workstation tier** — a Mac Studio Ultra running a 70B-class model. Reasoning:

- ~70–80% of Opus 4.7 quality is the first tier where the agent contributes positively to code review rather than being a net drag on script quality (§2.5).
- Apple Silicon's unified memory is the only sub-$10K way to run a 70B-class model with usable token throughput.
- The Mac Studio is procurable as a single line item, has a long Apple support window, runs silently, and fits on a desk.
- The deployment pattern shifts: the workstation itself is the airgap, USB 2 (data) plugs in fresh each session, no live-USB scaffolding needed.

This is a meaningfully different deployment path than §6 and should be documented as a **parallel Path B**, not a replacement for Path A:

- **Path A (laptop + live USB)** — maximizes portability and zero-host-persistence. Caps at ~60–65% of Opus 4.7 quality with the 32B model on a high-end laptop.
- **Path B (Mac Studio + data USB)** — maximizes script quality. Caps at ~80–90% of Opus 4.7 quality with DeepSeek-V3 on a 192 GB Mac Studio Ultra. Less portable; "the airgap" is the workstation, not the USB.

Which path PSI wants depends on whether **portability** or **script quality** is the harder constraint. They are not the same answer.

**What changes year-over-year (favorable trend lines):**

1. **Open-weight frontier advances.** Llama 4, DeepSeek successors, and Qwen3 will likely each shift every tier above by 10–15 percentage points of quality without new hardware.
2. **Quantization improves.** AWQ, GPTQ, and emerging 2-bit / 1.58-bit schemes let larger models fit in smaller memory with diminishing quality loss. Re-evaluate quantization choices annually.
3. **Hardware memory grows.** Apple Silicon Ultra memory ceilings rise each generation. Consumer GPU VRAM ceilings rise on a slower cadence.

**Plan implication:** the model choice in §4.3 is a **snapshot**, not a permanent commitment. The build script should make model swaps routine — drop in a new GGUF, update one config line — so the stack tracks the open frontier without a redesign every time something better ships.

### 4.4 Git host

- **Preferred:** GitHub Enterprise Server on-prem (if PSI has a license).
- **Fallback:** Gitea or Forgejo self-hosted on the same airgapped LAN. `gh`-compatible CLIs exist; the `/task` and `/needs-update` commands described below need to be ported to whichever CLI is chosen.
- **Transient model:** for a single engineer with no on-prem host, the USB clones from public GitHub during the bootstrap window, then operates on the local clone only. Pushes happen on the next bootstrap.

---

## 5. Workflow primitives to preserve

These are the parts of Bruce's current Claude Code workflow that must survive the port. Each gets a concrete realization on the airgapped stack.

### 5.1 One GitHub issue per task

- Implementation: a custom opencode command `/task <description>`.
- Behavior: before any code change, the command calls `gh issue create` (or Gitea equivalent) with a templated body. The issue number is captured and referenced in subsequent commits and the eventual PR.
- This is a **hard gate**, not a convention. The command refuses to proceed if issue creation fails.

### 5.2 Agent self-sufficiency via "request updates"

- Implementation: a custom command `/needs-update <area> <reason>`.
- Behavior: when the local model hits a gap it cannot reasonably bridge (missing context, ambiguous desk instruction, capability beyond the local model), it files an issue tagged `agent-blocked` instead of guessing or producing low-confidence output.
- The human's backlog-management role lives on these tags. Triage = decide whether to feed more context, escalate to a teammate, or accept a deterministic fallback.

### 5.3 Deterministic scripts wrap documented desk instructions

- Pattern: every recurring PSI activity that has a written desk instruction gets a Python wrapper. The wrapper is **the** source of truth for *how* the work is done; the LLM's job is to *invoke* it correctly and *narrate* the result.
- Example shape:
  - `scripts/spl_form_quality_check.py` — implements DI-PSI-313 rules directly.
  - The agent reads the desk instruction, decides which script to run, runs it, and analyzes the output.
  - If a rule is missing from the script, the agent files a `/needs-update` issue rather than re-implementing the rule in chat.

### 5.4 Local reference corpus (no WebFetch)

- Mirror needed documents into a versioned local directory (e.g. `refs/`):
  - DI-PSI-313, relevant ICDs, PSCP excerpts where releasable, internal style guides.
- The agent greps `refs/` instead of fetching from the web.
- This is **a feature, not a workaround** — the corpus becomes auditable and reproducible.

### 5.5 Memory equivalent

- opencode `AGENTS.md` files in each repo root hold project-specific context (analogous to `CLAUDE.md`).
- A `~/.config/opencode/memory/` directory holds user/feedback/project memory files in the same shape Claude Code uses today. The system prompt loads an index file at session start.
- Same hygiene rules as today: short index, semantic organization, no duplicates, prune stale entries.

---

## 6. Deployment — two tracks

The Flight Node deploys in one of two physical configurations. Both are valid; the choice depends on what hardware the engineer has and how the work is scoped.

- **Track A — Native install on the existing Tier 3 Windows laptop.** Primary near-term path. The Tier 3 laptop is **permanently connected** to the corporate network, internet, and enterprise Git host — it is **not network-airgapped**. The "airgap" in Track A is a *data* airgap: CUI stays on the Tier 3 laptop and is processed by the local Ollama model rather than being exfiltrated to a hosted frontier model. CUI segregation is enforced by Boeing's existing data-classification policy, endpoint DLP, audit logging, and user discipline — *not* by network isolation. Zero new procurement. **Concrete install steps are in [`PHASE-0-WINDOWS-BOOTSTRAP.md`](PHASE-0-WINDOWS-BOOTSTRAP.md)** — currently a draft runbook with open assumptions to validate on an actual Tier 3 environment.
- **Track B — Live Ubuntu boot on a dedicated SBC-class machine.** Broader-deployment path. A dedicated single-purpose machine (mini-PC, NUC-class, small-form-factor box — *not* a managed laptop) boots a live Ubuntu image and serves as a Flight Node distinct from any engineer's primary laptop. Cheaper per unit (~$300–$2,000 depending on inference performance targeted), easier to wipe and re-image, scales to additional users without provisioning more Tier 3 laptops, single-purpose by construction.

The remainder of §6 below describes **Track B** in detail and is the SBC-track reference (network model, USB layout, host clean-state, integrity assurance). Track A's install procedure will be authored once Phase 0 lands.

**Note on Track B hardware class.** "SBC-class" is shorthand for *dedicated single-purpose single-board-computer-style hardware*, which spans a wide cost/performance range:

- True SBCs (Raspberry Pi 5, Orange Pi, etc.) — $50–$150. CPU-only inference; useful only for 3B-class models and orchestration roles. Not a serious Flight Node candidate alone.
- NUC / mini-PC class with capable iGPU (Intel NUC with Arc, AMD Ryzen 7/9 mini-PCs with Radeon, Vulkan-accelerated `llama.cpp`) — $300–$800. Runs 7B–14B models at acceptable speed.
- Mini-PC with discrete GPU (Beelink GTi, MinisForum HX-series with mobile dGPU) — $1,000–$2,000. Comparable to Tier 3 performance, often better thermally because the box exists to be a workstation, not a laptop.

The plan does not currently lock a specific Track B class. That decision waits until there's a concrete use case demanding it.

### 6.1 Network model (Track B — the key design constraint for live-USB SBC deployment)

The live OS is **never** connected to the network during a work session. The only network event in the lifecycle is the **build phase**, which happens on a separate trusted machine.

```
   === BUILD PHASE (separate trusted online machine) ===
        - clone target Git repo(s) onto USB 1
        - pull GGUF model files onto USB 1
        - mirror Python wheels onto USB 1
        - sync local reference corpus onto USB 1
        - install opencode, Ollama, gh CLI onto USB 1 image
        - checksum and seal the image
        |
        v   (USB physically moves)
   === RUNTIME PHASE (target host, airgapped) ===
        [USB 1 inserted, host boots]
                |
                v
        [Live OS comes up — network stays disabled at boot]
                |
                v
        [Verify network is down] <-- belt-and-suspenders check
                |
                v
        [USB 2 (data) inserted, mounted with controlled write paths]
                |
                v
        [Offline work session]
              - opencode + Ollama + deterministic scripts
              - scripts on USB 1 process data on USB 2
              - derived products written to USB 2 output area
              - all code changes committed locally to USB 1's repo clone
              - agent-blocked items captured as local issues
                |
                v
        [End of session]
              - USB 2 disconnected first
              - host shut down
              - USB 1 retained for next session, or returned for sync

   === SYNC-BACK PHASE (separate trusted online machine, deliberate) ===
        - USB 1 moved back to the build/sync machine
        - local commits pushed to GHES / GitHub / Gitea
        - new issues filed for `agent-blocked` items if appropriate
        - models / refs / wheels refreshed
        - USB 1 re-sealed
```

Two consequences worth calling out:

- **No `bootstrap-online.sh` at runtime.** The agent and scripts start from a fully self-contained USB. Anything they need must already be on the USB at boot time.
- **Sync-back is a deliberate human action**, not an end-of-session automation. Physically moving USB 1 to the trusted online machine is the audit boundary.

### 6.2 Base image

- **Ubuntu 24.04 LTS** (current LTS, well-supported drivers, 5-year support window).
- Built via `mkusb` + `casper-rw` persistence, or **Ventoy** with a persistence plugin. Ventoy is easier to update.
- Encrypted persistence partition (LUKS) — required if the USB will ever leave a secured space.

### 6.3 Config-driven self-setup with default-deny firewall

The Track B SBC is **not** hand-configured. It boots, reads a single configuration file from the USB persistence partition, and applies that configuration to bring itself into its operational state. Everything about the box's behavior — services running, network interfaces enabled, firewall rules, repo paths, model selection, audit destinations — is declared in the config file and applied at boot by an init script.

**Why declarative config:**

- **Reproducibility.** Two SBCs built from the same image with the same config file produce identical behavior.
- **Auditability.** The config file is a versioned artifact; reviewers see exactly what the box is allowed to do without inspecting hidden state.
- **Drift detection.** Pre-session verification (§7.1, §7.2) re-applies the config and confirms the running state matches; any deviation triggers an alert and aborts the session.

**Default-deny firewall posture:**

- Linux `nftables` starts with **default-deny** on input, output, and forward chains.
- The bootstrap script applies *only* the rules explicitly listed in the config file's `firewall.allow` section.
- Typical airgapped operational profile: **zero external network rules.** No outbound or inbound allows. Loopback-only rules for the Ollama ↔ opencode IPC. The box does not connect to anything ever, and nothing connects to it.
- Sync-back mode (different config file, different boot profile) may relax the policy briefly for explicit Git push operations — but the operational config is fully closed.

**Minimum ports and protocols:**

- Operationally: only `lo` (loopback) is up. No DHCP request, no NTP query, no mDNS, no Bluetooth. RTC handles time; clock skew is a known cost.
- Loopback ports: Ollama bound to `127.0.0.1` only (default port 11434). opencode talks to Ollama through loopback. No other listeners.
- If any service tries to bind a non-loopback interface, the bootstrap script logs the attempt and **aborts session start**.

**Config file shape (illustrative — schema specified in Phase 0):**

```yaml
flight_node:
  id: "psi-flight-01"
  audit_log: /usb1/audit

network:
  interfaces:
    - { name: "lo", state: "up" }
    - { name: "*",  state: "down" }
  firewall:
    default: deny
    allow:
      - { chain: input,  src: 127.0.0.1, dport: 11434, proto: tcp }
      - { chain: output, dst: 127.0.0.1, dport: 11434, proto: tcp }

services:
  ollama:
    model: qwen2.5-coder:14b-instruct-q4_K_M
    bind: 127.0.0.1:11434
  opencode:
    workspace: /usb1/repos/orbit-spl-review
    model_endpoint: http://127.0.0.1:11434

usb:
  data_usb_mount: /mnt/usb2
  data_usb_writable: true
  output_dir: /mnt/usb2/output/${session_id}
  manifest_required: true
```

This is illustrative; the actual schema is a Phase 0 deliverable. The point is that **the SBC's behavior is one file, version-controlled, reviewable** — not the residue of many manual setup steps.

**Track A note:** the same declarative posture applies to Track A (Tier 3 Windows) but with platform-appropriate implementation — Windows Defender Firewall in default-deny outbound mode, PowerShell DSC or similar configuration-management tooling reading the same config schema. Track A's specific bootstrap is a Phase 0 deliverable.

### 6.4 What's pre-installed on the persistence partition

- `ollama` (binary + service config)
- `opencode` (binary + global config, with `AGENTS.md` and memory dir pre-seeded)
- `gh` CLI (or Gitea/Forgejo CLI) — used at sync-back time only
- `git`, `python3`, `pipx`, `uv`
- Curated Python wheels mirror (`refs/wheels/`) for offline `pip install`
- Pre-pulled GGUF model files (`refs/models/`)
- Local reference corpus (`refs/docs/`)
- **Target Git repo(s) pre-cloned** at `repos/<name>/` — full history, ready to commit against offline
- The build and lifecycle scripts themselves (see §7)

### 6.5 Host treatment — zero persistence on host

The target host is a **generic laptop with no persistent storage capability used by this stack**. The host is treated as a disposable compute substrate. Concretely:

- The live OS does **not** mount the host's internal drives by default. If detection is unavoidable, internal drives are mounted **read-only** at most, and never written to.
- **No swap** is configured on host disk. The system runs entirely from RAM; if it OOMs, that's a signal the model is too large for this host, not a reason to add swap.
- `/tmp` and other write paths point at `tmpfs` (RAM-backed) or at USB 1's persistence partition. Nothing falls back to the host filesystem.
- At shutdown, **nothing about the work session remains on the host**. Pull both USBs, walk away, the laptop is unchanged.
- This is verified by a startup check that lists all mounts and fails the session start if any host-disk write mount is present.

### 6.6 Minimum hardware specification (Track B SBC class)

"Generic laptop" needs a concrete floor. Below the floor, the workflow degrades to unusable; above the recommended tier, returns diminish.

| Component | Floor (works, will feel slow) | Recommended (workflow flows) | Comfortable |
|---|---|---|---|
| CPU arch | x86_64 with **AVX2** | Intel 11th gen / Ryzen 5000+ | Intel 12th+ / Ryzen 7000+ |
| CPU cores | 4c / 8t | 8c / 16t | 8c+ / 16t+ |
| RAM | **16 GB** (hard floor for 7B) | 32 GB | 32 GB+ |
| GPU | Integrated only acceptable | 6–8 GB VRAM dGPU (NVIDIA preferred for CUDA) | 12 GB+ VRAM dGPU |
| Model that fits | Qwen2.5-Coder-7B Q4 | Qwen2.5-Coder-7B Q4, comfortable | Qwen2.5-Coder-14B Q4 |
| USB ports | 2x USB 3.0+ | 2x USB 3.2 Gen 2 | 2x USB 3.2 Gen 2 |
| Expected throughput | 5–10 tok/s | 20–40 tok/s | 40+ tok/s |
| Host storage | Not used | Not used | Not used |

**Hard disqualifiers — the stack will not work on:**

- Less than 16 GB RAM (force-downgrade to 3B fallback, but agent quality drops sharply)
- CPU without AVX2 (pre-2013-ish hardware) — inference is unusably slow
- USB 2.0 only — model load times make session startup intolerable
- Locked-down UEFI without ability to disable Secure Boot or trust a signed image — can't boot the USB

**Other host requirements:**

- UEFI boot from USB, with Secure Boot either disabled or the image signed appropriately. This is often a Boeing IT consideration in practice.
- BIOS settings allow USB boot priority. Some corporate laptops have this locked.
- GPU detection on Optimus laptops can be flaky on Linux live boots; validate per laptop model before committing.

**USB hardware:**

- USB 1 (tools): treat as the workstation drive. A small SSD in a USB-C enclosure (USB 3.2 Gen 2, 256+ GB) is strongly preferred over any thumb drive — startup speed and longevity both matter.
- USB 2 (data): a USB 3.0 thumb drive is usually fine since CSV files are small.

---

## 7. Secure factory reset & data integrity assurance

The lifecycle of an airgapped session has three integrity boundaries that must be verifiable: (1) the **host** is in a known clean state before and after the session, (2) **USB 1 (tools)** has not been tampered with between sessions, and (3) **USB 2 (data)** is authentic on intake, and its derived products are correctly attributed on output.

This section applies to both deployment paths (live-USB on a generic laptop, or a dedicated Mac mini workstation); path-specific differences are called out where they matter.

### 7.1 Host clean-state guarantee

The live-USB pattern is mostly self-cleaning by construction: nothing the live OS does writes to host disk, and RAM is gone on power-off. But "by construction" isn't evidence — the stack produces explicit checks at session start and session end.

**Pre-session (`verify-host-clean.sh`, runs immediately after the live OS boots):**

- Enumerate all block devices and their mount state. Fail the session start if any host-internal drive is mounted writable.
- Confirm `/etc/fstab` and current `mount` show only USB-backed and `tmpfs` write targets.
- Confirm no swap is configured (`swapon --show` returns empty), or swap is on `tmpfs` / USB 1 only.
- Confirm `/tmp`, `/var/tmp`, and `/var/log` are `tmpfs` or on USB 1.
- Confirm no unexpected network interfaces are up (`ip -br link` shows only `lo` as `UP`).
- Log all results to `audit/<session-id>/pre-host-check.log` on USB 1.

**Post-session (`secure-shutdown.sh`, runs at session end):**

- `shred` any tmpfs files that were marked CUI during the session.
- Flush filesystem caches (`sync && echo 3 > /proc/sys/vm/drop_caches`).
- Unmount USB 2 first, then USB 1.
- `poweroff` (not `reboot` — full power-off clears DRAM in seconds on modern hardware).
- Optional cold-boot-attack mitigation: wait 30 seconds after power-off before removing USBs.

**For a dedicated host (Mac mini path):**

- Factory reset (DFU restore on Apple Silicon, or full reinstall on a PC) happens at **intake** and **decommission**, not per session.
- A SHA-256 baseline of the OS volume is captured after initial install and stored offline (printed and sealed, or on a separate trusted machine).
- Pre-session: re-hash the OS volume, compare to baseline. Drift = forensic event, not an auto-recoverable error.

### 7.2 USB 1 (tools) integrity

USB 1 is the trusted boot medium. Tampering between sessions is the primary threat.

- **Build-phase:** the build script produces a SHA-256 of the sealed rootfs partition and prints it for transcription onto a physical seal applied to the drive case. The hash is also stored inside the partition itself (so the device can self-verify) and on a separate sealed manifest.
- **Read-mostly runtime:** the rootfs partition is mounted **read-only** during the session. Writes go to a `casper-rw` overlay file on a separate writable partition (audit logs, session work product, repo commits).
- **Pre-session verification (`verify-usb1.sh`):** re-hash the rootfs partition and the model files, compare to the build manifest. Mismatch = abort, do not start the session.
- **Sync-back-phase:** after pushing commits to GHES/Gitea, the writable overlay can be wiped (`mkfs` the overlay partition) to return USB 1 to a known clean baseline for the next session.

### 7.3 USB 2 (data) integrity and output marking

USB 2 is the variable input. Each session's data must be authenticated on intake, and the derived products must carry correct attribution and markings.

**Intake (`accept-usb2.sh`, runs when USB 2 is inserted):**

- Compute SHA-256 of every file on USB 2.
- Compare against a `MANIFEST.sha256` file that must be present on USB 2 — provided by the data owner through a known-good channel.
- Refuse to mount USB 2 writable until manifest verification passes. Mount read-only on failure so the session can still produce a clean report of what was received and why it was rejected.
- Log all hashes and verification status to the session audit log.

**Output discipline (enforced by the script library, not optional):**

- Derived products land in `USB2:/output/<session-id>/` — never mixed with input files.
- Every output file carries a CUI / classification banner appropriate to the input, mirroring the highest marking of any input it derived from.
- Every output file's SHA-256 is recorded in `USB2:/output/<session-id>/MANIFEST.sha256`.
- Every output references the script path, Git commit hash, and input file hashes that produced it — in a CSV header comment or sidecar JSON for binary outputs.

**End-of-session attestation:**

- A one-page Markdown attestation is auto-generated listing: session ID, host machine identifier, USB 1 hash, USB 2 input hashes, all output hashes, all Git commits made, all `agent-blocked` issues filed.
- This attestation is the unit of evidence handed to a reviewer.

### 7.4 Audit trail

Every session produces a directory at `USB1:/audit/<session-id>/` containing:

- `pre-host-check.log` — output of `verify-host-clean.sh`
- `usb1-verify.log` — output of `verify-usb1.sh`
- `usb2-intake.log` — manifest verification of USB 2
- `commands.log` — full transcript of every shell command (via `script(1)` or equivalent)
- `opencode-transcript.jsonl` — the agent's tool-use and message log
- `commits.log` — `git log` of every commit made during the session
- `outputs.manifest` — copy of the USB 2 output manifest
- `attestation.md` — the human-readable summary

At sync-back time, audit directories can be pushed to GHES/Gitea as release assets or a separate `audit/` branch for long-term retention.

### 7.5 Threats this design does **not** mitigate

Honest list, so a reviewer knows the boundary:

- **Firmware-level malware on the host** (UEFI/SMM implants, hypervisor-below-the-OS attacks). Live USB doesn't help. Mitigated only by host hardware choice and BIOS update discipline.
- **Compromised USB 1 build machine.** Hashes on USB 1 only mean "what we built is what we boot." Build-machine hygiene is the upstream control.
- **Insider with physical access** swapping drives, photographing the screen, or capturing display output. Physical security and procedure, not a stack-layer concern.
- **Side channels** (acoustic, EM, power, thermal). Beyond scope here.

These exclusions are intentional — calling them out explicitly is part of the integrity story.

---

## 8. Scripts to build (deferred — no code written yet)

This is the work that follows plan approval. Listed here so the scope is visible.

1. **`build-usb.sh`** — produces a fresh USB image with all pre-baked content. Runs on a connected machine. Pulls models, mirrors wheels, copies reference corpus.
2. **`bootstrap-online.sh`** — runs on first boot during the online window. Clones the configured target repo, verifies model hashes, syncs reference updates.
3. **`go-offline.sh`** — disables all network interfaces (`nmcli radio all off`, `rfkill block all`), confirms with `ip a` that nothing is up, logs the timestamp. Idempotent.
4. **`start-session.sh`** — launches Ollama, then opencode, with the correct model and project context loaded.
5. **`opencode-commands/task.md`** — slash command implementing §5.1.
6. **`opencode-commands/needs-update.md`** — slash command implementing §5.2.
7. **`reconnect-and-push.sh`** — explicit, user-invoked. Re-enables networking, runs `git push`, then either re-disconnects or shuts down per flag.

---

## 8. Phased roadmap

Incremental. Each phase ends with something testable before moving on.

- **Phase 0 — Validate stack on a normal laptop** (not airgapped, not USB).
  Install Ollama + opencode locally, pull Qwen2.5-Coder-32B, run it against an existing PSI repo, confirm `/task` and `/needs-update` workflows feel right. Tune prompts.
- **Phase 1 — Deterministic script library** for one desk instruction (suggest: DI-PSI-313 form-quality check, since `ORBIT_SPL_Review` already has the rule logic). Prove the "agent invokes script, narrates result" pattern.
- **Phase 2 — Reference corpus and `AGENTS.md`** assembled for the target project. Memory directory populated. Run a full task end-to-end under the new harness.
- **Phase 3 — Build the USB image** (`build-usb.sh`). Boot it on a clean machine over the air gap. Run the same task. Confirm parity with Phase 2.
- **Phase 4 — Boeing IT review** of the USB image. Address findings. Document the approved configuration.
- **Phase 5 — Hand-off doc** so another PSI engineer can rebuild from scratch in a day.

---

## 9. Honest tradeoffs and risks

- **Capability gap.** Qwen2.5-Coder-32B is roughly 30–40% as strong as Claude Opus on novel reasoning. The "deterministic scripts wrap desk instructions" pattern is the primary mitigation. Tasks that fall outside the scripted scope will need a `/needs-update` issue, not a heroic local-model attempt.
- **Live USB friction.** Boot time, USB write speed, and GPU detection vary by host machine. Expect 1–2 days per new host to validate.
- **Boeing IT.** A bootable USB with persistence is a security-sensitive artifact. The image will need review and probably a signed-and-checksummed distribution mechanism before broad use.
- **Maintenance.** Models, opencode, and Ollama all update on their own cadences. The USB image needs a rebuild cadence (suggest: quarterly, or whenever a target model has a meaningful upgrade).
- **Backlog discipline.** The human-on-the-loop pattern only works if the engineer actually triages the `agent-blocked` issues. If those pile up, the system degrades into "agent guesses badly." This is a process risk, not a technical one.
- **Human-engagement burden.** This workflow forces the human to engage with the software at every meaningful step — file issues against observed failures, run anonymizers, escalate to the Equivalent Node, apply fixes back, capture regression tests, audit outputs. The friction is real and it is **not a bug**. It is the price of operating on CUI without trusting any single automated component to make a CUI-handling decision unsupervised. Reducing it further would require either trusting the anonymizer absolutely (we don't — it gets continuously audited) or trusting the local LLM with CUI-bearing decisions (the airgap forbids this by construction). Accept the engagement burden as the design's deliberate cost; do not optimize for "fewer human touches" if that comes at the expense of supervised CUI handling.

---

## 10. Open questions for Bruce

1. Target host machine(s) — Boeing-issued laptop, dedicated workstation, or "any host that boots a USB"? Affects driver and GPU strategy.
2. Git host — is GHES available on PSI's network, or is this Gitea/Forgejo / offline-only?
3. First desk instruction to wrap — DI-PSI-313 (lifts directly from `ORBIT_SPL_Review`) or something else higher-priority?
4. Persistence partition encryption — LUKS passphrase per engineer, or a managed key escrow?
5. Is the goal a personal Bruce-only USB first, then a team-shareable image? Or team-shareable from day one?

---

## 11. Where to look next

- Once Phase 0 starts, individual component setup notes live in sibling files in this directory (`PHASE-0-LOCAL-VALIDATION.md`, etc.).
- Cross-reference: `~/ORBIT_SPL_Review/` already has the DI-PSI-313 rule logic that Phase 1 will lift from.
- Memory anchor: `~/.claude/projects/-Users-brucedombrowski/memory/` is the source of the workflow conventions this plan ports.
