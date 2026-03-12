# Issue: `bin/qmd` wrapper picks wrong runtime for source builds with npm

## Title

`bin/qmd` wrapper runs bun for source builds that used npm, causing ABI mismatches

## Labels

bug

## Body

### Description

The `bin/qmd` wrapper detects the runtime by checking for `bun.lock` or `bun.lockb` in the package directory. Since `bun.lock` is checked into the repo, anyone who clones and builds from source — even using `npm install` — gets routed to bun. This causes native module ABI mismatches because npm compiled `better-sqlite3` and `sqlite-vec` for Node, but the wrapper runs them under Bun.

The published npm package doesn't have this problem because `bun.lock` isn't in the `"files"` whitelist, so registry installs correctly resolve to node. This only affects source builds: forks, contributors, and anyone running from a git checkout.

### Steps to Reproduce

```bash
git clone https://github.com/tobi/qmd.git
cd qmd
npm install
npm run build
npm link
qmd status   # or any command that touches sqlite-vec
```

### Expected

The wrapper should detect that npm was used (e.g. `package-lock.json` exists) and run under node.

### Actual

The wrapper finds the checked-in `bun.lock`, runs under bun, and native modules compiled for Node fail with ABI mismatches (e.g. `SQLiteError: no such module: vec0` for sqlite-vec).

### Current workaround

Delete `bun.lock` before running `npm install`:

```bash
rm bun.lock
npm install
npm run build
npm link
```

This works because the wrapper falls through to the `else` branch and execs node. But it's not documented and counter-intuitive — deleting a checked-in file to make the build work.

### Possible fixes

**Option A**: Check for `package-lock.json` first — if it exists, prefer node regardless of `bun.lock`:

```sh
if [ -f "$DIR/package-lock.json" ]; then
  exec node "$DIR/dist/cli/qmd.js" "$@"
elif [ -f "$DIR/bun.lock" ] || [ -f "$DIR/bun.lockb" ]; then
  exec bun "$DIR/dist/cli/qmd.js" "$@"
else
  exec node "$DIR/dist/cli/qmd.js" "$@"
fi
```

**Option B**: Add `bun.lock` to `.gitignore` so it's only present when the developer actively uses bun. The maintainer's local workflow is unaffected (bun install regenerates it), but source checkouts start with a clean slate.

**Option C**: Document the workaround in the README's "Building from source" section.

### Related

- #380 — `qmd cleanup` crashes when sqlite-vec is unavailable (the downstream consequence of this issue when running under Bun)
- The `bin/qmd` comment on line 22 describes the ABI mismatch scenario but only in the npm-install direction; the source-checkout direction isn't addressed

### Environment

- Node v24.12.0
- npm 10.x
- macOS arm64 (Apple M3)
- QMD v2.0.1 (source build)
