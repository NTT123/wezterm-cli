# Bookmark-Based Buffer Clearing (SetMark / ClearToMark)

## Overview

This feature implements iTerm2-compatible `SetMark` and `ClearToMark` escape sequences that allow terminal applications to:

1. **Set a bookmark** that snapshots the current screen state
2. **Restore to that snapshot** when clearing, discarding all changes made since

This is useful for shell integrations and terminal applications that want to clear command output while preserving previous content, similar to the "clear to start of command" functionality in modern shells.

## Escape Sequences

| Sequence | Description |
|----------|-------------|
| `\e]1337;SetMark\a` | Snapshots the current screen state (buffer + cursor) |
| `\e]1337;ClearToMark\a` | Restores the screen to the saved snapshot |

Both sequences use the OSC (Operating System Command) format with the iTerm2 proprietary prefix `1337`.

## Design

### Screen Snapshot Approach

The bookmark stores a complete snapshot of the terminal state:

```rust
struct Bookmark {
    /// Cloned screen buffer (includes all scrollback + visible lines)
    screen: Screen,

    /// Cursor position at bookmark time
    cursor: CursorPosition,

    /// Wrap pending state
    wrap_next: bool,
}
```

**Key design decisions:**

1. **Full state snapshot**: Clones the entire screen buffer, cursor position, and wrap state. This provides a clean "save state / restore state" mechanism.

2. **Resize handling**: If the terminal was resized between `SetMark` and `ClearToMark`:
   - The snapshot is restored first
   - Then resized to the current terminal dimensions
   - Cursor is clamped to valid range

3. **Scrollback preservation**: The snapshot contains all scrollback that existed at bookmark time. Even if current scrollback has been purged, restoring brings back the original state.

4. **Single bookmark**: Only one bookmark is stored at a time. Calling `SetMark` multiple times overwrites the previous snapshot.

5. **Bookmark consumption**: `ClearToMark` consumes (removes) the bookmark after use. To restore to the same state again, `SetMark` must be called again.

### Why Snapshot vs Position-Based

| Aspect | Position-Based | Snapshot-Based |
|--------|---------------|----------------|
| Resize handling | Complex recalculation | Just restore then resize |
| Scrollback purge | Track purged lines | Irrelevant - restore old state |
| Buffer integrity | Risk of corruption | Clean swap |
| Code complexity | High | Low |
| Memory usage | Minimal | Higher (full buffer clone) |

The snapshot approach was chosen for its simplicity and robustness. Memory usage is acceptable for a single bookmark.

### Restore Behavior

When `ClearToMark` is executed:

1. **Bookmark exists**:
   - Restore the saved screen buffer
   - If dimensions changed, resize to current dimensions
   - Restore cursor position (clamped to valid range)
   - Restore wrap state

2. **No bookmark set**: No-op (does nothing)

## Implementation

### Files Modified

| File | Purpose |
|------|---------|
| `wezterm-escape-parser/src/osc.rs` | Parse `SetMark` and `ClearToMark` escape sequences |
| `term/src/terminalstate/mod.rs` | `Bookmark` struct, `set_bookmark()`, `clear_to_bookmark()` |
| `term/src/terminalstate/performer.rs` | Wire escape sequences to terminal state methods |
| `term/src/test/mod.rs` | Comprehensive test suite |

### Key Functions

#### `set_bookmark()` (terminalstate/mod.rs)

```rust
pub(crate) fn set_bookmark(&mut self) {
    log::debug!(
        "set_bookmark: cursor=({}, {}), screen_lines={}",
        self.cursor.x,
        self.cursor.y,
        self.screen().scrollback_rows()
    );

    self.bookmark = Some(Bookmark {
        screen: self.screen().clone(),
        cursor: self.cursor,
        wrap_next: self.wrap_next,
    });
}
```

Creates a snapshot of the current screen state by cloning the screen buffer and saving cursor/wrap state.

#### `clear_to_bookmark()` (terminalstate/mod.rs)

```rust
pub(crate) fn clear_to_bookmark(&mut self) {
    let bookmark = match self.bookmark.take() {
        Some(b) => b,
        None => return,  // No-op if no bookmark
    };

    // Get current screen dimensions
    let current_cols = self.screen().physical_cols;
    let current_rows = self.screen().physical_rows;
    let current_dpi = self.screen().dpi;

    // Restore screen state
    *self.screen_mut() = bookmark.screen;

    // If dimensions changed, rewrap to current size
    if self.screen().physical_cols != current_cols
        || self.screen().physical_rows != current_rows
    {
        let size = TerminalSize { rows: current_rows, cols: current_cols, ... };
        let new_cursor = self.screen_mut().resize(size, bookmark.cursor, ...);
        self.cursor = new_cursor;
    } else {
        self.cursor = bookmark.cursor;
    }

    // Clamp cursor to valid range
    self.cursor.y = self.cursor.y.min((current_rows - 1) as i64).max(0);
    self.cursor.x = self.cursor.x.min(current_cols.saturating_sub(1));

    self.wrap_next = bookmark.wrap_next;
}
```

Restores the saved snapshot and handles dimension changes.

## Usage

### Basic Usage

```bash
# Set a mark (snapshot current state)
printf '\e]1337;SetMark\a'

# ... do some work ...

# Restore to the snapshot
printf '\e]1337;ClearToMark\a'
```

### Shell Integration Example

A common use case is clearing command output in shell prompts:

```bash
# In your shell's precmd/preexec hooks:

# Before command execution - snapshot the state
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

# Snapshot before command
printf '\e]1337;SetMark\a'

# Run a command with lots of output
ls -la /

# Restore to snapshot, discarding ls output
printf '\e]1337;ClearToMark\a'
```

**Result:** Screen is restored to show just "Line 1" with cursor at the mark position.

### Example: SetMark Override

```bash
echo "Line 1"
printf '\e]1337;SetMark\a'    # First snapshot (shows "Line 1")
echo "Line 2"
printf '\e]1337;SetMark\a'    # Second snapshot (shows "Line 1" and "Line 2")
echo "Line 3"
printf '\e]1337;ClearToMark\a'  # Restores to second snapshot
```

**Result:** "Line 1" and "Line 2" are visible, "Line 3" is gone.

### Example: Scrollback Preservation

```bash
# Generate scrollback content
for i in $(seq 1 100); do echo "Scrollback line $i"; done

# Snapshot (includes all 100 lines)
printf '\e]1337;SetMark\a'

# Add more content
echo "This will be discarded"
echo "This too"

# Restore snapshot
printf '\e]1337;ClearToMark\a'

# Scroll up - all 100 "Scrollback line N" entries are restored
```

### Example: Cursor Positioning

```bash
echo -n "Before mark"
printf '\e]1337;SetMark\a'
echo "After mark"
printf '\e]1337;ClearToMark\a'
echo "New content"
```

**Result:** Output is `Before markNew content` - the cursor returns to the snapshot position.

## Testing

### Automated Tests

The implementation includes 13 comprehensive tests in `term/src/test/mod.rs`:

| Test | Description |
|------|-------------|
| `test_set_mark_clear_to_mark_basic` | Basic snapshot and restore |
| `test_set_mark_with_newlines` | Snapshot with multi-line content |
| `test_clear_to_mark_without_mark` | ClearToMark without SetMark (no-op) |
| `test_set_mark_replacement` | Second SetMark replaces first snapshot |
| `test_set_mark_with_scrollback` | Snapshot includes scrollback content |
| `test_clear_to_mark_preserves_scrollback` | Scrollback restored from snapshot |
| `test_cursor_position_after_clear_to_mark` | Cursor restored to snapshot position |
| `test_bookmark_survives_resize` | Snapshot restored then resized to current dimensions |
| `test_bookmark_wrapped_lines_resize` | Wrapped content adapts to new dimensions |
| `test_bookmark_resize_narrower_then_wider` | Multiple resize operations handled |
| `test_bookmark_scrollback_purged` | Snapshot preserves state even if current scrollback purged |
| `test_bookmark_resize_then_scroll` | Resize + scroll handled correctly |
| `test_bookmark_logical_line_with_multiple_wraps` | Long wrapped lines adapt to new width |

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
set_bookmark: cursor=(x, y), screen_lines=N
clear_to_bookmark: restoring to cursor=(x, y), screen_lines=N
clear_to_bookmark: dimensions changed from WxH to WxH, resizing
clear_to_bookmark: restored cursor=(x, y), screen_lines=N
```

Enable debug logging with `RUST_LOG=debug` to see these messages.

## Compatibility

This implementation is compatible with iTerm2's `SetMark` escape sequence. The `ClearToMark` sequence is a WezTerm extension that provides the restore functionality.

| Terminal | SetMark | ClearToMark |
|----------|---------|-------------|
| iTerm2 | Yes | No |
| WezTerm | Yes | Yes |

## Edge Cases

1. **No bookmark set**: `ClearToMark` is a no-op
2. **Multiple SetMark calls**: Last one wins (overwrites previous snapshot)
3. **Terminal resized**: Snapshot restored then resized to current dimensions
4. **Scrollback purged in current screen**: Snapshot still has original state
5. **Cursor out of bounds after restore**: Clamped to valid range

## Memory Considerations

The snapshot approach clones the entire screen buffer. For a terminal with 10,000 lines of scrollback at 200 columns:
- ~2M cells x ~16 bytes/cell = ~32MB per snapshot

This is acceptable for a single bookmark. If multiple bookmarks were needed in the future, consider:
- Limiting scrollback in snapshots
- Copy-on-write data structures

## Future Considerations

- Support for multiple named bookmarks
- Bookmark persistence across sessions
- Integration with semantic zones/prompts
- Visual indicator for bookmark position
