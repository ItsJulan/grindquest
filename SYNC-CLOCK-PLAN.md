# Sync & Clock — findings and proposal

**Status:** two bugs already fixed and deployed (2026-09-02). The rest is proposed, not built.

---

## Already fixed and live

**1. Dropped cloud writes.** `cloudSave()` used the single `syncPaused` flag for two
unrelated jobs — "we are applying remote data" *and* "a push is in flight". Any
`save()` that happened while a push was awaiting hit `if(syncPaused)return` and was
**silently discarded, never retried**. Completing three quests in quick succession
pushed only the first snapshot; the rest existed only on that device until some later
save happened to land outside the window.

Now writes coalesce: a save during an in-flight push sets a dirty flag, and the newest
state is re-pushed when the write settles. Verified — four rapid saves at coins 0→3
push `[0, 3]`, ending on the true state. Previously it pushed `[0]` and lost the rest.

**2. Nothing ever noticed the day rolling over.** `checkMiniReset()`, `genD()` and
`genW()` only ran on interaction. There was no `setInterval`, no `visibilitychange`
handler anywhere in the file. An app left open — or, on a phone, restored from bfcache,
which never re-runs `load()` — kept yesterday's mini quests, dailies and "today" until
you happened to touch the right screen.

Now there's a watchdog on `visibilitychange`, `focus`, `pageshow` and a 60s timer.

> **Re-test before we build anything below.** These two shipped after the report of
> "still glitching", and either could be the whole cause.

---

## The root problem: the device clock is the referee

```js
save = function(){ D._lastSync = Date.now(); _origSave(); cloudSave() };
```

Every conflict is resolved by comparing `_lastSync` values that each device wrote using
**its own clock**:

```js
if (cloud._lastSync > (D._lastSync || 0)) { /* overwrite local */ }
```

Consequences when two clocks disagree, even by a minute:

| skew | result |
|---|---|
| Phone 2 min fast | Phone always wins. A stale phone write silently overwrites newer laptop data. |
| Phone 2 min slow | Phone writes are **permanently ignored** — its `_lastSync` never exceeds the laptop's. Work vanishes on reload. |
| Clock corrected backwards | Device stops being able to write until real time catches up. |

This is not hypothetical: the sandbox browser I tested in reported **Sep 3**, then
**Sep 2** an hour later. Phone clocks drift, auto-correct, and jump on timezone change.

The same device clock also drives `tdy()`, `miniDay()` and `wkS()`, so it decides when
mini quests reset and whether a streak survives.

**Your instinct is right — the fix is a clock that isn't the device's.**

---

## Proposal

### A. Server clock — *recommended, this is the one you suggested*

Firebase already exposes it; no new infrastructure.

```js
let srvOffset = +(localStorage.getItem('gq_offset') || 0);   // survives cold start
firebase.database().ref('/.info/serverTimeOffset').on('value', s => {
  srvOffset = s.val() || 0;
  localStorage.setItem('gq_offset', srvOffset);
});
function now(){ return Date.now() + srvOffset; }
```

Then `tdy()`, `miniDay()`, `wkS()` and `updateStreak()` all read `now()` instead of
`Date.now()`. The 6am rollover becomes the same instant on every device regardless of
what the phone thinks the time is.

Offline it falls back to the last known offset, which is far better than nothing —
drift over a day is seconds, not hours.

**~25 lines. Low risk.** The only care needed: the offset is unknown on a very first
cold start with no network, where it degrades to today's behaviour.

### B. Server-stamped `_lastSync` — *recommended, do with A*

```js
_lastSync: firebase.database.ServerValue.TIMESTAMP
```

The server fills the value in. Both sides then compare numbers written by the *same*
clock, so skew cannot decide a conflict. Requires reading the resolved value back after
the write.

**~15 lines. Low risk. Highest value per line of anything here.**

### C. Connection status in the UI — *recommended, trivial*

Sync failures are currently `console.warn` only, so a broken sync looks identical to a
working one. `/.info/connected` gives live state; surface it on the existing sync button
as synced / syncing / offline.

**~20 lines. No risk.**

### D. Revision counter + guarded write — *optional*

A monotonic `_rev` incremented per write, compared before `_lastSync`, applied through a
transaction that refuses to overwrite a higher `_rev`. Immune to clocks entirely.
Belt-and-braces on top of B.

**~40 lines. Medium risk** — transactions retry, so the write path needs care.

### E. Field-scoped writes — *optional, biggest win for real multi-device use*

Today every save is a whole-document `set()`, so a dungeon run on the laptop and a quest
completed on the phone overwrite each other wholesale. Splitting into
`saves/CODE/core`, `/quests`, `/dg`, `/log` and writing only what changed makes those two
edits merge instead of collide.

**~120 lines and a migration. Medium-high risk** — worth it only if you genuinely use
two devices at once.

---

## Suggested order

1. Re-test now that the two fixes are live.
2. If it still misbehaves: **B → A → C** (~60 lines total, all low risk).
3. Only then consider D and E.
