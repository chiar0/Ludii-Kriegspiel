# Kriegspiel (Ludii)

Kriegspiel is imperfect-information chess with an umpire. Each player sees only their own pieces, tries a move, and receives `legal` / `illegal` arbitration plus check/capture announcements.

This project refactors Kriegspiel into a filtered interface that only exposes moves that remain plausible under the player known state. That reduces wasted search on impossible actions and matches the way a strong human player reasons about the game.

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

## Description

This implementation pre-filters moves that fail feasibility checks, eliminating `Hell no` / `Impossible` / `Nonsense` announcements when the attempted move is always illegal regardless of the opponent's hidden pieces. Every filter uses only information the mover can legitimately derive from game rules, their own pieces, and prior umpire announcements.

### FILTERING RULES

#### A — Memory
- **A1:** Rejected moves remain excluded for the rest of the turn.
- **A2:** Sliding trajectories stop at previously rejected destinations, except while in check.

#### B — Geometry
- **B1:** Pieces keep canonical chess movement.
- **B2:** Sliding moves stop before friendly pieces.
- **B3:** Castling needs unmoved King/Rook and clear intermediate squares.
- **B4:** Early-game pawn captures are restricted by reachability.
    - **B4.1 (Turns 1–2):** A pawn on its starting rank cannot make a capture (no enemy piece can yet occupy ranks 3 or 6).
    - **B4.2 (Turns 3–4):** A pawn on its starting rank may capture only on `A3`, `H3`, `A6`, or `H6` (squares reachable by a bishop in that window).

#### C — Umpire inference
- **C1:** Pawn captures are enabled only when `Tries` > 0.
- **C2:** Sliding pieces stop at `AttackerSite`.
- **C3:** Pawns cannot advance onto `AttackerSite`.
- **C4:** If `Tries` equals the number of pawns that can reach `AttackerSite`, other captures are excluded.

#### Glossary
- **`Score` (`Tries`)**: Number of legal pawn captures this turn, including en passant.
- **`AttackerSite`**: Square occupied by the captured enemy piece, or inferred from particular rejected moves. Encoded as `site`, `site+100` (one pawn can recapture), or `site+200` (two pawns can recapture).
- **`PinnedPieces`**: Memory of hypothetically pinned pieces, encoded as `direction*100+site`, where direction 1-8 is N, S, E, W, NE, NW, SE, SW.
- **`PinRay`**: The ray from the king through a pinned piece toward the pinner.
- A piece is pinned when it lies on a king-to-enemy slider line with no friendly pieces in between; a valid pin also needs at least one empty square beyond the piece along that ray.
- A pinned piece may move along its pin axis, but not perpendicular to it.
- **`CheckSites`**: Candidate squares for interposition or capture to resolve a check. Encoded as `site` (intermediate) and `site+100` (terminal). For sliding checks, the ray is projected from the king until a known `AttackerSite`, a blocking friendly piece, or the board edge; for knight checks, only the capture square is recorded when applicable.
- **`VacatedSite`**: Square emptied by an en passant capture.
- **`IllegalMoves`**: Rejected moves remembered for the current turn, encoded as `from*100+to`.
- Note: Additional implausible-moves can be derived from game state and umpire announcements. The ones documented here are not exhaustive.

### Extensions

These remain documented for future work, even though they are not part of the current implemented framework:

— **PIN DETECTION** *(NOT IMPLEMENTED)*: deductions from rejected moves when a piece is *pinnable*.
  - A sliding piece move rejected means the piece is likely pinned. An enemy exists along the `PinRay`. Restrict the piece to moves along that ray.
    - If only one square is visible along the `PinRay` away from the King, mark that square as `AttackerSite` (apply C2–C4).
  - Pawn capture rejected when C4 applies means the pawn is likely pinned. Freeze pawn moves outside the `PinRay`.

### Map

| Concept | Code anchor | Purpose |
| --- | --- | --- |
| `AttackerSite` | `SetAttackerSite`, `AttackerSiteAt`, `CheckLineSites` | Encode inferred or captured attacker squares, including recapture counts. |
| `CheckSites` | `CheckType`, `RecordDiagonalCheck`, `RecordOrthogonalCheck`, `RecordKnightCheck` | Build the legal reply set when the mover is in check. |
| `PinnedPieces` | `GetPinDirection`, `UpdatePinStatus`, `PinnedPawnMoveAllowed` | Track absolute pins and keep stale pin state out of the move filter. |
| `IllegalMoves` | `LegalMove`, `PerformLegalMove` | Remove already-rejected attempts from the remainder of the turn. |
| `VacatedSite` | `PerformLegalMove` | Block pawn captures onto the square emptied by en passant. |
| `Tries` | `CountTries`, `CanPawnCapture`, `PawnCaptureDirectionAllowed` | Count legal pawn captures and gate pawn-capture filtering. |
| `King safety` | `KingInCheck`, `KingMovement`, `SafePassingLocation` | Enforce check, adjacency, and castling constraints. |

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

#### Credits
- Game Author: Henry Michael Temple
- Source: https://en.wikipedia.org/wiki/Kriegspiel_(chess)
