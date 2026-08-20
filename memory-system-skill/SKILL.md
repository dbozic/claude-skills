---
name: persistent-memory
description: Use when setting up or maintaining a cross-session memory system for an AI assistant — capturing user context, corrections/feedback, project state, or pointers to external systems so future conversations don't start from zero. Triggers on "remember this", "save this for later", "why do I keep repeating myself", "set up memory", "build a knowledge base for Claude".
---

# Persistent Memory

## Overview

A file-based memory system that lets an assistant carry context across sessions without bloating every conversation. Two tiers: a small always-loaded **index** (behavioral memories that apply on every turn) and on-demand **hub** files (domain knowledge, loaded only when relevant). The index stays cheap; depth lives one hop away.

Core principle: **write once, load selectively.** Everything gets a file; only the index is force-loaded every turn.

## When to Use

- You're repeating the same corrections to an assistant across sessions.
- You want an assistant to know your role/preferences without re-explaining them.
- A project has decisions, deadlines, or in-flight work that should inform future suggestions.
- You reference external systems (trackers, dashboards, docs) the assistant should know to check.

**Don't use for:** anything derivable from the current state of the world — code conventions (read the code), git history (`git log`/`git blame`), one-off task details, or content already in a project's own instructions file. If it's derivable or ephemeral, it doesn't belong in memory.

## The Four Memory Types

| Type | Captures | Decays? |
|---|---|---|
| **User** | Who they are — role, goals, knowledge level, how to tailor explanations | Slowly |
| **Feedback** | Corrections ("don't do X") and confirmations ("yes, keep doing Y") about *how* to work | Slowly |
| **Project** | Live decisions, ownership, deadlines — context not visible in the code/files themselves | Fast |
| **Reference** | Pointers to where live information lives in other systems (trackers, dashboards, docs) | Slowly |

**Save feedback from both directions.** Corrections are easy to notice ("stop doing X"). Confirmations are quieter — a preference validated without pushback ("yes exactly", accepting an unusual choice silently). Miss the confirmations and the assistant only ever learns what to avoid, never what already works; it drifts from approaches you already approved.

**Project and Reference memories need a "why," not just a fact.** A fact without motivation can't be judged for continued relevance later. Structure feedback/project entries as: the rule or fact, then **Why:** (the reasoning or triggering event), then **How to apply:** (when this should change behavior).

## Architecture

```
memory/
  MEMORY.md              # always-loaded index — one line per memory, links out
  MEMORY_ARCHITECTURE.md # (optional) conventions doc, if the system grows complex
  MEMORY_ARCHIVE.md       # (optional) stale/superseded entries, kept for history
  user_<topic>.md         # individual User memories
  feedback_<topic>.md     # individual Feedback memories
  HUB_<domain>.md         # domain hub — routes to project/reference files for one subject area
  <project-file>.md       # individual Project/Reference memories, linked from a hub
```

**MEMORY.md** is an index, not a memory. Every line is a one-line pointer: `- [Title](file.md) — one-line hook`. Cap it — once behavioral memories (User + Feedback) get long, everything else moves behind a hub, and MEMORY.md just lists the hubs by topic:

```markdown
# Memory Index

## User
- [Profile](user_profile.md) — role, background, current goal.

## Feedback
- [Working Style](feedback_working_style.md) — concise unless introducing a new concept.

## Domain Hubs
- Topic A → [HUB_topic_a.md](HUB_topic_a.md) — subtopics covered.
- Topic B → [HUB_topic_b.md](HUB_topic_b.md) — subtopics covered.
```

A **hub** is just a bigger index scoped to one subject: it lists the Project/Reference files under that subject with one-line hooks, and is only opened when that subject comes up. This is the mechanism that lets the system scale — new project knowledge gets its own file plus one line in a hub, never a growing wall of text in the always-loaded index.

## File Format

Every memory file (except MEMORY.md and hub files, which are pure indexes) uses:

```markdown
---
name: short-kebab-case-slug
description: one-line summary — used to judge relevance in future conversations
metadata:
  type: user | feedback | project | reference
---

{{content — for feedback/project, use: rule/fact, then **Why:**, then **How to apply:**}}
```

Link related memories inline with `[[other-memory-slug]]`. A link that doesn't resolve yet is fine — it flags something worth writing later, not an error.

## Saving a Memory (two steps, always)

1. Write the memory to its own file using the format above.
2. Add one line to `MEMORY.md` (or the relevant hub) pointing to it.

Never write memory content directly into the index — the index only ever holds pointers.

## Using Memory

- Memory is context, not a source of truth for the present. Before acting on a memory that names a specific file, function, flag, or system state, verify it still holds — things get renamed, removed, or change. "The memory says X exists" is not the same as "X exists now."
- If the user asks about *current* or *recent* state, prefer checking live sources over recalling a memory — a memory that summarizes a snapshot is frozen at the time it was written.
- If a memory conflicts with what you observe right now, trust the observation and update or delete the stale memory — don't act on it anyway.
- If the user says to ignore memory for this task, don't apply it, cite it, or mention it.

## What NOT to Save

- Anything derivable by reading current files, code, or history.
- Debugging solutions — the fix lives in the code/commit, not in memory.
- Content already covered by a project's own instructions file.
- Ephemeral task state (use a plan or task list for that instead — see below).

These exclusions hold even if explicitly asked to save something covered by them — if someone asks to persist a status summary or activity log, ask what was *surprising or non-obvious* about it; that's the part actually worth keeping.

## Memory vs. Other Persistence

Memory is one of several ways an assistant can carry state. Don't overload it:

- **Mid-task alignment on approach** → a plan, not memory.
- **Multi-step in-progress work** → a task list, not memory.
- **Anything future conversations should recall** → memory.

## Common Mistakes

| Mistake | Fix |
|---|---|
| Writing memory content straight into the index | Index only holds one-line pointers; content goes in its own file |
| Saving only corrections, never confirmations | Watch for quiet validation ("yes, keep doing that") — save it too |
| A project memory with no "why" | Add the motivating reason so future-you can judge if it's still load-bearing |
| Letting the index grow unbounded | Once a topic accumulates multiple files, promote it to a hub |
| Treating a memory as current truth | Verify file/function/flag still exists before acting on it |
| Duplicating an existing memory | Check the index/hub first; update in place instead of writing a new file |
