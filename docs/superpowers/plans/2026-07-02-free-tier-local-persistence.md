# Free Tier Local Persistence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Persist per-level best score and last-session stats in the browser via `localStorage`, and surface them on the start screen's level cards and the round-complete screen — with zero accounts, logins, or network calls.

**Architecture:** Everything lives inside the existing single `<script>` block in `flag-challenge.html`, matching the file's established single-file, no-build-step pattern (confirmed with the user — no new JS files). A small persistence layer (`defaultStats`/`loadStats`/`saveStats`) reads/writes one JSON blob under the `flagChallengeStats` `localStorage` key. `finishLevel()` updates and saves stats; two small rendering functions push the saved values into the level cards and the complete screen.

**Tech Stack:** Plain HTML/CSS/JS, `localStorage`. No frameworks, no build step, no test runner — verification is done via the `mcp__Claude_Preview__*` browser tools (a static file server + real DOM/localStorage inspection), per the approved spec's manual-verification plan.

**Spec:** `docs/superpowers/specs/2026-07-02-free-tier-local-persistence-design.md`

---

## Task 0: Set up the local preview server

**Files:**
- Create: `.claude/launch.json`

- [ ] **Step 1: Create the launch config for a static file server**

`flag-challenge.html` has no build step, so a plain static server is enough to preview it. Create `.claude/launch.json`:

```json
{
  "version": "0.0.1",
  "configurations": [
    {
      "name": "flag-challenge-static",
      "runtimeExecutable": "python3",
      "runtimeArgs": ["-m", "http.server", "8420"],
      "port": 8420
    }
  ]
}
```

- [ ] **Step 2: Start the server and confirm the page loads**

Call `mcp__Claude_Preview__preview_start` with `name: "flag-challenge-static"`. Then call `mcp__Claude_Preview__preview_eval` with `expression: "window.location.href = 'http://localhost:8420/flag-challenge.html'; document.title"` (or navigate via the tool's normal mechanism) and confirm the page loads.

Expected: `preview_snapshot` shows the "The Flag Challenge" start screen with three level cards (Easy, Medium, Hard).

- [ ] **Step 3: Commit**

```bash
git add .claude/launch.json
git commit -m "chore: add static server config for local preview"
```

---

## Task 1: Persistence data layer + save-on-finish

**Files:**
- Modify: `flag-challenge.html` (inside the `<script>` block)

- [ ] **Step 1: Add the persistence functions**

Find this block (the end of the "Scoring" section, right before "Game state"):

```js
  /* ---------------- Scoring ---------------- */
  function computeScore(elapsedSec){
    const e = Math.floor(Math.max(0, elapsedSec));
    if(e <= 0) return 20;
    if(e < 5) return 20 - 2 * e;
    if(e === 5) return 10;
    if(e < 10) return 10 - (e - 5);
    return 5;
  }

  /* ---------------- Game state ---------------- */
```

Insert a new section between them:

```js
  /* ---------------- Scoring ---------------- */
  function computeScore(elapsedSec){
    const e = Math.floor(Math.max(0, elapsedSec));
    if(e <= 0) return 20;
    if(e < 5) return 20 - 2 * e;
    if(e === 5) return 10;
    if(e < 10) return 10 - (e - 5);
    return 5;
  }

  /* ---------------- Local stats persistence ---------------- */
  const STATS_KEY = "flagChallengeStats";

  function defaultStats(){
    return {
      easy:   { bestScore: 0, lastSession: null },
      medium: { bestScore: 0, lastSession: null },
      hard:   { bestScore: 0, lastSession: null }
    };
  }

  function loadStats(){
    try{
      const raw = localStorage.getItem(STATS_KEY);
      if(!raw) return defaultStats();
      const parsed = JSON.parse(raw);
      const stats = defaultStats();
      for(const key of Object.keys(stats)){
        if(parsed && parsed[key]){
          if(typeof parsed[key].bestScore === "number") stats[key].bestScore = parsed[key].bestScore;
          if(parsed[key].lastSession) stats[key].lastSession = parsed[key].lastSession;
        }
      }
      return stats;
    } catch(e){
      return defaultStats();
    }
  }

  function saveStats(stats){
    try{
      localStorage.setItem(STATS_KEY, JSON.stringify(stats));
    } catch(e){
      // localStorage unavailable (private browsing, quota, etc.) — game
      // continues with in-memory-only stats for this session.
    }
  }

  /* ---------------- Game state ---------------- */
```

- [ ] **Step 2: Initialize `stats` alongside `state`**

Find:

```js
  const state = {
    level: null,        // {key, label, total}
    pool: [],           // shuffled countries for this run
    idx: 0,             // current question index (0-based)
    answered: 0,        // questions answered so far
    correctCount: 0,
    score: 0,
    streak: 0,
    bestStreak: 0,
    prevScore: 0,        // "ps" for bonus formula
    questionStart: 0,
    timerHandle: null,
    current: null,       // {code, name, options:[...]}
    locked: false
  };
```

Replace with:

```js
  const state = {
    level: null,        // {key, label, total}
    pool: [],           // shuffled countries for this run
    idx: 0,             // current question index (0-based)
    answered: 0,        // questions answered so far
    correctCount: 0,
    score: 0,
    streak: 0,
    bestStreak: 0,
    prevScore: 0,        // "ps" for bonus formula
    questionStart: 0,
    timerHandle: null,
    current: null,       // {code, name, options:[...]}
    locked: false
  };

  let stats = loadStats();
```

- [ ] **Step 3: Update and save stats when a level finishes**

Find:

```js
  function finishLevel(){
    clearInterval(state.timerHandle);
    document.getElementById("complete-level-name").textContent = state.level.label + " · Complete";
    document.getElementById("final-score").textContent = state.score;
    const acc = state.level.total > 0 ? Math.round((state.correctCount / state.level.total) * 100) : 0;
    document.getElementById("final-accuracy").textContent = acc + "%";
    document.getElementById("final-streak").textContent = state.bestStreak;
    showScreen("complete");
  }
```

Replace with:

```js
  function finishLevel(){
    clearInterval(state.timerHandle);
    const levelKey = state.level.key;
    const acc = state.level.total > 0 ? Math.round((state.correctCount / state.level.total) * 100) : 0;

    if(state.score > stats[levelKey].bestScore){
      stats[levelKey].bestScore = state.score;
    }
    stats[levelKey].lastSession = {
      score: state.score,
      accuracy: acc,
      streak: state.bestStreak,
      date: new Date().toISOString()
    };
    saveStats(stats);

    document.getElementById("complete-level-name").textContent = state.level.label + " · Complete";
    document.getElementById("final-score").textContent = state.score;
    document.getElementById("final-accuracy").textContent = acc + "%";
    document.getElementById("final-streak").textContent = state.bestStreak;
    showScreen("complete");
  }
```

(The `best-indicator` UI text is added in Task 3 — this step only makes `finishLevel` persist data, no visible change yet.)

- [ ] **Step 4: Verify with the browser tool**

With the server running (Task 0), reload the page and clear any prior stats:

`mcp__Claude_Preview__preview_eval`: `expression: "localStorage.clear(); location.reload(); 'reloaded'"`

Then, after the reload completes (confirm with `preview_snapshot`), play a full "Easy" round (20 questions) by clicking the level card and then any answer button 20 times, waiting for each question transition:

`mcp__Claude_Preview__preview_eval`:
```js
(async () => {
  document.querySelector('.level-card[data-level="easy"]').click();
  for(let i = 0; i < 20; i++){
    await new Promise(r => setTimeout(r, 300));
    document.querySelectorAll('.answer-btn')[0].click();
    await new Promise(r => setTimeout(r, 900));
  }
  await new Promise(r => setTimeout(r, 300));
  return localStorage.getItem('flagChallengeStats');
})()
```

Expected: the returned string is valid JSON with `easy.bestScore` matching a positive number and `easy.lastSession` populated with `score`, `accuracy`, `streak`, and a `date` string. `medium` and `hard` remain at `bestScore: 0, lastSession: null`.

- [ ] **Step 5: Commit**

```bash
git add flag-challenge.html
git commit -m "feat: persist per-level best score and last session to localStorage"
```

---

## Task 2: Level card "Best" display

**Files:**
- Modify: `flag-challenge.html` (markup, CSS, and script)

- [ ] **Step 1: Add a place for the best-score line in each level card's markup**

Find:

```html
      <div class="level-card" data-level="easy" data-total="20">
        <div>
          <p class="level-name">Easy</p>
          <p class="level-meta">20 flags · gentle start</p>
        </div>
        <div class="level-stamp">I</div>
      </div>
      <div class="level-card" data-level="medium" data-total="50">
        <div>
          <p class="level-name">Medium</p>
          <p class="level-meta">50 flags · know your atlas</p>
        </div>
        <div class="level-stamp">II</div>
      </div>
      <div class="level-card" data-level="hard" data-total="150">
        <div>
          <p class="level-name">Hard</p>
          <p class="level-meta">150 flags · true vexillologist</p>
        </div>
        <div class="level-stamp">III</div>
      </div>
```

Replace with:

```html
      <div class="level-card" data-level="easy" data-total="20">
        <div>
          <p class="level-name">Easy</p>
          <p class="level-meta">20 flags · gentle start</p>
          <p class="level-best"></p>
        </div>
        <div class="level-stamp">I</div>
      </div>
      <div class="level-card" data-level="medium" data-total="50">
        <div>
          <p class="level-name">Medium</p>
          <p class="level-meta">50 flags · know your atlas</p>
          <p class="level-best"></p>
        </div>
        <div class="level-stamp">II</div>
      </div>
      <div class="level-card" data-level="hard" data-total="150">
        <div>
          <p class="level-name">Hard</p>
          <p class="level-meta">150 flags · true vexillologist</p>
          <p class="level-best"></p>
        </div>
        <div class="level-stamp">III</div>
      </div>
```

- [ ] **Step 2: Style the new line**

Find:

```css
  .level-meta{ font-size:12.5px; color:var(--muted); }
```

Replace with:

```css
  .level-meta{ font-size:12.5px; color:var(--muted); }
  .level-best{ font-size:12px; color:var(--gold); margin:4px 0 0; }
```

- [ ] **Step 3: Add a render function and call it on load + whenever the start screen is shown**

Find:

```js
  function showScreen(name){
    Object.values(screens).forEach(s => s.classList.remove("active"));
    screens[name].classList.add("active");
  }
```

Replace with:

```js
  function showScreen(name){
    Object.values(screens).forEach(s => s.classList.remove("active"));
    screens[name].classList.add("active");
  }

  function renderLevelCards(){
    document.querySelectorAll(".level-card").forEach(card => {
      const key = card.dataset.level;
      const best = stats[key].bestScore;
      card.querySelector(".level-best").textContent = best > 0 ? ("Best: " + best.toLocaleString() + " pts") : "";
    });
  }
```

- [ ] **Step 4: Call `renderLevelCards()` on initial load and when returning to the start screen**

Find:

```js
  document.getElementById("btn-replay").addEventListener("click", () => startLevel(state.level.key));
  document.getElementById("btn-choose").addEventListener("click", () => showScreen("start"));
```

Replace with:

```js
  document.getElementById("btn-replay").addEventListener("click", () => startLevel(state.level.key));
  document.getElementById("btn-choose").addEventListener("click", () => {
    renderLevelCards();
    showScreen("start");
  });
```

Find the end of the IIFE:

```js
  document.getElementById("btn-restart-game").addEventListener("click", () => {
    if(confirm("Restart this round from the beginning? Current progress will be lost.")){
      startLevel(state.level.key);
    }
  });

})();
```

Replace with:

```js
  document.getElementById("btn-restart-game").addEventListener("click", () => {
    if(confirm("Restart this round from the beginning? Current progress will be lost.")){
      startLevel(state.level.key);
    }
  });

  renderLevelCards();

})();
```

- [ ] **Step 5: Verify with the browser tool**

Reload the page (`localStorage` still has the `easy` stats from Task 1's verification):

`mcp__Claude_Preview__preview_eval`: `expression: "location.reload(); 'reloaded'"`

Then check the rendered card text:

`mcp__Claude_Preview__preview_eval`:
```js
document.querySelector('.level-card[data-level="easy"] .level-best').textContent
```

Expected: a non-empty string like `"Best: 340 pts"` (exact number depends on Task 1's playthrough). Also check:

```js
document.querySelector('.level-card[data-level="medium"] .level-best').textContent
```

Expected: `""` (empty — medium has never been played).

- [ ] **Step 6: Commit**

```bash
git add flag-challenge.html
git commit -m "feat: show best score on level cards"
```

---

## Task 3: Complete screen "New Best!" indicator

**Files:**
- Modify: `flag-challenge.html` (markup, CSS, and script)

- [ ] **Step 1: Add the indicator element to the complete screen markup**

Find:

```html
      <h2 class="complete-title">Round Sealed</h2>
      <div class="final-score" id="final-score">0</div>
      <div class="final-score-label">Total Points</div>
```

Replace with:

```html
      <h2 class="complete-title">Round Sealed</h2>
      <div class="final-score" id="final-score">0</div>
      <div class="final-score-label">Total Points</div>
      <p class="best-indicator" id="best-indicator"></p>
```

- [ ] **Step 2: Style the indicator and tighten the spacing above it**

Find:

```css
  .final-score-label{ font-size:11px; letter-spacing:2px; text-transform:uppercase; color:var(--muted); margin-bottom:26px; }
```

Replace with:

```css
  .final-score-label{ font-size:11px; letter-spacing:2px; text-transform:uppercase; color:var(--muted); margin-bottom:6px; }
  .best-indicator{ font-size:12px; letter-spacing:1px; text-transform:uppercase; color:var(--gold-bright); margin:0 0 26px; }
```

- [ ] **Step 3: Populate the indicator in `finishLevel()`**

Find (this is the block Task 1 introduced):

```js
    if(state.score > stats[levelKey].bestScore){
      stats[levelKey].bestScore = state.score;
    }
    stats[levelKey].lastSession = {
      score: state.score,
      accuracy: acc,
      streak: state.bestStreak,
      date: new Date().toISOString()
    };
    saveStats(stats);

    document.getElementById("complete-level-name").textContent = state.level.label + " · Complete";
    document.getElementById("final-score").textContent = state.score;
    document.getElementById("final-accuracy").textContent = acc + "%";
    document.getElementById("final-streak").textContent = state.bestStreak;
    showScreen("complete");
```

Replace with:

```js
    const isNewBest = state.score > stats[levelKey].bestScore;
    if(isNewBest){
      stats[levelKey].bestScore = state.score;
    }
    stats[levelKey].lastSession = {
      score: state.score,
      accuracy: acc,
      streak: state.bestStreak,
      date: new Date().toISOString()
    };
    saveStats(stats);

    document.getElementById("complete-level-name").textContent = state.level.label + " · Complete";
    document.getElementById("final-score").textContent = state.score;
    document.getElementById("final-accuracy").textContent = acc + "%";
    document.getElementById("final-streak").textContent = state.bestStreak;
    document.getElementById("best-indicator").textContent = isNewBest
      ? "New Best!"
      : ("Best: " + stats[levelKey].bestScore.toLocaleString() + " pts");
    showScreen("complete");
```

- [ ] **Step 4: Verify with the browser tool — first completion shows "New Best!"**

Clear stats and play one "Easy" round:

`mcp__Claude_Preview__preview_eval`: `expression: "localStorage.clear(); location.reload(); 'reloaded'"`

Then (after confirming reload via `preview_snapshot`):

```js
(async () => {
  document.querySelector('.level-card[data-level="easy"]').click();
  for(let i = 0; i < 20; i++){
    await new Promise(r => setTimeout(r, 300));
    document.querySelectorAll('.answer-btn')[0].click();
    await new Promise(r => setTimeout(r, 900));
  }
  await new Promise(r => setTimeout(r, 300));
  return document.getElementById('best-indicator').textContent;
})()
```

Expected: `"New Best!"`

- [ ] **Step 5: Verify a lower-scoring replay shows the stored best instead**

Note the `bestScore` just saved:

```js
JSON.parse(localStorage.getItem('flagChallengeStats')).easy.bestScore
```

Then play again, but click answer button 0 with no delay between question load and click (guaranteeing a much lower score than a paced playthrough, since `computeScore` rewards speed — but a 0-delay click still scores at least 5-20 points per correct/incorrect guess, which is fine, we only need *some* run whose total is lower than the noted best; if this run happens to score higher, rerun once more with slower deliberate clicking is not possible to force lower reliably via speed alone). To reliably get a lower score, play "Medium" instead (a different level, so its `bestScore` starts at 0) is not a valid test of "lower score, same level." Instead, deliberately answer as slowly as possible isn't controllable either since `computeScore` floors at 5 points after 10s elapsed — the minimum per question is 5 (not 0), so a full replay can still reach a nontrivial score. Accept this: to test the "not a new best" branch deterministically, temporarily inflate the stored best above what's reachable:

```js
(() => {
  const s = JSON.parse(localStorage.getItem('flagChallengeStats'));
  s.easy.bestScore = 999999;
  localStorage.setItem('flagChallengeStats', JSON.stringify(s));
  return s.easy.bestScore;
})()
```

Reload, then replay "Easy" once more using the same 20-click loop from Step 4.

Expected: `document.getElementById('best-indicator').textContent` is `"Best: 999,999 pts"` (not "New Best!"), and `document.querySelector('.level-card[data-level="easy"] .level-best').textContent` still reads `"Best: 999,999 pts"` after returning to the start screen (click `#btn-choose`).

- [ ] **Step 6: Reset test data**

```js
(() => { localStorage.clear(); return 'cleared'; })()
```

- [ ] **Step 7: Commit**

```bash
git add flag-challenge.html
git commit -m "feat: show New Best / current best on the round-complete screen"
```

---

## Task 4: Final verification pass against the spec

No code changes — this task re-runs the full manual verification checklist from the spec end-to-end in one sitting, using the browser tool, to confirm nothing regressed across Tasks 1-3.

- [ ] **Step 1: Fresh load shows no "Best" line and first completion shows "New Best!"**

```js
(() => { localStorage.clear(); location.reload(); return 'reloaded'; })()
```

Confirm via `preview_snapshot`/`preview_eval` that all three level cards' `.level-best` are empty, then play "Easy" once (20-click loop as in Task 3 Step 4) and confirm `#best-indicator` reads `"New Best!"`.

- [ ] **Step 2: Replay with a lower score shows the prior best**

Inflate `easy.bestScore` as in Task 3 Step 5, reload, replay "Easy" once, confirm `#best-indicator` shows `"Best: <inflated value> pts"` and the level card matches on return to the start screen.

- [ ] **Step 3: Replay with a higher score shows "New Best!" again**

```js
(() => {
  const s = JSON.parse(localStorage.getItem('flagChallengeStats'));
  s.easy.bestScore = 1;
  localStorage.setItem('flagChallengeStats', JSON.stringify(s));
  return s.easy.bestScore;
})()
```

Reload, replay "Easy" once. Expected: `#best-indicator` reads `"New Best!"` (any real playthrough score beats `1`).

- [ ] **Step 4: Reload between rounds persists stats**

```js
(() => { location.reload(); return 'reloaded'; })()
```

Then confirm `document.querySelector('.level-card[data-level="easy"] .level-best').textContent` still shows the best from Step 3.

- [ ] **Step 5: Disabled localStorage doesn't break the game**

```js
(() => {
  const orig = Storage.prototype.setItem;
  Storage.prototype.setItem = function(){ throw new Error('quota exceeded (simulated)'); };
  window.__restoreSetItem = () => { Storage.prototype.setItem = orig; };
  return 'patched';
})()
```

Play a full "Easy" round via the 20-click loop. Expected: no thrown/uncaught errors (check `mcp__Claude_Preview__preview_console_logs` with `level: "error"` — should be empty), and the complete screen still renders final score/accuracy/streak correctly (the `best-indicator` and level card just won't persist across reload in this simulated state, which is expected).

Restore normal behavior afterward:

```js
window.__restoreSetItem(); delete window.__restoreSetItem; 'restored'
```

- [ ] **Step 6: Clean up test data and stop the server**

```js
(() => { localStorage.clear(); return 'cleared'; })()
```

Call `mcp__Claude_Preview__preview_stop` with the server's `serverId`.

- [ ] **Step 7: Final commit (if any fixes were needed)**

If Steps 1-5 all passed with no code changes, there is nothing to commit here. If a fix was needed during verification, stage and commit it with a message describing the bug found and fixed.

---

## Self-Review Notes

- **Spec coverage:** data model (Task 1), level-card display (Task 2), complete-screen display (Task 3), error handling for unavailable `localStorage` (Task 1 Step 1 `saveStats`, verified in Task 4 Step 5), and all 5 manual verification scenarios from the spec (Task 4). Player-name field and paid tier are explicitly out of scope per the spec and not included here.
- **Type consistency:** `stats` shape (`{easy|medium|hard: {bestScore: number, lastSession: {score, accuracy, streak, date} | null}}`) is defined once in `defaultStats()`/`loadStats()` (Task 1) and used identically in `finishLevel()` (Tasks 1 & 3) and `renderLevelCards()` (Task 2) — no renamed fields across tasks.
- **No placeholders:** every step has literal code or a literal browser-tool expression with a stated expected result.
