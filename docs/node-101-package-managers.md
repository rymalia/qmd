# Node 101: Runtimes, Version Managers & Package Managers

A reference for understanding how Node.js, nvm, npm, pnpm, and Bun relate to each other — and how to keep global CLI tools stable.

## The Cast of Characters

| Tool | What it is | Analogy |
|------|-----------|---------|
| **Node.js** | JavaScript runtime — executes JS outside the browser | Like Python the interpreter |
| **npm** | Node's built-in package manager — installs packages | Like pip |
| **npx** | npm's companion — runs packages without installing them globally | Like `pipx run` or `python -m` |
| **nvm** | Node *version* manager — switches between Node versions | Like pyenv |
| **pnpm** | Alternative package manager (content-addressable storage, strict) | Like npm but with deduplication |
| **Bun** | Independent runtime + package manager + bundler in one | Like if Python and pip were a single fast binary |

## How They Relate

```
nvm (version manager)
 └── manages multiple Node installations:
      ├── Node v24.12.0 → has its own npm → has its own global packages
      ├── Node v26.0.0  → has its own npm → has its own global packages (separate!)
      └── ...

Homebrew
 └── single Node installation:
      └── Node v25.6.1 → has its own npm → has its own global packages
           path: /opt/homebrew/bin/node (stable, never changes)

Bun (standalone)
 └── ~/.bun/bin/bun (runtime + package manager)
      └── global packages at ~/.bun/bin/ (stable path)
```

## What is npx?

`npx` ships with npm and solves a common annoyance: running a CLI tool without permanently installing it.

```sh
# Without npx: install globally, then run
npm install -g create-react-app
create-react-app my-app

# With npx: just run it (downloads temporarily, discards after)
npx create-react-app my-app
```

It's also useful for running a specific version of a tool, or running a project's local copy of a binary (from `node_modules/.bin/`) without typing the full path:

```sh
# Runs the project-local vitest, not a global one
npx vitest run

# Runs a specific version temporarily
npx node-gyp@10 rebuild
```

**Important caveat:** `npx` resolves `node` through your shell's PATH. If nvm is active, `npx` uses nvm's Node — even if you're trying to build something for a different Node installation. This can cause [native module ABI mismatches](#native-addons-and-abi-compatibility).

## The nvm Fragility Problem

nvm puts everything under versioned directories:

```
~/.nvm/versions/node/v24.12.0/bin/node
~/.nvm/versions/node/v24.12.0/bin/npm
~/.nvm/versions/node/v24.12.0/bin/qmd      ← globally installed CLI
~/.nvm/versions/node/v24.12.0/lib/node_modules/@tobilu/qmd/
```

When you `nvm install v26`:
- A **new** directory is created: `~/.nvm/versions/node/v26.0.0/`
- It has its own `npm`, its own `lib/node_modules/` — empty
- **All your global packages are gone** from the new version's perspective
- Any LaunchAgents, scripts, or cron jobs pointing at the old path break

### Migration helper

nvm can copy globals from one version to another:

```sh
nvm install v26 --reinstall-packages-from=v24.12.0
```

But this is easy to forget, and symlinks from `npm link` still need manual re-linking.

## Recommended Setup

**Use nvm for project-level version switching only.** Use a stable location for global CLI tools.

### Homebrew node for global CLIs (recommended)

```sh
# Install globals here — path never changes across upgrades
/opt/homebrew/bin/npm install -g @tobilu/qmd

# Link local dev versions of projects
cd ~/projects/bird && /opt/homebrew/bin/npm link
cd ~/projects/xc && /opt/homebrew/bin/npm link

# Stable paths for LaunchAgents, scripts, etc.
# /opt/homebrew/bin/qmd
# /opt/homebrew/bin/bird
# /opt/homebrew/bin/xc
```

`brew upgrade node` updates Node in-place — the path stays `/opt/homebrew/bin/node`. No path migration needed.

**Caveat:** If the upgrade crosses a major version (e.g. v24 → v25), packages with native addons (like `better-sqlite3`, `sqlite-vec`) need rebuilding — see [Native Addons](#native-addons-and-abi-compatibility) below.

### Why not Bun for globals?

Bun works (`bun install -g`, `bun link`) and has a stable path (`~/.bun/bin/`), but:
- Bun moves fast and has occasional npm compatibility edge cases
- Some packages with native addons (like node-llama-cpp, sqlite-vec) may not work under Bun's install
- For running code locally (`bun src/qmd.ts`), Bun is great. For installing published npm packages globally, Homebrew's npm is more reliable.

## Global Package Locations

| Manager | Global bin path | Stable? |
|---------|----------------|---------|
| nvm npm | `~/.nvm/versions/node/v{X}/bin/` | No — changes per version |
| Homebrew npm | `/opt/homebrew/bin/` | Yes — survives upgrades |
| Bun | `~/.bun/bin/` | Yes — version-independent |
| pnpm | `~/.local/share/pnpm/` | Yes, but pnpm is often installed via nvm *(fragile)*|

## npm link (dev/pre-release workflow)

`npm link` creates a global symlink so you can run the development version of a local project:

```sh
cd ~/projects/bird     # your local clone
npm link               # creates global symlink: bird → ~/projects/bird

# Now "bird" command runs your local source
bird --version         # runs from ~/projects/bird, not the published version
```

The symlink lives in whichever npm's global bin you used:
- `nvm npm link` → symlink in `~/.nvm/versions/node/v24/bin/bird` (fragile)
- `/opt/homebrew/bin/npm link` → symlink in `/opt/homebrew/bin/bird` (stable)

To unlink and go back to the published version:

```sh
cd ~/projects/bird
npm unlink
npm install -g @steipete/bird   # reinstall published version
```

## Package Managers Are Parallel Dimensions

This is the single most confusing thing about the Node ecosystem: **npm, bun, and pnpm are completely independent installation registries.** They share the same package format (npm registry) but maintain separate databases of what's installed.

```
bun install -g @tobilu/qmd   →  ~/.bun/bin/qmd           (bun's registry)
npm install -g @tobilu/qmd   →  ~/.nvm/.../bin/qmd        (npm's registry)
pnpm install -g @tobilu/qmd  →  ~/.local/share/pnpm/qmd   (pnpm's registry)
```

There is **no shared ledger.** `npm list -g` has no idea what `bun` installed. `bun` doesn't tell `npm` anything. This means:

- You can install with `bun link`, check with `npm list -g`, see nothing, and conclude "it's not installed"
- You can then `npm install -g` the same package — now you have **two copies**
- Which one runs depends on which directory appears first in your PATH
- `which qmd` only shows the first match — use `which -a qmd` to see all copies

### The wrong-package-manager trap

If a project says "use Bun" (like QMD's CLAUDE.md does), you **must** use Bun for local development:

```sh
# WRONG — will fail with cryptic errors
git clone https://github.com/tobi/qmd
cd qmd
npm install    # ← wrong package manager for a Bun project
npm link       # ← installs to npm's registry, not bun's

# RIGHT
git clone https://github.com/tobi/qmd
cd qmd
bun install    # ← matches the project's lockfile (bun.lock)
bun link       # ← installs to ~/.bun/bin/
```

Using the wrong package manager won't give you a clear "wrong tool" error — it'll give you dependency resolution failures, native module build errors, or subtle runtime bugs that look like the project's code is broken. It's not. You're just in the wrong dimension.

### Diagnostic commands

```sh
# See ALL copies of a binary on your PATH
which -a qmd

# Check what each package manager knows about
npm list -g --depth=0              # npm's globals
ls ~/.bun/bin/                     # bun's globals
pnpm list -g --depth=0            # pnpm's globals

# Check which Node a binary will actually use
head -1 $(which qmd)              # shows the shebang line
```

## Native Addons and ABI Compatibility

Some npm packages include **native addons** — compiled C/C++ code that talks directly to Node's internals. Examples: `better-sqlite3`, `sqlite-vec`, `sharp`, `bcrypt`.

These are compiled against a specific **NODE_MODULE_VERSION** (ABI version). Each major Node release gets a new one:

| Node version | NODE_MODULE_VERSION |
|-------------|-------------------|
| Node 22 LTS | 127 |
| Node 24 LTS | 137 |
| Node 25 (current) | 141 |

**The golden rule:** the Node that *builds* the addon must be the same Node that *runs* it. If they don't match, you get:

```
Error: The module was compiled against a different Node.js version using
NODE_MODULE_VERSION 137. This version of Node.js requires NODE_MODULE_VERSION 141.
```

### Why this bites you with multiple Node installations

If you have both nvm (Node 24) and Homebrew (Node 25), running `npm rebuild` from an interactive shell will use nvm's Node for compilation (because nvm injects itself at the front of PATH in `.zshrc`). The resulting binary then fails when Homebrew's Node tries to load it.

Fix: explicitly control which Node runs the build:

```sh
# Override PATH so the right node is used for compilation
PATH="/opt/homebrew/bin:$PATH" npm rebuild better-sqlite3

# Or point node-gyp at specific headers
npm_config_nodedir=/opt/homebrew/Cellar/node/25.6.1_1 npx node-gyp rebuild
```

### After a major Node upgrade

```sh
# Rebuild all native addons for the new Node version
npm rebuild

# Or reinstall the affected package
npm install -g @tobilu/qmd --force
```

## LaunchAgents & Background Services

macOS LaunchAgents (`~/Library/LaunchAgents/`) need absolute paths. Always use stable paths:

```xml
<!-- Good: Homebrew path, survives node upgrades -->
<string>/opt/homebrew/bin/qmd</string>

<!-- Bad: nvm path, breaks on node upgrade -->
<string>/Users/you/.nvm/versions/node/v24.12.0/bin/qmd</string>
```

**Important:** LaunchAgents don't source `~/.zshrc`, so they don't inherit nvm. The `EnvironmentVariables` dict in the plist must explicitly set PATH to include the Node binary directory. If using Homebrew node alongside nvm, make sure the plist PATH puts the intended Node first — otherwise the service may work in your terminal but fail at boot.

### Use `bootstrap`/`bootout`, not `load`/`unload`

Apple deprecated `launchctl load`/`unload`. The modern API:

```sh
# Start a user LaunchAgent
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.example.plist

# Stop it
launchctl bootout gui/$(id -u)/com.example

# Check if it's running
launchctl list | grep com.example
```

The old `load/unload` API can leave **ghost state** when a service crash-loops — launchd thinks the label is still registered but the process is dead, so `load` silently does nothing and `unload` fails with I/O errors. `bootout` clears this state properly. Always use the modern API.

## Case Study: QMD MCP Server Migration (Feb 2025)

Real-world lessons from migrating the QMD MCP HTTP daemon from nvm to Homebrew.

### What happened

1. QMD's LaunchAgent originally pointed at `~/.nvm/versions/node/v24.12.0/bin/qmd` — **this worked fine**
2. To avoid nvm's version-fragile paths, we migrated global packages to Homebrew (`/opt/homebrew/bin/`)
3. Homebrew had Node **v25.6.1** (NODE_MODULE_VERSION 141) while the published `@tobilu/qmd` ships prebuilt native binaries for Node **v24** (NODE_MODULE_VERSION 137)
4. `npm rebuild` and `node-gyp rebuild` both compiled against Node 24 headers — because nvm was first in the interactive shell's PATH, poisoning every build tool
5. Even tests that appeared to pass (`node -e "require('better-sqlite3')"`) were false positives — the interactive shell's `node` was nvm's v24, which matched the v24 binary. The LaunchAgent ran Homebrew's v25, which didn't match.

### Lessons

- **Test with the same Node that will run in production.** Use the full path: `/opt/homebrew/bin/node -e "require('better-sqlite3')"`, not just `node`.
- **nvm pollutes PATH for all build tools.** `npm rebuild`, `npx node-gyp`, and `prebuild-install` all inherit the shell's PATH. Override it explicitly when targeting a different Node: `PATH="/opt/homebrew/bin:$PATH" npm rebuild`.
- **Homebrew `node@24` is keg-only.** `brew install node@24` puts binaries in `/opt/homebrew/opt/node@24/bin/`, *not* `/opt/homebrew/bin/`. Global packages installed via its npm may land in unexpected prefix directories.
- **Don't fight the ABI — match it.** Instead of forcing compilation against different headers, it's often simpler to use the same Node version the prebuilt binaries target (Node 24 LTS).
- **The simplest solution was the starting point.** The nvm path in the LaunchAgent worked from the start. The migration to Homebrew introduced complexity that didn't exist before. Sometimes "fragile but working" beats "elegant but broken."
- **Use `launchctl bootstrap/bootout`, not `load/unload`.** The deprecated API left ghost state after the crash loop, making the service appear to silently refuse to load. `bootout` cleared the stale label and `bootstrap` loaded it cleanly.
