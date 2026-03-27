# pgdev

A command-line tool for managing PostgreSQL multi-version development environments using git worktrees.

`pgdev` wraps the PostgreSQL build-from-source workflow into simple commands: create a worktree for any branch, configure and build with developer flags, initialize and manage clusters — all without leaving the terminal.

## Why pgdev?

Building PostgreSQL from source across multiple branches is tedious. You need to remember configure flags, manage separate data directories, juggle PATH and environment variables, and keep track of which version is running where.

`pgdev` handles all of this:

- **Per-worktree isolation** — each worktree gets its own `install/`, `data/`, and `.envrc` with correct PATH and PG environment variables
- **direnv integration** — `cd` into a worktree and your shell automatically picks up the right `pg_ctl`, `psql`, `PGDATA`, and `PGPORT`
- **Cluster lifecycle** — `pgdev initdb`, `pgdev start`, `pgdev stop` manage the cluster in the current worktree
- **Multi-version dashboard** — `pgdev ls` shows all worktrees with their version and running status
- **Clean removal** — `pgdev rm` stops the cluster and removes the worktree properly (no stale git references)

## Quick Start

```bash
# Check dependencies
pgdev setup

# Clone the PostgreSQL repo (one-time)
git clone https://github.com/postgres/postgres.git ~/src/postgres
cd ~/src/postgres

# Create a worktree for a release branch
pgdev new REL_17_STABLE 17

# Move into it, build, init, start
cd ../pg-17
pgdev install
pgdev initdb
pgdev start

# You now have a running PG 17 dev build
psql -c "SELECT version();"
```

## Commands

| Command | Description |
|---------|-------------|
| `pgdev new <branch> [name]` | Create a worktree for `<branch>` and write `.envrc`. Optional `name` overrides the directory name (`pg-<name>`) |
| `pgdev install` | Configure + build + install the current worktree (full pipeline) |
| `pgdev configure` | Run `meson setup` with dev flags |
| `pgdev build` | Compile only (`ninja`) |
| `pgdev clean` | Remove `build/` and `install/` directories (stops the cluster first if running) |
| `pgdev initdb [--port PORT]` | Initialize `data/` for the current worktree. Optional `--port` sets a custom port in both `postgresql.conf` and `.envrc` |
| `pgdev deletedb` | Stop the cluster (if running) and remove `data/` |
| `pgdev start` | Start the cluster |
| `pgdev stop` | Stop the cluster |
| `pgdev restart` | Restart the cluster |
| `pgdev status` | Show postgres version and `pg_ctl` status |
| `pgdev test [target] [args]` | Run regression tests (`meson test`). Targets: `check` (default), `world`, `installcheck` |
| `pgdev rm <name>` | Stop the cluster and remove the worktree cleanly (e.g. `pgdev rm 17`) |
| `pgdev ls` | List all `pg-*` worktrees with version and running/stopped status |
| `pgdev setup` | Check that all dependencies are installed |

## How It Works

### Worktree Layout

`pgdev new` creates worktrees as siblings of the main PostgreSQL repo:

```
~/src/
├── postgres/              # main repo (git clone)
├── pg-17/                 # pgdev new REL_17_STABLE 17
│   ├── build/             #   pgdev configure → meson setup (out-of-tree)
│   ├── install/           #   pgdev install → ninja install
│   ├── data/              #   pgdev initdb
│   ├── .envrc             #   direnv: PATH, PGDATA, PGPORT, PGHOST
│   └── (postgres source)
├── pg-16/                 # pgdev new REL_16_STABLE 16
└── pg-my-patch/           # pgdev new my-feature-branch my-patch
```

### Build Configuration

pgdev uses [meson](https://mesonbuild.com/) + [ninja](https://ninja-build.org/) for building (requires PostgreSQL 16+). Every build uses developer-friendly defaults:

| Meson option | Purpose |
|------|---------|
| `-Dcassert=true` | Enable assertion checks |
| `--debug --optimization=g` | Debug symbols with `-Og` optimization |
| `-Dssl=openssl` | SSL support |
| `-Dreadline=enabled` | readline support for `psql` |
| `-Dicu=enabled` | ICU collation support |
| `-Dtap_tests=enabled` | Enable TAP regression tests |
| `-Dc_args=['-ggdb', '-g3', '-fno-omit-frame-pointer']` | Extra debug info, frame pointers for profiling |

Meson automatically uses [ccache](https://ccache.dev/) if it's installed — no extra configuration needed.

`compile_commands.json` is generated automatically during configure and symlinked to the project root for IDE integration (clangd, VS Code, Cursor).

### Per-Worktree Overrides

Create a `pgdev.conf` file in the worktree root to add extra meson flags:

```bash
# pgdev.conf
EXTRA_MESON_FLAGS=(-Dplpython=enabled -Dplperl=enabled -Dllvm=enabled)
```

### direnv Integration

Each worktree gets a `.envrc` that sets:

```bash
export PGDIR="$PWD/install"
export PGDATA="$PWD/data"
export PGHOST="/tmp"
export PGPORT="5432"
export PGUSER="<your-user>"
export PGDATABASE="postgres"
PATH_add "$PGDIR/bin"
```

When you `cd` into a worktree, direnv loads these automatically — `psql`, `pg_ctl`, and all PG tools resolve to the correct version.

### First-Run Setup

On first use, if `PGDEV_REPOS_DIR` is not set and you're not inside a git repo, pgdev will interactively ask for your repos directory and tell you exactly what to add to your shell rc file:

```
  pgdev needs to know where your PostgreSQL repo and worktrees live.
  This is the parent directory that contains (or will contain) the
  postgres/ clone and pg-* worktrees.

  Example: if your repo is at ~/src/postgres, enter ~/src

  Repos directory: ~/src

  Add this to /home/user/.bashrc so pgdev remembers next time:

    export PGDEV_REPOS_DIR="/home/user/src"
```

### Dependency Checks

Every command validates its dependencies before running. If something is missing, you get a clear message pointing you to `pgdev setup`:

```
pgdev: error: 'ninja' is not installed — run 'pgdev setup' to check all dependencies
```

`pgdev setup` checks everything at once and tells you exactly what to install and how.

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PGDEV_REPOS_DIR` | Auto-detected from current git repo | Parent directory where `pg-*` worktrees are created |

## Platform Support

| Platform | Status |
|----------|--------|
| macOS (Apple Silicon) | Fully supported — auto-detects Homebrew at `/opt/homebrew` |
| macOS (Intel) | Fully supported — uses `/usr/local` |
| Linux (Debian/Ubuntu) | Fully supported — `apt` packages |
| Linux (Fedora/RHEL) | Fully supported — `dnf`/`yum` packages |
| Linux (Arch) | Fully supported — `pacman` packages |
| Linux (Alpine) | Fully supported — `apk` packages |

## License

MIT
