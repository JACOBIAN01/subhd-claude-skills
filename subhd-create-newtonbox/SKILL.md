---
name: "subhd-create-newtonbox"
description: "Create a Newton Box coding question (terminal / Node CLI by default) on Newton School's Django admin via the user's Chrome: statement, starter, solution, mocha tests, playgrounds, verify and topics; reply with the admin URL."
---

# subhd-create-newtonbox

Builds a complete **Newton Box** question: the question record, three playgrounds (Boilerplate = student starter, Solution, Test Case), tests that **fail on the starter and pass on the solution**, verification and topic mappings. About 90% of requests are **terminal (Node.js CLI) questions**, in the style of 28689–28693 and 28616.

URLs:
- Question add: `https://django.newtonschool.co/admin/assignments/assignmentquestion/add/`
- Question change: `https://django.newtonschool.co/admin/assignments/assignmentquestion/<id>/change/`
- Playground mapping: `https://django.newtonschool.co/admin/assignments/assignmentquestionprojectplaygroundmapping/<mappingId>/change/`
- Playground: `https://my.newtonschool.co/playground/newton-box/<hash>`
- Code-server for a playground: `https://<hash>.edison-codeserver.newtonschool.co/?folder=/app/code`
- Drone sync: `https://drone-gitlab.newtonschool.co/`

Fixed IDs:
- Template (project playground boilerplate) **34 = Node 20 Assignment Question (Code server)**, for terminal/Node CLI questions. Other templates only if the user says (e.g. 45 = Node 20 + Prisma/Postgres).
- Test runner (project playground test boilerplate) = **Node 20 Chai Mocha**.

Final reply = **only the question change URL**. The only other messages allowed: the topic question (if no topic list is available), and a one-line blocker report.

## 0. Before starting
- **Topic list:** use the project doc `AD Topic Tree` if it is attached (Name → ID). Otherwise ask the user for the topic list. Never guess IDs.
- **Utility type:** Application unless the user says otherwise.
- **Difficulty / Bloom:** use them if given; otherwise judge them (most terminal questions are Easy).
- **Browser:** load the chrome-browser skill. If several browsers are connected, ask which one; it must be signed in to django.newtonschool.co and my.newtonschool.co. A redirect to `/admin/login/` means wrong browser or signed out: ask, and never type passwords.

## 1. Write everything locally first (workspace)
Create `starter/src/index.js`, `starter/README.md`, `solution/src/index.js`, `tests/__tests__/tests.js` and the question HTML.

**Run the tests locally** (`npm i mocha@8 chai@4 mochawesome@6.1.1`, then `PROGRAM_PATH=<file> npx mocha '__tests__/*.js' --timeout 60000 --exit`):
- the solution must pass **all**, 5 runs in a row (no flakiness);
- the starter must fail **all**.

Fix until both hold. Keep md5 checksums of every file for later verification.

### Question text rules (user's standing rules)
- Clean, proper **HTML**, LeetCode-style, easy for every student to read. **No title and no difficulty** inside the text. **No "Run Your Code" section.**
- **Guide the student** (what to build, exact expected behaviour, examples, constraints) but **never give the answer or solution code** in the statement.
- Structure:
  1. intro `<p>` with key terms in `<strong>`;
  2. `<h3>Screen Layout</h3>` with the exact screen in `<pre><code class="language-bash">` plus a short `<ul>` of rules;
  3. `<h3>Requirements</h3>` as an `<ol>`;
  4. `<h3>Example 1..3</h3>` each with **Input** / **Output** (code block) / optional **Explanation**;
  5. `<h3>Constraints</h3>` as a `<ul>`;
  6. `<h3>Project Files</h3>` as a `<ul>`.
- Exact strings and identifiers in `<code>`; escape `<` and `>` as `&lt;` and `&gt;`. Code snippets clean and concise.

### Starter (`src/index.js`) conventions
- Provide the setup: constants, data, key byte arrays (`UP=[27,91,65]`, `DOWN=[27,91,66]`, `ENTER=[13]`, `CTRL_C=[3]`), `clearScreen="\x1b[2J\x1b[H"`, `exitProgram()`, a `matches(data,key)` helper, and the block:
  ```js
  // DO-NOT EDIT THIS CHECK
  if (process.stdin.isTTY) { process.stdin.setRawMode(true); }
  process.stdin.resume();
  ```
- Leave `draw()` and `process.stdin.on('data', ...)` with **numbered `// TODO:` comments** that describe the steps, never the code. End with `draw();`.
- `README.md` = a **syntax reference** (raw mode, reading keys, byte table, ANSI clear/redraw, any helpers like `repeat`, `Math.min/max`, `process.exit`), not the solution.

### Solution
The same file as the starter, with only the TODO parts filled in. Keep every name, constant and structure identical.

### Tests (`__tests__/tests.js`, chai + mocha)
The solution sits next to `__tests__`, so resolve `../src/index.js` with a fallback one level higher:
```js
const PROGRAM_PATH = process.env.PROGRAM_PATH ||
  (fs.existsSync(path.resolve(__dirname, '..', 'src', 'index.js'))
    ? path.resolve(__dirname, '..', 'src', 'index.js')
    : path.resolve(__dirname, '..', '..', 'src', 'index.js'));
```
- `runProgram(keys)`: `spawn(process.execPath,[PROGRAM_PATH],{stdio:['pipe','pipe','pipe']})`. Collect stdout/stderr. Use a `MAX_RUNTIME_MS=3000` watchdog that `SIGKILL`s and sets `timedOut`. Send keys `'\u001b[A'`, `'\u001b[B'`, `'\r'`, `'\u0003'` with `KEY_DELAY_MS=60`, the first one after a 60 ms delay. Resolve `{stdout, stderr, code, timedOut}`.
- `describe(<title>)` with `this.timeout(60000)`. One `it()` per requirement: initial screen, each key's effect, lower and upper clamps, Enter output + `code 0`, Ctrl+C output without confirmation. Use `expect(timedOut).to.equal(false)` and `to.include` / `to.not.include`. 6–9 tests.

## 2. Create the question (Django)
Form `#assignmentquestion_form` (scope every selector to it; the page also has `#previewquestionform`). Select options by visible text. Save with `input[name=_continue]`.

**Save 1 (add page):**
- `question_title`;
- `question_text` via `CKEDITOR.instances.id_question_text.setData(html,{callback})` then `updateElement()`;
- `question_type` = **Newton Box**;
- `max_points` 10;
- `question_for_assignment_type` = Normal Assignment + Competitive Assignment;
- `bloom_taxonomy` given or judged;
- `is_codelens_enabled` on, `feedback_evaluable` off, `peer_reviewed` off, `test_case_group_enabled` off;
- editorial and `topic_input` empty.

**Save 2:** in `#assignment_question_project_playground_mappings-group` set the row's `-project_playground_boilerplate` = **34**, then save. This creates **only the Boilerplate** playground. Open the row's **admin url** (the mapping page).

## 3. Mapping page steps
1. Set `project_playground_test_boilerplate` = **Node 20 Chai Mocha**, then Save and continue. This creates the **Test Case** playground.
2. Write and save the **Boilerplate** playground (section 4).
3. Back on the mapping page: Save and continue, then click **Clone And Link Solution from Boilerplate** (`_clone-and-link-solution-playground-from-boilerplate`). Message: "Created and linked required playground". The **Solution** is a copy of the boilerplate.
4. Write and save the **Solution** (section 4), then write and save the **Test Case** (section 5).
5. Mapping page: Save and continue. Click **Run test cases against boilerplate** (`_run-test-cases-against-boilerplate-project-playground`) and wait: reload or fetch the page every ~30 s until "Boilerplate test case json" shows the run. Every status must be **failed**; "All test case verified against boilerplate" shows **True** (a green tick), which is correct.
6. Click **Run test cases against solution** and wait the same way. Every status must be **passed**; "All test case passing against solution" = **True**. Save and continue.
7. Drone sync: open `https://drone-gitlab.newtonschool.co/`, click **SYNC**, and reload every 30 s until the button shows **SYNC** again. It can take 5+ minutes; the user may say it's done.
8. Mapping page: run against boilerplate again and confirm all failed, then Save and continue.
9. Click the **Assignment question:** link, click **Verify Project Question** (`_verify-project-question`), and expect "Problem verified" and Verified = True.

## 4. Writing Boilerplate / Solution files (code-server)
These playgrounds embed **code-server** from another domain, so the parent page can't script it.
- **Open the playground page first and keep that tab open.** Never navigate the playground tab itself to code-server: that breaks the session ("This session was disrupted"). If that pop-up appears, click **Restart this session**; the files on disk survive.
- Open the code-server URL in a **second tab**. That tab must be **in front and visible**, otherwise the terminal and editor don't render or run anything.
- Open a terminal: `cmd+shift+p` (macOS), type `Terminal: Create New Terminal`, Enter. Focus the last `.xterm-helper-textarea` via JS, then use the computer `type` action:
  `cd /app/code && echo <base64> | base64 -d > src/index.js && echo <base64> | base64 -d > README.md && touch src/ok_$(md5sum src/index.js | cut -c1-8)_$(md5sum README.md | cut -c1-8)`
  then press Return.
- Verify by reading the Explorer rows (`.explorer-folders-view .monaco-list-row` aria-labels): the `ok_<md5>_<md5>` marker must match the local checksums. Then `rm -f src/ok_*`.
- Close that tab: type `exit` in the terminal. If Chrome still asks to confirm leaving, run `window.addEventListener('beforeunload',e=>{e.stopImmediatePropagation()},{capture:true})` in that tab, then close it.
- Back on the playground tab, click **Save Project** and expect "files saved successfully / Playground Saved". If Save is greyed out, the page didn't detect the change: do one real keystroke in the embedded editor, or ask the user to. Don't continue to the test runs until the Boilerplate is saved: unsaved, students get the template's Express code.

## 5. Writing the Test Case playground (in-page editor)
The Test Case playground uses an editor built into the page (`window.monaco`).
- Click `tests.js` in the file tree.
- Set the content: `m=monaco.editor.getModels().find(x=>x.uri.path==='/__tests__/tests.js'); m.pushEditOperations([], [{range:m.getFullModelRange(), text:src}], ()=>null)`. Pass the source as base64 and `atob` it (the file must be ASCII).
- Verify the length and a hash against the local file.
- **Save Project only turns on after a real keystroke:** focus the editor, then use computer keys `End space BackSpace`. Click Save and expect "Playground Saved".
- Leave `.drone.yml`, `parser.js`, `package.json` and `index.js` untouched.

## 6. Topics
On the question page:
1. Set `topic_input` to the **2 best-matching topic IDs** (comma-separated), ranked, then Save and continue.
2. Rows appear in `#assignment_question_topic_mappings-group`. On saved rows the topic is **text**, so match each row by name.
3. Per row: `-question_difficulty_type` = the question's difficulty (this drives the read-only "Difficulty type"); `-question_utility_type` = Application (or as given); `-relevance` 10 then 8; `-main_topic` and `-mandatory` checked **only** on the first row. Save and continue.
4. Re-read the rows and check that Difficulty type matches.

## Browser rules
- Use `javascript_tool` for reading and clicking; screenshots usually time out on these pages.
- Sanitize returned strings (the extension blocks `= & ? % ;` and base64-looking output): `.replace(/[=&?%;]/g,' ')`. Never return csrf or token values. Don't call the my.newtonschool.co API directly (it needs auth headers).
- Links with `target=_blank`: remove the attribute and click, so they open in the same tab group.
- A JS call is cut off at ~45 s, so wait at most ~25–30 s per call when polling.
- Close tabs you opened before finishing.

## Reply
Only the question change URL. If something blocks you (Save Project greyed out, sync stuck, login), report it in one or two lines with what you need from the user.