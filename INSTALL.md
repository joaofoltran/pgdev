# Installing pgdev

## Prerequisites

### Required

| Tool | Purpose | Install |
|------|---------|---------|
| **git** | Worktree management, PostgreSQL source | Pre-installed on macOS; `apt install git` on Linux |
| **make** | Build PostgreSQL | Xcode CLT on macOS (`xcode-select --install`); `apt install build-essential` on Linux |
| **ccache** | Caches compilation artifacts for fast rebuilds | `brew install ccache` / `apt install ccache` |
| **direnv** | Auto-loads per-worktree environment variables | `brew install direnv` / `apt install direnv` |

### Build Dependencies (macOS — Homebrew)

```bash
brew install openssl readline icu4c ccache direnv
```

### Build Dependencies (Linux — Debian/Ubuntu)

```bash
sudo apt install build-essential libssl-dev libreadline-dev libicu-dev ccache direnv
```

### Optional

| Tool | Purpose | Install |
|------|---------|---------|
| **bear** | Generates `compile_commands.json` for IDE support (clangd, VS Code) | `brew install bear` / `apt install bear` |

## Install pgdev

`pgdev` is a single bash script. Copy it anywhere on your `PATH`:

```bash
# Option 1: Copy to ~/.local/bin (recommended)
mkdir -p ~/.local/bin
cp pgdev ~/.local/bin/pgdev
chmod +x ~/.local/bin/pgdev
```

```bash
# Option 2: Copy to /usr/local/bin
sudo cp pgdev /usr/local/bin/pgdev
sudo chmod +x /usr/local/bin/pgdev
```

Make sure the target directory is in your `PATH`. For `~/.local/bin`, add this to your shell rc if it's not already there:

```bash
# ~/.zshrc or ~/.bashrc
export PATH="$HOME/.local/bin:$PATH"
```

## Post-Install Setup

### 1. Configure direnv hook

direnv must be hooked into your shell to auto-load `.envrc` files:

```bash
# For zsh (add to ~/.zshrc)
eval "$(direnv hook zsh)"

# For bash (add to ~/.bashrc)
eval "$(direnv hook bash)"
```

Restart your shell or `source` the rc file.

### 2. Configure ccache

ccache should skip hashing the working directory so that builds across worktrees share the cache:

```bash
mkdir -p ~/.config/ccache
echo "hash_dir = false" >> ~/.config/ccache/ccache.conf
```

### 3. Clone PostgreSQL

If you haven't already, clone the PostgreSQL source:

```bash
mkdir -p ~/public_repos
git clone https://github.com/postgres/postgres.git ~/public_repos/postgres
```

### 4. Verify

Run the built-in dependency checker:

```bash
pgdev setup
```

You should see checkmarks for all dependencies:

```
==> Checking dependencies...
  ✓ ccache (/opt/homebrew/bin/ccache)
  ✓ direnv (/opt/homebrew/bin/direnv)
  ✓ git (/usr/bin/git)
  ✓ make (/usr/bin/make)
  ✓ openssl
  ✓ readline
  ✓ icu4c
  ✓ ccache hash_dir=false
  ✓ direnv hook in shell rc
==> All good! Run 'pgdev build' to build the current worktree.
```

## First Build

```bash
cd ~/public_repos/postgres

# Create a worktree for the latest stable branch and build it
pgdev new REL_17_STABLE

# Move into it
cd ../pg-REL_17_STABLE

# Initialize a data directory and start the cluster
pgdev initdb
pgdev start

# Verify
psql -c "SELECT version();"
```

## Uninstall

```bash
rm ~/.local/bin/pgdev    # or wherever you installed it
```

pgdev does not write any state outside of the git worktrees it creates. To fully clean up, remove the `pg-*` worktree directories and prune them from git:

```bash
cd ~/public_repos/postgres
git worktree list           # see all worktrees
git worktree remove pg-REL_17_STABLE
```
