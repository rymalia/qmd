# Issue: `qmd cleanup` crashes with "no such module: vec0" when sqlite-vec is unavailable

## Title

`qmd cleanup` crashes on `cleanupOrphanedVectors()` when sqlite-vec fails to load

## Labels

bug

## Body

### Description

`qmd cleanup` crashes with `SQLiteError: no such module: vec0` when the sqlite-vec extension isn't available. This happens because `cleanupOrphanedVectors()` in `src/store.ts` executes a `DELETE FROM vectors_vec` statement unconditionally, without checking whether the vec0 module was loaded.

The rest of the codebase handles this gracefully — `src/store.ts:635` treats sqlite-vec as optional ("sqlite-vec is optional — vector search won't work but FTS is fine"), and vector search functions guard against the missing module. But the cleanup path doesn't have this guard.

### Steps to Reproduce

Run `qmd cleanup` in an environment where sqlite-vec's `loadExtension()` fails. The most common case is Bun, where `bun:sqlite` doesn't support dynamic extension loading.

### Expected

`qmd cleanup` should skip the orphaned vectors step (or log a warning) when sqlite-vec is unavailable, the same way vector search degrades gracefully.

### Actual

```
✓ Cleared 0 cached API responses
SQLiteError: no such module: vec0
      at run (bun:sqlite:322:21)
      at cleanupOrphanedVectors (/Users/rymalia/projects/qmd/dist/store.js:1217:8)
      at /Users/rymalia/projects/qmd/dist/cli/qmd.js:2710:34
```

The first cleanup step (clearing cached API responses) succeeds, but the process crashes before vacuuming or cleaning up inactive documents.

### Suggested Fix

Guard `cleanupOrphanedVectors()` the same way vector search operations are guarded — check if the `vectors_vec` table exists (or if sqlite-vec loaded successfully) before executing:

```typescript
function cleanupOrphanedVectors(db: Database): number {
  // Check if sqlite-vec is available before touching vectors_vec
  try {
    db.prepare("SELECT count(*) FROM vectors_vec LIMIT 0").get();
  } catch {
    return 0; // sqlite-vec not loaded, nothing to clean
  }
  // ... existing cleanup logic
}
```

Or wrap the existing call in the CLI's cleanup command with a try/catch that logs a warning.

### Environment

- Bun v1.3.10 (macOS arm64) — `bun:sqlite` does not support `loadExtension()`
- Also affects any environment where sqlite-vec fails to load (missing `.dylib`, Homebrew path issues, etc.)
- QMD v2.0.1
