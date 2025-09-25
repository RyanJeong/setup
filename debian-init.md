# Debian-based System Initial Setup

## 1. Install base tools

```bash
sudo apt install -y screen build-essential libssl-dev zlib1g-dev libncurses5-dev libncursesw5-dev libreadline-dev libsqlite3-dev libgdbm-dev libdb5.3-dev libbz2-dev libexpat1-dev liblzma-dev libffi-dev wget
screen
```

Notes: `openssh-server` provides `sshd`.

---

## 2. Install `authorized_keys` for `<USER>`

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

* Do not leave `authorized_keys` file writable.
* Do not enable password authentication for `<USER>` if you want key-only access.

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

# Install a Docker
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
popd temp
rm -rf temp
```

Detach the screen to push `Ctrl` + `D`.

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
chmod +x mount_samba.sh \
  && echo "" >> ~/.bashrc \
  && echo "# Samba mount script" >> ~/.bashrc \
  && echo "~/mount_samba.sh" >> ~/.bashrc \
  && source ~/.bashrc
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
