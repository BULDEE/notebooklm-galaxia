# notebooklm-galaxia

Claude Code plugin: full programmatic access to Google NotebookLM, positioned inside the Galaxia stack.

## What it does

NotebookLM exposes no official query API. This plugin builds on the community CLI `notebooklm-py` (browser automation, cookies) to drive NotebookLM from Claude Code:

- Create notebooks, add sources (URL, YouTube, PDF, audio, video, image)
- Query a corpus with grounded citations
- Generate every artifact: audio podcast, video, slides, quiz, flashcards, infographic, mind map, report
- Download results (mp3, mp4, pdf, pptx, md, csv, json)

## Positioning (important)

In the Galaxia stack, the **retrieval memory is gbrain** (Postgres+pgvector, with the Obsidian vault as the system of record). NotebookLM is NOT the agent's memory. This plugin keeps it to its real strengths:

- **Media generator**: podcasts, videos, slides, infographics.
- One-off analysis of a **large external corpus** (dozens of documents) with citations.

What we do NOT do: wire NotebookLM as an MCP server for the autonomous agent (Hermes), or use it as memory storage. Browser automation is too fragile (rate limits, ToS, breaks on any UI change) for a 24/7 agent. To trigger a generation from the autonomous stack, go through N8N (deterministic workflow, out of the agent hot path).

## Skills

| Skill | Role |
|-------|------|
| `notebooklm` | Full NotebookLM CLI access (setup, auth, sources, query, generation, download). |
| `wrapup` | gbrain-first session wrap-up: writes durable knowledge into gbrain (canonical memory), updates Claude memories, and optionally generates a NotebookLM media artifact. |

## Installation

The plugin handles setup on first use (`/notebooklm`). Manually:

```bash
python3 -m venv ~/.notebooklm-venv
source ~/.notebooklm-venv/bin/activate
pip install "notebooklm-py[browser]"
playwright install chromium
ln -sf ~/.notebooklm-venv/bin/notebooklm ~/bin/notebooklm
```

Authentication: see the `notebooklm` skill (browser login, session capture).

## Requirements

- Python 3.10+
- Chromium (via `playwright install chromium`)
- A Google account with NotebookLM
- For `wrapup`: the `gbrain` CLI pointed at the Galaxia brain

## Credits

Based on the community CLI `notebooklm-py` and the community NotebookLM / WrapUp skills, adapted and repositioned for the Galaxia stack (gbrain as canonical memory). NotebookLM is an unofficial API and can break without notice.

## License

Apache-2.0. (c) Alexandre Mallet / BULDEE.
