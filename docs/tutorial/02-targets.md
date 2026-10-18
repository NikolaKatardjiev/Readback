# 02 — Targets

[← 01. TypeScript crash course](01-typescript-crash-course.md) · [03. Asking git →](03-git.md)

## Goal

Every Readback run starts with a **target** string the user types on the
command line: `a1b2c3..d4e5f6`, `main...feature`, or `path:src/auth/`. This
chapter builds the first half of `src/core/scope.ts`: the type that
represents "what part of the repo are we quizzing" (`Scope`), and the parser
that turns a target string into one, `parseTarget`. Everything downstream —
building context, validating citations — only ever looks at a `Scope`, never
at the raw string the user typed.

This is also where you start running real git commands from Node, safely.

## Concepts

### Discriminated unions, for real this time

Chapter 01 introduced union types. `Scope` is the pattern that makes unions
actually useful in a program, not just a toy:

```ts
type Scope =
  | { kind: "range"; base: string; head: string }
  | { kind: "path"; path: string; rev: string };
```

Both branches share a `kind` field, but with a different *literal string
type* each time ("range" vs "path" — not `string` in general). That shared
field is called the **tag** (or **discriminant**), and `switch`/`if` on it
lets TypeScript narrow which other fields exist:

```ts
function describe(scope: Scope): string {
  switch (scope.kind) {
    case "range":
      // TypeScript knows scope.base and scope.head exist here.
      return `${scope.base}..${scope.head}`;
    case "path":
      // and here it knows scope.path and scope.rev exist instead.
      return `${scope.path}@${scope.rev}`;
  }
}
```

Try `scope.path` inside the `"range"` case — tsc rejects it at compile time.
This is the TypeScript equivalent of a Python `@dataclass` tagged union, a Go
interface with a type switch, or a C# pattern-matched `record` hierarchy —
except here it's a plain object shape with no base class or interface
declaration needed. The union *is* the type.

One more thing worth noticing: `switch` over a union's tag also gets
*exhaustiveness checking* if you assign the result to a variable typed
`never` in a `default` case — useful once a union has three or more members,
less important here with just two, but keep it in mind.

### Custom errors

```ts
export class ScopeError extends Error {}
```

That's the whole class — no constructor, no extra fields. The exercise is
"subclass `Error` so `catch` blocks and tests can tell *what kind* of thing
failed", not "add new behavior". `instanceof ScopeError` is how `cli.ts` will
later decide "print a clean message" versus "print a stack trace", and how
tests assert `assert.rejects(fn, ScopeError)`.

### `unknown` in `catch` blocks

With `strict` mode on, TypeScript types a caught value as `unknown`, not
`any` (older TS, and plenty of other languages, give you something you can
call methods on immediately — `unknown` refuses until you check):

```ts
try {
  // ...
} catch (err: unknown) {
  const message = err instanceof Error ? err.message : String(err);
  throw new ScopeError(`... ${message}`);
}
```

You can't write `err.message` directly — `err` might not even be an `Error`
(JavaScript lets you `throw "a plain string"`, unlike Python or C#, where
everything thrown is at least an exception/`Exception`). `instanceof Error`
narrows `unknown` down to `Error` for that branch; otherwise you fall back to
`String(err)`.

### Running git: `execFile`, not `exec`

> **Watch out:** `exec` runs your command line through a shell. `execFile`
> with an argument array never invokes a shell at all. A branch name or path
> containing `;`, `` ` ``, or `$()` is just a string argument to `execFile` —
> with `exec` it could be interpreted as shell syntax. Every git call in this
> project builds an argument array (`git(repo, ["diff", ...])`), never a
> concatenated command string. This matters because target strings and paths
> come from the user (or, later, a web form).

`child_process.execFile` is callback-based by default:

```ts
import { execFile } from "node:child_process";
execFile("git", ["status"], (err, stdout, stderr) => { /* ... */ });
```

`node:util`'s `promisify` turns that callback signature into a function that
returns a `Promise`, so you can `await` it instead of nesting callbacks —
Node's standard escape hatch for any older callback-style API:

```ts
import { promisify } from "node:util";
const execFileAsync = promisify(execFile);
const { stdout } = await execFileAsync("git", ["status"], { cwd: repo });
```

(C#'s `Task.FromAsync`, or wrapping a Go callback API in a channel, are the
closest analogues — "adapt an old-style async API to the modern one,
once, in one place.")

### Object parameters instead of overloads

`execFile`'s third argument is an options object (`{ cwd, maxBuffer }`)
rather than extra positional parameters. You'll use this pattern yourself:
it's how you add optional configuration to a function without juggling
parameter order or writing overloads (as you might in C# or Java).

## Exercise

Create `src/core/scope.ts` with these exports:

```ts
export type Scope =
  | { kind: "range"; base: string; head: string }
  | { kind: "path"; path: string; rev: string };

export class ScopeError extends Error {}

export async function isGitRepo(repo: string): Promise<boolean>;
export async function resolveRevision(repo: string, rev: string): Promise<string>;
export async function parseTarget(repo: string, target: string): Promise<Scope>;
```

Plus a private (non-exported) helper function `git(repo, args)` that both
`isGitRepo` and `resolveRevision` can use, and a private `mergeBase(repo, a,
b)` that `parseTarget` uses for three-dot ranges.

Required behavior:

- `git(repo, args)`: runs `git` with `args` in `cwd: repo`, returns trimmed
  stdout. On failure, throws `ScopeError` with a message that includes the
  command and the underlying error message. Use a generous `maxBuffer`
  (64 MiB) — git output for large diffs can exceed Node's small default.
- `isGitRepo(repo)`: `true` if `repo` is a path git recognizes as inside a
  work tree (`git rev-parse --is-inside-work-tree`), `false` for anything
  that errors (not a repo, doesn't exist) — don't let this one throw.
- `resolveRevision(repo, rev)`: resolves any revision expression (branch,
  tag, SHA, `"HEAD"`) to a full commit SHA, via
  `git rev-parse --verify <rev>^{commit}`.
- `parseTarget(repo, target)`:
  - `"path:DIR"` → `{ kind: "path", path: "DIR", rev: "HEAD" }`. Empty path
    (`"path:"`) throws `ScopeError`.
  - Three-dot range `"a...b"` → `{ kind: "range", base: <merge-base of a and
    b>, head: b }`. Check for `"..."` *before* `".."`, since every three-dot
    string also contains the two-dot substring.
  - Two-dot range `"a..b"` → `{ kind: "range", base: a, head: b }`, kept
    exactly as given — no git lookup needed.
  - Anything else throws `ScopeError`.

## Tests

Create `test/helpers.ts` (not a test file itself — no `.test.ts` suffix, so
`node --test` won't try to run it). `parseTarget` needs a *real* git
repository to test three-dot ranges against, so this helper creates a
disposable one per test under the OS temp directory
(`os.tmpdir()`), and returns a cleanup function to delete it afterwards:

```ts
// helpers.ts — not itself a test file (no *.test.ts suffix, so
// `node --test` ignores it), just shared setup used by several tests: a
// throwaway git repository under the OS temp dir.

import { execFile } from "node:child_process";
import { promisify } from "node:util";
import { mkdir, mkdtemp, rm, writeFile } from "node:fs/promises";
import * as os from "node:os";
import * as path from "node:path";

const execFileAsync = promisify(execFile);

export type TmpRepo = {
  dir: string;
  git(...args: string[]): Promise<string>;
  commitFile(filePath: string, content: string, message: string): Promise<string>;
  cleanup(): Promise<void>;
};

/** Create an empty git repo in a fresh temp directory. */
export async function createTmpRepo(): Promise<TmpRepo> {
  const dir = await mkdtemp(path.join(os.tmpdir(), "readback-test-"));

  async function git(...args: string[]): Promise<string> {
    const { stdout } = await execFileAsync("git", args, { cwd: dir });
    return stdout.trim();
  }

  await git("init", "--initial-branch=main");
  // A commit needs an author; set one locally so the test doesn't depend on
  // the machine's global git config.
  await git("config", "user.email", "test@example.com");
  await git("config", "user.name", "Readback Test");

  async function commitFile(filePath: string, content: string, message: string): Promise<string> {
    const fullPath = path.join(dir, filePath);
    await mkdir(path.dirname(fullPath), { recursive: true });
    await writeFile(fullPath, content, "utf8");
    await git("add", filePath);
    await git("commit", "-m", message);
    return git("rev-parse", "HEAD");
  }

  return {
    dir,
    git,
    commitFile,
    cleanup: () => rm(dir, { recursive: true, force: true }),
  };
}
```

Then create `test/scope.test.ts`:

```ts
// scope.test.ts — exercises parseTarget, resolveCommit and the git query
// helpers against a real (throwaway) git repository.

import { test } from "node:test";
import assert from "node:assert/strict";
import { createTmpRepo } from "./helpers.ts";
import {
  parseTarget,
  isGitRepo,
  ScopeError,
} from "../src/core/scope.ts";

test("isGitRepo", async () => {
  const repo = await createTmpRepo();
  try {
    assert.equal(await isGitRepo(repo.dir), true);
    assert.equal(await isGitRepo("/definitely/not/a/repo"), false);
  } finally {
    await repo.cleanup();
  }
});

test("parseTarget: path: target defaults to HEAD", async () => {
  const repo = await createTmpRepo();
  try {
    const scope = await parseTarget(repo.dir, "path:src/");
    assert.deepEqual(scope, { kind: "path", path: "src/", rev: "HEAD" });
  } finally {
    await repo.cleanup();
  }
});

test("parseTarget: two-dot range keeps both sides as given", async () => {
  const repo = await createTmpRepo();
  try {
    const first = await repo.commitFile("a.txt", "one\n", "first");
    const second = await repo.commitFile("a.txt", "two\n", "second");
    const scope = await parseTarget(repo.dir, `${first}..${second}`);
    assert.deepEqual(scope, { kind: "range", base: first, head: second });
  } finally {
    await repo.cleanup();
  }
});

test("parseTarget: three-dot range resolves to the merge base", async () => {
  const repo = await createTmpRepo();
  try {
    const base = await repo.commitFile("a.txt", "one\n", "base");
    await repo.git("checkout", "-b", "feature");
    const head = await repo.commitFile("a.txt", "two\n", "on feature");

    const scope = await parseTarget(repo.dir, "main...feature");
    assert.equal(scope.kind, "range");
    // base is resolved to the merge-base commit; head is kept as the ref
    // expression the user typed (git itself resolves it on every later use).
    assert.equal(scope.kind === "range" ? scope.base : "", base);
    assert.equal(scope.kind === "range" ? scope.head : "", "feature");
    assert.ok(head.length > 0); // the commit exists; we just don't store its SHA in the scope
  } finally {
    await repo.cleanup();
  }
});

test("parseTarget: rejects nonsense targets", async () => {
  const repo = await createTmpRepo();
  try {
    await assert.rejects(() => parseTarget(repo.dir, "not-a-target"), ScopeError);
  } finally {
    await repo.cleanup();
  }
});
```

## Hints

<details><summary>Hint 1</summary>

`git(repo, args)` should wrap `execFileAsync` in a `try`/`catch`. Both
`isGitRepo` and `resolveRevision` can be one-liners that call it.

</details>

<details><summary>Hint 2</summary>

For `parseTarget`, check the prefixes/substrings in this order: `"path:"`
first, then `"..."`, then `".."`. If you check `".."` before `"..."` you'll
split a three-dot target in the wrong place, since `"a...b".includes("..")`
is also `true`.

</details>

<details><summary>Hint 3</summary>

`target.split("...", 2)` (or `.split("..", 2)`) gives you an array; destructure
it as `const [base, head] = ...`. Because of `noUncheckedIndexedAccess`,
array elements accessed by destructuring from `split` are typed as possibly
missing (`string | undefined` in general), so check `!base || !head` and
throw `ScopeError` before using them as plain `string`s.

</details>

## Solution

<details><summary>Solution</summary>

```ts
// scope.ts — turns a user-supplied "target" string into a `Scope`, and runs
// the small set of read-only git commands the rest of the pipeline needs.
//
// Rule for this whole src/core/ directory: never touch process.env or
// console. Functions take plain arguments and either return a value or
// throw an Error. That makes every function here trivially unit-testable
// (see test/scope.test.ts) and reusable from a future web server.

import { execFile } from "node:child_process";
import { promisify } from "node:util";

// execFile runs a program directly (no shell), so a filename or branch name
// containing e.g. "; rm -rf /" is just an argument string, never executed.
const execFileAsync = promisify(execFile);

// A Scope is the one shape every later pipeline stage (context, quiz
// validation) needs to understand. This is a *discriminated union*: the
// `kind` field is the "tag" TypeScript uses to narrow which other fields
// are available. Once you check `scope.kind === "range"`, TypeScript knows
// `scope.base` and `scope.head` exist; in the `"path"` branch it knows
// `scope.path` and `scope.rev` exist instead.
export type Scope =
  | { kind: "range"; base: string; head: string }
  | { kind: "path"; path: string; rev: string };

// Thrown for anything the user got wrong: bad target syntax, a path outside
// a git repo, an unknown revision. cli.ts turns this into a clean message
// instead of a stack trace.
export class ScopeError extends Error {}

/** Run a read-only git subcommand in `repo` and return trimmed stdout. */
async function git(repo: string, args: string[]): Promise<string> {
  try {
    const { stdout } = await execFileAsync("git", args, { cwd: repo, maxBuffer: 1024 * 1024 * 64 });
    return stdout.trimEnd();
  } catch (err: unknown) {
    // `err` from child_process is typed `unknown` in strict mode — we can't
    // assume it has a `.message` until we check. Node's error objects from
    // execFile do, so we narrow with `instanceof Error` and fall back for
    // anything stranger (e.g. someone throws a string).
    const message = err instanceof Error ? err.message : String(err);
    throw new ScopeError(`git ${args.join(" ")} failed in ${repo}: ${message}`);
  }
}

/** True if `repo` is a path git recognises as (inside) a work tree. */
export async function isGitRepo(repo: string): Promise<boolean> {
  try {
    const out = await git(repo, ["rev-parse", "--is-inside-work-tree"]);
    return out === "true";
  } catch {
    return false;
  }
}

/** Resolve any revision expression (branch, tag, SHA, "HEAD") to a full SHA. */
export async function resolveRevision(repo: string, rev: string): Promise<string> {
  return git(repo, ["rev-parse", "--verify", `${rev}^{commit}`]);
}

/** The commit where the two sides of a "triple dot" range diverge. */
async function mergeBase(repo: string, a: string, b: string): Promise<string> {
  return git(repo, ["merge-base", a, b]);
}

/**
 * Parse a target string into a Scope.
 *
 * - "path:some/dir"      -> { kind: "path", path: "some/dir", rev: "HEAD" }
 * - "a1b2c3..d4e5f6"     -> { kind: "range", base: "a1b2c3", head: "d4e5f6" }
 * - "main...feature"     -> { kind: "range", base: <merge-base of the two>, head: "feature" }
 *
 * The three-dot form needs git (to compute the merge base) so this function
 * is async and takes the repo path, unlike a pure string parser.
 */
export async function parseTarget(repo: string, target: string): Promise<Scope> {
  if (target.startsWith("path:")) {
    const path = target.slice("path:".length);
    if (path.length === 0) {
      throw new ScopeError(`empty path in target "${target}"`);
    }
    return { kind: "path", path, rev: "HEAD" };
  }

  // Order matters: "a...b" also contains "..", so check the three-dot form
  // first, otherwise it would be split in the wrong place.
  if (target.includes("...")) {
    const [base, head] = target.split("...", 2);
    if (!base || !head) {
      throw new ScopeError(`invalid range target "${target}"`);
    }
    const base2 = await mergeBase(repo, base, head);
    return { kind: "range", base: base2, head };
  }

  if (target.includes("..")) {
    const [base, head] = target.split("..", 2);
    if (!base || !head) {
      throw new ScopeError(`invalid range target "${target}"`);
    }
    return { kind: "range", base, head };
  }

  throw new ScopeError(
    `could not parse target "${target}" (expected "path:DIR", "base..head" or "base...head")`,
  );
}
```

</details>

## Watch out

> **Watch out:** Git's two-dot and three-dot ranges mean genuinely different
> things, and `Scope` has no field to remember which one the user typed.
> `parseTarget` resolves the three-dot form's merge-base *immediately*
> (`git merge-base`) and stores the result as an ordinary `base`/`head` pair.
> That means a three-dot target's `base` ends up as a resolved commit SHA,
> while its `head` is left as whatever ref the user typed (e.g. `"feature"`,
> not a SHA) — asymmetric on purpose. See the three-dot test above: `base`
> is checked against a known SHA, `head` against the literal string
> `"feature"`.

> **Watch out:** `execFile`, not `exec`. If you reach for `child_process.exec`
> out of habit (it's the more commonly copy-pasted one), every path and
> branch name in this project becomes a potential shell-injection point.
> `execFile` plus an argument array is non-negotiable here.

## Checkpoint

```
$ npm run typecheck
$ npm test
```

Expected: `tsc` reports no errors, and `node --test` reports 5 passing tests
(all in `scope.test.ts`) — `test/helpers.ts` isn't picked up since it has no
`.test.ts` suffix.

```
$ git add src/core/scope.ts test/helpers.ts test/scope.test.ts
$ git commit -m "core: parse targets into a Scope"
```

Next: [03 — Asking git](03-git.md), where `scope.ts` grows the functions that
actually fetch diffs and file contents.
