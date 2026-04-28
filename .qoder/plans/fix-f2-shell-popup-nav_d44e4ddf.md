# Fix F2 Footer Navigation: Shell Popup + Left/Right Keys

## Root Causes

### Issue 1: F2 blocked when shell popup is open
In [handle_key](file://src/tui/app.rs#L2729-L2748), the F2 guard at line 2732 checks `self.state.shell_popup.is_none()`. When the shell popup (Claude Code in tmux) is open, F2 is swallowed silently (returns Ok but does nothing). The shell popup's own key handler at `handle_shell_popup_key` has no footer navigation support. The shell popup footer is rendered as plain text — not interactive FooterItems with click regions.

### Issue 2: Left/Right in footer nav mode not working reliably
The footer nav interception at lines 2750-2777 looks correct and unit tests pass. However, there may be edge cases where `footer_items` is empty at the moment of keypress (e.g., stale state between draw frames), or the shell popup handler at line 2833-2834 silently consumes keys before the footer nav block runs. The plan hardens this by ensuring footer nav is checked/active within the shell popup context too.

---

## Task 1: Allow F2 in shell popup — modify `handle_key` guard

**File**: `src/tui/app.rs`

Remove `self.state.shell_popup.is_none()` from the `no_popup` check in the F2 handler (line 2732). Reason: the shell popup handler will manage footer nav locally; F2 must be allowed to reach it.

Replace:
```rust
let no_popup = self.state.shell_popup.is_none()
    && self.state.pr_confirm_popup.is_none()
    && self.state.diff_popup.is_none()
    && self.state.task_search.is_none()
    && self.state.plugin_select_popup.is_none()
    && self.state.move_confirm_popup.is_none()
    && self.state.done_confirm_popup.is_none()
    && self.state.delete_confirm_popup.is_none()
    && self.state.review_confirm_popup.is_none();
```
With the same list but WITHOUT `shell_popup.is_none()`.

Also remove `shell_popup.is_none()` from the footer-nav-active check condition by moving the footer nav block BEFORE the popup dispatch chain (lines 2779-2835), so that when `footer_nav_active` is true, Left/Right/Enter/Esc are intercepted regardless of popup state. Only the shell popup will additionally handle these in its own handler.

**Verification**: `cargo check`

---

## Task 2: Add shell-popup footer items builder

**File**: `src/tui/app.rs`

Add `build_shell_popup_footer_items()` returning `Vec<FooterItem>` with items for the shell popup's footer actions. Each item maps a visible label to a Ctrl+key trigger that `handle_shell_popup_key` already handles:

```
Label                      Trigger
"[C-j]↓"      → Ctrl+j   (scroll down)
"[C-k]↑"      → Ctrl+k   (scroll up)
"[C-d]PgDn"   → Ctrl+d   (page down)
"[C-u]PgUp"   → Ctrl+u   (page up)
"[C-g]End"    → Ctrl+g   (go to bottom)
"[C-f]Full"   → Ctrl+f   (fullscreen attach)
"[C-q]Close"  → Ctrl+q   (close popup)
```

**Verification**: `cargo check`

---

## Task 3: Add F2 + footer nav handling in `handle_shell_popup_key`

**File**: `src/tui/app.rs`

Modify `handle_shell_popup_key` (line 3396) to:

1. If escalation note is showing, dismiss it (existing behavior) but do NOT dismiss for F2/Left/Right/Enter/Esc when footer nav is active.
2. If F2 is pressed: toggle `self.state.footer_nav_active` and populate `self.state.footer_items` with `build_shell_popup_footer_items()`. Return Ok.
3. If `self.state.footer_nav_active` is true:
   - Left/Right/h/l: navigate `footer_nav_index` within `footer_items`
   - Enter: deactivate nav, lookup selected item, convert trigger to a simulated key press and re-enter `handle_shell_popup_key` with it
   - Esc: deactivate nav
   - All other keys: forward to tmux as usual
4. Otherwise: existing behavior (Ctrl+q to close, Ctrl+j/k to scroll, etc.)

**Verification**: `cargo build`

---

## Task 4: Render shell popup footer as interactive items when nav is active

**File**: `src/tui/shell_popup.rs`, `src/tui/app.rs`

Modify `render_shell_popup` to accept an optional `&[FooterItem]` + `nav_active: bool` + `nav_index: usize`. When footer nav is active, render the shell popup footer using `build_and_render_footer` with the shell popup footer items (instead of plain text). This gives visual highlight feedback and registers click regions.

Also update `draw_shell_popup` in `app.rs` to pass these new parameters, and update the click_regions writeback in `draw()` to merge shell popup footer regions.

**Verification**: `cargo build`

---

## Task 5: Ensure board footer items are populated when shell popup is open

**File**: `src/tui/app.rs`

In `draw_board`, when the shell popup is active, still build the shell-popup-specific footer items and store them in `state.footer_items` so that the `handle_key` footer nav block can use them. Modify `App::draw()` to conditionally use `build_shell_popup_footer_items()` when shell popup is open and `footer_nav_active` is true.

**Verification**: `cargo build`

---

## Task 6: Run tests and verify

Run the full test suite:
```
cargo test --features test-mocks
```

Fix any test failures. The existing `test_f2_ignored_when_shell_popup_open` test expects F2 to be blocked — update it to expect F2 to work (toggle footer nav) when shell popup is open.

**Verification**: All tests pass.

---

## Task 7: Manual integration test checklist
- Open agtx, press Enter on a task to open the shell popup (Claude Code in tmux)
- Press F2 — verify the shell popup footer items highlight
- Press Left/Right — verify navigation moves between footer items
- Press Enter on "[C-q]Close" — verify the popup closes
- Press Enter on "[C-f]Full" — verify fullscreen attach works
- Press Esc — verify footer nav deactivates without action
- Without popup open: press F2, navigate left/right, verify board footer items highlight and work
