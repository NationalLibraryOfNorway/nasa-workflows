# TypeScript Reference

Rules for TypeScript / JavaScript work. Apply on top of [`../AGENTS.md`](../AGENTS.md).

**Default runtime + tooling: [Bun](https://bun.com/docs/runtime/typescript).** Use Node-based tools only when an existing project requires them.

## Style

- **TypeScript strict mode** (`"strict": true` in `tsconfig.json`). No implicit `any`.
- Prefer `type` for unions / aliases, `interface` for object shapes that may be extended.
- No `any`. Use `unknown` and narrow. `// @ts-ignore` / `// @ts-expect-error` only with comment explaining why.
- Prefer `const` over `let`. No `var`.
- Use optional chaining (`?.`) and nullish coalescing (`??`).
- ES modules (`import`/`export`). No CommonJS unless project requires it.

## Bun

- Bun runs `.ts` / `.tsx` directly — **no separate transpile or build step needed** for execution.
- Bun does **not** type-check at runtime. Run `bun run tsc --noEmit` (or `bunx tsc --noEmit`) in CI and pre-commit.
- Type definitions for Bun globals (`Bun`, `Bun.serve`, `Bun.file`, ...): install `@types/bun` and add `"types": ["bun"]` to `tsconfig.json`.
- Recommended `tsconfig.json` for Bun-first projects: `"module": "ESNext"`, `"moduleResolution": "bundler"`, `"target": "ESNext"`, `"jsx": "react-jsx"` (if using JSX), `"allowImportingTsExtensions": true`, `"noEmit": true`.
- Lockfile: `bun.lock` (text) — commit it. Older `bun.lockb` (binary) is being phased out.
- `bunx <pkg>` is the equivalent of `npx <pkg>`.

## Tooling

- **Package mgmt**: `bun install`. Fall back to `pnpm` → `npm` → `yarn` only if project lockfile demands it.
- **Lint**: `eslint` (often `@typescript-eslint`). Run with `bun run eslint .` or via package script.
- **Format**: `prettier` or `biome` (Biome pairs well with Bun — fast, single binary).
- **Test**: `bun test` (built-in, Jest-compatible API). Use `vitest` / `jest` only when porting an existing suite or when a feature isn't supported.
- **Build**: `bun build` for bundling. `tsc` only for emitting `.d.ts` or type-checking.

Detect existing tooling by lockfile (`bun.lock`, `pnpm-lock.yaml`, `package-lock.json`, `yarn.lock`) and `package.json` scripts before assuming.

## Logging

- Library code: don't `console.log`. Use the project's logger (`pino` preferred, `winston`, `bunyan`).
- App entrypoints / CLIs: `console.log` is OK.
- Emit JSON in production (one event per line) — see [`structured-logging.md`](structured-logging.md).

## Testing

- **`bun test`** is the default. API is Jest-compatible: `describe`, `test`/`it`, `expect`, `beforeAll`/`afterAll`, `mock()`, `spyOn()`.
- Co-locate tests (`foo.test.ts` next to `foo.ts`) or `__tests__/` — match project.
- One scenario per `test`/`it`. Descriptive names.
- Mock at the boundary (network, filesystem, time). Don't mock what you own.
- Time-dependent tests: `Bun.nanoseconds()` / fake timers. With Vitest/Jest use `vi.useFakeTimers()` / `jest.useFakeTimers()`.
- Watch mode: `bun test --watch`. Coverage: `bun test --coverage`.

## Build / Test commands

```bash
bun install                      # install deps
bun test                         # run tests
bun test --watch                 # watch mode
bun run tsc --noEmit             # type-check
bun run lint                     # eslint via script
bun build ./src/index.ts --outdir ./dist   # bundle
bun run dev                      # whatever 'dev' script is
```

If the project uses Node tooling instead, swap `bun run` for `pnpm`/`npm`/`yarn` and `bun test` for `vitest` / `jest`.

## React / frontend (if applicable)

- Functional components + hooks. No class components in new code.
- Memoize wisely — `useMemo`/`useCallback` only for measurable wins or referential identity for deps.
- Inline object/array props create new refs every render — wrap if passed to memoized children.
- Keep components small. Lift state only when needed.

## Node / backend (if applicable)

- Async: `async`/`await` over `.then()` chains.
- No blocking sync FS in hot paths (`fs.readFileSync` only for startup config).
- Validate input at boundaries (`zod`, `valibot`, `joi`).
- HTTP server in Bun: `Bun.serve({ fetch })` is the idiomatic primitive. Express/Hono/Elysia all work on Bun if preferred.

## Security

- Never hardcode secrets. Use `Bun.env.X` / `process.env.X` + validate at startup. `Bun.env` is the same object as `process.env` in Bun.
- HTML output: escape user input. React does this by default — don't use `dangerouslySetInnerHTML` with untrusted data.
- SQL: parameterized queries / ORM. Never string-concat user input. Bun's built-in `Bun.sql` and `bun:sqlite` use parameter binding — use it.
- `eval`, `Function()`, `Bun.spawn` / `child_process.exec` with user input — don't.

## Common pitfalls

- `==` vs `===` — always `===`.
- Forgetting `await` on async functions — enable `@typescript-eslint/no-floating-promises`.
- Mutating props/state directly in React.
- Type assertions (`as Foo`) hiding real type errors — narrow properly.
- Date handling: prefer `Temporal` (when available) or `date-fns` / `dayjs` over native `Date` for arithmetic.
- Assuming Bun behaves like Node everywhere — most APIs match, but a few (streams, some `crypto` corners, native addons) differ. Check before depending on edge cases.
- Forgetting that `bun test` ≠ `vitest`/`jest` exactly — most APIs match, but some matchers/plugins don't. Verify when porting.
