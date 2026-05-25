# Catwars.io — Agent Instructions

These rules apply to every change in this project. Read them before editing
`index.html`.

> **Companion document:** a feature spec (`CatWars-Feature-Spec.md`) describes
> the game's *intended behavior* as of a stated version. It is a **reference,
> not a mandate**: if the code and the spec disagree, **the code wins** and the
> spec should be updated. Check the spec's version header against `GAME_VERSION`
> in the code — if `GAME_VERSION` is higher, treat the spec as possibly stale
> and verify against the code before relying on it. AGENTS.md (this file) is the
> mandatory rules-of-the-road; the spec is background on what each system does.

---

## 1. Project shape

- **Single-file project.** The entire game lives in `index.html`
  (~4 000+ lines: HTML + inline CSS + inline `<script>`). **Do not** split it
  into separate `.js`/`.css` files, do not introduce a build step, and do not
  add `npm`/bundler dependencies unless explicitly asked.
- **Vanilla JavaScript only.** No frameworks (React, Vue, jQuery, …), no
  TypeScript, no transpilation. Plain `<script>` tags and ES2020 syntax that
  runs directly in modern browsers.
- The file is long. **Locate edit targets by function name or unique nearby
  string, never by absolute line number** — lines shift constantly.

## 2. Editing style

- Make **minimal, targeted edits**. Don't rewrite untouched blocks just to
  reformat them.
- Preserve surrounding **indentation and brace structure exactly**. The
  game loop has deeply nested `if` / `for` / `try` blocks where a single
  stray `}` causes `Unexpected token 'catch'` and a hard parser failure.
- Before any non-trivial change, briefly state which function / which
  `if` / which loop you're inserting into.

---

## 3. Game context & lore

- **Genre:** 2D top-down `.io` survival action (like Vampire Survivors, but
  with active mouse-click attacks).
- **Goal:** the player is a cat, collects food (XP) and fights endless dog
  hordes. 50 evolution levels (Kitten → God Cat).
- **Modes:** Solo Survival · Hardcore (with difficulty modifiers) ·
  Cats vs Dogs (with AI cat allies) · Deathmatch (FFA — cats vs cats PvP plus
  the dog horde).
- **Meta-progression:** Gems are the currency. Spent on permanent weapon
  and helmet upgrades in the menu (Encyclopedia / Codex).
- **Backend:** Firebase (Auth, Firestore for progress + leaderboard) with a
  robust **Offline Guest** fallback.

## 4. Core arrays & entities

Central in-memory arrays that hold game state:

| Array | Contents |
|---|---|
| `player` | The main player object |
| `enemies` | Dogs — small grunts up to minibosses and bosses |
| `aiCats` | Fake multiplayer — bots with their own AI that level up, kill, and steal |
| `foods`, `droppedGems`, `chests`, `particles`, `damageNumbers` | Other game entities |

---

## 5. Critical traps (must follow)

### 5.1 The `gameLoop` trap

`gameLoop()` (starts around `function gameLoop(currentTime)`) is huge.
Inside it there are **three similar attack / collision passes** that look
alike but use different loop variables and field names:

```js
// A) AI cats — iterates aiCats, variable `c`, target is `targetDog`
aiCats.forEach(c => {
    // ... c.xp, c.level, c.invuln ...
});

// B) Player vs enemies — iterates enemies, variable `e`
enemies.forEach(e => {
    // ... e.playerDamaged ...
});

// C) Deathmatch (FFA) only — player vs AI cats, variable `ac`
// (a backwards index loop over aiCats inside the player-attack block).
// Named `ac` ("ai cat") on purpose so it stays distinct from `c` (the AI-cat
// pass above) and `c` used for chests in the same attack block.
for (let ai = aiCats.length - 1; ai >= 0; ai--) {
    let ac = aiCats[ai];
    // ... ac.playerDamaged ...
}
```

**Never copy-paste logic between these blocks.** They use different variable
names (`c` vs `e` vs `ac`) and different field names (`c.xp` vs
`e.playerDamaged` vs `ac.playerDamaged`). Mixing them up corrupts brace
pairing and crashes the parser on `Unexpected token 'catch'`.

### 5.2 Performance — use the spatial grid

Never do `enemies.forEach` / `foods.forEach` **nested inside every frame**
for distance / proximity checks (`O(N²)` blow-up). The game uses spatial
hashing. For neighborhood lookups always call:

```js
queryGridMap(gridEnemies, x, y, radius)
queryGridMap(gridFoods,   x, y, radius)
queryGridMap(gridGems,    x, y, radius)
```

The grids are **rebuilt only on even frames** (`frameCounter % 2 === 0`),
so results can be one frame stale — that's fine, don't "fix" it.

### 5.3 AI separation (Hive-mind fix)

Bots and enemies **must** have a personal separation force, otherwise they
all chase the same target and clump into a single blob. When you add or
modify movement:

- Compute overlap with neighbours and apply a repulsion vector.
- Multiply movement deltas by **`dt * 60`** so the force is consistent
  across refresh rates (Fixed-Timestep convention used throughout the file).

### 5.4 Audio & visual throttling

With up to 8 AI cats on the map, dozens of effects can trigger per second.

- **Visibility gate** — gate sounds and heavy particle bursts on player
  view distance:

  ```js
  if (Math.hypot(player.x - x, player.y - y) < viewRadius) {
      playSound('hit');
  }
  ```

  Don't play sounds from the far side of the map.

- **Audio throttle inside `playSound`** intentionally caps overlap (e.g.
  one "hit" sound per ~0.05 s). **Do not touch the throttle logic** in
  `playSound(type)` — it exists on purpose.

### 5.5 Firebase & offline fallback

The game has `window.startLocalSession()` as a fallback for iframes /
browsers that block third-party cookies. **Do not change the connection-wait
logic** — without it the loading screen freezes forever.

All database writes through `window.saveToFirestore(...)` must run
**asynchronously in the background**. Never call it every frame.

---

## 6. Third-party integrations (production — DO NOT MODIFY)

These IDs in `<head>` are live production values. Don't change, remove, or
"clean up" them:

- **GA4 (gtag.js):** `G-R74QC0ZMG5`
- **Google AdSense:** `ca-pub-8480814290653565`

---

## 7. Workflow checklist (before finishing an edit)

1. Did I locate the target by function name, not line number?
2. Am I editing the right block (AI-cat `c` vs player/enemy `e` vs FFA `ac`)?
3. For per-frame proximity work: am I using `queryGridMap(...)`?
4. For new movement code: did I apply separation × `dt * 60`?
5. For new sound / VFX: is it gated by `viewRadius`?
6. Are brace pairs still balanced around `try` / `catch` blocks?
7. Did I avoid splitting the file or adding dependencies?

## 8. Add version number in the footer of the menu. Number each change and use 1.1.11 system and change number depending on the size of the change.