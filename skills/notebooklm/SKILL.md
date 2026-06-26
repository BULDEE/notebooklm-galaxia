---
name: notebooklm
description: Full programmatic access to Google NotebookLM via the notebooklm-py CLI. Create notebooks, add sources (URL, YouTube, PDF, audio, video, image), query a corpus, generate every artifact (podcast, video, slides, quiz, flashcards, infographic, mind map, report) and download them. Triggers on /notebooklm or intent like "make a podcast about X", "analyze these documents", "install notebooklm".
---
<!-- notebooklm-galaxia v0.1.0 - base: notebooklm-py (community CLI, browser automation) -->

# NotebookLM (Galaxia)

Full programmatic access to Google NotebookLM, including capabilities not exposed in the web UI.

## Positioning in the Galaxia stack (read first)

NotebookLM is NOT the agent's memory. The retrieval brain is **gbrain + Postgres+pgvector** (MCP-native, agent write-back, private). NotebookLM has no official query API: this CLI drives browser automation (cookies), which is fragile and against Google's automated-access terms. So use it for what it is genuinely good at, and never in an autonomous agent's hot path.

**Use NotebookLM for:**
- Generating **media**: audio podcast, video explainer, slides, infographic, mind map, quiz, flashcards.
- One-off **analysis of a large external corpus** (dozens of PDFs/YouTube/URLs) with grounded citations, without building an ingestion pipeline.

**Do NOT use NotebookLM for:**
- The agent's long-term memory (that is gbrain: `gbrain put` / the put_page MCP tool).
- Retrieval inside an autonomous agent loop (fragile, ~50 requests/day, breaks on any Google UI change).

## Step 0: Setup (on first use)

If `notebooklm` is not installed or authenticated, do the setup first.

### Prerequisite: Python 3.10+

```bash
python3 --version
```

If Python < 3.10 (the macOS default 3.9.x), install a compatible version:

macOS (Homebrew):
```bash
brew install python@3.12
```
Then use `/opt/homebrew/bin/python3.12` (Apple Silicon) or `/usr/local/bin/python3.12` (Intel).

Linux (apt):
```bash
sudo apt update && sudo apt install -y python3.12 python3.12-venv
```

### Install the CLI

Always use a venv (avoids "externally-managed-environment" and PATH issues):

```bash
PYTHON=$(command -v python3.12 2>/dev/null || command -v python3.11 2>/dev/null || command -v python3.10 2>/dev/null || command -v python3)
$PYTHON -c "import sys; assert sys.version_info >= (3,10), f'Python {sys.version} too old, need 3.10+'; print(f'Python {sys.version}')"
$PYTHON -m venv ~/.notebooklm-venv
source ~/.notebooklm-venv/bin/activate
pip install "notebooklm-py[browser]"
playwright install chromium
```

Symlink on PATH:
```bash
mkdir -p ~/bin
ln -sf ~/.notebooklm-venv/bin/notebooklm ~/bin/notebooklm
export PATH="$HOME/bin:$PATH"
notebooklm --help
```

### Authenticate

IMPORTANT: `notebooklm login` needs interactive terminal input (pressing Enter after sign-in), which Claude Code's bash tool does not support. Use this login script instead.

Tell the user:

> I am going to open a browser window: sign into your Google account and go to notebooklm.google.com. Take your time, I will wait for your confirmation before closing it.

Then:

```bash
cat > /tmp/nlm_login.py << 'PYEOF'
import json, time
from pathlib import Path
from playwright.sync_api import sync_playwright

STORAGE_PATH = Path.home() / ".notebooklm" / "storage_state.json"
PROFILE_PATH = Path.home() / ".notebooklm" / "browser_profile"
SIGNAL_FILE = Path("/tmp/nlm_save_signal")

SIGNAL_FILE.unlink(missing_ok=True)
STORAGE_PATH.parent.mkdir(parents=True, exist_ok=True)

with sync_playwright() as p:
    browser = p.chromium.launch_persistent_context(
        user_data_dir=str(PROFILE_PATH),
        headless=False,
        args=["--disable-blink-features=AutomationControlled"],
    )
    page = browser.pages[0] if browser.pages else browser.new_page()
    page.goto("https://notebooklm.google.com/")
    print("Browser open. Waiting for the save signal...")
    while not SIGNAL_FILE.exists():
        time.sleep(1)
    storage = browser.storage_state()
    with open(STORAGE_PATH, "w") as f:
        json.dump(storage, f)
    print(f"Saved {len(storage.get('cookies', []))} cookies")
    browser.close()

SIGNAL_FILE.unlink(missing_ok=True)
print(f"Auth saved: {STORAGE_PATH}")
PYEOF

source ~/.notebooklm-venv/bin/activate
python3 /tmp/nlm_login.py > /tmp/nlm_login_output.txt 2>&1 &
echo "Login started (PID=$!). Browser opens in a few seconds..."
```

Wait ~10s, ask the user if they see the browser and are signed in. Once on the NotebookLM home:

```bash
touch /tmp/nlm_save_signal
sleep 8
cat /tmp/nlm_login_output.txt
```

Verify:
```bash
export PATH="$HOME/bin:$PATH"
notebooklm auth check
notebooklm list
```

If OK (SID cookie present), confirm and clean up:
```bash
rm -f /tmp/nlm_login.py /tmp/nlm_login_output.txt /tmp/nlm_save_signal
```

If it fails (SID missing), reset the profile and retry:
```bash
rm -rf ~/.notebooklm/browser_profile ~/.notebooklm/storage_state.json
```

## Use from Hermes (Galaxia, optional)

NotebookLM is NOT wired as an MCP server in Hermes (no official query API, browser automation too fragile for a 24/7 agent). To trigger a NotebookLM generation from the autonomous stack, go through **N8N** (deterministic workflow: receive a request, run the CLI on a machine with a session, deliver the media). Keep NotebookLM out of the agent's hot path.

## Autonomy rules

Run without confirmation:
- `notebooklm status`, `auth check`, `list`, `source list`, `artifact list`, `language list/get/set`, `artifact wait`, `source wait`, `research status/wait`, `use <id>`, `create`, `ask "..."` (without `--save-as-note`), `history`, `source add`

Ask before:
- `delete` (destructive), `generate *` (long, may fail), `download *` (writes to disk), `ask "..." --save-as-note` (writes a note), `history --save`

## Quick reference

| Task | Command |
|------|---------|
| List notebooks | `notebooklm list` |
| Create notebook | `notebooklm create "Title"` |
| Set context | `notebooklm use <notebook_id>` |
| Context status | `notebooklm status` |
| Add URL | `notebooklm source add "https://..."` |
| Add file | `notebooklm source add ./file.pdf` |
| Add YouTube | `notebooklm source add "https://youtube.com/..."` |
| List sources | `notebooklm source list` |
| Wait for source processing | `notebooklm source wait <source_id>` |
| Web research (fast) | `notebooklm source add-research "query"` |
| Web research (deep) | `notebooklm source add-research "query" --mode deep --no-wait` |
| Query | `notebooklm ask "question"` |
| Query (specific sources) | `notebooklm ask "question" -s src_id1 -s src_id2` |
| Query (with citations) | `notebooklm ask "question" --json` |
| Save answer as note | `notebooklm ask "question" --save-as-note` |
| Source fulltext | `notebooklm source fulltext <source_id>` |
| Generate podcast | `notebooklm generate audio "instructions"` |
| Generate video | `notebooklm generate video "instructions"` |
| Generate report | `notebooklm generate report --format briefing-doc` |
| Generate quiz | `notebooklm generate quiz` |
| Generate flashcards | `notebooklm generate flashcards` |
| Generate infographic | `notebooklm generate infographic` |
| Generate mind map | `notebooklm generate mind-map` |
| Generate slides | `notebooklm generate slide-deck` |
| Artifact status | `notebooklm artifact list` |
| Wait for completion | `notebooklm artifact wait <artifact_id>` |
| Download audio | `notebooklm download audio ./output.mp3` |
| Download video | `notebooklm download video ./output.mp4` |
| Download slides PDF | `notebooklm download slide-deck ./slides.pdf` |
| Download report | `notebooklm download report ./report.md` |

## Generation types

All `generate` commands support `-s/--source`, `--language`, `--json`, `--retry N`.

| Type | Command | Options | Output |
|------|---------|---------|--------|
| Podcast | `generate audio` | `--format [deep-dive\|brief\|critique\|debate]`, `--length [short\|default\|long]` | .mp3 |
| Video | `generate video` | `--format [explainer\|brief]`, `--style [auto\|classic\|whiteboard\|kawaii\|anime\|watercolor\|retro-print\|heritage\|paper-craft]` | .mp4 |
| Slides | `generate slide-deck` | `--format [detailed\|presenter]`, `--length [default\|short]` | .pdf / .pptx |
| Infographic | `generate infographic` | `--orientation [landscape\|portrait\|square]`, `--detail [concise\|standard\|detailed]` | .png |
| Report | `generate report` | `--format [briefing-doc\|study-guide\|blog-post\|custom]`, `--append "..."` | .md |
| Mind Map | `generate mind-map` | (instant) | .json |
| Quiz | `generate quiz` | `--difficulty [easy\|medium\|hard]`, `--quantity [fewer\|standard\|more]` | .json/.md/.html |
| Flashcards | `generate flashcards` | same as quiz | .json/.md/.html |

## Common workflows

### Research to podcast
1. `notebooklm create "Research: [topic]"`
2. `notebooklm source add` for each URL/document
3. Wait: `notebooklm source list --json` until status=ready
4. `notebooklm generate audio "Focus on [angle]"`
5. `notebooklm artifact list` for status
6. `notebooklm download audio ./podcast.mp3`

### Large-corpus analysis (where NotebookLM beats gbrain)
1. `notebooklm create "Analysis: [project]"`
2. `notebooklm source add` (dozens of PDFs/URLs)
3. `notebooklm ask "Synthesize the key points" --json` (grounded answers)
4. If durably useful: write the synthesis back into gbrain (`gbrain put`), do not leave it in NotebookLM.

## Error handling

| Error | Cause | Action |
|-------|-------|--------|
| Auth/cookie | Session expired | Re-login (Step 0 script) |
| "No notebook context" | Context not set | `notebooklm use <id>` |
| Rate limit | Google throttle | Wait 5-10 min |
| Download fails | Generation incomplete | `artifact list` |

## Known limitations

- Audio/video/quiz/flashcards/infographic/slides generation can fail (Google rate limits).
- Timing: audio 10-20 min, video 15-45 min, quiz/flashcards 5-15 min.
- Unofficial API (browser automation): Google can break it without notice. Do not depend on it for a production agent.
