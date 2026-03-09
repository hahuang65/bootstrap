# Research: Bootstrap & Dotfiles Usability and Maintainability

## Overview

This is a personal dotfiles repository that bootstraps and synchronizes package installations and tool configurations across macOS and Linux (Arch/CachyOS) machines. It uses a modular architecture with 115 git submodules, each representing one tool's configuration, linked together by a top-level `install.sh` and a `bootstrap/configure` script.

The primary concern: this repo is edited infrequently, so when something breaks the developer spends significant time re-orienting.

## Architecture

```
~/.dotfiles/
├── install.sh              # Loops all subdirs, runs their install.sh
├── bootstrap/
│   ├── configure           # Machine setup: pkg manager + packages + dotfiles
│   ├── Brewfile            # macOS packages (brew, cask)
│   ├── Parufile            # Linux user packages (AUR)
│   └── Parufile.system     # Linux system packages
├── mise/
│   ├── install.sh          # Symlinks default-*-packages, installs mise
│   ├── default-go-packages
│   ├── default-npm-packages
│   ├── default-python-packages
│   └── default-ruby-gems
├── bashrc/
│   ├── bashrc              # Main init (source_file/source_dir helpers)
│   ├── aliases
│   ├── functions/          # 6 utility functions
│   └── customizations/     # 18 shell customization modules
├── git/config              # Comprehensive git config (146 lines)
├── nvim/                   # Neovim config (lazy.nvim, 49 plugins)
├── ai/                     # Claude Code + OpenCode configuration
├── scripts/shared/         # 152+ utility scripts
└── [35 more tool config dirs, each a git submodule]
```

## Key Files

| File | Role |
|------|------|
| `bootstrap/configure` | Entry point for new machines. Installs pkg manager, packages, dotfiles |
| `bootstrap/Brewfile` | macOS Homebrew package manifest (97 formulae, 33 casks) |
| `bootstrap/Parufile` | Linux user-level AUR packages (71 packages) |
| `bootstrap/Parufile.system` | Linux system packages (217 packages) |
| `install.sh` | Top-level installer: loops all subdirs with install.sh |
| `mise/install.sh` | Symlinks default package files to `~/`, installs mise, sets global tool versions |
| `mise/default-go-packages` | 20 Go CLI tools installed via mise |
| `bashrc/bashrc` | Shell initialization with benchmark mode and modular sourcing |

## Data Flow

### Bootstrap flow (new machine)

1. Run `bootstrap/configure` directly
2. Detects OS (Darwin/Linux)
3. Ensures package manager exists (Homebrew or Paru)
4. Calls `sync` → installs packages from Brewfile or Parufile
5. Calls `install_dotfiles` → clones repo, inits submodules, runs `./install.sh`
6. Top-level `install.sh` loops every subdir, running each module's `install.sh`

### Sync flow (existing machine)

```bash
source bootstrap/configure  # Source the file to get functions
sync                         # Call sync function directly
```

On macOS: `brew bundle check || brew bundle install && brew bundle cleanup --force && brew bundle dump --force`
On Linux: Custom `sync_paru` function that diffs installed vs. desired packages

### Mise flow

`mise/install.sh` symlinks `default-*-packages` files to `~/`, then runs `mise use -gy golang@latest node@latest python@latest awscli@latest`. Mise reads the `~/.default-*-packages` files when installing new language versions.

## Patterns & Conventions

- **One submodule per tool**: Each config dir is a separate sr.ht repo
- **install.sh per module**: Convention for self-contained setup
- **OS detection**: `uname` or `$OSTYPE` used throughout
- **Shell helpers**: `source_file`/`source_dir` with optional `BENCHMARK=1` timing
- **Load ordering**: Functions → aliases → customizations (minus direnv) → direnv last

## Dependencies

- **Homebrew** (macOS): Main package manager
- **Paru** (Linux): AUR helper wrapping pacman
- **Mise**: Polyglot version manager (Go, Node, Python, Ruby, awscli)
- **Git submodules**: 115 submodules for modular config management

## Edge Cases & Gotchas

### 1. Package duplication between Brew and Mise (HIGH)

Several tools are installed by **both** Homebrew and mise's `default-go-packages`:

| Package | In Brewfile | In mise default-go-packages |
|---------|------------|---------------------------|
| `gum` | `brew "gum"` (line 49) | `github.com/charmbracelet/gum` |
| `hut` | `brew "hut"` (line 53) | `git.sr.ht/~emersion/hut` |
| `go` | `brew "go"` (line 47) | Managed by `mise use -gy golang@latest` |
| `pre-commit` | `brew "pre-commit"` (line 74) | In `default-python-packages` |
| `uv` | — | In `default-python-packages`, also in Parufile |

This creates confusion about which version is actually on `$PATH` (depends on PATH ordering), wastes disk space, and makes it unclear where to add/remove a package. The `brew "go"` is particularly concerning since mise also manages Go — you end up with two Go installations.

### 2. `sync_paru` is destructive with no dry-run (HIGH)

The Linux `sync_paru` function (lines 51-128 of `configure`) computes packages to remove and runs `paru -Rns --noconfirm` automatically. If the Parufile is accidentally truncated or a package name changes, this will silently uninstall packages. There's no confirmation prompt, no dry-run mode, and no log of what was removed.

### 3. `install.sh` processes directories in glob order (MEDIUM)

The top-level `install.sh` iterates `*/` which gives filesystem-sorted order. Some modules may depend on others (e.g., `mise/install.sh` installs mise which `nvim/install.sh` might need). There's no explicit dependency ordering — it works by coincidence of alphabetical order (`mise` < `nvim`).

### 4. `brew bundle dump --force` overwrites the Brewfile (MEDIUM)

In the `sync` function (line 136):
```bash
brew bundle check || brew bundle install && brew bundle cleanup --force && brew bundle dump --force
```

The `brew bundle dump --force` at the end overwrites the curated Brewfile with whatever is currently installed. This means:
- Manually removed packages get re-added if they were installed as dependencies
- The Brewfile ordering/organization is lost
- Comments would be destroyed (though there are none currently)
- Any edits you make to the Brewfile get overwritten on next sync

This also has an operator precedence issue: `&&` binds tighter than `||`, so the actual evaluation is: `check || (install && cleanup && dump)`. If `check` succeeds, none of the rest runs, which is probably the intent — but if `check` fails and `install` succeeds, it always runs cleanup and dump.

### 5. No error handling in top-level `install.sh` (MEDIUM)

```bash
#!/bin/sh
for dir in */; do
  pushd $dir > /dev/null
  if test -e "install.sh"; then
    ./install.sh
  fi
  popd > /dev/null
done
```

- Uses `#!/bin/sh` but `pushd`/`popd` are bash-isms (not POSIX)
- No `set -e` — a failing install.sh is silently ignored
- No logging of which modules were installed
- `$dir` is unquoted (would break on spaces, though unlikely here)
- `rm -rf "$scripts_dir"` at the top wipes the scripts directory before any module runs — if the scripts install fails partway, you lose the old scripts and don't get new ones

### 6. The `bootstrap` function ordering is fragile (MEDIUM)

```bash
function bootstrap {
  sync        # Install packages
  install_dotfiles  # Clone and install dotfiles
}
```

`sync` runs first, which installs packages. But `install_dotfiles` clones the repo and runs `./install.sh`, which includes `mise/install.sh` which tries to `brew install mise` — but brew should already be available since `sync` ran. However, if this is a first-time bootstrap, the Brewfile includes `brew "mise"` so mise gets installed during `sync`. Then `mise/install.sh` checks for mise and finds it. This works but is implicit and confusing.

### 7. mise `install.sh` hardcodes global tool versions (LOW)

```bash
mise use -gy golang@latest node@latest python@latest awscli@latest
```

This pins to `@latest` which means every `install.sh` run potentially upgrades all language runtimes. On an infrequently-synced machine, this could trigger large downloads and break projects expecting specific versions.

### 8. `asdf` compatibility shim is macOS-only and uncommented (LOW)

`mise/install.sh` line 9:
```bash
if [ "$(uname)" = "Darwin" ]; then
  ln -sf "${PWD}/asdf" "${HOME}/.local/bin/asdf"
fi
```

And `bashrc/customizations/mise.bash`:
```bash
if [[ $OSTYPE == darwin* ]]; then
  export MISE_ASDF_COMPAT=1
fi
```

No explanation of why asdf compatibility is needed only on macOS, or what depends on it.

## Current State: Specific Improvement Recommendations

### A. Resolve Brew/Mise package duplication

**Problem**: `gum`, `hut`, and `go` are in both Brewfile and mise. `pre-commit` is in both Brewfile and `default-python-packages`.

**Impact**: Unclear which version runs, wasted disk space, two places to update.

**Suggestion**: Decide on one source of truth per package:
- Language runtimes (go, python, node, ruby): mise only → remove `brew "go"` from Brewfile
- Go CLI tools: mise's `default-go-packages` only (already done in this session for most)
- `gum` and `hut`: Choose one — brew formula is simpler and auto-updated, but mise's go install tracks latest source
- `pre-commit`: Choose brew or mise's python packages, not both

### B. Remove `brew bundle dump --force` from sync

**Problem**: Overwrites the curated Brewfile with auto-generated content.

**Impact**: Loses intentional organization, re-adds removed packages, makes Brewfile a moving target.

**Suggestion**: Remove `brew bundle dump --force` from the sync chain. The Brewfile should be manually curated. Use `brew bundle cleanup --force` to remove unlisted packages, but don't dump back.

### C. Add dry-run mode to `sync_paru`

**Problem**: Auto-removes packages with `--noconfirm` and no preview.

**Impact**: Accidental data loss if Parufile is wrong.

**Suggestion**: Add a `--dry-run` flag or at least print the removal list and pause for confirmation before `paru -Rns`.

### D. Add error handling and logging to `install.sh`

**Problem**: Silent failures, non-POSIX shell usage.

**Suggestion**:
- Change shebang to `#!/bin/bash`
- Add `set -euo pipefail`
- Log each module being installed
- Report failures at the end instead of silently continuing

### E. Document the expected sync workflow

**Problem**: It's unclear how to use this on an existing machine. The README only mentions "new computer" usage. Running `bootstrap` on an existing machine would try to re-clone dotfiles (which fails silently since the dir exists) and re-sync packages.

**Suggestion**: Add a brief usage section:
```
# New machine:
./bootstrap/configure

# Existing machine (sync packages):
source bootstrap/configure && sync

# After editing Brewfile/Parufile:
source bootstrap/configure && sync
```

### F. Consider replacing git submodules with a simpler structure

**Problem**: 115 git submodules create significant maintenance overhead: slow clones, complex updates, easy to get into detached HEAD states.

**Impact**: On an infrequently-edited repo, stale submodule pointers are a common source of confusion.

**Suggestion**: This is a large change, but worth considering: collapse some or all submodules into the main repo. The per-tool repos could remain as read-only mirrors if needed for sharing.

### G. Pin mise tool versions or add an update command

**Problem**: `mise use -gy golang@latest` upgrades every run.

**Suggestion**: Either pin versions (`golang@1.22`) or separate "install" from "upgrade" — have `install.sh` only install if missing, and a separate `upgrade` command for intentional upgrades.

### H. Clarify the Go toolchain story

**Problem**: `brew "go"` installs Go via Homebrew. `mise use -gy golang@latest` installs Go via mise. `default-go-packages` installs Go tools via `go install`. `GOPATH` is set in `bashrc/customizations/go.bash`. Mise shims are on PATH in `mise.bash`.

**Impact**: It's genuinely unclear which Go binary runs when you type `go` — it depends on PATH ordering between `/opt/homebrew/bin`, `~/.local/share/mise/shims`, and the Homebrew Go cellar bin.

**Suggestion**: Pick one Go installation method (mise is the natural choice since it manages other runtimes too) and remove `brew "go"`.
