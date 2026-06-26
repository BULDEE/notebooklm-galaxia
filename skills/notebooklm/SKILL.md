---
name: notebooklm
description: Acces programmatique complet a Google NotebookLM via la CLI notebooklm-py. Creer des notebooks, ajouter des sources (URL, YouTube, PDF, audio, video, image), interroger un corpus, generer tous les artefacts (podcast, video, slides, quiz, flashcards, infographie, mind map, rapport) et les telecharger. Se declenche sur /notebooklm ou une intention type "fais un podcast sur X", "analyse ces documents", "installe notebooklm".
---
<!-- notebooklm-galaxia v0.1.0 - base: notebooklm-py (CLI communautaire, automation navigateur) -->

# NotebookLM (Galaxia)

Acces programmatique complet a Google NotebookLM, y compris des capacites non exposees dans l'UI web.

## Positionnement dans la stack Galaxia (a lire avant usage)

NotebookLM n'est PAS la memoire de l'agent. Le brain de retrieval, c'est **gbrain + Postgres+pgvector** (MCP-native, write-back agent, prive). NotebookLM n'a pas d'API de query officielle : cette CLI passe par de l'automation navigateur (cookies), fragile et soumise aux conditions de Google. On l'utilise donc pour ce qu'elle fait de mieux, et pas dans le hot path d'un agent autonome.

**Utiliser NotebookLM pour :**
- Generer du **media** : podcast audio, video explainer, slides, infographie, mind map, quiz, flashcards.
- **Analyser un gros corpus externe** ponctuel (dizaines de PDF/YouTube/URL) avec reponses citees, sans construire d'ingestion.

**Ne PAS utiliser NotebookLM pour :**
- La memoire long-terme de l'agent (c'est gbrain : `gbrain put` / outils MCP put_page).
- Le retrieval dans une boucle agent autonome (fragile, ~50 requetes/jour, casse au moindre changement d'UI Google).

## Step 0 : Setup (au premier usage)

Si `notebooklm` n'est pas installe ou authentifie, faire le setup d'abord.

### Pre-requis : Python 3.10+

```bash
python3 --version
```

Si Python < 3.10 (le defaut macOS 3.9.x), installer une version compatible :

macOS (Homebrew) :
```bash
brew install python@3.12
```
Puis utiliser `/opt/homebrew/bin/python3.12` (Apple Silicon) ou `/usr/local/bin/python3.12` (Intel).

Linux (apt) :
```bash
sudo apt update && sudo apt install -y python3.12 python3.12-venv
```

### Installer la CLI

Toujours via un venv (evite "externally-managed-environment" et les soucis de PATH) :

```bash
PYTHON=$(command -v python3.12 2>/dev/null || command -v python3.11 2>/dev/null || command -v python3.10 2>/dev/null || command -v python3)
$PYTHON -c "import sys; assert sys.version_info >= (3,10), f'Python {sys.version} trop vieux, besoin 3.10+'; print(f'Python {sys.version}')"
$PYTHON -m venv ~/.notebooklm-venv
source ~/.notebooklm-venv/bin/activate
pip install "notebooklm-py[browser]"
playwright install chromium
```

Symlink sur le PATH :
```bash
mkdir -p ~/bin
ln -sf ~/.notebooklm-venv/bin/notebooklm ~/bin/notebooklm
export PATH="$HOME/bin:$PATH"
notebooklm --help
```

### Authentifier

IMPORTANT : `notebooklm login` exige une entree terminal interactive (Entree apres connexion), que le bash tool de Claude Code ne supporte pas. Utiliser ce script de login a la place.

Dire a l'utilisateur :

> Je vais ouvrir une fenetre navigateur : connecte-toi a ton compte Google et va sur notebooklm.google.com. Prends ton temps, j'attends ta confirmation avant de fermer.

Puis :

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
    print("Navigateur ouvert. En attente du signal de sauvegarde...")
    while not SIGNAL_FILE.exists():
        time.sleep(1)
    storage = browser.storage_state()
    with open(STORAGE_PATH, "w") as f:
        json.dump(storage, f)
    print(f"Sauve {len(storage.get('cookies', []))} cookies")
    browser.close()

SIGNAL_FILE.unlink(missing_ok=True)
print(f"Auth sauvee : {STORAGE_PATH}")
PYEOF

source ~/.notebooklm-venv/bin/activate
python3 /tmp/nlm_login.py > /tmp/nlm_login_output.txt 2>&1 &
echo "Login lance (PID=$!). Navigateur dans quelques secondes..."
```

Attendre ~10s, demander a l'utilisateur s'il voit le navigateur et s'il est connecte. Une fois sur la home NotebookLM :

```bash
touch /tmp/nlm_save_signal
sleep 8
cat /tmp/nlm_login_output.txt
```

Verifier :
```bash
export PATH="$HOME/bin:$PATH"
notebooklm auth check
notebooklm list
```

Si OK (cookie SID present), confirmer et nettoyer :
```bash
rm -f /tmp/nlm_login.py /tmp/nlm_login_output.txt /tmp/nlm_save_signal
```

Si echec (SID manquant), reset profil et reessayer :
```bash
rm -rf ~/.notebooklm/browser_profile ~/.notebooklm/storage_state.json
```

## Usage depuis Hermes (Galaxia, optionnel)

NotebookLM n'est PAS branche en MCP sur Hermes (pas d'API query officielle, automation navigateur trop fragile pour un agent 24/7). Pour declencher une generation NotebookLM depuis le stack autonome, passer par **N8N** (workflow deterministe : reception d'une demande, lancement de la CLI sur une machine avec session, livraison du media). Garder NotebookLM hors du hot path de l'agent.

## Regles d'autonomie

Lancer sans confirmation :
- `notebooklm status`, `auth check`, `list`, `source list`, `artifact list`, `language list/get/set`, `artifact wait`, `source wait`, `research status/wait`, `use <id>`, `create`, `ask "..."` (sans `--save-as-note`), `history`, `source add`

Demander avant :
- `delete` (destructif), `generate *` (long, peut echouer), `download *` (ecrit sur disque), `ask "..." --save-as-note` (ecrit une note), `history --save`

## Reference rapide

| Tache | Commande |
|------|---------|
| Lister notebooks | `notebooklm list` |
| Creer notebook | `notebooklm create "Titre"` |
| Definir contexte | `notebooklm use <notebook_id>` |
| Etat contexte | `notebooklm status` |
| Ajouter URL | `notebooklm source add "https://..."` |
| Ajouter fichier | `notebooklm source add ./file.pdf` |
| Ajouter YouTube | `notebooklm source add "https://youtube.com/..."` |
| Lister sources | `notebooklm source list` |
| Attendre traitement source | `notebooklm source wait <source_id>` |
| Recherche web (rapide) | `notebooklm source add-research "query"` |
| Recherche web (profonde) | `notebooklm source add-research "query" --mode deep --no-wait` |
| Interroger | `notebooklm ask "question"` |
| Interroger (sources precises) | `notebooklm ask "question" -s src_id1 -s src_id2` |
| Interroger (avec citations) | `notebooklm ask "question" --json` |
| Sauver reponse en note | `notebooklm ask "question" --save-as-note` |
| Fulltext d'une source | `notebooklm source fulltext <source_id>` |
| Generer podcast | `notebooklm generate audio "instructions"` |
| Generer video | `notebooklm generate video "instructions"` |
| Generer rapport | `notebooklm generate report --format briefing-doc` |
| Generer quiz | `notebooklm generate quiz` |
| Generer flashcards | `notebooklm generate flashcards` |
| Generer infographie | `notebooklm generate infographic` |
| Generer mind map | `notebooklm generate mind-map` |
| Generer slides | `notebooklm generate slide-deck` |
| Etat artefact | `notebooklm artifact list` |
| Attendre fin | `notebooklm artifact wait <artifact_id>` |
| Telecharger audio | `notebooklm download audio ./output.mp3` |
| Telecharger video | `notebooklm download video ./output.mp4` |
| Telecharger slides PDF | `notebooklm download slide-deck ./slides.pdf` |
| Telecharger rapport | `notebooklm download report ./report.md` |

## Types de generation

Toutes les commandes `generate` supportent `-s/--source`, `--language`, `--json`, `--retry N`.

| Type | Commande | Options | Sortie |
|------|---------|---------|--------|
| Podcast | `generate audio` | `--format [deep-dive\|brief\|critique\|debate]`, `--length [short\|default\|long]` | .mp3 |
| Video | `generate video` | `--format [explainer\|brief]`, `--style [auto\|classic\|whiteboard\|kawaii\|anime\|watercolor\|retro-print\|heritage\|paper-craft]` | .mp4 |
| Slides | `generate slide-deck` | `--format [detailed\|presenter]`, `--length [default\|short]` | .pdf / .pptx |
| Infographie | `generate infographic` | `--orientation [landscape\|portrait\|square]`, `--detail [concise\|standard\|detailed]` | .png |
| Rapport | `generate report` | `--format [briefing-doc\|study-guide\|blog-post\|custom]`, `--append "..."` | .md |
| Mind Map | `generate mind-map` | (instantane) | .json |
| Quiz | `generate quiz` | `--difficulty [easy\|medium\|hard]`, `--quantity [fewer\|standard\|more]` | .json/.md/.html |
| Flashcards | `generate flashcards` | idem quiz | .json/.md/.html |

## Workflows courants

### Recherche vers podcast
1. `notebooklm create "Recherche: [sujet]"`
2. `notebooklm source add` pour chaque URL/document
3. Attendre : `notebooklm source list --json` jusqu'a status=ready
4. `notebooklm generate audio "Focus sur [angle]"`
5. `notebooklm artifact list` pour le statut
6. `notebooklm download audio ./podcast.mp3`

### Analyse gros corpus (le cas ou NotebookLM bat gbrain)
1. `notebooklm create "Analyse: [projet]"`
2. `notebooklm source add` (dizaines de PDF/URL)
3. `notebooklm ask "Synthetise les points cles" --json` (reponses citees)
4. Si pertinent durablement : reporter la synthese dans gbrain (`gbrain put`), pas la laisser dans NotebookLM.

## Gestion d'erreurs

| Erreur | Cause | Action |
|-------|-------|--------|
| Auth/cookie | Session expiree | Re-login (script Step 0) |
| "No notebook context" | Contexte non defini | `notebooklm use <id>` |
| Rate limit | Throttle Google | Attendre 5-10 min |
| Download echoue | Generation incomplete | `artifact list` |

## Limites connues

- Generation audio/video/quiz/flashcards/infographie/slides peut echouer (rate limits Google).
- Temps : audio 10-20 min, video 15-45 min, quiz/flashcards 5-15 min.
- API non officielle (automation navigateur) : Google peut casser sans prevenir. Ne pas en dependre pour un agent en production.
