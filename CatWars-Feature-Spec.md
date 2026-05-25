# CatWars.io — Feature Specification

**Version described:** v1.7.55
**Date:** 2026-05-21
**Status:** This document describes how the game **currently works** in the version above. It is the intended-behavior reference. When an AI tool edits the game and a feature stops matching this document, that mismatch is a regression — fix the code, or deliberately update this spec and bump the version.

> **How to use this file:** Keep it next to `index.html` in git. Before letting an AI change a system, re-read the relevant section here. After the change, confirm the behavior still matches — or update this doc in the same commit and note what changed.

---

## 1. Overview

CatWars.io is a single-file HTML5 game: one `index.html` containing all HTML, CSS, and JavaScript (~3,600 lines), rendered on a 2D `<canvas>`. It is a top-down `.io`-style survival action game where the player controls a cat fighting endless waves of dogs, collecting food for XP, and evolving through 50 cat forms from Kitten to God Cat.

The same file serves two distribution targets, detected at load:

- **catwars.io / beta** — full experience with Firebase (cloud save, accounts, leaderboards), Google Analytics, and Google AdSense.
- **CrazyGames iframe** — a stripped, self-contained build: no Firebase, no external analytics/ads, ads routed through the CrazyGames SDK, progress saved only to `localStorage`.

The detection flag is `window.IS_CRAZYGAMES`, set by the first `<head>` script by inspecting `ancestorOrigins` / `document.referrer` for `crazygames.com`. A separate `window.IS_BETA` flag (hostname starts with `beta.`) selects the beta Firebase project.

---

## 2. Distribution & Environment Branching

### 2.1 CrazyGames build differences
When `IS_CRAZYGAMES` is true:
- The CrazyGames SDK v3 is loaded; `gameplayStart` / `gameplayStop` are reported on run start/stop, and rewarded ads route through `CrazyGames.SDK.ad.requestAd`.
- **No external network**: Firebase, Firestore, Google Auth, GA4, AdSense, and the hot-linked GitHub banner are all skipped or hidden via CSS (`html.crazygames`).
- The auth UI (`#auth-section`, `#login-screen`) is hidden — players are anonymous guests, progress in `localStorage` under key `catwars_save_cg_guest`.
- First-time visitors land on a minimal **`#cg-landing`** overlay (single PLAY button), not the full main menu. Returning players (`totalKills > 0`) get the full menu.
- The death-screen title reads **DEFEATED** (a checklist item; the build comment flags "not WASTED").

### 2.2 Loading sequence
- A real progress bar (`#loading-bar-fill`) is bound to milestones: ~25% on boot, 45/70% during Firebase fetch, 100% ("Welcome!") when the session is ready. The old fake "85%-then-wait" ramp was intentionally removed.
- After loading, control flows to `showLobby()`.

### 2.3 Lobby / entry routing (`showLobby`)
- **CrazyGames:** new player → `#cg-landing`; returning player → full menu.
- **Desktop (mouse/keyboard, `pointer: fine`):** lobby is skipped entirely; goes straight to the main menu. The lobby exists mainly to provide a user-gesture for mobile fullscreen + orientation lock.
- **Touch devices (`pointer: coarse`):** the `#lobby-screen` is shown (yellow title, green PLAY button, About text). Clicking PLAY (`enterFromLobby`) shows the main menu, sets `post-lobby` (arms the rotate-device overlay on portrait), and requests fullscreen + landscape lock.

---

## 3. Core Game Loop & Controls

### 3.1 Controls
- **Move:** the cat follows the mouse cursor (desktop) or virtual joystick (touch, bottom-left). The cat always faces its movement direction.
- **Attack:** Left Click (desktop) / ⚔️ button (touch). Hits a **cone in front of the cat** — facing matters; a target behind the cat is not hit.
- **Dash:** Space or Right Click (desktop) / ⚡ button (touch). Short burst with brief invulnerability; has a cooldown (default 3.0s). Disabled by the No Dash hardcore modifier.
- **Pause:** Esc or the pause button. Esc also closes the Codex if open.

### 3.2 Game states
`gameState` is one of: `menu`, `playing`, `paused`, `death_menu`, `game_over`. The main `gameLoop` runs continuously via `requestAnimationFrame` but only simulates when `playing`. Delta time is clamped to a max of 0.1s per frame.

### 3.3 World
- World size: **6000 × 6000**. Spatial grid cell size: **400** (used for spatial-hash collision/proximity queries via `gridEnemies`, `gridFoods`, `gridGems`, `gridAiCats`, rebuilt every 2nd frame).
- Initial entity counts: **800** food, **45** enemies, **300** decorations, **25** chests.
- Camera zoom scales down as the player levels (`targetZoom = max(0.45, 1/(1 + (level-1)*0.02))`), so higher-level cats see more of the map.

### 3.4 Hit-stop
On a successful hit, `hitStop` briefly freezes simulation (≈0.05s normal, ≈0.08–0.09s on heavier/crit hits) for impact feel. Timers still tick during hit-stop so dashes/cooldowns don't stall.

---

## 4. Cat Evolution (50 Levels)

The `TIER_VISUALS` array defines 50 tiers, each with a cat name, paired dog name, weapon type, colors, and a visual pattern. The `evolutions` array derives per-level stats from these:

- `size = 20 + level * 1.2`
- `speed = max(120, 310 - level * 0.8)`
- `weaponLength = size * 2.5`
- Per-level scaling multiplier: `1.25^(level-1)`, applied to:
  - `nextXp = floor(50 * mult)` (XP to next level)
  - `maxHp = floor(100 * mult)`
  - `playerDamage = floor(45 * mult)`
  - `enemyDamage = floor(20 * mult)`

Level 1 = Kitten, Level 50 = God Cat. The matching dog at each tier is the "Bestiary" counterpart (Level 1 dog = Chihuahua … Level 50 = Supreme Overlord).

**Leveling up** (`updateLevel`): while `xp >= nextXp`, consume XP, increment level, recompute max HP (including helmet + boost multipliers), **fully heal**, grant 1.5s invuln, show a "LEVEL UP!" effect, and update `maxCatLevel` in saved progress if it's a new record. Capped at level 50.

---

## 5. Weapons, Upgrades & Archetypes

### 5.1 Weapon archetypes (`ARCH_STATS`)
Every weapon maps to one of five archetypes (via `getWeaponArchetype` keyword matching), each with its own cooldown/damage/reach/knockback/sweep-angle profile:

| Archetype | CD mult | Dmg mult | Reach mult | Knockback | Sweep (half-cone) |
|-----------|---------|----------|------------|-----------|-------------------|
| balanced  | 1.0     | 1.0      | 1.0        | 150       | π/2               |
| fast      | 0.5     | 0.8      | 0.9        | 20        | π/3               |
| heavy     | 1.5     | 1.5      | 1.0        | 600       | π/2.2             |
| wide      | 1.2     | 0.8      | 1.1        | 80        | π/1.5             |
| reach     | 1.15    | 0.85     | 1.35       | 250       | π/2.5             |

### 5.2 Weapon upgrade tiers
Permanent, per-cat-level weapon upgrades bought with gems in the Codex.
- Tiers: `DEFAULT, BRONZE, SILVER, GOLD, DIAMOND, LEGENDARY` (indices 0–5).
- Damage multipliers (`UPGRADE_MULT`): `1.0, 1.25, 1.6, 2.2, 3.5, 7.0`.
- Size/reach multipliers (`UPGRADE_SIZE_MULT`): `1.0, 1.1, 1.2, 1.3, 1.5, 1.7`.
- Buy costs scale with level: `level * UPGRADE_COSTS[currentTier]`, where `UPGRADE_COSTS = [10, 25, 60, 150]` (Bronze→Diamond). Diamond (tier 4) is the buyable max; Legendary (tier 5) is forged separately.

### 5.3 Helmets
Permanent, per-cat-level HP upgrades.
- Tiers: `NO HELMET, BRONZE, IRON, SILVER, GOLD, DIAMOND` (indices 0–5).
- HP multipliers (`HELMET_HP_MULT`): `1.0, 1.2, 1.5, 1.9, 2.5, 3.5`.
- Costs: `level * HELMET_UPGRADE_COSTS[currentTier]`, where `HELMET_UPGRADE_COSTS = [15, 35, 80, 200, 450]`.
- Helmet visual variant is chosen by cat name (knight/barbarian/spartan/valkyrie/samurai-ninja themes).

### 5.4 Legendary Forge
Unlocks per cat level once that level's **weapon reaches Diamond** (tier 4).
- 5 legendary archetypes (Dagger=fast, Katana=reach, Hammer=heavy, Scythe=wide, Broadsword=balanced).
- 5 visual brackets by level range (1–10 / 11–20 / 21–30 / 31–40 / 41–50), each with its own neon palette and hilt/guard geometry → **25 unique visual forms**.
- Cost: `level * 1000` gems. First purchase auto-equips.
- All legendaries apply **7× base damage** (tier-5 `UPGRADE_MULT`) × the archetype's damage multiplier.
- Owned legendaries stored in `prog.unlockedLegendaries[level]` (array); equipped in `prog.equippedLegendary[level]`. When equipped, it overrides the level's default weapon type, archetype, and tier for both rendering and combat (`resolveCatWeapon`).

---

## 6. Combat, Combos & Damage

### 6.1 Player attack resolution
On attack, once `attackProgress >= 0.25` and not yet swung this stroke, the game checks all chests/enemies (and AI cats in Deathmatch) within reach. Reach = `playerSize * 1.5 + weaponLength * reachMult`. Enemies must also be within the archetype's sweep cone. Chests ignore the angle check (inanimate — hit on overlap).

### 6.2 Combo system (`BALANCE` constants)
Combo increments on each kill; resets if no kill within `COMBO_WINDOW` (3.0s) or when the player is hit (unless Hunter's Eye boost is active). Tiered bonuses:

- **5× (`COMBO_TIER_SPEED`)** — +25% movement speed (and a golden trail).
- **10× (`COMBO_TIER_CRIT`)** — 15% crit chance (`COMBO_CRIT_CHANCE`); crit = ×2 damage.
- **20× (`COMBO_TIER_AOE`)** — hits splash nearby enemies for 25% (`COMBO_AOE_RATIO`) within radius 200 (`COMBO_AOE_RADIUS`).
- **30× (`COMBO_TIER_PIERCE`)** — "unstoppable" swings ignore hit-invulnerability frames on enemies.

Combo damage curve: base `+3% per stack up to +30%`; Hunter's Eye adds `+1% per stack up to +50%`.

> **AoE kill safety:** combo-AoE kills are deferred into `_pendingAoEKills` and flushed after the enemies loop finishes, to avoid array-index corruption from splicing mid-iteration.

### 6.3 Player kill rewards (`finalizePlayerDogKill`)
On a player-credited kill: grant XP (`floor(25 * 1.20^(level-1))`), increment kills + career totals, record Bestiary defeat, advance combo + quests, and drop gems/XP-orbs depending on enemy type (boss = 100 orbs, miniboss = 30, elite = bonus gems, higher-level dog = a few gems, otherwise a 25% chance of 1 gem). Killing the final boss triggers the win.

---

## 7. Enemies, Bosses & Elites

### 7.1 Regular dogs (`spawnEnemy`)
Spawn level is weighted around the player's level (weaker early, wider spread past level 10). Stats come from the `evolutions` table for that level, with archetype modifiers. Dogs use a pack-based state machine (`wander` / `chase` / `flee`): they flee unless the local pack is strong enough or the player is high level. Enemies have a windup before attacking (default 350ms).

### 7.2 Elites
Rare purple-tinted dogs. Conditions: not Hardcore, player level ≥ 15 (`ELITE_MIN_LEVEL`), 12% chance (`ELITE_CHANCE`). They get ×1.5 HP, ×1.3 damage, ×1.15 size, and drop bonus gems (`ELITE_GEM_BONUS = 3` + random).

### 7.3 Minibosses (`MINIBOSS_VARIANTS`)
Two variants (Elite Hound, Bone Crusher). Spawn timer starts at 30s, repeats ~every 50s, capped at 2 alive. ~1.3× size, scaled HP/damage by player level.

### 7.4 Bosses (`BOSS_VARIANTS`)
Three variants (Swift, Tank, Mega) with distinct HP/damage/speed/ability (`dash` or `stomp`) and auras. Timing constants:
- `INITIAL_BOSS_DELAY = 60s` (first boss)
- `BOSS_RESPAWN_AFTER_KILL = 90s`
- `BOSS_TIMEOUT = 120s`
Boss abilities telegraph with on-canvas danger zones (stomp AoE ring, dash cone). Double Bosses modifier spawns two. An on-screen arrow (`#boss-arrow`) points to off-screen bosses.

### 7.5 Final boss / win (`spawnFinalBoss`, `triggerWin`)
When the highest cat level on the field reaches **50** (player, or any AI cat in Cats vs Dogs), a 5-second "cosmic disturbance" warning plays, then **THE OMEGA BOSS** spawns at world center (~2.8× size, infinity-blade, omega aura, very high HP/damage). Defeating it triggers the win: a celebration burst plays for ~2s, best time is recorded for the mode, then the victory death-screen shows. (`gameState` is intentionally left as `playing` during the celebration so explosions animate.)

---

## 8. Game Modes

Selected via the mode pills; `selectedGameMode` ∈ `solo`, `hardcore`, `team`, `ffa`.

- **Solo Survival** — classic mode; survive, evolve, beat the Omega Boss.
- **Hardcore** — opens a modifier modal before starting. Modifiers and gem-multiplier bonuses: Double Damage (+200%), Fast Dogs (+50%), No Food Heal (+100%), No Dash (+50%), Double Bosses (+100%). The total multiplier (`calcModMultiplier`) starts at 1.0 and sums these. **Competitive Hardcore** locks all five on and feeds a separate leaderboard.
- **Cats vs Dogs (`team`)** — 8 AI cat allies fight alongside the player (level up, eat food, fight dogs). A teammate-kills counter is shown.
- **Deathmatch (`ffa`)** — free-for-all cats vs cats PvP plus the dog horde. AI cats target weaker/equal cats and the player; flee from much stronger ones. Dead bots respawn after ~8s (capped at 8 bots). Anti-stomp rules throttle bot leveling relative to the player; the death screen attributes who killed you.

---

## 9. AI Cats

Spawned in `team` and `ffa` modes (`spawnAICat`). Each has a personality (`aggro` / `defender` / `greedy`), a level near the player's, weapon/helmet tiers derived from level, and a name from `BOT_NAMES` (with `II`, `III`… suffixes once the pool is exhausted). They run a state machine — `wander`, `chase`, `flee`, `follow_player`, `greedy_gem`, and (FFA only) `pvp_chase` — using cached proximity queries refreshed every 4th frame. They eat food, collect gems, level up, and (in FFA) attack/kill other cats and the player. Dead AI cats drop gems and spawn a temporary tombstone decoration.

---

## 10. Items: Food, Gems, Chests

### 10.1 Food (`spawnFood`)
Types weighted by rarity: `dot` (most common, small XP), `fish`, `mouse`, `milk` (rarest). Eating heals if below max HP (milk 12% / mouse 6% / fish 3% / dot 1% of max HP); otherwise converts to XP. The No Food Heal hardcore modifier forces all food to XP.

### 10.2 Gems & XP orbs (`dropGems`)
Dropped gems drift outward then get pulled in within the player's pickup radius (`GEM_PICKUP_RADIUS = 250`, ×2.4 with Magnet Collar). Boss "gems" are larger gold XP orbs (`isXp`). Pickup has an 0.8s delay so they visibly explode before collection. Gem value scales: base 4 (boss orb) or 1, × hardcore multiplier × level scale (+5%/level) × Lucky Paw multiplier.

### 10.3 Chests (`CHEST_TYPES`)
Four types by spawn weight: wood (70%, 500 HP, 3–6 gems), iron (20%, 1500 HP, 10–18), gold (8%, 4000 HP, 30–60), diamond (2%, 10000 HP, 120–250). Broken by attacking; gems scale with Lucky Paw. Opening counts toward quests; diamond chests have their own quest.

---

## 11. Meta-Progression

### 11.1 Gems
Permanent currency. Earned in-run, then **banked** to the meta profile when a run ends (`bankEndedRunToMeta`). Spent in the Codex on weapon/helmet/legendary upgrades and boosts.

### 11.2 Global rank & XP (`getGlobalRankInfo`)
A separate progression from cat level: 60 global levels across 12 named rank tiers (Stray Recruit → The One True Meow), 5 stars per tier. `nextXp = level * 1000`. Banking a run grants global XP = `(catLevel * 50) + (kills * 10) + (bossKills * 500)`. Top tiers use animated gradient stars.

### 11.3 Boosts (`BOOSTS`)
Pre-game consumables; equip up to **3** (`EQUIP_SLOTS`), consumed (1 charge each) at run start. Examples: Cream of Champions (start at level 5), Whetstone (+25% dmg), Iron Whiskers (+30% HP), Catnip Rush (dash CD 3.0→1.5s), Lucky Paw (+50% gems), Magnet Collar (×2.4 pickup), Nine Lives (auto-revive once at 50% HP), Hunter's Eye (combo never resets + dmg-per-stack), Healing Purr (1.5% HP/s regen). Bought with gems or earned via daily spin / ad.

### 11.4 Daily Spin & streaks
One spin per ~22h (`DAILY_SPIN_COOLDOWN_MS`). Streak increments if played within 48h (`DAILY_STREAK_RESET_MS`), else resets; 7-day streak = ×2 rewards. Reward table: 200/500/1000 gems, a random boost, or a 2500 jackpot.

### 11.5 Daily Quests (`QUEST_POOL`)
5 active quests per day, rolled fresh each calendar day. Types: kills, level reached, chests, combo, elite kills, boss kills, diamond chest, hardcore win, career total kills. Completing grants gems on claim.

### 11.6 Ad rewards
Rewarded ads (AdSense H5 or CrazyGames SDK) can grant a random boost (4h cooldown, `AD_BOOST_COOLDOWN_MS`). **Ad-revive is currently disabled on all platforms** — the "Watch Ad to Revive" button is force-hidden in code (one-line revert re-enables it).

---

## 12. Death & Revive

On player death (`showDeathScreen`): if **Nine Lives** is available, auto-revive at 50% HP and clear nearby enemies (no death screen). Otherwise the death screen shows run stats (level, kills, best combo, gems found, current bank). Revive options:
- **Gems:** cost = `ceil((level*10 + 20) * hardcoreMultiplier)`, paid from run gems first, then bank.
- **Ad revive:** currently hidden (see 11.6).

Death buttons fade in after ~900ms. "New Game" and "Quit to Menu" both bank the run before proceeding.

---

## 13. Persistence & Sync (Firebase)

### 13.1 Save model
`prog` (the meta profile) is saved to `localStorage` keyed by UID, and — for registered/anonymous Firebase users on catwars.io/beta — to Firestore. CrazyGames and offline guests are localStorage-only.

- **Local saves** are cheap and frequent (debounced).
- **Cloud saves** happen only at meaningful moments: banking a run, Codex purchases, daily spin, nickname change.

### 13.2 Auth bootstrap (important)
The code deliberately does **not** call `signInAnonymously()` eagerly at load. It waits for the first `onAuthStateChanged` callback (which fires only after Firebase restores the persisted user from IndexedDB). A `user === null` there means a genuine first visit/sign-out — only then is an anonymous session created. This prevents the "every refresh logs me out" bug. A 5s timeout falls back to an offline guest if Firebase hangs.

### 13.3 Conflict protection
- On load, cloud vs local is reconciled by `totalKills` (and a write block if the cloud fetch failed, to avoid overwriting good cloud data with defaults).
- On save, the code re-reads cloud `lastSavedAt`; if the cloud copy is newer than the snapshot in memory (beyond a 2s clock-skew tolerance), it **blocks the write** and shows a "newer progress on another device — reload" banner.
- A snapshot older than 60s triggers a pre-run cloud refresh before starting.

### 13.4 Accounts
Email (mapped to a fake `@catwars.local` address from the nickname) or Google sign-in. Guest→account uses credential **linking** to preserve progress; if the account already exists, it signs in cleanly **without** merging guest data (anti account-duplication rule). Logout from a registered account starts a fresh zeroed guest.

### 13.5 Leaderboard
Public Firestore collection. Categories: Total Kills, Best Solo, Best Hardcore (competitive), Best Cats vs Dogs, Best Deathmatch. Time-based tabs sort ascending (fastest run), kills descending; ties broken by max cat level. The current player's row is pinned below the table.

---

## 14. Nickname Handling

`commitPlayerNickname` is the **only** writer of `prog.nickname`. Triggered by the ✓ button, Enter, blur, and Play (if non-empty). Empty names are never saved, and the runtime default `'Hero'` is never persisted (it's in-game only) — this keeps the leaderboard human-readable. Registered accounts push the new name to cloud immediately so the leaderboard updates without waiting for a banking save.

---

## 15. Onboarding & Tutorial

- **Onboarding overlay** (`OB_STEPS`): a 6-step first-run carousel (welcome → move → attack → eat → dash → gems) shown once for new players before their first run. `startGame()` re-invokes itself after onboarding completes/skips. Has desktop and mobile (`descMobile`) copy variants. Completion sets `prog.tutorialDone`.
- **In-game tips** (`tutorialSteps`): timed contextual hints shown during the first run only (`prog.inGameTipsDone`), also with mobile text variants.

---

## 16. Audio

Web Audio API oscillator-based SFX (`playSound`): swing, hit, meow, gem, dash, boss, combo, heartbeat. Throttled per type to avoid spam. A **heartbeat** ticker plays while HP is under 25% (and stops above it / when not playing). Sound toggle persists and syncs to ad-video sound state. Audio context is initialized lazily on first user input.

---

## 17. Rendering Notes

Entities, weapons, helmets, and faces are drawn procedurally on canvas (no image assets). Sprites are cached as data URLs (`spritesCache`, `weaponCache`) keyed by level/tier/helmet/legendary so they aren't re-rendered every frame. The Codex builds preview thumbnails from the same sprite functions. Legendary weapons (tier 5) use flat-shaded bracket geometry; standard weapons use a shared `drawWeaponShape` with per-weapon shapes. Particles and damage numbers use object pools (`MAX_PARTICLES = 250`) to limit GC churn.

---

## 18. Tuning Surface (`BALANCE` constants)

Central place to adjust balance without touching the game loop:

| Constant | Value | Meaning |
|----------|-------|---------|
| COMBO_WINDOW | 3.0 | Seconds before combo resets |
| COMBO_TIER_SPEED / CRIT / AOE / PIERCE | 5 / 10 / 20 / 30 | Combo thresholds |
| COMBO_CRIT_CHANCE | 0.15 | Crit chance at 10× |
| COMBO_AOE_RADIUS / RATIO | 200 / 0.25 | AoE splash at 20× |
| DASH_COOLDOWN | 3.0 | Base dash cooldown (s) |
| GEM_PICKUP_RADIUS | 250 | Base pickup radius |
| ELITE_MIN_LEVEL / CHANCE | 15 / 0.12 | Elite spawn gate |
| ELITE_HP_MULT / DMG_MULT / GEM_BONUS | 1.5 / 1.3 / 3 | Elite buffs |
| DAILY_SPIN_COOLDOWN_MS | 22h | Spin cooldown |
| DAILY_STREAK_RESET_MS | 48h | Streak window |
| AD_BOOST_COOLDOWN_MS | 4h | Ad-boost cooldown |

Other top-level constants worth knowing: `WORLD_SIZE` 6000, `FOOD_COUNT` 800, `ENEMY_COUNT` 45, `DECORATION_COUNT` 300, `CHEST_COUNT` 25, `GRID_CELL` 400, `GAME_VERSION` '1.7.55'.

---

## Change log

| Version | Date | Notes |
|---------|------|-------|
| 1.7.55 | 2026-05-25 | Hardcore gem multiplier now applies to gem rewards and revive cost via `calcModMultiplier()` at run start. |
| 1.7.54 | 2026-05-25 | Combo tier player aura; attack cone telegraph during swing; reduced enemy shadowBlur cost; Hardcore label "Gem Multiplier". |
| 1.7.53 | 2026-05-21 | Initial spec written from current code. |

*(Add a row each time behavior changes. Keep `GAME_VERSION` in the code and the version at the top of this file in sync.)*
