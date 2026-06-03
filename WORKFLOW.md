# Ember Paper Writing — Collaboration Workflow

This repository is used to record and organize a PhD student's research ideas in the direction of **cloud-native vector databases (with emphasis on tail latency)**, supporting paper writing.

## Directory Structure

| Path | Purpose |
|------|---------|
| `discussions/` | Raw records of each discussion (archived by date/session) |
| `summaries/` | Synthesized knowledge entries organized by topic |
| `related-work/` | Reading notes on related papers and systems |
| `WORKFLOW.md` | This file: human–agent collaboration guidelines (authoritative specification) |

## Research Background (Fixed Context)

- **Role**: Computer Science PhD student
- **Target System**: Cloud-native vector database with strong tail latency performance
- **Repository Purpose**: Accumulate ideas, surface conflicts, and support various sections of the paper (motivation, design, experiments, related work, etc.)

## What the Agent Should Do During Each Discussion

### Before the Discussion

1. Review recent and relevant historical records in `discussions/`
2. Review topic files in `summaries/` related to the current discussion
3. Review literature notes in `related-work/` related to the current topic
4. If accessible, reference historical agent transcripts from the same project
5. Base the discussion on existing knowledge to avoid redundant work or unnoticed contradictions

### During the Discussion

- Stay focused on the user's current question; reference conclusions from existing records when necessary
- Conduct web searches during the discussion to verify the reliability of the user's arguments and proactively raise issues the user may not have considered
- **If a new idea conflicts with existing conclusions in `summaries/` or `discussions/`**, explicitly list:
  - The point of conflict (old view vs. new view)
  - Potential impact on paper sections or design decisions
  - Ask the user to decide which view should take precedence

### After the Discussion (Mandatory)

1. Add or append a session record in `discussions/`, including:
   - Date and topic tags
   - Core user questions/ideas
   - Discussion points and reached consensus
   - Open questions and assumptions to be verified
   - Relation to prior records (continuation / revision / conflict pending decision)
2. If the user confirms that an **old view was incorrect**, update the corresponding topic file in `summaries/` and note in the discussion record that "a previous conclusion has been superseded"
3. If stable conclusions emerge, update or create new topic files under `summaries/`
4. All records should be written in English, using professional terminology and referencing professional blogs and research papers

## Periodic Synthesis

Actively perform topic-level synthesis at the following times or upon user request:

- When discussion records accumulate to a certain volume
- When the user explicitly requests synthesis
- When a topic appears across multiple discussions and views have stabilized

Synthesis principles:

- Split files by **topic** (e.g., `tail-latency.md`, `index-structure.md`), not by date
- Recommended structure for each topic file: Overview → Core Arguments → Design/Experiment Implications → Open Questions → Source Discussions (linking to filenames in `discussions/`)
- Do not delete historical discussions; corrections are made by updating the summary and adding an erratum note in the discussion record

## File Naming Conventions

- **Discussion**: `discussions/YYYY-MM-DD-<short-topic>.md`
- **Topic Summary**: `summaries/<short-english-topic-name>.md`
- **Related Work**: `related-work/<system-or-paper-abbreviation>-<conference-year>.md`

## Conflict Handling Process

```
New Idea ──► Compare with summaries / discussions
              │
              ├─ Consistent → Record as reinforcement/supplement
              ├─ Supplementary → Merge into corresponding summary
              └─ Conflict → Explicitly alert in response → User decides
                                    │
                                    ├─ New idea correct → Update summary, mark old discussion as "superseded"
                                    └─ Old idea correct → Mark new discussion as "rejected/deferred"
```

## Language

- All records and responses should be written in English
- Technical terms, system names, and paper titles may remain in English
