# Debian-based System Initial Setup

## 1. Install base tools

Run as root:

```bash
apt update
apt install -y git curl wget vim openssh-server sudo \
  python3.6 python3.6-dev \
  python3.7 python3.7-dev \
  python3.8 python3.8-dev \
  g++
```

```bash
apt install -y build-essential libssl-dev zlib1g-dev libncurses5-dev libncursesw5-dev libreadline-dev libsqlite3-dev libgdbm-dev libdb5.3-dev libbz2-dev libexpat1-dev liblzma-dev libffi-dev wget 

mkdir temp
pushd temp
for ver in "3.9.23" "3.10.9" "3.11.9" "3.12.9" "3.13.5"; do 
  wget https://www.python.org/ftp/python/${ver}/Python-${ver}.tgz
  tar -xzvf *${ver}.tgz
  rm -rf *${ver}.tgz
  pushd Python-${ver}
  ./configure --enable-optimizations && make -j$(nproc) && make altinstall
  popd
done
popd temp
rm -rf temp
```

Notes:

* `sudo` is required only so administrators can use it to run commands.
* `openssh-server` provides `sshd`.

---

## 2. Create user `<USER>` (no sudo by default)

Run as root:

```bash
GUEST_USER=<USER>

useradd -m -s /bin/bash "$GUEST_USER"
passwd -l "$GUEST_USER"
```

Explanation:

* `-m` creates `/home/<USER>`.
* `-s /bin/bash` sets the shell.
* `passwd -l <USER>` locks the password so password login is not available.

## Opt. Create user `admin`

```bash
# create full-sudo user 'admin' and set a password interactively
useradd -m -s /bin/bash admin
passwd admin # enter a secure password when prompted

# grant full sudo via ubuntu/debian sudo group
usermod -aG sudo admin
```

---

## 3. Prepare `install.sh` and `uninstall.sh`

Place the scripts in `/home/<USER>` and set ownership and permissions so they cannot be modified by `<USER>`.

Run as root:

```bash
# place your scripts under /home/<USER>
chown root:root "/home/$GUEST_USER/install.sh" "/home/$GUEST_USER/uninstall.sh"
chmod 0755 "/home/$GUEST_USER/install.sh" "/home/$GUEST_USER/uninstall.sh"
```

Security note:

* Making the scripts owned by `root` prevents `<USER>` from editing them and gaining privilege escalation.
* If the scripts need editable content, keep editable parts in `/home/<USER>/data` and ensure scripts perform controlled operations only.

---

## 4. Configure sudoers to allow only specific commands

Create a file under `/etc/sudoers.d/<USER>` with mode `0440`.

Run as root (recommended using `visudo` to validate syntax):

```bash
cat > /etc/sudoers.d/$GUEST_USER <<EOF
# Allow $GUEST_USER to run two scripts and two rm commands without password.
$GUEST_USER ALL=(root) NOPASSWD: /home/$GUEST_USER/install.sh, /home/$GUEST_USER/uninstall.sh, /bin/rm -rf /var/foo, /bin/rm -rf /home/$GUEST_USER/.foo
EOF
chmod 0440 "/etc/sudoers.d/$GUEST_USER"
```

Notes:

* Use the absolute paths shown above. Sudoers matches the full path and the given argument pattern.
* The `rm -rf` entries are exact. If `<USER>` runs a different argument set for `rm`, sudo will deny it.
* Test sudoers syntax with `visudo -c`.

---

## 5. Configure SSH to allow only key-based logins

Edit `/etc/ssh/sshd_config` and ensure these lines exist and are not duplicated with conflicting values.

Recommended changes (run as root or use an editor):

```bash
# recommended sshd_config settings for key-only login
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM no
AuthorizedKeysFile %h/.ssh/authorized_keys
```

After editing restart the SSH service:

```bash
systemctl restart sshd
# or on some systems
service ssh restart
```

---

## 6. Install `authorized_keys` for `<USER>`

On the administrator machine, collect the public key(s) for `<USER>` and copy them to the target server.

Run as root on the server:

```bash
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

* Do not leave `authorized_keys` world writable.
* Do not enable password authentication for `<USER>` if you want key-only access.

---

## 7. Testing the configuration

1. From a client with the private key try:

    ```bash
    ssh -i /path/to/private_key $GUEST_USER@server.example.com
    ```

2. Verify password login is denied and key login succeeds.

3. Test sudo restrictions as `<USER>`:

    ```bash
    # allowed
    sudo "/home/$GUEST_USER/install.sh"
    sudo "/home/$GUEST_USER/uninstall.sh"
    sudo /bin/rm -rf /var/foo
    sudo /bin/rm -rf "/home/$GUEST_USER/.foo"
    
    # not allowed examples
    sudo apt update
    sudo /bin/rm -rf /etc
    ```

Sudo should allow only the listed commands.

---

## 8. Hardening and notes

* Keep `install.sh` and `uninstall.sh` controlled and audited. They run as root.
* Avoid allowing `<USER>` to run a shell via sudo.
* If a script requires running other commands as root, include those operations inside the root-owned script rather than giving `<USER>` sudo for arbitrary commands.
* Use `visudo -c` after editing sudoers files to verify syntax.

---

## 9. Rollback

To remove the special sudo privileges and lock down again, run as root:

```bash
rm -f "/etc/sudoers.d/$GUEST_USER"
chmod 0440 "/etc/sudoers.d/$GUEST_USER" || true
```

To re-enable password login (not recommended), edit `/etc/ssh/sshd_config` and set `PasswordAuthentication yes` then restart `sshd`.

---

## Appendix A. Set Up Samba

* `mount_samba.sh`

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
echo "" >> ~/.bashrc \
  && echo "# Samba mount script" >> ~/.bashrc \
  && echo "~/mount_samba.sh" >> ~/.bashrc
```

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

    * `/etc/apache2/ports.conf`

        ```text
        Listen 80
        Listen 1234
        ```

    * `/etc/apache2/sites-available/000-default.conf`

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

```shell
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
    echo "[✓] proxy.conf already enabled."
  else
    echo "[*] Enabling proxy.conf..."
    sudo ln -s "$MODS_AVAILABLE" "$MODS_ENABLED"
    sudo $APACHE_RELOAD
    echo "[✓] proxy.conf enabled."
  fi
}

function disable_proxy() {
  if [ -L "$MODS_ENABLED" ]; then
    echo "[*] Disabling proxy.conf..."
    sudo rm "$MODS_ENABLED"
    sudo $APACHE_RELOAD
    echo "[✓] proxy.conf disabled."
  else
    echo "[✗] proxy.conf already disabled."
  fi
}

function check_status() {
  if [ -L "$MODS_ENABLED" ]; then
    echo "[✓] proxy.conf is currently ENABLED."
  else
    echo "[✗] proxy.conf is currently DISABLED."
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

## Appendix C. SSH Toggle

* `toggle_ssh.sh`

```bash
DIR="$HOME/.ssh"
TARGET="authorized_keys"
if [ -e ${DIR}/${TARGET} ]; then
    mv ${DIR}/${TARGET} ${DIR}/${TARGET}.backup
elif [ -e ${DIR}/${TARGET}.backup ]; then
    mv ${DIR}/${TARGET}.backup ${DIR}/${TARGET}
fi
```
