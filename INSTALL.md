# Installing pgdev

## Prerequisites

### Required

| Tool | Purpose | Install |
|------|---------|---------|
| **git** | Worktree management, PostgreSQL source | Pre-installed on macOS; `apt install git` / `dnf install git` |
| **make** | Build PostgreSQL | Xcode CLT on macOS (`xcode-select --install`); `apt install build-essential` / `dnf groupinstall "Development Tools"` |
| **direnv** | Auto-loads per-worktree environment variables | `brew install direnv` / `apt install direnv` / `dnf install direnv` |

### Optional

| Tool | Purpose | Install |
|------|---------|---------|
| **ccache** | Caches compilation artifacts for fast rebuilds (pgdev falls back to plain `cc` without it) | `brew install ccache` / `apt install ccache` / `dnf install ccache` |
| **bear** | Generates `compile_commands.json` for IDE support (clangd, VS Code, Cursor) | `brew install bear` / `apt install bear` |

### Build Dependencies (macOS — Homebrew)

```bash
brew install openssl readline icu4c direnv ccache
```

### Build Dependencies (Linux — Debian/Ubuntu)

```bash
sudo apt install build-essential libssl-dev libreadline-dev libicu-dev direnv ccache
```

### Build Dependencies (Linux — Fedora/RHEL)

```bash
sudo dnf groupinstall "Development Tools"
sudo dnf install openssl-devel readline-devel libicu-devel direnv ccache
```

### Build Dependencies (Linux — Arch)

```bash
sudo pacman -S base-devel openssl readline icu direnv ccache
```

## Install pgdev

`pgdev` is a single bash script. Copy or symlink it anywhere on your `PATH`:

```bash
# Option 1: Symlink from a clone (recommended — stays up to date with git pull)
git clone https://github.com/jfoltran/pgdev.git ~/path/to/pgdev
ln -s ~/path/to/pgdev/pgdev ~/.local/bin/pgdev
```

```bash
# Option 2: Copy to ~/.local/bin
mkdir -p ~/.local/bin
cp pgdev ~/.local/bin/pgdev
chmod +x ~/.local/bin/pgdev
```

```bash
# Option 3: Copy to /usr/local/bin
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

# For fish (run once)
direnv hook fish | source
```

Restart your shell or `source` the rc file.

### 2. Configure ccache (optional but recommended)

ccache should skip hashing the working directory so that builds across worktrees share the cache:

```bash
mkdir -p ~/.config/ccache
echo "hash_dir = false" >> ~/.config/ccache/ccache.conf
```

### 3. Clone PostgreSQL

If you haven't already, clone the PostgreSQL source:

```bash
git clone https://github.com/postgres/postgres.git ~/src/postgres
```

You can put it wherever you want — pgdev auto-detects the parent directory when you run commands from inside the repo. Or set `PGDEV_REPOS_DIR` explicitly:

```bash
# ~/.zshrc or ~/.bashrc
export PGDEV_REPOS_DIR="$HOME/src"
```

On first use, if pgdev can't detect the repos directory, it will prompt you interactively and tell you exactly what to add to your rc file.

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
cd ~/src/postgres

# Create a worktree for a release branch
pgdev new REL_17_STABLE 17

# Move into it
cd ../pg-17

# Build, initialize, start
pgdev build
pgdev initdb
pgdev start

# Verify
psql -c "SELECT version();"
```

## Uninstall

```bash
rm ~/.local/bin/pgdev    # or wherever you installed it
```

pgdev does not write any state outside of the git worktrees it creates. To fully clean up, remove worktrees with pgdev itself:

```bash
pgdev rm 17
pgdev rm 16
```

Or manually:

```bash
cd ~/src/postgres
git worktree list
git worktree remove pg-17
```
