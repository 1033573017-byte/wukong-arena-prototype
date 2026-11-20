# Wukong Arena Prototype — Design & Architecture

A playable browser action prototype inspired by Black Myth: Wukong.
One arena, one boss. The core loop is stamina management plus precision
parries: attacks and dodges drain stamina, a well-timed parry refunds
stamina and opens a counter window.

- Stack: TypeScript + Vite, Canvas 2D rendering
- Simulation: fixed 60 Hz timestep, logic fully decoupled from rendering
- All combat numbers live in JSON frame data — no hard-coded values

## 1. Game loop (fixed timestep)

- Accumulator pattern: `acc += min(frameTime, 100ms)`, then run logic
  steps of 1/60s while `acc >= STEP`, at most **5 steps per frame**.
- If backlog remains after 5 steps it is discarded: the game briefly
  slows down instead of entering a spiral of death.
- Rendering interpolates between the last two logic states.
- All hit/parry decisions are made on logic frames only, never on
  render frames, so display hitches cannot change combat fairness.

## 2. Player state machine

States: `idle`, `run`, `attack`, `parry`, `hurt`, `dodge`, `counter`.
Transitions are an explicit table in code; illegal transitions are
rejected and logged in debug builds.

Attack is three phases, all frame counts read from `data/frames.json`:

| Phase   | Frames | Notes                       |
| ------- | ------ | --------------------------- |
| startup | 6      | no hitbox                   |
| active  | 4      | hitbox live                 |
| recovery| 10     | cancellable into dodge only |

### Parry (the core mechanic)

- Parry action total: 18 frames — 2 startup, **6 active parry frames
  (frames 3–8, 100 ms at 60 Hz)**, 10 recovery.
- Input buffering: 6 frames. An early parry press is buffered and fires
  the moment the active window opens.
- Per-tick resolution order is fixed: (1) gather buffered input,
  (2) update defender state machine, (3) hit detection, (4) apply
  damage. Therefore, if an attack's first active frame and a parry
  trigger land on the **same logic frame, the parry wins**.
- Stamina ledger (max 100):
  - parry attempt pre-costs 10 on startup;
  - successful parry refunds the 10 and adds +15 (net +15) and opens a
    24-frame counter window;
  - whiffed parry refunds nothing;
  - if stamina < 10 the parry cannot start at all — no negative stamina.
- Normal attacks cost 12, dodge costs 20, sprint drains 8/s.
  Stamina regens at 14/s after 45 frames without spending.

## 3. Frame data format

See `data/frames.json` (player) and `data/boss_moves.json` (boss).
Every move declares `startup`, `active`, `recovery`, `damage`,
`staminaDamage`, and for boss moves also `parryable` and `telegraph`.
The engine refuses to load a move whose phases do not sum to its
declared total length.

## 4. Boss design

One boss. Decision-making is a **behavior tree**; move execution is a
state machine; the move table is data-driven from
`data/boss_moves.json`.

- Tree root selects by HP phase: P1 100–60%, P2 60–30%, P3 below 30%.
  Later phases get longer strings and more feints.
- Distance layer: far → gap-closer dash attack; mid → combo strings;
  close → fast swipes and a grab.
- Move selection is cooldown-weighted random: the same move cannot be
  picked twice in a row above a small probability floor.

### Telegraphs

- Parryable moves: weapon flashes **white** plus a high audio cue at
  startup; startup is 18–24 frames, enough to react to.
- Unblockable "red" moves: weapon and eyes glow **red** plus a low
  audio cue, red starts on startup frame 1, startup is **30+ frames**,
  `parryable: false` in the data. These must be dodged. P1 uses one
  slow red move as a tutorial.

### Anti-turtle rules (standing still must lose)

- Variable startup: key moves exist in fast and slow startup variants,
  so memorized rhythm alone fails.
- The third hit of the main string is deliberately delayed to catch
  early parries.
- After the player parries twice in a row, the next pick is forced to
  be a red move or the grab.
- Successful parries build the boss's stagger meter; full meter →
  5-second punish window. Attacking pays far more than waiting.
- Player and boss resolve under identical frame-data rules.

## 5. Delivery plan (PR slices for the Devin Bot)

One branch + one PR per slice, off `main`; every PR lists changed
files and includes a gameplay capture. The human reviews and merges;
the bot never merges its own work.

1. `feat/core-loop-fsm` — Vite+TS scaffold, fixed-step loop, player
   FSM, empty arena + training dummy, hitbox debug overlay.
2. `feat/parry-window` — parry state, input buffer, same-frame
   priority, stamina ledger, counter window.
3. `feat/boss-bt` — behavior tree, move table loader, telegraphs,
   anti-turtle rules, stagger meter.
4. `feat/polish` — HUD (stamina/stagger), hit-stop, screen shake,
   GitHub Pages deployment.
