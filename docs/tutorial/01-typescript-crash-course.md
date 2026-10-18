# 01 — TypeScript crash course

[← 00. Setup](00-setup.md) · [02. Targets →](02-targets.md)

## Goal

Get comfortable with the TypeScript/JavaScript features you'll use
constantly from [chapter 02](02-targets.md) onward, before mixing them with
new Readback-specific concepts. Everything here lives in a throwaway
`scratch/` directory — nothing in this chapter becomes part of the CLI.

## Setup for this chapter

```
$ mkdir scratch
$ echo "scratch/" >> .gitignore
```

You won't run `npm test` this chapter — there's nothing to test, only code
to typecheck and run. Two ways to check a snippet:

- **Run it**: `node scratch/01-variables.ts` (Node runs `.ts` directly, as
  you saw in chapter 00).
- **Typecheck it**: the root `tsconfig.json` only covers `src/`, `test/`
  and `scripts/`, so give `scratch/` its own small config that reuses the
  root settings. Create `scratch/tsconfig.json`:

  ```json
  {
    "extends": "../tsconfig.json",
    "include": ["./**/*.ts"]
  }
  ```

  VS Code picks it up automatically (if it doesn't, run **TypeScript:
  Restart TS Server** from the command palette), and you can check from
  the command line with:

  ```
  $ npx tsc -p scratch
  ```

> **Watch out:** without `scratch/tsconfig.json`, VS Code checks scratch
> files with its default settings. Those treat a file with no
> `import`/`export` as a *script*: every top-level `const` in every such
> file shares one global scope. The defaults also include the browser's
> built-in globals, such as `name`. You then get errors like *"Cannot
> redeclare block-scoped variable 'name'"* for perfectly valid code. With
> the project settings (`"module": "nodenext"` plus `"type": "module"` in
> `package.json`), every file is its own module and the errors go away.
> Also don't run `npx tsc scratch/*.ts`: when you pass file names, tsc
> ignores `tsconfig.json` completely and refuses to run.

## Concepts

Each section below is one concept, a tiny example, and a comparison to a
language you may already know. The table of contents doubles as a
reference you can come back to:

| # | Concept | Used starting in |
|---|---------|-------------------|
| 1 | `let`/`const`, primitives, inference | everywhere |
| 2 | Object types, `type` vs `interface` | [02](02-targets.md) (`Scope`) |
| 3 | Arrays and tuples | [04](04-context.md) |
| 4 | Functions, arrow functions, defaults | everywhere |
| 5 | Union and literal types | [02](02-targets.md) (`Scope`'s `kind`) |
| 6 | `null`/`undefined`, `?.`, `??` | [04](04-context.md), [05](05-quiz-schemas.md) |
| 7 | `any` vs `unknown` | [06](06-llm.md) (parsing network responses) |
| 8 | `as const`, template literals | [05](05-quiz-schemas.md) |
| 9 | Destructuring and spread | [09](09-generate.md) (`process.argv`) |
| 10/11 | ES modules, `import type` | every `src/` file |
| 12 | Classes (just `extends Error`) | [02](02-targets.md) (`ScopeError`) |
| 13 | Promises and `async`/`await` | [02](02-targets.md) onward (git, I/O) |

### 1. `let`/`const`, primitives, inference

TypeScript infers types from the initial value — you rarely need to write
`: number` yourself. `const` means the *binding* can't be reassigned; it
says nothing about whether the value itself can change.

```ts
let score = 0; // inferred: number
score = score + 1;

const list: number[] = [];
list.push(1); // fine — same array, mutated in place; only `list = []` would error
```

Prefer `const` by default, `let` only when you'll reassign. (Python/Go/C#
have no equivalent of this distinction at the binding level — closest is
C#'s `readonly`/`const`, but those are compile-time constants, stricter
than TS's `const`.)

**Exercise:** create `scratch/01-variables.ts`. Declare a `const` string
`label`, a `let` number `count` starting at `0` that you increment once,
and a `const` array of numbers that you `.push()` one value into. Log all
three.

<details><summary>Hint 1</summary>

You don't need to write type annotations for any of these — let inference
do it, then hover over each variable in your editor to confirm what it
inferred.

</details>

<details><summary>Solution</summary>

```ts
const label = "quiz";
let count = 0;
count = count + 1;
const list: number[] = [];
list.push(1);
console.log(label, count, list);
```

</details>

### 2. Object types and `type` vs `interface`

```ts
type User = {
  name: string;
  age: number;
  email?: string; // optional: `string | undefined`
};

const alice: User = { name: "Alice", age: 30 };
```

TypeScript is **structurally typed**: `alice` is a valid `User` because its
*shape* matches, regardless of how it was constructed — there's no "class
implements interface" declaration required, unlike C# or Go's (also
structural, actually — Go interfaces are structural too) explicit
`implements`/interface-satisfaction syntax. Python has no static
equivalent at all (duck typing, checked only if you bother with `mypy`
Protocols).

`interface` can describe the same shapes:

```ts
interface Point {
  x: number;
  y: number;
}
```

This project uses `type` for every data shape (`Scope`, `Question`,
`Quiz`, ...) because they're plain data, never classes. `interface` is
mainly useful when you want "declaration merging" (two `interface Foo { }`
blocks combine) or a `class` to `implements` it — neither comes up here.
Rule of thumb: default to `type`, reach for `interface` only if something
specifically needs it.

**Exercise:** create `scratch/02-objects.ts`. Define a `type Book` with
`title: string`, `pages: number`, and an optional `rating?: number`. Create
one `Book` with a rating and one without.

<details><summary>Solution</summary>

```ts
type Book = {
  title: string;
  pages: number;
  rating?: number;
};
const a: Book = { title: "A", pages: 100, rating: 5 };
const b: Book = { title: "B", pages: 200 };
console.log(a, b);
```

</details>

### 3. Arrays and tuples

```ts
const names: string[] = ["a", "b", "c"];
const alsoNames: Array<string> = ["a", "b"]; // same thing, generic syntax

// Tuple: fixed length, specific type per position
const pair: [string, number] = ["age", 30];
```

A tuple is TypeScript's way of saying "this array always has exactly these
types in this order" — closer to a small Go struct or a C# `(string,
int)` value tuple than Python's loosely-typed tuples.

**Exercise:** create `scratch/03-arrays.ts`. Make a `string[]` of three
file extensions, and a tuple `[string, boolean]` representing a filename
and whether it's binary.

<details><summary>Solution</summary>

```ts
const extensions: string[] = [".ts", ".md", ".json"];
const file: [string, boolean] = ["photo.png", true];
console.log(extensions, file);
```

</details>

### 4. Functions, arrow functions, optional/default params

```ts
function add(a: number, b: number): number {
  return a + b;
}

const multiply = (a: number, b: number): number => a * b; // arrow function

function greet(name: string, greeting = "Hello"): string { // default param
  return `${greeting}, ${name}!`;
}

function describe(name: string, title?: string): string { // optional param
  return title === undefined ? name : `${title} ${name}`;
}
```

Put optional (`?`) and default-valued params after the required ones.
For `?` params TypeScript enforces this. A default-valued param *may* come
first, but arguments are matched by position only (there are no keyword
arguments like Python's), so the only way to skip it is to pass `undefined`
explicitly: `f(undefined, 30)`. In practice, keep defaults at the end. Arrow functions are
mostly interchangeable with `function` here; this project uses `function`
for top-level named functions (easier to grep, shows up better in stack
traces) and arrows for short inline callbacks (you'll see this in
[chapter 04](04-context.md)'s sort comparators).

**Exercise:** create `scratch/04-functions.ts` with a `function` that takes
two numbers and returns their difference, and an arrow function version of
the same, plus one function with a default parameter.

<details><summary>Solution</summary>

```ts
function subtract(a: number, b: number): number {
  return a - b;
}
const subtractArrow = (a: number, b: number): number => a - b;
function pad(text: string, width = 10): string {
  return text.padStart(width);
}
console.log(subtract(5, 2), subtractArrow(5, 2), pad("x"));
```

</details>

### 5. Union and literal types

```ts
type Id = string | number;

function formatId(id: Id): string {
  if (typeof id === "string") {
    return id.toUpperCase(); // TS knows `id` is `string` here — "narrowing"
  }
  return id.toFixed(0); // and `number` here, in the other branch
}

type Direction = "up" | "down" | "left" | "right"; // a literal type: not
                                                     // "any string", one of
                                                     // these four specific values

function move(direction: Direction): void {
  console.log(`moving ${direction}`);
}
move("up");
// move("sideways"); // type error: not a valid Direction
```

This is the single most important pattern in this codebase: `Scope` (built
in [chapter 02](02-targets.md)) is a union of two object shapes
distinguished by a literal `kind` field, and the compiler narrows which
shape you have based on an `if (scope.kind === "range")` check — no casting,
no `instanceof`, just a plain comparison the compiler understands. Go has
no direct equivalent (closest is a tagged struct + type switch on an
interface); Python's closest is `Literal` + `match` from `typing`, rarely
used; C# got discriminated unions informally via pattern matching on
records in recent versions.

**Exercise:** create `scratch/05-unions.ts`. Define `type Status = "pending"
| "done" | "failed"` and a function `describe(status: Status): string` that
returns a different message per status (use `if`/`else` or `switch`).

<details><summary>Hint 1</summary>

A `switch` works well here and `tsconfig.json`'s
`noFallthroughCasesInSwitch` will catch a missing `break`.

</details>

<details><summary>Solution</summary>

```ts
type Status = "pending" | "done" | "failed";

function describe(status: Status): string {
  switch (status) {
    case "pending":
      return "still working";
    case "done":
      return "finished";
    case "failed":
      return "something went wrong";
  }
}
console.log(describe("done"));
```

</details>

### 6. `null` vs `undefined`, `?.`, `??`

JavaScript has both `null` ("deliberately no value") and `undefined`
("never set"). An optional object property (`value?: string`) is
`undefined` when omitted, never `null`.

```ts
type Maybe = { value?: string };
const a: Maybe = {};

console.log(a.value?.toUpperCase()); // "?." (optional chaining): undefined
                                      // instead of throwing when a.value
                                      // is null/undefined

const count: number | null = 0;
console.log(count ?? 10); // "??" (nullish coalescing): 0 — only null/undefined fall through
console.log(count || 10); // 10 — "||" also falls through on 0, "", false: a classic footgun
```

Python only has `None`; Go only has zero values (no null at all for value
types) plus explicit pointers/`nil`; C# has `null` and (for value types)
`Nullable<T>`/`?`. Having *both* `null` and `undefined` in JS is a
historical wart you just need to know about, not a feature to exploit —
this codebase mostly uses `undefined` (optional properties, optional
params) and reserves explicit `null` for "looked it up and there really is
no value" (you'll see `Map<string, number | null>` in
[chapter 07](07-citations.md)).

**Exercise:** create `scratch/06-nullish.ts`. Write a function
`shout(text?: string): string` that returns `text` upper-cased, or
`"(nothing)"` if `text` is missing — using `??`, not an `if`.

<details><summary>Solution</summary>

```ts
function shout(text?: string): string {
  return text?.toUpperCase() ?? "(nothing)";
}
console.log(shout("hi"), shout()); // HI (nothing)
```

Order matters: `(text ?? "(nothing)").toUpperCase()` would upper-case the
fallback too and print `(NOTHING)`. Upper-case first (`?.` skips it when
`text` is missing), then fall back.

</details>

### 7. `any` vs `unknown`

```ts
function parseUnknown(raw: string): unknown {
  return JSON.parse(raw);
}
const u = parseUnknown('"hello"');
// console.log(u.toUpperCase()); // type error: 'u' is of type 'unknown'
if (typeof u === "string") {
  console.log(u.toUpperCase()); // allowed after narrowing: prints HELLO
}

function parseAny(raw: string): any {
  return JSON.parse(raw);
}
const a = parseAny("42");
console.log(a.toUpperCase()); // compiles! crashes at runtime: 42 has no .toUpperCase
```

The `unknown` half comes first on purpose. The `any` crash stops the
program, so anything after it would never run.

`any` turns off type checking for that value completely — it's a hole in
the type system, not a type. `unknown` is the honest version: "I don't
know what this is yet", and the compiler refuses to let you call methods
on it until you've proven (via `typeof`, `instanceof`, a custom check...)
what it actually is. `JSON.parse`'s real return type is `any` for backward
compatibility — this project never trusts it directly:
[chapter 05](05-quiz-schemas.md) narrows the model's quiz JSON with `zod`,
which checks the whole shape declaratively, and [chapter 06](06-llm.md)
narrows a `fetch` response by hand with `typeof`/`in` checks.

**Exercise:** create `scratch/07-any-unknown.ts` with both functions above
and both call sites, in the same order. Run it with `node`: you should see
`HELLO`, then a `TypeError` from the `any` call. That's the point: `tsc`
won't save you from `any`. Also try un-commenting the first
`u.toUpperCase()` line to see `tsc` refuse it.

<details><summary>Solution</summary>

See the code block above — copy it as-is into the file.

</details>

### 8. `as const` and template literal types

```ts
const options = ["a", "b", "c"] as const; // readonly ["a","b","c"], not string[]
// options.push("d"); // type error: readonly arrays have no push

type OptionLetter = (typeof options)[number]; // "a" | "b" | "c"

type Level = "low" | "high";
type Flag = `--${Level}`; // "--low" | "--high" — built from pieces
const flag: Flag = "--high";
```

`as const` tells TS "narrow this to the most specific literal type
possible, and make it read-only" instead of widening `"a"` to `string`.
Template literal types build string-literal unions out of smaller pieces —
useful for things like CLI flags. Neither has a real equivalent in
Python/Go/C# (closest: C# wishing `const` worked on arrays, which it
doesn't).

**Exercise:** create `scratch/08-as-const.ts` with:

1. `const colors = ["red", "green", "blue"] as const;` and a `type Color`
   derived from it.
2. A template literal type `ColorFlag` built **from `Color`**, giving
   `"--red" | "--green" | "--blue"`.
3. A type `Hex` of the form `` `#${string}` `` (any string starting with `#`).
4. A lookup table `hexOf` that gives every `Color` a `Hex` value. Use
   `Record<Color, Hex>`, which means "an object with exactly one key per
   `Color`, each holding a `Hex`".

Then try breaking it: remove `blue` from the table, write `"00ff00"`
without the `#`, or assign `"--purple"` to a `ColorFlag`. Each one is a
type error.

<details><summary>Solution</summary>

```ts
const colors = ["red", "green", "blue"] as const;
type Color = (typeof colors)[number];  // "red" | "green" | "blue"

type ColorFlag = `--${Color}`;         // "--red" | "--green" | "--blue"
type Hex = `#${string}`;               // any string starting with "#"

const hexOf: Record<Color, Hex> = {
  red: "#ff0000",
  green: "#00ff00",
  blue: "#0000ff",
};

const flag: ColorFlag = "--green";
console.log(flag, hexOf.green);        // --green #00ff00

for (const color of colors) {
  console.log(`--${color}`, hexOf[color]);
}
```

All the types come from the one `colors` array. Add `"yellow"` to it and
`Color` and `ColorFlag` gain a member automatically, and `tsc` immediately
reports that `hexOf` is missing a `yellow` entry. Write a list once, derive
the types from it, and let the compiler catch every place you forgot to
update. Readback uses this idea with zod in [chapter 05](05-quiz-schemas.md).

</details>

### 9. Destructuring and spread

```ts
const point = { x: 1, y: 2, z: 3 };
const { x, y } = point;                     // pick fields
const { x: px, ...rest } = point;           // rename + collect the rest

const numbers = [1, 2, 3, 4];
const [first, second, ...restOfNumbers] = numbers;

const extended = { ...point, w: 4 };        // new object, original untouched
const combined = [...numbers, 5, 6];        // new array, original untouched
```

You'll use array destructuring with "collect the rest" almost immediately
in [chapter 09](09-generate.md):
`const [, , command, ...rest] = process.argv;` — the leading commas skip
elements you don't need (here, `node` and the script path).

**Exercise:** create `scratch/09-destructuring.ts`. Destructure `name` and
`age` out of a `{ name, age, city }` object while collecting the rest, and
destructure the first two elements plus "the rest" out of a five-element
array.

<details><summary>Solution</summary>

```ts
const person = { name: "Kim", age: 40, city: "Oslo" };
const { name, age, ...others } = person;
console.log(name);    // Kim             (from person.name)
console.log(others);  // { city: 'Oslo' } (every field not picked)
console.log(person);  // unchanged: { name: 'Kim', age: 40, city: 'Oslo' }

const values = [1, 2, 3, 4, 5];
const [a, b, ...more] = values;
console.log(a, b);    // 1 2             (by position)
console.log(more);    // [ 3, 4, 5 ]     (everything after)
```

Each `console.log` shows one idea. The expected output is in the
comment next to it.

</details>

### 10/11. ES modules, named vs default exports, `import type`

```ts
// 10-math-utils.ts
export function square(n: number): number {
  return n * n;
}
export type Operation = "add" | "subtract";
export default function cube(n: number): number {
  return n * n * n;
}
```

```ts
// 11-modules.ts
import cube from "./10-math-utils.ts";                // 1. default: any name you like
import { square } from "./10-math-utils.ts";          // 2. named: must match the export
import type { Operation } from "./10-math-utils.ts";  // 3. type-only: erased at runtime

const op: Operation = "add";
console.log(square(3), cube(3), op); // 9 27 add
```

Styles 1 and 2 are often combined into one line,
`import cube, { square } from "./10-math-utils.ts";`. That is the same two
imports written together.

A module can have any number of **named exports** (imported with the exact
name, or renamed with `as`) and at most one **default export** (imported
under any name you choose). This project uses named exports almost
exclusively — easier to grep, no ambiguity about what a bare `import X`
refers to. `import type` imports *only* the type information, erased
completely at runtime; it's required by `verbatimModuleSyntax`
(`tsconfig.json`, chapter 00) whenever you import something used purely as
a type, and you'll write it constantly starting in
[chapter 04](04-context.md).

**Exercise:** create `scratch/10-math-utils.ts` as shown, then
`scratch/11-modules.ts` that imports from it using all three import styles
(default, named, type-only).

<details><summary>Hint 1</summary>

Remember the `.ts` extension in the import path — this is the same rule
from chapter 00's "Watch out", not optional here either.

</details>

<details><summary>Solution</summary>

Copy both code blocks above verbatim into the two files. `node
scratch/11-modules.ts` prints `9 27 add`.

</details>

### 12. Classes — just enough for `extends Error`

```ts
class ScopeError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "ScopeError"; // shows up in stack traces / logs
  }
}

function parseTarget(target: string): string {
  if (target === "") {
    throw new ScopeError("target must not be empty");
  }
  return target;
}

try {
  parseTarget("");
} catch (err) {
  // `catch (err)` is typed `unknown` under `strict` — narrow before use.
  if (err instanceof ScopeError) {
    console.log("caught:", err.message, err.name);
  }
}
```

This codebase has no other use of classes or inheritance — just custom
`Error` subclasses (`ScopeError`, `LlmError`, `QuizParseError`, ...) so
callers can `instanceof`-check *which kind* of error happened, the same
idea as custom exception types in Python or C#, or wrapped error types in
Go, just expressed with `extends`.

**Exercise:** create `scratch/12-classes.ts` with the code above, then add
a second custom error class `ValidationError extends Error` and a
`try`/`catch` that only handles `ValidationError`, re-throwing anything
else (`throw err;` in an `else` branch).

<details><summary>Solution</summary>

```ts
class ValidationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = "ValidationError";
  }
}

try {
  throw new ValidationError("bad input");
} catch (err) {
  if (err instanceof ValidationError) {
    console.log("validation failed:", err.message);
  } else {
    throw err;
  }
}
```

</details>

### 13. Promises and `async`/`await`

```ts
function delay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function fetchGreeting(name: string): Promise<string> {
  await delay(10); // pretend this is a network call
  return `Hello, ${name}`;
}

const greeting = await fetchGreeting("Readback"); // top-level await: fine in an ES module

async function sequential(): Promise<void> {
  await delay(10);
  await delay(10); // starts only after the first finishes: ~20ms total
}

async function concurrent(): Promise<void> {
  await Promise.all([delay(10), delay(10)]); // both start immediately: ~10ms total
}
```

A `Promise<T>` represents a value of type `T` that will exist later —
comparable to Python's `asyncio.Future`/coroutine, Go's "send the result
over a channel", or C#'s `Task<T>`. `async function` always returns a
`Promise`; `await` pauses *that function* (not the whole program) until
the promise settles. Sequential `await`s in a loop (you'll do this
deliberately in [chapter 07](07-citations.md) for citation checks, one
file at a time) are slower but simpler than firing everything with
`Promise.all` — a real tradeoff, not just a beginner mistake, worth
choosing on purpose per situation.

**The event loop, briefly.** JavaScript runs on a single thread — there is
no `Thread`/goroutine-style parallelism for your own code. When you
`await` something I/O-bound (a file read, a `fetch`, `setTimeout`), Node
hands that operation off to the underlying system and immediately goes on
to run other pending code; your `async function` only resumes once the
result is ready, on a future turn of the "event loop" (Node's main
scheduling loop). This is why async code doesn't need locks or mutexes the
way threaded code in Go/C# might: at any instant, only one piece of your
JS is actually executing.

That single-threaded model is also why a CPU-heavy loop *can* block
everything else (there's no pre-emption), but it's rarely an issue for a
CLI like this one that's almost entirely waiting on git subprocesses and
network calls rather than crunching numbers — exactly the kind of workload
`async`/`await` was designed for.

**Exercise:** create `scratch/13-async.ts` with `delay`, `sequential` and
`concurrent` as shown, and call both with top-level `await`. Then make the timing visible: use longer delays (500 ms)
and print a timestamp whenever a task starts and finishes.

<details><summary>Solution</summary>

```ts
const start = performance.now();

// Prints a message prefixed with milliseconds since the program started.
function log(message: string): void {
  const ms = Math.round(performance.now() - start);
  console.log(`[${String(ms).padStart(4)} ms] ${message}`);
}

function delay(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

async function task(name: string, ms: number): Promise<string> {
  log(`${name}: start (${ms} ms)`);
  await delay(ms); // pretend this is a network call
  log(`${name}: done`);
  return name;
}

async function sequential(): Promise<void> {
  await task("A", 500);
  await task("B", 500); // starts only after A finishes
}

async function concurrent(): Promise<void> {
  await Promise.all([task("C", 500), task("D", 500)]); // both start immediately
}

log("sequential: begin");
await sequential();
log("sequential: end (~1000 ms after begin)");

log("concurrent: begin");
await concurrent();
log("concurrent: end (~500 ms after begin)");

// Calling an async function without awaiting it: it runs until its first
// `await`, then control comes straight back here.
const pending = task("E", 300);
log("not waiting for E: this line runs before E finishes");
await pending;
log("E awaited");
```

Output (your numbers will differ by a few ms):

```
[   0 ms] sequential: begin
[   2 ms] A: start (500 ms)
[ 502 ms] A: done
[ 502 ms] B: start (500 ms)
[1005 ms] B: done
[1005 ms] sequential: end (~1000 ms after begin)
[1005 ms] concurrent: begin
[1005 ms] C: start (500 ms)
[1005 ms] D: start (500 ms)
[1507 ms] C: done
[1507 ms] D: done
[1507 ms] concurrent: end (~500 ms after begin)
[1507 ms] E: start (300 ms)
[1507 ms] not waiting for E: this line runs before E finishes
[1808 ms] E: done
[1808 ms] E awaited
```

What to look for:

- **Sequential:** B starts only when A is done, so 500 + 500 ≈ 1000 ms.
- **Concurrent:** C and D start at the same moment and finish together,
  so ≈ 500 ms in total. That's what `Promise.all` buys you.
- **Not awaited:** `E: start` prints immediately, because an async
  function runs synchronously until its first `await`. Then the caller
  carries on, and `E: done` appears 300 ms later. Nothing waits for a
  promise unless you `await` it.

</details>

## Tests

There's no test file this chapter — the check is that every snippet
typechecks and runs. Validate the whole directory at once:

```
$ npx tsc -p scratch
$ for f in scratch/*.ts; do echo "== $f =="; node "$f"; done
```

## Watch out

> **Watch out:** `scratch/07-any-unknown.ts`'s `any` branch is *meant* to
> crash when you run it with `node` (`a.toUpperCase is not a function`),
> even though `npx tsc` reports no errors on it. That gap — compiles clean,
> fails at runtime — is the entire lesson of `any`: it's an escape hatch
> from the type checker, not a type.

## Checkpoint

```
$ npx tsc -p scratch
```
Expected: no output, exit code `0` — every snippet typechecks.

```
$ for f in scratch/*.ts; do echo "== $f =="; node "$f"; done
```
Expected: every file prints its `console.log` output; only
`07-any-unknown.ts` ends with an uncaught `TypeError` (on purpose — see
"Watch out").

There's nothing to commit from this chapter — `scratch/` is gitignored.
If you want a marker commit anyway (e.g. to record the `.gitignore`
change):

```
$ git add .gitignore
$ git commit -m "Ignore scratch/ (chapter 01)"
```

Next: [02 — Targets](02-targets.md).
