# 07 — Checking citations

[← 06. Calling the LLM](06-llm.md) · [08. Rendering →](08-render.md)

## Goal

Finish `src/core/quiz.ts` by adding `validateCitations` and `assignIds`. This
is the part of Readback that makes the whole project trustworthy: it's the
code that checks the model didn't make something up.

## Concepts

### Why a hallucinated citation is the core risk

Every design decision in `docs/design.md` traces back to one sentence:
*"every question cites `file:line-range` so the answer can be verified
against source, not trusted blindly."* An LLM can write a perfectly
plausible-sounding question and point it at a citation that doesn't exist,
or exists but says something else. If Readback shipped that straight to the
user, it would be teaching wrong things with the appearance of authority —
worse than teaching nothing. `validateCitations` is the one place in the
codebase that closes that gap: it re-reads the actual file at the actual
revision and checks the cited lines are real, *before* a question is ever
shown to anyone.

### `Map` as a cache, with `null` meaning "checked, and it's not there"

Several questions can cite the same file. Re-running `git show` for every
citation would be wasteful, so `validateCitations` keeps a cache:

```ts
const lineCounts = new Map<string, number | null>();
```

The value type is `number | null`, not just `number`, because there are
*three* states to distinguish, not two:

1. key absent from the map → "haven't checked this path yet"
2. key present, value a `number` → "checked, file exists, this many lines"
3. key present, value `null` → "checked, file does **not** exist at this revision"

Without the `null` state you couldn't tell state 1 from state 3 using
`.has()` alone, and you'd end up re-doing the (failing) file read every
time. Python's dict or a Go `map[string]*int` would need the same kind of
sentinel; TS just makes you spell out the union.

### Sequential `await` in a `for...of` loop

```ts
for (const citation of question.citations) {
  const count = await lineCountOf(citation.path);
  // ...
}
```

Each iteration `await`s before starting the next one — the loop runs
strictly in order, one citation at a time, not concurrently. That's a
deliberate choice here, not an oversight. `Promise.all` would run every
`lineCountOf` call at once, which is faster when the work is independent —
but here it isn't: the whole point of `lineCountOf` is to let the second
citation to the same file skip the actual `git show` by hitting the cache
that the first citation just populated. Running them concurrently would
make most of the cache hits race the write that was supposed to produce
them, which means every citation would do real I/O instead of most of them
being free. When you reach for `Promise.all` elsewhere in this project
(there isn't a case later in this tutorial, but it's a general rule).
sequential `await` in a loop is the right default whenever later iterations
can benefit from what earlier ones did; `Promise.all` is for genuinely
independent work you just want to run faster.

### Returning a result instead of throwing

```ts
export type ValidationResult = {
  valid: LlmQuestion[];
  dropped: { question: LlmQuestion; reason: string }[];
};
```

A question with a bad citation isn't an exceptional situation — it's an
expected outcome of asking a model to write JSON, and the correct response
is "drop it and keep going", not "abort the whole quiz". That's a good
signal for *not* throwing: `validateCitations` always succeeds (barring a
genuine I/O error) and reports what it found in its return value, the same
way `context.ts`'s `ContextResult.warnings` did back in chapter 04. Compare
this to `QuizParseError` from chapter 05, which *does* throw — a reply that
isn't even valid JSON matching the schema is not something the caller can
partially recover from, so an exception is the right tool there. The rule
of thumb: throw for "the caller cannot proceed", return structured data for
"the caller should decide what to do with some good and some bad".

### The untrusted/trusted type boundary

Chapter 05 introduced two types for a question:

- `LlmQuestion` — exactly what the model returned, no `id` yet, not yet
  checked against the real repo. *Untrusted.*
- `Question` — has an `id`, and (after this chapter) has survived citation
  validation. *Trusted* — this is the shape written to `quiz.json` and
  handed to `render.ts` and `take.ts`.

`assignIds` is the single function that turns one into the other:

```ts
export function assignIds(questions: LlmQuestion[]): Question[] {
  return questions.map((q, i) => ({ ...q, id: i + 1 }));
}
```

It only runs on the `valid` array `validateCitations` returned — questions
that never crossed the trust boundary never get an id, and (per the design)
never reach the user. Keeping "data that came from outside" and "data our
own code has vetted" as two distinct TypeScript types, rather than one type
with an optional `id`, means the compiler itself stops you from
accidentally rendering an unvalidated question: `renderQuizMarkdown` (next
chapter) takes a `Quiz`, which is built only from `Question[]`, so there is
no code path that skips validation by accident.

## Exercise

Add to the existing `src/core/quiz.ts` from chapter 05:

1. Restore the import of the git-reading functions you'll need:

   ```ts
   import { changedFiles, readFileAtRevision } from "./scope.ts";
   ```

   (alongside the `import type { Scope } from "./scope.ts";` you already have).

2. Export:

   ```ts
   export type ValidationResult = {
     valid: LlmQuestion[];
     dropped: { question: LlmQuestion; reason: string }[];
   };

   export async function validateCitations(
     repo: string,
     scope: Scope,
     questions: LlmQuestion[],
   ): Promise<ValidationResult>;

   export function assignIds(questions: LlmQuestion[]): Question[];
   ```

Required behaviour of `validateCitations`:

- For a `range` scope, compute the set of changed files with `changedFiles`
  (chapter 02/03) once, up front.
- The revision to read files at is `scope.head` for a `range` scope, or
  `scope.rev` for a `path` scope.
- For each question, check every one of its citations:
  - The citation's `path` must exist at that revision (read it with
    `readFileAtRevision`, catch failure, treat as "doesn't exist"). Cache
    line counts per path so repeated citations to the same file don't
    re-read it.
  - `1 <= start <= end <= <line count>` must hold.
  - For `range` scopes only: **at least one** of the question's citations
    must point at a file that's in the changed-files set. (`path` scopes
    have nothing to compare against — treat that check as automatically
    satisfied.)
- A question that passes every check goes into `valid`. A question that
  fails any check goes into `dropped`, paired with a human-readable string
  explaining which check failed.
- Return `{ valid, dropped }`.

Required behaviour of `assignIds`: number the given questions `1, 2, 3, ...`
in order, copying every other field unchanged.

## Tests

Replace `test/quiz.test.ts` with the full file:

```ts
// quiz.test.ts — schema parsing and citation validation, including the
// "invalid citations get dropped" case called out in the task spec.

import { test } from "node:test";
import assert from "node:assert/strict";
import { createTmpRepo } from "./helpers.ts";
import { parseLlmQuiz, QuizParseError, validateCitations, assignIds, buildPrompt } from "../src/core/quiz.ts";
import type { LlmQuestion } from "../src/core/quiz.ts";

function makeQuestion(overrides: Partial<LlmQuestion> = {}): LlmQuestion {
  return {
    kind: "why",
    question: "Why does this check exist?",
    options: [
      { id: "a", text: "Because X", why: "Correct: X is true" },
      { id: "b", text: "Because Y", why: "Wrong: Y never happens here" },
      { id: "c", text: "Because Z", why: "Wrong: unrelated" },
      { id: "d", text: "No reason", why: "Wrong: there is a reason" },
    ],
    correct: "a",
    citations: [{ path: "a.ts", start: 1, end: 1 }],
    ...overrides,
  };
}

test("buildPrompt mentions the scope and asks for JSON", () => {
  const messages = buildPrompt({ kind: "range", base: "x", head: "y" }, "the context", 5);
  assert.equal(messages.length, 2);
  assert.equal(messages[0]?.role, "system");
  assert.match(messages[0]?.content ?? "", /JSON only/);
  assert.match(messages[1]?.content ?? "", /5 questions/);
  assert.match(messages[1]?.content ?? "", /commit x and commit y/);
});

test("parseLlmQuiz accepts a well-formed reply", () => {
  const raw = JSON.stringify({ questions: [makeQuestion()] });
  const quiz = parseLlmQuiz(raw);
  assert.equal(quiz.questions.length, 1);
});

test("parseLlmQuiz rejects non-JSON", () => {
  assert.throws(() => parseLlmQuiz("not json"), QuizParseError);
});

test("parseLlmQuiz rejects JSON that doesn't match the schema", () => {
  assert.throws(() => parseLlmQuiz(JSON.stringify({ questions: "oops" })), QuizParseError);
});

test("validateCitations keeps a question whose citation checks out", async () => {
  const repo = await createTmpRepo();
  try {
    const base = await repo.commitFile("a.ts", "line one\nline two\n", "base");
    const head = await repo.commitFile("a.ts", "line one\nline two changed\n", "change");
    const scope = { kind: "range" as const, base, head };

    const question = makeQuestion({ citations: [{ path: "a.ts", start: 2, end: 2 }] });
    const { valid, dropped } = await validateCitations(repo.dir, scope, [question]);

    assert.equal(valid.length, 1);
    assert.equal(dropped.length, 0);
  } finally {
    await repo.cleanup();
  }
});

test("validateCitations drops a question whose citation path does not exist", async () => {
  const repo = await createTmpRepo();
  try {
    const sha = await repo.commitFile("a.ts", "line one\n", "base");
    const scope = { kind: "path" as const, path: ".", rev: sha };

    const question = makeQuestion({ citations: [{ path: "no-such-file.ts", start: 1, end: 1 }] });
    const { valid, dropped } = await validateCitations(repo.dir, scope, [question]);

    assert.equal(valid.length, 0);
    assert.equal(dropped.length, 1);
    assert.match(dropped[0]?.reason ?? "", /does not exist/);
  } finally {
    await repo.cleanup();
  }
});

test("validateCitations drops a question with an out-of-range line citation", async () => {
  const repo = await createTmpRepo();
  try {
    const sha = await repo.commitFile("a.ts", "only one line\n", "base");
    const scope = { kind: "path" as const, path: ".", rev: sha };

    // File has 2 "lines" once split on \n (trailing empty string), but 99 is
    // still well out of range.
    const question = makeQuestion({ citations: [{ path: "a.ts", start: 1, end: 99 }] });
    const { valid, dropped } = await validateCitations(repo.dir, scope, [question]);

    assert.equal(valid.length, 0);
    assert.equal(dropped.length, 1);
    assert.match(dropped[0]?.reason ?? "", /out of range/);
  } finally {
    await repo.cleanup();
  }
});

test("validateCitations drops a range-scope question that cites only unchanged files", async () => {
  const repo = await createTmpRepo();
  try {
    await repo.commitFile("untouched.ts", "line one\n", "base");
    const base = await repo.commitFile("changed.ts", "before\n", "base2");
    const head = await repo.commitFile("changed.ts", "after\n", "change");
    const scope = { kind: "range" as const, base, head };

    // Cites a real file with a valid line range, but that file never changed
    // in this range — so it fails the "touches a changed file" rule.
    const question = makeQuestion({ citations: [{ path: "untouched.ts", start: 1, end: 1 }] });
    const { valid, dropped } = await validateCitations(repo.dir, scope, [question]);

    assert.equal(valid.length, 0);
    assert.equal(dropped.length, 1);
    assert.match(dropped[0]?.reason ?? "", /no citation touches a file that changed/);
  } finally {
    await repo.cleanup();
  }
});

test("validateCitations keeps good questions and drops only the bad one, mixed", async () => {
  const repo = await createTmpRepo();
  try {
    const base = await repo.commitFile("a.ts", "before\n", "base");
    const head = await repo.commitFile("a.ts", "after\n", "change");
    const scope = { kind: "range" as const, base, head };

    const good = makeQuestion({ question: "good", citations: [{ path: "a.ts", start: 1, end: 1 }] });
    const bad = makeQuestion({ question: "bad", citations: [{ path: "missing.ts", start: 1, end: 1 }] });

    const { valid, dropped } = await validateCitations(repo.dir, scope, [good, bad]);
    assert.equal(valid.length, 1);
    assert.equal(valid[0]?.question, "good");
    assert.equal(dropped.length, 1);
    assert.equal(dropped[0]?.question.question, "bad");
  } finally {
    await repo.cleanup();
  }
});

test("assignIds numbers questions sequentially from 1", () => {
  const questions = assignIds([makeQuestion(), makeQuestion()]);
  assert.deepEqual(questions.map((q) => q.id), [1, 2]);
});
```

## Hints

<details><summary>Hint 1</summary>

Build the "changed files" set once, before the loop over questions, not once
per question:

```ts
const changed = scope.kind === "range" ? new Set(await changedFiles(repo, scope)) : null;
const rev = scope.kind === "range" ? scope.head : scope.rev;
```

`changed` being `null` for path scopes is what lets you skip the
"touches a changed file" check for them later with a short-circuit like
`changed === null`.

</details>

<details><summary>Hint 2</summary>

The cache-lookup helper only needs to handle three cases — "seen and a
number", "seen and null", "not seen" — and the first two collapse into one
`.has()` check:

```ts
async function lineCountOf(path: string): Promise<number | null> {
  if (lineCounts.has(path)) return lineCounts.get(path) ?? null;
  // ... compute, lineCounts.set(path, result), return result
}
```

</details>

<details><summary>Hint 3</summary>

Inside the per-question loop, track two running values as you check each
citation: a `reason: string | null` (set and `break` the citations loop the
moment something fails) and a `touchesChanged: boolean` that starts `true`
for path scopes (`changed === null`) and `false` otherwise, flipped to
`true` the moment any citation's path is in `changed`.

</details>

<details><summary>Hint 4</summary>

Counting lines: `buf.toString("utf8").split("\n").length`. A trailing
newline in the file means this counts one more "line" than you might expect
— that's fine, it's consistent, and the tests are written against this exact
behaviour.

</details>

## Solution

<details><summary>Solution</summary>

```ts
// quiz.ts — the zod schemas that define what a quiz looks like, the prompt
// we send the model, and validation that keeps the model honest:
// every citation must point at a real file and real lines.

import { z } from "zod";
import type { Scope } from "./scope.ts";
import { changedFiles, readFileAtRevision } from "./scope.ts";

// ---------------------------------------------------------------------------
// Schemas
// ---------------------------------------------------------------------------

const optionId = z.enum(["a", "b", "c", "d"]);

// What the *model* returns for one option. We don't trust it to assign
// unique, well-ordered ids, so llmQuestionSchema keeps `id` loose; render.ts
// is what finally reassigns "a".."d" after shuffling.
const llmOptionSchema = z.object({
  id: optionId,
  text: z.string().min(1),
  why: z.string().min(1),
});

const citationSchema = z.object({
  path: z.string().min(1),
  start: z.int().min(1),
  end: z.int().min(1),
});

const questionKind = z.enum(["failure", "why", "predict", "bug"]);

// What we ask the model to return for one question. `id` is intentionally
// absent here — we number questions ourselves once we know how many survive
// citation validation (see assignIds below).
const llmQuestionSchema = z.object({
  kind: questionKind,
  question: z.string().min(1),
  options: z.array(llmOptionSchema).length(4),
  correct: optionId,
  citations: z.array(citationSchema).min(1),
});

// The whole chat completion response body we ask the model to produce.
export const llmQuizSchema = z.object({
  questions: z.array(llmQuestionSchema).min(1),
});

// Inferred TypeScript types: `z.infer` reads the *runtime* schema and
// produces the matching compile-time type, so the two can never drift apart.
export type LlmQuestion = z.infer<typeof llmQuestionSchema>;
export type LlmQuiz = z.infer<typeof llmQuizSchema>;

// A question once it has a final id and has passed citation validation.
// This is the shape documented in docs/design.md, and the shape we write to
// quiz.json.
export type Question = {
  id: number;
  kind: z.infer<typeof questionKind>;
  question: string;
  options: { id: "a" | "b" | "c" | "d"; text: string; why: string }[];
  correct: "a" | "b" | "c" | "d";
  citations: { path: string; start: number; end: number }[];
};

export type Quiz = {
  version: 1;
  createdAt: string;
  repo: string;
  target: string;
  commit: string;
  model: string;
  questions: Question[];
};

// ---------------------------------------------------------------------------
// Prompt
// ---------------------------------------------------------------------------

export type ChatMessage = { role: "system" | "user"; content: string };

/** Build the two chat messages we send the model: instructions + context. */
export function buildPrompt(scope: Scope, context: string, questionCount: number): ChatMessage[] {
  const scopeDescription =
    scope.kind === "range"
      ? `the changes between commit ${scope.base} and commit ${scope.head}`
      : `the files under "${scope.path}" at revision ${scope.rev}`;

  const system = [
    "You write multiple-choice comprehension quizzes about source code changes.",
    "The goal is to test understanding, not trivia: every question should require",
    "actually reading and reasoning about the code, not just pattern matching.",
    "",
    "Question kinds:",
    "- failure: what happens if X fails, throws, or is empty/missing?",
    "- why: why is this line, branch, or check needed?",
    "- predict: what does this code produce for a given input?",
    "- bug: what is wrong with, or risky about, this code?",
    "",
    "Rules:",
    "- Every question has exactly 4 options, ids a-d, exactly one correct.",
    "- Every wrong option must be a plausible misreading of the code, never a joke.",
    "- Every option has a `why` explaining why it is right or wrong.",
    "- Every question has at least one citation: a real file path from the",
    "  context (as shown in its heading) and a start/end line range that",
    "  actually appears there (line numbers are shown in the context).",
    "- Reply with JSON only, matching the given schema. No prose outside JSON.",
  ].join("\n");

  const user = [
    `Write ${questionCount} questions about ${scopeDescription}.`,
    "",
    "Context (line numbers on the left are NOT part of the source code):",
    "",
    context,
  ].join("\n");

  return [
    { role: "system", content: system },
    { role: "user", content: user },
  ];
}

// ---------------------------------------------------------------------------
// Parsing + validation
// ---------------------------------------------------------------------------

/** Thrown when the model's reply isn't even valid JSON matching the schema. */
export class QuizParseError extends Error {}

/** Parse the model's raw reply text into an LlmQuiz, or throw. */
export function parseLlmQuiz(raw: string): LlmQuiz {
  let json: unknown;
  try {
    json = JSON.parse(raw);
  } catch (err: unknown) {
    const message = err instanceof Error ? err.message : String(err);
    throw new QuizParseError(`model reply was not valid JSON: ${message}`);
  }

  const result = llmQuizSchema.safeParse(json);
  if (!result.success) {
    throw new QuizParseError(`model reply did not match the quiz schema: ${result.error.message}`);
  }
  return result.data;
}

export type ValidationResult = {
  valid: LlmQuestion[];
  dropped: { question: LlmQuestion; reason: string }[];
};

/**
 * Check every citation against the real repository: the path must exist in
 * the scoped revision, 1 <= start <= end <= line count, and (for range
 * scopes) at least one citation must land inside a file that actually
 * changed. Questions that fail any check are dropped rather than shown with
 * a citation that doesn't check out.
 */
export async function validateCitations(
  repo: string,
  scope: Scope,
  questions: LlmQuestion[],
): Promise<ValidationResult> {
  // For range scopes we need the set of files that changed, so "at least
  // one citation touches a changed file" can be checked.
  const changed = scope.kind === "range" ? new Set(await changedFiles(repo, scope)) : null;
  const rev = scope.kind === "range" ? scope.head : scope.rev;

  // Cache line counts per path: several questions likely cite the same file.
  const lineCounts = new Map<string, number | null>();
  async function lineCountOf(path: string): Promise<number | null> {
    if (lineCounts.has(path)) return lineCounts.get(path) ?? null;
    let count: number | null;
    try {
      const buf = await readFileAtRevision(repo, rev, path);
      count = buf.toString("utf8").split("\n").length;
    } catch {
      count = null; // path does not exist at this revision
    }
    lineCounts.set(path, count);
    return count;
  }

  const valid: LlmQuestion[] = [];
  const dropped: { question: LlmQuestion; reason: string }[] = [];

  for (const question of questions) {
    let reason: string | null = null;
    let touchesChanged = changed === null; // path scopes have nothing to check here

    for (const citation of question.citations) {
      const count = await lineCountOf(citation.path);
      if (count === null) {
        reason = `citation path "${citation.path}" does not exist at ${rev}`;
        break;
      }
      if (!(1 <= citation.start && citation.start <= citation.end && citation.end <= count)) {
        reason = `citation ${citation.path}:${citation.start}-${citation.end} is out of range (file has ${count} lines)`;
        break;
      }
      if (changed?.has(citation.path)) {
        touchesChanged = true;
      }
    }

    if (reason === null && !touchesChanged) {
      reason = "no citation touches a file that changed in this range";
    }

    if (reason === null) {
      valid.push(question);
    } else {
      dropped.push({ question, reason });
    }
  }

  return { valid, dropped };
}

/** Assign final sequential ids (1, 2, 3, ...) once we know the survivors. */
export function assignIds(questions: LlmQuestion[]): Question[] {
  return questions.map((q, i) => ({ ...q, id: i + 1 }));
}
```

</details>

## Watch out

> **Watch out:** for a `range` scope, `rev` is set to `scope.head` — but
> remember from chapter 02/03 that for a three-dot target (`main...feature`)
> this reference's `parseTarget` resolves `base` to a merge-base SHA while
> leaving `head` as whatever ref the user typed (e.g. `"feature"`, not a
> SHA). That asymmetry is intentional, not a bug — `readFileAtRevision` and
> `git` itself are happy to take a branch name as a revision — but it's worth
> remembering if you ever find yourself wanting to log or compare `base` and
> `head` as if they were the same kind of value.

> **Watch out:** `validateCitations` always resolves successfully (it never
> throws for a "bad" question) — the only way it throws is a genuine
> unexpected I/O error from `changedFiles`/`readFileAtRevision` that isn't a
> plain "file doesn't exist". Dropping is the normal path for a hallucinated
> citation, not an edge case.

## Checkpoint

```
$ npx tsc
$ node --test test/quiz.test.ts
```

Expected: no type errors, and a summary like:

```
ℹ tests 10
ℹ pass 10
ℹ fail 0
```

Then run the whole suite to make sure nothing earlier broke:

```
$ npm test
```

Expected: `ℹ tests 28`, `ℹ pass 28`, `ℹ fail 0`.

Commit:

```
$ git add src/core/quiz.ts test/quiz.test.ts
$ git commit -m "feat: validate citations and assign question ids"
```

Next: [08. Rendering](08-render.md) — turn a validated `Quiz` into
`quiz.json`, `quiz.md` and `answers.md`.
