# 09 — The `generate` command

[← 08 Rendering](08-render.md) · [README](README.md) · [10 The `take` command →](10-take.md)

## Goal

Every `src/core/*` module so far is a pure function: plain arguments in, a
value or a thrown error out. Nothing in `src/core` has ever read an
environment variable, parsed `process.argv`, or called `console.log`. That
was deliberate (see [`../design.md`](../design.md)) — it's what keeps those
modules trivially testable and reusable from a future web server.

Today you write the one file that's allowed to break that rule: `src/cli.ts`.
It wires `scope` → `context` → `llm` → `quiz` → `render` together into
`readback generate <repo> <target>`, reads the three `READBACK_*` env vars,
and turns thrown errors into clean one-line messages instead of stack traces.

You'll also build a tiny fake LLM server so you can run the whole pipeline
end-to-end without a real API key.

## Concepts

### `process.argv`

Node puts the full command line in `process.argv`: `argv[0]` is the path to
the `node` binary, `argv[1]` is the script being run, and the rest are your
actual arguments.

```js
// node script.js generate foo bar
process.argv; // ["/usr/bin/node", "/path/to/script.js", "generate", "foo", "bar"]
```

That's different from Python's `sys.argv` (script path at index 0, no
interpreter) or Go's `os.Args` (program path at index 0). The idiom here is
to destructure and throw away the first two:

```ts
const [, , command, ...rest] = process.argv;
```

### `node:util`'s `parseArgs`

Once you're past the command name (`generate` / `take`), you still have
`--questions 5 --out ./here repo-path target` to split into named options and
positional arguments. `node:util` has this built in — no dependency needed:

```ts
import { parseArgs } from "node:util";

const { values, positionals } = parseArgs({
  args: argv,
  allowPositionals: true,
  options: {
    questions: { type: "string" },
    out: { type: "string" },
  },
});
```

`values` has your declared options (as strings, since every option is
declared `type: "string"` here — `parseArgs` also supports `type: "boolean"`
for flags); `positionals` is everything else, in order.

> **Watch out:** `parseArgs` is stricter than it looks. Leave out
> `allowPositionals: true` and it throws the moment you pass anything that
> isn't a declared `--flag`. Every option needs an explicit `type` — there's
> no inferring it from how you call it. Passing `--questions 5` without
> declaring `questions` in `options` throws immediately with a clear message,
> which is actually a nice property: it fails loudly instead of silently
> ignoring a typo'd flag.

### Why only `cli.ts` touches the environment and the console

This isn't a style nitpick, it's the thing that makes the rest of the
codebase testable. Compare:

```ts
// core/llm.ts — bad: reads the environment itself
const apiKey = process.env.READBACK_API_KEY; // now untestable without env juggling

// core/llm.ts — what this project actually does
export async function complete(messages: ChatMessage[], options: CompleteOptions) {
  // options.apiKey is just a string the caller passed in
}
```

Every `core/*` function receives secrets and config as plain arguments. Only
`cli.ts` calls `process.env.READBACK_API_KEY`. That single rule is also what
would let a future web server pass a key that arrived per-request from a
browser, with zero changes to `core/`.

### Exit codes: `process.exitCode` vs. `process.exit()`

In Python you might `sys.exit(1)`; in Go, `os.Exit(1)`. Node has an
equivalent, `process.exit(code)`, but it's usually the wrong choice: it kills
the process **immediately**, even if there's buffered stdout or a pending
write that hasn't flushed yet. This project always does:

```ts
console.error("something went wrong");
process.exitCode = 1;
return; // let main() finish and Node exit naturally
```

`process.exitCode` just *records* what the exit code will be once the
process has nothing left to do — all pending I/O gets a chance to flush
first. You'll see why this matters even more once `take` holds an open
`readline` interface in chapter 10.

### The shebang and `npm link`

`src/cli.ts` starts with:

```ts
#!/usr/bin/env node
```

That's a shebang: on Unix, if a file is executable and starts with `#!`,
the kernel runs it with whatever interpreter follows. Combined with a `bin`
entry in `package.json`:

```json
"bin": { "readback": "./src/cli.ts" }
```

`npm link` creates a symlink from a `readback` command on your `PATH`
straight to `src/cli.ts` — no build step, no `dist/`. Node 24+'s native
`.ts` type stripping handles the rest.

> **Watch out:** `npm link` (global) needs write access to npm's global
> `node_modules`/`bin` directories. On a locked-down machine without `sudo`,
> point npm at a writable prefix instead:
> ```
> $ NPM_CONFIG_PREFIX=~/.npm-global npm link
> $ export PATH="$HOME/.npm-global/bin:$PATH"
> $ readback --help
> ```
> (`npm link --prefix <dir>` looks like the obvious flag, but in current npm
> it expects `<dir>` to *already be a package* to link, not a global prefix
> — it'll fail with an `ENOENT` reading `<dir>/package.json`. Setting
> `NPM_CONFIG_PREFIX` instead is the fallback that actually works.)

### A minimal `node:http` server

`scripts/fake-llm-server.ts` stands in for a real OpenAI-compatible endpoint
so you can test `generate` without a network call or an API key. This is
also a good look at what frameworks like Express do underneath:

```ts
import * as http from "node:http";

const server = http.createServer((req, res) => {
  if (req.method === "POST" && req.url === "/chat/completions") {
    res.writeHead(200, { "content-type": "application/json" });
    res.end(JSON.stringify({ /* ... */ }));
    return;
  }
  res.writeHead(404, { "content-type": "text/plain" });
  res.end("not found");
});

server.listen(8787);
```

No routing library, no middleware — just a function that gets called per
request. It's a dev/test script, not library code, so (like `cli.ts`) it's
fine for it to use `console`/`process` directly.

## Exercise

### `src/cli.ts`

No exports — it's an entry point, run for its side effects. Write:

- `async function main(): Promise<void>` — reads `process.argv`, dispatches
  on the command (`"generate"` → `runGenerate`, anything else → print usage
  and set `process.exitCode` to `0` for `undefined`/`--help`/`-h`, `1`
  otherwise).
- `function printUsage(): void` — prints the usage/targets/environment text
  with `console.log`.
- `async function runGenerate(argv: string[]): Promise<void>` —
  - Parse `--questions` (string) and `--out` (string) with `parseArgs`,
    `allowPositionals: true`.
  - Positionals are `[repo, target]`; if either is missing, print usage, set
    exit code `1`, return.
  - Parse `--questions` to a number (default `8`); if not a positive
    integer, print an error and set exit code `1`.
  - Default `--out` to `./readback-quiz-<ISO-timestamp-with-:-and-.-replaced-by-->`.
  - Read `READBACK_API_KEY`, `READBACK_MODEL`, `READBACK_BASE_URL` from
    `process.env`. If the key or model is missing, print an error and exit
    `1`. (Base URL is optional — `complete` already defaults it.)
  - Check `isGitRepo(repo)`; if false, error and exit `1`.
  - In a `try`: `parseTarget` → `resolveCommit` → `buildContext` (print each
    warning with `console.warn`) → `generateQuizWithRetry` (below) →
    `writeQuizFiles`, then print the three written paths. In the `catch`,
    print `generate failed: <message>` and set exit code `1`.
- `async function generateQuizWithRetry(input): Promise<Quiz>` — per the
  design's "retry once if too few questions survive citation validation"
  rule:
  - Loop `attempt` from 1 to 2.
  - Build the prompt, call `complete`, `parseLlmQuiz`, then
    `validateCitations`.
  - Print a `console.warn` for every dropped question, with its reason.
  - If `valid.length >= Math.min(3, questionCount)` **or** this was the
    second attempt: `assignIds` the survivors, `shuffleOptions` each one,
    and return the full `Quiz` object (`version: 1`, `createdAt`, `repo`,
    `target`, `commit`, `model`, `questions`).
  - Otherwise print a warning that you're retrying, and loop again.
- At the bottom of the file:
  ```ts
  main().catch((err: unknown) => {
    const message = err instanceof Error ? err.message : String(err);
    console.error(`unexpected error: ${message}`);
    process.exitCode = 1;
  });
  ```

Don't add a `"take"` branch yet — that's chapter 10.

### `scripts/fake-llm-server.ts`

A standalone script (not imported by anything). Reads `PORT` from the
environment (default `8787`). Serves `POST /chat/completions` with a canned
3-question reply shaped like a real OpenAI response (`choices[0].message
.content` is a JSON string matching `llmQuizSchema`). Make **one** of the
three questions cite a line that doesn't exist (e.g. line `9999`) — that's
what lets you watch `validateCitations` actually drop something during a
real run, instead of only trusting the unit tests.

### `package.json`

Add:
```json
"bin": { "readback": "./src/cli.ts" }
```
and a script to start the fake server:
```json
"fake-llm": "node scripts/fake-llm-server.ts"
```

## Tests

There's no unit test file for `cli.ts` in this project — it's thin
glue around code that's already tested, and talking to `process.env`/
`console` makes it awkward to unit-test anyway. Instead, you verify it the
way you'd actually use it: run it end-to-end against a throwaway repo and a
fake LLM.

1. **Make a throwaway git repo** with one real change:
   ```
   $ mkdir /tmp/readback-demo && cd /tmp/readback-demo
   $ git init -q --initial-branch=main
   $ git config user.email you@example.com && git config user.name You
   $ cat > math.ts <<'EOF'
   export function add(a: number, b: number): number {
     return a + b;
   }
   EOF
   $ git add math.ts && git commit -q -m "add"
   $ BASE=$(git rev-parse HEAD)
   $ cat >> math.ts <<'EOF'

   export function subtract(a: number, b: number): number {
     return a - b;
   }
   EOF
   $ git add math.ts && git commit -q -m "add subtract"
   $ HEAD=$(git rev-parse HEAD)
   ```

2. **Terminal 1, from your `readback` checkout**, start the fake server:
   ```
   $ npm run fake-llm
   > readback@0.1.0 fake-llm
   > node scripts/fake-llm-server.ts

   fake LLM server listening on http://localhost:8787
   ```
   Leave it running.

3. **Terminal 2**, run `generate` against the throwaway repo, pointed at the
   fake server:
   ```
   $ READBACK_API_KEY=dummy READBACK_MODEL=fake READBACK_BASE_URL=http://localhost:8787 \
     npm start -- generate /tmp/readback-demo "$BASE..$HEAD"
   ```
   Expected output (your commit SHAs will differ):
   ```
   warning: dropped a question (citation math.ts:9999-9999 is out of range (file has 8 lines))
   warning: only 2 question(s) survived validation, retrying once
   warning: dropped a question (citation math.ts:9999-9999 is out of range (file has 8 lines))
   Wrote 2 question(s) to:
     ./readback-quiz-<timestamp>/quiz.json
     ./readback-quiz-<timestamp>/quiz.md
     ./readback-quiz-<timestamp>/answers.md
   ```
   That's `generateQuizWithRetry` doing exactly what it's supposed to: the
   fake server's deliberately-bad citation gets dropped both attempts (it's
   a canned reply, so retrying doesn't change it), but 2 surviving questions
   is enough once attempt 2 is the last one allowed. Open the written
   `quiz.md`/`answers.md` and check they look like chapter 08's render
   tests, just for real files.

4. **Loading secrets from a file instead of typing them every time:** Node
   can read a dotenv-style file natively, no package needed:
   ```
   $ cat > .env <<'EOF'
   READBACK_API_KEY=dummy
   READBACK_MODEL=fake
   READBACK_BASE_URL=http://localhost:8787
   EOF
   $ node --env-file=.env src/cli.ts generate /tmp/readback-demo "$BASE..$HEAD"
   ```
   Same output as above. `.env` is already in `.gitignore` from chapter 00 —
   never commit it. (Chapter 11 covers using this with a *real* key.)

## Hints

<details><summary>Hint 1</summary>

Start with just the dispatch and usage text — get `npm start`, `npm start --
--help`, and `npm start -- bogus` printing usage with the right exit codes
before writing any of `runGenerate`.

</details>

<details><summary>Hint 2</summary>

`parseArgs`'s `values.questions` is `string | undefined`. Convert with
`Number(values.questions)` and check `Number.isInteger(...) && ... >= 1` —
`Number("abc")` is `NaN`, and `Number.isInteger(NaN)` is `false`, so bad
input is rejected for free.

</details>

<details><summary>Hint 3</summary>

`generateQuizWithRetry` always returns inside the loop by the time
`attempt === 2`, but TypeScript can't prove that a `for` loop always
`return`s — it still expects a code path after the loop. A `throw new
Error("unreachable")` after it satisfies the compiler without lying about
behavior.

</details>

<details><summary>Hint 4</summary>

For the fake server, build the canned JSON body once as a top-level
constant, then `JSON.stringify` it inside the request handler. Keep the
`content` field a *string* (the model always replies with a string you then
`JSON.parse`) — don't nest a real object there.

</details>

## Solution

<details><summary>Solution</summary>

`src/cli.ts`:

```ts
#!/usr/bin/env node
// cli.ts — the ONLY file in this project allowed to read process.env, read
// process.argv, or print to the console. Everything it does is wire together
// the pure functions in src/core/* and report the result.

import { parseArgs } from "node:util";

import { parseTarget, resolveCommit, isGitRepo } from "./core/scope.ts";
import { buildContext } from "./core/context.ts";
import { complete } from "./core/llm.ts";
import { buildPrompt, parseLlmQuiz, validateCitations, assignIds } from "./core/quiz.ts";
import type { Quiz } from "./core/quiz.ts";
import { shuffleOptions, writeQuizFiles } from "./core/render.ts";

const DEFAULT_QUESTIONS = 8;
// Retry once if too many questions get dropped by citation validation.
const MIN_SURVIVING_QUESTIONS = 3;

async function main(): Promise<void> {
  const [, , command, ...rest] = process.argv;

  if (command === "generate") {
    await runGenerate(rest);
  } else {
    printUsage();
    process.exitCode = command === undefined || command === "--help" || command === "-h" ? 0 : 1;
  }
}

function printUsage(): void {
  console.log(
    [
      "Usage:",
      "  readback generate <repo-path> <target> [--questions N] [--out DIR]",
      "",
      "Targets:",
      '  "a1b2c3..d4e5f6"   changes between two commits',
      '  "main...feature"   changes on feature since it left main',
      '  "path:src/auth/"   files under a path at HEAD',
      "",
      "Environment:",
      "  READBACK_API_KEY   required, your LLM API key",
      "  READBACK_MODEL     required, model id",
      "  READBACK_BASE_URL  optional, default https://api.openai.com/v1",
    ].join("\n"),
  );
}

async function runGenerate(argv: string[]): Promise<void> {
  const { values, positionals } = parseArgs({
    args: argv,
    allowPositionals: true,
    options: {
      questions: { type: "string" },
      out: { type: "string" },
    },
  });

  const [repo, target] = positionals;
  if (repo === undefined || target === undefined) {
    printUsage();
    process.exitCode = 1;
    return;
  }

  const questionCount = values.questions !== undefined ? Number(values.questions) : DEFAULT_QUESTIONS;
  if (!Number.isInteger(questionCount) || questionCount < 1) {
    console.error(`--questions must be a positive integer, got "${values.questions}"`);
    process.exitCode = 1;
    return;
  }

  const timestamp = new Date().toISOString().replace(/[:.]/g, "-");
  const outDir = values.out ?? `./readback-quiz-${timestamp}`;

  // Only cli.ts reads secrets from the environment. Everything downstream
  // receives them as plain function arguments.
  const apiKey = process.env.READBACK_API_KEY;
  const model = process.env.READBACK_MODEL;
  const baseUrl = process.env.READBACK_BASE_URL;
  if (apiKey === undefined) {
    console.error("READBACK_API_KEY is not set");
    process.exitCode = 1;
    return;
  }
  if (model === undefined) {
    console.error("READBACK_MODEL is not set");
    process.exitCode = 1;
    return;
  }

  if (!(await isGitRepo(repo))) {
    console.error(`${repo} is not a git repository`);
    process.exitCode = 1;
    return;
  }

  try {
    const scope = await parseTarget(repo, target);
    const commit = await resolveCommit(repo, scope);

    const { text: context, warnings } = await buildContext(repo, scope);
    for (const warning of warnings) console.warn(`warning: ${warning}`);

    const quiz = await generateQuizWithRetry({
      repo,
      target,
      commit,
      model,
      apiKey,
      baseUrl,
      scope,
      context,
      questionCount,
    });

    const written = await writeQuizFiles(quiz, outDir);
    console.log(`Wrote ${quiz.questions.length} question(s) to:`);
    console.log(`  ${written.quizPath}`);
    console.log(`  ${written.quizMdPath}`);
    console.log(`  ${written.answersMdPath}`);
  } catch (err: unknown) {
    const message = err instanceof Error ? err.message : String(err);
    console.error(`generate failed: ${message}`);
    process.exitCode = 1;
  }
}

async function generateQuizWithRetry(input: {
  repo: string;
  target: string;
  commit: string;
  model: string;
  apiKey: string;
  baseUrl: string | undefined;
  scope: Awaited<ReturnType<typeof parseTarget>>;
  context: string;
  questionCount: number;
}): Promise<Quiz> {
  const { repo, target, commit, model, apiKey, baseUrl, scope, context, questionCount } = input;

  // One attempt, then (per design) a single retry if too few questions
  // survive citation validation.
  for (let attempt = 1; attempt <= 2; attempt++) {
    const messages = buildPrompt(scope, context, questionCount);
    const raw = await complete(messages, { apiKey, model, baseUrl });
    const llmQuiz = parseLlmQuiz(raw);
    const { valid, dropped } = await validateCitations(repo, scope, llmQuiz.questions);

    for (const d of dropped) {
      console.warn(`warning: dropped a question (${d.reason})`);
    }

    if (valid.length >= Math.min(MIN_SURVIVING_QUESTIONS, questionCount) || attempt === 2) {
      const questions = assignIds(valid).map((q) => shuffleOptions(q));
      return {
        version: 1,
        createdAt: new Date().toISOString(),
        repo,
        target,
        commit,
        model,
        questions,
      };
    }

    console.warn(`warning: only ${valid.length} question(s) survived validation, retrying once`);
  }

  // Unreachable (the loop always returns by attempt 2), but keeps tsc happy
  // about every code path returning a value.
  throw new Error("unreachable");
}

main().catch((err: unknown) => {
  const message = err instanceof Error ? err.message : String(err);
  console.error(`unexpected error: ${message}`);
  process.exitCode = 1;
});
```

`scripts/fake-llm-server.ts`:

```ts
// fake-llm-server.ts — a tiny stand-in for an OpenAI-compatible endpoint, so
// `readback generate` can be exercised end-to-end without a real API key or
// network access. Point READBACK_BASE_URL at this server's address.
//
// This is a dev/test script, not part of the library, so it's fine for it to
// use console and process directly (same rules as cli.ts apply to it).

import * as http from "node:http";

const PORT = Number(process.env.PORT ?? 8787);

// A canned quiz reply. It deliberately includes one citation that does NOT
// exist (line 9999) so you can see `generate` warn and drop that question
// during a real end-to-end run, alongside two that validate fine.
const CANNED_QUESTIONS = [
  {
    kind: "why",
    question: "Why does `add` take two parameters instead of an array?",
    options: [
      { id: "a", text: "Because the call sites always pass exactly two numbers", why: "Right: that's the only way `add` is called in this file." },
      { id: "b", text: "Because arrays are slower in JavaScript", why: "Wrong: not why this function looks the way it does." },
      { id: "c", text: "Because TypeScript doesn't support array parameters", why: "Wrong: TypeScript supports array parameters fine." },
      { id: "d", text: "Because of a historical bug fix", why: "Wrong: no such history here." },
    ],
    correct: "a",
    citations: [{ path: "math.ts", start: 1, end: 3 }],
  },
  {
    kind: "predict",
    question: "What does `add(2, 3)` return?",
    options: [
      { id: "a", text: "23", why: "Wrong: that would be string concatenation, but both args are numbers." },
      { id: "b", text: "5", why: "Right: `add` returns a + b." },
      { id: "c", text: "undefined", why: "Wrong: the function always returns a value." },
      { id: "d", text: "An error", why: "Wrong: nothing here throws." },
    ],
    correct: "b",
    citations: [{ path: "math.ts", start: 2, end: 2 }],
  },
  {
    // This one has a bogus citation on purpose, to show citation validation
    // dropping a question during a real `generate` run.
    kind: "bug",
    question: "What is risky about this made-up citation?",
    options: [
      { id: "a", text: "Nothing, it's fine", why: "n/a" },
      { id: "b", text: "It cites a line that doesn't exist", why: "n/a" },
      { id: "c", text: "It's too long", why: "n/a" },
      { id: "d", text: "It's not risky", why: "n/a" },
    ],
    correct: "b",
    citations: [{ path: "math.ts", start: 9999, end: 9999 }],
  },
];

const server = http.createServer((req, res) => {
  if (req.method === "POST" && req.url === "/chat/completions") {
    const body = JSON.stringify({
      choices: [
        {
          message: {
            content: JSON.stringify({ questions: CANNED_QUESTIONS }),
          },
        },
      ],
    });
    res.writeHead(200, { "content-type": "application/json" });
    res.end(body);
    return;
  }

  res.writeHead(404, { "content-type": "text/plain" });
  res.end("not found");
});

server.listen(PORT, () => {
  console.log(`fake LLM server listening on http://localhost:${PORT}`);
});
```

</details>

## Watch out

> **Watch out:** `parseArgs` throws immediately on an undeclared `--flag` or
> a positional when `allowPositionals` isn't set — a loud failure, not a
> silent ignore. Declare every option your command accepts, with an explicit
> `type`.

> **Watch out:** `execFile` (not `exec`) throughout `scope.ts` is a
> deliberate security choice, not style: `exec` runs through a shell, so a
> branch name or path containing `;`, `` ` ``, or `$()` could do something
> you didn't intend. `generate`'s `repo`/`target` arguments ultimately flow
> into git argument arrays, never a concatenated shell string.

> **Watch out:** `npm link` needs write access to npm's *global*
> `node_modules`/`bin`. Without `sudo`, set `NPM_CONFIG_PREFIX` to a writable
> directory first (see Concepts above) — `npm link --prefix <dir>` is not
> the same thing and will fail with an `ENOENT`.

## Checkpoint

```
$ npm run typecheck
$ npm test
```
All existing unit tests should still pass (`cli.ts` has none of its own).
Then run the end-to-end check from the Tests section above and confirm your
output matches (modulo real SHAs and the timestamp in the output directory
name).

Optionally, confirm the shebang + `bin` entry work:
```
$ NPM_CONFIG_PREFIX=~/.npm-global npm link
$ PATH="$HOME/.npm-global/bin:$PATH" readback --help
```

Suggested commit:
```
$ git add src/cli.ts scripts/fake-llm-server.ts package.json
$ git commit -m "Add generate command and fake LLM server"
```

[← 08 Rendering](08-render.md) · [README](README.md) · [10 The `take` command →](10-take.md)
