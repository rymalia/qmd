# Issue: `tsc` build fails when npm resolves zod >4.2.1

## Title

`tsc` fails with 14 type errors in `src/mcp/server.ts` when zod resolves to 4.3.x

## Labels

bug, build

## Body

### Description

Building from source with `npm install && npm run build` fails with 14 TypeScript errors in `src/mcp/server.ts`. All errors are `TS2322: Type 'ZodFoo' is not assignable to type 'AnySchema'` on the MCP tool input schemas.

The root cause is `package.json` declaring `"zod": "^4.2.1"` — the caret allows npm to resolve `zod@4.3.6`, which has breaking internal type changes that are incompatible with `@modelcontextprotocol/sdk@1.27.0`'s `AnySchema` type.

### Steps to Reproduce

```bash
git clone https://github.com/tobi/qmd.git
cd qmd
npm install
npm run build
```

### Expected

Clean build.

### Actual

```
src/mcp/server.ts:287:9 - error TS2322: Type 'ZodArray<ZodObject<{ type: ZodEnum<{ lex: "lex"; vec: "vec"; hyde: "hyde"; }>; query: ZodString; }, $strip>>' is not assignable to type 'AnySchema'.
  Type 'ZodArray<...>' is missing the following properties from type 'ZodType<any, any, any>': _type, _parse, _getType, _getOrReturnCtx, and 7 more.
```

14 errors total, all in `src/mcp/server.ts`, all the same `AnySchema` mismatch pattern.

### Fix

Pin zod to the exact version the MCP SDK expects:

```diff
-    "zod": "^4.2.1"
+    "zod": "4.2.1"
```

### Who This Affects

Anyone building from source without a published lockfile — forks, contributors, CI pipelines that run `npm install` fresh. Published npm installs are unaffected since they ship pre-compiled `dist/`.

### Environment

- Node v24.12.0
- npm 10.x
- macOS arm64
- `@modelcontextprotocol/sdk@1.27.0`
- `zod@4.3.6` (as resolved by npm from `^4.2.1`)
