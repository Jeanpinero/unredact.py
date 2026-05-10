# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A single-script PyMuPDF tool that audits PDFs for weak redactions. It does not edit the input — it builds a fresh PDF by re-extracting text + drawings + images from the source, dropping anything that looks like a redaction overlay (large black/white vector rects, near-black/near-white images, annotations). Recovered text is overlaid in red by default so leaks are visible.

## Commands

Setup (Windows / PowerShell):
```powershell
python -m venv venv
.\venv\Scripts\Activate
pip install -r .\docs\requirements.txt
```

Run:
```powershell
python .\src\unredact.py -i <input.pdf-or-folder> -o <output-folder> [options]
```

Flags (see `src/unredact.py:208-217` for the source of truth):
- `-i / --input`, `-o / --output`, `-n / --name` (single-file output rename)
- `-b / --bbox` `1|0` — remove black/white vector boxes (default 1)
- `--highlight / --hl` `1|0` — recovered text in red vs black (default 1)
- `--hits` — discard outputs that had zero redactions removed (undocumented in ReadMe.md)

There is no test suite, no linter config, and no build step. `test/` holds sample input PDFs alongside `_UNREDACTED` reference outputs for manual visual diffing — running the script against `test/<name>.pdf` and comparing to `test/<name>_UNREDACTED.pdf` is the smoke test.

## Architecture notes

**Per-page pipeline** in `process_file` (`src/unredact.py:46`): for each source page, create a blank page in a new doc, then re-insert content in three layered passes:

1. **Images** — for each xref, optionally compute average pixel brightness (`<15` ⇒ black box, `>240` ⇒ white box) and skip; otherwise re-insert at its original rect.
2. **Vector drawings** — `clean_vector_redactions` (`src/unredact.py:11`) drops paths whose fill or stroke is near-black or near-white **and** whose bbox is larger than 5×5 (or stroke width >10). Survivors are redrawn via `Shape` primitives. When `--bbox 0`, this filtering step is skipped and *all* drawings pass through unchanged (note: with `--bbox 0`, only `l` and `re` items are redrawn — the `qu`/`c` branches are bbox-mode only, see `src/unredact.py:126-132`).
3. **Text** — every span from `page.get_text("dict")` is re-inserted at its original origin/size, colored red or black per `--highlight`.

This re-build approach (rather than `add_redact_annot` / `apply_redactions`) is deliberate: it strips anything not explicitly re-emitted, so embedded redaction layers that PyMuPDF wouldn't otherwise touch get dropped.

**Working-directory side effect**: `src/unredact.py:8-9` does `os.chdir(BASE_DIR)` at import time, where `BASE_DIR` is the script's own directory. Relative `-i` / `-o` paths are therefore resolved against `src/`, not the shell's CWD. When debugging path issues, prefer absolute paths or invoke from `src/`.
