---
name: wrapup
description: Cloture de session - resume la session, sauve la connaissance durable dans le brain gbrain (memoire canonique), met a jour les memoires Claude, et optionnellement genere un media NotebookLM (podcast/briefing). Se declenche sur /wrapup ou "wrap up", "save this session", "fin de session", "resume de session".
---

# Cloture de session (Galaxia)

A lancer en fin de session pour capturer ce qui s'est passe et le verser dans la memoire long-terme.

## Principe (lire avant)

La memoire canonique de Galaxia, c'est **gbrain** (Postgres+pgvector, vault Obsidian comme system of record). PAS NotebookLM. Ce wrapup ecrit donc la connaissance durable dans gbrain. NotebookLM reste optionnel : on l'utilise seulement pour generer un media de la session (podcast, briefing) quand ca a du sens, jamais comme stockage memoire.

## Step 0 : Verifier l'acces au brain

Le wrapup ecrit via la CLI `gbrain` (qui doit pointer sur le brain Galaxia). Verifier :

```bash
gbrain stats 2>&1 | head -3
```

Si `gbrain` n'est pas configure sur le bon backend, exporter la connexion (Postgres cloud) avant d'ecrire :
```bash
export DATABASE_URL='<connexion Postgres du brain>'   # ou la config gbrain locale
```

Si gbrain est injoignable : ne PAS bloquer. Sauver les memoires Claude localement et prevenir l'utilisateur que la capture brain est skippee.

## Step 1 : Revue de session

Relire toute la conversation et identifier :
- Decisions prises (quoi + pourquoi)
- Travail accompli (construit, fixe, configure, livre)
- Apprentissages cles (surprenant ou non-evident)
- Fils ouverts (a reprendre)
- Preferences utilisateur revelees (comment il aime bosser)

Filtre qualite : ne capturer que ce qui a du sens et sera utile plus tard. Le bavardage et l'ephemere ne vont pas dans le brain.

## Step 2 : Ecrire dans gbrain (memoire canonique)

Pour chaque element durable, ecrire une **page atomique** dans gbrain :

```bash
# Resume de session (un slug par jour, suffixe si plusieurs sessions)
cat > /tmp/session.md << 'MD'
---
title: "Session [sujet] - YYYY-MM-DD"
type: note
tags: [type/session, source/wrapup]
---

# Session [sujet] - YYYY-MM-DD

## Fait
- ...

## Decisions
- ... (quoi + pourquoi)

## Apprentissages
- ...

## Fils ouverts
- ...
MD
gbrain put "sessions/YYYY-MM-DD-[sujet]" < /tmp/session.md
```

Pour une decision structurante ou une entite nouvelle (projet/personne/societe) qui merite sa propre page, ecrire une page dediee plutot que de la noyer dans le resume. Lier les pages avec `gbrain link <from> <to>` si pertinent.

Regles : page atomique, titre clair, ne pas dupliquer (verifier d'abord via `gbrain search "..."`), dates absolues.

## Step 3 : Memoires Claude

Mettre a jour l'index memoire Claude (MEMORY.md + fichiers) comme d'habitude :
- feedback (corrections/approches confirmees, avec Why + How to apply)
- project (travail en cours, objectifs, contexte)
- user (role, preferences, savoir)
- reference (ressources/outils externes)

Ne pas dupliquer, ne pas sauver ce qui est derivable du code/git, dates absolues.

## Step 4 : Media NotebookLM (optionnel)

Si la session est significative et que l'utilisateur veut un media (sinon skip) :

```bash
# Pousser le resume comme source puis generer un briefing ou un podcast
notebooklm create "Galaxia Sessions" --json   # une seule fois, reutiliser l'id ensuite
notebooklm source add /tmp/session.md --notebook <id>
notebooklm generate audio "Resume cette session en deep-dive de 5 minutes"   # ou: generate report --format briefing-doc
```

Ne PAS faire de NotebookLM le stockage memoire : la source de verite reste gbrain. Ici NotebookLM ne sert qu'a produire un media ecoutable/lisible.

## Step 5 : Confirmer

Dire a l'utilisateur, brievement :
- Combien de pages ecrites dans gbrain + combien de memoires Claude maj
- Si un media NotebookLM a ete genere (ou skippe)
- Les fils ouverts a reprendre

## Gestion d'erreurs

- gbrain injoignable : sauver les memoires Claude, skip la capture brain, prevenir.
- Rien de significatif a sauver : le dire, ne pas forcer des memoires vides.
- `notebooklm` absent : skip le media (optionnel), ne pas bloquer la cloture.

## Pre-requis

- gbrain CLI configure sur le brain Galaxia (obligatoire pour la memoire).
- NotebookLM CLI (optionnel, pour le media) : voir le skill `notebooklm`.
