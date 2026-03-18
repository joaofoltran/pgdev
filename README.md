# pgdev

A command-line tool for managing PostgreSQL multi-version development environments using git worktrees.

`pgdev` wraps the PostgreSQL build-from-source workflow into simple commands: create a worktree for any branch, configure and build with developer flags, initialize and manage clusters — all without leaving the terminal.

## Why pgdev?

Building PostgreSQL from source across multiple branches is tedious. You need to remember configure flags, manage separate data directories, juggle PATH and environment variables, and keep track of which version is running where.

`pgdev` handles all of this:

- **One command to build** — `pgdev new REL_17_STABLE` creates a worktree, configures with debug/assert flags, and builds with ccache
- **Per-worktree isolation** — each worktree gets its own `install/`, `data/`, and `.envrc` with correct PATH and PG environment variables
- **direnv integration** — `cd` into a worktree and your shell automatically picks up the right `pg_ctl`, `psql`, `PGDATA`, and `PGPORT`
- **Cluster lifecycle** — `pgdev initdb`, `pgdev start`, `pgdev stop` manage the cluster in the current worktree
- **Multi-version dashboard** — `pgdev ls` shows all worktrees with their version and running status

## Quick Start

```bash
# Check dependencies
pgdev setup

# Clone the PostgreSQL repo (one-time)
git clone https://github.com/postgres/postgres.git ~/public_repos/postgres
cd ~/public_repos/postgres

# Create a worktree for a release branch, build it
pgdev new REL_17_STABLE

# Move into it
cd ../pg-REL_17_STABLE

# Initialize a cluster and start it
pgdev initdb
pgdev start

# You now have a running PG 17 dev build
psql -c "SELECT version();"
```

## Commands

| Command | Description |
|---------|-------------|
| `pgdev new <branch> [name]` | Create a worktree for `<branch>`, configure, and build. Optional `name` overrides the directory name (`pg-<name>`) |
| `pgdev build` | Configure + build + install the current worktree |
| `pgdev configure` | Run `./configure` with dev flags (no build) |
| `pgdev initdb [--port PORT]` | Initialize `data/` for the current worktree. Optional `--port` sets a custom port in both `postgresql.conf` and `.envrc` |
| `pgdev deletedb` | Stop the cluster (if running) and remove `data/` |
| `pgdev start` | Start the cluster |
| `pgdev stop` | Stop the cluster |
| `pgdev restart` | Restart the cluster |
| `pgdev status` | Show postgres version and `pg_ctl` status |
| `pgdev test [target] [args]` | Run regression tests. Targets: `check` (default), `world`, `installcheck` |
| `pgdev ls` | List all `pg-*` worktrees with version and running/stopped status |
| `pgdev setup` | Check that all dependencies are installed |

## How It Works

### Worktree Layout

`pgdev new` creates worktrees as siblings of the main PostgreSQL repo:

```
~/public_repos/
├── postgres/              # main repo (git clone)
├── pg-REL_17_STABLE/      # pgdev new REL_17_STABLE
│   ├── install/           #   ./configure --prefix=$PWD/install → make install
│   ├── data/              #   pgdev initdb
│   ├── .envrc             #   direnv: PATH, PGDATA, PGPORT, PGHOST
│   └── (postgres source)
├── pg-REL_16_STABLE/      # pgdev new REL_16_STABLE
└── pg-my-patch/           # pgdev new my-feature-branch my-patch
```

### Build Configuration

Every build uses developer-friendly defaults:

| Flag | Purpose |
|------|---------|
| `--enable-cassert` | Enable assertion checks |
| `--enable-debug` | Include debug symbols |
| `--enable-depend` | Track header dependencies for incremental builds |
| `--with-openssl` | SSL support |
| `--with-readline` | readline support for `psql` |
| `--with-icu` | ICU collation support |
| `CFLAGS="-ggdb -Og -g3 -fno-omit-frame-pointer"` | Full debug info, minimal optimization |
| `CC="ccache cc"` | ccache for fast rebuilds |

If [bear](https://github.com/rizsotto/Bear) is installed, `pgdev build` automatically wraps the build to generate `compile_commands.json` for IDE integration.

### Per-Worktree Overrides

Create a `pgdev.conf` file in the worktree root to add extra configure flags:

```bash
# pgdev.conf
EXTRA_CONFIGURE_FLAGS=(--with-python --with-perl --with-llvm)
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

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PGDEV_REPOS_DIR` | Parent of current git repo | Directory where `pg-*` worktrees are created |

## Platform Support

| Platform | Status |
|----------|--------|
| macOS (Apple Silicon) | Fully supported — auto-detects Homebrew at `/opt/homebrew` |
| macOS (Intel) | Fully supported — uses `/usr/local` |
| Linux (Debian/Ubuntu) | Supported |

## License

MIT
