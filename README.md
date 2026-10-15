# fleetdeck

The deck you run a fleet of [Claude Code](https://claude.com/claude-code) sessions from: a tmux + Ghostty setup for running several of them side by side on Linux.

Every Claude session gets its own pane or window. Tab bars at the top show every window and every pane, and you can click and drag them. Any session that is waiting for your input lights up, so you can see which one needs you without checking each pane.

## What you get

- **Clickable tab bars.** Windows are listed on the top row, and the panes of the current window are listed under them. Click a tab to switch to it, double-click to rename it, right-click to close it, and drag it to reorder. Drag a window onto another window to combine them.
- **Drag panes around.** Grab a pane by its title, or by its tab, and drop it to swap it with another pane, move it into another window, or turn it into a window of its own.
- **Idle highlighting.** Tabs, borders and pane backgrounds change color when a Claude session finishes and waits for you.
- **Claude in one keystroke.** Split shortcuts open a new pane that already runs `claude`.
- **Sessions that survive.** Closing the terminal or rebooting brings back your windows, panes, pane names and running `claude` sessions.
- **`claude-fleet`.** Opens one window per git worktree, each one already running `claude`.
- **Theme cycling.** `Alt+t` and `Alt+r` step through Ghostty's themes.

## Requirements

- **Linux with an X11 session.** Drag and drop reads the mouse pointer from the X server, so it does not work under Wayland. To check, run `echo $XDG_SESSION_TYPE`; it should print `x11`.
- **tmux 3.3 or newer.** The pane tab bar needs 3.3. Dragging a pane by its title needs a newer tmux; it is tested on tmux 3.7c.
- **Ghostty**
- **Claude Code**
- **Tools:** `git`, `jq`, `python3`, `xdotool`, `xinput`, `xprop`

## Install

1. Install the dependencies (Ubuntu):

   ```
   sudo snap install tmux --classic
   sudo apt remove tmux
   sudo snap install ghostty --classic
   sudo apt install git jq python3 xdotool xinput x11-utils
   ```

   The tmux in Ubuntu's apt repos is often older than 3.3, so this installs the snap build instead. Removing the apt package matters: with both installed, whichever comes first on `PATH` wins, and scripts that call plain `tmux` can talk to the wrong binary.

2. Clone the repo and symlink everything into place. With symlinks, the live config *is* the repo: your edits show up in `git status`, and `git pull` updates the live setup. Existing config files are kept as `.bak`.

   ```
   git clone https://github.com/iuliuzamfirescu/fleetdeck.git ~/fleetdeck
   cd ~/fleetdeck
   for f in ~/.tmux.conf ~/.config/ghostty/config; do [ -e "$f" ] && [ ! -L "$f" ] && mv "$f" "$f.bak"; done
   mkdir -p ~/.config/ghostty ~/.local/bin
   ln -sf "$PWD/tmux.conf" ~/.tmux.conf
   ln -sf "$PWD/ghostty.config" ~/.config/ghostty/config
   for f in bin/*; do ln -sf "$PWD/$f" ~/.local/bin/; done
   ```

   `~/.local/bin` must be on your `PATH`. It is on most distributions after you log in again.

3. Tell Ghostty which directory to open in. This goes in an untracked file next to the config, so it stays out of the repo:

   ```
   echo 'command = tmux new-session -A -s main -c /path/to/your/project claude' > ~/.config/ghostty/local
   ```

   Every new Ghostty window then attaches to the tmux session `main`, or creates it with `claude` running in that directory.

4. Install the tmux plugin manager:

   ```
   git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
   ```

   Start tmux, then press `Ctrl-a I` (capital i) to install the plugins: catppuccin, tmux-sensible, tmux-resurrect and tmux-continuum.

5. Add the busy/idle hooks to your Claude Code settings. The merge adds to any hooks you already have and does not replace them. Running it twice is safe.

   ```
   [ -f ~/.claude/settings.json ] || echo '{}' > ~/.claude/settings.json
   jq -s '.[0] as $s | .[1].hooks as $h | $s | .hooks = (reduce ($h | keys[]) as $k ($s.hooks // {}; .[$k] = ((.[$k] // []) + $h[$k] | unique)))' \
     ~/.claude/settings.json claude-hooks.json > /tmp/claude-settings.json && mv /tmp/claude-settings.json ~/.claude/settings.json
   ```

6. Restart Ghostty.

### Updating

```
git -C ~/fleetdeck pull
```

Then press `Ctrl-a r` in tmux to reload the config.

### Uninstalling

```
rm ~/.tmux.conf ~/.config/ghostty/config
find ~/.local/bin -lname "$HOME/fleetdeck/*" -delete
```

Then restore the `.bak` files if you had any.

## Keyboard

The tmux prefix is `Ctrl-a`.

| Key | Action |
|---|---|
| `Ctrl-a \|` | split side by side, plain shell |
| `Ctrl-a -` | split top and bottom, plain shell |
| `Ctrl-a \` | split side by side, runs `claude` |
| `Ctrl-a _` | split top and bottom, runs `claude` |
| `Alt+v` | split side by side, plain shell, no prefix needed |
| `Alt+s` | split top and bottom, plain shell, no prefix needed |
| `Alt+Ctrl+v` | split side by side, runs `claude`, no prefix needed |
| `Alt+Ctrl+s` | split top and bottom, runs `claude`, no prefix needed |
| `Alt+Ctrl+h` | same as `Alt+Ctrl+s` |
| `Alt+Left/Right/Up/Down` | move between panes |
| `Alt+H` / `Alt+L` | previous / next window |
| `Ctrl-a c` | new window |
| `Ctrl-a T` | arrange all panes in a grid |
| `Ctrl-a V` | stack all panes in one column |
| `Ctrl-a N` | name the current pane |
| `Ctrl-a r` | reload the config |
| `Alt+t` / `Alt+r` | next / previous Ghostty theme |

## Mouse

### Tab bars

The top row lists windows. The row under it lists the panes of the current window. Both work the same way:

- Click a tab to switch to it.
- Double-click a tab to rename it.
- Right-click a window tab for a menu (close, rename, swap, respawn). Right-click a pane tab to close the pane, after a confirmation.
- Drag a tab onto another to reorder. A marker shows where it will land: the left half of a tab puts it before that tab, and the right half puts it after. Drop on the right half of the last tab to move it to the end.
- Drag a window tab onto the **middle** of another window tab to combine the two windows. The tab turns yellow while you hover it. On release, every pane of the dragged window moves into the other window, as an evenly split column on its right, and the view follows. The outer quarters of the tab still reorder.

### Moving panes

Drag a pane by the name in its top border, or by its tab in the pane bar:

- **Drop on another pane** to swap the two. This works for a title drag only. Focus stays on the pane you dragged.
- **Drop on the middle of a window tab** to move the pane into that window, as a new column on its right. The view follows the pane.
- **Drop near the edge of a window tab**, or in the gap between two tabs, to make the pane a new window there. The new window takes the pane's name.
- **Release anywhere else**, or outside the terminal, to cancel.

While you drag a title, its name turns peach with a `⇄`. The pane under the pointer gets a yellow title and a yellow outline, and a window tab you hover turns yellow. To put a new window last, drop it on the right edge of the last window tab, because tmux sends no mouse events for the empty bar after it.

Double-click a pane's title to rename it.

### Inside a pane

Clicks and ctrl+clicks go through to the app in the pane. With Claude Code's fullscreen renderer (`"tui": "fullscreen"` in `~/.claude/settings.json`), a click in the prompt moves the cursor there, and ctrl+click opens URLs and `file://` links. tmux's default ctrl+click pane swap is turned off for this.

## Idle highlighting

When a Claude session is waiting for your input, the bars, the pane border and the pane background change color:

| | Busy | Idle |
|---|---|---|
| Tab, not selected | dark | dark amber |
| Tab, selected | lavender | amber |
| Pane border | gray (magenta when focused) | mauve (light mauve when focused) |
| Pane background | normal | blue-gray tint |

Claude Code reports busy and idle through the hooks from install step 5. Sessions without the hooks fall back to reading the screen, which works less reliably. One gap: no hook fires when you interrupt Claude with Esc, so an interrupted session turns idle only when Claude Code's idle notification fires, after about 60 seconds.

## claude-fleet

```
claude-fleet <session-name> <worktree-path> [<worktree-path> ...]
```

Creates a tmux session with one window per worktree. Each window is named after the worktree's directory and already runs `claude` in it. Run it again with the same session name to jump back into the session instead of creating a new one.

## Customizing

- **Ghostty:** put machine-specific settings in `~/.config/ghostty/local`. They override `ghostty.config`, and the repo never sees them. `Alt+t`/`Alt+r` write the chosen theme there too.
- **Tab colors:** the four tab colors and the drop-target color are the `@pill_*` options in `tmux.conf` (`@pill_working_inactive`, `@pill_working_active`, `@pill_idle_inactive`, `@pill_idle_active`, `@pill_drop_target`).
- **Anything else in tmux:** fork the repo, edit `tmux.conf` in your clone, and press `Ctrl-a r`.

Don't edit the symlinked files with `sed -i` or similar tools. They replace the symlink with a plain copy, which then stops following the repo.

## How it works

The rest of this file is for anyone who wants to change the scripts.

### Files

- `tmux.conf`, `ghostty.config`: the configs, symlinked to `~/.tmux.conf` and `~/.config/ghostty/config`
- `claude-hooks.json`: Claude Code hooks that report busy/idle per pane
- `bin/tmux-window-bar`, `bin/tmux-pane-bar`: render the two tab bars
- `bin/tmux-window-bar-move`, `bin/tmux-pane-bar-move`: reorder windows or pane tabs on drop
- `bin/tmux-bar-drag-update`, `bin/tmux-bar-drag-watchdog`, `bin/tmux-bar-drag-commit`: the drag machinery both bars share
- `bin/tmux-pane-drag`, `bin/tmux-pane-drag-hover`: drag a pane by its title
- `bin/tmux-window-bar-drop`, `bin/tmux-pane-to-window`: drop a dragged pane on the window bar
- `bin/tmux-window-merge`: move all panes of a dragged window into the window it is dropped on
- `bin/tmux-claude-state`, `bin/tmux-claude-idle-loop`: idle detection and highlighting
- `bin/tmux-continuum-loop`: runs tmux-continuum's save check on its own timer
- `bin/tmux-resurrect-save-panenames`, `bin/tmux-resurrect-restore-panenames`: keep pane names through a restore
- `bin/claude-fleet`, `bin/ghostty-theme-cycle`: the two commands described above

### Why two custom tab bars

tmux shows one window at a time, so its own status bar lists windows only. A second status line, under the window bar, lists the panes of the window on screen.

Both bars are scripts rather than tmux's built-in lists. tmux's pane order (used by the `#{P:...}` format loop) is fixed when panes are created: `swap-pane` swaps what is shown where, but never reorders that list, so a pane bar built on it can't be reordered by dragging. `tmux-pane-bar` keeps its own ordered list of pane ids in a window option and repairs it on every render, dropping dead panes and appending new ones. Windows do have a real order (`move-window` changes it), but `tmux-window-bar` is still a script, so both bars share the same drag code and look the same.

### Dragging tabs

tmux can't reliably say when the mouse button was released. A release outside the terminal window generates no tmux event at all. So `tmux-bar-drag-watchdog` starts once per drag and asks the X server for the button state about every 30 ms, through `xinput query-state`. When the button reads as released, it commits the drop at the marker's last position, wherever the release happened.

`MouseDragEnd1Status`, `MouseUp1Pane` and `MouseUp1Border` call the same commit script (`tmux-bar-drag-commit`), so a release that tmux does see is handled at once. The watchdog waits half a second after the release before it commits, since each snap `tmux` call takes about 80 ms and a fast drag can leave tmux's mouse queue briefly behind. The commit script reads and clears the drag state in a single tmux command list, which tmux runs atomically, so whichever caller comes second finds no drag and does nothing. This matters because moving a pane into another window, unlike a reorder, would do damage if it ran twice.

A plain click on a tab arms a drag on press, and the release disarms it again, so no drag state is left behind.

A window drag over the window bar also runs `tmux-window-bar-drop`, the same hover script a dragged pane uses. Over the middle of another window's tab it records that window as a merge target, and the reorder marker hides. Over the outer quarters it agrees with the reorder marker, so reordering works as before. Like the marker, the merge target stays frozen when the pointer leaves the bar, so a release over a pane still merges.

### Dragging a pane by its title

The name in each pane's top border is wrapped in `#[range=control|0]` in `pane-border-format`. Clicks on it arrive as their own `MouseDown1Control0`, `MouseDrag1Control0` and `DoubleClick1Control0` keys, separate from the rest of the border, which still resizes splits.

tmux pins every drag event to the pane the drag started in. During a drag, `#{pane_id}` stays the source pane, and `mouse_x`/`mouse_y` are empty outside it, so tmux never says which pane is under the pointer. Two things work around that:

- **The drop** comes from tmux's release event (`MouseDragEnd1Pane` and the others). That is the one event that resolves to the pane actually under the pointer. On the window bar, the drop comes from the hover state that `tmux-window-bar-drop` keeps, because status line events are never pinned.
- **The hover highlight** comes from `tmux-pane-drag-hover`, which polls the X pointer and converts pixels to terminal cells. It uses the terminal's pixel size from `TIOCGWINSZ`, which Ghostty fills in, and the Ghostty window geometry. It assumes `window-padding-y` equals `window-padding-x`, as in Ghostty's defaults. It focuses the hovered pane, because tmux always draws a border shared with the focused pane in the focused style, so focusing is the only way to get a complete outline.

While any pane or tab drag is in progress, `MouseDrag1Pane` and `MouseDrag1Border` do nothing, so a drag across a pane doesn't start a text selection in Claude Code or resize a split.

All title-drag state is in five global options (`@paneDragSrc`, `@paneDragPrevActive`, `@paneDropTgt`, `@paneDropWin`, `@paneDropWinBefore`). The same atomic command list that reads them also clears them. tmux reports no release outside the window, and none on the terminal's very first cell. In those cases the hover helper finishes the drag: it commits what is highlighted if the pointer is inside the terminal, and cancels otherwise. Any click also clears a leftover drag, since a press can't happen in the middle of one.

### Idle detection

`tmux-claude-idle-loop` checks every pane that runs `claude` every 2 seconds. The hooks in `claude-hooks.json` call `tmux-claude-state`, which stores `busy` or `idle` in a `@claudeState` pane option, so detection works at any pane width. Sessions without that option fall back to searching the screen for `esc to interrupt` or the `Verb… (elapsed` spinner line.

The loop sets a `@paneIdle` option on each idle pane, which the border and background styles read. It also writes marker files that the bar scripts read.

### Notes

- Claude Code sets its own pane title through a terminal escape sequence, which overwrites any title tmux sets. Pane names are therefore stored in a separate `@paneName` option.
- `@resurrect-processes 'claude'` makes tmux-resurrect start `claude` again, not just a shell, in panes that had it running. The two `tmux-resurrect-*-panenames` scripts carry pane names through a restore, since resurrect's save format only stores tmux's own pane title.
- `message-style` and `message-command-style` set an explicit `fill=` color, so rename and close prompts cover the whole bar row instead of only their text.
- tmux-continuum's save script takes about 300 ms just to check whether a save is due. `bin/tmux-continuum-loop` runs it on its own 60 second timer instead of on every status redraw, which would add that delay to every click. It expects TPM's default plugin path, `~/.tmux/plugins`.
- Clicking inside a pane forces an immediate status refresh. Without it, the bar's highlight would wait for the next `status-interval` tick, up to 5 seconds.
