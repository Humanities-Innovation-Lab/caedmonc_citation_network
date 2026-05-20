---
title: "Bibnow Pipeline — Team Note"
date: 2026-05-20
tags: [bibnow, caedmon, pipeline, zotero, obsidian]
---

# Bibnow Pipeline — Team Note

## What it does

Bibnow takes citations in **CSL-JSON** and, with one command:
1. Uploads each entry to the **Caedmon Citation Network** collection on Zotero.
2. Writes a templated Markdown note into the Obsidian vault at `/home/lab/Desktop/caedmoncitation/`, sorted into century subfolders (`1800s/`, `1900s/`, `2000s/`, etc.).

## What's in place (as of 2026-05-20)

- Notes saved as `Author Year.md`, auto-sorted into century folders.
- `keywords:` field removed from the frontmatter template.
- Uploads go directly to the Caedmon Citation Network Zotero collection.
- **SEENET parser** (`parse_seenet.py`) — converts SEENET-style HTML bibliography directly to CSL-JSON, preserving the existing citekeys.
- **DOI enrichment** (`enrich_dois.py`) — looks up DOIs on Crossref before upload.
- **287-entry SEENET bibliography** ingested end-to-end on 2026-05-20.

## Workflow

All commands run from `/home/lab/Desktop/Caedmon/bibnow/v2/`. Always use the venv python: `/home/lab/Desktop/Caedmon/bibnow/.venv/bin/python3`.

### 1. Get citations into CSL-JSON

**SEENET HTML?** Use the parser. Writes `input.txt` directly:
```bash
/home/lab/Desktop/Caedmon/bibnow/.venv/bin/python3 parse_seenet.py /path/to/your.html > input.txt
```

**Anything else** (plain text, BibTeX, Word doc)? Paste into Claude or ChatGPT:
> "Convert these citations to CSL-JSON, wrapped in a JSON array. Output only the JSON, no commentary. Do not invent DOIs."

Then save the output to `/home/lab/Desktop/Caedmon/bibnow/v2/input.txt`.

### 2. Dry-run (sanity check)

```bash
cd /home/lab/Desktop/Caedmon/bibnow/v2 && /home/lab/Desktop/Caedmon/bibnow/.venv/bin/python3 pipeline.py
```

Scroll the output. Spot-check author, year, and target folder. Fix `input.txt` if anything looks broken.

### 3. Add DOIs (optional)

```bash
/home/lab/Desktop/Caedmon/bibnow/.venv/bin/python3 enrich_dois.py input.txt
```

Queries Crossref and fills in any DOIs it can find. `input.txt.bak` is the safety copy. Hit rate is ~20% for older humanities corpora, higher (50%+) for modern journal articles.

### 4. Commit for real

```bash
/home/lab/Desktop/Caedmon/bibnow/.venv/bin/python3 pipeline.py --commit
```

Each entry uploads to Zotero and a Markdown note is written into the vault. **For batches over ~50 entries, split `input.txt` into chunks first** — there's no resume-on-failure.

### 5. Bulk-attach OA PDFs in Zotero

In Zotero, open the **Caedmon Citation Network** collection → Ctrl+A → right-click → **Find Available PDF**. Unpaywall finds the open-access copies it can.

## File reference

| File | Purpose |
|------|---------|
| `parse_seenet.py` | SEENET HTML → CSL-JSON |
| `enrich_dois.py` | Adds DOIs via Crossref |
| `pipeline.py` | Uploads to Zotero + writes Obsidian notes |
| `input.txt` | Working CSL-JSON file (overwritten each batch) |
| `.env` | Zotero API key + collection key |
