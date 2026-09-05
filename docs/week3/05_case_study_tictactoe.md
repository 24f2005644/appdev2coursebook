# 5. Case Study / Application: Tic-Tac-Toe

---

## Why Tic-Tac-Toe?

Tic-Tac-Toe is a deliberately simple game — but that simplicity makes it a **perfect teaching tool** for frontend concepts. It is complex enough to have meaningful state, user input, and display logic, yet small enough to reason about completely.

By working through Tic-Tac-Toe, we can concretely apply every concept from Topics 1–4:
- **What does the frontend display, and how does it know what to show?** (Topic 1)
- **How is the UI described — imperatively or declaratively?** (Topic 2)
- **What state exists, and at which level?** (Topic 3)
- **Who owns that state — client or server?** (Topic 4)

---

## The Game: A Quick Recap

```
     1   2   3
   ┌───┬───┬───┐
 A │ X │   │ O │
   ├───┼───┼───┤
 B │   │ X │   │
   ├───┼───┼───┤
 C │ O │   │ X │
   └───┴───┴───┘

  X wins! (diagonal A1→B2→C3)
```

- A 3×3 grid of cells.
- Two players: **X** and **O**, taking turns.
- A player wins by filling a row, column, or diagonal with their symbol.
- The game ends in a draw if all 9 cells are filled with no winner.

State to track:
- The current contents of each of the 9 cells.
- Whose turn it is.
- Whether the game is over (and who won, or if it's a draw).

---

## 5.1 Display Determination

> **What is rendered on screen, and who decides it?**

### What Needs to Be Displayed?

At any point in the game, the UI must render:

| UI Element | What Determines It |
|---|---|
| Each cell's content (X, O, or empty) | The **board state** — the 9-cell grid data |
| Whose turn it currently is | The **turn state** — which player is active |
| A win/draw announcement | The **game result state** — computed from board state |
| Highlighted winning cells | Derived from the winning combination in board state |
| A "Reset / Play Again" button | Visibility tied to **game-over state** |

### Who Decides What to Display?

In a declarative frontend (`UI = f(state)`), the answer is clear:

```
UI = f(board, currentPlayer, gameStatus)
```

- The **rendering function** (`f`) takes the current game state as input.
- It **computes** the full UI from that state.
- There is **no manual DOM manipulation** — every cell renders as X, O, or empty based on the state array.

#### Example (Declarative / React-style pseudocode)

```jsx
function Board({ cells, currentPlayer, winner }) {
  return (
    <div className="board">
      {cells.map((cell, index) => (
        <Cell key={index} value={cell} />  // renders "X", "O", or ""
      ))}
      <StatusBar player={currentPlayer} winner={winner} />
    </div>
  );
}
```

The Board component doesn't *decide* what to show — it *derives* it directly from the state passed to it. Change the state, the UI updates automatically.

#### Contrast: Imperative Approach

```javascript
// Imperative: manually find and update each cell
function markCell(index, player) {
  const cell = document.getElementById(`cell-${index}`);
  cell.innerText = player;  // manually update DOM
}
function showWinner(player) {
  document.getElementById('status').innerText = `${player} wins!`;
}
```

Here, the developer must manually track which DOM elements to update. Forgetting one → UI gets out of sync with the actual game state.

### Key Insight
> The display is always a **function of the game state**. The frontend's job is to faithfully render whatever state it is given — not to make decisions about game logic.

---

## 5.2 User Input Lifecycle

> **How are user inputs captured, validated, and processed?**

User input in Tic-Tac-Toe is a **cell click**. Let's trace the complete lifecycle of that event.

### The Input Lifecycle — Step by Step

```
User clicks Cell (B2)
        │
        ▼
① CAPTURE — Event listener fires (onClick / addEventListener)
        │
        ▼
② VALIDATE — Is this move legal?
   ├── Is the cell already occupied? → If yes: REJECT (ignore click)
   ├── Is the game already over?    → If yes: REJECT (ignore click)
   └── Otherwise: ACCEPT
        │
        ▼
③ PROCESS — Apply the move to state
   ├── Update board: cells[4] = "X"   (mark B2 as X)
   ├── Check for win: did X win?
   │     └── If yes: set gameStatus = "X wins"
   ├── Check for draw: are all 9 cells filled?
   │     └── If yes: set gameStatus = "Draw"
   └── Switch turn: currentPlayer = "O"
        │
        ▼
④ STATE UPDATE — New state is committed
        │
        ▼
⑤ RE-RENDER — UI = f(new state) → board updates automatically
```

### ① Capture
- The frontend **binds a click handler** to each cell.
- In declarative frameworks: `onClick={handleCellClick}` on the cell component.
- The handler receives which cell was clicked (by index or coordinate).

### ② Validate
- **Is the cell already occupied?** → If `cells[index] !== null`, do nothing. A player cannot overwrite an existing mark.
- **Is the game over?** → If `gameStatus !== "ongoing"`, do nothing. No moves after the game ends.
- Validation lives on the **client** for UX (instant feedback) but would also be enforced on the **server** in a multiplayer scenario.

### ③ Process (Game Logic)
This is the core **business logic** of the game. It computes the new state:

```javascript
function handleCellClick(index) {
  // Validate
  if (cells[index] !== null || gameOver) return;

  // Process
  const newCells = [...cells];
  newCells[index] = currentPlayer;          // mark the cell

  const winner = checkWinner(newCells);     // check for win
  const isDraw = !winner && newCells.every(c => c !== null);

  // Update state
  setCells(newCells);
  setCurrentPlayer(currentPlayer === 'X' ? 'O' : 'X');
  if (winner) setGameStatus(`${winner} wins!`);
  if (isDraw) setGameStatus('Draw!');
}
```

> Note: `checkWinner()` inspects all 8 possible winning lines (3 rows, 3 columns, 2 diagonals) — this is game logic, not UI logic.

### ④ State Update
- The new state is committed (e.g., via `setState` in React, or a store mutation in Vuex/Redux).
- This triggers a **re-render**.

### ⑤ Re-render
- The framework re-runs `UI = f(new state)`.
- The board now shows the new mark, the status bar shows whose turn it is (or announces a winner), and the winning cells may be highlighted.
- All of this happens **automatically** — the developer did not manually update the DOM.

---

## 5.3 State Categorization in Practice

> **Distinguishing board data (system/application state) from UI-level interactions.**

Now we apply the **three-level state hierarchy** from Topic 3 to the Tic-Tac-Toe game.

### Mapping Game Data to State Levels

#### System State

In the context of a **multiplayer / persistent** Tic-Tac-Toe (e.g., an online game with history):

| Data | Why It's System State |
|---|---|
| All historical games played | Stored in the backend DB; exists independently of any player |
| All user accounts / player profiles | Server-side, persists across sessions |
| Leaderboards and statistics | Aggregated from all games; lives on the server |

> For a simple **local single-player** game, system state is minimal or absent — there's no server.

#### Application State

| Data | Why It's Application State |
|---|---|
| **The 9-cell board** (`cells` array) | The core game data for this session; drives the main UI |
| **Current player** (`currentPlayer = 'X' \| 'O'`) | Determines whose turn it is; scoped to this game session |
| **Game status** (`"ongoing"`, `"X wins"`, `"Draw"`) | The result of this game; lives for the duration of the session |

These are the **meaningful, session-level facts** about the game. They would need to be sent to a server in a multiplayer scenario so the other player can see the same board.

> Board state is to Tic-Tac-Toe what the shopping cart is to e-commerce — it's the user's data for this session.

#### UI State (Ephemeral)

| Data | Why It's UI State |
|---|---|
| **Hover highlight** on a cell | Which cell the cursor is over — purely visual, lost the moment the mouse moves |
| **"Are you sure?"** confirmation modal visibility | Whether the reset confirmation dialog is open |
| **Loading indicator** | Shown while fetching game data from server (multiplayer); gone once data loads |
| **Animation state** | Whether the winning-cells highlight animation is playing |
| **Tooltip visibility** | Whether the "It's your turn!" tooltip is shown |

These are **throwaway** — they serve the moment and have no meaning beyond the current render.

### Summary Table

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        TIC-TAC-TOE STATE MAP                            │
├──────────────────┬────────────────────────────┬────────────────────────┤
│   State Level    │         Data               │      Lifetime          │
├──────────────────┼────────────────────────────┼────────────────────────┤
│  System State    │ All past games, user        │ Permanent (DB)         │
│                  │ profiles, leaderboards      │                        │
├──────────────────┼────────────────────────────┼────────────────────────┤
│  Application     │ Board (cells array),        │ Session (one game)     │
│  State           │ current player, game result │                        │
├──────────────────┼────────────────────────────┼────────────────────────┤
│  UI State        │ Hover highlight, modal      │ Milliseconds /         │
│  (Ephemeral)     │ open/closed, loading,       │ until re-render        │
│                  │ animation playing           │                        │
└──────────────────┴────────────────────────────┴────────────────────────┘
```

### The Critical Distinction

A common mistake is treating board state as UI state and storing it in a component's local state when it actually needs to be:
- **Shared** between multiple components (the board, the status bar, the reset button all need to know the game result).
- **Persisted** to a server in multiplayer.
- **Reproducible** — you should be able to serialize the board state and fully reconstruct the game.

UI state (hover, modals, animations) by contrast **should not** be elevated to application state — doing so creates unnecessary complexity and couples visual concerns to business logic.

---

## Putting It All Together

Tic-Tac-Toe illustrates all the core frontend principles in one simple package:

| Concept | How It Appears in Tic-Tac-Toe |
|---|---|
| **Frontend = presentation** | The game logic (win check) is separate from rendering |
| **`UI = f(state)`** | Board renders as a pure function of the `cells` array |
| **Imperative vs. Declarative** | Declarative: cells render themselves; no manual `innerText` calls |
| **State Hierarchy** | Board = app state; hover = UI state; past games = system state |
| **Stateless HTTP** | In multiplayer: each move is an HTTP request; session identifies the game |
| **Client vs. Server state** | Local game = client-maintained; multiplayer = server-maintained |

> Tic-Tac-Toe is a microcosm of every web application. Master the state and rendering model here, and the same principles apply whether you're building a game, a dashboard, or a social platform.
