---
name: chess-automation
version: 1.0.0
description: |
  Automate macOS Chess.app via AppleScript accessibility API. Read board state,
  make moves, track game log, and integrate chess engines for optimal play.
  Use when asked to "play chess", "automate chess", "auto chess", or
  "help me play chess on desktop".
allowed-tools:
  - Bash
  - Read
  - Write
  - mcp__computer-use__screenshot
  - mcp__computer-use__request_access
  - mcp__computer-use__open_application
---

# macOS Chess.app Automation Skill

## Quick Start

```bash
# 1. Request access (required once per session)
# Use mcp__computer-use__request_access with apps: ["Chess"]

# 2. Open Chess
# Use mcp__computer-use__open_application with app: "Chess"

# 3. Start a new game if needed
osascript -e 'tell application "Chess" to activate'
# Menu: Game > New Game
```

## Core Technique: AppleScript Accessibility API

The Chess.app exposes every square as an accessibility button via System Events.
This is the **only reliable** method for making moves.

### Button Naming Convention

| Square State | Button Name Example |
|---|---|
| Occupied by white | `"white queen, d3"` |
| Occupied by black | `"black knight, e4"` |
| Empty square | `"d1"` (just the coordinate) |

### Making a Move (Click-Click)

```applescript
tell application "Chess" to activate
delay 0.3
tell application "System Events"
    tell process "Chess"
        tell group 1 of window 1
            click button "white pawn, e2"   -- click the piece
            delay 0.5
            click button "e4"               -- click the destination
        end tell
    end tell
end tell
```

**Key rules:**
- Source button: use full piece name (`"white rook, a1"`)
- Destination button: use square name if empty (`"d1"`), or full piece name if capturing (`"black knight, e4"`)
- Always `activate` the Chess app before making moves
- Always include `delay 0.5` between clicks
- Castling: click king, then click the king's destination (`"g1"` for kingside, `"c1"` for queenside)

### Reading the Board State

```applescript
tell application "System Events"
    tell process "Chess"
        tell group 1 of window 1
            set allPieces to {}
            repeat with b in (every button)
                set bName to name of b
                if bName contains "white" or bName contains "black" then
                    set end of allPieces to bName
                end if
            end repeat
        end tell
    end tell
end tell
return allPieces
```

Returns comma-separated list like:
```
white rook, a1, white knight, b1, ..., black king, g8
```

### Reading the Window Title (Turn Indicator)

```applescript
tell application "System Events"
    tell process "Chess"
        set winTitle to name of window 1
    end tell
end tell
return winTitle
-- Example: "Game 1 | charles qin - Computer   (White to Move)"
-- Or: "Game 1 | charles qin - Computer   (Black wins!)"
```

### Game Log (Move History)

```applescript
-- Open/close game log panel
tell application "System Events"
    tell process "Chess"
        keystroke "l" using command down
    end tell
end tell
```

### Take Back a Move

```applescript
tell application "System Events"
    tell process "Chess"
        click menu item "Take Back Move" of menu "Moves" of menu bar 1
    end tell
end tell
```

Note: This undoes BOTH the player's move AND the computer's response.

### Request a Hint

```applescript
tell application "System Events"
    tell process "Chess"
        keystroke "]" using command down
    end tell
end tell
```

## Critical Gotchas & Lessons Learned

### 1. Game Log Panel Breaks Click Input

**Problem:** Opening the Game Log panel (`Cmd+L`) can cause the click-click
method to silently stop working. Moves appear to execute (no errors) but the
board state doesn't change.

**Fix:** Close the game log with `Cmd+L` before making moves.
If clicks still don't work after closing, use **Take Back Move** to reset the
app's input state, then replay from there.

### 2. Never Use Raw Screen Coordinates for Moves

**Problem:** The 3D perspective board makes pixel-coordinate dragging unreliable.
Coordinates shift when:
- The game log panel opens/closes (resizes the board)
- The window is moved or resized
- The board animation is playing

**Fix:** Always use the **AppleScript accessibility button API** (click-click method).
It addresses squares by name, not coordinates.

### 3. Coordinate System Mismatch (Accessibility vs Screenshot)

If you need screen coordinates (e.g., for `mcp__computer-use__left_click_drag`):

| System | Origin | Scale |
|---|---|---|
| System Events (accessibility) | macOS points (logical) | ~2056x1329 on Retina |
| computer-use screenshot | Pixel in captured image | ~1366x868 |
| Physical display | Hardware pixels | 3456x2234 |

The accessibility coordinates do NOT directly map to screenshot coordinates.
**Do not mix them.** Use accessibility buttons for moves, screenshots only for visual verification.

### 4. Wait for Computer's Response

Always `sleep 3` (or poll) after making a move before reading the board state.
The computer needs time to calculate and animate its response.

```bash
sleep 3 && osascript -e '... read board state ...'
```

### 5. Captures Require Correct Button Name

When capturing, the destination button is named after the **enemy piece**:

```applescript
-- CORRECT: capture using the piece's button name
click button "white queen, f3"
delay 0.5
click button "black knight, e4"    -- destination has a piece

-- WRONG: using just the square name for occupied squares
click button "e4"    -- this is the EMPTY square button, not the knight
```

### 6. Pawn Captures Check Legality

Before attempting pawn captures, verify the move is legal. A pawn on f3 can
capture on e4 diagonally, but your OWN pieces (e.g., a pawn on f2) may block
rook movement along ranks. The Chess app silently rejects illegal moves.

### 7. Full Move Automation Loop

```bash
# Pattern for continuous auto-play
while true; do
    # 1. Read board state
    STATE=$(osascript -e '...read pieces...')
    
    # 2. Check if game is over
    TITLE=$(osascript -e '...read window title...')
    if echo "$TITLE" | grep -q "wins"; then break; fi
    
    # 3. Calculate best move (use Stockfish or opening theory)
    # 4. Execute move via click-click
    # 5. Wait for computer response
    sleep 3
done
```

## Chess Engine Integration (Recommended)

Install Stockfish for optimal play:

```bash
brew install stockfish
```

### Convert Board State to FEN

Parse the accessibility button output into FEN notation, then feed to Stockfish:

```bash
echo "position fen <FEN_STRING>
go movetime 2000
quit" | stockfish
```

The engine returns the best move in UCI format (e.g., `bestmove e2e4`).

### UCI Move to AppleScript

Map UCI moves (e.g., `e2e4`) to button clicks:

| UCI | Source Button | Dest Button |
|---|---|---|
| `e2e4` | `"white pawn, e2"` | `"e4"` |
| `g1f3` | `"white knight, g1"` | `"f3"` |
| `e1g1` | `"white king, e1"` | `"g1"` (castling) |

## Opening Theory Quick Reference

For play without an engine, these are solid openings:

**As White (Ruy Lopez):**
1. e4 e5 2. Nf3 Nc6 3. Bb5 a6 4. Ba4 Nf6 5. O-O b5 6. Bb3 Be7 7. Re1 O-O 8. c3 d6 9. h3

**Key principles:**
- Develop knights before bishops
- Castle early (before move 10)
- Control the center with pawns (e4, d4)
- Don't move the queen early
- Don't leave bishops exposed to pawn attacks (retreat to safety: Bb5 > Ba4 > Bb3 > Bc2)

## Debugging Checklist

When moves stop working:

1. Is Chess.app frontmost? Run `tell application "Chess" to activate`
2. Is the game log panel open? Close it with `Cmd+L`
3. Is it actually your turn? Check window title for "(White to Move)"
4. Is the game over? Check for "wins!" or "Draw" in title
5. Is the move legal? Read board state and verify piece positions
6. Try **Take Back Move** to reset input state
7. As last resort: start a new game (`Game > New Game`)
