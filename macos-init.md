# macOS Initial Setup

## 1. Configure SSH with ed25519

```bash
# Generate ed25519 key
ssh-keygen -t ed25519 -C "your_email@example.com"

# Start the ssh-agent
eval "$(ssh-agent -s)"

# Add key to ssh-agent
ssh-add ~/.ssh/id_ed25519

# Copy public key to clipboard
pbcopy < ~/.ssh/id_ed25519.pub
```

Add the copied key to your GitHub or GitLab account.

---

## 2. Install Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

After installation:

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

---

## 3. Keyboard Shortcuts (App Shortcuts)

Go to: **System Settings -> Keyboard -> Keyboard Shortcuts -> App Shortcuts**. Add the following:

- **All Applications**: Show Help menu, Zoom, Top, Bottom, Left, Right
- **Terminal**: Show Next Tab, Show Previous Tab

![Keyboard Shortcuts Screenshot](img/macos-init-keyboard-shortcuts.png)

---

## 4. Swap Caps Lock and Control

Go to: **System Settings -> Keyboard -> Keyboard Shortcuts -> Modifier Keys**. Set:

- Caps Lock -> Control
- Control -> Caps Lock (optional, if needed)

---

## 5. Display Sleep Settings

Go to: **System Settings -> Display -> Advanced -> Energy**.

- When charger is connected -> **Never sleep**
- When on battery -> **Sleep after 5 minutes**

---

## 6. Install clang

```bash
brew install llvm
```

Ensure `clang` is available:

```bash
clang --version
```

---

## 7. Install and Configure GitHub

```bash
brew install git

# Configure Git identity
git config --global user.name "Your Name"
git config --global user.email "your_email@example.com"
```

---

## 8. Install VSCode

```bash
brew install --cask visual-studio-code
```

---

## 9. Install Android Studio

```bash
brew install --cask android-studio
```

---

## 10. Install Docker

```bash
brew install --cask docker
```

Launch Docker.app once after installation.

---

## 11. Install Netron

```bash
brew install --cask netron
```

---

## 12. Install Rosetta

```bash
softwareupdate --install-rosetta --agree-to-license
```

---

## 13. iCloud Symlink

```bash
ln -s ~/Library/Mobile\ Documents/com\~apple\~CloudDocs ~/iCloud
```

---

## 14. tmux

```bash
brew install tmux
```

### tmux Session Persistence

Persist tmux sessions across reboots using **tmux-resurrect** (save/restore) and **tmux-continuum** (auto-save, auto-restore, auto-start on boot), managed by **TPM**.

- Survives a reboot: sessions, windows, panes, layout, per-pane working directory, visible pane contents.
- Does *not* survive: programs running inside panes (e.g. `claude`, Jenkins, scripts). Panes reopen, but you must re-run the program.

#### 1. Installation (one-time)

1. Install TPM (plugin manager):

    ```bash
    git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
    ```

2. Add to `~/.tmux.conf` (see the provided `.tmux.conf`):

    ```tmux
    set -g @plugin 'tmux-plugins/tpm'
    set -g @plugin 'tmux-plugins/tmux-resurrect'
    set -g @plugin 'tmux-plugins/tmux-continuum'   # keep continuum LAST
    set -g @resurrect-capture-pane-contents 'on'
    set -g @continuum-restore 'on'
    set -g @continuum-boot 'on'
    run '~/.tmux/plugins/tpm/tpm'                  # keep this at the very bottom
    ```

3. Install the plugins (downloads resurrect + continuum):

    ```bash
    ~/.tmux/plugins/tpm/bin/install_plugins
    ```

    - Alternative: inside tmux, press `prefix + I` (default prefix `Ctrl-b`).
    - `tmux source-file` only reloads config; it does *not* download plugins.

4. Reload config into the running server:

    ```bash
    tmux source-file ~/.tmux.conf
    ```

5. Create an initial save (so the first reboot has something to restore):

    ```bash
    ~/.tmux/plugins/tmux-resurrect/scripts/save.sh
    ```

    - Alternative: inside tmux, press `prefix + Ctrl-s`.

6. Boot auto-start (`@continuum-boot 'on'`) opens a Terminal window at login via osascript and needs Accessibility permission (System Settings -> Privacy & Security -> Accessibility). If `launchctl list | grep -i tmux` shows `- 0 ...` after a reboot but `tmux ls` reports no server, the boot script ran but failed to start tmux. For a more reliable, window-less start, use a personal LaunchAgent that runs `tmux new-session -d` at login instead (drop `@continuum-boot`, keep `@continuum-restore`).

#### 2. Usage

##### Restore after a reboot

- Automatic: on login, launchd starts tmux and continuum restores the last save. Just attach:

    ```bash
    tmux ls
    tmux attach -t <session-name>
    ```

- Manual (only if auto-restore didn't run):
  - Inside tmux: `prefix + Ctrl-r`
  - From a shell:

    ```bash
    ~/.tmux/plugins/tmux-resurrect/scripts/restore.sh
    ```

- Auto-restore runs only on server start, and is skipped if a tmux server is already running. Let boot auto-start handle it -- don't start tmux manually first.

##### Save

- Automatic: every 15 min (continuum). Lower the interval with `@continuum-save-interval` if you want faster capture.
- Manual: `prefix + Ctrl-s`.

##### Verify the latest save

```bash
DIR="${XDG_DATA_HOME:-$HOME/.local/share}/tmux/resurrect"
[ -d "$DIR" ] || DIR="$HOME/.tmux/resurrect"
ls -la "$DIR"                                            # expect 'last' + tmux_resurrect_*.txt
awk -F '\t' '$1=="pane"{print $2}' "$DIR/last" | sort -u | wc -l   # session count
```

#### 3. Notes & tuning

- Do NOT add a `session-created` hook that runs `save.sh`: creating a session during restore overwrites `last` with a near-empty save and breaks restore.
- Manual restore: start the server first, then restore. Never `kill-session` the last remaining session before restore finishes -- killing the last session kills the server, and `restore.sh` cannot start one.
- Recover an older save: point `last` at the wanted file, e.g. `ln -sf tmux_resurrect_<timestamp>.txt "$DIR/last"`, then run `restore.sh`. resurrect keeps the last several saves.
- Auto-save interval (minutes): `set -g @continuum-save-interval '5'` (`0` disables auto-save).
- Temporarily disable auto-restore: `touch ~/tmux_no_auto_restore` (delete to re-enable).
- Auto-restart specific programs on restore: `set -g @resurrect-processes 'ssh psql "~vim"'`.
- Default prefix `Ctrl-b`. Save = `prefix + Ctrl-s`, Restore = `prefix + Ctrl-r`.
- Keep `tmux-continuum` last; a theme plugin that overwrites `status-right` after it silently stops auto-save.

---

### Reliable boot auto-start (headless LaunchAgent)

`@continuum-boot 'on'` opens a Terminal window at login via osascript and needs Accessibility/Automation permission, so it can silently fail (server never starts after a reboot). A personal LaunchAgent starts the tmux **server** at login with no window; `@continuum-restore 'on'` then restores the last save. This replaces `@continuum-boot`, not resurrect/continuum saving.

#### 1. Save current sessions first

Safe to do now (no `session-created` hook):

```bash
~/.tmux/plugins/tmux-resurrect/scripts/save.sh
```

Confirm it captured your sessions:

```bash
DIR="${XDG_DATA_HOME:-$HOME/.local/share}/tmux/resurrect"
[ -d "$DIR" ] || DIR="$HOME/.tmux/resurrect"
awk -F '\t' '$1=="pane"{print $2}' "$DIR/last" | sort -u | wc -l   # expected session count
```

#### 2. Disable continuum boot (keep restore)

Otherwise continuum regenerates its own boot plist on the next tmux start.

```bash
sed -i '' "s/@continuum-boot 'on'/@continuum-boot 'off'/" ~/.tmux.conf
tmux source-file ~/.tmux.conf
```

`@continuum-restore 'on'` must stay -- it does the actual restoring.

#### 3. Remove continuum's old boot agent

```bash
launchctl unload ~/Library/LaunchAgents/Tmux.Start.plist 2>/dev/null
rm -f ~/Library/LaunchAgents/Tmux.Start.plist
```

#### 4. Create the LaunchAgent

Uses your actual tmux path automatically:

```bash
TMUX_BIN="$(command -v tmux)"
cat > ~/Library/LaunchAgents/local.tmux-autostart.plist <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>local.tmux-autostart</string>
    <key>ProgramArguments</key>
    <array>
        <string>${TMUX_BIN}</string>
        <string>new-session</string>
        <string>-d</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/tmux-autostart.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/tmux-autostart.log</string>
</dict>
</plist>
EOF
```

#### 5. Validate and load

```bash
plutil -lint ~/Library/LaunchAgents/local.tmux-autostart.plist   # expect: OK
launchctl unload ~/Library/LaunchAgents/local.tmux-autostart.plist 2>/dev/null
launchctl load  ~/Library/LaunchAgents/local.tmux-autostart.plist
```

Modern alternative to load: `launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.tmux-autostart.plist`.

#### 6. Test

A tmux server is already running now, so loading won't trigger a restore (continuum skips restore when a server exists). The real test is a reboot:

```bash
# after reboot, in any new terminal:
tmux ls              # sessions should already be there
tmux attach -t <session-name>
```

#### Notes

- The agent only **starts the server**; restore is done by `@continuum-restore 'on'`.
- If a reboot leaves one stray empty session, it's harmless. To avoid it, replace the `new-session` and `-d` arguments with a single `start-server` argument (starts the server without creating a session; restore then populates all sessions).
- Boot log for debugging: `cat /tmp/tmux-autostart.log`.
- Uninstall: `launchctl unload ~/Library/LaunchAgents/local.tmux-autostart.plist && rm ~/Library/LaunchAgents/local.tmux-autostart.plist` (re-enable `@continuum-boot 'on'` only if you want the Terminal-window method back).
