---
name: wrapup
description: End-of-session wrap-up. Summarizes the session, saves durable knowledge into the gbrain brain (canonical memory), updates Claude memories, and optionally generates a NotebookLM media artifact (podcast/briefing). Triggers on /wrapup or "wrap up", "save this session", "end of session", "session summary".
---

# Session wrap-up (Galaxia)

Run at the end of a session to capture what happened and commit it to long-term memory.

## Principle (read first)

Galaxia's canonical memory is **gbrain** (Postgres+pgvector, with the Obsidian vault as the system of record). NOT NotebookLM. This wrap-up writes durable knowledge into gbrain. NotebookLM stays optional: use it only to generate a media artifact of the session (podcast, briefing) when it makes sense, never as memory storage.

## Step 0: Check brain access

The wrap-up writes via the `gbrain` CLI (which must point at the Galaxia brain). Check:

```bash
gbrain stats 2>&1 | head -3
```

If `gbrain` is not pointed at the right backend, export the connection (cloud Postgres) before writing:
```bash
export DATABASE_URL='<brain Postgres connection string>'   # or the local gbrain config
```

If gbrain is unreachable: do NOT block. Save the Claude memories locally and tell the user the brain capture is skipped.

## Step 1: Review the session

Re-read the whole conversation and identify:
- Decisions made (what + why)
- Work completed (built, fixed, configured, shipped)
- Key learnings (surprising or non-obvious)
- Open threads (to pick up later)
- User preferences revealed (how they like to work)

Quality filter: only capture what is meaningful and will be useful later. Small talk and ephemeral exchanges do not go into the brain.

## Step 2: Write into gbrain (canonical memory)

For each durable item, write an **atomic page** into gbrain:

```bash
# Session summary (one slug per day, suffix if multiple sessions)
cat > /tmp/session.md << 'MD'
---
title: "Session [topic] - YYYY-MM-DD"
type: note
tags: [type/session, source/wrapup]
---

# Session [topic] - YYYY-MM-DD

## Done
- ...

## Decisions
- ... (what + why)

## Learnings
- ...

## Open threads
- ...
MD
gbrain put "sessions/YYYY-MM-DD-[topic]" < /tmp/session.md
```

For a structural decision or a new entity (project/person/company) that deserves its own page, write a dedicated page rather than burying it in the summary. Link pages with `gbrain link <from> <to>` when relevant.

Rules: atomic page, clear title, do not duplicate (check first with `gbrain search "..."`), absolute dates.

## Step 3: Claude memories

Update the Claude memory index (MEMORY.md + files) as usual:
- feedback (corrections/confirmed approaches, with Why + How to apply)
- project (ongoing work, goals, context)
- user (role, preferences, knowledge)
- reference (external resources/tools)

Do not duplicate, do not save what is derivable from code/git, use absolute dates.

## Step 4: NotebookLM media (optional)

If the session is significant and the user wants a media artifact (otherwise skip):

```bash
# Push the summary as a source, then generate a briefing or podcast
notebooklm create "Galaxia Sessions" --json   # once, reuse the id afterwards
notebooklm source add /tmp/session.md --notebook <id>
notebooklm generate audio "Summarize this session as a 5-minute deep-dive"   # or: generate report --format briefing-doc
```

Do NOT make NotebookLM the memory store: the source of truth stays gbrain. Here NotebookLM only produces a listenable/readable media artifact.

## Step 5: Confirm

Tell the user, briefly:
- How many pages written to gbrain + how many Claude memories updated
- Whether a NotebookLM media artifact was generated (or skipped)
- Open threads to pick up next time

## Error handling

- gbrain unreachable: save Claude memories, skip the brain capture, tell the user.
- Nothing significant to save: say so, do not force empty memories.
- `notebooklm` missing: skip the media (optional), do not block the wrap-up.

## Prerequisites

- gbrain CLI pointed at the Galaxia brain (required for memory).
- NotebookLM CLI (optional, for media): see the `notebooklm` skill.
