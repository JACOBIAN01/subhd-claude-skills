# subhd-claude-skills

Customized Claude skills for day-to-day work at Newton School.

| Skill | What it does |
|---|---|
| [`subhd-create-mbr`](#subhd-create-mbr) | Builds a multi-tab Google Doc MBR (Monthly Business Report) for any course. |
| [`subhd-create-mcq`](#subhd-create-mcq) | Creates MCQ, Fill-in-the-Blank, Match-the-Columns and Comprehension questions on Newton School's Django admin. |

## Install
1. Download this repository as a zip (Code > Download ZIP), or zip a single skill folder so `SKILL.md` is inside it.
2. In Claude, go to Customize > Skills and upload the zip.

Claude Code users: copy the skill folder into `~/.claude/skills/`.

---

## subhd-create-mbr

A Claude skill that builds a multi-tab Google Doc **MBR (Monthly Business Report)** for any course: monthly, mid-sem or end-sem reviews backed by evidence, RCSBE analysis (Recall, Comprehend, Solve, Build, Evaluate) and a decision for each finding.

### What it does
- Connects to Google Docs and Google Drive, creates the MBR doc, then checks read and update.
- Asks for course data once. Every input can be skipped; gaps are marked in the doc.
- If RCSBE tags are missing, offers three options: auto-generate, provide them, or omit RCSBE.
- Builds a 12-tab Google Doc: objectives, delivery, initiatives, retros, class hygiene, contests, at-risk students, projects, feedback, outcomes, targets, RCSBE appendix.

### Requirements
- A Claude account with skills enabled.
- Google Docs and Google Drive connectors connected to Claude.

### Use
Ask Claude: "Create an MBR for my course."

---

## subhd-create-mcq

A Claude skill that creates questions on Newton School's internal Django admin (`django.newtonschool.co/admin/assessments/`) by driving your logged-in Chrome, then returns each question's admin URL. Works for 1 question or 200.

### Question types
- **Multiple Choice**: options A–E, correct answer, explanation, shuffle on.
- **Fill in the Blanks**: `[[key]]` blanks with one answer each, space- and case-insensitive, shuffle off.
- **Match the Columns**: one key / display text / correct answer row per pair, shuffle off, no validators.
- **Comprehension**: creates the MCQs first, then the passage, then maps the MCQs with order 1, 2, 3…

### What it does
- Takes questions from anywhere: chat, a Google Doc, a Google Sheet, or a sheet index plus a doc. Question + options + correct option is enough.
- Works out the rest: question type, difficulty, Bloom level, topics, and a short explanation when none is given.
- Given only a topic, writes the questions, shows the drafts, and creates them after you approve.
- Asks once, only for what's missing: topic list, utility type (Application / Learning, or "Always"), where the links go, and whether to include spare rows.
- Validates every question before saving (answer keys, blanks, pairs, topic fit).
- Always sets Normal + Competitive assessment types.
- Maps 2–3 best-fit topics (relevance 10/8/6, main topic on the top one); a comprehension and its MCQs share one topic.
- Follows the portal's required save order (create, then topics, then mappings) and checks every saved question.
- For big batches, uses a resumable runner kept in the browser, so a Chrome disconnect never loses work or creates duplicates.
- Delivers links in the reply, next to each question in a sheet column, as new sheet rows, or in a Google Doc.

### Requirements
- Claude in Chrome connected to a Chrome browser that is logged into `django.newtonschool.co`.
- Google Sheets / Google Docs connectors only if the questions come from there or you want links delivered there.

### Use
Run `/subhd-create-mcq` and paste your questions, share a doc or sheet link, or name a topic.

---

## Files
- `subhd-create-mbr/SKILL.md`
- `subhd-create-mcq/SKILL.md`
