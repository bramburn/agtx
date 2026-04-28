Here is the complete, Claude Code–ready implementation plan. Everything below is derived directly from the live codebase read above .

***

## Implementation Plan: Clickable Footer + F2 Keyboard Navigation

### Architecture Decision: Why Mouse Works Here

`crossterm` 0.29 (already in your `Cargo.toml`)  includes `EnableMouseCapture` / `DisableMouseCapture` and the `Event::Mouse` variant natively — **zero new dependencies required**. Mouse VT sequences are raw bytes that travel transparently through SSH's PTY layer, making them work perfectly from Windows Terminal → macOS host, unlike `Ctrl+` key encoding which breaks at the SSH layer.

The existing `App::run()` event loop  already has the correct pattern — you just need to add `Event::Mouse` alongside the existing `Event::Key` branch.

***

## File/Directory Tree of Changes

```
agtx/
├── Cargo.toml          ← NO CHANGE (crossterm 0.29 already has mouse)
└── src/
    └── tui/
        ├── app.rs      ← PRIMARY FILE (~200 lines added/modified)
        │   ├── [NEW]    FooterItem struct
        │   ├── [NEW]    ClickRegion struct
        │   ├── [MODIFY] AppState — 4 new fields
        │   ├── [NEW]    build_footer_items() replaces build_footer_text()
        │   ├── [NEW]    build_and_render_footer() helper
        │   ├── [MODIFY] App::with_ops() — EnableMouseCapture + panic hook
        │   ├── [MODIFY] App::run() — Event::Mouse branch
        │   ├── [MODIFY] draw_board() — returns (Vec<ClickRegion>, Vec<FooterItem>)
        │   ├── [MODIFY] App::draw() — writes back regions to state
        │   ├── [NEW]    App::handle_mouse_click()
        │   ├── [MODIFY] App::handle_key() — F2 intercept at top
        │   ├── [NEW]    impl Drop for App — DisableMouseCapture teardown
        │   └── [MODIFY] new_for_test() — 4 new state fields
        └── app_tests.rs ← [ADD] footer_nav_tests module
```

***

## New Data Structures

```rust
/// One item in the footer — maps display label to a simulated KeyEvent.
/// Rebuilt every draw frame; x is absolute terminal column (set at render time).
#[derive(Debug, Clone)]
struct FooterItem {
    label: String,                       // " [o] new "
    trigger: crossterm::event::KeyEvent, // KeyEvent{Char('o'), NONE}
    x: u16,                              // absolute terminal column
    width: u16,                          // character width of label
}

/// A rectangular clickable region, rebuilt every frame during draw_board().
#[derive(Debug, Clone)]
struct ClickRegion {
    area: ratatui::layout::Rect,
    trigger: crossterm::event::KeyEvent,
}
```

Add to `AppState` (after `warning_message`):
```rust
click_regions:     Vec<ClickRegion>,
footer_nav_active: bool,
footer_nav_index:  usize,
footer_items:      Vec<FooterItem>,
```

***

## Sequence Diagram — Mouse Click

```
Windows Terminal (SSH) — User clicks "[o] new"
        │
        ▼ raw VT mouse bytes → PTY → crossterm::event::read()
Event::Mouse { kind: Down(Left), column: 5, row: 23 }
        │
        ▼
App::run() — Event::Mouse branch (NEW)
        │
        ▼
App::handle_mouse_click(col=5, row=23)
   iterates state.click_regions
   finds ClickRegion { area:{x:4,y:23,w:8,h:1},
                       trigger: KeyEvent{Char('o'), NONE} }
        │ col 5 is inside 4..12 ✓
        ▼
self.handle_key(KeyEvent{Char('o'), NONE})   ← re-enters existing handler
        │
        ▼
handle_normal_key(KeyCode::Char('o'))
   state.input_mode = InputMode::InputTitle ✓
```

***

## Sequence Diagram — F2 Keyboard Navigation

```
User presses F2
        ▼
handle_key: F2 (no popup open, Normal mode)
   state.footer_nav_active = true
   state.footer_nav_index  = 0
   draw() renders item[0] with bg=selected_color ◄── visual feedback

User presses → twice
        ▼
handle_key: Right  → footer_nav_index = 1
handle_key: Right  → footer_nav_index = 2   (item = "[Enter] open")

User presses Enter
        ▼
handle_key: Enter (footer_nav_active = true)
   footer_nav_active = false
   trigger = footer_items[2].trigger = KeyEvent{Enter, NONE}
   → self.handle_key(trigger) → opens selected task ✓
```

***

## Workflow Diagram

```
┌─────────────────────────────────────────────────────────┐
│  App::with_ops() startup                                │
│  + EnableMouseCapture  (NEW)                            │
│  + panic hook → DisableMouseCapture  (NEW)              │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│  App::run() main loop (100ms tick)                      │
│                                                         │
│  ┌── draw() ─────────────────────────────────────────┐  │
│  │  draw_board() → build_and_render_footer()         │  │
│  │    ∟ build_footer_items() → Vec<FooterItem>       │  │
│  │    ∟ renders Spans (highlighted if nav active)    │  │
│  │    ∟ pushes ClickRegion per item                  │  │
│  │  App::draw() writes back click_regions,           │  │
│  │              footer_items to state                │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  ┌── event::poll() ──────────────────────────────────┐  │
│  │  Key(F2)     → toggle footer_nav_active  (NEW)    │  │
│  │  Key(nav)    → Left/Right/Enter/Esc nav  (NEW)    │  │
│  │  Key(other)  → existing dispatch (UNCHANGED)      │  │
│  │  Mouse(Down) → handle_mouse_click() → handle_key  │  │
│  │  Paste       → handle_paste() (UNCHANGED)         │  │
│  └───────────────────────────────────────────────────┘  │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│  Drop impl (teardown)                                   │
│  + DisableMouseCapture  (NEW)                           │
└─────────────────────────────────────────────────────────┘
```

***

## Key Code Snippets

**`App::run()` — add Mouse event branch:**
```rust
Event::Mouse(mouse) => {
    use crossterm::event::{MouseEventKind, MouseButton};
    if mouse.kind == MouseEventKind::Down(MouseButton::Left) {
        self.handle_mouse_click(mouse.column, mouse.row)?;
    }
}
```

**`App::handle_mouse_click()` — hit-test and dispatch:**
```rust
fn handle_mouse_click(&mut self, col: u16, row: u16) -> Result<()> {
    let regions = self.state.click_regions.clone();
    for region in &regions {
        if col >= region.area.x && col < region.area.x + region.area.width
        && row >= region.area.y && row < region.area.y + region.area.height {
            return self.handle_key(region.trigger); // re-enter existing handler
        }
    }
    Ok(())
}
```

**F2 intercept at top of `handle_key()`:**
```rust
pub fn handle_key(&mut self, key: crossterm::event::KeyEvent) -> Result<()> {
    // F2 = toggle footer nav (only in Normal mode, no popup open)
    if key.code == KeyCode::F(2) {
        let no_popup = self.state.shell_popup.is_none()
            && self.state.pr_confirm_popup.is_none()
            && self.state.diff_popup.is_none()
            && self.state.task_search.is_none();
        if no_popup && self.state.input_mode == InputMode::Normal {
            self.state.footer_nav_active = !self.state.footer_nav_active;
            if self.state.footer_nav_active { self.state.footer_nav_index = 0; }
        }
        return Ok(());
    }
    if self.state.footer_nav_active {
        match key.code {
            KeyCode::Left | KeyCode::Char('h') => {
                self.state.footer_nav_index = self.state.footer_nav_index.saturating_sub(1);
                return Ok(());
            }
            KeyCode::Right | KeyCode::Char('l') => {
                let max = self.state.footer_items.len().saturating_sub(1);
                if self.state.footer_nav_index < max { self.state.footer_nav_index += 1; }
                return Ok(());
            }
            KeyCode::Enter => {
                self.state.footer_nav_active = false;
                if let Some(item) = self.state.footer_items.get(self.state.footer_nav_index).cloned() {
                    return self.handle_key(item.trigger);
                }
                return Ok(());
            }
            KeyCode::Esc => { self.state.footer_nav_active = false; return Ok(()); }
            _ => {}
        }
    }
    // ... rest of existing handle_key unchanged
```

***

## Atomic Implementation Checklist (20 steps)

Give this ordered list to Claude Code — each step must `cargo check` clean before the next:

1. **Add `FooterItem` + `ClickRegion` structs** in `app.rs` after line ~400
2. **Add 4 fields to `AppState`** (`click_regions`, `footer_nav_active`, `footer_nav_index`, `footer_items`)
3. **Add same 4 fields** to both `AppState` initialisers (`with_ops` + `new_for_test`) → `cargo check`
4. **Add `make_key()` + `make_ctrl()` helpers** + **`build_footer_items()`** function (all 8 column variants — see full map in plan section 16) → `cargo check`
5. **Add `build_and_render_footer()` helper** function → `cargo check`
6. **Add panic hook** before `enable_raw_mode()` in `with_ops()` → `cargo check`
7. **Add `EnableMouseCapture`** to the `execute!` call in `with_ops()` → `cargo check`
8. **Change `draw_board()` return type** to `(Vec<ClickRegion>, Vec<FooterItem>)` → `cargo check`
9. **Update `App::draw()`** to capture and write back regions/items to `state` → `cargo build`
10. **Replace footer `Paragraph` widget** with `build_and_render_footer()` call inside `draw_board` → `cargo build`
11. **Delete `build_footer_text()`** → `cargo build`
12. **Add `Event::Mouse` branch** in `App::run()` event poll block → `cargo check`
13. **Add `handle_mouse_click()` method** → `cargo build`
14. **Add F2 + nav intercept** at top of `handle_key()` → `cargo build`
15. **Add popup footer click regions** (PR confirm, diff popup footers) → `cargo build`
16. **Add `impl Drop for App`** with `DisableMouseCapture` → `cargo build`
17. **Add `footer_nav_tests` module** to `app_tests.rs` → `cargo test`
18. **Manual test**: run `agtx`, click a footer item, verify action fires
19. **Manual test**: press F2, navigate with arrows, press Enter, verify action fires
20. **Update `README.md`** with Mouse Support + F2 navigation documentation

***

## Critical Gotchas for Claude Code

- **`draw_board` is a `static fn(&AppState)`** — never change it to take `&mut self`. Use the return-value pattern (steps 8–9) to write regions back 
- **`handle_key` is re-entrant safely** — `handle_mouse_click` calls `handle_key` with a simulated event; this is one-level recursion only, not a loop
- **F2 guard**: only activate footer nav when `input_mode == InputMode::Normal` AND no popup is open — check all `Option<XPopup>` fields 
- **`[C-f] fullscreen` item trigger**: `KeyEvent::new(KeyCode::Char('f'), KeyModifiers::CONTROL)` — the existing `handle_key` intercept at line ~2512 already handles this correctly; no special case needed
- **Shell popup content**: do NOT register click regions inside the mirrored tmux pane — only register the popup's own footer bar items