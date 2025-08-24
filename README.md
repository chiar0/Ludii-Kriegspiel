# Kriegspiel Refactoring for Ludii

While Ludii's original implementation of Kriegspiel correctly models the game's core rules of umpire arbitration, its unconstrained design poses a significant challenge for artificial intelligence. The model presents the AI with a raw action space, allowing it to explore and attempt moves that are geometrically impossible (e.g., a Rook moving diagonally).

This is highly inefficient for any search algorithm. The AI wastes valuable computational resources evaluating moves that a human player would instantly recognize as irrational. Furthermore, a rational player would not repeatedly attempt the same illegal move after it has been rejected by the umpire within the same turn. This behavior, common in the default AI, often leads to games ending in draws by the 50-move limit.

This project refactors Kriegspiel to address these issues by creating a **guided interface** that pre-filters the action space. By only presenting geometrically valid moves, we align the AI's decision-making process more closely with rational human play.

The impact of this change is significant. This refactoring has already enabled a subset of MCTS-based agents—specifically Bandit Tree Search (Avg), EPT, and EPT-QB—to produce meaningful, **non-zero value estimates**, a critical step toward developing truly competitive Kriegspiel agents.

## Game Versions

You will find three `.lud` files in this repository, each representing a different stage of development.

### 1. `Kriegspiel (Chess)(Refactored-Stable).lud` - ✅ **Recommended Version**

This is the **stable, fully playable, and recommended version** for all users. It offers a superior gameplay experience compared to the original game file.

**Key Features & Differences from the Original:**

*   **Guided User Interface:** The original version allowed players to attempt a move from any of their pieces to any of the 64 squares not occupied by friendly pieces, leaving a vast number of geometrically and logically impossible moves. This refactored version introduces a "guided" UI that **pre-filters moves**, only presenting the player with geometrically plausible and contextually valid options. Some specific improvements include:

    *   **Sliding Pieces (Rook, Bishop, Queen):** The UI now highlights a clear path. This path extends along valid lines (orthogonal/diagonal) and stops at the first piece encountered. The UI does not allow selecting squares occupied by friendly pieces or beyond the first encountered.
    *   **Pawns:** The UI is context-aware. The initial two-square advance is **only offered as an option for pawns on their starting rank**.
    *   **Castling Handling:** The UI displays castling as a distinct two-square King move. This option is **only made available to the player when all preconditions are met**: neither the King nor the chosen Rook has moved, the squares between them are empty, and the King does not pass through or land on a square under attack.

*   **Complete Draw Rules:** This version correctly implements the **draw by insufficient material** rule. This includes:
    *   King vs. King
    *   King vs. King + Bishop
    *   King vs. King + Knight
    *   King + Bishop vs. King + Bishop (where bishops are on same-colored squares).

**Limitation:** A player can still attempt the same geometrically valid but illegal move multiple times in a single turn (e.g., trying to capture with a pawn on an empty square). This version does not prevent this.

### 2. `Kriegspiel (Chess)(Refactored-Unstable-Blacklist).lud` - ⚠️ **Experimental**

This is an **unstable, experimental version intended for debugging and development purposes only**. It is **not recommended for gameplay**.

This version builds upon the `Refactored-Stable` file by introducing a "blacklist" system to track illegal move attempts.

**Intended Functionality:**
-   When a player attempts a move that is declared "Illegal" by the umpire, that specific move (e.g., A2 to B3) is added to a temporary blacklist for the current player's turn.
-   The UI should then dynamically filter its move generation, hiding the blacklisted move from the player's options.
-   This list is cleared after a successful legal move, as the game state change may render previously illegal moves valid.

**Known Issue:**
This feature is currently not fully functional due to a bug in the state and expression evaluation. When an illegal move is attempted, the `(remember Value ...)` function fails to store the correct ID of the failed move. Instead, it often stores an incorrect value, a stale value from a previous attempt, or `-1`. This makes the blacklist ineffective and prevents the UI from correctly filtering moves.

> **Note for Developers:** This file contains extensive (but commented-out) debug logging macros (`DebugLegalMove`) used to diagnose this issue. It serves as the primary test case for the bug report submitted to the Ludii development team. For more information, please refer to the related [GitHub Issue](https://github.com/Ludeme/Ludii/issues/35).

### 3. `Kriegspiel (Chess).lud` -  archive **Legacy Version**

This is the original, unmodified Kriegspiel game file as found in the official Ludii library. It is included for reference and comparison purposes.

---

## How to Play

1.  Download the desired `.lud` game file from this repository. For the best experience, choose **`Kriegspiel (Refactored-Stable).lud`**.
2.  Launch the Ludii application.
3.  Navigate to `File > Load Game from File (CTRL+F)` and select the downloaded `.lud` file.
4.  Enjoy the game!