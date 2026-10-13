# claude-terminal-setup

A tmux + Ghostty setup for running several Claude Code sessions side by side, on Linux.

## What's in here

- `tmux.conf`, goes to `~/.tmux.conf`
- `ghostty.config`, goes to `~/.config/ghostty/config`
- `bin/claude-fleet`, launches one tmux window per git worktree, each running `claude`
- `bin/ghostty-theme-cycle`, rotates the Ghostty theme forward or backward
- `bin/tmux-window-bar`, custom-rendered window tab bar, clickable and drag-to-reorder
- `bin/tmux-pane-bar`, custom-rendered pane tab bar, same
- `bin/tmux-pane-bar-move`, `bin/tmux-window-bar-move`, reorder helpers the bars call on drop
- `bin/tmux-bar-drag-update`, `bin/tmux-bar-drag-watchdog`, `bin/tmux-bar-drag-commit`, the drag machinery both bars share
- `bin/tmux-pane-drag`, `bin/tmux-pane-drag-hover`, drag a pane by its title: swap it with another pane, or drop it on the window bar
- `bin/tmux-window-bar-drop`, `bin/tmux-pane-to-window`, a pane (dragged by title or pill) dropped on the window bar moves into that window or opens as a new one
- `bin/tmux-continuum-loop`, runs the tmux-continuum plugin's save check on its own timer instead of on every status redraw
- `bin/tmux-claude-idle-loop`, polls every claude pane and highlights the ones waiting for input, in both bars and as a pane background/border tint
- `bin/tmux-resurrect-save-panenames`, `bin/tmux-resurrect-restore-panenames`, resurrect hooks so pane renames survive a session restore, which resurrect's own save format has no slot for

## Install

```
cp tmux.conf ~/.tmux.conf
mkdir -p ~/.config/ghostty
cp ghostty.config ~/.config/ghostty/config
cp bin/* ~/.local/bin/
chmod +x ~/.local/bin/claude-fleet ~/.local/bin/ghostty-theme-cycle ~/.local/bin/tmux-*
# merge the busy/idle hooks into your Claude Code settings
jq -s ".[0] * .[1]" ~/.claude/settings.json claude-hooks.json > /tmp/claude-settings.json && mv /tmp/claude-settings.json ~/.claude/settings.json
```

Requirements: `tmux` 3.3 or newer (for `range=pane` support in status-format, needed for the clickable pane bar), Ghostty, `xdotool` (theme cycling and window geometry), `xinput` (release detection for drag and drop, see below), `python3` and `xprop` (hover highlight while dragging a pane by its title). `xinput` and `xprop` ship with most X11 desktops already; check with `which xinput xprop`. Dragging a pane by its title also needs `range=control` support in `pane-border-format`; it is tested on tmux 3.7c.

On Ubuntu, tmux in the default apt repos is often older than 3.3. Install a current build instead:

```
sudo snap install tmux --classic
sudo apt remove tmux
```

Removing the apt package matters: if both are installed, whichever comes first on PATH wins, and scripts calling plain `tmux` can end up talking to the wrong binary.

Ghostty and xdotool:

```
sudo snap install ghostty --classic
sudo apt install xdotool
```

Edit the `command =` line in `ghostty.config` to point at whatever directory you want Ghostty to drop you into by default. Also edit the `continuum_save.sh` path in `tmux.conf`'s `status-format[0]` line if `~/.tmux/plugins` is not where TPM installs plugins for you.

## tmux keybindings

Prefix is `Ctrl-a`.

| Key | Action |
|---|---|
| `Ctrl-a \|` | vertical split, plain shell |
| `Ctrl-a -` | horizontal split, plain shell |
| `Ctrl-a \` | vertical split, launches claude |
| `Ctrl-a _` | horizontal split, launches claude |
| `Alt+v` | vertical split, plain shell, no prefix needed |
| `Alt+s` | horizontal split, plain shell, no prefix needed |
| `Alt+Ctrl+v` | vertical split, launches claude, no prefix needed |
| `Alt+Ctrl+s` | horizontal split, launches claude, no prefix needed |
| `Alt+Ctrl+h` | same as `Alt+Ctrl+s` |
| `Alt+Left/Right/Up/Down` | move between panes |
| `Alt+H` / `Alt+L` | previous/next window |
| `Ctrl-a c` | new window |
| `Ctrl-a T` | tile all panes into a grid |
| `Ctrl-a V` | stack all panes in one column |
| `Ctrl-a N` | name the current pane |
| `Ctrl-a r` | reload the config |
| `Alt+t` | cycle to the next Ghostty theme and apply it |
| `Alt+r` | cycle to the previous theme and apply it |

## Mouse

Both bars (window tabs on top, pane tabs underneath) work the same way:

- click an entry to switch to it
- double click to rename it
- right click a window tab for a menu (kill, rename, swap, respawn); right click a pane tab to kill it after confirming
- click and drag an entry onto another to reorder: a marker shows exactly where it will land, insert-before-this-item on the left half of the target, insert-after on the right half, past the last item to append at the end

Dragging a window pill onto the pane bar does not cross-contaminate: the window drag keeps driving its own bar, and the marker freezes rather than jumping to a nonsensical position while you're over the other bar's row.

Panes can also be moved by dragging, either by the name in the pane's top border or by its pill in the pane bar:

- drop on another pane (title drag only) to swap the two; focus stays on the pane you dragged
- drop on the middle of another window's pill to move the pane into that window, as a new full-height column on its right; the view follows it
- drop on the outer quarter of any window pill, the gap between two, to open the pane as a new window there, named after the pane; the same insertion marker as a window reorder shows where
- release anywhere else, or outside the terminal, to cancel

While dragging a title, the dragged pane's title turns peach with a `⇄`, and the pane under the pointer gets a yellow title and a complete yellow outline. A targeted window pill turns yellow. Dropping past the last window pill does nothing, since tmux sends no mouse events for the empty bar there; drop on the right edge of the last pill to put the new window at the end.

Inside a pane, clicks and ctrl+clicks go through to the app. With Claude Code's fullscreen renderer (`/tui fullscreen`), a click in the prompt moves the cursor there, and ctrl+click opens URLs and `file://` links. tmux's default ctrl+click pane swap is replaced for this.

## claude-fleet

```
claude-fleet <session-name> <worktree-path> [<worktree-path> ...]
```

Creates one tmux window per worktree given, named after the worktree's directory, each already running `claude` in that directory. Run it again with the same session name to jump back into it instead of recreating anything.

## Why two status lines

tmux only shows one window's content on screen at a time, so its top bar can only ever list windows, not panes, since panes from other windows aren't visible anyway. To also see and manage the panes inside the current window, this config adds a second status line underneath: same idea as the window bar, but scoped to whatever panes are currently on screen.

## Why both bars are custom scripts, not tmux's built-in window/pane list

tmux's pane iteration order (used by the `#{P:...}` format loop) is fixed at creation time. `swap-pane` swaps which pane's content sits at a given position, but explicitly does not reorder the underlying list, so dragging to reorder never had any effect on tmux's own pane bar no matter what was tried. `tmux-pane-bar` keeps its own ordered list of pane ids in a window option instead, self-healing on every render (drops dead panes, appends new ones), and renders from that. Windows do have a real, mutable, native order (`move-window` genuinely reorders them), so `tmux-window-bar` does not need this workaround, but it is still a custom script so both bars share the same drag machinery and visual style.

## How drag and drop actually works

tmux cannot reliably tell you when a mouse button was released. An app running in a pane (Claude Code included) can have its own mouse mode and swallow the release before any tmux binding sees it, and a release outside the terminal window generates no tmux event at all. Chasing this with more tmux-side event bindings does not converge: `MouseDragEnd1Status` only fires for a release that lands back on the status line, and there is no tmux event for "the button came up somewhere else."

So release detection does not use tmux events at all. `tmux-bar-drag-watchdog` starts once per drag and polls the physical pointer's actual button state directly, via `xinput query-state` against every detected pointer device, roughly every 30ms. This bypasses tmux and any app's mouse mode entirely; it is asking the X server what the hardware is doing, not asking tmux what it thinks happened. The instant the button reads as released, it commits using wherever the marker last showed and cleans up, regardless of where on (or off) screen that release physically happened.

`MouseDragEnd1Status`, `MouseUp1Pane` and `MouseUp1Border` also independently call the same commit logic (`tmux-bar-drag-commit`), so a release that does land somewhere tmux can see is handled instantly. The watchdog waits half a second after the release before committing, since every snap `tmux` call takes around 80ms and a fast drag can leave tmux's mouse queue briefly behind. `tmux-bar-drag-commit` reads and clears the drag state in a single tmux command list, which tmux runs atomically, so whichever caller comes second finds no drag and does nothing. This matters for moving a pane into another window, which unlike a reorder would not be harmless to run twice.

One known limitation: if a drag leaves the exact row of the bar it started on and later returns to it, tmux stops delivering drag-motion events for that gesture and never resumes, even once the cursor is back over the bar. The marker freezes at its last position rather than continuing to track the cursor. The drop still lands correctly on release, using wherever it was frozen, so this is a display quirk rather than a functional one. Fixing it fully would mean polling cursor position the same way release detection already does, translating screen pixels into terminal columns, which is a bigger change than the rest of this file represents.

## How dragging a pane by its title works

The name in each pane's top border sits inside `#[range=control|0]` in `pane-border-format`, so clicks on it arrive as their own `MouseDown1Control0` / `MouseDrag1Control0` / `DoubleClick1Control0` keys, separate from the rest of the border, which still resizes splits.

tmux pins every drag event to the pane the drag started in, so mid-drag it never reports which pane is under the pointer: `#{pane_id}` stays the source pane, and `mouse_x`/`mouse_y` go empty outside it. Two things work around that:

- The drop itself comes from tmux's release event (`MouseDragEnd1Pane` and friends), the one event that resolves to the pane actually under the pointer. On the window bar it comes from the hover state that `tmux-window-bar-drop` keeps, since status line events are never pinned.
- The hover highlight comes from `tmux-pane-drag-hover`, which polls the X pointer and maps pixels to terminal cells. It uses the client tty's pixel size (`TIOCGWINSZ`, which Ghostty fills in) and the Ghostty window geometry, and assumes `window-padding-y` equals `window-padding-x`, as in Ghostty's defaults. It focuses the pane under the pointer, because tmux always draws borders shared with the active pane in the active style, so focusing is the only way to get a complete outline.

While any pane or pill drag is in progress, `MouseDrag1Pane` and `MouseDrag1Border` do nothing, so a drag over pane content doesn't start a text selection in Claude Code or resize a split. All title drag state lives in five global options (`@paneDragSrc`, `@paneDragPrevActive`, `@paneDropTgt`, `@paneDropWin`, `@paneDropWinBefore`), cleared in the same atomic command list that reads them. A release tmux never sees (outside the window, or on the very first cell, which tmux also drops) is finished by the hover helper: it commits what is highlighted if the pointer is inside the terminal, and cancels otherwise. Any click is also a safe point to clear a leftover drag, since a press can't happen mid-drag.

## Highlighting idle Claude sessions

`tmux-claude-idle-loop` checks every pane running `claude` every 2 seconds. Claude Code hooks from `claude-hooks.json` call `tmux-claude-state`, which stores `busy` or `idle` in a `@claudeState` pane option, so detection works at any pane width. Sessions without that option fall back to screen scraping for `esc to interrupt` or the `Verb… (elapsed` spinner line. The result drives four background colors on both bars, matching the same states as pane borders and backgrounds:

- not idle, not active: unchanged, the normal dark surface color
- not idle, active: a step lighter gray than the above
- idle, not active: purple (`@thm_mauve`)
- idle, active: a lighter purple than the above (no catppuccin token matches this exactly, so it is computed by lightening mauve)

A pane's actual border and content background pick up the same idle state through a `@paneIdle` tmux option the loop sets per pane, since `pane-border-style`/`window-active-style` can reference tmux options directly in a format conditional but cannot see the marker files the bars use.

One gap: no hook fires when you interrupt with Esc, so an interrupted pane turns idle only when Claude Code's `idle_prompt` notification fires, after about 60 seconds.

## Notes

- Claude Code sets its own pane title through a terminal escape sequence and will overwrite anything tmux's built-in pane title mechanism sets. Pane names here are stored as a separate tmux user option instead, so renames stick.
- `@resurrect-processes 'claude'` tells tmux-resurrect to relaunch `claude` (not just a plain shell) in panes that had it running, and the two `tmux-resurrect-*-panenames` scripts carry pane renames through a restore too, since resurrect's own save format only knows about tmux's built-in pane_title, not custom options.
- `message-style`/`message-command-style` are overridden with an explicit `fill=` color (catppuccin's own default leaves `fill=default`, which only highlights the message text itself, not the full bar width behind it) so rename and kill confirmation prompts cover the whole row instead of overlapping the pills underneath.
- `tmux-resurrect` and `tmux-continuum` are included as plugins so sessions survive closing the terminal and reboots. Install the plugins from inside tmux with `Ctrl-a I`. Continuum's own save script takes around 300ms just to check whether a save is due, so `bin/tmux-continuum-loop` runs it on its own 60 second background timer instead of tying it to every status redraw, which would otherwise add that delay to every click.
- Clicking an actual pane (not a bar entry) also forces an immediate status refresh, since without it the bar's active-pane highlight would only catch up on the next natural `status-interval` tick, up to 5 seconds later.
