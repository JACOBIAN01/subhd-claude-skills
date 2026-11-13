---
name: "subhd-create-mcq"
description: "Create MCQ, Fill-in-the-Blank, Match-the-Columns and Comprehension questions on Newton School's Django admin (django.newtonschool.co) via the user's Chrome, with topic mappings; deliver the admin URLs where the user wants them."
---

# subhd-create-mcq

Creates **Multiple Choice**, **Fill in the Blanks**, **Match the Columns** and **Comprehension** questions on Newton School's internal Django admin, one at a time. Fills every relevant field, maps topics, and delivers each admin change URL.

Question add: `https://django.newtonschool.co/admin/assessments/multiplechoicequestion/add/`
Question change: `https://django.newtonschool.co/admin/assessments/multiplechoicequestion/<id>/change/`
Question list (duplicate search): `https://django.newtonschool.co/admin/assessments/multiplechoicequestion/?q=<text>`
Comprehension add: `https://django.newtonschool.co/admin/assessments/comprehension/add/`
Comprehension change: `https://django.newtonschool.co/admin/assessments/comprehension/<id>/change/`

Do **not** keep or write any run log.

## Step 0 — Before starting (once per batch, ONE AskUserQuestion call)
1. **Topic list (mandatory).** Ask for the subject's topic list (`Name  ID`). The numbers ARE topic IDs. Topics differ per subject; never guess or reuse another subject's list. Reuse it for the whole batch unless a new one is given.
2. **Question Utility Type**: Application / Always Application / Learning / Always Learning.
   - "Always …" applies to the whole batch; never ask again.
   - A plain "Application"/"Learning" applies to the first question only; ask again (alone) before each later question's mapping save.
3. **How the user wants the URLs:** In response / Google Sheet / Google Doc.

## Step 1 — Validate the input before ANY save
Check every question in the batch first. If anything is wrong, stop and tell the user in one short message what's wrong with which question. Don't save broken questions.
- **MCQ:** 2–5 options (A–E supported); exactly one correct letter, and it exists among the options.
- **FIB:** every blank has exactly one answer. Convert `____` blanks to `[[key]]` and make sure each key has an answer.
- **Match:** Column A and Column B have the same count, and the answer key is a complete one-to-one mapping (every A item matched, no B letter used twice).
- **Comprehension:** a passage plus at least one valid MCQ.
- **Topics:** if **no topic in the list reasonably fits** a question, ask the user which topic to use. Don't force a weak match.

## Delivering URLs
- **In response**: the final reply is **ONLY the change URL(s)**, one per line in input order, with no other words. For a comprehension: the comprehension URL first, then its MCQ URLs in order.
- **Google Sheet**: Google Sheets connector (read the google-workspace skill first). Columns: `#`, `Type` (MCQ/FIB/Match/Comprehension/Comp-MCQ), `Question` (short gist), `Admin URL`. Append one row per item. Reply with only the sheet link.
- **Google Doc**: Google Docs/Drive connector (read the google-workspace skill first). Append `<n>. <Type> — <gist> — <Admin URL>` per item. Reply with only the doc link.
- If the connector is unavailable, say so in one line and fall back to the URLs in the response.

The only other messages allowed are the Step 0 question, validation problems, a per-question utility ask (only if not "Always"), and a one-line error report if a save keeps failing.

## Hard constraints
- **No API.** The workspace is network-blocked from the portal. Always drive the user's **logged-in Chrome** (Claude in Chrome). Load the chrome-browser skill first; batch-load `tabs_context_mcp, navigate, javascript_tool, tabs_close_mcp`. If several browsers are connected, ask which one (it must be logged into django.newtonschool.co).
- **Use ONE tab for the whole batch.** Navigate it from question to question; close it only at the end.
- **Never use Import/Export.**
- **Use `javascript_tool` for everything** (set + read back). Screenshots, `read_page` and `get_page_text` time out on these heavy pages.
- **Sanitize JS return strings**: the extension blocks output containing `= & ? % ;`. Use `.replace(/[=&?%;]/g,' ')`. Never return csrf/hidden values.
- After clicking save, return immediately; in the **next** JS call `await new Promise(r=>setTimeout(r,3000))` before reading the result.
- A question supplied by the user = approval to save it.

## Difficulty & Bloom level — judge yourself, never ask
If given, use it. If missing, judge and assign it silently.
- Bloom levels: **L1 Remember · L2 Understand · L3 Apply · L4 Analyze · L5 Evaluate · L6 Create**. If the number and name conflict (e.g. "L1 · Understand"), **the name wins**.
  - Remember: recall a fact or term. Understand: explain why/how, interpret, match a concept to its role. Apply: use in a scenario, predict simple output. Analyze: trace, debug, break down architecture or cause and effect. Evaluate: choose and justify. Create: design (rare).
- Difficulty: **Beginner** (basic recall) · **Easy** (single concept, weak distractors) · **Medium** (real understanding, plausible distractors) · **Hard** (multi-step, subtle distractors) · **Challenge** (deep or tricky).

## Fixed defaults per type
| | MCQ | Fill in the Blanks | Match the Columns |
|---|---|---|---|
| question_type (by text) | Multiple Choice Question | Fill in the Blanks | Match the Columns |
| question_for_assessment_type | Normal + Competitive | Normal + Competitive | Normal + Competitive |
| shuffle_options | **ON** | **OFF** | **OFF** |
| answer_validator_rules | none | Space insensitivity + Case insensitivity | **none (empty)** |
| question_text | `<p>question</p>` | text with `[[key]]` blanks | `<p>instruction</p>` (no placeholders) |
| choices A–E / correct_choice | fill (up to 5) | empty | empty |
| correct_choice_explanation | fill | **empty** | fill only if the user gives one |
| key-value inline | — | per blank: name + correct_answer | per pair: name + display_text + correct_answer |
| user_generated / peer_reviewed / are_variables_enabled | off | off | off |

## Text handling
- **CKEditor fields** (`question_text`, `comprehension_text`) take HTML. **Escape user text first**: `&`→`&amp;`, `<`→`&lt;`, `>`→`&gt;`. Then wrap inline code (backticks) in `<code>…</code>` and the whole thing in `<p>…</p>`. Otherwise `<div>` or `a < b` gets swallowed as HTML.
- **Plain textareas** (choices, explanation, key-value fields): set the raw text, with no escaping and no HTML. Backticks are kept as-is.

## Field map — question form `#multiplechoicequestion_form`
- `question_text`: CKEditor `CKEDITOR.instances.id_question_text`. Use `setData(html)` then `updateElement()`.
- `question_type`, `difficulty_level`, `bloom_taxonomy`: selects. Pick by visible text.
- `question_for_assessment_type`, `answer_validator_rules`: plain `<select multiple>`. Set by text (pass `[]` to clear).
- `choice_A_text` … `choice_E_text`, `correct_choice_explanation`: plain textareas.
- `correct_choice`: **numeric values** (1=A … 5=E). Pick by text, never set `value='C'`.
- `topic_input`: raw-id M2M text, **comma-separated topic IDs**.
- Topic mappings inline: `#multiple_choice_question_topic_mappings-group`, fields `…-<i>-{topic,question_utility_type,relevance,main_topic}`.
- Key-value inline: `#multiple_choice_question_text_key_value_mappings-group`, fields `…-<i>-{name,display_text,correct_answer}`. It has 0 rows on /add/: add a row per item.

Helpers to include in each JS call (every setter throws a clear error instead of half-filling):
```js
const g=n=>{const e=document.querySelector(`[name="${n}"]`); if(!e) throw new Error('MISSING FIELD '+n); return e;};
const setv=(e,v)=>{e.value=v; e.dispatchEvent(new Event('input',{bubbles:true})); e.dispatchEvent(new Event('change',{bubbles:true}));};
const pick=(n,t)=>{const e=g(n); const o=[...e.options].find(o=>o.text.trim()===t); if(!o) throw new Error('NO OPTION '+n+' : '+t); e.value=o.value; e.dispatchEvent(new Event('change',{bubbles:true}));};
const multi=(n,ts)=>{const e=g(n); const have=[...e.options].map(o=>o.text.trim()); ts.forEach(t=>{if(!have.includes(t)) throw new Error('NO OPTION '+n+' : '+t)}); [...e.options].forEach(o=>o.selected=ts.includes(o.text.trim())); e.dispatchEvent(new Event('change',{bubbles:true}));};
const esc=s=>s.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;');
const html=s=>'<p>'+esc(s).replace(/`([^`]+)`/g,'<code>$1</code>')+'</p>';
const save=()=>{for(const k in (window.CKEDITOR?.instances||{})) CKEDITOR.instances[k].updateElement(); document.querySelector('input[name=_continue]').click();};
const realRows=sel=>[...document.querySelectorAll(sel+' tr.form-row:not(.empty-form)')].filter(r=>![...r.querySelectorAll('[name]')].some(e=>/__prefix__/.test(e.name)));
const addKV=(name,display,answer)=>{const sel='#multiple_choice_question_text_key_value_mappings-group'; document.querySelector(sel+' .add-row a').click(); const r=realRows(sel).pop(); setv(r.querySelector('[name$="-name"]'),name); if(display!=null) setv(r.querySelector('[name$="-display_text"]'),display); setv(r.querySelector('[name$="-correct_answer"]'),answer);};
```
Wrap each fill script in `try{…}catch(e){'ERROR: '+e.message}`, and **only call `save()` if no error was thrown**. On an error, stop and report it; don't save a half-filled form.

## A. MCQ / FIB / MATCH — THREE-SAVE PROCEDURE
**Topics can NOT be added on /add/.** Doing so fails with "Add topics after creating multiple choice question".

**Save #1 — create (topic_input EMPTY).** Navigate the batch tab to the question add URL, then in one JS call:
- **MCQ:** `CKEDITOR.instances.id_question_text.setData(html(q))`; `pick('question_type','Multiple Choice Question')`; `multi('question_for_assessment_type',['Normal Assessment','Competitive Assessment'])`; choices A–E as given; `pick('correct_choice','C')`; explanation; difficulty; bloom; `g('shuffle_options').checked=true`; `save()`.
- **FIB:** setData with escaped text containing `[[key]]`; `pick('question_type','Fill in the Blanks')`; assessment types; difficulty; bloom; `multi('answer_validator_rules',['Space insensitivity','Case insensitivity'])`; shuffle off; per blank `addKV(key, null, answer)` (key exactly matches `[[key]]`); `save()`.
- **Match:** `setData(html(instruction))`; `pick('question_type','Match the Columns')`; assessment types; difficulty; bloom; `multi('answer_validator_rules',[])`; shuffle off; explanation only if given. Per correct pair, in Column A order, call `addKV(key, ColumnA_text, matched_ColumnB_text)`: `key` is a short unique lowercase key, numbers and letters are stripped, and the answer key is resolved to texts. `save()`.

**Verify save #1:** wait 3s. Success = `location.pathname` matches `/multiplechoicequestion/<id>/change/` **and** `.messagelist .success` exists. If it's still on `/add/` with `.errornote`/`.errorlist`, fix and re-save (form values are retained).

**Duplicate guard:** if the result is unclear (timeout, 500 page, tab lost), **do not just recreate**. First fetch the list search `?q=<first ~6 distinctive words of the question>` and look for a matching question created just now. If found, continue on that `<id>/change/` page. Only create again if none exists.

**Save #2 — topics.** Standalone question: pick the **2–3 best-related topics**, ranked. Comprehension MCQ: the **single comprehension topic** (see B). Set `topic_input` = `ID1,ID2,ID3` and `save()`. Rows appear; **match them by `-topic` value**, not order.

**Save #3 — mappings.** Use the batch utility type from Step 0 (or ask now if it wasn't "Always"). Per row: `-question_utility_type` by text; `-relevance` = **10, 8, 6…** in rank order; `-main_topic` **only** on the relevance-10 row. Leave the blank extra rows untouched. `save()`, wait, then re-read and verify: type, difficulty, bloom, shuffle, validators, assessment types, topic rows and key-value rows. Fix and re-save if anything is wrong.

## B. COMPREHENSION (passage + its MCQs)
**Rules:**
- Pick **ONE best-fit topic** for the whole passage, at **relevance 10**.
- **Every MCQ of that comprehension uses exactly that same single topic**: relevance 10, Main Topic, same utility type.
- The comprehension's **difficulty**, **utility type** and **relevance** must match its MCQs. If the MCQs' difficulties differ, use the most common one.
- MCQ question text = just the question (no passage); the passage lives only in the comprehension.

**Order:**
1. **Create each MCQ first** with procedure A (3 saves) using the single topic. Note each MCQ id.
2. **Comprehension save #1 — create.** Navigate to the comprehension add URL. It has only `comprehension_text` (CKEditor `CKEDITOR.instances.id_comprehension_text`; `setData(html(passage))`; leave out any "Story: …" title line) and `difficulty_level` (required). `save()` -> `/comprehension/<id>/change/`. Use the duplicate guard as above (comprehension list `?q=`).
3. **Comprehension save #2 — topic ONLY.** Inline `#comprehension_topic_mappings-group`, fields `comprehension_topic_mappings-<i>-{topic,question_utility_type,relevance}`. There is no main_topic and no topic_input: type the topic ID into the blank row's `-topic`, set utility by text and relevance 10. **Do NOT touch the MCQ mapping inline in this save.** `save()`, verify the row has an id.
   - Mapping an MCQ in the same save fails with "comprehension <id> does not have topic <t> with same relevance and utility type".
   - Clearing a filled mapping row and re-saving caused a **500 Server Error** (nothing saved). On an error, reload the change page and redo cleanly.
4. **Comprehension save #3 — map MCQs.** Inline `#comprehension_multiple_choice_question_mappings-group` (0 rows initially): for each MCQ click `.add-row a`, then on the newest real row set `-multiple_choice_question` = MCQ id and `-order` = 1, 2, 3… `save()`.
5. Verify after reload: text, difficulty, topic row (topic:utility:10), MCQ rows (id:order), success message.

## Finish
After the last question, close the batch tab, then deliver all URL(s) as chosen in Step 0.

## Editing an existing item
Navigate to its change URL, change the needed fields with the same helpers, `save()`, verify, then deliver the URL the same way.

## Failure handling
- Fill script error (missing field or option): stop and report it in one line; nothing is saved.
- Save error: read the message, fix, and retry once. On a 500 page or unclear result, run the duplicate guard before redoing. If it still fails, report the exact error in one line.
- Tool timeouts: retry once with JS only; after 2–3 failures stop and tell the user.