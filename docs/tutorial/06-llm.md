# 06 — Calling the LLM

[← 05. Quiz schemas](05-quiz-schemas.md) · [07. Checking citations →](07-citations.md)

## Goal

Build `src/core/llm.ts`: the one function in this whole program that talks to
the network. It sends the prompt built in chapter 05 to an OpenAI-compatible
chat-completions endpoint and hands back the model's raw reply text (still
just a string — chapter 05's `parseLlmQuiz` is what turns it into data).

This is a deliberately small, boring file. It does HTTP and nothing else:
no parsing, no validation, no `process.env`. That's on purpose — see
`docs/design.md`'s "Access and secrets" section, which says core code
receives the API key as a function argument and only `cli.ts` is allowed to
read the environment.

## Concepts

### What the request actually looks like

Readback talks to any "OpenAI-compatible" endpoint — OpenAI itself, or a
local fake server, or another provider that copies OpenAI's API shape. The
request is a plain POST with a JSON body:

```
POST https://api.openai.com/v1/chat/completions
Authorization: Bearer sk-...
Content-Type: application/json

{
  "model": "gpt-4o-mini",
  "messages": [
    { "role": "system", "content": "..." },
    { "role": "user", "content": "..." }
  ],
  "response_format": {
    "type": "json_schema",
    "json_schema": { "name": "quiz", "schema": { ... }, "strict": true }
  }
}
```

`messages` is exactly the `ChatMessage[]` that `buildPrompt` returns from
chapter 05. `response_format` is how you ask an OpenAI-compatible model to
reply with JSON matching a specific shape instead of free text — useful,
because the whole point of this call is to get back something `parseLlmQuiz`
can parse.

### `fetch` and `Response`

Node has `fetch` built in globally — no `node-fetch`, no import, same API as
the browser. In Python terms it's roughly `requests`/`httpx`; in Go, `net/http`'s
client; in C#, `HttpClient`. You call it, `await` the promise it returns, and
get back a `Response`:

```ts
const res = await fetch(url, { method: "POST", headers: {...}, body: "..." });
res.ok;           // boolean: status in 200-299
res.status;       // number
await res.text(); // body as a string
await res.json(); // body parsed as JSON — typed `any` in older TS, `unknown`-ish here
```

Nothing throws just because the server replied with a 404 or 500 — `fetch`
only rejects on *network* failure (DNS, connection refused, timeout). You
have to check `res.ok` yourself and turn a bad status into your own error.

### `await res.json()` is not safe to trust

`res.json()` parses *whatever bytes the server sent* as JSON. TypeScript has
no way to know the shape of someone else's HTTP response, so the result is
effectively `unknown` (in practice typed `any`, but treat it as `unknown` —
see chapter 01/02 on why `unknown` forces you to check before use). You
can't write `data.choices[0].message.content` and trust it compiles safely;
you have to narrow first.

That's what `extractContent` does in this chapter: a chain of `typeof`/`in`
checks that only return a `string` once every step of the shape has been
confirmed. It's tedious by hand — which is exactly the argument for zod.
Chapter 05 already validates the model's *quiz content* with a schema;
`extractContent` is doing the same kind of job (turn `unknown` into something
typed) for the outer HTTP envelope, by hand, so you feel the difference. If
you ever find yourself writing a third or fourth manual narrowing function
like this one, that's the signal to reach for a schema library instead.

### `z.toJSONSchema` and structured output

Zod 4 can convert a schema you already wrote into plain [JSON Schema](https://json-schema.org/):

```ts
const jsonSchema = z.toJSONSchema(llmQuizSchema);
```

That's the schema we hand the model via `response_format`. One schema,
defined once in chapter 05, used both to ask the model for the right shape
*and* to validate what it sends back. No second hand-written copy to let
drift out of sync.

### Dependency injection instead of a mocking framework

Python's `unittest.mock.patch` and C#'s mocking frameworks (Moq, NSubstitute)
work by intercepting a call after the fact — patching a module attribute or
generating a proxy from an interface. JS/TS code in this style usually does
it simpler: make the dependency a parameter with a default.

```ts
export type CompleteOptions = {
  // ...
  fetchImpl?: typeof fetch;
};

export async function complete(messages: ChatMessage[], options: CompleteOptions) {
  const fetchImpl = options.fetchImpl ?? fetch;
  // ...
}
```

`typeof fetch` means "a function with the same call signature as the global
`fetch`" — not a named interface you have to declare, just "shaped like this
existing thing". Tests pass in a fake function with that same shape and
TypeScript is satisfied with zero ceremony; production code just omits the
option and gets the real `fetch`. No patching, no magic, no framework.

### Secret discipline

The API key must reach exactly one place: the `Authorization` header. It
must never end up in a thrown error message, a log line, or a URL — all
things that get copied into bug reports, CI logs, or shell history.
`AGENTS.md`'s security decisions are explicit about this for the whole
project: the LLM key is the only secret this app handles, it's never
persisted, and it must never leak into logs or error output. This file is
the one place in the codebase where the key exists as a value in memory, so
it's the one place that discipline has to be airtight. Notice in the
exercise below that both error paths (`fetchImpl` throwing, and a non-2xx
response) build their message from the *response*, never from `options`.

## Exercise

Create `src/core/llm.ts` exporting:

```ts
export type CompleteOptions = {
  apiKey: string;
  model: string;
  baseUrl?: string;         // defaults to https://api.openai.com/v1
  fetchImpl?: typeof fetch; // defaults to the global fetch
};

export class LlmError extends Error {}

export async function complete(
  messages: ChatMessage[],
  options: CompleteOptions,
): Promise<string>;
```

(`ChatMessage` is the type you already exported from `src/core/quiz.ts` in
chapter 05 — import it with `import type`.)

Required behaviour:

- POST to `` `${baseUrl}/chat/completions` `` with:
  - `authorization: Bearer <apiKey>`
  - `content-type: application/json`
  - a JSON body: `{ model, messages, response_format: { type: "json_schema", json_schema: { name: "quiz", schema: <JSON Schema from llmQuizSchema>, strict: true } } }`
- If the fetch call itself throws (network failure), catch it and throw
  `LlmError` with a message that includes the underlying error's message —
  never the API key.
- If the response is not `ok`, read the body as text, truncate it to 500
  characters, and throw `LlmError` including the status code and the
  (truncated) body.
- On success, parse the body as JSON and pull out
  `data.choices[0].message.content` *without* trusting its shape —
  write a small helper (`extractContent`, not exported) that narrows
  `unknown` step by step and returns `string | null`.
- If that content can't be found, throw `LlmError` saying so.
- Otherwise return the content string.

## Tests

Create `test/llm.test.ts`:

```ts
// llm.test.ts — exercises complete() against a fake `fetch`, so we never
// make a real network call in tests. The fake returns a canned
// chat-completions-shaped response.

import { test } from "node:test";
import assert from "node:assert/strict";
import { complete, LlmError } from "../src/core/llm.ts";

/** Build a fake `fetch` that always returns `body` with the given status. */
function fakeFetch(status: number, body: unknown): typeof fetch {
  return (async () =>
    new Response(typeof body === "string" ? body : JSON.stringify(body), {
      status,
      headers: { "content-type": "application/json" },
    })) as typeof fetch;
}

test("complete returns the message content on success", async () => {
  const canned = {
    choices: [{ message: { content: '{"questions":[]}' } }],
  };

  const result = await complete([{ role: "user", content: "hi" }], {
    apiKey: "sk-secret",
    model: "test-model",
    fetchImpl: fakeFetch(200, canned),
  });

  assert.equal(result, '{"questions":[]}');
});

test("complete sends the API key as a bearer token, not in the body", async () => {
  let seenAuth: string | null = null;
  let seenBody: string | null = null;

  const capturingFetch = (async (_url: string, init?: RequestInit) => {
    seenAuth = (init?.headers as Record<string, string>)?.authorization ?? null;
    seenBody = typeof init?.body === "string" ? init.body : null;
    return new Response(JSON.stringify({ choices: [{ message: { content: "{}" } }] }), { status: 200 });
  }) as typeof fetch;

  await complete([{ role: "user", content: "hi" }], {
    apiKey: "sk-secret-value",
    model: "test-model",
    fetchImpl: capturingFetch,
  });

  assert.equal(seenAuth, "Bearer sk-secret-value");
  assert.notEqual(seenBody, null);
  assert.ok(!(seenBody ?? "").includes("sk-secret-value"));
});

test("complete throws LlmError with status + truncated body, never the key", async () => {
  await assert.rejects(
    () =>
      complete([{ role: "user", content: "hi" }], {
        apiKey: "sk-should-not-leak",
        model: "test-model",
        fetchImpl: fakeFetch(401, "unauthorized: bad key"),
      }),
    (err: unknown) => {
      assert.ok(err instanceof LlmError);
      assert.match(err.message, /401/);
      assert.match(err.message, /unauthorized/);
      assert.ok(!err.message.includes("sk-should-not-leak"));
      return true;
    },
  );
});

test("complete truncates very long error bodies", async () => {
  const longBody = "x".repeat(2000);
  await assert.rejects(
    () =>
      complete([{ role: "user", content: "hi" }], {
        apiKey: "sk-key",
        model: "test-model",
        fetchImpl: fakeFetch(500, longBody),
      }),
    (err: unknown) => {
      assert.ok(err instanceof LlmError);
      assert.ok(err.message.length < longBody.length);
      return true;
    },
  );
});
```

## Hints

<details><summary>Hint 1</summary>

`options.baseUrl ?? DEFAULT_BASE_URL` and `options.fetchImpl ?? fetch` are
both the same pattern: "use the caller's value if given, otherwise my
default" — the nullish-coalescing operator `??` from chapter 01.

</details>

<details><summary>Hint 2</summary>

`extractContent` is a chain of early returns, not one big `&&`. Something
like:

```ts
function extractContent(data: unknown): string | null {
  if (typeof data !== "object" || data === null || !("choices" in data)) return null;
  const choices = (data as { choices: unknown }).choices;
  if (!Array.isArray(choices) || choices.length === 0) return null;
  // ... repeat for choices[0], .message, .content
}
```

Each `as` cast is only safe *because* the line above it already checked
enough to justify it — cast to the minimum shape you need for the next
`in` check, not to the final type.

</details>

<details><summary>Hint 3</summary>

For the "fetch itself throws" case, wrap only the `fetchImpl(...)` call in
`try`/`catch`, not the rest of the function — you want the later `LlmError`
throws (bad status, missing content) to be regular throws, not re-caught.

</details>

<details><summary>Hint 4</summary>

String slicing for truncation: `text.length > 500 ? \`${text.slice(0, 500)}...\` : text`.

</details>

## Solution

<details><summary>Solution</summary>

```ts
// llm.ts — the one network call in this whole program: ask an
// OpenAI-compatible chat-completions endpoint to write quiz JSON.
//
// `fetchImpl` defaults to the real global `fetch`, but callers (tests, or a
// future web server) can inject any function with the same shape — that's
// how test/llm.test.ts exercises this file without touching the network.

import { z } from "zod";
import { llmQuizSchema } from "./quiz.ts";
import type { ChatMessage } from "./quiz.ts";

export type CompleteOptions = {
  apiKey: string;
  model: string;
  /** Defaults to the real OpenAI API; override for other OpenAI-compatible
   * providers, or for a local fake server in tests/dev. */
  baseUrl?: string;
  /** Injectable for tests; defaults to the global `fetch`. */
  fetchImpl?: typeof fetch;
};

const DEFAULT_BASE_URL = "https://api.openai.com/v1";

/** Thrown for any non-2xx response. Never includes the API key. */
export class LlmError extends Error {}

// How much of a failing response body to quote in the error message — enough
// to debug, not so much we dump megabytes of HTML error pages.
const MAX_ERROR_BODY_CHARS = 500;

/**
 * Send `messages` to the model and return its reply text (expected to be
 * JSON matching `llmQuizSchema` — the caller still parses it with
 * `parseLlmQuiz`, this function only talks HTTP).
 */
export async function complete(messages: ChatMessage[], options: CompleteOptions): Promise<string> {
  const baseUrl = options.baseUrl ?? DEFAULT_BASE_URL;
  const fetchImpl = options.fetchImpl ?? fetch;

  // Ask for structured output matching our schema where the provider
  // supports it (OpenAI's `json_schema` response format). z.toJSONSchema
  // converts our zod schema to plain JSON Schema so there's exactly one
  // definition of "what a quiz looks like" in this codebase.
  const jsonSchema = z.toJSONSchema(llmQuizSchema);

  const body = {
    model: options.model,
    messages,
    response_format: {
      type: "json_schema",
      json_schema: {
        name: "quiz",
        schema: jsonSchema,
        strict: true,
      },
    },
  };

  let response: Response;
  try {
    response = await fetchImpl(`${baseUrl}/chat/completions`, {
      method: "POST",
      headers: {
        "content-type": "application/json",
        // The key only ever appears here, in a header, never in a thrown
        // error message below.
        authorization: `Bearer ${options.apiKey}`,
      },
      body: JSON.stringify(body),
    });
  } catch (err: unknown) {
    const message = err instanceof Error ? err.message : String(err);
    throw new LlmError(`request to ${baseUrl}/chat/completions failed: ${message}`);
  }

  if (!response.ok) {
    const text = await response.text().catch(() => "");
    const truncated = text.length > MAX_ERROR_BODY_CHARS ? `${text.slice(0, MAX_ERROR_BODY_CHARS)}...` : text;
    throw new LlmError(`LLM request failed with status ${response.status}: ${truncated}`);
  }

  const data = (await response.json()) as unknown;
  const content = extractContent(data);
  if (content === null) {
    throw new LlmError("LLM response did not contain a chat completion message");
  }
  return content;
}

/** Pull `choices[0].message.content` out of an OpenAI-shaped response body,
 * without trusting its shape (it's `unknown` until we check). */
function extractContent(data: unknown): string | null {
  if (typeof data !== "object" || data === null || !("choices" in data)) return null;
  const choices = (data as { choices: unknown }).choices;
  if (!Array.isArray(choices) || choices.length === 0) return null;

  const first = choices[0] as unknown;
  if (typeof first !== "object" || first === null || !("message" in first)) return null;
  const message = (first as { message: unknown }).message;
  if (typeof message !== "object" || message === null || !("content" in message)) return null;
  const content = (message as { content: unknown }).content;
  return typeof content === "string" ? content : null;
}
```

</details>

## Watch out

> **Watch out:** `z.toJSONSchema` is a **zod-4-only** API — it didn't exist
> in zod 3, which needed a separate `zod-to-json-schema` package. This
> tutorial pins zod `4.6.7` in `package.json` (chapter 00); if you search for
> older zod examples online and can't find this function, that's why.

> **Watch out:** `fetch` does not throw on HTTP error statuses (404, 401,
> 500, ...) — only on network-level failures. Forgetting to check
> `response.ok` is the single most common bug with `fetch`, in any language
> that has an equivalent (`requests.get(...).raise_for_status()` exists in
> Python precisely because `requests` has the same non-throwing default).

> **Watch out (security):** never interpolate `options.apiKey` into a string
> that isn't the `Authorization` header — not in a template literal passed to
> `LlmError`, not in a `console.log` during debugging. Per `AGENTS.md`, the
> LLM key is the only secret this app handles and it must never be logged or
> persisted.

## Checkpoint

```
$ npx tsc
$ node --test test/llm.test.ts
```

Expected: no type errors, and a summary like:

```
ℹ tests 4
ℹ pass 4
ℹ fail 0
```

Then run the whole suite to make sure nothing earlier broke:

```
$ npm test
```

Expected: `ℹ tests 22`, `ℹ pass 22`, `ℹ fail 0`.

Commit:

```
$ git add src/core/llm.ts test/llm.test.ts
$ git commit -m "feat: call the LLM (src/core/llm.ts)"
```

Next: [07. Checking citations](07-citations.md) — close the loop by
validating that the model's citations actually point at real code.
