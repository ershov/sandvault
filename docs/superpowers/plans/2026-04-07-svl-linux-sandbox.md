# SVL — Linux Sandbox User Tool Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a standalone bash script `svl` that manages a sandboxed Linux user with passwordless shell/SSH access, shared file ownership via setgid+group, and clean teardown commands.

**Architecture:** Single bash file `svl` at the repo root (next to the existing macOS `sv`). All logic in functions, dispatched by a case statement on the subcommand. Uses only standard Linux utilities — no external dependencies.

**Tech Stack:** Bash 5.x, standard Linux tools (`useradd`, `usermod`, `groupadd`, `sudo`, `ssh`, `rsync`, `pkill`, `setfacl`, `getent`, `visudo`)

**Spec:** `docs/superpowers/specs/2026-04-07-svl-linux-sandbox-design.md`

---

### Task 1: Script skeleton with help and argument parsing

**Files:**
- Create: `svl`

- [ ] **Step 1: Create the script with shebang, constants, and helpers**

```bash
#!/bin/bash
set -Eeuo pipefail

readonly SVL_VERSION="0.1.0"

# Resolve the invoking (host) user — works even under sudo
HOST_USER="${SUDO_USER:-$USER}"
readonly HOST_USER
readonly SANDBOX_USER="sandvault-${HOST_USER}"
readonly SANDBOX_HOME="/home/${SANDBOX_USER}"
readonly SUDOERS_FILE="/etc/sudoers.d/50-${SANDBOX_USER}"
HOST_SHELL="$(getent passwd "$HOST_USER" | cut -d: -f7)"
readonly HOST_SHELL

info()  { echo >&2 "[svl] $*"; }
warn()  { echo >&2 "[svl] WARNING: $*"; }
die()   { echo >&2 "[svl] ERROR: $*"; exit 1; }

sandbox_user_exists() { getent passwd "$SANDBOX_USER" >/dev/null 2>&1; }
sandbox_group_exists() { getent group "$SANDBOX_USER" >/dev/null 2>&1; }
require_sandbox() { sandbox_user_exists || die "Sandbox user '$SANDBOX_USER' does not exist. Run 'svl build' first."; }
```

- [ ] **Step 2: Add usage/help function**

```bash
show_help() {
    cat <<EOF
Usage: svl <command> [args...]

Commands:
  build       Create sandbox user, group, sudoers, home skeleton, SSH keys
  shell       Open login shell as sandbox user (extra args forwarded to sudo)
  ssh         SSH into sandbox user on localhost (extra args forwarded to ssh)
  run         Run a command as the sandbox user: svl run <cmd> [args...]
  stop        Kill all sandbox user processes
  uninstall   Full teardown (user, home, group, sudoers)
  status      Show sandbox user status, processes, disk usage
  sync        Re-copy dotfiles/credentials from host to sandbox home
  help        Show this help
  version     Show version

Sandbox user: $SANDBOX_USER
Sandbox home: $SANDBOX_HOME
EOF
}

show_version() {
    echo "svl $SVL_VERSION"
}
```

- [ ] **Step 3: Add main dispatch**

```bash
main() {
    [[ $# -ge 1 ]] || { show_help; exit 1; }

    local cmd="$1"
    shift

    case "$cmd" in
        build)      cmd_build "$@" ;;
        shell)      cmd_shell "$@" ;;
        ssh)        cmd_ssh "$@" ;;
        run)        cmd_run "$@" ;;
        stop)       cmd_stop "$@" ;;
        uninstall)  cmd_uninstall "$@" ;;
        status)     cmd_status "$@" ;;
        sync)       cmd_sync "$@" ;;
        help|-h|--help) show_help ;;
        version|--version) show_version ;;
        *)          die "Unknown command: $cmd. Run 'svl help' for usage." ;;
    esac
}

# Stub functions — implemented in subsequent tasks
cmd_build()     { die "Not yet implemented"; }
cmd_shell()     { die "Not yet implemented"; }
cmd_ssh()       { die "Not yet implemented"; }
cmd_run()       { die "Not yet implemented"; }
cmd_stop()      { die "Not yet implemented"; }
cmd_uninstall() { die "Not yet implemented"; }
cmd_status()    { die "Not yet implemented"; }
cmd_sync()      { die "Not yet implemented"; }

main "$@"
```

- [ ] **Step 4: Make executable and test help output**

```bash
chmod +x svl
./svl help
./svl version
./svl          # should show help and exit 1
./svl bogus    # should show error and exit 1
```

Expected:
- `svl help` prints usage, exits 0
- `svl version` prints `svl 0.1.0`, exits 0
- `svl` (no args) prints usage, exits 1
- `svl bogus` prints error, exits 1

- [ ] **Step 5: Commit**

```bash
git add svl
git commit -m "feat(svl): add script skeleton with help and argument dispatch"
```

---

### Task 2: `svl build` — user and group creation

**Files:**
- Modify: `svl`

- [ ] **Step 1: Add the group and user creation logic**

Replace the `cmd_build` stub with:

```bash
cmd_build() {
    [[ "$(id -u)" -ne 0 ]] || die "Do not run as root. Run as your normal user (sudo is used internally)."

    info "Building sandbox: $SANDBOX_USER"

    # Create group
    if sandbox_group_exists; then
        info "Group '$SANDBOX_USER' already exists"
    else
        info "Creating group '$SANDBOX_USER'"
        sudo groupadd "$SANDBOX_USER"
    fi

    # Create user
    if sandbox_user_exists; then
        info "User '$SANDBOX_USER' already exists"
    else
        info "Creating user '$SANDBOX_USER'"
        sudo useradd \
            --create-home \
            --gid "$SANDBOX_USER" \
            --shell "$HOST_SHELL" \
            --home-dir "$SANDBOX_HOME" \
            "$SANDBOX_USER"
        # Lock password — disables password login, SSH key-only
        sudo usermod -L "$SANDBOX_USER"
    fi

    # Add host user to sandbox group (for file access)
    if id -nG "$HOST_USER" | grep -qw "$SANDBOX_USER"; then
        info "Host user '$HOST_USER' already in group '$SANDBOX_USER'"
    else
        info "Adding '$HOST_USER' to group '$SANDBOX_USER'"
        sudo usermod -aG "$SANDBOX_USER" "$HOST_USER"
    fi

    # Set home directory permissions: setgid + owner/group full access
    info "Setting permissions on $SANDBOX_HOME"
    sudo chmod 2770 "$SANDBOX_HOME"

    # Install sudoers rule
    build_sudoers

    # Copy dotfiles and tools
    sync_dotfiles

    # Set up SSH access
    build_ssh

    # Fix ownership
    info "Fixing ownership"
    sudo chown -R "${SANDBOX_USER}:${SANDBOX_USER}" "$SANDBOX_HOME"

    info "Build complete. Sandbox user: $SANDBOX_USER"
    info "NOTE: You may need to log out and back in for group membership to take effect."
}
```

- [ ] **Step 2: Test build creates user and group**

```bash
sudo ./svl build
```

Expected: creates group, user, adds host user to group, sets permissions. Running again should print "already exists" for each step.

Verify:

```bash
getent passwd sandvault-ubuntu
getent group sandvault-ubuntu
ls -ld /home/sandvault-ubuntu
```

(Will fail on sudoers/sync/ssh since those aren't implemented yet — that's expected. We'll implement them next.)

- [ ] **Step 3: Commit**

```bash
git add svl
git commit -m "feat(svl): implement build command — user and group creation"
```

---

### Task 3: `svl build` — sudoers setup

**Files:**
- Modify: `svl`

- [ ] **Step 1: Add the sudoers function**

```bash
build_sudoers() {
    if [[ -f "$SUDOERS_FILE" ]]; then
        info "Sudoers rule already exists at $SUDOERS_FILE"
        return
    fi

    info "Installing sudoers rule at $SUDOERS_FILE"
    local rule="${HOST_USER} ALL=(${SANDBOX_USER}) NOPASSWD: ALL"

    # Write via tee to a temp file, validate, then move into place
    local tmp
    tmp="$(mktemp)"
    echo "$rule" > "$tmp"
    # visudo -c -f validates the syntax of the file
    if sudo visudo -c -f "$tmp" >/dev/null 2>&1; then
        sudo cp "$tmp" "$SUDOERS_FILE"
        sudo chmod 0440 "$SUDOERS_FILE"
        rm -f "$tmp"
    else
        rm -f "$tmp"
        die "Sudoers rule failed validation. Aborting."
    fi
}
```

- [ ] **Step 2: Test sudoers works**

```bash
sudo ./svl build
cat /etc/sudoers.d/50-sandvault-ubuntu
sudo -u sandvault-ubuntu whoami
```

Expected:
- File contains `ubuntu ALL=(sandvault-ubuntu) NOPASSWD: ALL`
- `sudo -u sandvault-ubuntu whoami` prints `sandvault-ubuntu` without prompting for password

- [ ] **Step 3: Commit**

```bash
git add svl
git commit -m "feat(svl): add sudoers rule setup with visudo validation"
```

---

### Task 4: `svl build` — dotfile sync

**Files:**
- Modify: `svl`

- [ ] **Step 1: Add the dotfile sync function**

```bash
# Files and directories to copy from host home to sandbox home.
# Each entry is relative to $HOME. Directories must end with /.
# Only entries that exist on the host are copied.
SYNC_DOTFILES=(
    # Shell
    .bashrc
    .bash_profile
    .profile
    .zshrc
    .zshenv
    .zprofile
    .zlogin
    .inputrc
    .screenrc
    .hushlogin

    # Editor / TUI
    .vimrc
    .nanorc
    .tmux.conf
    .tmux.tcl
    .config/nvim/
    .config/htop/
    .config/kitty/
    .config/alacritty/
    .config/wezterm/
    .config/nnn/
    .config/vifm/
    .config/sc-im/
    .config/lazygit/
    .config/lazy-github/
    .config/gitui/
    .config/starship.toml

    # Dev tools
    .gitconfig
    .gitignore
    .bazelrc
    .editorconfig
    .config/git/
    .config/gh/
    .config/uv/
    .config/opencode/
    .config/pip/
    .npmrc
    .gemrc
    .curlrc
    .wgetrc
    .ripgreprc
    .fdignore
    .prettierrc
    .eslintrc.json

    # Agent credentials
    .claude/.credentials.json
    .claude/settings.json
    .claude/plugins/
    .codex/auth.json
    .codex/config.toml

    # Binaries / runtimes
    .local/bin/
    .bun/
    .cargo/
    bin/
)

sync_dotfiles() {
    local host_home
    host_home="$(getent passwd "$HOST_USER" | cut -d: -f6)"
    info "Syncing dotfiles from $host_home to $SANDBOX_HOME"

    for entry in "${SYNC_DOTFILES[@]}"; do
        local src="${host_home}/${entry}"
        local dest="${SANDBOX_HOME}/${entry}"

        # Skip if source doesn't exist
        [[ -e "$src" ]] || continue

        # Ensure parent directory exists
        local dest_dir
        dest_dir="$(dirname "$dest")"
        sudo mkdir -p "$dest_dir"

        if [[ -d "$src" ]]; then
            # Directory: rsync with trailing slash to copy contents
            # Ensure source has trailing slash for rsync
            [[ "$src" == */ ]] || src="${src}/"
            [[ "$dest" == */ ]] || dest="${dest}/"
            sudo rsync -a --quiet "$src" "$dest"
        else
            # File: copy
            sudo cp -a "$src" "$dest"
        fi
        info "  Copied: $entry"
    done

    # Inject umask 002 at the top of sandbox .bashrc if not already present
    local sandbox_bashrc="${SANDBOX_HOME}/.bashrc"
    if [[ -f "$sandbox_bashrc" ]]; then
        if ! sudo grep -q '^umask 002' "$sandbox_bashrc"; then
            info "Injecting umask 002 into $sandbox_bashrc"
            sudo sed -i '1i umask 002' "$sandbox_bashrc"
        fi
    else
        info "Creating $sandbox_bashrc with umask 002"
        echo "umask 002" | sudo tee "$sandbox_bashrc" >/dev/null
    fi
}
```

- [ ] **Step 2: Test dotfile sync**

```bash
sudo ./svl build
sudo ls -la /home/sandvault-ubuntu/.gitconfig
sudo ls -la /home/sandvault-ubuntu/.tmux.conf
sudo ls -la /home/sandvault-ubuntu/.local/bin/claude
sudo head -1 /home/sandvault-ubuntu/.bashrc
```

Expected:
- Dotfiles that exist on host are copied to sandbox home
- First line of `.bashrc` is `umask 002`
- Missing dotfiles are silently skipped

- [ ] **Step 3: Commit**

```bash
git add svl
git commit -m "feat(svl): add dotfile sync with umask injection"
```

---

### Task 5: `svl build` — SSH key setup

**Files:**
- Modify: `svl`

- [ ] **Step 1: Add the SSH setup function**

```bash
build_ssh() {
    local host_home
    host_home="$(getent passwd "$HOST_USER" | cut -d: -f6)"
    local ssh_dir="${SANDBOX_HOME}/.ssh"
    local auth_keys="${ssh_dir}/authorized_keys"

    info "Setting up SSH access"

    sudo mkdir -p "$ssh_dir"
    sudo chmod 700 "$ssh_dir"

    # Collect all public keys from host
    local pub_keys=()
    for f in "${host_home}"/.ssh/*.pub; do
        [[ -f "$f" ]] || continue
        pub_keys+=("$f")
    done

    if [[ ${#pub_keys[@]} -eq 0 ]]; then
        warn "No SSH public keys found in ${host_home}/.ssh/*.pub — skipping SSH setup"
        return
    fi

    # Concatenate all public keys into authorized_keys
    sudo bash -c "cat $(printf '%q ' "${pub_keys[@]}") > ${auth_keys}"
    sudo chmod 600 "$auth_keys"
    sudo chown -R "${SANDBOX_USER}:${SANDBOX_USER}" "$ssh_dir"

    info "Installed ${#pub_keys[@]} public key(s) into $auth_keys"
}
```

- [ ] **Step 2: Test SSH setup**

```bash
sudo ./svl build
sudo cat /home/sandvault-ubuntu/.ssh/authorized_keys
sudo ls -la /home/sandvault-ubuntu/.ssh/
```

Expected:
- `.ssh/` is mode 700
- `authorized_keys` contains all public keys from host, mode 600
- Owned by `sandvault-ubuntu:sandvault-ubuntu`

- [ ] **Step 3: Commit**

```bash
git add svl
git commit -m "feat(svl): add SSH authorized_keys setup from host public keys"
```

---

### Task 6: `svl shell`, `svl ssh`, `svl run`

**Files:**
- Modify: `svl`

- [ ] **Step 1: Implement the three access commands**

Replace the stubs:

```bash
cmd_shell() {
    require_sandbox
    exec sudo -u "$SANDBOX_USER" -i "$@"
}

cmd_ssh() {
    require_sandbox
    exec ssh "$SANDBOX_USER@localhost" "$@"
}

cmd_run() {
    [[ $# -ge 1 ]] || die "Usage: svl run <command> [args...]"
    require_sandbox
    exec sudo -u "$SANDBOX_USER" -- "$@"
}
```

- [ ] **Step 2: Test shell access**

```bash
./svl shell -c "whoami && pwd && umask"
```

Expected output:
```
sandvault-ubuntu
/home/sandvault-ubuntu
0002
```

- [ ] **Step 3: Test run access**

```bash
./svl run id
./svl run ls -la /home/sandvault-ubuntu/
```

Expected: shows sandbox user's id, lists sandbox home.

- [ ] **Step 4: Test piping**

```bash
echo "whoami; exit" | ./svl shell
```

Expected: prints `sandvault-ubuntu`

- [ ] **Step 5: Test SSH access**

```bash
./svl ssh whoami
```

Expected: prints `sandvault-ubuntu` (if sshd is running and host key is accepted).

- [ ] **Step 6: Commit**

```bash
git add svl
git commit -m "feat(svl): implement shell, ssh, and run commands"
```

---

### Task 7: `svl stop`

**Files:**
- Modify: `svl`

- [ ] **Step 1: Implement stop**

Replace the stub:

```bash
cmd_stop() {
    require_sandbox
    local uid
    uid="$(id -u "$SANDBOX_USER")"

    info "Stopping all processes for $SANDBOX_USER (uid $uid)"

    # SIGTERM first
    if sudo pkill -u "$SANDBOX_USER" 2>/dev/null; then
        info "Sent SIGTERM, waiting 3 seconds..."
        sleep 3
        # SIGKILL stragglers
        if sudo pkill -9 -u "$SANDBOX_USER" 2>/dev/null; then
            info "Sent SIGKILL to remaining processes"
        fi
    else
        info "No processes running"
    fi
}
```

- [ ] **Step 2: Test stop**

```bash
# Start a background process as sandbox user
./svl run sleep 300 &
sleep 1
./svl status    # should show processes
./svl stop      # should kill them
./svl status    # should show no processes
```

- [ ] **Step 3: Commit**

```bash
git add svl
git commit -m "feat(svl): implement stop command with SIGTERM/SIGKILL"
```

---

### Task 8: `svl status`

**Files:**
- Modify: `svl`

- [ ] **Step 1: Implement status**

Replace the stub:

```bash
cmd_status() {
    echo "Sandbox user:  $SANDBOX_USER"
    echo "Sandbox home:  $SANDBOX_HOME"

    # Check if user exists
    if sandbox_user_exists; then
        echo "User exists:   yes"
        echo "User shell:    $(getent passwd "$SANDBOX_USER" | cut -d: -f7)"
        echo "User uid:      $(id -u "$SANDBOX_USER")"
        echo "User groups:   $(id -nG "$SANDBOX_USER")"
    else
        echo "User exists:   no"
        return
    fi

    # Check running processes
    local procs
    procs="$(pgrep -u "$SANDBOX_USER" -c 2>/dev/null || true)"
    echo "Processes:     ${procs:-0}"

    # Check home directory
    if [[ -d "$SANDBOX_HOME" ]]; then
        local usage
        usage="$(sudo du -sh "$SANDBOX_HOME" 2>/dev/null | cut -f1)"
        echo "Home disk:     ${usage:-unknown}"
    else
        echo "Home disk:     (directory does not exist)"
    fi

    # Check sudoers
    if [[ -f "$SUDOERS_FILE" ]]; then
        echo "Sudoers rule:  $SUDOERS_FILE"
    else
        echo "Sudoers rule:  not installed"
    fi

    # Check SSH
    if [[ -f "${SANDBOX_HOME}/.ssh/authorized_keys" ]]; then
        local nkeys
        nkeys="$(sudo wc -l < "${SANDBOX_HOME}/.ssh/authorized_keys")"
        echo "SSH keys:      $nkeys"
    else
        echo "SSH keys:      none"
    fi
}
```

- [ ] **Step 2: Test status**

```bash
./svl status
```

Expected: shows user exists, uid, groups, process count, disk usage, sudoers and SSH status.

- [ ] **Step 3: Commit**

```bash
git add svl
git commit -m "feat(svl): implement status command"
```

---

### Task 9: `svl sync`

**Files:**
- Modify: `svl`

- [ ] **Step 1: Implement sync**

Replace the stub:

```bash
cmd_sync() {
    require_sandbox
    info "Syncing dotfiles and credentials to $SANDBOX_HOME"
    sync_dotfiles
    build_ssh
    info "Fixing ownership"
    sudo chown -R "${SANDBOX_USER}:${SANDBOX_USER}" "$SANDBOX_HOME"
    info "Sync complete"
}
```

- [ ] **Step 2: Test sync**

```bash
./svl sync
```

Expected: re-copies dotfiles and SSH keys, fixes ownership. No errors.

- [ ] **Step 3: Commit**

```bash
git add svl
git commit -m "feat(svl): implement sync command"
```

---

### Task 10: `svl uninstall`

**Files:**
- Modify: `svl`

- [ ] **Step 1: Implement uninstall**

Replace the stub:

```bash
cmd_uninstall() {
    require_sandbox

    echo "This will permanently remove:"
    echo "  - User: $SANDBOX_USER"
    echo "  - Home: $SANDBOX_HOME"
    echo "  - Group: $SANDBOX_USER"
    echo "  - Sudoers: $SUDOERS_FILE"
    echo ""
    read -r -p "Are you sure? [y/N] " confirm
    [[ "$confirm" =~ ^[Yy]$ ]] || { info "Aborted."; return; }

    info "Stopping processes..."
    cmd_stop

    info "Removing sudoers rule"
    sudo rm -f "$SUDOERS_FILE"

    info "Removing user and home directory"
    sudo userdel -r "$SANDBOX_USER" 2>/dev/null || warn "userdel failed (user may already be removed)"

    info "Removing group"
    sudo groupdel "$SANDBOX_USER" 2>/dev/null || warn "groupdel failed (group may already be removed)"

    # Remove host user from sandbox group (may already be gone after groupdel)
    sudo gpasswd -d "$HOST_USER" "$SANDBOX_USER" 2>/dev/null || true

    info "Uninstall complete"
}
```

- [ ] **Step 2: Test uninstall**

```bash
./svl uninstall
# Type 'y' when prompted
./svl status
```

Expected:
- Asks for confirmation
- Stops processes, removes user, home, group, sudoers
- `svl status` shows user does not exist

- [ ] **Step 3: Verify clean state**

```bash
getent passwd sandvault-ubuntu    # should return nothing
getent group sandvault-ubuntu     # should return nothing
ls /home/sandvault-ubuntu         # should not exist
ls /etc/sudoers.d/50-sandvault-ubuntu  # should not exist
```

- [ ] **Step 4: Commit**

```bash
git add svl
git commit -m "feat(svl): implement uninstall command with confirmation"
```

---

### Task 11: End-to-end test

**Files:**
- None modified — this is a manual verification task

- [ ] **Step 1: Full build-to-uninstall cycle**

```bash
# Clean start
./svl status

# Build
./svl build

# Verify status
./svl status

# Test shell
./svl shell -c "whoami && pwd && umask && id"

# Test run
./svl run ls -la ~/

# Test file access from host
touch /home/sandvault-ubuntu/test-from-host
ls -la /home/sandvault-ubuntu/test-from-host
# Should be owned by ubuntu:sandvault-ubuntu

# Test file access from sandbox
./svl run touch /home/sandvault-ubuntu/test-from-sandbox
ls -la /home/sandvault-ubuntu/test-from-sandbox
# Should be owned by sandvault-ubuntu:sandvault-ubuntu

# Test SSH
./svl ssh whoami

# Test piping
echo "echo hello from pipe; exit" | ./svl shell

# Test stop
./svl run sleep 300 &
sleep 1
./svl stop

# Sync after changing a dotfile
./svl sync

# Clean teardown
./svl uninstall
./svl status
```

- [ ] **Step 2: Verify idempotent build**

```bash
./svl build
./svl build   # should succeed, printing "already exists" messages
./svl uninstall
```

- [ ] **Step 3: Commit any fixes discovered during testing**

```bash
git add svl
git commit -m "fix(svl): fixes from end-to-end testing"
```
