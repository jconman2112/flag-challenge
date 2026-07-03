# Free Tier Local Persistence — Design

## Context

`flag-game-two-tier-structure.pdf` proposes a two-tier product: a free tier with
local-only score storage, and a $1 paid tier that unlocks cloud sync,
accounts, and leaderboards via Supabase and native in-app purchases.

The PDF's technical shape (App Store/Play Store IAP, UserDefaults/
SharedPreferences) assumes a native mobile app. `flag-challenge.html` is
currently a single, self-contained static HTML/JS page with no backend, no
build step, and no persistence at all — scores are lost on every reload.
The eventual mobile packaging approach (Capacitor/Cordova wrapper, native
rewrite, or PWA) hasn't been decided, and that decision determines how real
payments/IAP would work.

**This spec covers only the Free Tier: local score persistence.** The paid
tier (accounts, Supabase, IAP, leaderboards) is out of scope and will get
its own spec once the mobile packaging approach is decided.

## Goal

Persist per-level score history in the browser so it survives reloads,
without any account, login, or network calls — satisfying the PDF's "no
account, no data leaves the device" free-tier requirement.

## Data model

A single `localStorage` key, `flagChallengeStats`, holding one JSON object
keyed by level:

```json
{
  "easy":   { "bestScore": 1240, "lastSession": { "score": 980, "accuracy": 85, "streak": 6, "date": "2026-07-02T14:22:00.000Z" } },
  "medium": { "bestScore": 0, "lastSession": null },
  "hard":   { "bestScore": 0, "lastSession": null }
}
```

- `bestScore`: highest total score ever achieved for that level.
- `lastSession`: snapshot of the most recently *completed* round for that
  level — score, accuracy (%), best streak reached during that run, and an
  ISO timestamp. `null` if the level has never been completed.

A level not yet played simply has `bestScore: 0, lastSession: null`; no
separate "never played" flag is needed.

## Behavior

- **On page load**: read and `JSON.parse` the `flagChallengeStats` key. If
  missing, unparsable, or `localStorage` throws (e.g. some private-browsing
  modes), fall back to an in-memory default object (`{easy:{bestScore:0,
  lastSession:null}, medium:{...}, hard:{...}}`) and continue — persistence
  failures never block gameplay.
- **On level completion** (inside the existing `finishLevel()`): update
  that level's `lastSession` unconditionally, and update `bestScore` only
  if the new score is higher. Write the whole object back to
  `localStorage` in a `try/catch` (swallow write errors silently, same
  fallback behavior as above).
- No writes happen mid-round — only on completion, matching the existing
  `finishLevel()` call site.

## UI changes

**Level cards (start screen)** — append a line under the existing
`.level-meta` text:
- If `bestScore > 0`: `Best: 1,240 pts`
- If never played: no extra line (unchanged from today).

**Complete screen** — next to the existing final score:
- If this run's score beats the previous `bestScore` (or it's the first
  completion): show `New Best!`
- Otherwise: show the stored best, e.g. `Best: 1,450 pts`, for comparison.

No changes to the game screen itself, scoring logic, or question flow.

## Error handling

- `localStorage` read/write errors (quota exceeded, private browsing,
  disabled storage) are caught and ignored; the game continues with
  in-memory-only stats for that session.
- Corrupt/unexpected JSON in the stored key is treated as if the key were
  absent (reset to defaults) rather than throwing.

## Testing / verification

Since this is a static HTML page with no test runner, verification is
manual in a browser:
1. Fresh load (cleared localStorage): level cards show no "Best" line;
   complete screen shows `New Best!` after finishing a level.
2. Replay the same level with a lower score: complete screen shows the
   prior best instead of "New Best!"; level card's "Best" line unchanged.
3. Replay with a higher score: "New Best!" shown again; level card updates
   on return to start screen.
4. Reload the page between rounds: stats persist.
5. Disable localStorage (or simulate a throwing `localStorage`) and
   confirm the game still plays end-to-end without errors, just without
   persistence.

## Out of scope (future spec)

- Paid tier: account creation, Supabase backend (`users`/`scores` tables,
  RLS), auth, cloud sync, leaderboards.
- Player display name / handle (mentioned by user as a paid-tier need).
- IAP or web payment integration (Stripe, native App Store/Play Store IAP)
  — blocked on deciding the mobile packaging approach.
- Privacy policy copy.
