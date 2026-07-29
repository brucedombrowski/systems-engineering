# Pattern: Analyst-Dashboard Source Ingestion

Distilled 2026-07-29 from converging implementations in ORBIT-RIFD-Status
(`ingested/` + history backfill) and KuIP_Analyzer (`_local/docs/ingested/` +
provenance cards). Apply to any analyst tool that consumes exported files
from official systems (ORBIT, PDL, EDMS, share drives).

## The pattern

1. **Downloads is an inbox, never a store.** The analyst's only gesture is
   "save the export to Downloads" (or click the tool's regenerate link).
   On the next run, the tool MOVES recognized files out — CUI must not
   accumulate in a general-purpose folder.

2. **Recognition by filename pattern, never wildcard-everything.** Each
   source registers a glob (`KuIP*.csv`, `Tasks_List*.csv`, `*275*.pdf`).
   Unrecognized files are untouched — the tool only claims what it owns.

3. **`ingested/` is the accumulating CM snapshot layer.**
   - Move preserves mtimes (newest-wins discovery stays correct).
   - Exact-content duplicates dedupe (hash compare).
   - Name collisions with *different* content version the old file aside
     (`name__<mtime>.ext`) — never overwrite history.
   - Content-hash-versioned copies (`stem__<sha12>.ext`) for documents.
   - History enables later derivation: RIFD backfills due-date change
     trails from archived snapshots; KuIP_Analyzer's change-verification
     loop diffs successive pulls.

4. **Discovery searches inbox + ingested transparently.** Analyzers never
   know or care whether a file was just downloaded or long archived.
   Fresh (mtime) always wins.

5. **Every source is hash-stamped and dual-dated.** Record `sha256`,
   `ingested_at` (when the tool first saw this content — a real recorded
   moment), and `file dated` (the file's own mtime; for share-drive docs
   that's the author's save date, not ingestion). Display both — they
   answer different questions.

6. **Provenance is user-facing.** A sources card renders every registered
   source whether present or absent: present → name, hash, both dates,
   regenerate recipe; absent → "file not found" + the desk instruction to
   go get it. The card doubles as the fresh-clone bootstrap checklist.

7. **The ingested store lives under the tool's gitignored local area**
   (`_local/`, `input/`), inside the CUI boundary. Whole-disk encryption
   covers at-rest; OS-level folder encryption (Windows EFS) is an optional
   hardening with zero tool changes.

## Anti-patterns this replaces

- Reading sources in place from Downloads (deletion loses the source;
  CUI lingers indefinitely; no history).
- Copy-without-move (inbox grows forever — the halfway state KuIP_Analyzer
  briefly shipped before 2026-07-29).
- Using file mtime as "ingested" for share-drive files (it's the author's
  save date; record the tool's own intake moment separately).
- Wildcard ingestion of an entire folder (claims files the tool doesn't
  own).

## Reference implementations

- KuIP_Analyzer: `src/main.py` `_ingest_inbox_files()`,
  `src/doc_onboarding.py` (hash-gated onboarding, dual dates, versioned
  archive), provenance card in `src/reporters/components.py`.
- ORBIT-RIFD-Status: `ingested/` CSV archive + due-date history backfill.
