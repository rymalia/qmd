# Issue: `validateSemanticQuery` rejects hyphenated/kebab-case words as negation syntax

## Title

vec/hyde queries reject hyphenated words (kebab-case, compound adjectives) as negation syntax

## Labels

bug, search

## Body

### Description

The `validateSemanticQuery` function in `src/store.ts` rejects any vec or hyde query containing a hyphenated word — including kebab-case identifiers (`node-llama-cpp`, `better-sqlite3`, `sqlite-vec`), compound adjectives (`real-time`, `long-lived`, `multi-client`), and multi-hyphen phrases (`state-of-the-art`, `end-to-end`, `copy-on-write`).

The validator is intended to catch lex-only negation syntax (`-term`, `-"phrase"`) in semantic queries, but the regex `/-\w/` matches any hyphen followed by a word character **anywhere** in the string — not just at token boundaries where negation actually occurs.

### Steps to Reproduce

```bash
# All of these fail with: "Negation (-term) is not supported in vec/hyde queries"
qmd query 'vec: how does the rate-limiter handle burst traffic'
qmd query 'vec: multi-client session architecture'
qmd query 'hyde: HTTP transport runs a single long-lived daemon shared across all clients'
qmd query 'vec: better-sqlite3 native module loading'
```

Via MCP:
```json
{
  "searches": [
    { "type": "vec", "query": "how does the rate-limiter handle burst traffic" }
  ]
}
```
Returns: `Structured search (vec): Negation (-term) is not supported in vec/hyde queries. Use lex for exclusions.`

### Expected

Queries with hyphenated words should be accepted. Only queries with negation syntax — a `-` at the start of the query or after whitespace — should be rejected.

### Actual

Any query containing a hyphen followed by a letter is rejected, regardless of position in the string.

### Root Cause

**`src/store.ts:2601`:**
```typescript
if (/-\w/.test(query) || /-"/.test(query)) {
```

This is a character-level check, but negation is a token-level concept:
- `-redis` after whitespace → negation (exclude "redis")
- `long-lived` mid-word → hyphenated compound (not negation)

The regex cannot distinguish between these because it doesn't check what precedes the hyphen.

### Scope of Impact

Hyphenated words are extremely common in technical queries — exactly the kind of content QMD indexes:

**Kebab-case identifiers** (package names, CLI flags, CSS properties):
`node-llama-cpp`, `better-sqlite3`, `sqlite-vec`, `font-size`, `--no-verify`

**Compound adjectives**:
`real-time`, `long-lived`, `self-hosted`, `multi-client`, `cross-platform`, `non-blocking`, `single-threaded`

**Prepositional compounds**:
`in-memory`, `on-device`, `pre-built`, `write-ahead`, `out-of-band`

**Multi-hyphen phrases**:
`state-of-the-art`, `end-to-end`, `man-in-the-middle`, `copy-on-write`

**Short hyphenated terms**:
`e-commerce`, `A-B`, `re-index`, `co-located`

Any vec or hyde query containing these patterns is currently unusable.

### Proposed Fix

Anchor the negation check to token boundaries — start of string or after whitespace:

```diff
- if (/-\w/.test(query) || /-"/.test(query)) {
+ if (/(^|\s)-[\w"]/.test(query)) {
```

This also consolidates the two separate regex checks (`-\w` and `-"`) into a single pattern.

**Still correctly rejected** (negation syntax):
- `performance -sports` — negation after space
- `-redis connection pooling` — negation at start of query
- `-"exact phrase"` — negated quoted phrase
- `error handling -java -python` — multiple negations
- `  -term` — negation after leading whitespace
- `foo\t-bar` — negation after tab

**Now correctly accepted** (hyphenated words):
- `long-lived server shared across clients`
- `real-time voice processing pipeline`
- `state-of-the-art embedding models`
- `better-sqlite3 native module loading`
- `built-in vs add-on features`

### Environment

- QMD v2.0.1
- Affects both CLI (`qmd query`) and MCP server (`query` tool)
