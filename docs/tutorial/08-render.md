# 08 — Rendering

[← 07. Checking citations](07-citations.md) · [09. The `generate` command →](09-generate.md)

## Goal

Build `src/core/render.ts`: the last stage of the `generate` pipeline. It
takes a validated `Quiz` (chapter 07's output) and does two jobs that are
deliberately *our* code's responsibility, not the model's:

1. Shuffle each question's options, so the correct answer isn't
   predictable from its position.
2. Write `quiz.json`, `quiz.md` and `answers.md` to disk.

## Concepts

### Fisher–Yates shuffle

Models tend to over-use certain option positions for the correct answer
(often "c") — ask an LLM for a few dozen quiz questions and you'll see the
pattern. Shuffling in our own code removes that as a way to game the quiz
without even reading the material.

The standard unbiased way to shuffle an array in place is the
[Fisher–Yates shuffle](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle):
walk the array backwards, and for each position swap it with a random
position at or before it.

```ts
for (let i = shuffled.length - 1; i > 0; i--) {
  const j = Math.floor(random() * (i + 1)); // random index in [0, i]
  [shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
}
```

It's worth implementing by hand once: it's short, it's a genuinely common
interview/algorithms exercise in any language, and — unlike `array.sort(() =>
Math.random() - 0.5)`, a shuffle "trick" that shows up in blog posts and is
statistically biased — it's actually correct.

### Injectable randomness

Same idea as `fetchImpl` in chapter 06, applied to `Math.random` instead of
`fetch`:

```ts
export function shuffleOptions(question: Question, random: () => number = Math.random): Question
```

A test can pass `() => 0` or `() => 0.99` and get a fully deterministic
shuffle to assert against, instead of writing a flaky test that has to
tolerate randomness (or worse, mocking the *global* `Math.random`, which
leaks into every other piece of code running in the same process). Any time
you see "this function's behaviour depends on something non-deterministic
— time, randomness, the network, the filesystem's current state", a
parameter with a sensible default is usually the cleanest way to make it
testable.

### Immutability: copy before you mutate

```ts
const shuffled = [...question.options];
```

`[...array]` makes a shallow copy — a new array containing the same
elements. Fisher–Yates swaps elements of `shuffled`, never `question.options`
itself, so the caller's `Question` is untouched. The function then returns a
*new* object:

```ts
return { ...question, options: relabelled, correct: newCorrect };
```

`{ ...question, options: ..., correct: ... }` spreads every field of
`question` into a new object, then overrides two of them. This "copy, then
override" pattern is how you do "an updated version of this value" in a
language without records/`with`-expressions (C# 9+ has `with`, F#/Python's
`dataclasses.replace` are the same idea) — in plain JS objects, the spread
operator *is* that mechanism. One of the tests in this chapter exists purely
to assert the input question comes out of `shuffleOptions` byte-for-byte
unchanged.

### `noUncheckedIndexedAccess` makes array indexing honest

Chapter 00 turned this flag on in `tsconfig.json`. Its effect: `shuffled[i]`
is typed `Option | undefined`, not `Option` — even directly inside a
`for (let i = 0; i < shuffled.length; i++)` loop where you *know* `i` is in
range. TypeScript can't prove that from the loop condition alone, so it
makes you handle the `undefined` case explicitly:

```ts
const a = shuffled[i];
const b = shuffled[j];
if (a === undefined || b === undefined) continue; // unreachable, keeps the compiler honest
```

That `if` never actually triggers — but writing it is the price of a
compiler that doesn't let you *assume* an index is in bounds anywhere, which
catches real off-by-one bugs elsewhere in a codebase. If you want to see the
difference, try commenting out `"noUncheckedIndexedAccess": true` in
`tsconfig.json` for a minute: the same line type-checks as `Option` with no
guard needed, which looks more convenient and is exactly the kind of
unchecked assumption that causes a crash the one time an index actually is
out of range.

### Building Markdown with template literals and arrays

No templating library here — just an array of lines, pushed to as you go,
joined with `\n` at the end:

```ts
const lines: string[] = [`# Quiz: ${quiz.target}`, ""];
for (const q of quiz.questions) {
  lines.push(`## Q${q.id} (${q.kind})`, "", q.question, "");
}
return lines.join("\n");
```

This is a common, low-ceremony way to generate structured text in JS/TS —
readable, and each pushed item is one line, which keeps diffs in generated
output easy to read too.

### Why options are hidden in `<details>`

`docs/design.md`'s "Taking a quiz" section explains the pedagogy: the user
should answer in their own words *before* seeing the multiple-choice
options, because recall and recognition are different skills and the whole
point of Readback is testing the former. `quiz.md` is a static rendering of
that same idea for anyone reading it outside the `take` command (e.g. in a
PR description, or printed): wrapping the options in a collapsed
`<details><summary>Options</summary>...</details>` block (standard HTML,
rendered collapsed by GitHub, GitLab, and most Markdown viewers) means
opening the file doesn't immediately spoil the answer.

### `fs/promises` and `path.join`

```ts
import { mkdir, writeFile } from "node:fs/promises";
import * as path from "node:path";

await mkdir(outDir, { recursive: true });
await writeFile(path.join(outDir, "quiz.json"), text, "utf8");
```

`node:fs/promises` is the promise-based sibling of the older callback-style
`node:fs` — same idea as `node:util`'s `promisify(execFile)` from chapter 02,
but these functions already return promises natively, no wrapping needed.
`{ recursive: true }` on `mkdir` is "create parent directories too, and
don't error if the directory already exists" — the equivalent of Python's
`os.makedirs(path, exist_ok=True)` or Go's `os.MkdirAll`. `path.join` builds
a path from segments using the right separator for the OS, instead of
string-concatenating `/` by hand.

## Exercise

Create `src/core/render.ts` exporting:

```ts
export function shuffleOptions(
  question: Question,
  random?: () => number, // defaults to Math.random
): Question;

export function renderQuizMarkdown(quiz: Quiz): string;

export function renderAnswersMarkdown(quiz: Quiz): string;

export async function writeQuizFiles(
  quiz: Quiz,
  outDir: string,
): Promise<{ quizPath: string; quizMdPath: string; answersMdPath: string }>;
```

Required behaviour:

- `shuffleOptions`: Fisher–Yates shuffle `question.options` using `random`,
  relabel the shuffled options' `id`s back to `"a" | "b" | "c" | "d"` in
  their new order, and update `correct` to still point at whichever option
  was correct before shuffling (matched by its *old* id, since the identity
  of "the right answer" doesn't change — only its position and label do).
  Must not mutate the input `question` or its `options` array.
- `renderQuizMarkdown`: a `# Quiz: <target>` heading, the repo and commit,
  then for every question a `## Q<id> (<kind>)` heading, the question text,
  and its options inside a collapsed `<details>` block. Must **not** include
  any option's `why` text (that belongs only in the answers file).
- `renderAnswersMarkdown`: for every question, a heading naming the correct
  option, then every option marked ✅/❌ with its `why`, then its citations
  as `` `path:start-end` ``.
- `writeQuizFiles`: create `outDir` (and any missing parent directories),
  write `quiz.json` (the quiz, pretty-printed, trailing newline),
  `quiz.md` and `answers.md` into it, and return the three paths.

## Tests

Create `test/render.test.ts`:

```ts
// render.test.ts — option shuffling with an injected random function (for a
// deterministic test) and the Markdown/JSON file writers.

import { test } from "node:test";
import assert from "node:assert/strict";
import { mkdtemp, readFile, rm } from "node:fs/promises";
import * as os from "node:os";
import * as path from "node:path";
import { shuffleOptions, renderQuizMarkdown, renderAnswersMarkdown, writeQuizFiles } from "../src/core/render.ts";
import type { Question, Quiz } from "../src/core/quiz.ts";

function makeQuestion(): Question {
  return {
    id: 1,
    kind: "why",
    question: "Why?",
    options: [
      { id: "a", text: "A", why: "right" },
      { id: "b", text: "B", why: "wrong" },
      { id: "c", text: "C", why: "wrong" },
      { id: "d", text: "D", why: "wrong" },
    ],
    correct: "a",
    citations: [{ path: "a.ts", start: 1, end: 1 }],
  };
}

test("shuffleOptions keeps the same 4 texts and correct option's text", () => {
  const question = makeQuestion();
  // A "random" function that always returns 0 still exercises the swap loop
  // deterministically without needing a real RNG.
  const shuffled = shuffleOptions(question, () => 0);

  assert.equal(shuffled.options.length, 4);
  assert.deepEqual(
    shuffled.options.map((o) => o.text).sort(),
    ["A", "B", "C", "D"],
  );

  const correctOption = shuffled.options.find((o) => o.id === shuffled.correct);
  assert.equal(correctOption?.text, "A"); // "A" was the originally-correct option's text
});

test("shuffleOptions relabels ids a-d regardless of new order", () => {
  const shuffled = shuffleOptions(makeQuestion(), () => 0.99);
  assert.deepEqual(
    shuffled.options.map((o) => o.id),
    ["a", "b", "c", "d"],
  );
});

test("shuffleOptions does not mutate the input question", () => {
  const question = makeQuestion();
  const before = JSON.stringify(question);
  shuffleOptions(question, () => 0.5);
  assert.equal(JSON.stringify(question), before);
});

test("renderQuizMarkdown hides options inside <details> and omits answers", () => {
  const quiz: Quiz = {
    version: 1,
    createdAt: "2024-01-01T00:00:00.000Z",
    repo: "/repo",
    target: "main...feature",
    commit: "abc123",
    model: "test-model",
    questions: [makeQuestion()],
  };
  const md = renderQuizMarkdown(quiz);
  assert.match(md, /<details>/);
  assert.match(md, /Why\?/);
  assert.doesNotMatch(md, /why.*wrong/is); // the `why` explanations belong in answers.md, not quiz.md
});

test("renderAnswersMarkdown marks the correct option and lists citations", () => {
  const quiz: Quiz = {
    version: 1,
    createdAt: "2024-01-01T00:00:00.000Z",
    repo: "/repo",
    target: "main...feature",
    commit: "abc123",
    model: "test-model",
    questions: [makeQuestion()],
  };
  const md = renderAnswersMarkdown(quiz);
  assert.match(md, /✅ \*\*a\.\*\*/);
  assert.match(md, /a\.ts:1-1/);
});

test("writeQuizFiles writes quiz.json, quiz.md and answers.md", async () => {
  const dir = await mkdtemp(path.join(os.tmpdir(), "readback-render-test-"));
  try {
    const quiz: Quiz = {
      version: 1,
      createdAt: "2024-01-01T00:00:00.000Z",
      repo: "/repo",
      target: "path:src/",
      commit: "abc123",
      model: "test-model",
      questions: [makeQuestion()],
    };

    const written = await writeQuizFiles(quiz, dir);
    const parsedBack = JSON.parse(await readFile(written.quizPath, "utf8"));
    assert.equal(parsedBack.commit, "abc123");

    const quizMd = await readFile(written.quizMdPath, "utf8");
    assert.match(quizMd, /Why\?/);

    const answersMd = await readFile(written.answersMdPath, "utf8");
    assert.match(answersMd, /correct: a/);
  } finally {
    await rm(dir, { recursive: true, force: true });
  }
});
```

## Hints

<details><summary>Hint 1</summary>

Keep `shuffleOptions` to three steps: (1) copy and Fisher–Yates-shuffle
`question.options` into a local array, (2) `.map` that array into relabelled
options while tracking which new id ended up holding the originally-correct
option, (3) return a new object built with `{ ...question, options: ..., correct: ... }`.

</details>

<details><summary>Hint 2</summary>

A fixed array of the four ids is enough for relabelling in order:

```ts
const OPTION_IDS: OptionId[] = ["a", "b", "c", "d"];
const relabelled = shuffled.map((option, i) => ({ ...option, id: OPTION_IDS[i]! }));
```

(or handle the `undefined` from `noUncheckedIndexedAccess` with a real
check, as the solution below does, rather than the non-null assertion `!` —
either compiles; the explicit check is the more honest option given the
point of the flag.)

</details>

<details><summary>Hint 3</summary>

To find which *new* id the previously-correct option ended up with, compare
by the option's *old* id while building `relabelled`, not by its text or
object identity:

```ts
if (option.id === previousCorrectId) newCorrect = newId;
```

</details>

<details><summary>Hint 4</summary>

`writeQuizFiles` is three independent `writeFile` calls after one `mkdir` —
no need for `Promise.all` here, but it wouldn't hurt either since the three
writes don't depend on each other (unlike chapter 07's citation cache).

</details>

## Solution

<details><summary>Solution</summary>

```ts
// render.ts — our code (deliberately, not the model) shuffles each
// question's options, then writes quiz.json plus two generated Markdown
// views. File writes return promises / throw on failure, same as every
// other core function — still no console.log here.

import { mkdir, writeFile } from "node:fs/promises";
import * as path from "node:path";
import type { Question, Quiz } from "./quiz.ts";

type OptionId = "a" | "b" | "c" | "d";
const OPTION_IDS: OptionId[] = ["a", "b", "c", "d"];

/**
 * Shuffle one question's options and relabel them a-d, updating `correct`
 * to still point at the right one. Models tend to over-use certain
 * positions (often "c") for the correct answer, so always shuffling removes
 * that pattern as a way to game the quiz.
 *
 * `random` defaults to Math.random but can be injected — pass a seeded
 * generator in a test to get a deterministic shuffle.
 */
export function shuffleOptions(question: Question, random: () => number = Math.random): Question {
  const previousCorrectId = question.correct;

  // Fisher-Yates shuffle: walk the array backwards, swapping each element
  // with a random earlier (or equal) one. Copy first so we never mutate the
  // caller's array.
  const shuffled = [...question.options];
  for (let i = shuffled.length - 1; i > 0; i--) {
    const j = Math.floor(random() * (i + 1));
    const a = shuffled[i];
    const b = shuffled[j];
    if (a === undefined || b === undefined) continue; // unreachable, keeps noUncheckedIndexedAccess happy
    shuffled[i] = b;
    shuffled[j] = a;
  }

  let newCorrect: OptionId = previousCorrectId;
  const relabelled = shuffled.map((option, i) => {
    const newId = OPTION_IDS[i];
    if (newId === undefined) throw new Error("unreachable: exactly 4 options expected");
    if (option.id === previousCorrectId) newCorrect = newId;
    return { ...option, id: newId };
  });

  return { ...question, options: relabelled, correct: newCorrect };
}

/** Render the quiz's questions as Markdown, options hidden in `<details>`. */
export function renderQuizMarkdown(quiz: Quiz): string {
  const lines: string[] = [`# Quiz: ${quiz.target}`, "", `Repo: \`${quiz.repo}\` at \`${quiz.commit}\``, ""];

  for (const q of quiz.questions) {
    lines.push(`## Q${q.id} (${q.kind})`, "", q.question, "", "<details><summary>Options</summary>", "");
    for (const option of q.options) {
      lines.push(`- **${option.id}.** ${option.text}`);
    }
    lines.push("", "</details>", "");
  }

  return lines.join("\n");
}

/** Render the answer key: correct option, every option's `why`, citations. */
export function renderAnswersMarkdown(quiz: Quiz): string {
  const lines: string[] = [`# Answers: ${quiz.target}`, ""];

  for (const q of quiz.questions) {
    lines.push(`## Q${q.id} — correct: ${q.correct}`, "");
    for (const option of q.options) {
      const marker = option.id === q.correct ? "✅" : "❌";
      lines.push(`- ${marker} **${option.id}.** ${option.text} — ${option.why}`);
    }
    lines.push("", "Citations:");
    for (const c of q.citations) {
      lines.push(`- \`${c.path}:${c.start}-${c.end}\``);
    }
    lines.push("");
  }

  return lines.join("\n");
}

/**
 * Write quiz.json, quiz.md and answers.md into `outDir` (created if
 * missing). Returns the three file paths written.
 */
export async function writeQuizFiles(quiz: Quiz, outDir: string): Promise<{ quizPath: string; quizMdPath: string; answersMdPath: string }> {
  await mkdir(outDir, { recursive: true });

  const quizPath = path.join(outDir, "quiz.json");
  const quizMdPath = path.join(outDir, "quiz.md");
  const answersMdPath = path.join(outDir, "answers.md");

  await writeFile(quizPath, `${JSON.stringify(quiz, null, 2)}\n`, "utf8");
  await writeFile(quizMdPath, renderQuizMarkdown(quiz), "utf8");
  await writeFile(answersMdPath, renderAnswersMarkdown(quiz), "utf8");

  return { quizPath, quizMdPath, answersMdPath };
}
```

</details>

## Watch out

> **Watch out:** `noUncheckedIndexedAccess` makes `shuffled[i]` type as
> `Option | undefined`, even right after a loop condition that guarantees
> `i` is in range. The `if (a === undefined || b === undefined) continue;`
> guard in the shuffle loop is genuinely unreachable at runtime — it exists
> purely to satisfy the compiler. Try toggling the flag off in
> `tsconfig.json` for a minute to see the same line type-check as `Option`
> with no guard, and judge for yourself which version you trust more the
> next time you touch this function.

> **Watch out:** `shuffleOptions` matches the previously-correct option by
> its **old `id`**, not by text or array position — options are plain data,
> so two different options could coincidentally have identical `text`, and
> the array position is exactly what's being shuffled. Matching on anything
> but the stable identifier would silently point `correct` at the wrong
> option some fraction of the time.

## Checkpoint

```
$ npx tsc
$ node --test test/render.test.ts
```

Expected: no type errors, and a summary like:

```
ℹ tests 6
ℹ pass 6
ℹ fail 0
```

Then run the whole suite to make sure nothing earlier broke:

```
$ npm test
```

Expected: `ℹ tests 34`, `ℹ pass 34`, `ℹ fail 0`.

Commit:

```
$ git add src/core/render.ts test/render.test.ts
$ git commit -m "feat: shuffle options and render quiz output files"
```

Next: [09. The `generate` command](09-generate.md) — wire `scope`, `context`,
`llm`, `quiz` and `render` together behind a real CLI command.
