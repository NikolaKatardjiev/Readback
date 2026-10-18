# 04 — Building context

[← 03. Asking git](03-git.md) · [05. Quiz schemas →](05-quiz-schemas.md)

## Goal

The model never sees your repo directly — it sees whatever text you send it.
This chapter builds `src/core/context.ts`: it turns a `Scope` into that text.
For a range, that's the diff plus the full post-change content of every
changed file; for a path, it's the content of every file under that path.
Every line gets a number so later questions can cite `file:line`. There's
also a character budget: if the content would be too big, the largest files
are dropped first, and the caller is *told* via a `warnings` array — core
code never prints anything itself.

## Concepts

### `import type` and `verbatimModuleSyntax`

`context.ts` needs `Scope` from `scope.ts` only as a *type* (it never
constructs one), but it needs `changedFiles`, `filesUnderPath`, etc. as
*values* it actually calls. With `verbatimModuleSyntax` on (set in chapter
00's `tsconfig.json`), you must say explicitly which is which:

```ts
import type { Scope } from "./scope.ts";
import { changedFiles, filesUnderPath, readFileAtRevision, unifiedDiff } from "./scope.ts";
```

> **Watch out:** this isn't just style. At runtime, Node's type stripping
> erases `import type` lines completely (nothing is left to resolve); a
> plain `import { Scope }` line, if `Scope` is only ever used as a type,
> would compile away in a confusing, implementation-dependent way if TS
> tried to guess for you. `verbatimModuleSyntax` forces every import
> statement to say, up front, whether it survives into the emitted/stripped
> JavaScript. Mixing a type and a value from the same module means **two**
> import statements, as above — more verbose than old-style TypeScript, but
> it matches exactly what happens when Node reads the file.

Compare to Python's `from __future__ import annotations` + `TYPE_CHECKING`
dance, or C#'s `using` being erased the same for types and values (C# has no
such split, which is part of why this feels new) — this is a TypeScript/Node
specific wrinkle that only bites because there's no bundler doing cleanup.

### A private type isn't a design mistake

```ts
type Section = {
  path: string;
  heading: string;
  body: string;
};
```

Not every type needs `export`. `Section` is an implementation detail of how
`buildContext` assembles its output — nothing outside this file needs to
know it exists. Keep internal types private; it's one less thing for a
reader of the module's public API to look at.

### Strings, arrays, `Set`, and `sort`

A handful of standard library pieces do most of the work here:

- `text.split("\n")` / `.join("\n")` — splitting and rejoining lines.
- `String(i + 1).padStart(width)` — right-align a number in a fixed-width
  field, like `printf("%3d", n)` in C or `str(n).rjust(width)` in Python.
- `new Set([...])` with `.has(x)` — O(1) membership testing. Checking
  `SKIP_BASENAMES.includes(base)` on an array would re-scan the whole list
  every call; a `Set` looks it up directly. Reach for `Set` any time you're
  about to write `array.includes(x)` in a loop.
- `array.sort((a, b) => sizeOf(b) - sizeOf(a))` — a custom comparator for
  "biggest first": return negative if `a` should come first, positive if
  `b` should, `0` for a tie. The same idea as Python's
  `sorted(xs, key=..., reverse=True)` or Go's `sort.Slice`, just inlined as
  a two-argument function.

Also note: `Array.prototype.sort` **mutates in place**. `buildContext` is
careful to sort a *copy* (`[...sections].sort(...)`), never the original
`sections` array, so the function's caller isn't surprised by a reordered
array they still hold a reference to.

### Return warnings, don't print them

```ts
export type ContextResult = {
  text: string;
  warnings: string[];
};
```

When the budget is exceeded, `buildContext` doesn't call `console.warn` —
it adds a string to `warnings` and returns it. This is the same discipline
`scope.ts` follows with thrown errors: `src/core/` only ever returns values
or throws; only `cli.ts` is allowed to touch `console`. That split is what
makes `buildContext` trivially testable (assert on the returned array) and
reusable later from a web server that has no terminal to print to.

## Exercise

Create `src/core/context.ts` with these exports:

```ts
export type ContextResult = { text: string; warnings: string[] };
export type ContextOptions = { maxChars?: number };

export function numberLines(text: string): string;
export async function buildContext(repo: string, scope: Scope, options?: ContextOptions): Promise<ContextResult>;
```

Required behavior:

- `numberLines(text)`: prefix every line with a right-aligned line number and
  `": "`, like `cat -n`. The number width should be wide enough for the
  largest line number (so line 1 of a 10-line file is padded to ` 1`, not
  `1`).
- Skip files that "look generated": known lockfile basenames
  (`package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `composer.lock`,
  `Cargo.lock`), minified JS/CSS (`*.min.js`, `*.min.css`, including
  `.min.cjs`/`.min.mjs`), and anything with a path segment of
  `node_modules`, `dist`, `build`, `vendor`, or `.git`.
- Skip files that "look binary": any `Buffer` containing a NUL byte.
- `buildContext(repo, scope, options)`:
  - Default character budget is 60,000 if `options.maxChars` is omitted.
  - For a `"range"` scope: always include the full unified diff (never
    dropped for budget), under a heading mentioning the base/head. Then
    include the full line-numbered content of every non-generated,
    non-binary changed file, under a heading mentioning the file path.
  - For a `"path"` scope: no diff; include the full line-numbered content of
    every non-generated, non-binary file under the path.
  - If the sections don't all fit in the budget, drop the **largest**
    sections first (by heading + body length) until the rest fit, and append
    one warning naming every dropped path. Kept sections stay in their
    original (not size-sorted) order in the output.

## Tests

Create `test/context.test.ts`:

```ts
// context.test.ts — buildContext against a real repo, checking that diffs,
// line numbering, skip rules and the character budget all work together.

import { test } from "node:test";
import assert from "node:assert/strict";
import { createTmpRepo } from "./helpers.ts";
import { buildContext, numberLines } from "../src/core/context.ts";

test("numberLines prefixes each line with a right-aligned number", () => {
  const result = numberLines("a\nb\nc");
  assert.equal(result, "1: a\n2: b\n3: c");
});

test("numberLines pads width for 10+ lines", () => {
  const lines = Array.from({ length: 10 }, (_, i) => `line${i}`).join("\n");
  const result = numberLines(lines);
  // First line gets a 2-character-wide number (padded), to line up with "10".
  assert.ok(result.startsWith(" 1: line0"));
  assert.ok(result.includes("10: line9"));
});

test("buildContext for a range scope includes the diff and full file content", async () => {
  const repo = await createTmpRepo();
  try {
    const base = await repo.commitFile("a.ts", "export const a = 1;\n", "base");
    const head = await repo.commitFile("a.ts", "export const a = 2;\n", "change");

    const { text, warnings } = await buildContext(repo.dir, { kind: "range", base, head });
    assert.equal(warnings.length, 0);
    assert.match(text, /## Diff/);
    assert.match(text, /-export const a = 1;/);
    assert.match(text, /\+export const a = 2;/);
    assert.match(text, /## File \(full content at/);
    assert.match(text, /1: export const a = 2;/);
  } finally {
    await repo.cleanup();
  }
});

test("buildContext for a path scope includes every file under the path", async () => {
  const repo = await createTmpRepo();
  try {
    await repo.commitFile("src/a.ts", "const a = 1;\n", "a");
    await repo.commitFile("src/b.ts", "const b = 2;\n", "b");

    const { text } = await buildContext(repo.dir, { kind: "path", path: "src/", rev: "HEAD" });
    assert.match(text, /src\/a\.ts/);
    assert.match(text, /src\/b\.ts/);
    assert.match(text, /1: const a = 1;/);
  } finally {
    await repo.cleanup();
  }
});

test("buildContext skips generated/vendored files", async () => {
  const repo = await createTmpRepo();
  try {
    await repo.commitFile("src/real.ts", "const a = 1;\n", "real");
    await repo.commitFile("package-lock.json", '{"lockfileVersion": 1}\n', "lockfile");
    await repo.commitFile("dist/bundle.js", "console.log(1)\n", "built file");

    const { text } = await buildContext(repo.dir, { kind: "path", path: ".", rev: "HEAD" });
    assert.match(text, /src\/real\.ts/);
    assert.doesNotMatch(text, /package-lock\.json/);
    assert.doesNotMatch(text, /dist\/bundle\.js/);
  } finally {
    await repo.cleanup();
  }
});

test("buildContext drops the largest files first and warns when over budget", async () => {
  const repo = await createTmpRepo();
  try {
    await repo.commitFile("small.ts", "x".repeat(10), "small");
    await repo.commitFile("big.ts", "y".repeat(10_000), "big");

    const { text, warnings } = await buildContext(
      repo.dir,
      { kind: "path", path: ".", rev: "HEAD" },
      { maxChars: 200 },
    );

    assert.match(text, /small\.ts/);
    assert.doesNotMatch(text, /big\.ts/);
    assert.equal(warnings.length, 1);
    assert.match(warnings[0] ?? "", /dropped 1 file/);
  } finally {
    await repo.cleanup();
  }
});
```

## Hints

<details><summary>Hint 1</summary>

`numberLines`: split into lines, compute `width = String(lines.length).length`,
then `lines.map((line, i) => \`${String(i + 1).padStart(width)}: ${line}\`).join("\n")`.

</details>

<details><summary>Hint 2</summary>

Structure `buildContext` as two branches on `scope.kind`, each building an
array of `Section` objects (`{ path, heading, body }`), then handing off to a
shared `assembleBudgeted(required, sections, maxChars, warnings)` helper that
does the size-sorting and dropping once, used by both branches.

</details>

<details><summary>Hint 3</summary>

For "drop largest first, keep original order": sort a *copy* of the sections
biggest-to-smallest, greedily keep ones that still fit a shrinking budget
counter, then — separately — filter the *original* array down to the set of
paths you decided to keep, so the final text reads top-to-bottom like the
repo, not biggest-to-smallest.

</details>

## Solution

<details><summary>Solution</summary>

```ts
// context.ts — turns a Scope into the plain text we send the model: a
// unified diff (for ranges) plus line-numbered file contents, trimmed to a
// character budget. Still no process.env, no console: budget overruns come
// back as a `warnings: string[]` field instead of a printed message.

import * as path from "node:path";
import type { Scope } from "./scope.ts";
import { changedFiles, filesUnderPath, readFileAtRevision, unifiedDiff } from "./scope.ts";

export type ContextResult = {
  text: string;
  warnings: string[];
};

export type ContextOptions = {
  /** Character budget for the whole context blob. Characters are a cheap,
   * dependency-free stand-in for "tokens". */
  maxChars?: number;
};

const DEFAULT_MAX_CHARS = 60_000;

// Files we never send to the model: they're either huge, generated, or add
// no understanding (the model can't meaningfully reason about a minified
// bundle or a lockfile). This is a simple heuristic, not a parser — good
// enough for an MVP.
const SKIP_BASENAMES = new Set([
  "package-lock.json",
  "yarn.lock",
  "pnpm-lock.yaml",
  "composer.lock",
  "Cargo.lock",
]);

const SKIP_PATH_SEGMENTS = ["node_modules", "dist", "build", "vendor", ".git"];

function looksGenerated(filePath: string): boolean {
  const base = path.basename(filePath);
  if (SKIP_BASENAMES.has(base)) return true;
  if (/\.min\.[cm]?js$/.test(base) || base.endsWith(".min.css")) return true;

  const segments = filePath.split("/");
  return segments.some((segment) => SKIP_PATH_SEGMENTS.includes(segment));
}

/** Buffers containing a NUL byte are treated as binary and skipped. */
function looksBinary(content: Buffer): boolean {
  return content.includes(0);
}

/** Prefix every line with a right-aligned line number, like `cat -n`. */
export function numberLines(text: string): string {
  const lines = text.split("\n");
  const width = String(lines.length).length;
  return lines.map((line, i) => `${String(i + 1).padStart(width)}: ${line}`).join("\n");
}

type Section = {
  path: string;
  heading: string;
  body: string;
};

function sectionChars(section: Section): number {
  return section.heading.length + section.body.length;
}

/**
 * Build the text context for a scope. For a range scope this is the diff
 * (always kept in full) plus the full, line-numbered content of every
 * changed file (dropped largest-first if the budget is tight). For a path
 * scope it is the line-numbered content of every file under the path.
 */
export async function buildContext(
  repo: string,
  scope: Scope,
  options: ContextOptions = {},
): Promise<ContextResult> {
  const maxChars = options.maxChars ?? DEFAULT_MAX_CHARS;
  const warnings: string[] = [];

  if (scope.kind === "range") {
    const diff = await unifiedDiff(repo, scope);
    const diffSection = `## Diff (${scope.base}..${scope.head})\n\n${diff}\n`;

    const files = (await changedFiles(repo, scope)).filter((p) => !looksGenerated(p));
    const sections: Section[] = [];
    for (const filePath of files) {
      const buf = await readFileAtRevision(repo, scope.head, filePath);
      if (looksBinary(buf)) continue;
      const body = numberLines(buf.toString("utf8"));
      sections.push({
        path: filePath,
        heading: `\n## File (full content at ${scope.head}): ${filePath}\n\n`,
        body,
      });
    }

    return assembleBudgeted(diffSection, sections, maxChars, warnings);
  }

  // Path scope: no diff, just every file's full content.
  const files = (await filesUnderPath(repo, scope)).filter((p) => !looksGenerated(p));
  const sections: Section[] = [];
  for (const filePath of files) {
    const buf = await readFileAtRevision(repo, scope.rev, filePath);
    if (looksBinary(buf)) continue;
    const body = numberLines(buf.toString("utf8"));
    sections.push({
      path: filePath,
      heading: `\n## File (at ${scope.rev}): ${filePath}\n\n`,
      body,
    });
  }

  return assembleBudgeted("", sections, maxChars, warnings);
}

/**
 * Concatenate `required` (never dropped) with as many `sections` as fit in
 * `maxChars`, dropping the *largest* sections first and recording a warning
 * for each one dropped.
 */
function assembleBudgeted(
  required: string,
  sections: Section[],
  maxChars: number,
  warnings: string[],
): ContextResult {
  let budget = maxChars - required.length;
  const kept: Section[] = [];

  // Drop largest-first: sort a copy biggest-to-smallest, then walk it,
  // keeping a section only if it still fits in the remaining budget.
  const bySizeDesc = [...sections].sort((a, b) => sectionChars(b) - sectionChars(a));
  const droppedPaths: string[] = [];
  for (const section of bySizeDesc) {
    if (sectionChars(section) <= budget) {
      kept.push(section);
      budget -= sectionChars(section);
    } else {
      droppedPaths.push(section.path);
    }
  }

  if (droppedPaths.length > 0) {
    warnings.push(
      `context exceeded ${maxChars} char budget; dropped ${droppedPaths.length} file(s): ${droppedPaths.join(", ")}`,
    );
  }

  // Keep the original (not size-sorted) order for anything we kept, so the
  // text reads top-to-bottom the way the repo does.
  const keptPaths = new Set(kept.map((s) => s.path));
  const inOrder = sections.filter((s) => keptPaths.has(s.path));

  const text = required + inOrder.map((s) => s.heading + s.body).join("\n");
  return { text, warnings };
}
```

</details>

## Watch out

> **Watch out:** `path.basename` and the `looksGenerated` checks operate on
> `/`-separated paths because these come from *git*, not from
> `node:path`'s own path-joining on your OS. Git always reports paths with
> forward slashes, even on Windows, so splitting on `"/"` directly (as
> `looksGenerated` does for `SKIP_PATH_SEGMENTS`) is correct and
> OS-independent here — don't reach for `path.sep`.

> **Watch out:** `Array.prototype.sort` mutates the array it's called on.
> `assembleBudgeted` sorts `[...sections]` (a shallow copy made with the
> spread operator), never `sections` itself, specifically so the caller's
> array (and the original file order) survives intact for the final
> `inOrder` filter.

## Checkpoint

```
$ npm run typecheck
$ npm test
```

Expected: `tsc` reports no errors, 14 tests pass (8 from `scope.test.ts`, 6
new in `context.test.ts`).

```
$ git add src/core/context.ts test/context.test.ts
$ git commit -m "core: build model context from a Scope"
```

Next: [05 — Quiz schemas](05-quiz-schemas.md), where you define — with zod —
exactly what a quiz looks like, and write the prompt that asks the model for
one.
