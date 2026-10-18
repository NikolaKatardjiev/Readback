# 03 — Asking git

[← 02. Targets](02-targets.md) · [04. Building context →](04-context.md)

## Goal

`parseTarget` only figures out *which* part of the repo a `Scope` means.
This chapter adds the functions that actually ask git for content: which
files changed, what the diff looks like, which files live under a path, and
what a file looked like at a given revision. These finish `src/core/scope.ts`
and are what `context.ts` (next chapter) builds on.

## Concepts

### Buffers vs strings

Node represents raw bytes as a `Buffer`, and text as a `string` — decoding
between the two is something *you* ask for, not something that happens
automatically. Most languages hide this (Python 3's `str` vs `bytes` is the
closest analogue; Go's `[]byte` vs `string` is another). `execFile` lets you
choose per call via the `encoding` option:

```ts
// text — stdout comes back already decoded as UTF-8 (the default encoding)
const { stdout } = await execFileAsync("git", ["diff"], { cwd: repo });
// stdout: string

// raw bytes — nothing is decoded; you get exactly what git wrote
const { stdout } = await execFileAsync("git", ["show", "HEAD:img.png"], {
  cwd: repo,
  encoding: "buffer",
});
// stdout: Buffer
```

`readFileAtRevision` deliberately asks for `encoding: "buffer"`. If it asked
for a string instead, a binary file (an image, say) would get silently
mangled — UTF-8 decoding assumes the bytes *are* UTF-8 text, and binary data
usually isn't. The caller (`context.ts`) decides whether it's safe to call
`.toString("utf8")` on the result, after checking for binary content.

### `maxBuffer`

`execFile`'s default output limit is 1 MiB; exceed it and the call rejects
with an error instead of truncating silently. A large diff or `git show` on
a big file can easily cross that. Both `git()` and `readFileAtRevision` pass
an explicit, generous `maxBuffer` (64 MiB) rather than relying on the
default — a good habit whenever a subprocess's output size depends on user
data you don't control.

### Narrower parameter types

```ts
export async function changedFiles(
  repo: string,
  scope: { kind: "range"; base: string; head: string },
): Promise<string[]>
```

`changedFiles` doesn't take a full `Scope` — it takes an **inline object
type** that happens to match only the `"range"` branch. This documents, at
the type level, "this function only makes sense for range scopes" without
needing a second named type. Any `Scope` value narrowed to `kind === "range"`
(by an `if` or a ternary) satisfies this parameter type automatically;
TypeScript doesn't need you to name every shape you use.

## Exercise

Add to `src/core/scope.ts` (below `parseTarget`):

```ts
export async function resolveCommit(repo: string, scope: Scope): Promise<string>;

export async function changedFiles(
  repo: string,
  scope: { kind: "range"; base: string; head: string },
): Promise<string[]>;

export async function unifiedDiff(
  repo: string,
  scope: { kind: "range"; base: string; head: string },
): Promise<string>;

export async function filesUnderPath(
  repo: string,
  scope: { kind: "path"; path: string; rev: string },
): Promise<string[]>;

export async function readFileAtRevision(repo: string, rev: string, path: string): Promise<Buffer>;
```

Required behavior:

- `resolveCommit(repo, scope)`: the commit SHA that ends up recorded as
  `Quiz.commit`. For a `"range"` scope, resolve `scope.head`; for a `"path"`
  scope, resolve `scope.rev`. Reuse `resolveRevision` from chapter 02.
- `changedFiles(repo, scope)`: paths of files that differ between `base` and
  `head` (`git diff --name-only base..head`). Empty diff → `[]`, not `[""]`.
- `unifiedDiff(repo, scope)`: the raw unified diff text for the range
  (`git diff base..head`).
- `filesUnderPath(repo, scope)`: paths of files that exist under
  `scope.path` at `scope.rev` (`git ls-tree -r --name-only rev -- path`).
  Same empty-output rule as `changedFiles`.
- `readFileAtRevision(repo, rev, path)`: the content of `path` as it existed
  at `rev`, as a `Buffer` — may be binary. Throws `ScopeError` on failure
  (file didn't exist at that revision, bad revision, etc).

## Tests

Add to `test/scope.test.ts`. First, extend the import from `../src/core/scope.ts`
to also bring in the new functions:

```ts
import {
  parseTarget,
  resolveCommit,
  changedFiles,
  filesUnderPath,
  readFileAtRevision,
  isGitRepo,
  ScopeError,
} from "../src/core/scope.ts";
```

Then append these tests:

```ts
test("resolveCommit: resolves HEAD for a path scope", async () => {
  const repo = await createTmpRepo();
  try {
    const sha = await repo.commitFile("a.txt", "hello\n", "init");
    const scope = await parseTarget(repo.dir, "path:a.txt");
    assert.equal(await resolveCommit(repo.dir, scope), sha);
  } finally {
    await repo.cleanup();
  }
});

test("changedFiles and readFileAtRevision for a range scope", async () => {
  const repo = await createTmpRepo();
  try {
    await repo.commitFile("unrelated.txt", "x\n", "unrelated"); // exists before base; untouched after
    const base = await repo.commitFile("a.txt", "one\n", "base");
    const head = await repo.commitFile("a.txt", "two\n", "change a");

    const scope = { kind: "range" as const, base, head };
    assert.deepEqual(await changedFiles(repo.dir, scope), ["a.txt"]);

    const content = await readFileAtRevision(repo.dir, head, "a.txt");
    assert.equal(content.toString("utf8"), "two\n");
  } finally {
    await repo.cleanup();
  }
});

test("filesUnderPath lists files nested under a directory", async () => {
  const repo = await createTmpRepo();
  try {
    await repo.commitFile("src/a.ts", "export const a = 1;\n", "a");
    await repo.commitFile("src/nested/b.ts", "export const b = 2;\n", "b");
    await repo.commitFile("other.ts", "export const c = 3;\n", "c");

    const scope = { kind: "path" as const, path: "src/", rev: "HEAD" };
    const files = (await filesUnderPath(repo.dir, scope)).sort();
    assert.deepEqual(files, ["src/a.ts", "src/nested/b.ts"]);
  } finally {
    await repo.cleanup();
  }
});
```

Note the `{ kind: "range" as const, base, head }` object literals in the
tests: without `as const`, TypeScript would infer `kind: string` for that
property (widening the literal), which wouldn't match the narrower parameter
type `changedFiles` expects. `as const` tells it "keep this as the literal
`"range"`, not the general `string`."

## Hints

<details><summary>Hint 1</summary>

`resolveCommit` is a two-branch `if`/ternary on `scope.kind`, each branch
calling `resolveRevision` with a different field.

</details>

<details><summary>Hint 2</summary>

Both `changedFiles` and `filesUnderPath` call `git(...)` and then need to
turn one blob of stdout into a list of paths. Split on `"\n"` — but
`"".split("\n")` is `[""]`, not `[]`, so check `out.length === 0` first and
return `[]` directly in that case.

</details>

<details><summary>Hint 3</summary>

`readFileAtRevision` uses `execFileAsync` directly (not the `git()` helper),
because it needs `encoding: "buffer"` and `git()` always decodes to a
trimmed string. Wrap it in its own `try`/`catch` that throws `ScopeError`,
the same pattern as `git()`.

</details>

## Solution

<details><summary>Solution</summary>

Add below `parseTarget` in `src/core/scope.ts`:

```ts
/** The commit SHA that ends up recorded in Quiz.commit, for reproducibility. */
export async function resolveCommit(repo: string, scope: Scope): Promise<string> {
  const rev = scope.kind === "range" ? scope.head : scope.rev;
  return resolveRevision(repo, rev);
}

/** Paths of files that differ between `scope.base` and `scope.head`. */
export async function changedFiles(repo: string, scope: { kind: "range"; base: string; head: string }): Promise<string[]> {
  const out = await git(repo, ["diff", "--name-only", `${scope.base}..${scope.head}`]);
  return out.length === 0 ? [] : out.split("\n");
}

/** Unified diff text for a range scope. */
export async function unifiedDiff(repo: string, scope: { kind: "range"; base: string; head: string }): Promise<string> {
  return git(repo, ["diff", `${scope.base}..${scope.head}`]);
}

/** Paths of files that exist under `scope.path` at `scope.rev`. */
export async function filesUnderPath(repo: string, scope: { kind: "path"; path: string; rev: string }): Promise<string[]> {
  const out = await git(repo, ["ls-tree", "-r", "--name-only", scope.rev, "--", scope.path]);
  return out.length === 0 ? [] : out.split("\n");
}

/** The content of `path` as it existed at `rev`, as a Buffer (may be binary). */
export async function readFileAtRevision(repo: string, rev: string, path: string): Promise<Buffer> {
  try {
    const { stdout } = await execFileAsync("git", ["show", `${rev}:${path}`], {
      cwd: repo,
      encoding: "buffer",
      maxBuffer: 1024 * 1024 * 64,
    });
    return stdout;
  } catch (err: unknown) {
    const message = err instanceof Error ? err.message : String(err);
    throw new ScopeError(`could not read ${path} at ${rev} in ${repo}: ${message}`);
  }
}
```

</details>

## Watch out

> **Watch out:** `maxBuffer` is small by default (1 MiB) and `execFile`
> *rejects* rather than truncates when it's exceeded. Every git call that
> could plausibly return a lot of output in this project passes an explicit,
> generous `maxBuffer`. If you ever see a mysterious `ENOBUFS`-style failure
> from a subprocess call elsewhere, this is the first thing to check.

> **Watch out:** splitting empty stdout. `"".split("\n")` gives you `[""]`
> — a one-element array containing an empty string — not `[]`. Both
> `changedFiles` and `filesUnderPath` check `out.length === 0` first and
> return `[]` directly, otherwise a scope with zero changed files would
> incorrectly report one changed file with an empty-string path.

## Checkpoint

```
$ npm run typecheck
$ npm test
```

Expected: `tsc` reports no errors, 8 tests pass (5 from chapter 02, 3 new).

```
$ git add src/core/scope.ts test/scope.test.ts
$ git commit -m "core: finish scope.ts git queries"
```

`scope.ts` is done. Next: [04 — Building context](04-context.md), which
turns a `Scope` into the text block sent to the model.
