# Team Formation Changes — What's New

Everything added to the **Team Formation** page on the `3200-team-lock` branch, pushed to the shared repo. This file + `BRANCH_SETUP.md` (same folder) are the handoff: read both before touching this code in a new session.

## The problem this solves

Staff wanted to place certain 3200 students into teams *before* running the main algorithm on everyone else — so a returning student, or a student with special circumstances, can be locked into the right team first, and the algorithm only has to figure out everyone else around them. This grew into a full iterative workflow: generate, lock the teams you like, regenerate for the rest, repeat — plus the ability to manually fix individual students after the fact.

**Two contributors on this branch:** this session (Claude, working with the repo owner) and a teammate, Diego Valle, who independently built the minimum-3200-per-team requirement and a separate solver-lock mechanism. Those were merged in — see "The Diego merge" section below for what that actually involved and what got reconciled.

## What was built

### 1. Lock a student into a team manually

Section **"3. Lock 3200 Students"** lists every 3200 student for the selected semester/day with their ranked project choices. Pick a team from the dropdown, hit **Lock** — that student is placed immediately and skipped by the algorithm from then on. **Unlock** undoes it. Locking a student who already has an unlocked placement (e.g. from a prior Generate Teams run) correctly replaces it instead of erroring.

### 2. Let the algorithm suggest placements first

**"Suggest Placements"** runs the algorithm on just the unlocked 3200 students:
- Guarantees each student their **1st or 2nd choice** whenever possible, falling back to **3rd choice** only if capacity genuinely requires it (clearly flagged when it happens)
- Sets aside any student who **never submitted a bid**, instead of letting them block everyone else

Each suggestion pre-fills the team dropdown and explains why. Review, adjust anything that looks wrong, then lock individually or hit **"Lock All Suggested"**.

### 3. Live, honest labeling everywhere you look

- The reason text updates live as you change the dropdown.
- The dropdown is ordered by that student's own ranking, with their returning project (if any) flagged right in the list.
- Note: "returning" only activates once there's real multi-semester history in the database — Spring 2026 is currently the only semester with saved data.

### 4. Min/max 3200-per-project range

One combined control in section 3: **Min 3200 per Project** / **Max 3200 per Project**, both respected by **Suggest Placements** *and* **Generate Teams**.

- **Max** defaults to 4 — stops one popular project absorbing all the 3200s.
- **Min** defaults to **1** (every project needs at least one 3200) — this is Diego's intentional design (see below), not something this session chose. It's tight by nature; if there genuinely aren't enough 3200 students, it fails with a specific error telling you exactly how many more are needed, rather than failing silently.

### 5. Optional choice-rank display, and a 3200 badge, in the final results

A checkbox (off by default) shows each student's choice rank next to their name. A small "3200" badge is always visible on 3200 students in the results.

### 6. Lock a whole team and re-run the algorithm on the rest — iterative mode

Every team, in the always-visible team list *and* in the generated results, has a **Lock Team / Unlock Team** button. Locking a team locks every current member at once. Hit **Generate Teams** again and locked teams are left completely untouched (same database rows) while everyone else re-solves. Repeat as many rounds as needed.

### 7. Move a student to a different team after results are in

Each student in the generated results has a **Move** button. Opens a panel showing their **major** and **full ranked choices** (so you can judge whether the move actually helps them), a team dropdown sorted by their own preferences, and a **"Lock in new team"** checkbox. Left unchecked by default — the move stays fully flexible, a future Generate Teams run (or another manual move) can still touch that student freely.

### 8. Pin teams — removed (2026-10-21)

Pin was dropped entirely, at the repo owner's request: the buttons, the badges, "Lock All Pinned" and the `esa-pinned-teams` localStorage key. Setting teams aside now happens on the Workbench (see 10), which has nothing to do with pins.

**Step 1's team list is read-only (2026-10-21).** It shows each team's name, partner, member count and a "Locked" label. The Lock/Unlock Team buttons there were removed at the repo owner's request, so locking happens only in step 5 (Arrange Teams), and step 3 for individual 3200s.

### 9. One step at a time, gated on data

Only the current step is rendered. A `UStepper` bar at the top (Semester & Day → Bid File → Lock 3200s → Generate → Arrange Teams) shows where you are, and Back/Next sit at the bottom. **Gating (2026-10-21):**
- You can always go back to any step you've reached, but you can't jump ahead past the furthest one (`furthestStep`).
- Forward only goes through Next, and Next waits for the *data* that step exists for, not a button click (`nextBlocker`, shown beside Next):
  - Step 1 needs a semester.
  - Step 2 needs bids for that semester/day. Bids imported on an earlier visit count, and step 2 says so.
  - **Step 2 when bids are already saved** shows only a summary ("Bids from N students are saved for this day") plus a small **Replace bid file** button. The upload box appears only when nothing is saved yet, or after clicking Replace. Replace shows a warning describing what a re-upload really does (`bids.post.ts` is a partial update): students in the file are updated and their choices replaced, students *not* in the file are kept, and existing teams don't change until you generate again. A true replace that removes missing students was offered and not requested.
  - Step 3 is optional (zero locks is fine).
  - **Step 3 is folded by default (2026-10-21).** It shows the Min/Max 3200 per Project fields, which stay visible because they also drive Generate, plus a summary bar ("18 3200 student(s) this day · 0 locked") and a **Show 3200 students** button. The student list, Suggest Placements, Lock All Suggested and their alerts appear only after clicking it (`show3200Locking`), and "Hide 3200 students" folds them again. The folded view was checked in the browser on Fall 2026.
  - **Lock all / Unlock all (2026-10-21)** sit in the opened 3200 section, and both ask first (`useConfirm`).
    - **Lock all** locks every unlocked 3200 into the team picked in their dropdown, or else the team they're on now. Students with neither are skipped, and the dialog gives the count.
    - **Unlock all keeps students on their current team** and only clears the lock: `lock.post` with `locked: false, replaceLocked: true` on the same team. This differs from the per-student Unlock, which deletes the placement via `lock.delete`. A whole team locked in step 5 counts as unlocked afterwards, and the dialog says so.
    - "Lock All Suggested" is unchanged.
    - **Verified in the browser on Fall 2026:** Lock all locked 18 of 18; Unlock all returned them to unlocked. A before/after snapshot of all 76 Fall memberships showed 0 differences.
  - Step 4 needs teams with members (generated now or earlier).
- Changing the semester or day resets the walk to step 1.
- Nothing auto-advances.

The repo owner originally wanted "must do each step's action". That was discussed and rejected because it would force a re-upload on every visit and block semesters with no 3200 locks.

Gotcha for anyone editing this: `UStepper` treats a *numeric* `v-model` as a 0-based index, so `activeSection` (1–5) is bound through a `stepIndex` computed.

### 10. Arrange Teams — Workbench board (step 5)

**Branches:** the first version (three columns: Teams / Workspace / Locked, built on pins) is saved as `dadrean/team-board-v1` on the `myrepo` remote (`github.com/ezueb/esa`, the repo owner's personal repo), commit `23157c5`. The Workbench redesign lives on `dadrean/team-board-workbench`. **Don't push this work to the shared `origin` ESA repo.**

Step 4's old results list, with its per-student Move panel and the "show choice rank" checkbox, was removed. Arranging now happens only here. Step 4 just generates, then says to move on.

Layout:
- **All teams:** every team for the semester/day is a square in a grid (name, members/max, 3200 count, members' first names). Each square has **Set aside** and a lock button. Locked squares get a tone step plus "Locked", sort last, and can't be set aside.
- **Workbench:** sits above the grid. Teams arrive by dragging a square up or by clicking Set aside. There they sit side by side at full size, and students are dragged between them. Each Workbench card has Lock and Put back (×), and a card can also be dragged back down to the grid. Locking a team sends it back to the grid, because locked teams can't take part in moves. "Put all back" clears the Workbench.
- **The Workbench is only this page's view.** It isn't saved and resets when the semester or day changes. Only student moves and locks write to the database.

Preferences:
- Each student shows their rank for the team they're on: green for 1st/2nd, amber for 3rd, grey for lower or unranked. Colour is used only for fit, warnings and locks; the rest of the interface is grey (from the design skill's app-screen rules).
- While a student is being dragged, every other Workbench card shows that student's rank for it.
- Clicking a student expands their full ranking, flagging which choices are on the Workbench.

Preference help (added 2026-10-21):
- **"Prefers …" hints, kept quiet on purpose.** A Workbench student gets one grey line, "Prefers Alumni Tracker (1st)", with a **Move** button, *only* when another team **on the Workbench** that they ranked higher **has room** (`betterTeamFor()`; locked teams aren't on the Workbench). A first version also suggested teams off the Workbench and full teams, and added "N would rather be elsewhere" counts. On Fall 2026 that flagged 25 of 76 students, mostly pointing at full first choices, and the repo owner found it too noisy and confusing, so it was cut back to this.
- **Grid squares list every member with the rank they got** (green 1st/2nd, amber 3rd, grey lower or unranked), not just first names. The squares use `sm:min-h-56` instead of a strict aspect-square, so six names fit.

**Improve one team (branch `dadrean/team-board-improve`, 2026-10-21)**

This replaced a parked experiment: the Workbench-only "Suggest moves" panel, saved locally as `dadrean/team-board-suggestions`, commit `03acfdf`, not pushed. That one only looked at teams already set aside, so staff had to guess which teams to pull, and it allowed swaps where one student lost out.

- **Improve button.** Every team with members has one, on its grid square and on its Workbench card. It opens a panel at the top of step 5 listing every student on that team with their major, 3200 status and current rank. Each student is marked one of four ways:
  - already on their 1st choice;
  - locked, left as is;
  - a fix with a **Move** or **Swap** button;
  - "No better team they can get onto without someone else losing out."
- **The rule: nobody ever ends up worse off.** `fixFor()` walks the student's higher-ranked choices across **all** teams for the day, skipping locked teams:
  - A straight **move** is offered if that team has room, this team stays at or above `min_team_size`, and the 3200 min/max holds on both sides.
  - Otherwise it offers a **swap** with an unlocked student there who ranks this team at least as high as their current one (gain ≥ 0), preferring whoever gains most. The 3200 range is checked when a 3200 changes sides.
- **Applying** sets both teams aside onto the Workbench, then makes the move, or both moves for a swap; the second move is skipped if the first fails. `moveStudentOnBoard` now returns success. The panel recomputes live.
- **When there's no fix, the panel says why** (added after the first browser run, 2026-10-21). `fixFor()` returns `blocked`: each team the student ranked higher, with its real reason. The possible reasons are:
  - "full";
  - "this team would drop below N members";
  - "they're this team's only 3200" (or "…below N 3200s");
  - "that team already has N 3200s";
  - "that team is locked";
  - "no team for it on this day".

  Each reason except locked and no team also ends ", and nobody there would trade without losing out". A student with no bid gets "No bid submitted… Place them by hand." The old single generic line hid that Hamza was blocked by the 3200 minimum and that Sanjana had no bid. The panel also has `scroll-mt-20`, so opening it no longer hides its title under the sticky top bar.
- **First real browser run (2026-10-21), Fall 2026 Wednesday, driven in the owner's Chrome:**
  - Steps: gating, the saved-bids summary and the "teams already exist" note all behaved as designed.
  - Improve on Water Quality offered Lluvia → Modular Laser (4th → 1st). Applying it moved her, set both teams aside, and updated sizes (5/6 and 6/6).
  - Lluvia was then moved back to Water Quality (unlocked, as before), so the Fall data is unchanged.
- **Majors** (the semester's `Enrollment.major`) also show in each Workbench student's expanded panel. Projects have no major field, so major fit is left to staff.
- The quiet "Prefers X (rank) [Move]" Workbench hint is still there. It overlaps with Improve; the owner can decide whether to drop it.
- **Checked by replaying `fixFor()`** with read-only SQL on Fall 2026 Wednesday:
  - Water Quality: 4 on 1st choice; Lluvia (4th) → Modular Laser Workstation (1st), has room; Hamza (4th) has no better spot (Call Volume is full and nobody there would trade without losing out).
  - Leather Tumbler: Jaideep (3rd) has no better spot.
  - Chicken Tractor: the unranked student has no better spot.

Unchanged from v1:
- A moved student keeps their lock state (`lock.post.ts` `replaceLocked`).
- Over-max moves are allowed but flagged red.
- Uses native HTML5 drag and drop, no new library.

### Design rule: computed, not AI

The repo owner wants team-placement help to be **computed with fixed, explainable rules, not decided by AI** (stated 2026-10-21). The Improve panel follows this: deterministic rank-based rules in `fixFor()`, no external calls, the same answer for the same data. Don't add AI or LLM-based placement or recommendations without asking first. The backlog item about using free-text skill answers would raise this question, and would also mean sending real student data to an outside service.

## The Diego merge

Diego's branch added a parallel implementation of the same "minimum 3200" idea (`min_3200_per_team`, a separate UI field) plus a from-scratch locking mechanism (locks passed fresh on every solve call via `locked_student_assignments`/`locked_teams`, instead of being saved in the database). When the branches were merged:

- Diego's separate `min_3200_per_team` field and UI block did **not** survive — only this session's `min_upperclass_per_project` mechanism remains connected to anything.
- Diego's ephemeral lock-passing mechanism is still present in `team_generator.py` (dormant — nothing in this app's actual UI/API populates those config keys, since this session's DB-backed `Membership.locked` approach is what the app actually uses). It's untested-by-use but has its own regression tests (`test_locking.py`).
- **Real bug found and fixed:** the merge left the minimum-3200 default inconsistent between the Python solver, the UI's default value, and a hardcoded assumption in the shared `test.ts` regression script. Investigated thoroughly, confirmed Diego's own test suite (`test_minimum_3200.py`) proves the default-of-1 behavior was *intentional* on his part (not an accident), and reconciled everything to that: default is 1 everywhere now, consistently. Verified by running `test.ts`, `test_minimum_3200.py`, and `test_locking.py` together — all green.

**Worth raising with Diego:** the default-of-1 is tight (needs roughly one 3200 student per team) — same caveat flagged to the repo owner earlier applies to his design too.

## How to use it, step by step

1. Go to **Team Formation**, pick a **Semester** and **Meeting Day**
2. (Optional) Upload a bid CSV if this semester/day hasn't been imported yet
3. In section 3, hit **Suggest Placements** — review each suggested team + reason, adjust for any that look wrong
4. Hit **Lock All Suggested** (or lock individuals one at a time)
5. In section 4, set your min/max 3200 range, hit **Generate Teams**
6. Review the results — lock or pin whole teams you're happy with, move individual students if something looks off
7. Hit **Generate Teams** again to re-solve just the rest — repeat as needed
8. Or do steps 6–7 in **5. Arrange Teams**: set teams aside onto the Workbench, drag students between them using the rank badges, lock the teams you're done with

## What's still open / not built yet

- **Text-response skill matching** — using students' free-text experience answers to help break ties between students competing for the same spot. Not built; this is also the real fix for arbitrary tie-breaking when two students score identically for one contested spot.
- **Scale change for oversubscribed choices** — boosting the weight of lower-ranked choices when a student's top picks are already full. Applies to both 2200 and 3200 students; not built. (Same backlog item as the tie-break fix above — they're one piece of work.)
- **Fall 2026 data** — no real student-ID roster exists for that semester yet.
- **Bidless students** (never submitted a bid at all) — not addressed; needs a decision on manual-add UI vs. documentation-only approach, and whether assignment-by-major is automatic or staff-picked.
- **Red/green lock-status coloring** — a UI polish request from the repo owner's team lead, not yet built, and scope (individual-student locks only, or everywhere locking shows up) wasn't confirmed before work moved to other priorities.

Full detail on these, plus a live/editable list either of you can add to, is in the backlog tool described below.

## Bugs fixed along the way (not new features, but worth knowing)

- The solver only ever tried spawning `python`, not `python3` — fixed to try both.
- `ortools` wasn't installed on this machine — installed it.
- Lock Team / Unlock Team only appeared after clicking "Generate Teams" that page load — moved to the always-visible team list.
- The minimum-3200 default inconsistency from the Diego merge (see above).

## Files changed (for reference)

| File | What changed |
|---|---|
| `prisma/schema.prisma` + migration | Added a `locked` field to team memberships |
| `server/services/CPSAT/team_generator.py` | 3200-only suggestion solver; main solver respects locked students, min/max 3200-per-project, Diego's dormant lock-passing mechanism, specific infeasibility errors |
| `server/services/CPSAT/ortools.ts` | Wires through config/caps; python3-vs-python fix |
| `server/services/CPSAT/test_locking.py`, `test_minimum_3200.py` | Diego's regression tests (both passing) |
| `server/services/membershipService.ts` | Typed the new `locked` field |
| `server/api/team-formation/lock.post.ts` | Lock/unlock one student; now also used for the flexible "Move" feature (`locked` param, defaults true) and Team Board drags (`replaceLocked` param, defaults false) |
| `server/api/team-formation/lock.delete.ts` | Unlock one student |
| `server/api/team-formation/lock-team.post.ts` | Bulk lock/unlock a whole team (also powers "Lock All Pinned") |
| `server/api/team-formation/suggest-locks.post.ts` | Runs the 3200-only suggestion solve |
| `server/api/team-formation/generate.post.ts` | Skips locked students, preserves placements, passes through 3200 caps |
| `app/pages/team-formation.vue` | All UI: lock/suggest section, lock-team controls, move panel, pin feature, step-by-step accordion, Team Board (section 5) |
| `BRANCH_SETUP.md` | Setup steps for anyone pulling this branch fresh |

## How everything was verified

No test runner existed in this repo before Diego added unittest-based ones for his pieces. This session's verification: synthetic solver tests for each new constraint, the full-scale `test.ts` re-run after every solver change, and repeated real end-to-end runs against the actual Spring 2026 roster through the live app (signed in via the dev-mode OTP logged to console) — including creating isolated throwaway test students for the Move feature, confirming correct behavior, then fully deleting them. Real locked data from independent testing (by the repo owner) was identified and left untouched throughout.

**Team Board (2026-10-19):** tested in the running dev server against **Fall 2026 test data only**. All 76 Fall memberships were confirmed to be fake `f26wtest`/`f26ttest` IDs with no locks before anything was moved. The test covered dragging teams into Workspace, dragging a 3200 student between Workspace teams (the rank badge showed mid-drag and the move persisted), expanding a student's ranking, dropping a team into Locked (4 rows locked in the DB), and dragging it back out (0 locked). Everything was restored afterwards, and no console errors appeared. Drags were dispatched as real `DragEvent`s from JS because the browser window was too narrow for side-by-side columns, so a manual mouse drag at full width is still worth a look. `nuxi typecheck` can't run here (no `vue-tsc`). Spring 2026 was not touched.

The section 3 help text that wrongly said min-3200 "is 0 by default" was fixed on 2026-10-21 to say 1.

**Stepper (2026-10-21):** checked in the running app on Fall 2026. Only one section renders at a time, the step bar highlights correctly, jumping directly to step 3 works, Back works, Next is disabled without a semester and hidden on step 5, and step 5 still widens for the board. The only console error is a pre-existing hydration mismatch on the theme-toggle icon, which also appears on the Database page. Nothing has been committed yet.

**Workbench redesign (2026-10-21):** Chrome automation wasn't responding, so this was **not checked in a browser**. What was checked:
- The page compiles in the running dev server.
- Every template identifier resolves. This was done by compiling the SFC with the project's `@vue/compiler-sfc` and listing `_ctx.*` references (there were none).
- The design skill's `lint.mjs --app` gave 0 FAIL. Its 5 WARNs are the word "Unlock", flagged as marketing copy, which is a false positive.

Still to do: a hands-on pass (drag a square up, move students, lock and unlock, Next gating) on Fall 2026 test data.
