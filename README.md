# Kriegspiel (Ludii)

In this game each player can see their own pieces, but not those of their opponent. For this reason, it is necessary to have a third person (or computer) act as an umpire, with full information about the progress of the game. Players attempt to move on their turns, and the umpire declares their attempts `legal` or `illegal`. If the move is illegal, the player tries again; if it is legal, that move stands. Each player is given information about checks and captures. Since the position of the opponent's pieces is unknown, Kriegspiel is a game of imperfect information.

While Ludii's first implementation of Kriegspiel correctly models the game's core rules of umpire arbitration, its unconstrained design poses a significant challenge for artificial intelligence. The model presents the AI with a raw action space, allowing it to explore and attempt moves that are impossible (e.g., a Rook moving diagonally).

This is highly inefficient for any search algorithm. The AI wastes valuable computational resources evaluating moves that a human player would instantly recognize as irrational. Furthermore, a rational player would not repeatedly attempt the same illegal move after it has been rejected by the umpire within the same turn.

This project refactors Kriegspiel to address these issues by creating a **interface that pre-filters the action space**. By only presenting valid moves, we align the AI's decision-making process more closely with rational human play.

The impact of this change is significant, now games finish dramatically faster without wasted retries. This refactoring has also enabled a subset of MCTS-based agents—specifically `Bandit Tree Search (Avg)`, `EPT`, and `EPT-QB` to produce meaningful, non-zero value estimates, a critical step toward developing truly competitive Kriegspiel agents.

## Description

This implementation pre-filters moves that fail feasibility checks, eliminating `Hell no` (or `Impossible` or `Nonsense`) announcements (when the attempted move is always illegal regardless of the opponent's position). All filtering is based exclusively on information legitimately available to the player (game rules, own pieces, and information derivable from umpire announcements and move history), organized into the following domains:

### FILTERING RULES

#### A — Memory
- **A1:** Rejected moves are excluded for the remainder of the turn.
- **A2:** Sliding trajectories stop at previously rejected destinations (except when the side is *in check*).
- **A3** — **SONAR** *(NOT IMPLEMENTED)*: occupancy deduction from rejected pawn forward moves. Destination is marked as `AttackerSite` (applies C2–C4):
    - **A3.1:** When not under check, the pawn is pinnable or the move stays on the King–Pawn alignment.
    - **A3.2:** When under check, the destination is adjacent to the terminal `CheckSite` on the attacker side.
- **A4** — **PIN DETECTION** *(NOT IMPLEMENTED)*: deductions from rejected moves when a piece is *pinnable*:
    - **A4.1:** A sliding piece move rejected → the piece is (likely) pinned. An enemy exists along the `PinRay`. Restrict the piece to moves along that ray.
        - **A4.1.1:** If only one square is visible along the `PinRay` away from the King, mark that square as `AttackerSite` (apply C2–C4).
    - **A4.2:** Pawn capture rejected when C4 applies → the pawn is (likely) pinned. Freeze pawn moves outside the `PinRay`.

#### B — Geometry
- **B1:** Pieces follow canonical movement patterns.
- **B2:** Sliding trajectories stop at friendly pieces.
- **B3:** Castling requires unmoved King/Rook and intermediate squares not occupied by friendly pieces.
- **B4** — **Early-game** (pawns on starting rank)
    - **B4.1:** Turns 1–2: no pawn captures (enemies cannot reach rows 3/6).
    - **B4.2:** Turns 3–4: pawn captures limited to `A3`, `H3`, `A6`, `H6` (squares reachable by a bishop from the initial setup).

#### C — Umpire inference
- **C1:** Pawn captures are enabled only when `Tries` > 0.
- **C2:** Sliding pieces stop at `AttackerSite`.
- **C3:** Pawns cannot advance onto `AttackerSite`.
- **C4:** If `Tries` equals the number of pawns that can reach `AttackerSite`, other captures are excluded.
- **C5:** Pawn captures onto `VacatedSite` are excluded.
- **C6:** Under check, only moves to `CheckSites` or King moves are allowed; castling is excluded.
    - **C6.1** *(NOT IMPLEMENTED)*: Double check (multiple check directions) — only King moves would be allowed (interposition impossible).
- **C7:** Pawns cannot advance onto terminal `CheckSites`.
- **C8** *(NOT IMPLEMENTED)*: Check triggered by capture — if `AttackerSite` distance from the King > 1, the King cannot move toward it (attacker unreachable).
- **C9** *(NOT IMPLEMENTED)*: Check triggered by capture (non-Knight) — `CheckSites` computed only toward `AttackerSite`, not in the opposite direction from the King.

#### Glossary
- **`Tries`**: Number of legal pawn captures available this turn (announced by the umpire). Includes en passant.
- **`AttackerSite`**: Square where an enemy captured a piece and now occupies; encoded values: `[site]` (one pawn can recapture), `[site+100]` (one pawn can), `[site+200]` (two pawns can).
- **`CheckSites`**: Squares along a check line where interposition is possible; encoded as `[site]` (intermediate) and `[site+100]` (terminal — farthest visible square).
- **`VacatedSite`**: Square emptied by an en passant capture (the captured pawn's original location).
- **`IllegalMoves`**: Memory of moves rejected during the turn (encoded as `[from*100+to]`).
- **`Pinnable`**: A piece is pinnable if it lies on the line between a King and an enemy sliding piece with no friendly pieces in between.
- **`PinRay`**: The ray from the King through the pinned piece to the enemy pinner; a pinned piece is restricted to moves along this ray.

## Umpire Announcements and Rules

The game is played with three boards, one for each player; the third is for the umpire (and spectators). Each opponent knows the exact position of just their own pieces, and does not know where the opponent's pieces are (but can keep track of how many there are). Only the umpire knows the position of the game. The game proceeds in the following way:

The umpire makes the following announcements where appropriate:

- `Player 1 (White)'s [or Player 2 (Black)'s] turn`.
- `Pawn [or Piece] at (square) captured`, when a pawn or a piece is captured. The square of the victim is announced. (En passant captures are specifically announced as such, e.g. `Pawn at A4 captured en passant`)
- `Illegal move` when the attempted move is illegal, given the opponent's position (e.g. moving the king into check; moving a queen, rook, bishop, or pawn through squares occupied by the opponent's pieces; advancing a pawn into a square occupied by the opponent's pieces; castling through check or across occupied squares; moving a piece under an absolute pin).
- `Rank check`.
- `File check`.
- `Long-diagonal check` (the longer of the two diagonals, from the king's point of view).
- `Short-diagonal check` (e.g. for a king on E1, the short diagonal is E1 to H4).
- `Knight check`.
- `(number) tries` (the number of legal pawn-capture moves available in the turn; captures are not obligatory. Displayed in each player's score).
- `Checkmate`, `stalemate`
- `Draw by repetition` when the same position occurs for the third time, the umpire automatically declares the game a draw and the game ends immediately (no claim by the players is required).
- `Draw by insufficient force kings only [or king and bishop vs king; king and knight vs king; kings and bishops on same-colored squares]`.
- `50-move draw` (the illegal moves do not count towards the fifty moves).
- `Player 1 (White)'s [or Player 2 (Black)'s] proposes [or accepted] to end the game` when a player propose/accept to end the game in draw. Proposals are limited to one per player per turn to avoid infinite loops or spam.

Pawn promotions are not announced. The precise location of the checking piece is not announced (although it may be deduced).

Illegal move attempts are not announced to the opponent. As soon as the umpire rejects one it disappears from the mover’s options for the rest of that turn, eliminating wasted retries.

## Files in this repository

- `Kriegspiel (Chess).lud` — The guided Kriegspiel variant with pre-filtering and umpire inference (recommended).
- `LEGACY-Kriegspiel (Chess).lud` — Legacy original Ludii file (included for reference).

## How to Play

1.  Download the **`Kriegspiel (Chess).lud`** game file from this repository.
2.  Launch the Ludii application.
3.  Navigate to `File > Load Game from File (CTRL+F)` and select the downloaded `.lud` file.
4.  Enjoy the game!

**UI Tips in Ludii:**
-  Highlight legal moves for the selected piece: `View > Show Legal Moves` (`Alt+M`).
-  Show the last attempted move: `View > Show Last Move` (`Alt+L`).

## Notes

- Some advanced inference rules are marked as "not implemented" in code comments; they are documented in the game's metadata and can be considered for future extensions (e.g. advanced pin/sonar deductions).

#### Credits
- Game Author: Henry Michael Temple
- Source: https://en.wikipedia.org/wiki/Kriegspiel_(chess)