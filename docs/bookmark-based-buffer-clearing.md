# Bookmark-Based Buffer Clearing (SetMark / ClearToMark)

## Overview

This feature implements iTerm2-compatible `SetMark` and `ClearToMark` escape sequences that allow terminal applications to:

1. **Set a bookmark** at the current cursor position
2. **Clear all content** from that bookmark to the end of the buffer

This is useful for shell integrations and terminal applications that want to clear command output while preserving previous content, similar to the "clear to start of command" functionality in modern shells.

## Escape Sequences

| Sequence | Description |
|----------|-------------|
| `\e]1337;SetMark\a` | Sets a bookmark at the current cursor position |
| `\e]1337;ClearToMark\a` | Clears from the bookmark to the end of the buffer |

Both sequences use the OSC (Operating System Command) format with the iTerm2 proprietary prefix `1337`.

## Design

### Bookmark Storage

The bookmark is stored as a `Bookmark` struct containing:

```rust
struct Bookmark {
    stable_row: StableRowIndex,  // Row index that survives scrollback
    col: usize,                   // Column position
}
```

**Key design decisions:**

1. **Stable row index**: Uses `StableRowIndex` instead of physical or visible row index. This ensures the bookmark remains valid even as content scrolls into the scrollback buffer.

2. **Single bookmark**: Only one bookmark is stored at a time. Calling `SetMark` multiple times overwrites the previous bookmark.

3. **Bookmark consumption**: `ClearToMark` consumes (removes) the bookmark after use. To clear to the same position again, `SetMark` must be called again.

### Clear Behavior

When `ClearToMark` is executed:

1. **Bookmark valid**: If the bookmark position still exists in the buffer:
   - Clear from the bookmark column to end of line on the bookmark row
   - Remove all lines after the bookmark row
   - Add blank lines to maintain the visible area size
   - Position cursor at the bookmark location

2. **Bookmark purged**: If the bookmark has been purged from scrollback (due to scrollback limit):
   - Clear all scrollback
   - Clear the entire display
   - Position cursor at (0, 0)

3. **No bookmark set**: If `ClearToMark` is called without a prior `SetMark`:
   - No-op (does nothing)

### Scrollback Preservation

A critical design requirement is that content **before** the bookmark must be preserved. The implementation ensures:

- All lines from index 0 to the bookmark row are kept intact
- Only content from the bookmark position onward is cleared
- The visible area is properly maintained with blank lines after clearing

## Implementation

### Files Modified

| File | Purpose |
|------|---------|
| `wezterm-escape-parser/src/osc.rs` | Parse `SetMark` and `ClearToMark` escape sequences |
| `term/src/terminalstate/mod.rs` | `Bookmark` struct, `set_bookmark()`, `clear_to_bookmark()` |
| `term/src/terminalstate/performer.rs` | Wire escape sequences to terminal state methods |
| `term/src/screen.rs` | `clear_from_phys_row_to_end()` screen operation |
| `term/src/test/mod.rs` | Comprehensive test suite |

### Key Functions

#### `set_bookmark()` (terminalstate/mod.rs)

```rust
pub(crate) fn set_bookmark(&mut self) {
    let stable_row = self.screen().visible_row_to_stable_row(self.cursor.y);
    self.bookmark = Some(Bookmark {
        stable_row,
        col: self.cursor.x,
    });
}
```

Converts the current visible cursor position to a stable row index and stores it with the column.

#### `clear_to_bookmark()` (terminalstate/mod.rs)

```rust
pub(crate) fn clear_to_bookmark(&mut self) {
    let bookmark = match self.bookmark.take() {
        Some(b) => b,
        None => return,  // No-op if no bookmark
    };

    match self.screen().stable_row_to_phys(bookmark.stable_row) {
        None => {
            // Bookmark purged - clear everything
            self.erase_in_display(EraseInDisplay::EraseScrollback);
            self.erase_in_display(EraseInDisplay::EraseDisplay);
            self.cursor.x = 0;
            self.cursor.y = 0;
        }
        Some(phys_row) => {
            // Clear from bookmark to end
            self.screen_mut().clear_from_phys_row_to_end(...);

            // Position cursor at bookmark location
            let visible_row = phys_row.saturating_sub(scrollback) as i64;
            self.cursor.y = visible_row;
            self.cursor.x = bookmark.col;
        }
    }
}
```

#### `clear_from_phys_row_to_end()` (screen.rs)

```rust
pub fn clear_from_phys_row_to_end(
    &mut self,
    phys_row: PhysRowIndex,
    col: usize,
    seqno: SequenceNo,
    blank_attr: CellAttributes,
    bidi_mode: BidiMode,
) {
    // 1. Clear from col to end of line on phys_row
    // 2. Remove all lines after phys_row
    // 3. Calculate target line count to maintain visible area
    // 4. Add blank lines as needed
}
```

### Row Index Types

Understanding the different row index types is crucial:

| Type | Description |
|------|-------------|
| `VisibleRowIndex` | Row relative to visible area (0 = top of screen) |
| `PhysRowIndex` | Absolute index into the line buffer |
| `StableRowIndex` | Monotonically increasing index that survives scrollback |

The bookmark uses `StableRowIndex` for persistence, which is converted to `PhysRowIndex` when clearing.

## Usage

### Basic Usage

```bash
# Set a mark at current position
printf '\e]1337;SetMark\a'

# ... do some work ...

# Clear everything from mark to end
printf '\e]1337;ClearToMark\a'
```

### Shell Integration Example

A common use case is clearing command output in shell prompts:

```bash
# In your shell's precmd/preexec hooks:

# Before command execution - set mark
preexec() {
    printf '\e]1337;SetMark\a'
}

# User can then clear command output with a keybinding that runs:
clear_last_output() {
    printf '\e]1337;ClearToMark\a'
}
```

### Example: Clear Command Output

```bash
# Print some content
echo "Line 1"

# Set mark before command
printf '\e]1337;SetMark\a'

# Run a command with lots of output
ls -la /

# Clear the ls output, keeping "Line 1"
printf '\e]1337;ClearToMark\a'
```

**Result:** "Line 1" remains visible, `ls` output is cleared, cursor is at the mark position.

### Example: SetMark Override

```bash
echo "Line 1"
printf '\e]1337;SetMark\a'    # First mark
echo "Line 2"
printf '\e]1337;SetMark\a'    # Second mark (replaces first)
echo "Line 3"
printf '\e]1337;ClearToMark\a'  # Clears from second mark
```

**Result:** "Line 1" and "Line 2" remain, "Line 3" is cleared.

### Example: Scrollback Preservation

```bash
# Generate scrollback content
for i in $(seq 1 100); do echo "Scrollback line $i"; done

# Set mark
printf '\e]1337;SetMark\a'

# Add content to clear
echo "This will be cleared"
echo "This too"

# Clear to mark
printf '\e]1337;ClearToMark\a'

# Scroll up - all 100 "Scrollback line N" entries are preserved
```

### Example: Cursor Positioning

```bash
echo -n "Before mark"
printf '\e]1337;SetMark\a'
echo "After mark"
printf '\e]1337;ClearToMark\a'
echo "New content"
```

**Result:** Output is `Before markNew content` - the cursor returns to the mark position.

## Testing

### Automated Tests

The implementation includes 7 comprehensive tests in `term/src/test/mod.rs`:

| Test | Description |
|------|-------------|
| `test_set_mark_clear_to_mark_basic` | Basic SetMark + ClearToMark functionality |
| `test_set_mark_with_newlines` | Mark with content spanning multiple lines |
| `test_clear_to_mark_without_mark` | ClearToMark without SetMark (no-op) |
| `test_set_mark_replacement` | Second SetMark replaces first |
| `test_set_mark_with_scrollback` | Mark survives scrolling into scrollback |
| `test_clear_to_mark_preserves_scrollback` | Scrollback before mark is preserved |
| `test_cursor_position_after_clear_to_mark` | Cursor positioned at mark after clear |

Run tests with:

```bash
cargo test -p wezterm-term mark
```

### Manual Testing

Build the GUI for manual testing:

```bash
cargo build -p wezterm-gui --release
./target/release/wezterm-gui
```

Then run the example commands above to verify behavior.

## Debug Logging

The implementation includes debug logging for troubleshooting:

```
set_bookmark: cursor=(x, y), stable_row=N
clear_to_bookmark: bookmark=(stable_row=N, col=M)
clear_to_bookmark: phys_row=N, physical_rows=M, total_lines=L
clear_to_bookmark: new_total=N, scrollback=M, visible_row=V, cursor=(x, y)
```

Enable debug logging with `RUST_LOG=debug` to see these messages.

## Compatibility

This implementation is compatible with iTerm2's `SetMark` escape sequence. The `ClearToMark` sequence is a WezTerm extension that provides the clearing functionality.

| Terminal | SetMark | ClearToMark |
|----------|---------|-------------|
| iTerm2 | Yes | No |
| WezTerm | Yes | Yes |

## Edge Cases

1. **No bookmark set**: `ClearToMark` is a no-op
2. **Bookmark purged from scrollback**: Clears everything, cursor to (0,0)
3. **Multiple SetMark calls**: Last one wins (overwrites previous)
4. **Mark at column 0**: Entire line from mark onward is cleared
5. **Mark mid-line**: Content before mark column is preserved on that line
6. **Empty buffer after clear**: Visible area filled with blank lines

## Future Considerations

- Support for multiple named bookmarks
- Bookmark persistence across sessions
- Integration with semantic zones/prompts
- Visual indicator for bookmark position
