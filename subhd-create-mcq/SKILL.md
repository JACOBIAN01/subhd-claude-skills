---
name: "subhd-create-mcq"
description: "Create MCQ, Fill-in-the-Blank, Match-the-Columns and Comprehension questions on Newton School's Django admin (django.newtonschool.co) via the user's Chrome, with topic mappings; deliver the admin URLs where the user wants them."
---

# subhd-create-mcq

Creates questions on Newton School's Django admin and returns their admin links. Works for 1 question or 200.

Question add: `https://django.newtonschool.co/admin/assessments/multiplechoicequestion/add/`
Question change: `https://django.newtonschool.co/admin/assessments/multiplechoicequestion/<id>/change/`
Question search: `https://django.newtonschool.co/admin/assessments/multiplechoicequestion/?q=<text>`
Comprehension add/change: `https://django.newtonschool.co/admin/assessments/comprehension/add/` · `.../comprehension/<id>/change/`

Do **not** keep or write a run log.

## The whole flow (4 steps)

### 1. Get the questions
There is no standard input format. Questions can come from chat, a Google Doc, a Google Sheet, or a sheet plus a doc. Usually they are just **question + options + correct option**. Read them however they come:
- Chat: read the message.
- Google Doc / Sheet: read with the Google connector (load the google-workspace skill first). For many questions, parse with a script, not by hand. Clean invisible spaces (U+00A0) and markdown backslash escapes (`\<`, `\_`, `\[`, `\!`, `\#`).
- Sheet index + doc: match the rows by question ID. If both give an answer, they must agree.

**Only a topic given?** Write the questions yourself: a sensible count and type if not given (default 5 MCQs), 4 options, one clearly correct answer, plausible wrong options, and a short explanation. **Show the drafts in chat and wait for approval before creating anything.** Questions the user supplied need no approval; supplying them is the approval.

### 2. Figure out the rest yourself
- **Type:** options → MCQ; a blank (`____`) → Fill in the Blanks; two columns + a mapping → Match the Columns; a passage with questions under it → Comprehension.
- **Difficulty & Bloom:** use them if given; otherwise judge them silently (rules below).
- **Explanation:** if not given, write a short one (one or two sentences on why the answer is right). FIB explanation stays empty.
- **Topics:** pick from the user's topic list (rules below).

Check everything before ANY save, and report all problems in ONE message:
- MCQ: 2–5 options and exactly one correct letter, which exists among the options.
- FIB: every blank has one answer.
- Match: equal column counts and a complete one-to-one answer key.
- Comprehension: a passage plus at least one valid MCQ.

**Ask only what is missing, once, in ONE AskUserQuestion call:**
- **Topic list** (`Name  ID`), if not already given for this subject. Never guess IDs or reuse another subject's list. If topic names in the source have no ID in the list, list them with the questions that use them and offer: paste the IDs / use the closest listed topic. Flag a name that appears twice with different IDs.
- **Utility type:** Application / Always Application / Learning / Always Learning. "Always" covers the whole batch. A plain answer covers the first question only; ask again before each later question's mapping save.
- **Where to put the links**, if the user hasn't said. Default: in the reply.
- **Optional rows** (spares/backups): ask in plain words whether to include them, only if the source has any.

### 3. Create them on Django
First check the browser (below). Then use procedure A for each question and procedure B for passages. For more than ~5 questions, drive A and B with the **resumable runner**, so a disconnect never loses work or creates duplicates. For 1–5 questions, the same steps can be done call by call with the runner's helper functions.

### 4. Send the links
- **In the reply:** ONLY the change URLs, one per line, in input order. For a comprehension: its URL first, then its MCQ URLs.
- **Next to each question in a sheet** (e.g. a new column "Django Link"): write the header, then one URL per row, matched by question ID, never by position. A passage row gets the comprehension URL.
- **New rows in a sheet:** columns `#`, `Type`, `Question` (short gist), `Admin URL`.
- **Google Doc:** append `<n>. <Type> — <gist> — <Admin URL>`.
- Long runs: write finished links partway through as a checkpoint. At the end, read the destination back and compare it with the expected list. If the connector is down, say so in one line and put the links in the reply.

The only other messages allowed: the one setup question, validation problems, draft approval (topic-only mode), a per-question utility ask (only if not "Always"), and a one-line error report.

## Judging difficulty, Bloom and topics
- **Bloom:** L1 Remember · L2 Understand · L3 Apply · L4 Analyze · L5 Evaluate · L6 Create. If the number and name conflict, the name wins.
  - Remember: recall a fact. Understand: explain why/how, interpret. Apply: use in a scenario, predict simple output. Analyze: trace, debug, cause and effect. Evaluate: choose and justify. Create: design (rare).
- **Difficulty:** Beginner (basic recall) · Easy (one concept, weak wrong options) · Medium (real understanding, plausible wrong options) · Hard (multi-step, subtle wrong options) · Challenge.
- **Topics (standalone question):** 2–3 best fits, ranked, relevance 10/8/6, main topic on the first. If the source says `Parent › Child`, use the child first, then the parent. If only one topic fits, add its parent or nearest related topic. If nothing fits, ask; don't force a weak match.
- **Comprehension:** ONE topic for the passage. Every MCQ of that passage uses exactly that topic, at relevance 10, as Main Topic, with the same utility type. The passage's difficulty = the most common difficulty of its MCQs.

## Browser rules
- **No API:** the workspace can't reach the portal. Drive the user's logged-in Chrome (Claude in Chrome); load the chrome-browser skill first. Batch-load `tabs_context_mcp, navigate, javascript_tool, tabs_create_mcp, tabs_close_mcp, list_connected_browsers, select_browser`.
- **Check the login before anything else.** If several browsers are connected, ask which one. Open the question add URL; if it goes to `/admin/login/`, ask the user to log in (never type passwords) or pick another browser.
- Use ONE tab for the whole batch; close it at the end. Never use Import/Export. No iframes (the admin blocks them).
- Use `javascript_tool` for everything (screenshots, `read_page` and `get_page_text` time out on these pages).
- The extension blocks results containing `= & ? % ;`. Always end with `.replace(/[=&?%;]/g,' ')`. Never return csrf or hidden values.
- A save reloads the page and ends the script. Click save as the last action of a call; in the next call wait ~1.5–3 s, then read the result. Never save a half-filled form: if any fill step throws, stop and report.
- **Disconnects:** "can't determine which page" → call `tabs_context_mcp`. Extension gone → `list_connected_browsers` and ask the user to wake that computer; don't switch to a browser that isn't logged in. On return, read the runner state before doing anything.

## Fixed defaults
| | MCQ | Fill in the Blanks | Match the Columns |
|---|---|---|---|
| question_type (by text) | Multiple Choice Question | Fill in the Blanks | Match the Columns |
| question_for_assessment_type | Normal + Competitive | Normal + Competitive | Normal + Competitive |
| shuffle_options | ON | OFF | OFF |
| answer_validator_rules | none | Space insensitivity + Case insensitivity | none |
| question_text | the question | text with `[[blank1]]` | the instruction only |
| choices / correct_choice | fill | empty | empty |
| correct_choice_explanation | given or written | empty | given or written |
| key-value rows | — | name + correct_answer (no display_text field) | name + display_text (Column A) + correct_answer (matched Column B text) |
| user_generated / peer_reviewed / are_variables_enabled | off | off | off |

- With Case insensitivity on, Django saves FIB answers in **lowercase** (`BrowserWindow` → `browserwindow`). Compare FIB answers ignoring case when verifying.
- Match keys: short unique lowercase words (`one`, `two`…). Strip `1.` / `P.` prefixes from the column texts.

## Text → HTML (CKEditor fields: question_text, comprehension_text)
- Escape `&`, `<`, `>` first, otherwise `<div>` or `a < b` gets swallowed as HTML. Then: paragraphs → `<p>`; inline backticks → `<code>`; code blocks or a command on its own line → `<pre><code>…</code></pre>` (an FIB `[[blank1]]` may sit inside); tables → `<table border="1" cellpadding="4">` with `<thead>`; lists → `<ul>`/`<ol>`; bold → `<strong>`. Word bank → `<p><strong>Word bank:</strong> a · b · c</p>`.
- When wrapping backticks in code, use a callback, `(m,c)=>'<code>'+c+'</code>'`. Never a string replacement with a dollar-digit pattern: skill argument substitution overwrites it.
- MCQ text inside a comprehension = just the question; the passage lives only in the comprehension. Leave out any "Story: …" title line of a passage.
- Plain textareas (choices, explanation, key-value fields) get raw text, with no escaping and no HTML; backticks are kept as-is.
- Build the HTML with a script for anything bigger than a few lines.

## Field map — question form `#multiplechoicequestion_form`
- `question_text`: `CKEDITOR.instances.id_question_text` → `setData(html, {callback})`, then `updateElement()` before save.
- Selects, picked by visible text: `question_type`, `difficulty_level`, `bloom_taxonomy`, `correct_choice` (values are 1–5; never set `'C'` as a value). Multi-selects: `question_for_assessment_type`, `answer_validator_rules`.
- Textareas: `choice_A_text` … `choice_E_text`, `correct_choice_explanation`.
- `topic_input`: comma-separated topic IDs (only after the question exists).
- Topic rows `#multiple_choice_question_topic_mappings-group`: `-topic` (text), `-question_utility_type` (select: Learning/Application/…), `-relevance` (number), `-main_topic` (checkbox).
- Key-value rows `#multiple_choice_question_text_key_value_mappings-group`: 0 rows on /add/; click `.add-row a` per item.

## A. Question — three saves
Topics can NOT be added on /add/ ("Add topics after creating multiple choice question").
1. **Save 1, create:** on /add/ fill text, type, assessment types, difficulty, Bloom, flags, plus choices/answer/explanation (MCQ) or validators + key-value rows (FIB/Match). Leave topic_input empty. Save and continue. Success = URL `/multiplechoicequestion/<id>/change/` + `.messagelist .success`. If it's still on /add/ with errors, fix and re-save (values are kept).
2. **Save 2, topics:** set `topic_input` = `ID1,ID2,ID3` and save. Rows appear; match them by `-topic` value, not order.
3. **Save 3, mappings:** per row set utility by text, relevance 10/8/6 by rank, main_topic only on the first. Leave blank extra rows alone. Save.
4. **Verify** after reload: type, difficulty, Bloom, assessment types, shuffle, validators, answer, choices, explanation, key-value rows, topic rows (topic:utility:relevance:main), and text length.

**Duplicate guard:** if a result is unclear (timeout, 500 page, lost tab), don't recreate. Search the list (`?q=` first ~6 distinctive words) and continue on the found ID.

## B. Comprehension — after its MCQs exist
1. Create every MCQ of the passage with A, using the single passage topic. Note their IDs.
2. **Save 1:** comprehension /add/: `CKEDITOR.instances.id_comprehension_text` + `difficulty_level` (required). Save → `/comprehension/<id>/change/`. Duplicate guard via the comprehension list `?q=`.
3. **Save 2, topic ONLY:** `#comprehension_topic_mappings-group` already has one blank row: `-topic` = ID, utility by text, `-relevance` = 10 (there is no main_topic or topic_input here). Don't touch the MCQ rows in this save (it fails with "does not have topic … with same relevance"). Don't clear a filled row and re-save (gives a 500 error; reload and redo cleanly).
4. **Save 3, MCQs:** `#comprehension_multiple_choice_question_mappings-group` (0 rows): per MCQ click `.add-row a`, set `-multiple_choice_question` = ID and `-order` = 1, 2, 3… Save.
5. **Verify:** difficulty, topic row = `topic:<utility>:10`, MCQ rows `id:order`, text length, success message.

## Resumable runner (use for more than ~5 questions)
The progress note lives in the page's `localStorage`, so it survives reloads and disconnects.

**Load the questions:** build a JSON array with a script. Fields: `id, type ('MCQ'|'FIB'|'Match'), html, difficulty, bloom, topics:[ids], comp_topic?, choices:{A..E}, answer, explanation, blanks:[[key,answer]], pairs:[[key,colA,colB]]`. Send ~20 per call:
```js
const a=[/*chunk*/]; const s=JSON.stringify(a); let h=0; for(const ch of s) h=(Math.imul(h,31)+ch.charCodeAt(0))>>>0;
localStorage.setItem('__c0',s); a.length+' '+s.length+' '+h
```
Compare the length and hash with the same computation done locally (Python: compact `json.dumps(ensure_ascii=False, separators=(',',':'))`, `h=(h*31+ord(ch))&0xffffffff`). Then merge the chunks into `__qs` and set `__st = {i:0, stage:'fill', qid:null, res:{}, done:[]}` (add `util:'Learning'` if that is the batch utility type).

**Store the runner once.** Wrap the code below in `const __def=function(){ … };` and save `localStorage.setItem('__h','('+__def.toString()+')()')`. Then every call is:
```js
await new Promise(r=>setTimeout(r,1500)); eval(localStorage.__h); (await __step()).replace(/[=&?%;]/g,' ')
```
Repeat until it returns `DONE` or `HALT …`. On HALT, fix the problem by hand, set `__st.stage` to the right step, and continue. After a disconnect, read `__st` first: `fill` = nothing submitted; `chkN` = re-run that step (it is safe to repeat).

```js
window.__step = async function(){
  const sleep=ms=>new Promise(r=>setTimeout(r,ms)); const BASE='/admin/assessments/multiplechoicequestion/';
  const QS=JSON.parse(localStorage.getItem('__qs')); const st=JSON.parse(localStorage.getItem('__st'));
  const put=()=>localStorage.setItem('__st',JSON.stringify(st));
  if(window.__saving){await sleep(3000); return 'pending';}
  for(let k=0;k<40;k++){ if(document.readyState==='complete'&&(!document.getElementById('id_question_text')||(window.CKEDITOR&&CKEDITOR.instances.id_question_text&&CKEDITOR.instances.id_question_text.status==='ready'))) break; await sleep(500);} await sleep(500);
  if(st.stage==='halt') return 'HALTED: '+st.err; if(st.i>=QS.length) return 'DONE';
  const q=QS[st.i]; const ev=(e,t)=>e.dispatchEvent(new Event(t,{bubbles:true}));
  const g=n=>{const e=document.querySelector('[name="'+n+'"]'); if(!e) throw new Error('MISSING FIELD '+n); return e;};
  const setv=(e,v)=>{e.value=v; ev(e,'input'); ev(e,'change');};
  const pickEl=(e,t)=>{const o=[...e.options].find(o=>o.text.trim()===t); if(!o) throw new Error('NO OPTION '+e.name+' : '+t); e.value=o.value; ev(e,'change');};
  const pick=(n,t)=>pickEl(g(n),t);
  const multi=(n,ts)=>{const e=g(n); const have=[...e.options].map(o=>o.text.trim()); ts.forEach(t=>{if(!have.includes(t)) throw new Error('NO OPTION '+n+' : '+t)}); [...e.options].forEach(o=>o.selected=ts.includes(o.text.trim())); ev(e,'change');};
  const realRows=s=>[...document.querySelectorAll(s+' tr.form-row:not(.empty-form)')].filter(r=>![...r.querySelectorAll('[name]')].some(e=>/__prefix__/.test(e.name)));
  const savedRows=s=>realRows(s).filter(r=>{const i=r.querySelector('[name$="-id"]'); return i&&i.value;});
  const KV='#multiple_choice_question_text_key_value_mappings-group', TM='#multiple_choice_question_topic_mappings-group';
  const addKV=(n,d,a)=>{document.querySelector(KV+' .add-row a').click(); const r=realRows(KV).pop(); setv(r.querySelector('[name$="-name"]'),n); if(d!=null) setv(r.querySelector('[name$="-display_text"]'),d); setv(r.querySelector('[name$="-correct_answer"]'),a);};
  const save=()=>{for(const k in (window.CKEDITOR&&CKEDITOR.instances||{})) CKEDITOR.instances[k].updateElement(); window.__saving=true; document.querySelector('#multiplechoicequestion_form input[name=_continue]').click();};
  const errs=()=>[...document.querySelectorAll('.errornote, .errorlist')].map(e=>e.textContent.trim()).join(' / ');
  const okId=()=>{const m=location.pathname.match(/multiplechoicequestion\/(\d+)\/change\//); return m&&document.querySelector('.messagelist .success')?m[1]:null;};
  const halt=m=>{st.stage='halt'; st.err=q.id+' '+m; put(); return 'HALT '+st.err;};
  const topics=q.comp_topic?[q.comp_topic]:q.topics; const UTIL=st.util||'Application';
  const TYPE={MCQ:'Multiple Choice Question',FIB:'Fill in the Blanks',Match:'Match the Columns'}[q.type];
  try{
    if(st.stage==='fill'){
      if(!/\/multiplechoicequestion\/add\/$/.test(location.pathname)){window.__saving=true; location.href=BASE+'add/'; return 'navigating';}
      if(document.querySelector('.errornote')) return halt('errors on add page: '+errs());
      await new Promise(r=>CKEDITOR.instances.id_question_text.setData(q.html,{callback:r}));
      pick('question_type',TYPE); multi('question_for_assessment_type',['Normal Assessment','Competitive Assessment']);
      pick('difficulty_level',q.difficulty); pick('bloom_taxonomy',q.bloom);
      ['user_generated','peer_reviewed','are_variables_enabled'].forEach(n=>{g(n).checked=false;});
      if(q.type==='MCQ'){['A','B','C','D','E'].forEach(l=>setv(g('choice_'+l+'_text'),q.choices[l]||'')); pick('correct_choice',q.answer); setv(g('correct_choice_explanation'),q.explanation||''); g('shuffle_options').checked=true; multi('answer_validator_rules',[]);}
      else if(q.type==='FIB'){g('shuffle_options').checked=false; multi('answer_validator_rules',['Space insensitivity','Case insensitivity']); setv(g('correct_choice_explanation'),''); q.blanks.forEach(b=>addKV(b[0],null,b[1]));}
      else {g('shuffle_options').checked=false; multi('answer_validator_rules',[]); setv(g('correct_choice_explanation'),q.explanation||''); q.pairs.forEach(p=>addKV(p[0],p[1],p[2]));}
      if(g('topic_input').value) setv(g('topic_input'),'');
      st.stage='chk1'; put(); save(); return q.id+' save1';
    }
    if(st.stage==='chk1'){ const id=okId(); if(!id) return halt('save1 failed: '+(errs()||location.pathname));
      st.qid=id; st.res[q.id]=id; setv(g('topic_input'),topics.join(',')); st.stage='chk2'; put(); save(); return q.id+' created '+id; }
    if(st.stage==='chk2'){ if(okId()!==st.qid) return halt('save2 failed: '+(errs()||location.pathname));
      const f={}; realRows(TM).forEach(r=>{const t=r.querySelector('[name$="-topic"]'); if(t&&t.value) f[t.value]=r;});
      for(let i=0;i<topics.length;i++){ const r=f[String(topics[i])]; if(!r) return halt('no row for topic '+topics[i]);
        pickEl(r.querySelector('[name$="-question_utility_type"]'),UTIL); setv(r.querySelector('[name$="-relevance"]'),String(10-2*i));
        const mt=r.querySelector('[name$="-main_topic"]'); mt.checked=(i===0); ev(mt,'change'); }
      st.stage='chk3'; put(); save(); return q.id+' save3'; }
    if(st.stage==='chk3'){ const id=okId(); if(id!==st.qid) return halt('save3 failed: '+(errs()||location.pathname));
      const sel=n=>[...g(n).selectedOptions].map(o=>o.text.trim()).join('+'); const bad=[];
      if(sel('question_type')!==TYPE) bad.push('type'); if(sel('difficulty_level')!==q.difficulty) bad.push('diff'); if(sel('bloom_taxonomy')!==q.bloom) bad.push('bloom');
      if(sel('question_for_assessment_type')!=='Normal Assessment+Competitive Assessment') bad.push('assess');
      if(g('shuffle_options').checked!==(q.type==='MCQ')) bad.push('shuffle');
      if(sel('answer_validator_rules')!==(q.type==='FIB'?'Space insensitivity+Case insensitivity':'')) bad.push('validators');
      if(q.type==='MCQ'){ if(sel('correct_choice')!==q.answer) bad.push('answer'); ['A','B','C','D','E'].forEach(l=>{if(g('choice_'+l+'_text').value.trim()!==(q.choices[l]||'').trim()) bad.push('choice '+l);}); }
      if(g('correct_choice_explanation').value.trim()!==((q.type==='FIB'?'':q.explanation)||'').trim()) bad.push('explanation');
      const kvGot=savedRows(KV).map(r=>r.querySelector('[name$="-name"]').value+'>'+(q.type==='FIB'?r.querySelector('[name$="-correct_answer"]').value.toLowerCase():r.querySelector('[name$="-correct_answer"]').value)+(q.type==='Match'?'<'+r.querySelector('[name$="-display_text"]').value:'')).sort().join('|');
      const kvExp=(q.type==='FIB'?q.blanks.map(b=>b[0]+'>'+b[1].toLowerCase()):q.type==='Match'?q.pairs.map(p=>p[0]+'>'+p[2]+'<'+p[1]):[]).sort().join('|');
      if(kvGot!==kvExp) bad.push('key-values');
      const mp=savedRows(TM); if(mp.length!==topics.length) bad.push('topic count '+mp.length);
      mp.forEach(r=>{const t=r.querySelector('[name$="-topic"]').value; const i=topics.map(String).indexOf(t); if(i<0){bad.push('extra topic '+t); return;} if(r.querySelector('[name$="-relevance"]').value!==String(10-2*i)) bad.push('rel '+t); if(r.querySelector('[name$="-main_topic"]').checked!==(i===0)) bad.push('main '+t); if([...r.querySelector('[name$="-question_utility_type"]').selectedOptions].map(o=>o.text.trim()).join('')!==UTIL) bad.push('util '+t);});
      const strip=s=>s.replace(/<[^>]+>/g,'').replace(/&[a-z#0-9]+;/g,'x').replace(/\s+/g,'');
      if(Math.abs(strip(CKEDITOR.instances.id_question_text.getData()).length-strip(q.html).length)>3) bad.push('text');
      if(bad.length) return halt('verify: '+bad.join(', '));
      st.done.push(q.id+':'+id); st.i++; st.stage='fill'; st.qid=null; put();
      if(st.i<QS.length){window.__saving=true; location.href=BASE+'add/';}
      return 'OK '+q.id+' '+id+' ('+st.i+'/'+QS.length+')'; }
    return 'unknown stage';
  }catch(e){ return halt('script error: '+e.message); }
};
```

**Comprehensions with the runner:** use the same pattern with keys `__comps` / `__cst` and stages `fill → chk1 (fill the blank topic row) → chk2 (add MCQ rows) → chk3 (verify)`, following procedure B exactly. Each item: `{id, html, difficulty, topic, mcqs:[ids in order]}`. Click save with `#comprehension_form input[name=_continue]`.

**Finish:**
- Duplicate check: the question IDs from one run are assigned one after another with no gaps (N questions → N consecutive IDs). A gap or repeat means a duplicate to investigate.
- Remove all `__*` keys from localStorage, close the tab, and deliver the links.

## Editing an existing item
Open its change URL, change the fields with the same helpers, save, verify, then deliver the URL the same way.

## Failure handling
- Script error (missing field or option): stop and report it in one line; nothing was saved.
- Save error: read the message, fix it, and retry once. For a 500 page or an unclear result, run the duplicate guard first. If it still fails, report the exact error in one line.
- Tool timeouts: retry once; after 2–3 failures, stop and tell the user.