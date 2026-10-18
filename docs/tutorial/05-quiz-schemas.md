# 05 — Quiz schemas

[← 04. Building context](04-context.md) · [06. Calling the LLM →](06-llm.md)

## Goal

So far every type you've written describes data your *own* code produces.
The model's reply is different: it's JSON text from an external process, and
nothing stops it from being malformed, missing a field, or outright lying
about its shape. This chapter starts `src/core/quiz.ts`: the **zod** schemas
that define exactly what a valid quiz reply looks like, the TypeScript types
derived from them, the prompt sent to the model, and a parser that turns raw
text into validated data or throws a clear error. Citation checking against
the real repo — "does this file:line actually exist" — comes in
[chapter 07](07-citations.md), once [chapter 06](06-llm.md) has given you
something that actually talks to a model.

## Concepts

### Types are erased; external data isn't automatically validated

Here's the core problem this chapter solves. At runtime, all your `type` and
`interface` declarations vanish — Node's type stripping (and `tsc` itself)
erase them before anything executes. That's fine for data *you* construct in
code the compiler already checked. It's not fine for data that arrives from
outside: a model's HTTP response, a config file, `JSON.parse` of anything.

```ts
const data = JSON.parse(someResponseText) as LlmQuiz; // LIES to the compiler
```

That `as` cast is a promise to the compiler, not a check. If the model
returns `{ "oops": true }`, `data` is still typed `LlmQuiz` and your code
will happily try to read `data.questions`, get `undefined`, and fail
somewhere confusing three function calls later. This is true in every
statically typed language that deserializes external data (a Go
`json.Unmarshal` into the wrong struct silently zero-values fields; a C#
`JsonSerializer.Deserialize<T>` can throw, but only for *some* shape
mismatches) — the type system can't see data it didn't produce. You need a
runtime check, and you want the check and the compile-time type to never
drift apart from each other.

### zod: one definition, two uses

zod lets you describe a shape once, as a runtime value, and derive both a
validator *and* a TypeScript type from it:

```ts
import { z } from "zod";

const pointSchema = z.object({
  x: z.number(),
  y: z.number(),
});

type Point = z.infer<typeof pointSchema>; // { x: number; y: number }

pointSchema.parse({ x: 1, y: 2 });     // returns a Point, or throws
pointSchema.safeParse(unknownValue);   // returns { success, data } or { success: false, error }
```

`z.infer<typeof schema>` reads the schema you just built and produces the
matching compile-time type automatically. Compare that to hand-maintaining a
separate `interface Point` next to the validation logic: the moment someone
adds a field to one and forgets the other, they drift apart silently. With
`z.infer`, there's only one source of truth — the schema.

Building blocks you'll use here: `z.object({...})`, `z.string().min(1)`
(non-empty string), `z.enum([...])` (one of a fixed set of string literals),
`z.array(itemSchema).length(4)` / `.min(1)` (exact or minimum array length),
and `z.int().min(1)` (a whole number, at least 1).

### `safeParse`, not `parse` + try/catch

```ts
const result = schema.safeParse(json);
if (!result.success) {
  throw new QuizParseError(`model reply did not match the quiz schema: ${result.error.message}`);
}
return result.data; // typed, validated
```

`schema.parse(x)` throws on failure, which would mean `parseLlmQuiz` needs
*two* different try/catch blocks: one around `JSON.parse` (which can also
throw, for genuinely malformed JSON) and a separate one around schema
validation, each producing a different kind of error. `safeParse` instead
returns a result object — `{ success: true, data }` or
`{ success: false, error }` — so you check `.success` like you'd check an
`ok`/`Result` value in Go or Rust, and both failure modes (bad JSON, JSON
that doesn't match the schema) end up funneled through the same
`QuizParseError`.

### Untrusted vs. trusted: two types, one concept

```ts
export type LlmQuestion = z.infer<typeof llmQuestionSchema>; // no id — the model doesn't assign one
export type Question = {
  id: number;
  // ...the same fields, but validated and with an id
};
```

`LlmQuestion` is "whatever the model said, schema-shaped but not yet
fact-checked". `Question` is "survived citation validation and got a real
id" — the shape documented in `docs/design.md` and written to `quiz.json`.
Keeping these as two distinct types (rather than one type with an optional
`id?: number`) means the compiler itself won't let you accidentally write an
unvalidated `LlmQuestion` somewhere a trusted `Question` is expected — you'd
need to go through `assignIds` (chapter 07) first. This "parse, don't
validate repeatedly" pattern — turning an untyped/undertyped blob into a
narrower type once, near the boundary, and passing the narrower type
everywhere after — is worth remembering for any external data you handle.

> Preview: [chapter 06](06-llm.md) uses `z.toJSONSchema(llmQuizSchema)` to
> turn this same schema into a JSON Schema document, sent to the model so it
> knows the exact shape to reply with — one schema, three jobs (runtime
> validation, compile-time types, and the wire-format description the model
> sees).

## Exercise

Create `src/core/quiz.ts` with these exports:

```ts
export const llmQuizSchema: /* a zod object schema */;

export type LlmQuestion = z.infer<typeof llmQuestionSchema>;
export type LlmQuiz = z.infer<typeof llmQuizSchema>;

export type Question = {
  id: number;
  kind: "failure" | "why" | "predict" | "bug";
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

export type ChatMessage = { role: "system" | "user"; content: string };
export function buildPrompt(scope: Scope, context: string, questionCount: number): ChatMessage[];

export class QuizParseError extends Error {}
export function parseLlmQuiz(raw: string): LlmQuiz;
```

Required behavior — build the schemas bottom-up, matching `docs/design.md`'s
`Question`/`Quiz` shapes:

- An option id is one of `"a" | "b" | "c" | "d"`.
- An option (as the model returns it) has `id`, a non-empty `text`, and a
  non-empty `why` (explaining why the option is right or wrong).
- A citation has a non-empty `path`, and `start`/`end` as integers `>= 1`.
- A question kind is one of `"failure" | "why" | "predict" | "bug"`.
- A question (as the model returns it, **no `id` field** — the model never
  assigns one) has a `kind`, a non-empty `question` string, **exactly 4**
  options, a `correct` option id, and **at least 1** citation.
- `llmQuizSchema` is `{ questions: <array of the above, at least 1> }`.
- `buildPrompt(scope, context, questionCount)` returns a two-message array:
  a `"system"` message describing the question kinds and the rules (every
  question has 4 options with one correct and a `why` each, every question
  cites real `file:line`s, reply with JSON only), and a `"user"` message
  asking for `questionCount` questions about the scope, including the
  `context` text. Mention the scope's base/head (for a range) or path/rev
  (for a path) somewhere in the user message.
- `parseLlmQuiz(raw)`: `JSON.parse` the string, then validate against
  `llmQuizSchema`. Throw `QuizParseError` if the JSON itself is invalid, or
  if it's valid JSON that doesn't match the schema. Return the validated,
  typed `LlmQuiz` on success.

## Tests

Create `test/quiz.test.ts`:

```ts
// quiz.test.ts — schema parsing and citation validation, including the
// "invalid citations get dropped" case called out in the task spec.

import { test } from "node:test";
import assert from "node:assert/strict";
import { parseLlmQuiz, QuizParseError, buildPrompt } from "../src/core/quiz.ts";
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
```

(`buildPrompt`'s exact wording is yours to choose, as long as it matches the
required behavior above; the test only checks for substrings like `"5
questions"` and `"commit x and commit y"`, so phrase the rest however you
like — just keep those recognizable.)

## Hints

<details><summary>Hint 1</summary>

Build the schemas bottom-up as separate `const`s, each one feeding the next:
an `optionId` enum, then `llmOptionSchema`, `citationSchema`, and
`questionKind` (all independent), then `llmQuestionSchema` (uses the first
three), then `llmQuizSchema` (uses `llmQuestionSchema`). None of these
intermediate `const`s need to be exported — only `llmQuizSchema` does (plus
the two inferred types).

</details>

<details><summary>Hint 2</summary>

`Question` and `Quiz` are hand-written `type` aliases, not `z.infer` results
— there's no schema for them yet (they're never parsed from untrusted input;
they're *produced* by your own code in chapter 07). You can reuse
`z.infer<typeof questionKind>` for the `kind` field instead of retyping the
union, if you want one less place that lists the four kinds.

</details>

<details><summary>Hint 3</summary>

`parseLlmQuiz` needs two separate failure points: `JSON.parse` can throw
(catch it, rethrow as `QuizParseError`), and `llmQuizSchema.safeParse` can
return `{ success: false }` (check `result.success`, throw
`QuizParseError` using `result.error.message`). Don't try to catch both in
one `try` block — `safeParse` doesn't throw, by design.

</details>

## Solution

<details><summary>Solution</summary>

```ts
// quiz.ts — the zod schemas that define what a quiz looks like, the prompt
// we send the model, and validation that keeps the model honest:
// every citation must point at a real file and real lines.

import { z } from "zod";
import type { Scope } from "./scope.ts";

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
```

</details>

## Watch out

> **Watch out:** a zod `.safeParse()` failure's `.message` is one big
> formatted multi-line string, not a structured list — fine to drop straight
> into an `Error` message for a CLI's stderr, which is what `parseLlmQuiz`
> does. If you ever want to show a user something like "field `citations[0].start`:
> expected number, got string" instead of zod's default formatting, zod 4
> also exposes `result.error.issues` as a structured array — worth knowing
> about even though this project doesn't need it yet.

> **Watch out:** `z.toJSONSchema` (used in the next chapter to describe this
> same schema to the model over HTTP) only exists in **zod 4**. If you ever
> search for older examples and can't find it, you're probably looking at
> zod 3 docs, where the equivalent required a separate `zod-to-json-schema`
> package. This project pins zod 4.6.7 deliberately.

## Checkpoint

```
$ npm run typecheck
$ npm test
```

Expected: `tsc` reports no errors, 18 tests pass (14 from before, 4 new in
`quiz.test.ts`).

```
$ git add src/core/quiz.ts test/quiz.test.ts
$ git commit -m "core: quiz schemas and prompt"
```

Next: [06 — Calling the LLM](06-llm.md), where `llm.ts` actually sends this
prompt to a model and gets a reply back.
