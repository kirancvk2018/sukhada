# Hackathon — Implementation Notes

`index.html` only. No new files. `reference-runbooks.html` untouched.

110,074 → 165,080 bytes. Existing code was extended, not replaced.

---

## 1. Summary of changes

**Modified (10 existing functions)**

| Function | Change |
|---|---|
| `Store` | +3 methods for the `h:` namespace; `removeParticipant` now also deletes `h:<key>` |
| `showScreen()` | accepts `'hackathon'` |
| `participantLogin()` | resolves mode once at login and routes accordingly |
| `loadProblem()` | `blockedByHackathon()` guard as first statement |
| `submitSolution()` | same guard |
| `enterParticipant()` | same guard |
| `doLogout()` | clears hackathon state, resets the active tray |
| `makeSlot()` | writes `data-correct` **only** when a `correctId` is passed |
| `refreshRoster()` | Hackathon column, Assign/Edit button, split detail panel, `colSpan` 7→8, two new stat chips |
| `returnCardToTray` · `refreshTrayEmptyState` · `wireTrayTarget` · `buildTray` | `getElementById('tray')` → `getTray()` |

**Added** — six labelled sections (`HACKATHON DATA`, `ACCESS CONTROL`, `SCORING`, `PARTICIPANT EXPERIENCE`, `EMPLOYEE ADMIN`, plus persistence inside `Store`), one participant screen, one config modal, ~60 CSS rules.

**Untouched** — `RAW_PROBLEMS`, `PROBLEMS`, `CARD_INDEX`, `renderAnswerKey()`, `recordAttempt()`, `refreshParticipantStats()`, `submitSolution()` grading logic, all 19 practice scenarios, all Excel alignment.

### The tray decision

Four functions hardcoded `getElementById('tray')`. Rather than fork the drag/drop layer, a single `activeTrayId` variable and `getTray()` accessor were introduced. Practice sets it to `'tray'`, Hackathon to `'hackTray'`. Everything else — `buildCardEl`, `onDragStart`, `onCardClick`, `placeCardInSlot`, `wireSlotTargets`, keyboard tap-to-place — is shared verbatim.

---

## 2. Data structures

Extends the existing model. Nothing is duplicated and no existing record shape changed.

```js
p:<key>  = { name, empId, pin, createdAt, createdBy,
             hackathon: {                       // null / absent when unassigned
               enabled, fromDate, toDate, scenarioId, assignedBy, assignedAt
             } }

a:<key>  = { items: [...] }                     // UNCHANGED

h:<key>  = { attempts: [ {                      // NEW
               ts, submittedAt, scenarioId, scenarioTitle, attemptNo, completed,
               response:   [{kind, index, placedCardId}],
               earned, maximum, percentage,
               breakdown:  [{label, section, importance, weight, result, points}],
               distractors:[{label, section, importance, weight, result, points}]
             } ] }
```

**Why the assignment sits on `p:`** — login and the roster already read that record, so the access decision costs zero extra queries. Only employees ever write it.

**Why attempts sit in `h:`** — they carry the full response and breakdown, which would bloat `p:`; keeping them out of `a:` means practice statistics are mathematically unaffected; and one row per participant preserves the collision-free write model.

---

## 3. Access control

```js
getHackathonState(rec, attempts, now)  // NOT_ASSIGNED | SCHEDULED | ACTIVE | COMPLETED | EXPIRED
isHackathonActive(rec, attempts, now)
getParticipantMode(rec, attempts, now) // 'hackathon' | 'practice'
blockedByHackathon()                   // guard used by the three entry points
```

State is derived on every call, never stored. Evaluation order matters: **an existing submission wins over everything else**, so if an employee disables an assignment after someone has submitted, the result stays visible as COMPLETED rather than collapsing to NOT_ASSIGNED. COMPLETED is not ACTIVE, so the participant still gets practice back.

**This is not a hidden dropdown.** `loadProblem()`, `submitSolution()` and `enterParticipant()` each return at their first statement when the guard fires. Calling `loadProblem('disk-util')` from the browser console pushes the user back to the Hackathon screen and builds nothing — verified in the test suite.

**Refresh is inherently safe.** The app has no session persistence; init always calls `showScreen('login')`. A refresh therefore forces a re-login, which re-evaluates mode from stored state. There is no cached session to exploit, and none was added.

### Dates

The employee picks date-only values. Stored as `2026-09-16T00:00:00` and `2026-09-20T23:59:59` — no `Z`, no offset. `new Date()` parses these in the browser's local zone, so the window means what the instructor sees. No UTC conversion happens anywhere. A same-day From/To gives a valid one-day window (tested).

---

## 4. Scoring

```js
calculateHackathonScore(scenario, response) → {earned, maximum, percentage, breakdown, distractors}
```

Pure function — no DOM, no globals, asserted by test. The browser submits **placements only**; the score is recomputed from `HACK_SCENARIOS`. A score posted from the client would be ignored.

| Section | Rule |
|---|---|
| Pre-checks | order-sensitive |
| Resolve | order-sensitive |
| Escalate | presence only, position ignored |

| Result | Meaning | Points |
|---|---|---|
| `CORRECT` | right step, right place | full weight |
| `MISPLACED` | right step, wrong position | **40%** of weight |
| `MISSING` | step never placed | 0 |
| `INCORRECT` | distractor placed in a slot | 0, listed separately |

Weight lives on each step, not derived from importance — `hack-cpu` weights a CRITICAL step at 25, `hack-disk` at 24 and 22, `hack-shutdown` at 20/18/16/15. Importance is a report label only.

Worked example, `hack-cpu` with two pre-checks transposed: weights 5 and 10 both become MISPLACED, paying 2 and 4. Score **91/100**, not 0 — partial credit throughout.

Weights total 100 per scenario, so the employee reads a clean "72 / 100".

---

## 5. Participant experience

Login → active assignment → Hackathon screen. The practice screen is never rendered.

Problem statement, then the same flow skeleton and parts tray, same drag/drop and tap-to-place, same visual language. Distractors are drawn **only** from other Hackathon scenarios — `HACK_CARD_INDEX` is separate from `CARD_INDEX`, so practice steps never appear in a Hackathon tray and Hackathon steps never appear as practice distractors.

Submission is confirmed, then one-way. Afterwards the participant sees:

> ✓ **Submission recorded**
> Your Hackathon submission has been recorded successfully. Your results will be reviewed by your training administrator.

No score, no percentage, no breakdown, no marked slots, no answer key. `submitHackathonSolution()` never calls `renderAnswerKey()`.

If the write fails, the submission is rolled back — cards unlock, the button re-enables, a toast explains. The participant is not locked out by a network blip.

---

## 6. Employee experience

One column in the existing roster showing a state badge, plus an **Assign** / **Edit** button. The button stops event propagation so it doesn't toggle the detail row.

The modal takes: enable checkbox, From date, To date, scenario dropdown (all ten). Validation rejects a missing date, a missing scenario, and From-after-To. **Remove assignment** clears it.

Expanding a participant splits into two clearly-headed blocks — **Normal practice** and **Hackathon** — so the two are never conflated. The Hackathon block renders the full step-level table: step, section, importance, weight, result, points, plus the total and a footer documenting the rules. Two new stat chips count active and completed.

This breakdown exists nowhere in the participant-facing path.

---

## 7. Testing

**97 assertions, 97 passed.** Driven headlessly through the real DOM against a simulated Supabase.

All 20 of your listed edge cases are covered. Highlights:

| | |
|---|---|
| Five states (none / scheduled / active / expired / completed) | correct, including same-day boundary |
| Console call to `loadProblem('disk-util')` during Hackathon | blocked, returns to Hackathon screen |
| `enterParticipant()` / `submitSolution()` during Hackathon | blocked |
| `data-correct` in Hackathon DOM | **absent** — slot datasets carry no answer |
| weight / importance in participant HTML | absent |
| Participant result text | no score, no percentage, no breakdown |
| Double submit | second call is a no-op, one attempt row |
| Refresh mid-Hackathon | returns to login, re-evaluates, still Hackathon |
| Refresh after submission | practice restored |
| Expired / scheduled windows | practice, as expected |
| Employee disables after submission | result still visible, participant back on practice |
| 20 participants, 10 different scenarios, simultaneous submit | 20 rows, 20 attempts, no cross-writes |
| Supabase failure during submit | not marked submitted, button re-enabled, retry succeeds |

**Regression on the existing app:** 19 practice scenarios intact, 157 practice cards with zero collisions, no duplicate DOM IDs, practice grading unchanged, answer key still works, attempts still land in `a:`, no `h:` row created by practice.

**The original 45-user suite, re-run on this build:** 45 logins created, 45 trainees from 45 separate browsers, 135 concurrent submissions with **zero loss**; burst test 180/180 at 20/80/200 ms, zero errors.

**Excel alignment, re-verified:** 19/19 tickets, 100% of 66.5 hours, 0 cross-file conflicts.

### One bug the tests caught

`getHackathonState()` originally checked the `enabled` flag before checking for attempts. Disabling an assignment after a participant had submitted made their result disappear from the dashboard. Fixed by evaluating attempts first.

---

## 8. Known limitations

**The answer key is in the page source.** `HACK_SCENARIOS` ships in the JavaScript. Anyone who opens View Source or DevTools can read every step, weight and correct order. This is inherent to a static client-side application and cannot be fixed within it.

What *is* enforced:

1. Nothing about the answer enters participant-facing HTML — no `data-correct`, no weights, no importance.
2. Expected answers live only in `hackSlots[]` in memory.
3. Hackathon and practice card pools are separate indexes.
4. The score is computed from the authoritative config; a client-supplied score would be ignored.
5. The participant path never calls `renderAnswerKey()` and never renders a breakdown.

**The database policy is permissive.** `using (true) with check (true)` plus a publishable key in a public repo means a determined participant could write directly to their own `h:` row via the REST API and fabricate a result.

**Treat this as a training exercise, not a proctored examination.** It is robust against a participant clicking around the UI. It is not robust against a participant who opens DevTools. Given the audience — consultants learning runbook discipline — that is likely the right trade. But do not use these scores for anything with consequences attached without the hardening below.

Other notes: one submission per participant (the structure supports a future `maxAttempts`); two employees editing the same participant simultaneously is last-write-wins, unchanged from existing behaviour; if an assigned `scenarioId` no longer exists the participant sees a clear "not configured" message rather than an error.

---

## OPTIONAL FUTURE SECURITY HARDENING

Not required for this implementation, and deliberately not built.

To make the Hackathon a genuine assessment, the answer key has to leave the browser:

```
Hackathon Answer Key
        ↓
Supabase Edge Function  (holds HACK_SCENARIOS server-side)
        ↓
Server-side Scoring     (receives placements, computes score)
        ↓
Participant receives only "recorded"
```

**Shape of the change**

1. Move `HACK_SCENARIOS` into an Edge Function. The client keeps only `{id, title, problem, trigger, decision, close}` and an **unordered, unlabelled** card list — enough to render the board, not enough to know the answer.
2. Add `GET /hackathon-scenario?id=` returning that reduced payload.
3. Add `POST /hackathon-submit` taking `{participantKey, scenarioId, response}`, scoring it server-side with `calculateHackathonScore` (which is already pure and portable — it moves unchanged), writing `h:<key>` with the service-role key, and returning `{recorded: true}` and nothing else.
4. Tighten RLS so the anon role cannot write `h:` rows at all; only the function's service-role key can.
5. Optionally issue a short-lived submission token at login so a participant cannot submit outside their window.

`calculateHackathonScore` was written as a pure function with this move in mind — it takes config plus response and returns a result, with no DOM or global access. Steps 1, 3 and 4 are the substantive work; the participant UI barely changes.

---

## Hackathon Scoring — Interactive Demo (added)

`index.html` only. No new files. `reference-runbooks.html` is unchanged; it is included in the package because both dashboards link to it.

**Where:**
- **Instructor:** Employee dashboard → **Hackathon Scoring — Interactive Demo** → **▶ Open Scoring Demo**. You can show any of the 10 Hackathon scenarios (default `hack-cpu`), plus the participant demo so you can preview what participants see.
- **Participant:** a **🎯 How is my Hackathon scored?** button on the practice screen and on the Hackathon screen. Opening it during an active Hackathon keeps every card the participant has already placed. **← Back** returns them to where they came from.

**Participant demo-only scenario.** Participants are locked to `SD_DEMO_SCENARIO` (*Application Server — Memory Exhaustion*, 9 steps, weights total 100). It is **not** in `HACK_SCENARIOS` or `HACK_PROBLEMS`, so it can't be assigned, isn't in the assignment dropdown, and its cards never appear in a live Hackathon tray. Its four distractor cards are also demo-only. `renderHackathonScoringDemo()` forces this scenario for any participant, even if called from the console with a real scenario ID. Tests confirm that no Hackathon step name, scenario title or card ID is ever put into the page for a participant. On that scenario, Demo B gives: Correct 20, Misplaced 8, Missing 0, Incorrect 0, and 64/100 for the realistic mixed attempt.

**No second scoring engine.** `calculateDemoScore(scenario, placements)` turns the demo's placements into the same `{kind, index, placedCardId}` response that `submitHackathonSolution()` builds, then calls `calculateHackathonScore()`. Every point, badge, count and total on screen comes from that call. The "× 40%" labels read `ORDER_PARTIAL_CREDIT`. The demo only works out the multiplier to *write out* the calculation, and it logs a console warning if that ever disagrees with the engine. Scenario data comes from `HACK_PROBLEMS`; nothing is copied.

| Part | What it shows |
|---|---|
| Header | Title, message, ▶ PLAY DEMO / ↻ REPLAY / RESET, the four result rules, scenario + presenter-pace pickers (default `hack-cpu`) |
| Demo A — Relevance sets the weight | Each expected step: highlight → relevance (its own `desc` plus its rank by weight) → card moves from *Available Process Steps* into its slot → `w × 100% = w` → running score. Weight bars compare every step (e.g. 25 vs 2). Three distractors stay in the tray and end up marked "would score 0". Also has Pause/Resume and Next step ▸ |
| "Why is this step worth N points?" | Click any card or bar. Shows the step's importance, weight, share of the total, rank, where it belongs, and how the same importance maps to different weights across scenarios |
| Demo B — Placement | The same step in four cases, each starting from an empty workflow: **Correct** (20 × 100% = 20), **Misplaced** (20 × 40% = 8), **Missing** (20 × 0% = 0), **Incorrect/Distractor** (0, and the displaced step becomes Missing). Two more cases: the escalate steps swapped (still full credit, because only presence is graded) and a realistic mixed attempt (66/100 on `hack-cpu`). Live breakdown table, score and Correct/Misplaced/Missing/Incorrect counts |
| How Hackathon Scoring Works | The 8-point model, the goal statement, and a comparison of order-sensitive vs presence-only sections |

**Accessibility:** results never rely on colour alone. Each has an icon, a text label and a border style (Missing is dashed). Cards and bars can be reached and opened from the keyboard. The step panels use `aria-live`. `prefers-reduced-motion` turns off the moving cards and the count-up.

**Tested:** 105 automated checks in headless Chromium against a mocked Supabase, with no console errors:
- Demo A: play, pause, next step, replay and reset, ending at 100/100.
- Demo B: all six cases.
- All 10 scenarios play through both demos.
- 2,000 random placements give identical results from `calculateDemoScore` and `calculateHackathonScore`.
- No horizontal scroll at 1440, 1024 or 390 px.
- Regression checks: employee login, participant creation, live Hackathon submission (swapping the first two pre-checks scores 91/100, as in §4), practice submission, and roster status. No `data-correct` attribute appears on the participant's Hackathon screen.
- Participant access from practice and from the Hackathon screen: always locked to the demo-only scenario, nothing from a real Hackathon scenario on the page, and board state kept on return.

**Existing issue noticed, not changed:** "Acknowledge & Classify" is a step in 8 of the 10 Hackathon scenarios, so a live Hackathon tray can show two identical-looking cards. If the participant picks the one from another scenario, it scores INCORRECT and the real step scores MISSING. The demo leaves out distractors whose label matches a real step. The live tray does not.

---

## Reference Runbooks login gate (added)

`reference-runbooks.html` now opens only in a browser that is currently logged in to `index.html`. Anyone else who types the URL is sent to the login page with the message *"Please log in to view the Reference Runbooks."*

| Who | Can open the runbooks? |
|---|---|
| Employee, logged in | Yes |
| Participant in practice mode, logged in | Yes |
| Participant whose Hackathon is active | **No.** The Hackathon screen never linked to the runbooks, so it stays closed-book |
| After Log out, or after reloading the app (a reload already logs you out) | No |
| After `RUNBOOK_GATE_HOURS` (default 8 hours) | No |
| Scripts turned off | No. The page redirects to the login page |

**How it works:** a successful login writes a short-lived pass (`runbook-academy-gate`) to the browser's `localStorage`. A small script at the top of the reference page keeps the page hidden until it has checked that pass. The page is also marked `noindex` so search engines don't list it.

**What this gate does NOT protect:** GitHub Pages is static hosting and can't check a login before it sends a file. This gate stops people who follow a link, bookmark the page, or share the URL. It does not stop someone who knows how to use browser developer tools. It also doesn't stop anyone reading the file directly in the **public GitHub repository**. For real protection, the file must be served by something that checks a login first:

1. **Make the repository private.** This stops people reading the source on github.com. Note that GitHub Pages from a private repo needs a paid GitHub plan, and even then the published site is still public unless you're on GitHub Enterprise Cloud (which offers "private" Pages).
2. **Or move hosting to Cloudflare Pages and put Cloudflare Access in front of it.** The free tier covers up to 50 users, with email one-time codes, and the check happens before the file is sent. Keep the same two files; participants would verify their email once before reaching the app.

**Tested:** 11 checks over a local web server: a fresh visitor, employee, practice participant, active-Hackathon participant, logout, app reload, an expired pass, a corrupted pass, and scripts turned off. The full 105-check suite still passes.

---

## Hackathon score downloads (added)

The Employee dashboard's **Participant Roster & Scores** panel has two buttons:

- **Summary (Excel CSV)**: one row per participant. Columns: Rank, Participant, Employee ID, Hackathon status, Scenario, Window from and to, Submitted at, Score, Maximum, Percentage, then counts of Correct, Misplaced, Missing and Incorrect, and the number of attempts.
  - Participants who submitted come first, sorted by score. Tied scores share a rank.
  - Everyone else follows with their status (Not assigned, Scheduled, Active (not submitted), or Expired (not submitted)).
- **Detail (Excel CSV)**: one row per step for every submission, plus one row for each distractor placed. Columns: Expected position, Placed at, Step, Importance, Weight, Result, Points, and the calculation (e.g. `20 x 40% = 8`).
  - A participant's Points add up exactly to their total score.

Both downloads re-read the data from storage when you click, and use the scores saved at submission (the same numbers the roster shows).

Files are named `Hackathon_Summary_YYYY-MM-DD_HHMM.csv` and `Hackathon_Detail_…`. They are saved as UTF-8 with a BOM, so Excel opens them with the correct characters. PINs are never exported. A name that starts with `=`, `+`, `-` or `@` gets a leading `'` so Excel can't run it as a formula.

**Tested:** 18 checks: file names, columns, ranks and ties, participants who didn't submit, names containing commas or formula characters, no PINs, Points adding up to each total, expected vs placed positions, distractor rows, and escalate labels. The full 105-check suite still passes.

---

## Hackathon scenarios: 10 → 20 (added)

The practice **Problem State** list covers all 19 tracker tickets, but only 9 of them had a Hackathon scenario. The ten missing tickets now have one each. They were added after the original ten, so existing scenario IDs, card IDs, weights, assignments and saved results are unchanged.

| New scenario id | Title | Tracker ticket |
|---|---|---|
| `hack-pwd` | User Account — Domain Password Expired | INC000103 |
| `hack-user` | New Joiner — User Account Creation | INC000104 |
| `hack-policy` | Server Group — Security Policy Deployment | INC000107 |
| `hack-build` | New Production Server — Build to Baseline | INC000109 |
| `hack-decomm` | Retired Server — Decommissioning | INC000110 |
| `hack-diskinc` | Production Volume — Planned Capacity Increase | INC000111 |
| `hack-restore` | Production Server — Restore After File Corruption | INC000112 |
| `hack-alert` | Monitoring — Auto-Recovered Service Warning | INC000115 |
| `hack-dc` | Data Centre — Rack Temperature Rising | INC000116 |
| `hack-ups` | Network Rack — UPS Battery Critical | INC000117 |

Each new scenario follows the existing format: a problem statement, trigger, decision, 3 order-sensitive pre-checks, 4 order-sensitive resolve steps, and a presence-only escalate branch, with weights that add up to 100. The content is based on the ticket's practice runbook and the Excel resolution notes. No new step name repeats a name used elsewhere in the Hackathon, so the new cards can't look like duplicates in a participant's tray.

The new scenarios appear automatically in the employee assignment dropdown, the instructor Scoring Demo and the score downloads. They also join the pool of distractor cards shown in other scenarios' trays, which is how the original ten already worked. The participant demo scenario is still not assignable.

**Tested:**
- The original 10 scenarios are identical to before (IDs, card IDs and weights).
- The assignment dropdown lists all 20 scenarios.
- A participant assigned *Password Expired* gets that scenario, with no look-alike cards in the tray.
- Scoring uses the real engine: swapping the first two resolve steps gives 82/100.
- All 20 scenarios score 100 when every step is correct.
- The practice list still has 19 problems.
- The full suites pass: 125 app/demo checks, 18 download checks and 11 runbooks-login checks.


---

## Employee login: named user IDs (changed)

The single shared access code (`EMPLOYEE_CODE`) and its on-screen "Default code …" hint have been removed. The Employee tab now asks for a **User ID** and **Password**:

| User ID | Password |
|---|---|
| Admin | Admin@123 |
| Subbiah | Admin@123 |

- The user ID is not case-sensitive; the password is.
- The dashboard shows who is logged in (Admin or Subbiah).
- Accounts live in `EMPLOYEE_ACCOUNTS` near the top of the script. Each password is stored as a SHA-256 hash of `runbook-academy:<userid in lowercase>:<password>`, never in plain text. The comment above `EMPLOYEE_ACCOUNTS` explains how to add a user or change a password.
- **Limitation:** the check still runs in the browser, like the rest of this static site. A hash keeps the password out of plain sight in the public repo, but a short password can be guessed by anyone who copies the hash and runs a password-guessing tool on it. Use longer passwords for anything beyond a training exercise.

**Tested:** 16 login checks: both accounts, case rules, the old code rejected, unknown or blank input rejected, the Enter key, the password not in the page source, and the built-in and fallback hashing giving identical results. All existing suites pass with the new login.
