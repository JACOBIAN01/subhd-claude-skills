---
name: subhd-create-assessment
description: Create an Assessment (a collection of existing MCQs, FIB, Match or Comprehension questions) on Newton School's Django admin via the user's Chrome, non-competitive or proctored competitive, for FSD LU (278) or a batch course; reply with the admin URL.
---

# subhd-create-assessment

An **Assessment** is a collection of MCQ-type questions (MCQ, Fill in the Blanks, Match the Columns, Comprehension MCQs) used as a quiz, exam or contest. It is built from questions that **already exist** (create them first with `subhd-create-mcq`). Coding questions go into Assignments, not here.

Non-competitive add: `https://django.newtonschool.co/admin/assessments/assessment/add/`
Competitive (proctored) add: `https://django.newtonschool.co/admin/assessments/assessment/add/?_preset=nst`
Change: `https://django.newtonschool.co/admin/assessments/assessment/<id>/change/`
Search: `https://django.newtonschool.co/admin/assessments/assessment/?q=<title>`

Do **not** keep or write a run log.

## Step 0 — ALWAYS ask first (one AskUserQuestion call)
1. **Non-competitive or Competitive (proctored)?**
2. **Where to release: FSD LU or another course?** FSD LU = course code **278** ("Simulation Course FSD LU", for testing). Otherwise ask for the course/batch code(s) (the user types them via "Other").

Also collect, in the same call or from the user's message, only what is missing:
- **Title.**
- **MCQ IDs** in the order they should appear (IDs, admin links, or a list from `subhd-create-mcq` output; pull the number out of each `/multiplechoicequestion/<id>/change/` link).
- **Start and end** date and time (IST, as shown in the admin).
- Optional: instructions (competitive gets default proctoring instructions if none are given), sorting (default Sequential), duration (only if the user asks for a duration-style test).

Never guess the course code or the schedule.

## Step 1 — Check before any save
- **Course:** fetch `/admin/courses/course/<code>/change/`. It must return the course page; read its title. A missing course stops the run. Use the title to sanity-check: a code the user called FSD must be 278.
- **MCQs:** fetch each `/admin/assessments/multiplechoicequestion/<id>/change/`. Every ID must exist. Keep the first ~30 characters of each question's plain text; that is how the position rows are matched later.
- If anything fails, report all problems in one message and stop.

## Fixed defaults
| Field | Non-competitive (plain `/add/`) | Competitive (`/add/?_preset=nst`) |
|---|---|---|
| assessment_type | **Normal Assessment** (the page defaults to Duration; change it) | **Competitive Assessment** (preset) |
| is_proctored_exam | off | **on** (preset) |
| auto_unhide | off | on (preset: unhides 1 hour before start) |
| hidden | on (default) | on (preset) |
| show_honor_code | **off** (Django rejects it on Normal: "Honor code can only be shown for competitive and duration assessments") | **on** |
| random_multiple_choice_questions | **OFF always** (the page defaults to on) | **OFF always** |
| sorting_type | Sequential unless asked | Sequential unless asked |
| sub_type | General | General |
| generation_and_creation_type | Fully Automatic | Fully Automatic |
| max_attempts / clearing_marks | 1 / 0 | 1 / 0 |
| marking_scheme | Traditional Marking | Traditional Marking |
| instructions (required, JSON list) | `[""]` if none (`[]` is rejected) | given lines, else `["1. Stay in fullscreen for the entire exam.", "2. Tab switches and fullscreen exits are recorded as proctoring breaches."]` |
| course | the code from Step 0 | the code from Step 0 |
| duration, sub_title, syllabus, lecture_slot, topic_input, available_to_courses_input | empty | empty |

Proctoring settings (devices [4], permissions [4, 6, 10], breach thresholds) come automatically with the competitive preset. They are read-only on the form; don't try to set them.

## Step 2 — Create (three saves)
Form `#assessment_form`. Save with `#assessment_form input[name=_continue]`.

**Save 1, blank assessment with settings:** open the right add URL for the mode. Fill title, instructions, `course`, `start_timestamp_0` (YYYY-MM-DD), `start_timestamp_1` (HH:MM:SS), `end_timestamp_0`, `end_timestamp_1`, and every default above. For competitive, first check that the preset applied (`is_proctored_exam`, `auto_unhide`, `hidden` all checked). **No MCQs in this save.** Success = `/assessment/<id>/change/` + `.messagelist .success`. On errors the page stays on `/add/` with values kept; read `.errornote, .errorlist li`, fix and re-save.

**Save 2, MCQs:** `multiple_choice_question_input` is a select2 autocomplete `<select multiple>`; add the IDs directly, with no searching:
```js
const s=document.querySelector('#assessment_form [name=multiple_choice_question_input]');
ids.forEach(id=>{let o=[...s.options].find(x=>x.value===id); if(!o){o=new Option(id,id,true,true); s.add(o);} o.selected=true;});
s.dispatchEvent(new Event('change',{bubbles:true}));
```
Save. Mapping rows appear in `#assessment_multiple_choice_question_mappings-group`.

**Save 3, positions:** Django does **not** keep the input order. Each saved row has only `-id`, `-assessment` and `-position` inputs plus the question's display text. Match every row to its MCQ by the question-text prefix from Step 1 and set `-position` = 1, 2, 3… in the user's order. All rows must match. Save.

**Verify** after reload: success message, no errors, title, course, start/end, assessment type, proctored, hidden, auto-unhide, honor code, random off, sorting, instructions, and the MCQ rows `position:question` in the expected order with the right count. Max marks (read-only) is computed from the questions.

**Several courses:** create one assessment per course code, repeating saves 1–3 for each. Don't use "Available to courses" (it makes view-only copies) unless the user asks.

## Browser rules
- **No API:** drive the user's logged-in Chrome (Claude in Chrome); load the chrome-browser skill first. Batch-load `tabs_context_mcp, navigate, javascript_tool, tabs_close_mcp, list_connected_browsers, select_browser`.
- If several browsers are connected, ask which one. If a page redirects to `/admin/login/`, ask the user to log in (never type passwords) or pick another browser.
- One tab for the whole run; close it at the end. Use `javascript_tool` for everything (screenshots time out on these pages).
- The extension blocks results containing `= & ? % ;`. End results with `.replace(/[=&?%;]/g,' ')`. Never return csrf or hidden values.
- Click save as the last action of a call; in the next call wait ~1.5–2 s, then read the result. Wrap each fill in `try{…}catch(e){'ERROR: '+e.message}` and only save if nothing threw.
- **Duplicate guard:** if a save result is unclear (timeout, 500, lost tab), search `?q=<title>` before creating again, and continue on the found ID.

## Reply
Only the assessment change URL(s), one per line (one per course when there are several). The only other messages allowed: the Step 0 question, Step 1 problems, and a one-line error report if a save keeps failing.

## Editing an existing assessment
Open its change URL. Change settings, or add MCQ IDs with the save-2 snippet, then re-run save 3 for positions. Verify, then reply with the URL. Fields can be read-only on assessments you don't own ("View assessment"); report that instead of retrying.
