# SVL — Linux Sandbox User Tool

`svl` creates and manages a sandboxed Linux user for running AI agents, untrusted code, and general-purpose isolation. The sandbox user is a plain unprivileged Linux user — security comes from standard Unix user/group separation.

**Sandbox user:** `sandvault-<your-username>` (e.g., `sandvault-ubuntu`)
**Sandbox home:** `~/<your-username>/sandvault/`

## Quick start

```bash
./svl install    # create the sandbox user
./svl shell      # drop into a shell as the sandbox user
./svl ssh        # SSH in (works from other machines too)
./svl stop       # kill all sandbox processes
./svl uninstall  # remove everything
```

## Commands

### `svl install`

Creates the sandbox environment. Idempotent — safe to re-run.

What it does:
1. Creates a Linux group `sandvault-$USER`
2. Creates a Linux user `sandvault-$USER` with home inside your home directory, matching your login shell
3. Locks the user's password (no password login — SSH key-only)
4. Adds your user to the sandbox group (for shared file access)
5. Grants the sandbox user traverse permission on your home directory (execute-only ACL — cannot list or read your files)
6. Sets sandbox home to `0750` with subdirectories at `2770` (setgid + group-writable)
7. Installs a sudoers rule at `/etc/sudoers.d/50-sandvault-$USER` allowing you to run commands as the sandbox user without a password
8. Copies dotfiles, configs, editor settings, agent credentials, and CLI tools from your home (skips anything that doesn't exist)
9. Injects `umask 002` and agent aliases (`claude`, `codex`, `gemini` with permission-bypass flags) into the sandbox `.bashrc`
10. Copies all your `~/.ssh/*.pub` keys into the sandbox user's `authorized_keys`
11. Fixes ownership on all copied files

After install, you may need to log out and back in (or run `newgrp sandvault-$USER`) for group membership to take effect.

### `svl shell [args...]`

Opens an interactive login shell as the sandbox user via `sudo -u sandvault-$USER -i`. Extra arguments are forwarded to `sudo`.

Supports piping: `echo "pwd; exit" | svl shell`

### `svl ssh [args...]`

SSH into the sandbox user on localhost. Extra arguments are forwarded to `ssh`. Works from remote machines too — add public keys to the sandbox user's `~/.ssh/authorized_keys`.

### `svl run <command> [args...]`

Run a single command as the sandbox user. Non-interactive — returns the command's exit code.

Examples:
```bash
svl run whoami           # prints: sandvault-ubuntu
svl run ls -la ~/        # list sandbox home
svl run claude           # run claude as sandbox user
```

### `svl stop`

Kill all processes running as the sandbox user. Sends SIGTERM first, waits 3 seconds, then SIGKILL for any stragglers. No automatic cleanup — processes persist until you explicitly stop them.

### `svl uninstall`

Permanently removes the sandbox environment. Prompts for confirmation before proceeding.

What it removes:
1. All sandbox user processes (via `svl stop`)
2. The sudoers rule
3. The user account and home directory
4. The group
5. The host user's membership in the sandbox group
6. The traverse ACL on the host home directory

### `svl status`

Shows the current state of the sandbox:
- Whether the user exists, its UID, shell, and groups
- Number of running processes
- Disk usage of the sandbox home
- Whether the sudoers rule is installed
- Number of SSH authorized keys

### `svl sync`

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

- The sandbox user is **not** in `sudo`, `docker`, `adm`, or any privileged groups
- The sandbox user **cannot** read your home directory (only has execute/traverse permission via ACL)
- The sandbox user **cannot** modify system configurations
- Your host account **can** read all sandbox files (via group membership) and write inside subdirectories
- SSH access is key-only (password is locked)
- The sudoers rule only allows your user to run commands as the sandbox user — no root escalation

## Files created outside the sandbox home

| Path | Purpose |
|------|---------|
| `/etc/sudoers.d/50-sandvault-$USER` | Passwordless sudo rule |
| ACL on `$HOME` | Execute-only traverse for sandbox user |
