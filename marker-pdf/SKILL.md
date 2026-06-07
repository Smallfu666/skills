---
name: marker-pdf
description: 'Use when the user wants to convert PDF to Markdown, extract text from PDF, or run OCR on PDFs. Covers phrases like "把 PDF 轉成 Markdown", "PDF 轉 md", "跑 marker", "marker_single", "提取 PDF 內容", "PDF OCR", or "convert PDF". Marker-pdf is installed on the DGX Spark, not locally. Access is via double SSH: ssh wsl → ssh dgx.'
---

# Marker PDF Conversion Skill

## Overview

Marker-pdf is installed on the **DGX Spark** (accessible via `ssh wsl ssh dgx`), not locally.

## CLI commands

- `marker_single` — Convert a single PDF
- `marker` — Batch convert
- `marker_server` — Start server mode
- `marker_gui` — GUI mode
- `marker_chunk_convert` — Chunk-based conversion
- `marker_extract` — Extract text

## Binary path

```
/home/hsnl/.local/bin/marker_single
```

Add `~/.local/bin` to PATH in non-interactive SSH sessions.

## Basic usage via SSH

```bash
ssh wsl "ssh dgx \"export PATH=\$HOME/.local/bin:\$PATH && marker_single /path/to/file.pdf --output_format markdown --output_dir ./output\""
```

## Options

| Option | Description |
|---|---|
| `--use_llm` | Use LLM for enhanced extraction |
| `--force_ocr` | Force OCR on all pages |
| `--page_range` | Specify page range (e.g., `1-10`) |
| `--output_format` | `markdown`, `json`, `html`, or `chunks` |

## Repo

https://github.com/datalab-to/marker

## Safety rules

- Do not modify files on the remote machine.
- Confirm the output directory exists or let the remote side create it.
- Always specify `--output_format markdown` unless the user asks for another format.
- If the SSH route fails, report the exact error. Do not retry blindly.
