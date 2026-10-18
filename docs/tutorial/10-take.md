# 10 — The `take` command

[← 09 The `generate` command](09-generate.md) · [README](README.md) · [11 What next →](11-what-next.md)

## Goal

`generate` is done; now build the other half: `readback take <quiz.json>`,
the interactive self-grading flow from [`../design.md`](../design.md). No
network calls at all — everything it needs is already in `quiz.json`.

Per the design, taking a quiz is four steps *per question*, deliberately kept
separate:

1. **Question only.** The user types an answer in their own words — this is
   **recall**: committing to an answer before anything primes them.
2. **Options revealed.** The user picks a–d. Auto-graded against `correct`.
   This is **recognition** — easier than recall, which is exactly why step 1
   has to come first. Someone who can recognise the right-looking option
   under step 2 but couldn't produce anything under step 1 has shallower
   understanding than the auto-grade alone would suggest.
3. **Answer revealed.** The correct option, every option's `why`, and the
   `file:line` citations to go verify against the real source.
4. **Self-grade.** The user compares their own step-1 answer against the
   step-3 explanation: *got it / partly / missed*. This is the signal
   `generate`'s multiple-choice grading can't give you — a correct pick with
   a "missed" self-grade is the whole point of this tool.

You'll write this as `src/core/take.ts` (still obeying the "no
`console`/`process` in `core/`" rule from before — see below), then wire it
into `cli.ts`.

## Concepts

### User-defined type guards

The model picks `"a"`, `"b"`, `"c"`, or `"d"` and we type that as
`OptionId = "a" | "b" | "c" | "d"`. But whatever the user types at a prompt
is just a `string` — TypeScript has no way to know it's one of those four
values. A **type guard** is a function whose return type tells the compiler
how to narrow:

```ts
function isOptionId(value: string): value is OptionId {
  return (["a", "b", "c", "d"] as string[]).includes(value);
}

const raw: string = await ask();
if (isOptionId(raw)) {
  // raw is OptionId here, not string
}
```

The `value is OptionId` return type (instead of plain `boolean`) is what
makes this different from a normal predicate. Go and C# don't have an
equivalent built into the type system this way — the closest comparisons are
a runtime check plus a cast you write by hand, except here the compiler
checks your guard actually corresponds to the narrowing you claim.

### A first generic: `askUntilValid<T extends string>`

You need the exact same "keep asking until it parses" loop for option picks
(`OptionId`) and self-grades (`SelfGrade`) — two different literal unions.
Instead of writing it twice, write it once, generic over which one:

```ts
async function askUntilValid<T extends string>(
  io: TakeIO,
  prompt: string,
  isValid: (value: string) => value is T,
): Promise<T> {
  for (;;) {
    const raw = (await io.ask(prompt)).trim().toLowerCase();
    if (isValid(raw)) return raw;
    await io.write(`"${raw}" is not a valid answer, try again.`);
  }
}
```

`<T extends string>` means "`T` can be any type, as long as it's a `string`
(or a union of string literals)". Calling `askUntilValid(io, prompt,
isOptionId)` lets TypeScript infer `T = OptionId` purely from the type guard
you passed — no need to write `askUntilValid<OptionId>(...)` by hand. This is
the same idea as Go generics (`func AskUntilValid[T ~string](...)`) or C#
generics with a constraint (`where T : struct`), just with TypeScript's
structural union types instead of named enums.

`for (;;) { ... }` is this codebase's idiom for "infinite loop, `return`s
from inside" — equivalent to `while (true)`, pure style preference.

### `TakeIO`: injecting *two* things through one object

`core/take.ts` can't call `console.log` or `readline` directly — same rule as
every other `core/*` file. But unlike `llm.ts` (inject `fetch`) or
`render.ts` (inject `random`), here there are two different things to
inject: *output* and *input*. Rather than two separate parameters, one small
object does both jobs:

```ts
export type TakeIO = {
  write(text: string): void | Promise<void>;
  ask(prompt: string): Promise<string>;
};
```

`cli.ts` passes a real one backed by `console.log`/`readline`; tests pass a
fake one that records writes and returns scripted answers from an array —
see `test/take.test.ts`. Nothing in `takeQuiz` itself knows or cares which.

### The clock is the third injected non-determinism

`takeQuiz` takes an optional `now: () => string`, defaulting to
`() => new Date().toISOString()`. This is the same pattern you've now seen
three times for three different kinds of "this makes tests flaky if you
don't control it":

| What's non-deterministic | Where it's injected | Default |
|---|---|---|
| The network | `llm.ts`'s `fetchImpl` | the real `fetch` |
| Randomness | `render.ts`'s `random` | `Math.random` |
| The clock | `take.ts`'s `now` | `() => new Date().toISOString()` |

Once you've seen the pattern three times it stops looking like a trick and
starts looking like the obvious way to write testable code with real-world
side effects.

### `node:readline/promises`

Node's built-in interactive-prompt library, promise-based (there's also a
callback version under plain `node:readline`). The obvious-looking API is
`rl.question(prompt)`, which returns a promise for one line of input — but
see Watch out below for why this project doesn't use it.

## Exercise

### `src/core/take.ts`

Export:

- `type OptionId = "a" | "b" | "c" | "d"`
- `type SelfGrade = "got-it" | "partly" | "missed"`
- `type Answer = { questionId: number; ownAnswer: string; picked: OptionId; correct: boolean; selfGrade: SelfGrade }`
- `type Attempt = { quiz: string; startedAt: string; finishedAt: string; answers: Answer[] }`
  (`quiz` is the path to `quiz.json`, matching `design.md`'s `Attempt` shape)
- `type TakeIO = { write(text: string): void | Promise<void>; ask(prompt: string): Promise<string> }`
- `async function takeQuiz(quiz: Quiz, quizPath: string, io: TakeIO, now?: () => string): Promise<Attempt>` —
  for every question, in order:
  1. `io.write` the question header, then `io.ask` for the free-text answer.
  2. `io.write` each option, then `askUntilValid` for a pick (a/b/c/d,
     case-insensitive, trimmed — lowercase and trim before validating).
  3. `io.write` whether the pick was correct, then every option's `why`,
     then the citations as `path:start-end`, comma-separated.
  4. `askUntilValid` for a self-grade (`got-it`/`partly`/`missed`, same
     normalization).

     Push an `Answer` built from all of the above.

  After the loop, `io.write` a summary line per answer (pick, correct/wrong,
  self-grade), and return the full `Attempt` (`startedAt`/`finishedAt` from
  `now()`, called once at the start and once at the end).
- `async function saveAttempt(attempt: Attempt, outDir: string): Promise<string>` —
  `mkdir(outDir, { recursive: true })`, write
  `quiz.attempt-<finishedAt with ":" replaced by "-">.json` (pretty-printed,
  trailing newline) into `outDir`, return the path written.

You'll also need two unexported helpers: `isOptionId` and `isSelfGrade` type
guards, and the generic `askUntilValid` from Concepts above.

### `src/cli.ts`

Add the `take` command on top of what you built in chapter 09:

- In `main()`, dispatch `"take"` → a new `runTake(rest)`.
- `async function runTake(argv: string[]): Promise<void>`:
  - Parse the single positional (`quizPath`); missing → usage + exit 1.
  - `JSON.parse(await readFile(quizPath, "utf8"))` as `Quiz`, inside a
    `try`/`catch` that prints a clean error and exits 1 on failure.
  - Create a `readline.createInterface`, build a `TakeIO` around it (see
    Watch out — this is the one part of this chapter that isn't a
    straightforward translation of the exercise text above), call
    `takeQuiz` then `saveAttempt` (into the quiz's directory,
    `path.dirname(quizPath)`), print where the attempt was saved, and
    `rl.close()` in a `finally`.
- Update `printUsage()` to mention `readback take <quiz.json>`.

## Tests

Copy this into `test/take.test.ts`:

```ts
// take.test.ts — drives the interactive flow with a fake TakeIO (an array of
// scripted answers) instead of a real terminal, and checks the saved
// Attempt file.

import { test } from "node:test";
import assert from "node:assert/strict";
import { mkdtemp, readFile, rm } from "node:fs/promises";
import * as os from "node:os";
import * as path from "node:path";
import { takeQuiz, saveAttempt } from "../src/core/take.ts";
import type { TakeIO } from "../src/core/take.ts";
import type { Quiz } from "../src/core/quiz.ts";

const quiz: Quiz = {
  version: 1,
  createdAt: "2024-01-01T00:00:00.000Z",
  repo: "/repo",
  target: "main...feature",
  commit: "abc123",
  model: "test-model",
  questions: [
    {
      id: 1,
      kind: "why",
      question: "Why does the check exist?",
      options: [
        { id: "a", text: "Because X", why: "right" },
        { id: "b", text: "Because Y", why: "wrong" },
        { id: "c", text: "Because Z", why: "wrong" },
        { id: "d", text: "No reason", why: "wrong" },
      ],
      correct: "a",
      citations: [{ path: "a.ts", start: 1, end: 1 }],
    },
    {
      id: 2,
      kind: "predict",
      question: "What does f(2) return?",
      options: [
        { id: "a", text: "1", why: "wrong" },
        { id: "b", text: "2", why: "wrong" },
        { id: "c", text: "4", why: "right" },
        { id: "d", text: "throws", why: "wrong" },
      ],
      correct: "c",
      citations: [{ path: "a.ts", start: 2, end: 3 }],
    },
  ],
};

/** A fake TakeIO: `answers` is consumed in order for every `ask()` call,
 * every `write()` call is recorded for later assertions. */
function fakeIo(answers: string[]): { io: TakeIO; written: string[] } {
  const written: string[] = [];
  let i = 0;
  const io: TakeIO = {
    write: (text) => {
      written.push(text);
    },
    ask: async () => {
      const next = answers[i];
      i++;
      if (next === undefined) throw new Error("fakeIo ran out of scripted answers");
      return next;
    },
  };
  return { io, written };
}

test("takeQuiz walks every question and records picks + self-grades", async () => {
  const { io } = fakeIo([
    "my own answer 1",
    "a", // correct pick
    "got-it",
    "my own answer 2",
    "a", // wrong pick (correct is c)
    "missed",
  ]);

  const attempt = await takeQuiz(quiz, "/out/quiz.json", io, () => "2024-01-01T00:00:00.000Z");

  assert.equal(attempt.quiz, "/out/quiz.json");
  assert.equal(attempt.answers.length, 2);
  assert.deepEqual(attempt.answers[0], {
    questionId: 1,
    ownAnswer: "my own answer 1",
    picked: "a",
    correct: true,
    selfGrade: "got-it",
  });
  assert.deepEqual(attempt.answers[1], {
    questionId: 2,
    ownAnswer: "my own answer 2",
    picked: "a",
    correct: false,
    selfGrade: "missed",
  });
});

test("takeQuiz re-asks on an invalid option pick or self-grade", async () => {
  const { io, written } = fakeIo([
    "own answer",
    "z", // invalid pick, should be re-asked
    "a",
    "got-it",
    "own answer 2",
    "c",
    "maybe", // invalid self-grade, should be re-asked
    "partly",
  ]);

  const attempt = await takeQuiz(quiz, "/out/quiz.json", io);
  assert.equal(attempt.answers[0]?.picked, "a");
  assert.equal(attempt.answers[1]?.selfGrade, "partly");
  assert.ok(written.some((line) => line.includes("not a valid answer")));
});

test("saveAttempt writes a JSON file next to the quiz", async () => {
  const dir = await mkdtemp(path.join(os.tmpdir(), "readback-take-test-"));
  try {
    const { io } = fakeIo(["a1", "a", "got-it", "a2", "c", "got-it"]);
    const attempt = await takeQuiz(quiz, path.join(dir, "quiz.json"), io, () => "2024-01-01T00-00-00.000Z");

    const attemptPath = await saveAttempt(attempt, dir);
    assert.match(attemptPath, /quiz\.attempt-.*\.json$/);

    const saved = JSON.parse(await readFile(attemptPath, "utf8"));
    assert.equal(saved.answers.length, 2);
  } finally {
    await rm(dir, { recursive: true, force: true });
  }
});
```

## Hints

<details><summary>Hint 1</summary>

Write `isOptionId`/`isSelfGrade` first and test them mentally against the
type: `(["a","b","c","d"] as string[]).includes(value)` is the whole body,
same shape for both guards, just a different array and union.

</details>

<details><summary>Hint 2</summary>

`askUntilValid`'s loop body is three lines: ask, trim+lowercase, check. Don't
forget to `trim().toLowerCase()` *before* calling `isValid` — the test script
sends lowercase already, but a real human will capitalize sometimes.

</details>

<details><summary>Hint 3</summary>

In `cli.ts`, resist the urge to call `rl.question()` the obvious way twice
per question (once for free text, once for the pick, once for self-grade —
three times per question, really). Read the Watch out section below before
you write `runTake`.

</details>

<details><summary>Hint 4</summary>

`saveAttempt`'s filename needs `:` replaced with `-` (colons are awkward in
filenames on some filesystems) — same `.replace(/:/g, "-")` idea you've
already used for `generate`'s output directory name in chapter 09, just on
`attempt.finishedAt` instead of `new Date().toISOString()`.

</details>

## Solution

<details><summary>Solution</summary>

`src/core/take.ts`:

```ts
// take.ts — the interactive "take a quiz" flow, and saving the attempt.
// No LLM calls here at all.
//
// This file still can't call console.log or readline directly (that would
// break the "core never touches the terminal" rule and make it untestable
// without spawning a real process). Instead cli.ts hands it a small `TakeIO`
// object: `write` to print something, `ask` to print a prompt and read a
// line back. Tests pass a fake IO that records writes and returns scripted
// answers from an array — see test/take.test.ts.

import { mkdir, writeFile } from "node:fs/promises";
import * as path from "node:path";
import type { Quiz } from "./quiz.ts";

export type OptionId = "a" | "b" | "c" | "d";
export type SelfGrade = "got-it" | "partly" | "missed";

export type Answer = {
  questionId: number;
  ownAnswer: string;
  picked: OptionId;
  correct: boolean;
  selfGrade: SelfGrade;
};

export type Attempt = {
  quiz: string; // path to quiz.json
  startedAt: string;
  finishedAt: string;
  answers: Answer[];
};

export type TakeIO = {
  /** Print a line of information (shown answer, summary, ...). */
  write(text: string): void | Promise<void>;
  /** Print a prompt and return the user's typed line. */
  ask(prompt: string): Promise<string>;
};

const OPTION_IDS: OptionId[] = ["a", "b", "c", "d"];
const SELF_GRADES: SelfGrade[] = ["got-it", "partly", "missed"];

function isOptionId(value: string): value is OptionId {
  return (OPTION_IDS as string[]).includes(value);
}

function isSelfGrade(value: string): value is SelfGrade {
  return (SELF_GRADES as string[]).includes(value);
}

/** Keep asking `io.ask(prompt)` until the answer passes `isValid`. */
async function askUntilValid<T extends string>(
  io: TakeIO,
  prompt: string,
  isValid: (value: string) => value is T,
): Promise<T> {
  for (;;) {
    const raw = (await io.ask(prompt)).trim().toLowerCase();
    if (isValid(raw)) return raw;
    await io.write(`"${raw}" is not a valid answer, try again.`);
  }
}

/**
 * Run the four-step flow (own answer, pick, reveal, self-grade) for every
 * question in the quiz, in order, then return the full Attempt.
 *
 * `now` is injectable so tests get a fixed timestamp instead of the real
 * clock — see how llm.ts injects `fetchImpl` and render.ts injects
 * `random` for the same reason.
 */
export async function takeQuiz(
  quiz: Quiz,
  quizPath: string,
  io: TakeIO,
  now: () => string = () => new Date().toISOString(),
): Promise<Attempt> {
  const startedAt = now();
  const answers: Answer[] = [];

  for (const question of quiz.questions) {
    await io.write(`\nQ${question.id} (${question.kind}): ${question.question}`);

    // Step 1: recall, in the user's own words. We never grade this
    // automatically — it exists so the user commits to an answer before
    // seeing the options, instead of just recognising the right-looking one.
    const ownAnswer = await io.ask("Your answer, in your own words: ");

    // Step 2: options revealed, pick one.
    await io.write("Options:");
    for (const option of question.options) {
      await io.write(`  ${option.id}) ${option.text}`);
    }
    const picked = await askUntilValid(io, "Pick a/b/c/d: ", isOptionId);
    const correct = picked === question.correct;

    // Step 3: answer revealed — correct option, every option's reasoning,
    // and where to go read the real code.
    await io.write(correct ? "Correct!" : `Not quite — correct answer is ${question.correct}.`);
    for (const option of question.options) {
      await io.write(`  ${option.id}) ${option.why}`);
    }
    await io.write(
      `Citations: ${question.citations.map((c) => `${c.path}:${c.start}-${c.end}`).join(", ")}`,
    );

    // Step 4: self-grade against the explanation, not just the pick.
    const selfGrade = await askUntilValid(io, "Self-grade (got-it/partly/missed): ", isSelfGrade);

    answers.push({ questionId: question.id, ownAnswer, picked, correct, selfGrade });
  }

  const finishedAt = now();

  await io.write("\nSummary:");
  for (const answer of answers) {
    await io.write(
      `  Q${answer.questionId}: picked ${answer.picked} (${answer.correct ? "correct" : "wrong"}), self-graded "${answer.selfGrade}"`,
    );
  }

  return { quiz: quizPath, startedAt, finishedAt, answers };
}

/** Save the attempt as quiz.attempt-<timestamp>.json next to the quiz file. */
export async function saveAttempt(attempt: Attempt, outDir: string): Promise<string> {
  await mkdir(outDir, { recursive: true });
  // ISO timestamps contain colons, which are awkward in filenames on some
  // filesystems, so swap them for dashes.
  const safeTimestamp = attempt.finishedAt.replace(/:/g, "-");
  const attemptPath = path.join(outDir, `quiz.attempt-${safeTimestamp}.json`);
  await writeFile(attemptPath, `${JSON.stringify(attempt, null, 2)}\n`, "utf8");
  return attemptPath;
}
```

`src/cli.ts` additions (on top of chapter 09's version — new imports at the
top, a new branch in `main`, and `runTake` itself):

```ts
import { readFile } from "node:fs/promises";
import * as readline from "node:readline/promises";
import * as path from "node:path";
// ...plus the chapter 09 imports...
import { takeQuiz, saveAttempt } from "./core/take.ts";
import type { TakeIO } from "./core/take.ts";
```

```ts
  if (command === "generate") {
    await runGenerate(rest);
  } else if (command === "take") {
    await runTake(rest);
  } else {
    // ...
```

```ts
async function runTake(argv: string[]): Promise<void> {
  const { positionals } = parseArgs({ args: argv, allowPositionals: true });
  const [quizPath] = positionals;
  if (quizPath === undefined) {
    printUsage();
    process.exitCode = 1;
    return;
  }

  let quiz: Quiz;
  try {
    quiz = JSON.parse(await readFile(quizPath, "utf8")) as Quiz;
  } catch (err: unknown) {
    const message = err instanceof Error ? err.message : String(err);
    console.error(`could not read quiz: ${message}`);
    process.exitCode = 1;
    return;
  }

  const rl = readline.createInterface({ input: process.stdin, output: process.stdout });
  // NOTE: we deliberately do NOT use `rl.question()` here. When stdin is a
  // pipe (e.g. `printf '...' | readback take quiz.json`, exactly how our
  // end-to-end test and any scripted use works) rather than a real
  // keyboard, Node can buffer and end the whole input before the second
  // `rl.question()` call attaches its listener, so that promise never
  // resolves. Reading lines from the readline interface as an async
  // iterator instead doesn't have that problem.
  const lines = rl[Symbol.asyncIterator]();
  const io: TakeIO = {
    write: (text) => {
      console.log(text);
    },
    ask: async (prompt) => {
      process.stdout.write(prompt);
      const next = await lines.next();
      if (next.done) return ""; // input ended early; treat as an empty answer
      return next.value;
    },
  };

  try {
    const attempt = await takeQuiz(quiz, quizPath, io);
    const outDir = path.dirname(quizPath);
    const attemptPath = await saveAttempt(attempt, outDir);
    console.log(`\nSaved attempt to ${attemptPath}`);
  } finally {
    rl.close();
  }
}
```

And add the usage line (in `printUsage`, right after the `generate` line):
```
"  readback take <quiz.json>",
```

</details>

## Watch out

> **Watch out — this is the single biggest surprise in this whole project.**
> `rl.question()` + piped stdin can hang forever. If stdin is a pipe (exactly
> how the Tests section below works, and how a beginner will plausibly test
> their own code first) instead of a real keyboard, Node can buffer the
> *entire* input and signal end-of-stream before a *second* `rl.question()`
> call has even attached its listener. The result: the first prompt resolves
> fine, the second one just hangs — and under `node --test`'s default
> timeout that shows up as a confusing "unsettled top-level await" warning
> rather than a clear error.
>
> The fix used here: never call `rl.question()` more than once per
> interface. Instead, grab the interface's async iterator **once**
> (`rl[Symbol.asyncIterator]()`) and call `.next()` on it for every prompt,
> writing the prompt text yourself with `process.stdout.write` first. That's
> exactly what `runTake` above does. If you're curious, the symptom is easy
> to reproduce in isolation: two sequential `rl.question()` calls hang on
> instantly-piped input but work fine with deliberately slow/staggered
> input — a textbook "works on my machine, breaks in CI" bug.

> **Watch out:** `process.exitCode` vs. `process.exit()` matters even more
> here than in chapter 09 — `runTake` holds an open `readline` interface, and
> closing it (`rl.close()` in the `finally`) is what lets Node exit on its
> own once the attempt file is written. Force-exiting would risk cutting off
> that final write.

## Checkpoint

```
$ npm run typecheck
$ npm test
```
All 37 tests should pass (34 from before, plus 3 new ones in
`take.test.ts`).

Then take the quiz you generated in chapter 09, answering everything by
piping input in (this is exactly the scenario the Watch out above is about):

```
$ printf 'my first answer\na\ngot-it\nmy second answer\nb\npartly\n' \
  | npm start -- take ./readback-quiz-<timestamp>/quiz.json
```

Expected output (your question order/options may differ — they're shuffled
per run — but the shape is identical):

```
Q1 (why): Why does `add` take two parameters instead of an array?
Your answer, in your own words: Options:
  a) Because TypeScript doesn't support array parameters
  b) Because arrays are slower in JavaScript
  c) Because the call sites always pass exactly two numbers
  d) Because of a historical bug fix
Pick a/b/c/d: Not quite — correct answer is c.
  a) Wrong: TypeScript supports array parameters fine.
  b) Wrong: not why this function looks the way it does.
  c) Right: that's the only way `add` is called in this file.
  d) Wrong: no such history here.
Citations: math.ts:1-3
Self-grade (got-it/partly/missed): 
Q2 (predict): What does `add(2, 3)` return?
Your answer, in your own words: Options:
  a) 5
  b) 23
  c) undefined
  d) An error
Pick a/b/c/d: Not quite — correct answer is a.
  a) Right: `add` returns a + b.
  b) Wrong: that would be string concatenation, but both args are numbers.
  c) Wrong: the function always returns a value.
  d) Wrong: nothing here throws.
Citations: math.ts:2-2
Self-grade (got-it/partly/missed): 
Summary:
  Q1: picked a (wrong), self-graded "got-it"
  Q2: picked b (wrong), self-graded "partly"

Saved attempt to ./readback-quiz-<timestamp>/quiz.attempt-<timestamp>.json
```
(Blank-looking lines after a prompt are just the piped answer being echoed
inline by your terminal/shell, not Readback — they look normal when typed
interactively.)

Open the saved `quiz.attempt-*.json` and check it has `startedAt`,
`finishedAt`, and one `answers` entry per question, matching what you typed.

Suggested commit:
```
$ git add src/core/take.ts test/take.test.ts src/cli.ts
$ git commit -m "Add take command: interactive self-grading flow"
```

[← 09 The `generate` command](09-generate.md) · [README](README.md) · [11 What next →](11-what-next.md)
