# Build Readback: a TypeScript tutorial

This tutorial teaches TypeScript and Node.js by building the Readback CLI
described in [`../design.md`](../design.md), one module at a time.

It assumes you can already program in another language (Python, Go, C#, ...)
but are new to JavaScript, TypeScript and Node.

## How each chapter works

1. **Goal**: what you'll build and why it exists in Readback.
2. **Concepts**: the TypeScript/Node ideas you need, with small examples and
   comparisons to languages you may know.
3. **Exercise**: the file to create, the exact functions and types to export,
   and how they must behave.
4. **Tests**: a test file you copy in. The tests are the specification:
   you're done when they pass.
5. **Hints**: collapsed. Open them one at a time when stuck.
6. **Solution**: collapsed. A working reference version. Try first; compare after.
7. **Watch out**: real surprises that came up while building the reference.
8. **Checkpoint**: commands to run and what you should see, plus a suggested
   commit.

You write the code in this repository (`src/`, `test/`, `package.json` at the
repo root), so by the end the CLI here is the one you built.

## Chapters

| #  | Chapter | You build | You learn |
|----|---------|-----------|-----------|
| 00 | [Setup](00-setup.md) | `package.json`, `tsconfig.json`, first script and test | Installing Node, npm, running `.ts` directly, type checking, `node:test` |
| 01 | [TypeScript crash course](01-typescript-crash-course.md) | Scratch files only | Types, inference, unions, `null`/`undefined`, `any` vs `unknown`, modules, `async`/`await` |
| 02 | [Targets](02-targets.md) | `src/core/scope.ts` (part 1) | Discriminated unions, narrowing, custom errors, running git safely |
| 03 | [Asking git](03-git.md) | `src/core/scope.ts` (part 2) | Promises in practice, Buffers vs strings, test fixtures |
| 04 | [Building context](04-context.md) | `src/core/context.ts` | Strings and arrays, `Set`, sorting, `import type` |
| 05 | [Quiz schemas](05-quiz-schemas.md) | `src/core/quiz.ts` (part 1) | zod, `z.infer`, runtime vs compile-time types |
| 06 | [Calling the LLM](06-llm.md) | `src/core/llm.ts` | `fetch`, narrowing `unknown`, injecting dependencies for tests, keeping secrets out of errors |
| 07 | [Checking citations](07-citations.md) | `src/core/quiz.ts` (part 2) | `Map`, sequential `await`, untrusted vs trusted data |
| 08 | [Rendering](08-render.md) | `src/core/render.ts` | Immutability, Fisher–Yates, injectable randomness, `fs/promises` |
| 09 | [The `generate` command](09-generate.md) | `src/cli.ts` (part 1), fake LLM server | `parseArgs`, `process.env`, exit codes, `npm link` |
| 10 | [The `take` command](10-take.md) | `src/core/take.ts`, `src/cli.ts` (part 2) | Type guards, a first generic, `readline`, injectable clock |
| 11 | [What next](11-what-next.md) | Running on a real repo | Using a real LLM key safely, ideas for extending |

Do the chapters in order: each one builds on the previous one.

## Conventions

- Shell commands start with `$`. Run them from the repository root unless
  the chapter says otherwise.
- `// ...` inside a code block means "the code you already have stays here".
- Versions are pinned. Use exactly the versions given in chapter 00.
