# Debian-based System Initial Setup

## 1. Install base tools

```bash
sudo apt install -y screen build-essential libssl-dev zlib1g-dev libncurses5-dev libncursesw5-dev libreadline-dev libsqlite3-dev libgdbm-dev libdb5.3-dev libbz2-dev libexpat1-dev liblzma-dev libffi-dev wget
screen
```

Note: `openssh-server` provides `sshd`.

---

## 2. Install `authorized_keys` for `<USER>`

On the administrator machine, collect the public key(s) for `<USER>` and copy them to the target server.

Run as root on the server:

```bash
GUEST_USER=<USER>
PUBKEY=<PASTE_YOUR_KEY_HERE>

mkdir -p "/home/$GUEST_USER/.ssh"
chmod 0700 "/home/$GUEST_USER/.ssh"
cat > "/home/$GUEST_USER/.ssh/authorized_keys" <<EOF
$PUBKEY
EOF
chmod 0600 "/home/$GUEST_USER/.ssh/authorized_keys"
chown -R "$GUEST_USER":"$GUEST_USER" "/home/$GUEST_USER/.ssh"
```

Important:

- Do not leave the `authorized_keys` file writable.
- Do not enable password authentication for `<USER>` if you want key-only access.

---

## 3. Configure SSH to allow only key-based logins

Run the following script:

```bash
#!/bin/bash
# Purpose: Configure SSH to allow only public key login, disable password login, disable root login

CONFIG_FILE="/etc/ssh/sshd_config"
BACKUP_FILE="/etc/ssh/sshd_config.bak.$(date +%F_%T)"

# 1. Backup the original file
cp "$CONFIG_FILE" "$BACKUP_FILE" || { echo "Backup failed"; exit 1; }

# 2. Function to set or update an option
set_sshd_option() {
    local key="$1"
    local value="$2"
    if grep -qE "^\s*${key}\s+" "$CONFIG_FILE"; then
        sed -i -E "s|^\s*${key}\s+.*|${key} ${value}|g" "$CONFIG_FILE"
    else
        echo "${key} ${value}" >> "$CONFIG_FILE"
    fi
}

# 3. Apply the desired options
set_sshd_option "PermitRootLogin" "no"
set_sshd_option "PubkeyAuthentication" "yes"
set_sshd_option "PasswordAuthentication" "no"
set_sshd_option "ChallengeResponseAuthentication" "no"
set_sshd_option "UsePAM" "no"
set_sshd_option "AuthorizedKeysFile" "%h/.ssh/authorized_keys"

# 4. Restart SSH service
systemctl restart sshd || { echo "Failed to restart sshd"; exit 1; }

echo "sshd_config updated successfully. Backup: $BACKUP_FILE"
```

---

## 4. Install Docker

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y qemu-user-static binfmt-support locales

sudo locale-gen en_GB.UTF-8

echo "" >> ~/.bashrc
echo "export LANG=en_GB.UTF-8" >> ~/.bashrc
echo "export LC_ALL=en_GB.UTF-8" >> ~/.bashrc
source ~/.bashrc

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
sudo chmod 666 /var/run/docker.sock

sudo systemctl enable docker
sudo systemctl start docker
```

---

## 5. Install Extra Devkit

Open a new screen:

```bash
screen
```

In the screen:

```bash
mkdir temp
pushd temp
for ver in "3.6.15" "3.7.17" "3.8.17" "3.9.23" "3.10.9" "3.11.9" "3.12.9" "3.13.5"; do
  wget https://www.python.org/ftp/python/${ver}/Python-${ver}.tgz
  tar -xzvf *${ver}.tgz
  rm -rf *${ver}.tgz
  pushd Python-${ver}
  ./configure --enable-optimizations && make -j$(nproc) && sudo make altinstall
  popd
done
popd
rm -rf temp
```

Detach the screen with `Ctrl-a` then `d`.

---

## 6. tmux

```bash
sudo apt update && sudo apt install -y tmux git
```

### tmux Session Persistence (Ubuntu/Linux)

Persist tmux sessions across reboots using **tmux-resurrect** (save/restore) and **tmux-continuum** (auto-save, auto-restore, auto-start on boot via systemd), managed by **TPM**.

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

6. Boot auto-start (systemd): with `@continuum-boot 'on'`, the first tmux start generates and enables `~/.config/systemd/user/tmux.service`. Verify:

    ```bash
    systemctl --user is-enabled tmux.service   # expect: enabled
    ```

    - Headless servers only (start tmux at boot without logging in):

    ```bash
    sudo loginctl enable-linger "$USER"
    ```

#### 2. Usage

##### Restore after a reboot

- Automatic: on login (GUI, console, or ssh), systemd starts tmux and continuum restores the last save. Just attach:

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
- Service file: `~/.config/systemd/user/tmux.service` (auto-generated; regenerated on tmux start -- override via a drop-in, not by editing it directly).
- Start command: `set -g @continuum-systemd-start-cmd 'start-server'` (default `new-session -d`; `start-server` avoids a stray empty session and lets restore populate sessions).
- Inspect the service: `systemctl --user status tmux.service`.
- Auto-save interval (minutes): `set -g @continuum-save-interval '5'` (`0` disables auto-save).
- Temporarily disable auto-restore: `touch ~/tmux_no_auto_restore` (delete to re-enable).
- Auto-restart specific programs on restore: `set -g @resurrect-processes 'ssh psql "~vim"'`.
- Default prefix `Ctrl-b`. Save = `prefix + Ctrl-s`, Restore = `prefix + Ctrl-r`.
- Keep `tmux-continuum` last; a theme plugin that overwrites `status-right` after it silently stops auto-save.

### Reliable boot auto-start (static systemd unit)

continuum's `@continuum-boot 'on'` already starts the tmux **server** headlessly via a generated `~/.config/systemd/user/tmux.service`, then re-enables that unit on every tmux start (it recreates the file only if missing). Use a static unit you own if you want explicit control (start command, save-on-shutdown, no continuum management). `@continuum-restore 'on'` still does the actual restoring.

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

Otherwise continuum re-enables its own `tmux.service` on the next tmux start (and recreates it if missing).

```bash
sed -i "s/@continuum-boot 'on'/@continuum-boot 'off'/" ~/.tmux.conf
tmux source-file ~/.tmux.conf
```

`@continuum-restore 'on'` must stay -- it does the actual restoring.

#### 3. Remove continuum's generated service

```bash
systemctl --user disable --now tmux.service 2>/dev/null
rm -f ~/.config/systemd/user/tmux.service
systemctl --user daemon-reload
```

#### 4. Create the static unit

Uses your actual tmux path automatically:

```bash
TMUX_BIN="$(command -v tmux)"
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/tmux-autostart.service <<EOF
[Unit]
Description=tmux server (auto-start + resurrect restore)
Documentation=man:tmux(1)

[Service]
Type=forking
ExecStart=${TMUX_BIN} new-session -d
ExecStop=%h/.tmux/plugins/tmux-resurrect/scripts/save.sh
ExecStop=${TMUX_BIN} kill-server
KillMode=control-group
RestartSec=2

[Install]
WantedBy=default.target
EOF
```

#### 5. Enable and load

```bash
systemctl --user daemon-reload
systemctl --user enable --now tmux-autostart.service
systemctl --user is-enabled tmux-autostart.service   # expect: enabled
```

Headless servers only (start at boot without logging in):

```bash
sudo loginctl enable-linger "$USER"
```

#### 6. Test

A tmux server is already running now, so this won't trigger a restore (continuum skips restore when a server exists). The real test is a reboot:

```bash
# after reboot, in any new terminal:
tmux ls              # sessions should already be there
tmux attach -t <session-name>
```

#### Notes

- The unit only **starts the server** (and saves on stop); restore is done by `@continuum-restore 'on'`.
- If a reboot leaves one stray empty session, it's harmless. To avoid it, replace `new-session -d` with `start-server` in `ExecStart` (starts the server without a session; restore then populates all sessions).
- `ExecStop` saves before kill on `systemctl --user stop` / logout / shutdown, so the latest state is captured.
- Inspect / debug: `systemctl --user status tmux-autostart.service` and `journalctl --user -u tmux-autostart.service`.
- Uninstall: `systemctl --user disable --now tmux-autostart.service && rm ~/.config/systemd/user/tmux-autostart.service && systemctl --user daemon-reload` (re-enable `@continuum-boot 'on'` if you want continuum to manage the service again).

---

## Appendix A. Set Up Samba

- `mount_samba.sh`

```bash
#!/bin/bash

TARGET="$HOME/share"

SMB_HOST="<SMB_HOST>"
SMB_SERVICE="<SMB_SERVICE>"  # e.g., [global] -> global
USERNAME="<USER_NAME>"
PASSWORD="<USER_PASSWORD>"

mkdir -p ${TARGET}
cat /proc/mounts | grep home_munseongjeong_share >/dev/null 2>&1 \
  || sudo mount \
    -t cifs \
    -o username=${USERNAME},password=${PASSWORD},uid=$(id -u),gid=$(id -g) \
    //${SMB_HOST}/${SMB_SERVICE} \
    ${TARGET}
```

```bash
chmod +x mount_samba.sh \
  && echo "" >> ~/.bashrc \
  && echo "# Samba mount script" >> ~/.bashrc \
  && echo "~/mount_samba.sh" >> ~/.bashrc \
  && source ~/.bashrc
```

---

## Appendix B. Forward `192.168.0.100:1234` to `192.168.0.200:1234` via Apache reverse proxy

1. Install and enable modules

    ```bash
    apt update && apt install -y apache2 ufw
    a2enmod proxy proxy_http
    ```

2. Open port

    ```bash
    ufw allow 1234/tcp
    ```

3. Configure Apache

    - `/etc/apache2/ports.conf`

        ```text
        Listen 80
        Listen 1234
        ```

    - `/etc/apache2/sites-available/000-default.conf`

        ```text
        <VirtualHost *:80>
          ProxyPreserveHost On
          ProxyPass / http://192.168.0.200:1234/
          ProxyPassReverse / http://192.168.0.200:1234/
        </VirtualHost>
        ```

4. Restart service

    ```bash
    systemctl restart apache2
    ```

### Proxy Toggle

```bash
#!/bin/bash

MODS_AVAILABLE="/etc/apache2/mods-available/proxy.conf"
MODS_ENABLED="/etc/apache2/mods-enabled/proxy.conf"
APACHE_RELOAD="systemctl reload apache2"

function usage() {
  echo "Usage: $0 [on|off|status|--help]"
  echo
  echo "Controls proxy.conf by managing the symbolic link in mods-enabled."
  exit 0
}

function enable_proxy() {
  if [ -e "$MODS_ENABLED" ]; then
    echo "[OK] proxy.conf already enabled."
  else
    echo "[*] Enabling proxy.conf..."
    sudo ln -s "$MODS_AVAILABLE" "$MODS_ENABLED"
    sudo $APACHE_RELOAD
    echo "[OK] proxy.conf enabled."
  fi
}

function disable_proxy() {
  if [ -L "$MODS_ENABLED" ]; then
    echo "[*] Disabling proxy.conf..."
    sudo rm "$MODS_ENABLED"
    sudo $APACHE_RELOAD
    echo "[OK] proxy.conf disabled."
  else
    echo "[X] proxy.conf already disabled."
  fi
}

function check_status() {
  if [ -L "$MODS_ENABLED" ]; then
    echo "[OK] proxy.conf is currently ENABLED."
  else
    echo "[X] proxy.conf is currently DISABLED."
  fi
}

case "$1" in
  on)
    enable_proxy
    ;;
  off)
    disable_proxy
    ;;
  status)
    check_status
    ;;
  --help|-h)
    usage
    ;;
  *)
    echo "[!] Unknown argument: $1"
    usage
    ;;
esac
```

---

## Appendix C. SSH Toggle

- `toggle_ssh.sh`

```bash
DIR="$HOME/.ssh"
TARGET="authorized_keys"
if [ -e ${DIR}/${TARGET} ]; then
    mv ${DIR}/${TARGET} ${DIR}/${TARGET}.backup
elif [ -e ${DIR}/${TARGET}.backup ]; then
    mv ${DIR}/${TARGET}.backup ${DIR}/${TARGET}
fi
```
