# Chess Automation

Automate macOS Chess.app with Claude Code. Uses the AppleScript accessibility API to read board state, make moves, and play full games against the built-in computer opponent.

## How It Works

macOS Chess.app exposes every square on the board as an accessibility button through System Events. This skill leverages that API to:

1. **Read the board** — enumerate all 64 buttons, filter occupied squares, get a machine-readable position
2. **Make moves** — click the source piece button, then click the destination button
3. **Track the game** — read the window title for turn/result, open the game log for move history
4. **Play optimally** — integrate Stockfish for best-move calculation

No pixel coordinates, no image recognition, no fragile screen scraping. Button names like `"white knight, f3"` work regardless of window size, board theme, or 3D perspective angle.

## Install

Copy the skill into your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/chess-automation
cp SKILL.md ~/.claude/skills/chess-automation/
```

Or clone this repo:

```bash
git clone https://github.com/win4r/chess-automation.git ~/.claude/skills/chess-automation
```

For stronger play, install Stockfish:

```bash
brew install stockfish
```

## Usage

In Claude Code, just say:

```
play chess for me
```

```
automate the chess game on my desktop
```

```
help me beat the computer at chess
```

Claude will request access to Chess.app, read the board, and start making moves.

## Core API

### Make a Move

```applescript
tell application "Chess" to activate
delay 0.3
tell application "System Events"
    tell process "Chess"
        tell group 1 of window 1
            click button "white pawn, e2"
            delay 0.5
            click button "e4"
        end tell
    end tell
end tell
```

### Read the Board

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
-- Returns: "white rook, a1, white knight, b1, ..., black king, g8"
```

### Check Turn / Game Result

```applescript
tell application "System Events"
    tell process "Chess"
        set winTitle to name of window 1
    end tell
end tell
-- "(White to Move)" or "(Black wins!)"
```

### Castling

```applescript
-- Kingside: click king, then g1
click button "white king, e1"
delay 0.5
click button "g1"
```

### Capturing

```applescript
-- Use the enemy piece's full button name as destination
click button "white queen, f3"
delay 0.5
click button "black knight, e4"
```

## Button Naming Convention

| Square State | Button Name |
|---|---|
| White piece | `"white queen, d3"` |
| Black piece | `"black knight, e4"` |
| Empty | `"d1"` |

## Gotchas

| Problem | Cause | Fix |
|---|---|---|
| Moves silently fail | Game Log panel is open | Close with `Cmd+L`, or use Take Back Move to reset |
| Coordinates don't match | Accessibility points vs screenshot pixels are different scales | Never use screen coordinates for moves; use button names only |
| Board state unchanged after move | Computer still thinking/animating | Wait 3 seconds before reading state |
| Capture not registering | Using empty-square name for occupied square | Use full piece name: `"black pawn, e5"` not `"e5"` |
| Illegal move silently rejected | Chess.app doesn't error on bad moves | Read board state to verify legality first |

## Stockfish Integration

Parse the board state into FEN, then ask Stockfish for the best move:

```bash
echo "position fen rnbqkbnr/pppppppp/8/8/4P3/8/PPPP1PPP/RNBQKBNR b KQkq e3 0 1
go movetime 2000
quit" | stockfish
# Output: bestmove e7e5
```

Map UCI moves back to button clicks:

| UCI | Source | Destination |
|---|---|---|
| `e2e4` | `"white pawn, e2"` | `"e4"` |
| `g1f3` | `"white knight, g1"` | `"f3"` |
| `e1g1` | `"white king, e1"` | `"g1"` (castle) |

## Requirements

- macOS with Chess.app (pre-installed)
- Claude Code with computer-use MCP server
- Accessibility permissions for System Events
- Stockfish (optional, for optimal play): `brew install stockfish`

## License

MIT
