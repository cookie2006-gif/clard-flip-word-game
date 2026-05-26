# Card Flip Word Game — Server Design Document

**Course**: CS 220 Applied Data Structures
**Component**: Spring Boot REST Server
**Grade Weight**: 13%

---

## 1. Project Overview

Card Flip Word Game is a client-server word-guessing game. A single human player competes
against one or more bot players to reveal hidden English phrases letter by letter, earning
points by flipping cards that determine point multipliers and special effects. The first
player to solve the phrase wins the round; the player with the highest **cumulative** score
after all rounds wins the game.

On their turn, a player may take one of these phrase actions:

- **Guess a letter** — after flipping a points card, guess any single letter (a vowel **or**
  a consonant; both are allowed). A correct guess that reveals the last remaining letter wins
  the round.
- **Solve the puzzle** — type the whole answer to win the round. **Human players only**; bots
  never solve (§11) and can only win a round by revealing the last letter through guessing.

Whoever wins a round — by solving or by revealing the final letter — receives a 2500-point
bonus (§6.3), applied identically to humans and bots.

This document describes the **server-side** design exclusively. The server holds all secret
game state and enforces all rules. The client is described in `client-design.md`.

**Architecture principle — server-silent**: The client never knows the secret phrase, the
shuffled card order, or the face-down card contents. All randomness and validation happen on
the server.

**Networked, multi-client REST API**: The server exposes a fully networked REST interface
built around a game-room model (`create` / `start`). A second client or a test tool (curl,
Postman) can connect to a running server over the network, which makes the client–server
interoperability independently testable. The packaged desktop client launches a
single-player-vs-bots game by default, but the API itself is not limited to that scenario.

---

## 2. Technology Stack

| Layer | Technology | Reason |
|-------|-----------|--------|
| Language | Java 21 | LTS, pattern matching, records |
| Framework | Spring Boot 3.2 | Auto-configuration, embedded Tomcat |
| Serialization | Jackson (bundled) | JSON ↔ Java DTO mapping |
| Storage | In-memory (`ConcurrentHashMap`) | No database needed for demo scope |
| Build | Maven 3 | Standard Java build tool |

---

## 3. Package Structure

```
com.cs220.cardflip.server/
├── CardFlipServerApplication.java   (Spring Boot root config)
├── controller/
│   ├── GameController.java          (game room + turn REST endpoints)
│   ├── LeaderboardController.java   (top-10 score endpoints)
│   └── GlobalExceptionHandler.java  (@ControllerAdvice; maps exceptions to clean HTTP errors)
├── service/
│   ├── GameService.java             (core game logic, turn processing)
│   ├── DeckService.java             (deck creation + Fisher-Yates shuffle)
│   ├── PhraseService.java           (loads phrases.json at startup)
│   └── LeaderboardService.java      (maintains top-10 PriorityQueue)
├── model/
│   ├── Card.java
│   ├── CardType.java                (enum: POINTS / HALVE / DOUBLE / LOSE_TURN)
│   ├── Player.java
│   ├── GameSession.java             (all state for one active game)
│   ├── GameStatus.java              (enum: WAITING / IN_PROGRESS / FINISHED)
│   ├── TurnState.java               (enum: WAITING_FOR_FLIP / WAITING_FOR_GUESS / WAITING_FOR_SOLVE)
│   ├── GameAction.java              (one action log entry for the history stack)
│   ├── Phrase.java                  (one quiz item: topic + context/clue + answer)
│   └── ScoreEntry.java              (Comparable for PriorityQueue)
└── dto/
    ├── (request DTOs)  CreateGameRequest, StartGameRequest, FlipCardRequest,
    │                   GuessLetterRequest, SolvePuzzleRequest, PassRequest, SubmitScoreRequest
    └── (response DTOs) CreateGameResponse, StartGameResponse, GameStateResponse,
                        FlipCardResponse, GuessLetterResponse, SolvePuzzleResponse,
                        RoundResultDTO, BotTurnLog, BotStepDTO, LeaderboardResponse,
                        PlayerDTO, CardDTO, ScoreUpdateDTO, ScoreEntryDTO, ErrorResponse
```

There is no buy-vowel endpoint, request, or response in the design — vowels and consonants are
revealed through the same letter-guessing action.

---

## 4. Data Structures

Each data structure is chosen for its algorithmic properties, not convenience.

| Structure | Java Type | Location | Justification |
|-----------|-----------|----------|---------------|
| Card Deck | `ArrayList<Card>` | `GameSession.deck` | O(1) index access required by Fisher-Yates swap |
| Game Sessions | `ConcurrentHashMap<String,GameSession>` | `GameService.sessions` | Thread-safe O(1) gameId → session lookup |
| Players per Session | `LinkedHashMap<String,Player>` | `GameSession.players` | O(1) lookup, preserves order so the human is always first |
| Question Bank | `List<Phrase>` (+ `Map<String,List<Phrase>>` index by topic) | `PhraseService.phraseBank` | O(1) topic lookup; each Phrase holds topic + context + answer |
| Turn Queue | `ArrayDeque<String>` | `GameSession.turnOrder` | O(1) `poll()` + `offer()` for round-robin rotation |
| Guessed Letters | `HashSet<Character>` | `GameSession.guessedLetters` | O(1) duplicate detection per guess |
| Hidden Phrase | `char[]` | `GameSession.hiddenPhrase` | O(1) index access for letter reveal |
| Leaderboard | `PriorityQueue<ScoreEntry>` | `LeaderboardService.leaderboard` | Min-heap: O(log n) insert, O(1) peek at lowest |
| Turn History | `Deque<GameAction>` (as stack) | `GameSession.turnHistory` | LIFO action log / potential undo |

### 4.1 Fisher-Yates Shuffle (DeckService)

```java
/** O(n) in-place shuffle. Custom implementation — not Collections.shuffle() — to
    demonstrate algorithm understanding and allow seeded testing. */
public void shuffleDeck(ArrayList<Card> deck) {
    for (int i = deck.size() - 1; i > 0; i--) {
        int j = rand.nextInt(i + 1);
        Collections.swap(deck, i, j);   // O(1) swap on ArrayList
    }
}
```

**Time**: O(n) · **Space**: O(1) — in-place.

The deck is re-shuffled at the start of every flip, so the client's choice of card index
(0–11) is cosmetically meaningful but algorithmically irrelevant. All randomness lives
server-side.

### 4.2 Leaderboard Min-Heap Strategy

```java
// PriorityQueue uses ScoreEntry.compareTo → ascending score (min at top).
// Keep at most 10 entries. When full, evict the minimum if the new score is higher.
if (leaderboard.size() < MAX_SIZE)              leaderboard.offer(entry);
else if (score > leaderboard.peek().getScore()) { leaderboard.poll(); leaderboard.offer(entry); }
```

---

## 5. The 12-Card Deck

| # | Card | Type | Effect |
|---|------|------|--------|
| 1 | 100 | POINTS | Correct letter → +100 × occurrences to round score |
| 2 | 250 | POINTS | Correct letter → +250 × occurrences |
| 3 | 500 | POINTS | Correct letter → +500 × occurrences |
| 4 | 750 | POINTS | Correct letter → +750 × occurrences |
| 5 | 1000 | POINTS | Correct letter → +1000 × occurrences |
| 6 | 1500 | POINTS | Correct letter → +1500 × occurrences |
| 7 | 2000 | POINTS | Correct letter → +2000 × occurrences |
| 8 | 2000 | POINTS | Correct letter → +2000 × occurrences |
| 9 | Halve | HALVE | Round score ÷ 2 (floor), turn ends |
| 10 | Double | DOUBLE | Round score × 2, turn ends |
| 11 | Lose Turn #1 | LOSE_TURN | Turn ends (client shows a centered "Lose Turn" notice first — see §7.2) |
| 12 | Lose Turn #2 | LOSE_TURN | Same as above |

There is no Lucky card and no card that can zero a player's accumulated score; the most
punishing cards (Halve, Lose Turn) only affect the current round's progress. Two cards now
award 2000 points (cards #7 and #8).

---

## 6. Score Model — Cumulative Across Rounds

### 6.1 Two score fields per player

Each `Player` carries:

| Field | Meaning | Lifetime |
|-------|---------|----------|
| `roundScore` | points earned in the **current** round | reset to 0 at the start of each new round |
| `totalScore` | accumulated points across **all** rounds | persists for the whole game; reset only when a new game starts |

### 6.2 End-of-round banking

At the end of every round, one loop runs over **all** players — human and bot alike — with
identical logic:

```
for each player p in session.players:
    p.totalScore += p.roundScore     // bank this round's points for everyone
    p.roundScore  = 0                // reset round score for everyone
    p.eliminated  = false            // everyone is back in for the next round
```

Because every player passes through the same single code path, all cumulative scores carry
forward consistently into the next round. The round winner additionally receives the round-win
bonus (§6.3) added to `totalScore`.

### 6.3 Round-win bonus

Whoever **wins the round** receives a **2500-point bonus** added to their `totalScore`, then the
round ends. A round can be won in two ways, and both award the same bonus identically for humans
and bots:

1. **Solve** — a player submits the correct full answer via `POST /game/{gameId}/solve`
   (humans only; see §11, bots never solve).
2. **Full reveal by guessing** — a player's correct letter guess reveals the **last** remaining
   letter, so `isPhraseFullyRevealed()` becomes true. That player is the round winner. This is
   the only way a bot can win a round, but it applies to humans too.

In both cases the winner gets +2500 on top of any points already earned that round, the round
score is then banked into the total (§6.2), and the next round (if any) begins.

---

## 7. REST API Specification

**Base URL**: `http://localhost:8080/api`
All bodies are JSON (`Content-Type: application/json`).
Success: `200 OK`. Errors: `400`, `403`, `404`, `409` with an `ErrorResponse` body.

### 7.1 Game Room Management

#### `POST /game/create`
Create a new game room. `numBots` must be ≥ 1; `numRounds` must be ≥ 1.

**Request**:
```json
{ "hostName": "Alice", "numRounds": 3, "numBots": 1 }
```
**Response**:
```json
{ "gameId": "a1b2c3d4", "hostPlayerId": "p1x2y3z4", "status": "WAITING" }
```
**Errors**: `400` (`numBots < 1`, `numRounds < 1`, blank name).

> The `gameId` is still generated and returned (the API uses it to route requests), but the
> desktop client never shows it to the player and there is no join-by-id flow in the UI.

#### `POST /game/{gameId}/start`
Start the game. **The human player always takes the first turn** — `startGame()` builds the
turn order with the host (the human) at the front, followed by the bots in creation order.

**Request**: `{ "hostPlayerId": "p1x2y3z4" }`
**Response**:
```json
{
  "status": "IN_PROGRESS",
  "currentRound": 1,
  "currentPlayer": "p1x2y3z4",
  "hiddenPhrase": "_ _ _ _ _ _ _",
  "topic": "Science",
  "context": "The force that pulls objects toward the earth."
}
```

> There is no `join` endpoint in the gameplay flow. Bots are created together with the room in
> `POST /game/create`, so a game is fully populated at creation and starts immediately.

#### `GET /game/{gameId}/state`
Poll current state (client polls every 2 s).

**Response**:
```json
{
  "status": "IN_PROGRESS",
  "currentRound": 1,
  "totalRounds": 3,
  "currentPlayer": "p1x2y3z4",
  "turnState": "WAITING_FOR_FLIP",
  "hiddenPhrase": "G _ _ _ _ _ _",
  "guessedLetters": ["G"],
  "topic": "Science",
  "context": "The force that pulls objects toward the earth.",
  "players": [
    { "playerId": "p1x2y3z4", "name": "Alice",
      "totalScore": 3000, "roundScore": 500, "eliminated": false }
  ],
  "lastFlippedCard": null,
  "roundResult": null,
  "botTurnLog": null
}
```

`roundResult` is normally `null`. On the first poll after a round ends it carries a
`RoundResultDTO` (§7.3) so every player — not just the one who acted — sees the result. It is
cleared once the next round's first action occurs.

`botTurnLog` is normally `null`. After one or more bot players have taken their turns (between
the human's turns), the next `/state` poll carries a `BotTurnLog` (§7.4) — an ordered list of
every step each bot performed, so the client can replay those steps one by one with animation
instead of the human seeing only the final result. It is cleared once the human takes their
next action.

### 7.2 Turn Actions

#### `POST /game/{gameId}/flip`
Flip a card. The server runs Fisher-Yates first, then returns the card at the requested index.

**Request**: `{ "playerId": "p1x2y3z4", "cardIndex": 5 }`

**Response (POINTS)**:
```json
{ "card": { "type": "POINTS", "value": 1000, "displayName": "1000" },
  "nextAction": "GUESS_LETTER", "message": "Guess a letter!" }
```
**Response (LOSE_TURN)**:
```json
{ "card": { "type": "LOSE_TURN", "displayName": "Lose Turn" },
  "nextAction": "SHOW_LOSE_TURN_THEN_ADVANCE",
  "nextPlayer": "p5a6b7c8" }
```
**Response (DOUBLE)**:
```json
{ "card": { "type": "DOUBLE", "displayName": "Double" },
  "nextAction": "TURN_ENDED",
  "scoreUpdate": { "playerId": "p1x2y3z4", "field": "roundScore",
                   "oldValue": 1500, "newValue": 3000 } }
```
**Errors**: `403` (not your turn or eliminated), `400` (invalid index, must guess first).

On a Lose Turn the server advances the turn immediately and reports the next player via
`nextPlayer`; the `nextAction` value `SHOW_LOSE_TURN_THEN_ADVANCE` instructs the client to
display a centered "Lose Turn" notice **before** refreshing to the next player. The server
keeps no wall-clock state about this display — the delay is purely a client-side presentation
concern, which keeps the server deterministic and easy to test.

#### `POST /game/{gameId}/guess`
Guess a **single letter** after flipping a points card. The letter may be **any** letter A–Z —
a vowel or a consonant, both are accepted; vowels are not excluded. The server reveals every
occurrence of that letter in the phrase.

**Request**: `{ "playerId": "p1x2y3z4", "letter": "E" }`

**Response (correct)**:
```json
{ "correct": true, "matches": 2, "pointsEarned": 2000,
  "hiddenPhrase": "T _ _ _ E   _ _ _ _", "nextAction": "FLIP_AGAIN" }
```
**Response (correct guess that reveals the last letter → round won)**:
```json
{ "correct": true, "matches": 1, "pointsEarned": 1000,
  "hiddenPhrase": "G R A V I T Y",
  "roundWon": true, "bonus": 2500, "roundWinner": "p1x2y3z4",
  "revealedAnswer": "GRAVITY", "message": "Round won!",
  "nextRound": 2, "hasMoreRounds": true }
```
**Response (wrong)**:
```json
{ "correct": false, "matches": 0, "nextAction": "TURN_ENDED", "nextPlayer": "p5a6b7c8" }
```
**Errors**: `400` (not a single A–Z letter, or already guessed), `403` (not your turn / wrong
state).

If a correct guess fills the **last** blank (`isPhraseFullyRevealed()`), the guesser wins the
round: they receive the **2500 round-win bonus** (§6.3), the round ends, and the response
mirrors a successful `solve` (carrying `roundWinner`, `revealedAnswer`, `hasMoreRounds`). This
is how a bot wins a round, since bots never solve (§11). Otherwise a correct guess returns
`nextAction: FLIP_AGAIN` as usual.

#### `POST /game/{gameId}/solve`
Attempt to solve the full puzzle.

**Request**: `{ "playerId": "p1x2y3z4", "phrase": "gravity" }`

**Comparison rules (whitespace tolerant)**: the server normalises both the submitted guess
and the stored answer before comparing — trim leading/trailing whitespace, collapse internal
runs of whitespace to a single space, uppercase everything — then compares for exact equality.
So `"  Gravity "` matches `"GRAVITY"`. (If a future answer contains spaces, they are handled
the same way.)

**Response (correct)**:
```json
{ "correct": true, "bonus": 2500, "roundWinner": "p1x2y3z4",
  "revealedAnswer": "GRAVITY",
  "message": "Round won!",
  "nextRound": 2, "hasMoreRounds": true,
  "newPhrase": "_ _ _ _ _ _ _", "topic": "Science",
  "context": "The closest planet to the sun." }
```
**Response (wrong)**:
```json
{ "correct": false, "message": "Round lost!",
  "playerEliminated": true, "nextAction": "TURN_ENDED", "nextPlayer": "p5a6b7c8" }
```

#### `POST /game/{gameId}/pass`
Pass the turn without acting. The client calls this whenever a client-side countdown expires —
the 10-second letter-input timer or the 15-second solve timer — and also for a voluntary skip.
The server ends the current turn and advances to the next player. The server runs **no**
wall-clock timer of its own; all countdowns live in the client (see `client-design.md` §8.6
and §8.7), which keeps the server deterministic and easy to test.

**Request**: `{ "playerId": "p1x2y3z4" }`
**Response**: `{ "nextAction": "TURN_ENDED", "nextPlayer": "p5a6b7c8" }`

### 7.3 RoundResultDTO

Carried inside `GET /state` (`roundResult` field) on the first poll after a round ends, so
every player sees who won and what the answer was:

```json
{
  "roundNumber": 1,
  "winnerName": "Alice",
  "winnerPlayerId": "p1x2y3z4",
  "revealedAnswer": "GRAVITY",
  "hasMoreRounds": true,
  "nextRoundNumber": 2,
  "standings": [
    { "name": "Alice", "totalScore": 7500 },
    { "name": "Bot 1", "totalScore": 4200 }
  ]
}
```

If a round ends with no winner (e.g. all players eliminated), `winnerName` is `null` and the
client shows that no one solved it, along with the revealed answer.

### 7.4 BotTurnLog

Carried inside `GET /state` (`botTurnLog` field) after one or more bots have acted since the
human's last action. It is an ordered list of **steps**, one per discrete bot action, so the
client can replay each step with the same flying-card / flip / result animation a human turn
uses — letting the player actually follow what each bot did, rather than seeing only the final
board. Each step is self-contained (it names the bot and describes exactly one action).

```json
{
  "steps": [
    {
      "botName": "Bot 1",
      "botPlayerId": "b1",
      "action": "FLIP",
      "card": { "type": "POINTS", "value": 1000, "displayName": "1000" }
    },
    {
      "botName": "Bot 1",
      "botPlayerId": "b1",
      "action": "GUESS_LETTER",
      "letter": "R",
      "correct": true,
      "matches": 1,
      "pointsEarned": 1000,
      "roundScoreAfter": 1000
    },
    {
      "botName": "Bot 1",
      "botPlayerId": "b1",
      "action": "FLIP",
      "card": { "type": "LOSE_TURN", "displayName": "Lose Turn" }
    },
    {
      "botName": "Bot 1",
      "botPlayerId": "b1",
      "action": "TURN_ENDED",
      "reason": "LOSE_TURN"
    }
  ]
}
```

Step `action` values: `FLIP` (always includes the revealed `card`), `GUESS_LETTER` (includes
`letter`, `correct`, `matches`, `pointsEarned`, `roundScoreAfter`, and — when the guess reveals
the last letter — `roundWon: true` with `bonus: 2500`), and `TURN_ENDED` (includes a `reason`:
`WRONG_GUESS`, `LOSE_TURN`, `HALVE`, `DOUBLE`, or `SOLVED`). There is **no** `SOLVE` step in a
bot log, because bots never solve (§11); a bot that wins a round does so via a `GUESS_LETTER`
step that reveals the last letter, followed by a `TURN_ENDED` step with reason `SOLVED`. A
`DOUBLE`/`HALVE` flip is reported as a `FLIP` step whose card type is `DOUBLE`/`HALVE`, followed
by a `TURN_ENDED` step carrying the matching reason and the new round score.

The client replays the steps in order (see `client-design.md` §8.8), holding each step on
screen for ~1.5 s, and only resumes the human's turn after the whole log has been replayed.
The server itself does **not** pace anything — it computes all bot moves immediately and just
records them; the pacing is entirely a client-side presentation concern, which keeps the
server deterministic and testable.

### 7.5 Leaderboard

#### `GET /api/leaderboard`
Returns top-10 scores (descending).
```json
{ "topScores": [ { "playerName": "Alice", "score": 15000, "date": "2026-05-23" } ] }
```

#### `POST /api/leaderboard`
Submit a score. Called automatically at game end for every player's final `totalScore`, and
also available for manual submission.
```json
{ "playerName": "Alice", "score": 15000 }
```

When `status` becomes `FINISHED`, `GameService.endGame()` submits each player's final
`totalScore` to `LeaderboardService` before returning the final state, so the leaderboard is
always current at the end of a game.

---

## 8. Error Handling & Edge Cases

### 8.1 Input Validation

| Input | Rule |
|-------|------|
| Letter | A single letter A–Z, normalised to uppercase. Any letter is allowed (vowels are **not** excluded). |
| Card index | Integer 0–11 inclusive |
| Solve phrase | Normalised: trim, collapse internal spaces, uppercase |
| `numBots` | Integer ≥ 1 |
| `numRounds` | Integer ≥ 1 |

### 8.2 Game State Validation

| Scenario | HTTP | Message |
|----------|------|---------|
| Not your turn | 403 | "Not your turn" |
| Flip when must guess | 400 | "Must guess a letter first" |
| Guess when no points card | 400 | "Must flip a points card before guessing" |
| Already guessed letter | 400 | "Letter already guessed: X" |
| Eliminated player acts | 403 | "You are eliminated this round" |
| Create with 0 bots | 400 | "At least one bot is required" |
| Game not found | 404 | "Game not found: {id}" |

### 8.3 Score Edge Cases

- **Halve on 0**: stays 0 (`floor(0/2) = 0`).
- **Halve odd number**: floor division (e.g. 1501 → 750).
- **Double**: no cap.
- **All players eliminated**: the round ends with no winner; banking still runs for everyone
  (round scores are added to totals), then the next round starts.

### 8.4 Phrase Reveal

- The hidden phrase shows spaces and punctuation; letters become `_`.
- Display format: each character position is separated by a single space. Example:
  `"gravity"` → `"_ _ _ _ _ _ _"`, and after guessing G and R → `"G R _ _ _ _ _"`. (If an
  answer ever contains a space, the word boundary is shown as a wider gap.)
- When all `_` are replaced, `isPhraseFullyRevealed()` triggers round end.

### 8.5 Robust Turn Rotation

`startRound()` rebuilds `turnOrder` from scratch at the start of every round, re-adding all
non-eliminated players from `session.players` in order (human first), so the queue is always
non-empty and consistent. `advanceTurn()` checks for an empty queue and ends the round rather
than dereferencing a missing element. Eliminations are collected into a temporary list and
applied after iteration completes, so the queue is never mutated while being traversed. These
invariants keep multi-round games stable.

### 8.6 Global Exception Handling

`GlobalExceptionHandler` (`@ControllerAdvice`) maps every exception to a clean `ErrorResponse`
JSON with an appropriate status code, so a single bad request can never crash the server
process; the embedded Tomcat threads survive and the game continues.

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(GameException.class)
    public ResponseEntity<ErrorResponse> handleGame(GameException e) {
        return ResponseEntity.status(e.getStatus())
                             .body(new ErrorResponse(e.getMessage()));
    }
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAny(Exception e) {
        // log full stack trace server-side, return a safe message to the client
        return ResponseEntity.status(500)
                             .body(new ErrorResponse("Internal server error"));
    }
}
```

---

## 9. Phrase Bank (`phrases.json`)

Each quiz item carries three fields, so the player always has a clue to reason from — the
topic alone is not enough to guess a short answer:

| Field | Meaning |
|-------|---------|
| `topic` | The category, shown to the player (e.g. "Science", "Animals") |
| `context` | A one-line clue/question describing the answer, shown to the player |
| `answer` | The hidden keyword the player must reveal (never sent to the client until solved) |

`Phrase.java` is a small record/class with these three fields. `PhraseService` loads
`phrases.json` at startup (from the classpath, e.g. `src/main/resources/phrases.json`) into a
`List<Phrase>`, and also builds a `Map<String, List<Phrase>>` indexed by topic for O(1) topic
lookup.

The JSON is a single object with a `phrases` array:

```json
{
  "phrases": [
    { "topic": "Science", "context": "The force that pulls objects toward the earth.", "answer": "gravity" }
  ]
}
```

The bank ships with **10 phrases** across 5 topics; every answer is a single word of **6–10
letters** containing only letters A–Z (no spaces or punctuation), which keeps the reveal logic
(§8.4) simple:

| Topic | Context (clue) | Answer |
|-------|----------------|--------|
| Science | The force that pulls objects toward the earth. | gravity |
| Science | The closest planet to the sun. | mercury |
| Human Body | The clear liquid in your mouth that helps you digest food. | saliva |
| Human Body | The bony cage that protects your heart and lungs. | ribcage |
| Animals | The largest land animal, with a long trunk. | elephant |
| Animals | A tall African animal with a very long neck. | giraffe |
| Geography | The largest hot desert in the world, in Africa. | sahara |
| Geography | The capital city of England. | london |
| Technology | A program that protects a computer from network attacks. | firewall |
| Technology | The global network that connects computers worldwide. | internet |

At the start of each round the server picks a random `Phrase`, exposes its `topic` and
`context` in the game state, and keeps `answer` secret. The `solve` endpoint compares the
player's guess against `answer` using the normalisation rules in §7.2.

The full file is provided as `phrases.json` alongside these design docs.

---

## 10. Concurrency

- `ConcurrentHashMap` for the sessions map (thread-safe put/get).
- `synchronized(session)` inside each service method ensures only one action modifies a single
  game at a time.
- Multiple games run concurrently without global lock contention.
- Combined with `GlobalExceptionHandler`, a failure in one game's request can never take down
  the shared server.

---

## 11. Bot Player Logic

> **MANDATORY — the code must implement this exactly.** Two rules below are the source of the
> bugs reported during testing (bots solving instantly / guessing too many letters), so they are
> non-negotiable: **(A) bots never solve**, and **(B) bots are deliberately weak** via a random
> per-turn accuracy. A bot must never feel "too smart" or end the game quickly.

When the next player is a bot, `GameService.processBotTurn()` runs synchronously and records
every step it takes into the session's `BotTurnLog` (§7.4).

### 11.1 Deliberately weak bot — random per-turn accuracy (MANDATORY)

At the **start of each bot turn**, the server picks one random **accuracy** value `p` in the
range **30%–50%** (uniform) and keeps that same `p` for the **whole turn**. Each time the bot
flips a `POINTS` card and must guess a letter, it decides correct-vs-wrong by a single random
draw against `p`:

- **With probability `p`** (correct): the bot guesses a random not-yet-guessed letter that
  **does** appear in the phrase → correct guess → it earns points and flips again.
- **With probability `1 − p`** (wrong): the bot guesses a random not-yet-guessed letter that
  **does not** appear in the phrase → wrong guess → its turn ends.

Because a wrong guess ends the turn, a bot on average reveals only about **0–2 letters per turn**
before missing — without any hard cap. There is **no** "always guess a correct letter" path; the
old behaviour where a bot reliably guessed correct letters (and so revealed the whole phrase
quickly) is removed. This weakening applies to **bots only** — the human player is never limited.

### 11.2 Per-card handling and step logging

1. Shuffle the deck, pick a random index → record a `FLIP` step with the revealed card.
2. `POINTS` → apply the §11.1 accuracy draw, then guess a letter accordingly → record a
   `GUESS_LETTER` step (letter, correct, matches, pointsEarned, roundScoreAfter). On a correct
   guess the bot flips again (another `FLIP` step); on a wrong guess the turn ends.
3. `LOSE_TURN` → record the `FLIP` step, then a `TURN_ENDED` step with reason `LOSE_TURN`.
4. `DOUBLE` / `HALVE` → record the `FLIP` step, apply the effect, then a `TURN_ENDED` step with
   reason `DOUBLE` / `HALVE`.

### 11.3 Bots never solve (MANDATORY)

**A bot has no access to the `solve` action and never submits a full answer.** The only way a
bot can win a round is by **revealing the last letter through a correct guess**: if a bot's
`GUESS_LETTER` fills the final blank (`isPhraseFullyRevealed()`), the bot wins the round and
receives the same **2500 round-win bonus** as a human (§6.3). That winning guess is recorded as a
`GUESS_LETTER` step followed by a `TURN_ENDED` step with reason `SOLVED`. There is **no** `SOLVE`
step in a bot's log, ever.

The bot computes its entire turn immediately; the recorded steps let the client replay them at
a human-readable pace (§7.4, `client-design.md` §8.8). A recursion depth guard
(`botDepth < 10`) prevents infinite round loops. If several bots act in a row before the human's
next turn, all their steps are appended to the same `BotTurnLog` in order.

The log is attached to the session and returned on the next `GET /state`; it is cleared once
the human takes their next action.

---

## 12. Testing Strategy

### Unit Tests (JUnit 5)
- `DeckService`: exactly 12 cards after init; two 2000-point POINTS cards present and no Lucky
  card; all cards present after shuffle; statistical randomness over 1000 shuffles; seeded
  shuffle repeatable.
- `GameService`:
  - the human is always first in `turnOrder` after `start`;
  - turn rotation across at least 3 rounds with eliminations — assert no exception and a
    never-empty queue;
  - cumulative scoring — assert both human and bot `totalScore` carry across rounds while
    `roundScore` resets;
  - letter guessing accepts both vowels and consonants;
  - solve normalisation with spaces;
  - round-win bonus equals 2500 whether the round is won by `solve` **or** by a guess that
    reveals the last letter;
  - a bot wins a round only by revealing the last letter via guessing (never via `solve`), and
    still receives the 2500 bonus;
  - **bot accuracy** — each bot turn draws an accuracy `p` in 30–50% held for the whole turn;
    over many turns the bot reveals ~0–2 letters per turn on average and never reliably reveals
    the whole phrase; bots never call `solve`;
  - round-end produces a correct `RoundResultDTO`.
- `PhraseService`: phrase loading and random selection.

### Integration Test (curl / Postman)
Full game flow proving interoperability over REST:
`create → start → flip → guess → solve → round 2 → game end → leaderboard updated`.
