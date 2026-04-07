# SVL — Linux Sandbox User Tool

**Date**: 2026-04-07
**Status**: Approved

## Overview

`svl` is a standalone bash script that creates and manages a sandboxed Linux user
for running headless agents, untrusted code, and general-purpose isolation. It provides
passwordless local and remote shell access, shared file access from the host account,
and clean teardown commands.

The sandbox user is a plain unprivileged Linux user — no namespace isolation, no
AppArmor profiles, no container layers. Security comes from standard Unix user/group
separation: the sandbox user cannot read the host home directory, cannot sudo, and
has no membership in privileged groups.

## User & Group Setup

`svl build` creates:

- **Group** `sandvault-$USER` — shared group for file access between host and sandbox.
- **User** `sandvault-$USER` with:
  - Home directory at `/home/$USER/sandvault` (inside the host user's home)
  - Login shell matching the host user's shell (e.g., `/bin/bash`)
  - Primary group: `sandvault-$USER`
  - Password locked (`usermod -L`) — no password login possible
  - Not added to `sudo`, `docker`, `adm`, or any system groups
- **Host user added to `sandvault-$USER` group** — gives full access to sandbox home.
- **Sandbox home permissions**: `2770` (setgid + owner/group full access). All new
  files/directories inherit the `sandvault-$USER` group.
- **Default umask `002`** in sandbox `.bashrc` — ensures group-writable files.

### Access model

- Sandbox user **cannot** read `/home/$USER` (host home is `0750` owned by `$USER:$USER`).
- Sandbox user **cannot** modify system configs (not in sudo group).
- Host user **can** freely read/write/delete anything in `/home/sandvault-$USER` (via group).
- Files created by either user in sandbox home get group `sandvault-$USER` (setgid) and
  are group-writable (umask 002).

## Passwordless Access

### Local shell

Sudoers rule at `/etc/sudoers.d/50-sandvault-$USER`:

```
$USER ALL=(sandvault-$USER) NOPASSWD: ALL
```

Allows running anything as the sandbox user without password. The sandbox user itself
is unprivileged, so this grants no privilege escalation.

Usage: `svl shell` runs `sudo -u sandvault-$USER -i bash`. Extra args forwarded to sudo.

### SSH

- During `svl build`, all `~/.ssh/*.pub` from the host are concatenated into
  `/home/sandvault-$USER/.ssh/authorized_keys`.
- The sandbox user's account is password-locked, so SSH is key-only.
- `svl ssh` is a convenience for `ssh sandvault-$USER@localhost`. Extra args forwarded to ssh.
- For remote access: add additional public keys to the sandbox user's `authorized_keys`.
- No changes to `/etc/ssh/sshd_config` needed — default Ubuntu config allows key-based
  login for all users.

## Home Directory Setup

During `svl build`, dotfiles and tools are copied from the host user's home via `rsync`.
Each entry is copied **if it exists**, skipped silently if not.

### Shell
- `.bashrc`, `.bash_profile`, `.profile`
- `.zshrc`, `.zshenv`, `.zprofile`, `.zlogin`
- `.inputrc`, `.screenrc`, `.hushlogin`

### Editor / TUI
- `.vimrc`, `.nanorc`
- `.tmux.conf`, `.tmux.tcl`
- `.config/nvim/`
- `.config/htop/`
- `.config/kitty/`, `.config/alacritty/`, `.config/wezterm/`
- `.config/nnn/`, `.config/vifm/`, `.config/sc-im/`
- `.config/lazygit/`, `.config/lazy-github/`, `.config/gitui/`
- `.config/starship.toml`

### Dev tools
- `.gitconfig`, `.gitignore`, `.bazelrc`, `.editorconfig`
- `.config/git/`, `.config/gh/`, `.config/uv/`, `.config/opencode/`, `.config/pip/`
- `.npmrc`, `.gemrc`, `.curlrc`, `.wgetrc`
- `.ripgreprc`, `.fdignore`, `.prettierrc`, `.eslintrc.json`

### Agent credentials
- `.claude/.credentials.json`, `.claude/settings.json`, `.claude/plugins/`
- `.codex/auth.json`, `.codex/config.toml`

### Binaries / runtimes
- `.local/bin/`
- `.bun/`
- `.cargo/`
- `~/bin/`

### Post-copy

- Inject `umask 002` at the top of sandbox `.bashrc`
- Set ownership: `chown -R sandvault-$USER:sandvault-$USER /home/sandvault-$USER`

## Commands

| Command | Description |
|---------|-------------|
| `svl build` | Create sandbox user, group, sudoers, home skeleton, SSH keys. Idempotent — safe to re-run. |
| `svl shell [args...]` | Open login shell as sandbox user via sudo. Extra args forwarded to sudo. |
| `svl ssh [args...]` | SSH into sandbox user on localhost. Extra args forwarded to ssh. |
| `svl run <cmd> [args...]` | Run a single command as sandbox user via sudo (non-interactive, returns exit code). |
| `svl stop` | Kill all sandbox user processes (SIGTERM, then SIGKILL). |
| `svl uninstall` | Full teardown: stop, delete user+home, group, sudoers. Confirms first. |
| `svl status` | Show if sandbox user exists, running processes, home disk usage. |
| `svl sync` | Re-copy dotfiles/credentials from host to sandbox home. |

Piping supported: `echo "pwd; exit" | svl shell` forwards stdin.

No automatic rebuild on shell/ssh — explicitly `svl build` or `svl sync`.

## Build Sequence

1. Verify running as non-root user with sudo access.
2. `sudo groupadd sandvault-$USER`
3. `sudo useradd -m -g sandvault-$USER -s $SHELL -d /home/sandvault-$USER sandvault-$USER`
4. `sudo usermod -L sandvault-$USER`
5. `sudo usermod -aG sandvault-$USER $USER`
6. `sudo chmod 2770 /home/sandvault-$USER`
7. Install sudoers rule to `/etc/sudoers.d/50-sandvault-$USER`, validate with `visudo -c`.
8. Copy dotfiles/configs via `rsync` (skip missing sources).
9. Inject `umask 002` into sandbox `.bashrc`.
10. Copy `~/.ssh/*.pub` into sandbox `~/.ssh/authorized_keys`.
11. `sudo chown -R sandvault-$USER:sandvault-$USER /home/sandvault-$USER`

Each step checks if already done before acting (idempotent).

## Stop Sequence

1. `sudo pkill -u sandvault-$USER` (SIGTERM)
2. Brief pause
3. `sudo pkill -9 -u sandvault-$USER` (SIGKILL for stragglers)

## Uninstall Sequence

1. Prompt for confirmation.
2. `svl stop`
3. `sudo userdel -r sandvault-$USER` (removes user + home directory)
4. `sudo groupdel sandvault-$USER`
5. `sudo rm /etc/sudoers.d/50-sandvault-$USER`
6. `sudo gpasswd -d $USER sandvault-$USER` (remove host user from group if still present)

## Non-goals

- No namespace isolation, AppArmor profiles, or container layers.
- No automatic cleanup of sandbox processes.
- No automatic rebuild on shell entry.
- No shared workspace directory (host accesses sandbox home directly via group).
- No macOS support (use `sv` for macOS).
