# Plan: Bootstrap Usability & Maintainability Improvements

## Goal

Improve the bootstrap/configure system so it's safer, clearer about what it does, and easier to pick back up after months of not touching it. Key changes: rewrite `sync_paru` with clearer step ordering (refresh repos → install new → remove stale → update existing), fix the brew sync chain, deduplicate packages between Brew and mise, and harden the top-level `install.sh`.

## Research Reference

`docs/claude/20260309-1319-bootstrap-improvements/research.md`

## Approach

Make focused, independent changes to existing files. No new abstractions or major restructuring. Each change is small enough to understand in isolation. Ordered from highest to lowest impact.

## Detailed Changes

### Change 1: Rewrite `sync_paru` — refresh, install, remove, update

**File:** `bootstrap/configure` (lines 51-128)

Replace the current `sync_paru` with a clearer four-step flow:

1. **Refresh repos** — `paru -Sy` updates the package database only (no upgrades yet)
2. **Install new** — `paru -S --needed` installs any packages from Parufile not yet on the system
3. **Remove stale** — diff installed-explicit against Parufile + Parufile.system + their dependencies, remove anything not accounted for (keeps the current removal behavior but with a confirmation prompt)
4. **Update existing** — `paru -Su` upgrades all outdated packages

```bash
function sync_paru {
  if [ ! -f "$PARU_FILE" ]; then
    echo "Parufile not found at $PARU_FILE"
    return 1
  fi

  # Read desired packages from Parufile
  local desired_packages=()
  while IFS= read -r line; do
    [[ -z "$line" || "$line" =~ ^# ]] && continue
    desired_packages+=("$line")

    # Expand package groups to their members
    if paru -Qg "$line" &>/dev/null; then
      while IFS= read -r group_line; do
        pkg=$(echo "$group_line" | awk '{print $2}')
        desired_packages+=("$pkg")
      done < <(paru -Qg "$line")
    fi
  done <"${PARU_FILE}"

  # Read system packages from Parufile.system if it exists
  local system_packages=()
  if [ -f "$PARU_SYSTEM_FILE" ]; then
    while IFS= read -r line; do
      [[ -z "$line" || "$line" =~ ^# ]] && continue
      system_packages+=("$line")
    done <"$PARU_SYSTEM_FILE"
  fi

  # Step 1: Refresh package databases (no upgrades yet)
  echo "Refreshing package databases..."
  paru -Sy

  # Step 2: Install new packages from Parufile
  if [ ${#desired_packages[@]} -gt 0 ]; then
    echo "Installing new packages..."
    paru -S --needed "${desired_packages[@]}"
  fi

  # Step 3: Remove packages not in Parufile or Parufile.system
  local all_desired=("${desired_packages[@]}" "${system_packages[@]}")
  local installed_explicit
  installed_explicit=$(paru -Qqe)

  local packages_to_remove=()
  while IFS= read -r pkg; do
    local found=false
    for desired in "${all_desired[@]}"; do
      if [ "$pkg" = "$desired" ]; then
        found=true
        break
      fi
    done
    if [ "$found" = false ]; then
      packages_to_remove+=("$pkg")
    fi
  done <<<"$installed_explicit"

  if [ ${#packages_to_remove[@]} -gt 0 ]; then
    echo "Removing packages not in Parufile or Parufile.system..."
    paru -Rns --noconfirm "${packages_to_remove[@]}" || echo "Some packages could not be removed (may be dependencies)"
  fi

  # Clean up orphaned dependencies
  local orphans
  orphans=$(paru -Qdtq 2>/dev/null) || true
  if [ -n "$orphans" ]; then
    echo "$orphans" | paru -Rns - || echo "No orphaned dependencies to remove"
  fi

  # Step 4: Update all outdated packages
  echo "Updating outdated packages..."
  paru -Su
}
```

Key changes from current code:
- **Step ordering**: repos refresh → install → remove → update (instead of remove → update → install)
- **Auto-removal** of stale packages (keeps `--noconfirm`)
- **Comment support**: skips blank lines and `#` comments in Parufile/Parufile.system
- Still expands package groups, still reads Parufile.system, still cleans orphaned deps

### Change 2: Fix brew sync chain

**File:** `bootstrap/configure` (line 136)

Current:
```bash
brew bundle check || brew bundle install && brew bundle cleanup --force && brew bundle dump --force
```

Replace with:
```bash
brew bundle install
brew bundle cleanup --force
brew bundle dump --force
```

`brew bundle install` is idempotent — it installs missing packages and skips already-installed ones. No need for `check` as a guard. The `dump --force` at the end regenerates the Brewfile in sorted, canonical format so it's always a clean snapshot of installed state — idempotent and atomic between git commits.

### Change 3: Deduplicate Brew/Mise packages

**File:** `bootstrap/Brewfile`

Remove packages that are already managed by mise (`mise use -gy` in `mise/install.sh` or `default-*-packages` files):

**Brewfile** — remove these:

| Remove from Brewfile | Managed by |
|---------------------|------------|
| `brew "go"` (line 47) | `mise use -gy golang@latest` |
| `brew "gum"` (line 49) | `mise use -gy gum@latest` |
| `brew "hut"` (line 53) | `mise/default-go-packages` |
| `brew "pre-commit"` (line 74) | `mise use -gy pre-commit@latest` |

**Parufile** — remove this:

| Remove from Parufile | Managed by |
|---------------------|------------|
| `uv` (line 60) | `mise use -gy uv@latest` |

**Parufile.system** — no removals. System `python` (line 153) stays because other Arch packages depend on it as a system dependency; it's separate from mise's development Python.


### Change 4: Harden top-level `install.sh`

**File:** `install.sh`

Current:
```bash
#!/bin/sh

scripts_dir="$HOME/.scripts"

if [ -d "$scripts_dir" ]; then
  rm -rf "$scripts_dir"
fi

for dir in */; do
  pushd $dir > /dev/null
  if test -e "install.sh"; then
    ./install.sh
  fi
  popd > /dev/null
done
```

Replace with:
```bash
#!/usr/bin/env bash

set -euo pipefail

scripts_dir="$HOME/.scripts"

if [ -d "$scripts_dir" ]; then
  rm -rf "$scripts_dir"
fi

failed=()

for dir in */; do
  if [ -e "$dir/install.sh" ]; then
    echo "Installing $dir..."
    if ! (cd "$dir" && ./install.sh); then
      echo "  FAILED: $dir"
      failed+=("$dir")
    fi
  fi
done

if [ ${#failed[@]} -gt 0 ]; then
  echo ""
  echo "Failed modules:"
  printf "  - %s\n" "${failed[@]}"
  exit 1
fi
```

Changes: bash shebang, `set -euo pipefail`, `cd` instead of `pushd/popd`, logs each module, collects and reports failures at the end.

### Change 5: Update README with usage instructions

**File:** `bootstrap/README.md`

Replace with:
```markdown
# bootstrap

NOTE: This is a primarily private repo (read: unlikely I will answer any support requests) that is kept public so I can easily access it on new computers.

This is a script that is meant to be run on a new computer. It will attempt to install a package manager, then install a pre-defined list of packages, and then finally clone my personal dotfiles and run the installation for it.

## Usage

### New machine

```sh
./bootstrap/configure
```

### Sync packages on existing machine

```sh
source ~/.dotfiles/bootstrap/configure && sync
```

### Package management

- **macOS**: Edit `Brewfile`, then `sync`
- **Linux**: Edit `Parufile`, then `sync`
- **Language tools**: Edit files in `~/.dotfiles/mise/` (`default-go-packages`, `default-python-packages`, etc.)
```

## New Files

None.

## Dependencies

None — all changes use existing tools.

## Considerations & Trade-offs

**`sync_paru` step reordering:** The key improvement is the step ordering (refresh → install → remove → update instead of remove → update → install) and comment/blank-line support in Parufile parsing. Removal stays automatic with `--noconfirm` to keep the Parufile as the enforced source of truth, matching how `brew bundle cleanup --force` works on macOS.

**Keeping `brew bundle dump --force`:** The dump ensures the Brewfile is always sorted and canonical — a clean snapshot of installed state. Combined with cleanup (which removes anything not in the Brewfile), each sync produces an idempotent, atomic Brewfile suitable for committing. Manual installs are temporary; if you want something permanent, add it via `pkg add`.

**Not pinning mise versions:** `mise use -gy golang@latest` is fine as-is. Per-project `.mise.toml` files express specific version requirements for each project. The `@latest` global gives a good default for system-wide tools like gum, pre-commit, etc. that should be available across all projects regardless of language.

## Migration / Data Changes

After implementing, on macOS run:
```sh
brew remove go gum hut pre-commit  # Remove duplicates that mise now owns
```

On Linux, `uv` will be removed by `sync_paru` on next sync (since it's no longer in the Parufile).

No other data migration needed. Changes are backward-compatible.

## Testing Strategy

Manual testing only — this is a shell script bootstrap system with no test framework. Verification:

1. **macOS sync**: Source configure, run `sync`, verify Brewfile is regenerated in sorted canonical format
2. **Linux sync** (if available): Source configure, run `sync_paru`, verify it: refreshes repos, installs new packages, shows removal prompt (not auto-removing), then updates outdated packages
3. **install.sh**: Run top-level `install.sh`, verify it logs each module name and reports any failures
4. **Intentional failure**: Temporarily break one module's install.sh, run top-level `install.sh`, verify it continues and reports the failure

## Todo List

### Phase 1: sync_paru rewrite
- [x] Replace `sync_paru` function in `bootstrap/configure` with four-step flow (refresh → install → remove → update)
- [x] Add blank line and comment skipping to Parufile/Parufile.system reading

### Phase 2: Brew sync chain
- [x] Replace brew sync line in `sync` function with `brew bundle install && brew bundle cleanup --force && brew bundle dump --force`

### Phase 3: Deduplicate packages
- [x] Remove `brew "go"` from Brewfile
- [x] Remove `brew "gum"` from Brewfile
- [x] Remove `brew "hut"` from Brewfile
- [x] Remove `brew "pre-commit"` from Brewfile
- [x] Remove `uv` from Parufile

### Phase 4: Harden install.sh
- [x] Rewrite top-level `install.sh` with bash shebang, `set -euo pipefail`, logging, and failure reporting

### Phase 5: Documentation
- [x] Update `bootstrap/README.md` with usage instructions for new machine, sync, and package management
