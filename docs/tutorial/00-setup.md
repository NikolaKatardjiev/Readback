# 00 — Setup

[README](README.md) · [01. TypeScript crash course →](01-typescript-crash-course.md)

## Goal

Get a working Node + TypeScript project in the repository root: Node
installed, `package.json`, `tsconfig.json`, and a round trip of "write a
`.ts` file, run it, test it" before any real Readback code exists. Everything
later in this tutorial reuses exactly this setup.

## Concepts

**Node is the runtime, npm is the package manager.** If you know Python:
Node is the interpreter (like `python`), npm is `pip` + a chunk of
`venv`/`poetry`. If you know Go: Node is the `go` runtime, npm is `go mod`.
If you know C#: Node is the CLR-ish bit that runs your code, npm is NuGet.

**`package.json`** is your project's manifest — name, version, scripts,
and (crucially) a list of *dependencies with version ranges*, not exact
installed versions. It's the `.csproj`/`go.mod`/`pyproject.toml` of the
JS world.

**`node_modules/`** is where npm actually puts downloaded packages (and
their dependencies, and *their* dependencies...). It's regenerated from
`package.json` + the lockfile, never hand-edited, and never committed —
this repo's `.gitignore` already excludes it.

**`package-lock.json`** pins the *exact* version of every package and
sub-dependency that got installed, so `npm install` is reproducible on
another machine. Unlike `node_modules`, **the lockfile is committed** —
it's the equivalent of `go.sum` or a fully-pinned `requirements.txt`.

**Node can run `.ts` files directly** (no separate build step) on modern
versions, by *stripping* type annotations before running the file as
plain JavaScript — it does not check that the types are correct, it just
deletes them. Type *checking* is a separate, optional step you run with
`tsc` (the TypeScript compiler). In this project `tsc` never produces
output files (`noEmit: true`); it only checks and reports errors. This
split — "Node runs it regardless, `tsc` tells you if the types lie" — is
the single most important mental model shift from a language like Go or
C# where the compiler and the runtime are the same gate.

**ES Modules.** `package.json` sets `"type": "module"`, which tells Node
to treat every `.js`/`.ts` file as an ES module (`import`/`export`)
instead of Node's older CommonJS (`require`/`module.exports`). You'll see
why import paths need a literal `.ts` extension in a "Watch out" below.

**`node:test` and `node:assert/strict`** are Node's built-in test runner
and assertion library — no Jest/Mocha/pytest/xUnit install needed. You run
tests with `node --test`, and each `test(name, fn)` block is one test case.

## Exercise

### 1. Install Node 24+

Use a version manager rather than your distro's package manager, so you can
pin per-project and upgrade easily later.

**fnm** (recommended, fast, written in Rust):

```
$ curl -fsSL https://fnm.vercel.app/install | bash
# restart your shell, then:
$ fnm install 24
$ fnm use 24
```

**nvm** (alternative, older, more widely documented):

```
$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
# restart your shell, then:
$ nvm install 24
$ nvm use 24
```

Any Node `>=24` works — this was written against v26, which is fine.

Pin the version for this project and for anyone else who clones it:

```
$ echo "24" > .nvmrc
$ cp .nvmrc .node-version
```

(`.nvmrc` is read by nvm, `.node-version` by fnm and a few other tools —
harmless to have both.)

Verify:

```
$ node --version
$ npm --version
```

You should see `v24.x.x` or newer, and an npm version starting with `1` (10+).

### 2. Create `package.json`

Create `package.json` at the repo root with this content:

```json
{
  "name": "readback",
  "version": "0.1.0",
  "description": "Generate comprehension quizzes about code in a Git repository.",
  "type": "module",
  "engines": {
    "node": ">=24"
  },
  "scripts": {
    "start": "node src/cli.ts",
    "typecheck": "tsc",
    "test": "node --test test/**/*.test.ts"
  },
  "dependencies": {
    "zod": "4.6.7"
  },
  "devDependencies": {
    "typescript": "7.0.2",
    "@types/node": "24.19.2"
  }
}
```

(`start` points at `src/cli.ts`, which doesn't exist yet — that's fine, you
won't run it until [chapter 09](09-generate.md).)

Now install the dependencies with **exact, pinned versions** (no `^` or `~`
ranges — this tutorial pins every version deliberately so the code you write
matches what's documented here):

```
$ npm install --save-exact zod@4.6.7
$ npm install --save-exact --save-dev typescript@7.0.2 @types/node@24.19.2
```

This creates `node_modules/` and `package-lock.json`. Check that both
dependencies landed in `package.json` with exact versions (no `^`/`~`), and
commit the lockfile — it's not gitignored on purpose.

Field by field:
- `type: "module"` — use ES modules (`import`/`export`) everywhere, not
  Node's older CommonJS (`require`).
- `engines.node` — documents the minimum Node version; doesn't enforce it by
  itself, but tools like `npm` can warn if it's violated.
- `scripts` — named shell commands runnable as `npm run <name>` (`start` and
  `test` can drop the `run`: `npm start`, `npm test`).
- `dependencies` — packages your *shipped code* needs at runtime (here:
  `zod`, used from [chapter 05](05-quiz-schemas.md) onward).
- `devDependencies` — packages only needed while developing (the compiler,
  type declarations) — never required to just *run* the CLI.

`@types/node` deserves a note: TypeScript doesn't know about Node's built-in
modules (`node:fs`, `node:child_process`, ...) on its own. `@types/node`
is a dev-only package of pure `.d.ts` type declarations for them — it adds
nothing at runtime, only type information for `tsc`.

### 3. Create `tsconfig.json`

Create `tsconfig.json` at the repo root:

```json
{
  "compilerOptions": {
    /* We never emit .js files: Node runs .ts source directly (type stripping).
       tsc's only job here is to check types and fail the build on errors. */
    "noEmit": true,

    /* Modern Node module system: package.json "type": "module", imports
       resolve the same way Node itself resolves them at runtime. */
    "module": "nodenext",
    "moduleResolution": "nodenext",

    /* Node's type-stripping does NOT remove anything beyond type syntax -
       no enums, no namespaces, no parameter properties. This flag makes tsc
       refuse to compile if we accidentally use one of those. */
    "erasableSyntaxOnly": true,

    /* Node's type stripping (and some bundlers) require relative imports to
       keep the literal file extension, e.g. `import { x } from "./x.ts"`.
       This flag teaches tsc to expect and allow ".ts" in specifiers. */
    "allowImportingTsExtensions": true,

    /* Forces `import type { Foo }` for type-only imports instead of letting
       tsc guess. This matters once imports are erased at runtime: a type
       that sneaks in as a value import would crash Node at run time. */
    "verbatimModuleSyntax": true,

    /* Strongest type-safety net. Turns on all strict checks (null checks,
       no implicit any, etc). */
    "strict": true,

    /* Reading arr[i] or obj[key] normally types as T, pretending the index
       always exists. This makes it `T | undefined` instead, which is more
       honest and catches real off-by-one / missing-key bugs. */
    "noUncheckedIndexedAccess": true,

    /* Belt-and-braces strictness that pairs well with the above. */
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,

    /* Modern target/lib: we're only ever running on Node 24+. */
    "target": "esnext",
    "lib": ["esnext"],

    /* Bring in Node's own type declarations (fs, child_process, ...). */
    "types": ["node"],

    "skipLibCheck": true
  },
  "include": ["src/**/*.ts", "test/**/*.ts", "scripts/**/*.ts"]
}
```

The two ideas worth sitting with:

- **`noEmit: true`**: `tsc` here is a linter, not a compiler. It reads your
  `.ts` files, checks the types, prints errors, and exits — it writes no
  `.js` anywhere. Node is what actually runs your code, by stripping the
  type syntax out, not by reading `tsc`'s output.
- **`module`/`moduleResolution: "nodenext"`**: tells `tsc` "resolve imports
  exactly the way Node resolves them at runtime", which — combined with
  `"type": "module"` in `package.json` — means ES modules, and relative
  imports that must match a real file on disk (see "Watch out" below).

### 4. Write and run your first program

Create `src/hello.ts`:

```ts
export function greet(name: string): string {
  return `Hello, ${name}!`;
}

console.log(greet("Readback"));
console.log(greet(42));
```

That last line is a deliberate mistake — `greet` takes a `string`, we're
passing a `number`. VS Code will underline `42` straight away with
*"Argument of type 'number' is not assignable to parameter of type
'string'."* That's expected, and it's the point of this step: leave it for
now, you'll delete the line in a moment. Run it anyway:

```
$ node src/hello.ts
```

It prints **both** lines, including `Hello, 42!`. Node stripped the types
and ran the JavaScript underneath without checking anything. Now run the
type checker:

```
$ npm run typecheck
```

This reports one error, on the `greet(42)` line. **Nothing ran your code** —
`tsc` only read the source and checked it. This is the core idea of this
whole setup: *types are a static, optional check; they vanish before your
code ever executes.* In Go or C#, a type error like this is a compile
failure — your program never produces a binary. In TypeScript-on-Node, it's
a warning from a separate tool that you chose to run.

Fix it by deleting the bad line:

```ts
export function greet(name: string): string {
  return `Hello, ${name}!`;
}

console.log(greet("Readback"));
```

`npm run typecheck` should now exit cleanly (no output, exit code 0).

### 5. Write and run your first test

Create `test/hello.test.ts`.

## Tests

```ts
import { test } from "node:test";
import assert from "node:assert/strict";
import { greet } from "../src/hello.ts";

test("greet() returns a friendly message", () => {
  assert.equal(greet("Readback"), "Hello, Readback!");
});
```

Run it:

```
$ npm test
```

Note the import: `"../src/hello.ts"`, with the real `.ts` extension — see
"Watch out" below for why that's required, not a typo.

### 6. Clean up

Once both commands pass, delete the scratch files — they're not part of the
real CLI:

```
$ rm src/hello.ts test/hello.test.ts
```

[Chapter 02](02-targets.md) puts real files in `src/` and `test/` next.

## Hints

<details><summary>Hint 1</summary>

If `npm run typecheck` says `No inputs were found`, you either haven't
created `src/hello.ts` yet, or you already deleted it. `tsconfig.json`'s
`include` only looks inside `src/`, `test/`, and `scripts/`.

</details>

<details><summary>Hint 2</summary>

If `npm test` can't find `node:test`, double check your installed Node
version (`node --version`) — it must be 24 or newer, and the command is
`node --test` (with two dashes), not `node test`.

</details>

<details><summary>Hint 3</summary>

If the import in the test file gives you a red squiggle in your editor about
an unresolvable module, make sure you kept the `.ts` extension in
`"../src/hello.ts"` and that `hello.ts` actually exports `greet` (not just
defines it).

</details>

## Solution

<details><summary>Solution</summary>

`package.json`, `tsconfig.json`, `src/hello.ts` and `test/hello.test.ts` are
given verbatim above — there's no separate "solution" for a setup chapter.
If something doesn't run, compare your files line by line against the
listings in the Exercise section.

</details>

## Watch out

> **Watch out:** relative imports need the literal `.ts` extension
> (`import { greet } from "../src/hello.ts"`, not `"../src/hello"` or
> `"../src/hello.js"`). Node's type stripping only erases type *syntax* — it
> does not rewrite module specifiers the way a bundler would. `tsconfig.json`'s
> `allowImportingTsExtensions` exists specifically so `tsc` accepts this
> style instead of flagging it as wrong. You'll write every import in this
> project this way, starting in [chapter 02](02-targets.md).

> **Watch out:** `npm test`'s script is `node --test test/**/*.test.ts` —
> once you delete every `.test.ts` file, running it again will report "no
> tests found" (or similar), which is expected and harmless. There's nothing
> to test until the next chapter adds real source and test files.

> **Watch out:** commit `package-lock.json`. It's not listed in
> `.gitignore` (only `node_modules/` is) — the lockfile is what makes
> `npm install` reproducible for anyone else who clones this repo.

## Editor setup

If you're using VS Code, its built-in TypeScript support (no extension
needed) gives you inline red squigglies for the same errors `tsc` reports,
hover-for-type on any variable, and "Go to Definition" (F12) that jumps into
`node_modules` type declarations, including Node's own
(`@types/node`) — useful whenever a later chapter mentions a Node API you
haven't used before.

## Checkpoint

```
$ npm run typecheck
```
Expected: one error, pointing at `src/hello.ts`'s `greet(42)` line
(before you delete it).

```
$ node src/hello.ts
```
Expected output:
```
Hello, Readback!
Hello, 42!
```
(before you delete the bad line — this is the point: it runs anyway).

After fixing (deleting the `greet(42)` line):

```
$ npm run typecheck
```
Expected: no output, exit code `0`.

```
$ npm test
```
Expected: one passing test (`greet() returns a friendly message`).

Then delete `src/hello.ts` and `test/hello.test.ts`.

Suggested commit:

```
$ git add .nvmrc .node-version package.json package-lock.json tsconfig.json
$ git commit -m "Set up Node/TypeScript project (chapter 00)"
```

(The `hello.ts`/`hello.test.ts` scratch files are deleted, so there's
nothing of them to commit.)

Next: [01 — TypeScript crash course](01-typescript-crash-course.md).
