# Card Flip Word Game — Client Design Document

**Course**: CS 220 Applied Data Structures
**Component**: Java Swing Desktop Client + macOS App Packaging
**Grade Weight**: 12%

---

## 1. Overview

The client is a Java Swing desktop application that provides the graphical interface for Card
Flip Word Game. It communicates with the Spring Boot server exclusively through the REST API
defined in `server-design.md` §7.

**Key properties**:

- **Server-silent**: the client never holds secret state (the phrase or card order).
- **Polling**: it polls `GET /game/{id}/state` every 2 seconds to detect turn changes and
  round-end events (no WebSocket required for this scope).
- **Self-contained app**: client and server are bundled with `jpackage` into a single macOS
  `.app`. The launcher starts the embedded server, waits until it is ready, then opens the
  Swing window.
- **Single-player-vs-bots flow**: the human player always takes the first turn; opponents are
  bots. There is no Game ID display and no "join existing game" screen.
- **Screen sequence on first launch**: loading screen → rules/how-to-play screen (shown once,
  right after loading) → player-info entry screen → game screen.
- **Animated cards**: cards are dealt with a one-time shuffle animation at the start of each of
  the human's turns, and a flipped card flies to the centre of the screen and turns over to
  reveal its reward before play continues.

---

## 2. Technology Stack

| Layer | Technology | Reason |
|-------|-----------|--------|
| Language | Java 21 | LTS, same JDK as server |
| GUI Framework | Java Swing | Standard, bundled in JDK, no external dependency |
| HTTP Client | `java.net.http.HttpClient` (JDK 11+) | No extra dependency |
| JSON | Jackson Databind | Same library as the server side |
| App Packaging | `jpackage` (JDK 14+) | Native macOS `.app` with bundled JRE |

---

## 2A. Visual Style — Synthwave (applies to every screen)

All four screens — loading, rules, player-info, and the game screen — share one consistent
synthwave look, so the app feels like a single product rather than separate screens. The doc
fixes the **intent**; the agent picks concrete colours, sizes, and the bundled font.

- **Background**: dark (deep navy / purple), optionally with a subtle gradient and faint
  perspective-grid or star motifs to echo the loading screen.
- **Accents**: neon colours (e.g. pink, cyan, gold) for borders, highlights, and the current
  selection.
- **Fonts**: a **pixel-style font for large titles only** (bundled `.ttf`, loaded via
  `Font.createFont()`), and a **clean monospace font for all body text, inputs, scores, the
  clue, and buttons** — pixel fonts are reserved for headings because they are hard to read in
  long or dense text.
- **Controls**: inputs and buttons are **custom-styled**, not left as the OS default look —
  dark fills, neon borders, light monospace text, with a clear focus/hover state.

Wherever a later section says "synthwave style", it means this shared definition. A plain
`JOptionPane` confirmation (e.g. the Exit dialog, §8.11) is the one allowed exception and does
not need synthwave styling.

---

## 3. Package Structure

```
com.cs220.cardflip/
└── AppLauncher.java                 (main() — starts server thread, then GUI)

com.cs220.cardflip.client/
├── ClientApp.java                   (builds and shows the JFrame)
├── network/
│   └── ApiClient.java               (HTTP wrapper; all server calls go here)
├── state/
│   └── ClientState.java             (local game state cache, updated on every poll)
└── ui/
    ├── SplashPanel.java             (loading screen while the embedded server starts; shown once)
    ├── RulesPanel.java              (how-to-play / rules screen, shown once after loading)
    ├── MainMenuPanel.java           (player-info entry: name / rounds / bots)
    ├── GamePanel.java               (main game screen; hosts sub-panels, overlays, animations; serializes turn/bot animations)
    ├── InfoPanel.java               (top-left: player / round / round score / total score, all players)
    ├── StatusMessagePanel.java      (top-right: status message)
    ├── CardPanel.java               (12 cards in 2 rows; shuffle + flip animations)
    ├── CardView.java                (a single card component: face-down pattern or face-up reward)
    ├── PhraseDisplay.java           (centered topic, clue/context, horizontal letter boxes)
    ├── CenterOverlay.java           (centered transient notices: flying card, Lose Turn, Time's up, bot steps, round result)
    ├── LetterInputBar.java          (single-letter input with a 10-second countdown)
    ├── SolveDialog.java             (modal solve input with a 15-second countdown)
    ├── GameOverDialog.java          (final scores + Yes / No / Leaderboard)
    └── LeaderboardDialog.java       (top-10 modal popup)
```

There is no buy-vowel control, dialog, or `ApiClient` method anywhere in the client.

---

## 4. Application Launch Sequence

```
AppLauncher.main()
  │
  ├─ [Thread: serverThread]  SpringApplication.run(CardFlipServerApplication.class)
  │                          → Tomcat starts on :8080
  │
  ├─ waitForServer()         polls GET /api/leaderboard until 200 OK (max 30 s)
  │
  └─ SwingUtilities.invokeLater(ClientApp::launch)
       │
       ├─ SplashPanel       loading screen, shown once while the server starts
       ├─ RulesPanel        how-to-play / rules, shown once after loading completes
       ├─ MainMenuPanel     player-info entry (name / rounds / bots)
       └─ GamePanel         the game itself
```

The card-based `JFrame` uses a `CardLayout` (Swing's screen switcher) to move between these
panels. The loading screen and the rules screen are each shown **exactly once per app launch**:
after the player leaves the rules screen, later navigation (e.g. returning to the menu after a
game) goes to `MainMenuPanel`, never back to `SplashPanel` or `RulesPanel`.

Both the embedded server and the GUI run in the same JVM process. The REST API still separates
them architecturally — the GUI never accesses server internals directly.

---

## 5. Screens Before the Game

### 5.1 SplashPanel (loading)

> **MANDATORY — the code must implement this exactly.** During testing the loading screen only
> flashed for a split second before jumping to the rules screen. It must instead stay visible for
> a **guaranteed minimum of 5 seconds** so the player actually sees the synthwave intro.

Shown once at launch as a fixed intro. See §5.4 for its synthwave visual design and timing. It
**must stay on screen for at least 5 seconds** and advances to the rules screen only once
**both** conditions hold: at least 5 seconds have elapsed **and** the embedded server is ready
(`waitForServer()` returned 200). If the server is ready before 5 s, the screen still shows the
full 5 s; if the server is not ready after 5 s, the screen waits longer until it is. The 5-second
minimum must not be skipped even when the server starts almost instantly.

### 5.2 RulesPanel (how to play)

Shown once, immediately after loading, then never again during the session. A single
**Continue** button (bottom-centre) advances to the player-info screen.

**Visual style** — synthwave (see §2A): dark background, neon accents, a pixel-style font for
the big title "HOW TO PLAY", and **monospace** for the body text.

**Readability rules (MANDATORY).** During testing the card text looked broken: tiny font and
phrases chopped across several lines mid-sentence ("Reveal the / hidden keyword / from its
clue"), so the bullets read as disjointed fragments. The rewrite must fix this:

- Body text is **monospace, at least 16–18px** (clearly larger than before), with generous line
  spacing so the cards feel airy, not cramped.
- **Each bullet is one complete, self-contained idea** — a full short sentence, not a fragment.
- A bullet is **never broken mid-phrase**. If a bullet is long enough to need two lines, it must
  **word-wrap on whole words** (and ideally wrap only at natural points), never split a word or a
  phrase like "hidden keyword" across lines. Use a wrapping label (e.g. an HTML `<html>`-wrapped
  `JLabel`, or `JTextArea` with `setLineWrap(true)` + `setWrapStyleWord(true)`), sized to the
  card width, so wrapping is by word, not by character.
- The four cards are equal width, laid out horizontally to fill the screen (e.g.
  `GridLayout(1, 4)` with padding). Each has a neon-outlined border and a heading.

**Layout:**

```
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│       GOAL       │ │   ON YOUR TURN   │ │     SCORING      │ │     WINNING      │
│                  │ │                  │ │                  │ │                  │
│ • Reveal the     │ │ • Flip a card on │ │ • Your round     │ │ • The player     │
│   hidden keyword │ │   your turn.     │ │   score builds   │ │   with the       │
│   from its clue. │ │ • A points card  │ │   up during the  │ │   highest total  │
│ • The highest    │ │   lets you guess │ │   current round. │ │   score after    │
│   total score    │ │   one letter in  │ │ • At round end   │ │   the last round │
│   across all     │ │   10 seconds.    │ │   it is added to │ │   wins the game. │
│   rounds wins.   │ │ • Any letter     │ │   your total,    │ │ • Win a round by │
│                  │ │   counts.        │ │   then resets to │ │   solving it or  │
│                  │ │ • Double, Halve  │ │   0.             │ │   revealing the  │
│                  │ │   and Lose Turn  │ │ • Your total     │ │   last letter.   │
│                  │ │   change your    │ │   carries across │ │ • Top scores are │
│                  │ │   turn.          │ │   every round.   │ │   saved to the   │
│                  │ │ • Solve in 15s   │ │                  │ │   leaderboard.   │
│                  │ │   for +2500.     │ │                  │ │                  │
└──────────────────┘ └──────────────────┘ └──────────────────┘ └──────────────────┘
                                  [ Continue ]
```

**Card content (full sentences — this is the exact intended wording):**

- **GOAL**
  - Reveal the hidden keyword from its clue.
  - The highest total score across all rounds wins.
- **ON YOUR TURN**
  - Flip a card on your turn.
  - A points card lets you guess one letter in 10 seconds.
  - Any letter counts — vowels and consonants.
  - Double, Halve, and Lose Turn change your turn.
  - Solve the puzzle in 15 seconds for a +2500 bonus.
- **SCORING**
  - Your round score builds up during the current round.
  - At the end of a round it is added to your total, then resets to 0.
  - Your total score carries across every round.
- **WINNING**
  - The player with the highest total after the last round wins.
  - Win a round by solving it, or by revealing the last letter.
  - Top scores are saved to the leaderboard.

The ASCII above only shows wrapping; the live screen uses the full sentences listed under "Card
content", rendered at the 16–18px monospace size with word-wrap so nothing is chopped
mid-phrase.

### 5.3 MainMenuPanel (player-info entry)

The player-info screen — see §9 for its layout. This is the screen the app returns to after a
game when the player chooses not to replay, and the screen the in-game **Exit** button returns
to (§8.11).

### 5.4 Loading screen visual design (synthwave)

`SplashPanel` is drawn entirely in Swing (custom `paintComponent`) in the spirit of a retro
synthwave loading screen — **no external or copyrighted image is used**, since the deliverable
is a class submission. The scene, top to bottom, is composed of:

- a vertical **gradient background** from dark purple to near-black;
- scattered **stars** that twinkle (their brightness oscillates);
- a neon **perspective grid** on the lower half — horizontal lines converging toward a vanishing
  point, with the grid lines scrolling toward the viewer to suggest motion;
- near the top, the **game title "CARD FLIP WORD GAME"** in the pixel-style title font, which
  **pulses with a smooth fade in / fade out** (its opacity eases up and down in a loop) so it
  blinks gently rather than flicking on and off;
- below the title, the word **"LOADING"** and a **segmented progress bar** of coloured blocks
  with a red → orange → yellow gradient.

The progress bar is **indeterminate**: it animates in a continuous loop (blocks filling and
cycling) purely for visual effect; it does not track real server progress. The screen stays for
a **minimum of 5 seconds** and advances only when both 5 s have elapsed and the server is ready
(§5.1). The title fade, grid scroll, and star twinkle are all driven by a single ~60 fps Swing
`Timer` (consistent with §8.4's animation guidance) and stop when the screen is dismissed.

---

## 6. ClientState — Local Cache

`ClientState` is a plain Java object (no Swing dependency) caching the most recent server
state. It is updated after every action response and every poll response.

| Field | Type | Source |
|-------|------|--------|
| `gameId` | `String` | set on create (kept internally; never shown in the UI) |
| `myPlayerId` | `String` | set on create |
| `status` | `String` | `state.status` |
| `currentPlayerId` | `String` | `state.currentPlayer` |
| `turnState` | `String` | `state.turnState` |
| `hiddenPhrase` | `String` | `state.hiddenPhrase` |
| `topic` | `String` | `state.topic` (category shown to the player) |
| `context` | `String` | `state.context` (clue/question shown to the player) |
| `guessedLetters` | `List<Character>` | `state.guessedLetters` |
| `players` | `List<PlayerInfo>` | `state.players` (each has `roundScore` **and** `totalScore`) |
| `pendingAction` | `String` | last flip/guess response `nextAction` |
| `lastRoundResult` | `RoundResultInfo` | `state.roundResult` (non-null only at round end) |
| `botTurnLog` | `BotTurnLog` | `state.botTurnLog` (non-null after bots acted; replayed then cleared) |

Convenience: `isMyTurn()` → `myPlayerId.equals(currentPlayerId)`.

The scoreboard reads `totalScore` for the cumulative column and `roundScore` for the per-round
column, for **all** players including bots. The client always trusts the server's `totalScore`
as the authoritative cumulative value.

---

## 7. ApiClient — HTTP Layer

All HTTP calls go through `ApiClient` (`java.net.http.HttpClient`).

```
ApiClient
 ├─ createGame(hostName, numRounds, numBots)  → POST /game/create     (numBots ≥ 1)
 ├─ startGame(gameId, hostPlayerId)           → POST /game/{id}/start
 ├─ getState(gameId)                          → GET  /game/{id}/state
 ├─ flipCard(gameId, playerId, cardIndex)     → POST /game/{id}/flip
 ├─ guessLetter(gameId, playerId, letter)     → POST /game/{id}/guess
 ├─ solvePuzzle(gameId, playerId, phrase)     → POST /game/{id}/solve
 ├─ pass(gameId, playerId)                    → POST /game/{id}/pass
 └─ getLeaderboard()                          → GET  /leaderboard
```

`guessLetter` sends a single letter — any letter A–Z, vowel or consonant. The whole-answer
move is `solvePuzzle`. `pass` is called by the client whenever a countdown expires — both the
10-second letter-input timer (§8.6) and the 15-second solve timer (§8.7) — so the server
advances the turn. There is no buy-vowel method and no join method: a game is created fully
populated with bots and started immediately, with the human first.

Non-2xx responses throw `RuntimeException("Server error N: message")`; the Swing caller catches
these and shows them in the status message area, and never lets them crash the Event Dispatch
Thread.

---

## 8. UI Layout (Game Screen)

`GamePanel` is laid out as a top bar of fixed height plus a flexible body that fills the rest
of the window. The window runs maximized / resizable, so the body must stretch to fill all
available height — no large empty gap at the bottom.

```
┌───────────────────────────────────────────────────────────────────────────┐
│ InfoPanel (top-left, fixed height)              StatusMessagePanel (top-right)│
│  Player    Round  RoundScore  TotalScore        ★ Your turn — flip a card!   │
│  Alice ▶    2/3      500         3000                                         │
│  Bot 1      2/3        0         1500                                         │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                              CardPanel  (grows)                             │
│            [ ][ ][ ][ ][ ][ ]      ← row 1, cards 0–5                        │
│            [ ][ ][ ][ ][ ][ ]      ← row 2, cards 6–11                       │
│                                                                             │
├───────────────────────────────────────────────────────────────────────────┤
│                            PhraseDisplay  (grows)                           │
│                              Topic: Science                                 │
│           Clue: The force that pulls objects toward the earth.              │
│                    [ G ][ _ ][ _ ][ _ ][ _ ][ _ ][ _ ]                      │
├───────────────────────────────────────────────────────────────────────────┤
│           [ Solve a Puzzle ]     [ Leaderboard ]     [ Exit ]               │
└───────────────────────────────────────────────────────────────────────────┘
            (CenterOverlay paints on top of all of this when active)
```

The topic, clue, and letter-box row in PhraseDisplay are **horizontally centred** (see §8.9).
The bottom row now has three buttons: Solve a Puzzle, Leaderboard, and Exit (§8.11).

### 8.1 Filling the window height

`GamePanel` uses a `GridBagLayout` (or `BorderLayout` with the body in `CENTER`). The top bar
(InfoPanel + StatusMessagePanel) keeps a fixed preferred height. The two body regions —
**CardPanel** and **PhraseDisplay** — are given vertical grow weight so they expand to absorb
all remaining height as the window resizes:

- top bar: `weighty = 0` (fixed height).
- CardPanel: `weighty ≈ 0.65`, `fill = BOTH` — the cards region takes the larger share of the
  body so the cards are noticeably **bigger**.
- PhraseDisplay: `weighty ≈ 0.35`, `fill = BOTH` — its contents (topic, clue, letter boxes) are
  grouped as one compact cluster (§8.9).
- bottom action row: `weighty = 0` (fixed height).

Card size and letter-box size are computed from the region's current size in
`paintComponent` / `doLayout`, not hard-coded in pixels, so both regions stay centred and fill
the space at any window dimension. This removes the leftover empty band at the bottom.

Within CardPanel, the **gap between cards is small** (a few pixels) so the 12 cards read as one
tidy deck of large cards rather than small cards spread far apart; the cards grow with the
region while the inter-card spacing stays tight.

### 8.2 InfoPanel (top-left)

Pinned to the top-left, showing **one row per player** (the human and every bot) with **four
columns**: **Player**, **Round** (current round / total), **Round Score** (`roundScore`), and
**Total Score** (`totalScore`). The current player's row is marked (e.g. with ▶).

Update timing (driven by server state, shown as plain static numbers — no count-up animation):
- **Round Score** updates after each correct letter guess within the round.
- At the **end of each round**, each player's **Total Score** increases by their round score
  and their **Round Score** resets to 0. This banking is performed server-side (see
  `server-design.md` §6); the client simply displays the new values from the next state.

### 8.3 StatusMessagePanel (top-right)

Pinned to the top-right; the single place transient text feedback appears — prompts such as
"Your turn — flip a card!", "Guess a letter!", server error messages (in red), or
"Connection error, retrying…".

### 8.4 CardPanel and CardView — design, shuffle, and flip

**Card design (`CardView`)**. Each of the 12 cards is a custom component, not a plain button:
- **Face-down**: a uniform decorative back pattern (in the style of the back of a playing
  card) — the same pattern on all 12 cards. No "?" text.
- **Face-up**: a single shared colour scheme for all card types (types are **not**
  colour-coded); the card shows only its reward text/number, e.g. "1000", "Double", "Halve",
  "Lose Turn".

**Shuffle animation (start of the human's turn)**. When it becomes the human's turn, the 12
cards play a one-time shuffle animation — their positions are swapped around and settle back
into the 2×6 grid, all face-down — then become clickable. The shuffle plays **once per turn**:
if the player guesses correctly and earns another flip within the **same** turn, the cards are
already face-down and clickable, and clicking flips immediately with no re-shuffle.

The shuffle must look smooth, not stiff or jerky. To achieve that:
- it is driven by a **~60 fps** `javax.swing.Timer` (≈16 ms interval), not a coarse step timer,
  so motion is continuous rather than jumping in visible nudges;
- card positions are **interpolated along a path** between their start and end slots (cards
  glide, they do not teleport);
- the interpolation uses an **ease-in-out** curve (accelerate then decelerate) rather than
  constant/linear speed, so the motion feels natural;
- the whole shuffle lasts about **1 second**, then the cards settle and lock into the grid.

All repainting happens on the Event Dispatch Thread; the timer only updates positions and calls
`repaint()`, which avoids stutter.

**Flip animation (on click)**. When the player clicks a face-down card:
1. The client immediately calls `ApiClient.flipCard(...)` and receives the real card the server
   chose (the animation only presents a result that already exists).
2. The clicked card flies to the centre of the screen (via `CenterOverlay`) and turns over to
   reveal its reward.
3. The revealed card is held in the centre for **2 seconds** so the player can read it.
4. The centre card then fades out / disappears, and the next step begins depending on the card
   type (§8.5).

All cards are disabled when it is not the human's turn. Bot turns are not silently applied:
they are replayed step by step from the server's `BotTurnLog` (§8.8) so the human can clearly
see when and what each bot played.

### 8.5 After the flip — by card type

In all cases below, the client **serializes** presentation: the next player's turn (human or
bot) is not shown until the current animation sequence has fully finished. `GamePanel` holds an
"animating" lock; while it is set, polling results are queued, not applied, so a notice can
never be cut short by the next turn appearing underneath it.

- **Points card (100–2000)**: after the centre card fades, the **LetterInputBar** appears
  (§8.6) for a single-letter guess.
- **Double / Halve**: after the centre card fades, the round-score change is shown, then a
  centred "Double!" / "Halve!" notice is held for ~1.5 s, and only **then** does the turn end
  and the next player begin.
- **Lose Turn**: the flip response carries `nextAction == "SHOW_LOSE_TURN_THEN_ADVANCE"`; after
  the centre card fades, `CenterOverlay` shows "Lose Turn!" in the centre for ~1.5 s, and only
  **then** the UI moves to the next player (the server already advanced the turn).

Because the turn only advances after these notices finish, the bug where a bot started playing
while the player's "Lose Turn / Halve / Double" notice was still on screen no longer occurs.

### 8.6 LetterInputBar — 10-second letter guess

After a points card is revealed, the player guesses one letter under a 10-second limit:
1. A `LetterInputBar` appears (single-character field) with a visible countdown starting at
   **10**, driven by a `javax.swing.Timer` (1000 ms).
2. The letter may be **any** letter A–Z — vowels are allowed as well as consonants.
3. **If the player submits** before 0, the client calls `ApiClient.guessLetter(...)`:
   - correct → round score increases (server response), and the player flips again (no
     re-shuffle, §8.4);
   - wrong → the turn ends and passes to the next player.
4. **If the countdown reaches 0** with no submission, the input closes, `CenterOverlay` shows
   "Time's up!" in the centre for ~1–1.5 s, and the client then calls `ApiClient.pass(...)` so
   the turn passes to the next player.

This mirrors the solve timer's mechanism: the countdown lives in the client, and an expiry
triggers a `pass` to the server; the server itself runs no wall-clock timer.

### 8.7 SolveDialog — 15-second solve window

When the player clicks **Solve a Puzzle**:
1. A modal `SolveDialog` opens with a text field and a visible countdown starting at **15**,
   driven by a `javax.swing.Timer` (1000 ms).
2. **If the player submits** before 0, the client calls `ApiClient.solvePuzzle(...)`:
   - correct → the server returns "Round won!" with a 2500 bonus, then the round-result banner
     follows (§8.9);
   - wrong → the server returns "Round lost!", the player is eliminated, and the turn passes
     automatically.
3. **If the countdown reaches 0** with no submission, the dialog auto-closes and the client
   calls `ApiClient.pass(...)`, so the turn passes to the next player.

The text field accepts spaces; the entered text is sent verbatim and the server normalises
spacing and case. **Solve is available to the human player only — bots never solve (MANDATORY,
`server-design.md` §11).** The human can also win a round without solving: if a correct letter
guess reveals the last blank, the human wins the round and gets the same 2500 bonus
(server-design.md §6.3), exactly as a bot would.

### 8.8 Bot turn replay (from BotTurnLog)

> **MANDATORY — the code must implement this exactly.** During testing the bot's turn was
> invisible: from round 2 on, the player only saw letter boxes fill in and the turn jump back to
> them, with no pop-ups and no sense of the bot playing. That is the bug this section fixes. The
> client must **never** snap straight to the bot's final board; it must replay every recorded
> step, visibly, with the animation lock held.

When polling returns a non-null `botTurnLog` (`server-design.md` §7.4), the client replays the
bot's recorded steps one at a time so the player clearly sees what each bot did. While the replay
runs, the "animating" lock from §8.5 is held, so the human's controls stay disabled, polling
results are queued (not applied), and nothing changes underneath the notices.

For each step in `botTurnLog.steps`, in order:
1. Show a status/notice "**{botName} is playing…**" so it is obvious whose move this is.
2. **FLIP** step: animate a card flying to the centre and turning over to show the bot's drawn
   card — the same flying-card reveal used for the human (§8.4) — and hold it briefly so the
   player can read the card (e.g. "1000", "Double", "Lose Turn").
3. **GUESS_LETTER** step — **order matters (MANDATORY)**: first **show the result notice**
   (e.g. "Bot 1 guessed R — correct!" or "Bot 1 guessed Z — wrong"), **then** fill the revealed
   letter into the phrase boxes and update the bot's round score in `InfoPanel`. The notice must
   appear *before* the boxes fill, not at the same time. If this guess reveals the last letter,
   the notice also shows the round win and the +2500 bonus.
4. **TURN_ENDED** step: show the matching notice ("Lose Turn!", "Double!", "Halve!", or — when
   the bot won the round by revealing the last letter — the round-won result) consistent with
   §8.5.

Bots never produce a `SOLVE` step (bots don't solve, `server-design.md` §11); a bot wins a round
only through a `GUESS_LETTER` step that completes the phrase.

Each step is held on screen for about **1.5 seconds** before the next step plays (the FLIP card
reveal and the "result notice → fill boxes" sequence both happen within that step's window), so
the pacing is readable rather than instantaneous. After the last step has been replayed, the lock
is released and play resumes (typically the human's next turn, which begins with the shuffle
animation of §8.4). If several bots acted in a row, their steps appear in the same log and are
replayed continuously in order, each clearly labelled with its bot's name.

### 8.9 PhraseDisplay (clue + letter boxes)

A custom `JPanel` (`paintComponent`) showing, top to bottom: the **topic** (`state.topic`), the
**clue/context** line (`state.context`), and the answer as a horizontal row of letter boxes.
All three are **horizontally centred** within the region, and they are **grouped together as one
compact cluster** — the topic, the clue, and the letter boxes sit close to each other with only
small vertical gaps, rather than being spread far apart. The cluster as a whole is centred in
the region. Each box is a rounded rectangle: unrevealed = dark navy with no character; revealed
= gold box with the letter. Box size scales with the region (§8.1). The topic plus the clue
together give the player enough to reason toward the keyword.

### 8.10 CenterOverlay — centered transient notices

`CenterOverlay` is a translucent component painted over `GamePanel`, used for: the **flying
card reveal** (§8.4), the **"Lose Turn!" / "Double!" / "Halve!"** notices (§8.5), the
**"Time's up!"** notice (§8.6), the **bot step notices** (§8.8), and the **round-result
banner** below. Each notice shows for a fixed duration, then fires a callback.

**Round result**: when a `/state` poll returns a non-null `roundResult`, the overlay shows a
centred banner naming the winner and the answer:

```
        ─────────────────────────────
            ROUND 1 COMPLETE
          Alice won the round!
        The answer was: GRAVITY
        ─────────────────────────────
              (continues in 3 s…)
```

If `winnerName` is null, the banner reads "No one solved it — the answer was GRAVITY." After
the banner, if `hasMoreRounds` is true the UI refreshes into the next round; otherwise the
game-over flow runs (§8.12).

### 8.11 Exit button (return to menu)

The bottom action row of the game screen has three buttons: **Solve a Puzzle**, **Leaderboard**,
and **Exit** (§8 layout). Clicking **Exit** opens a standard Swing confirmation dialog:

```
JOptionPane.showConfirmDialog(
    "Are you sure you want to exit? Your current game will be lost.",
    options = Yes / No)
```

- **Yes** → abandon the current game and return to the **player-info screen** (`MainMenuPanel`),
  the same destination as **No** on the game-over dialog. The poll timer is stopped and any
  in-progress animation lock is cleared.
- **No** → dismiss the dialog and stay in the game with no change.

A plain `JOptionPane` is used (it does not need synthwave styling); it is quick and unambiguous.

### 8.12 GameOverDialog — end of game

When a `/state` poll shows `status == "FINISHED"`:
1. The poll timer is stopped.
2. `GameOverDialog` shows final standings (each player's cumulative `totalScore`, highest
   first) and "Game over!".
3. Three buttons are rendered in a `JPanel` with `FlowLayout`, sized with `pack()` so all are
   visible and clickable:
   - **Yes** → start a **new game immediately** reusing the **same settings as the game just
     played** (same player name, number of rounds, number of bots). The client calls
     `createGame(...)` + `startGame(...)` with those saved settings and goes straight into the
     game screen — it does **not** return to the menu or the rules screen.
   - **No** → return to the **player-info screen** (`MainMenuPanel`).
   - **Leaderboard** → open `LeaderboardDialog` as a modal popup over the game-over dialog;
     closing it returns to the game-over dialog with all three buttons still available.
4. The leaderboard has already been updated server-side at game end (`server-design.md` §7.5).

To support **Yes**, the client remembers the last game's settings (name, rounds, bots) in a
small holder so it can recreate an identical game without prompting.

### 8.13 Polling

A `javax.swing.Timer` fires every 2000 ms and calls `ApiClient.getState()` on a `SwingWorker`
background thread. The result updates `ClientState`, then `refreshUI()` updates all sub-panels
on the Event Dispatch Thread. Polling also drives the centre notices for actions taken by bots.

---

## 9. Player-Info Screen (MainMenuPanel)

`MainMenuPanel` is the player-info entry screen — no Game ID field and no "join existing game"
option. It uses the shared **synthwave style** (§2A) so it matches the loading and rules screens.

**Layout** — a single content block **centred** on the dark background, comfortably **enlarged**
(larger fields, larger text, generous spacing) rather than the small default-widget cluster:

```
┌───────────────────────────────────────────────────┐
│                                                     │
│              CARD FLIP WORD GAME                    │  ← pixel-font title, neon
│                                                     │
│            Your name        [ Alice            ]    │  ← styled text field
│            Number of rounds [   3   ▾ ]             │  ← styled dropdown
│            Number of bots   [   1   ▾ ]   (1–3)     │  ← styled dropdown
│                                                     │
│                 [  START GAME  ]                    │  ← styled neon button
│                 [  LEADERBOARD ]                    │  ← styled neon button
│                                                     │
└───────────────────────────────────────────────────┘
```

**Styling (synthwave, §2A)** — the controls are custom-styled, not the OS default look:
- The **text field** and the two **dropdowns** (rounds, bots) have a dark fill, a neon border,
  and light **monospace** text; they are sized noticeably larger than the default widgets, with
  a clear focus/selection highlight. The native combo-box arrow is restyled or replaced so it
  does not show the blue OS default seen before.
- **Start Game** and **Leaderboard** are custom buttons (dark / gradient fill, neon border or
  neon text), consistent with buttons elsewhere — not the default white buttons.
- The **title** uses the pixel title font; all field labels and control text use monospace.

The bots selector offers 1, 2, or 3 (at least one bot is required). "Start Game" calls
`createGame(name, rounds, bots)` then immediately `startGame(...)`, going straight into
`GamePanel`, and **saves these settings** so the game-over "Yes" action (§8.12) can recreate an
identical game. The host is the only human and always takes the first turn; all opponents are
bots.

---

## 10. User Flow

```
App launches
  └─ SplashPanel: "Starting server…"            (once, while server boots)
       └─ [server ready]
            └─ RulesPanel: how to play           (once, after loading)
                 └─ [Continue]
                      └─ MainMenuPanel: name / rounds / bots (≥ 1)
                           └─ Start Game  (settings saved)
                                └─ GamePanel  (human plays first)
                                     │  [your turn]
                                     │   ├─ shuffle animation (once at turn start)
                                     │   ├─ click card → fly to centre → flip → hold 2 s → fade
                                     │   │     ├─ points → guess a letter (10 s)
                                     │   │     │     ├─ correct → round score +, flip again
                                     │   │     │     ├─ wrong   → turn ends
                                     │   │     │     └─ timeout → "Time's up!" → pass → turn ends
                                     │   │     ├─ Double / Halve → score change → turn ends
                                     │   │     └─ Lose Turn → "Lose Turn!" → next player
                                     │   ├─ Solve a Puzzle (15 s)
                                     │   │     ├─ correct → "Round won!" → round-result banner
                                     │   │     └─ wrong / timeout → eliminated, turn passes
                                     │   └─ Exit → confirm dialog → MainMenuPanel
                                     ├─ [not your turn] poll → replay BotTurnLog step by step
                                     │     ("Bot is playing…" → fly+flip → result), ~1.5 s each
                                     └─ [round ends] round-result banner → next round or game over
                                          └─ [game over] GameOverDialog (Yes / No / Leaderboard)
                                               ├─ Yes        → new game, same settings → GamePanel
                                               ├─ No         → MainMenuPanel (player-info screen)
                                               └─ Leaderboard→ modal popup → back to GameOverDialog
```

---

## 11. Error Handling in the UI

| Scenario | UI Behaviour |
|----------|-------------|
| Server returns 4xx | StatusMessagePanel shows a red error message |
| Network timeout | StatusMessagePanel shows "Connection error, retrying…"; polling continues |
| Invalid letter input | Field rejects non-letters; re-prompts within the 10-second window |
| Letter guess times out | `CenterOverlay` "Time's up!" → `pass` → turn passes |
| Solve wrong / timed out | "Round lost!" message; the turn passes automatically |
| Player eliminated | `CenterOverlay` notice; flip/solve greyed out until next round |
| Round ends | `CenterOverlay` round-result banner |
| Exit clicked | `JOptionPane` confirm; Yes → return to player-info screen, No → stay |
| Game finished | Poll timer stopped; `GameOverDialog` with working Yes / No / Leaderboard |
| Unexpected server error | The server returns clean JSON; the UI shows the message and keeps polling instead of freezing |

---

## 12. macOS App Packaging

### 12.1 Build

```bash
# 1. Build the fat JAR (Spring Boot server + Swing client)
mvn package -DskipTests

# 2. Create the macOS .app bundle
jpackage \
  --name "Card Flip Word Game" \
  --input target/ \
  --main-jar card-flip-game-1.0.0.jar \
  --main-class com.cs220.cardflip.AppLauncher \
  --type app-image \
  --dest output/ \
  --java-options "-Xmx512m" \
  --java-options "-Dapple.awt.application.name=Card Flip Word Game" \
  --java-options "-Dapple.laf.useScreenMenuBar=true"
```

### 12.2 Resulting Structure

```
output/
└── Card Flip Word Game.app/
    └── Contents/
        ├── Info.plist
        ├── MacOS/
        │   └── Card Flip Word Game      (native launcher)
        └── runtime/                     (bundled JRE — no Java install needed)
```

### 12.3 Running

Double-click `Card Flip Word Game.app`. The launcher starts the embedded server, waits for
`http://localhost:8080/api/leaderboard` to return 200, then shows the loading screen followed
by the rules screen and the player-info screen.

For the development loop, running `AppLauncher` directly starts both the embedded server and
the Swing window in one step:

```bash
mvn package -DskipTests && \
java -cp target/card-flip-game-1.0.0.jar com.cs220.cardflip.AppLauncher
```
