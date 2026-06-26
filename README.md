# notebooklm-galaxia

Plugin Claude Code : acces programmatique complet a Google NotebookLM, positionne dans la stack Galaxia.

## Ce que ca fait

NotebookLM n'expose pas d'API de query officielle. Ce plugin s'appuie sur la CLI communautaire `notebooklm-py` (automation navigateur, cookies) pour piloter NotebookLM depuis Claude Code :

- Creer des notebooks, ajouter des sources (URL, YouTube, PDF, audio, video, image)
- Interroger un corpus avec reponses citees
- Generer tous les artefacts : podcast audio, video, slides, quiz, flashcards, infographie, mind map, rapport
- Telecharger les resultats (mp3, mp4, pdf, pptx, md, csv, json)

## Positionnement (important)

Dans la stack Galaxia, la **memoire de retrieval c'est gbrain** (Postgres+pgvector, vault Obsidian comme system of record). NotebookLM n'est PAS la memoire de l'agent. Ce plugin le cantonne a ses forces reelles :

- Generateur de **media** : podcasts, videos, slides, infographies.
- Analyse ponctuelle d'un **gros corpus externe** (dizaines de documents) avec citations.

Ce qu'on ne fait PAS : brancher NotebookLM en MCP sur l'agent autonome (Hermes), ni s'en servir comme stockage memoire. L'automation navigateur est trop fragile (rate limits, ToS, casse au moindre changement d'UI) pour un agent 24/7. Pour declencher une generation depuis le stack autonome, passer par N8N (workflow deterministe, hors hot path agent).

## Skills

| Skill | Role |
|-------|------|
| `notebooklm` | Acces CLI complet a NotebookLM (setup, auth, sources, query, generation, download). |
| `wrapup` | Cloture de session gbrain-first : ecrit la connaissance durable dans gbrain (memoire canonique), met a jour les memoires Claude, et genere optionnellement un media NotebookLM. |

## Installation

Le plugin gere le setup au premier usage (`/notebooklm`). Manuellement :

```bash
python3 -m venv ~/.notebooklm-venv
source ~/.notebooklm-venv/bin/activate
pip install "notebooklm-py[browser]"
playwright install chromium
ln -sf ~/.notebooklm-venv/bin/notebooklm ~/bin/notebooklm
```

Authentification : voir le skill `notebooklm` (login navigateur, capture de session).

## Pre-requis

- Python 3.10+
- Chromium (via `playwright install chromium`)
- Un compte Google avec NotebookLM
- Pour `wrapup` : la CLI `gbrain` configuree sur le brain Galaxia

## Credits

Base sur la CLI communautaire `notebooklm-py` et les skills communautaires NotebookLM / WrapUp, adaptes et repositionnes pour la stack Galaxia (gbrain comme memoire canonique). API NotebookLM non officielle, susceptible de casser sans preavis.

## Licence

Proprietary. (c) Alexandre Mallet / BULDEE.
