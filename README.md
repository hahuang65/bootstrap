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

### Reconcile dotfile submodules

The dotfiles superproject records each submodule's supported operating systems with repeated `os = Darwin` and `os = Linux` entries in `.gitmodules`.
Run the policy directly to initialize supported submodules and safely remove unsupported checkouts:

```sh
~/.dotfiles/configure-submodules.sh
```

The command stops if an unsupported submodule contains local work or if a tracked submodule has no operating-system policy.

### Package management

- **macOS**: Edit `Brewfile`, then `sync`
- **Linux**: Edit `Parufile`, then `sync`
- **Language tools**: Edit files in `~/.dotfiles/mise/` (`default-go-packages`, `default-python-packages`, etc.)
