# SVM — macOS Sandbox User Tool

`svm` creates and manages a sandboxed macOS user for running AI agents, untrusted code, and general-purpose isolation. The sandbox user is an unprivileged macOS user with kernel-level restrictions via `sandbox-exec` — file writes are denied everywhere except the sandbox home and temp directories, and external drives are inaccessible.

**Sandbox user:** `sandvault-<your-username>` (e.g., `sandvault-pat`)
**Sandbox home:** `/Users/sandvault-<your-username>/`

## Quick start

```bash
./svm install    # create the sandbox user
./svm shell      # drop into a sandboxed shell
./svm ssh        # SSH in (works from other machines too)
./svm stop       # kill all sandbox processes
./svm uninstall  # remove everything
```

## Commands

### `svm install`

Creates the sandbox environment. Idempotent — safe to re-run.

What it does:
1. Creates a macOS group `sandvault-$USER` via `dscl`
2. Creates a macOS user `sandvault-$USER` with a unique UID, matching your login shell, home at `/Users/sandvault-$USER`
3. Sets a random password and hides the user from the login screen (`IsHidden 1`)
4. Removes the sandbox user from the `staff` group (prevents access to `/usr/local`, `/opt/homebrew`, and other staff-writable paths)
5. Adds your user to the sandbox group (for shared file access)
6. Sets sandbox home to `0750` with subdirectories at `2770` (setgid + group-writable)
7. Installs a sudoers rule at `/etc/sudoers.d/50-sandvault-$USER` allowing you to run commands as the sandbox user without a password
8. Creates a `sandbox-exec` profile at `/var/sandvault/sandbox-sandvault-$USER.sb` that:
   - Denies all file writes except to sandbox home, `/tmp`, `/private/tmp`, `/var/folders`, and `/dev`
   - Blocks reading `/Volumes` (external and removable drives)
   - Allows reading the main disk (`/Volumes/Macintosh HD`)
   - Allows process execution and introspection
9. Copies dotfiles, configs, editor settings, agent credentials, and CLI tools from your home (skips anything that doesn't exist)
10. Injects `umask 002` and agent aliases (`claude`, `codex`, `gemini` with permission-bypass flags) into the sandbox `.zshrc` and `.bashrc`
11. Copies all your `~/.ssh/*.pub` keys into the sandbox user's `authorized_keys` and adds the user to `com.apple.access_ssh`
12. Fixes ownership on all copied files

After install, you may need to log out and back in (or run `newgrp sandvault-$USER`) for group membership to take effect.

### `svm shell [args...]`

Opens an interactive sandboxed login shell as the sandbox user. The shell runs through `sandbox-exec` with the installed profile — file writes outside the sandbox home are blocked at the kernel level. Extra arguments are forwarded.

The launch chain: `sudo` → `env -i` (clean environment) → `sandbox-exec -f profile.sb` → login shell.

### `svm ssh [args...]`

SSH into the sandbox user on localhost. Extra arguments are forwarded to `ssh`. Works from remote machines too — add public keys to the sandbox user's `~/.ssh/authorized_keys`.

Note: SSH sessions are not wrapped by `sandbox-exec`. The kernel sandbox only applies to `shell` and `run` commands.

### `svm run <command> [args...]`

Run a single sandboxed command as the sandbox user. The command runs through `sandbox-exec` with the same kernel-level restrictions as `svm shell`. Returns the command's exit code.

Examples:
```bash
svm run whoami           # prints: sandvault-ubuntu
svm run ls -la ~/        # list sandbox home
svm run claude           # run claude in sandboxed environment
```

### `svm stop`

Kill all processes running as the sandbox user. Uses `launchctl bootout` to terminate the user session, then `pkill -9` for any stragglers. No automatic cleanup — processes persist until you explicitly stop them.

### `svm uninstall`

Permanently removes the sandbox environment. Prompts for confirmation before proceeding.

What it removes:
1. All sandbox user processes (via `svm stop`)
2. The sudoers rule
3. The `sandbox-exec` profile
4. The user account (via `dscl`)
5. The home directory
6. The group (via `dscl`)
7. The host user's membership in the sandbox group
8. The sandbox user's membership in `com.apple.access_ssh`

### `svm status`

Shows the current state of the sandbox:
- Whether the user exists, its UID, shell, and hidden status
- Number of running processes
- Disk usage of the sandbox home
- Whether the sudoers rule is installed
- Whether the sandbox profile is installed
- Number of SSH authorized keys

### `svm sync`

Re-copies dotfiles, configs, and credentials from your home to the sandbox home. Also refreshes SSH authorized keys. Use this after you update your configs or credentials.

## What gets copied

### Shell configs
`.bashrc`, `.bash_profile`, `.profile`, `.zshrc`, `.zshenv`, `.zprofile`, `.zlogin`, `.inputrc`, `.screenrc`, `.hushlogin`

### Editor and TUI tools
`.vimrc`, `.nanorc`, `.tmux.conf`, `.tmux.tcl`, `.config/nvim/`, `.config/htop/`, `.config/kitty/`, `.config/alacritty/`, `.config/wezterm/`, `.config/nnn/`, `.config/vifm/`, `.config/sc-im/`, `.config/lazygit/`, `.config/lazy-github/`, `.config/gitui/`, `.config/starship.toml`

### Dev tools
`.gitconfig`, `.gitignore`, `.bazelrc`, `.editorconfig`, `.config/git/`, `.config/gh/`, `.config/uv/`, `.config/opencode/`, `.config/pip/`, `.npmrc`, `.gemrc`, `.curlrc`, `.wgetrc`, `.ripgreprc`, `.fdignore`, `.prettierrc`, `.eslintrc.json`

### Agent credentials
`.claude/.credentials.json`, `.claude/settings.json`, `.claude/plugins/`, `.codex/auth.json`, `.codex/config.toml`

### Binaries and runtimes
`.local/bin/`, `.bun/`, `.cargo/`, `bin/`

## Security model

### User isolation
- The sandbox user is **not** in the `staff` group (no access to `/usr/local`, `/opt/homebrew`)
- The sandbox user is hidden from the login screen
- Password is randomized (SSH key-only access)
- The sudoers rule only allows your user to run commands as the sandbox user — no root escalation

### Kernel sandbox (`sandbox-exec`)
- **All file writes denied** except to: sandbox home, `/tmp`, `/private/tmp`, `/var/folders`, `/dev`
- **External drives blocked** — cannot read anything under `/Volumes` except the main disk
- Process execution and introspection are allowed
- Applied to `svm shell` and `svm run` commands (not SSH sessions)

### File access
- Your host account **can** read all sandbox files (via group membership) and write inside subdirectories
- The sandbox user **cannot** read your home directory
- The sandbox user **cannot** modify system configurations

## Files created outside the sandbox home

| Path | Purpose |
|------|---------|
| `/etc/sudoers.d/50-sandvault-$USER` | Passwordless sudo rule |
| `/var/sandvault/sandbox-sandvault-$USER.sb` | Kernel sandbox profile (root-owned, read-only) |

## Differences from `svl` (Linux version)

| Feature | `svl` (Linux) | `svm` (macOS) |
|---------|---------------|---------------|
| Kernel sandbox | None | `sandbox-exec` restricts writes and drive access |
| User management | `useradd`/`groupadd` | `dscl` |
| Staff group | N/A | Removed from `staff` |
| Process kill | `pkill` | `launchctl bootout` + `pkill` |
| SSH group | Not needed | `com.apple.access_ssh` |
| Home location | Inside host home (`~/sandvault`) | Standard (`/Users/sandvault-$USER`) |
| Hidden user | N/A | `IsHidden 1` |
