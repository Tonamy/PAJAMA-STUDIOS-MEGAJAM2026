# THAT'S ALL... — Bar del Maletín
### Epic AI MegaJam 2026 · Technical Submission Notes

---

## 1. Project & Studio Overview

| | |
|---|---|
| **Studio** | Pijama Studios |
| **Category** | Experimental (Epic AI MegaJam 2026) |
| **Engine** | Unreal Engine 5.8 — **100% Blueprint project** (no C++ `Source/` module) |
| **Packaged entry map** | `/Game/BarMaletin/Maps/L_MainMenu` |
| **Gameplay map** | `/Game/DirectorsCut/BaseDirectorsCut` |

**Core experimental mechanic.**
*THAT'S ALL...* is a 1930s rubber‑hose cartoon set in a smoky harbor bar. Betty is a paper‑cutout performer (a 2D sprite in a 3D bar, with a paper‑style flip when she turns) who lives a **double shift**:

- **Undercover (exploration).** Between shows, Betty eavesdrops on tables. One randomly chosen table carries a scripted conversation that reveals a **randomly generated 4‑digit code**. She earns cash through micro‑games:
  - **Cans**: shake to read weight and sound, then trace a circle to open the lid.
  - **Glass catch**: a timing mini‑game.
- **On stage (rhythm).** On a fixed cycle the bar calls *GO TO STAGE!*. Betty must physically walk to the stairs and perform a 15‑second, 4‑lane rhythm show. Being late gives a strike; two strikes end the game.
- **Heist loop.** She bribes the guard Pedro with $200, cracks the briefcase with a mechanical 4‑wheel combination lock, and collects the **2 zipper pieces** needed to "close the cartoon" (*That's all, folks!*).

The experiment is the **tension between two clocks**: the stage cycle (10 s first, then 45 s between five shows) keeps interrupting the investigation. Every dialogue, can and glass becomes a time‑management decision.

**How MCP made it possible.**
The whole game was assembled and iterated inside a live Unreal Editor by **Claude, connected through the Model Context Protocol (MCP) server** (the `ModelContextProtocol` and `AllToolsets` plugins are enabled in the `.uproject`). Designers wrote short, ticket‑style requests. Claude then:
- inspected the live editor state (actors, Blueprint graphs, widget trees, logs);
- authored or rewired Blueprint logic;
- built UMG screens;
- compiled and saved;
- ran Play‑In‑Editor sessions to verify behaviour.

This turned a solo‑evening Blueprint workload into a ticket‑per‑hour cadence, while keeping the project fully Blueprint so non‑programmers can open and tune everything.

---

## 2. Technical Architecture & MCP Integration

### 2.1 Pipeline

```
 Designer ticket (natural language, ES/EN)
        │
        ▼
 Claude (Claude Code CLI)  ── reasoning, planning, verification
        │  MCP (JSON-RPC: list_toolsets → describe_toolset → call_tool)
        ▼
 Unreal MCP Server (UE 5.8 editor plugin: ModelContextProtocol + AllToolsets)
        │  ToolsetRegistry dispatch
        ▼
 Live UE5 Editor
   ├─ BlueprintTools      (read/write graph DSL, create/connect/retarget nodes, variables, functions)
   ├─ UMGToolSet          (widget trees, wrap/move/bind OnClicked, compile)
   ├─ ObjectTools         (CDO / instance property get/set/reset)
   ├─ SceneTools/ActorTools (find actors, transforms, components, level save)
   ├─ AssetTools/TextureTools (find, duplicate, save, texture size)
   ├─ EditorAppToolset    (Start/Stop PIE, capture viewport/editor/asset images)
   ├─ LogsToolset         (Output Log queries for runtime/compile errors)
   └─ ProgrammaticToolset (sandboxed Python batching many tool calls in one request)
```

**Key techniques used through MCP:**
- **Graph DSL round‑trips.** Claude reads a function as a Lisp‑like DSL (`read_graph_dsl`), rewrites it (`write_graph_dsl`), and then runs an *orphan‑node cleanup* pass. That pass walks exec and data reachability and deletes stale nodes.
- **Surgical splicing.** For large existing graphs (Tick, BeginPlay), Claude used `create_node` + `break_pins` + `connect_pins` to insert one call without rewriting the graph. This keeps designer‑authored logic untouched.
- **Node retargeting.** Some type IDs are ambiguous (e.g. `AudioComponent::FadeOut` vs `SynthComponent::FadeOut`). Claude fixed these with `retarget_node_class`.
- **Runtime verification.** Claude started PIE, teleported Betty with `set_actor_transform`, read live properties (`bPlayerInRange`, `CurrentInteractable`, HUD text), and queried the log for `Blueprint Runtime Error`.
- **Runtime‑created media.** The toolsets cannot create MediaPlayer assets, so the video screens construct a `MediaPlayer`, `MediaTexture` and `MediaSoundComponent` at runtime from Blueprint.

### 2.2 Systems inspected / generated via MCP

**Core** (`/Game/BarMaletin/Core`):

| Asset | Role | MCP work |
|---|---|---|
| `GM_Bar` | Match flow, show cycle, strikes, victory/game‑over | Exploration timers `FirstExploreSeconds`=10 / `ExploreSeconds`=45, `NumShows`=5, grace 7 s; English messages |
| `BP_BarGameState` | Money, code, closure pieces, clock pause | Inspected (`IsClockPaused`, flags) |
| `BP_BarPlayerController` | Modals, videos, input modes | Added `StartIntro`, `StartGameFlow` (retry skip), `ShowBribeScene`, `StartVictory`, `UpdateMusic`/`FondoFadeIn`/`FondoFadeOut`, `OpenPause`/`ResumeFromPause`, volume setters |
| `GM_Menu` | Main‑menu game mode | Created |

**Player & interactables:**

| Asset | Role | MCP work |
|---|---|---|
| `BP_Betty` | Sprite flipbook character | Paper‑flip turn animation (scale X, ease‑out‑back, 0.15 s); input path audit |
| `BP_Interactable` (13 instances) | Tables, cans, Pedro, briefcase, stairs | Stairs → explicit **E** interaction; stage‑gated prompt; Tick refresh |
| `BC_MesaDialogo` | Table dialogue component (29 lines, secret‑code script 99) | English localization; lines loaded from class defaults |

**UI** (`/Game/BarMaletin/UI`):

| Widget | Purpose |
|---|---|
| `Menu/WBP_MainMenu` | Splash → title → staggered buttons; Info/Credits popups |
| `WBP_Intro` | IntroBetty video → sealed letter → OPEN → open letter → START |
| `WBP_MainHUD` | Phase countdown synced to gameplay timers (`ClockSeconds`) |
| `WBP_CaseModal` | **Mechanical 4‑wheel combination lock** (rolling digits, shake/flash on error, zipper reward) |
| `WBP_CanModal`, `WBP_GlassModal` | Mouse‑capture fix, close buttons, catch‑window fixes |
| `WBP_DanceModal` | Arrow receptors, hit/miss feedback, music with pitch scaling |
| `WBP_BribeScene` | Static bribe cinematic |
| `WBP_GameOver` | Defeat image + SFX → darken → RESTART / BACK TO MENU |
| `WBP_Victory` | BettyEscapa video → THE END → EXIT |
| `WBP_PauseButton`, `WBP_PauseMenu` | Pause (click or ESC), options sliders |

**Levels & config:**
- `L_MainMenu` was created and set as `GameDefaultMap`.
- Interactable actors in `BaseDirectorsCut` were repositioned so their prompts stay on screen.
- `.mp4` media was moved into `Content/Movies` so it gets packaged.

**C++:** none. The project ships no `Source/` folder; all logic is Blueprint.

---

## 3. Representative Prompt & Execution Logs

> Condensed from the real development session (2026‑11‑15). Prompts were written in Spanish and are summarized in English here.

### Log 01 — Combination lock rework (Ticket 07A)
- **Step / Timestamp:** Mid‑session, ~15:10.
- **Intent / Prompt:** *"Replace the keypad with a mechanical 4‑wheel briefcase combination. ▲/▼ per wheel, short vertical roll animation, OPEN validates against `SecretCode`, WRONG COMBINATION with shake and red flash, BotonX to close. Don't touch rewards."*
- **MCP Tool Action & UE5 Result:**
  1. `UMGToolSet.GetWidgetDescription` and `BlueprintTools.read_graph_dsl` audited `WBP_CaseModal` and the existing `Submit` reward flow.
  2. `RemoveWidget` deleted the keypad grid. `AddWidget` built 4 clipped wheel windows (prev / current / next digits) and ▲/▼ buttons that reuse the dance‑arrow art.
  3. `write_graph_dsl` added `InitWheels`, `Spin`, `UpdateWheel`, `AnimStep` (ease‑out roll), `WrongFX` and `OnOpen`. `OnOpen` pre‑compares against `SecretCode` and then calls the **unchanged** `Submit`.
  4. `BindToEventProperty` wired 10 `OnClicked` events. An orphan cleanup removed 12 stale keypad events.
  5. `CompileWidgetBlueprint` and `save_assets` succeeded.
- **Result:** A tactile lock with a rolling‑digit animation, built in one pass. The reward and attempt logic stayed byte‑identical.

### Log 02 — "The briefcase won't open" live debugging
- **Step / Timestamp:** ~15:40–16:20.
- **Intent / Prompt:** *"URGENT: interacting with the briefcase does nothing. Audit the full flow and tell me exactly where it breaks."*
- **MCP Tool Action & UE5 Result:**
  1. `read_graph_dsl` traced the chain. `IA_Interact` (Started) → `OnInteractPressed` → `BP_Interactable.Interact` → switch case 3 → `OpenCaseModal` → `ShowModal` was intact.
  2. `SceneTools.find_actors` and `get_properties` checked `INT_Maletin`: type 3, radius 160, overlap collision OK.
  3. `StartPIE`, then `set_actor_transform` teleported Betty in and out of range. `bPlayerInRange=true` and `CurrentInteractable=INT_Maletin` confirmed the overlap works.
  4. `CaptureEditorImage` revealed the camera boom was clipped behind a sprite, so the world prompt was invisible to the player.
  5. Temporary `PrintString` probes were spliced via `create_node`/`connect_pins`, then removed. This proved where the input stopped.
- **Result:** Root cause isolated without guesswork: off‑screen prompts plus the bribe gate. The prompts were lowered and the bribe was temporarily set to $0 while testing. Interaction was restored, and the bribe is back at $200 in the final build.

### Log 03 — Show cycle redesign (exploration ↔ stage)
- **Step / Timestamp:** ~18:00.
- **Intent / Prompt:** *"New flow: 10 s free exploration → GO TO STAGE → Betty must walk to the stairs and press E → 15 s show → 45 s exploration, repeat. No teleport, no auto‑start."* Clarifying questions via `AskUserQuestion` settled 5 shows, a 7 s grace and removing the **B** shortcut.
- **MCP Tool Action & UE5 Result:**
  1. `add_variable` created `FirstExploreSeconds` and `ExploreSeconds`. `NewGame` inputs were rewired with `break_pins`/`connect_pins`.
  2. `set_properties` on the CDOs set `NumShows=5`, `DanceGracePeriod=7` and `WBP_DanceModal.Duration=15`.
  3. In `BP_Interactable`, the auto‑start on overlap was removed. `add_node_pin` added switch case **4** → `RequestEnterShow` on **E**. `RefreshVisuals` was rewritten so the stairs prompt only shows during the stage call.
  4. HUD literals (`/3` → `/5`, `AL ESCENARIO!` → `GO TO STAGE!`) were changed via `set_pin_value`.
- **Result:** A deterministic two‑clock loop. A follow‑up ticket synced the HUD to the real gameplay timers through a new `ClockSeconds` function.

### Log 04 — Dynamic music and Game Over menu
- **Step / Timestamp:** ~19:30.
- **Intent / Prompt:** *"Loop `fondo` during exploration, crossfade to `bailabetty` for the show and back, 0.5–1 s fades. Game Over: image + death SFX, wait for the audio, darken, then RESTART / BACK TO MENU."*
- **MCP Tool Action & UE5 Result:**
  1. Found that `WBP_DanceModal.DanceMusic` was empty and `fondo` was not looping. Fixed both through `set_properties`.
  2. `FadeOut` resolved to `SynthComponent`. `retarget_node_class` changed it to `AudioComponent`, and `StopMusic` now fades over 0.8 s.
  3. `UpdateMusic` was spliced at the head of the PlayerController Tick. A state machine fades `fondo` in or out based on dancing, video, game‑over and victory flags.
  4. `WBP_GameOver` waits for the death SFX via `IsPlaying`, darkens over 1.5 s and reveals the buttons. RESTART reloads with `?Retry=1`, which `StartGameFlow` reads to skip the intro.
  5. A PIE log query (`Blueprint Runtime Error`) exposed an auto‑destroyed audio component. It was fixed with `bAutoDestroy=false`.
- **Result:** Smooth audio transitions and a fast retry loop, with no gameplay timers touched.

### Log 05 — Full English localization pass
- **Step / Timestamp:** ~20:15.
- **Intent / Prompt:** *"Translate every player‑visible string to English. Don't rename anything internal."*
- **MCP Tool Action & UE5 Result:**
  1. A ProgrammaticToolset batch scan found every TextBlock and tooltip in 14 widgets, plus every literal String/Text/Name pin across 18 Blueprints.
  2. 87 literals were replaced by exact‑match `set_pin_value`, and 40+ widget texts via `set_properties`.
  3. The 29 dialogue lines were translated, with a Spanish backup in `Saved/`.
  4. Placed actors carried stale per‑instance copies of the dialogue, and PIE proved they still showed Spanish. A `GetClassDefaults` read was spliced into `BuildLines` so every table loads the localized data.
- **Result:** The game is fully playable in English, verified in PIE (`"Sailor: Did you see the ship that came in last night?…"`).

---

## 4. Human vs. AI Division of Labor

| Area | Pijama Studios (human) | Claude + MCP (AI‑orchestrated) |
|---|---|---|
| **Game design** | Core concept, double‑shift loop, bribe + 2‑piece win condition, show cadence (10/45 s, 5 shows, 7 s grace), difficulty targets | Translated tickets into Blueprint state changes; asked clarifying questions when rules conflicted |
| **Art direction & assets** | All 2D art (Betty frames, menu, letters, bribe/defeat/end images, buttons, BotonX), videos, music and SFX, fonts | Layout, scaling, crops (e.g. UV‑cropping the capsule button), fades and timing |
| **Narrative** | World, characters, Spanish dialogue scripts, tone | English translation of UI and dialogue (reviewed by the team) |
| **UX direction** | Every visual ticket: hierarchy, sizes, button feel, "theatrical vintage" style | UMG widget construction, hover/press states, animations via Tick timelines |
| **Gameplay wiring** | Acceptance criteria per ticket | Blueprint functions and events, input modes, pause system, retry flow, media players |
| **Tuning** | Final values (timers, sizes, volumes) after playtests | Exposed values as variables; applied requested numbers |
| **QA** | Hands‑on playtesting, reporting regressions ("the E prompt broke the briefcase", "button text too big") | PIE probes, log analysis, live property inspection, root‑cause reports, reverts on request |
| **Process rules** | Blueprints only, one ticket per task, compile + save, no rewriting working systems | Followed these rules; reported modified assets, variables and test steps after every ticket |

---

## 5. Notes for Judges

**No API keys, MCP server or Claude connection are needed to play.** MCP was used only at development time. The packaged build is a standard Blueprint UE5 game.

**Start:** launch the build. It opens on **`L_MainMenu`** (splash → *THAT'S ALL...* → PLAY). PLAY plays the IntroBetty video, then the letter: **OPEN**, then **START**.

**Controls**

| Action | Input |
|---|---|
| Move Betty | **WASD** |
| Interact (cans, Pedro, briefcase, stairs) | **E** |
| Clean a table (glass catch) | **L**, then **Space** or click to catch |
| Dance lanes | **Arrow keys** when notes hit the line |
| Pause | **ESC** or the on‑screen pause button (bottom‑left) |
| Toggle dialogue language (EN / "Orrin" gibberish) | **O** |
| Skip videos | Click |

**Fastest way to see the experimental mechanic (about 3 minutes)**
1. **Watch the two clocks.** The HUD counts down 10 s of free exploration. At 0, *GO TO STAGE!* appears with a 7 s countdown.
2. **Walk to the stairs.** Press **E** at the "[E] GO TO STAGE" prompt and play the 15 s show.
3. **Explore for 45 s.** After the show you have 45 s. Stand near tables to read subtitles. One table (random per run) has Valeria and Dante saying the **4‑digit code**.
4. **Get through Pedro.** Earn money with **cans** (E at the kitchen shelf) and **glasses** (L at a glowing table), then bribe Pedro with E once you have the $200. A short cinematic plays and he steps aside.
5. **Open the briefcase.** Press E at the briefcase, roll the four wheels to the code and press **OPEN**. The **zipper piece** reward appears.
6. **Finish.** Collect **2/2 zipper pieces**. Closing the last panel triggers the BettyEscapa ending → THE END → EXIT.
7. **Try losing.** Fail the lock 3 times, or miss two stage calls. The Game Over screen offers **RESTART**, which skips the intro and goes straight into the 10 s exploration.

**Known notes:** the on‑screen cursor stays visible during play so the pause button can be clicked. Some baked art (letters, end cards) may contain stylised text from the original artwork.

---

## 6. Prototyping Pipeline: Browser Validation Before Unreal MCP

Before executing any automated Blueprint wiring in Unreal Engine, **all core gameplay mechanics were prototyped, stressed, and validated in standalone HTML5 / JavaScript sandboxes**:

- **Why browser-first prototyping?** Testing game feel, timing thresholds, and physics parameters in lightweight HTML files allowed instant iteration in milliseconds, avoiding Unreal Engine compile/relaunch bottlenecks during early design exploration.
- **Validated math & parameters:** Exact values such as the falling-glass gravity (g = 408 px/s²), the 4-lane one-hand rhythm spacing, the 2.0-second circular lid-opening gesture using `tapa.png`, and the dual-clock tension were tuned in the browser first.
- **Seamless handoff to MCP:** Once a mechanic felt right in HTML, its exact variables, state rules, and existing Unreal asset names (`BP_Can`, `M_Can`, `CanArtTextures`, `tapa.png`) were locked into a living Markdown specification (`GDD_Mecanicas_Unreal.md` & `Logica_Blueprints_Unreal.txt`). Feeding those structured specs to Claude via MCP enabled surgical, phase-by-phase Blueprint generation without rebuilding working art assets.

---

## 7. Creative Exploration & Discarded Prompts (Design Log)

In line with open-process jam documentation, the team also records early creative prompts and mechanics that were explored during ideation and **actively cut after playtesting**:

- **Coat-search interaction:** A mechanic where the player searched hanging coats around the bar for clues.  
  *Reason cut:* HTML playtesting showed it slowed down the pacing and diluted the urgency of the Director's stage calls.
- **Darts mini-game:** A standalone aiming/throwing task inside the speakeasy.  
  *Reason cut:* Fun for ten seconds in isolation, but felt like dead weight during a tense, timed run.
- **Our starting point: the multidisciplinary ideation prompt:** Before the official theme was announced, we tested a prompt with a couple of sample themes. We briefed Claude to act as a multidisciplinary creative team (Creative Director, Game Designer, Narrative Designer, Technical Game Designer, plus mobile and game-jam specialists) and walk us through phases: theme exploration, mechanics exploration, creative experiments and feasibility analysis. Once we saw the livestream, we ran it again with the official theme, *Director's Cut*.  
  *Why it didn't work for us:* We ended up discarding everything the AI proposed.  
  *Why it still mattered:* It got us ideating together and showed us what we did not want. In the end, the whole team decided the game in a meeting, and the concept we built (a 1930s rubber-hose speakeasy, an invented language, *Orrin*, as an audio deduction puzzle, and the stage-vs-undercover dual-clock tension of *THAT'S ALL...*) came from our own ideas, not from the AI's.

---
*Pijama Studios · THAT'S ALL... · Epic AI MegaJam 2026*
