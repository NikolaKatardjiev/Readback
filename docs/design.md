# Readback — MVP design (CLI)

Status: agreed design, not yet implemented. Scope: **CLI only**. The web UI
and container come later and are out of scope here, except where the design
leaves room for them.

## Goal

Generate a multiple-choice comprehension quiz about a narrow slice of a Git
repository, then let the user *take* it in a way that rewards understanding
over recognition. One LLM call per quiz; taking a quiz needs no network.

## Commands

```
readback generate <repo-path> <target> [--questions N] [--out DIR]
readback take <quiz.json>
```

### Targets

| Target              | Example               | Meaning                                   |
|---------------------|-----------------------|-------------------------------------------|
| Revision range      | `a1b2c3..d4e5f6`      | Changes between two commits               |
|                     | `main...feature`      | Changes on `feature` since it left `main` |
| Path                | `path:src/auth/`      | Files under a path at `HEAD`              |

Internally every target is parsed into one of two shapes:

```ts
type Scope =
  | { kind: "range"; base: string; head: string }
  | { kind: "path"; path: string; rev: string }  // rev defaults to HEAD
```

Everything after the parser only sees a `Scope`.

**Not in the MVP: `mr:` / `pr:` targets.** Readback never talks to a Git
remote. To quiz a merge/pull request, fetch it yourself first:

```
git fetch origin pull/45/head:pr-45                 # GitHub
git fetch origin merge-requests/123/head:mr-123     # GitLab
readback generate . main...pr-45
```

*Adding it later:* an `mr:123` / `pr:45` target would run `git fetch` (with the
user's own git credentials) and resolve to a `range` scope. Only the parser
changes.

## Access and secrets

- **LLM API key** — the only secret Readback handles.
  - Read from the `READBACK_API_KEY` env var, **never** a CLI flag (flags leak
    into shell history and `ps`).
  - Never written to disk, never logged, never included in error messages.
  - Core code receives the key as a **function argument**; only `cli.ts` reads
    the environment. (This is what lets a future web server pass a key that
    arrived per-request from the browser.)
- **Repository access is not Readback's job.** The user points it at a local
  clone they can already read. Readback runs only read-only, local git
  commands (`diff`, `show`, `ls-files`, `merge-base`, `rev-parse`) and fails
  with a clear message if the path is not a repo or a revision is unknown.

## Pipeline (`generate`)

```
target ─► scope ─► context ─► llm ─► quiz (validate) ─► render
          (git)    (text)    (JSON)                     (files)
```

1. **scope** — parse the target; ask git which files/hunks are involved.
2. **context** — build the text sent to the model:
   - range: the unified diff **plus** the full post-change content of each
     touched file;
   - path: the content of each file under the path.
   - Every line is prefixed with its line number so the model can cite.
   - Skip binary, vendored and generated files (lockfiles, `dist/`, minified).
   - Enforce a size budget (characters as a cheap proxy for tokens); if
     exceeded, drop the largest unchanged-context files first and **warn**.
3. **llm** — one request to an OpenAI-compatible chat-completions endpoint,
   asking for JSON matching the quiz schema (structured output where the
   provider supports it).
4. **quiz** — parse with zod, then validate citations against the real files
   (see below). Invalid questions are dropped; if too few remain, retry once.
5. **render** — our code (not the model) shuffles options and writes files.

## Quiz data

`quiz.json` is the source of truth; Markdown files are generated views of it.

```ts
type Quiz = {
  version: 1;
  createdAt: string;            // ISO timestamp
  repo: string;                 // repo path as given
  target: string;               // target as given, e.g. "main...feature"
  commit: string;               // resolved HEAD/head SHA, for reproducibility
  model: string;
  questions: Question[];
};

type Question = {
  id: number;
  kind: "failure" | "why" | "predict" | "bug";
  question: string;
  options: { id: "a" | "b" | "c" | "d"; text: string; why: string }[]; // exactly 4
  correct: "a" | "b" | "c" | "d";
  citations: { path: string; start: number; end: number }[];          // at least 1
};
```

Question kinds:

| kind      | Asks                                         |
|-----------|----------------------------------------------|
| `failure` | What happens if X fails / throws / is empty? |
| `why`     | Why is this line/branch/check needed?        |
| `predict` | What does this produce for input Y?          |
| `bug`     | What is wrong with / risky about this code?  |

### Rules the prompt enforces

- Every wrong option is a **plausible misreading** of the code, not a joke.
- Every option has a `why` explaining why it is right or wrong.
- Every question cites the lines that justify the correct answer.

### Rules our code enforces

- **Citation validation**: the path exists in the scoped revision;
  `1 <= start <= end <= line count`; for `range` scopes, at least one citation
  overlaps a changed hunk.
- **Option shuffling**: models tend to favour certain positions for the
  correct answer, so `render` shuffles options and relabels `a`–`d`.

## Output files (`generate`)

Written to `--out` (default: `./readback-quiz-<timestamp>/`):

| File          | Contents                                                 |
|---------------|----------------------------------------------------------|
| `quiz.json`   | The full quiz (source of truth)                          |
| `quiz.md`     | Questions only; options inside a collapsed `<details>`   |
| `answers.md`  | Correct option, every option's `why`, citations          |

## Taking a quiz (`take`)

Readback is for **learning, not examining**, so each question is answered in
two steps that separate *recall* from *recognition*:

1. **Question only.** The user types an answer in their own words.
2. **Options revealed.** The user picks a–d. Auto-graded.
3. **Answer revealed.** Correct option, every option's `why`, and the
   `file:line` citations to go and read.
4. **Self-grade.** The user compares their own answer with the explanation:
   *got it / partly / missed*.

At the end, a summary shows both signals per question. A correct pick with a
"missed" self-grade marks understanding that is shallow — exactly what the
tool exists to surface.

The attempt is saved next to the quiz as `quiz.attempt-<timestamp>.json`:

```ts
type Attempt = {
  quiz: string;                 // path to quiz.json
  startedAt: string;
  finishedAt: string;
  answers: {
    questionId: number;
    ownAnswer: string;
    picked: "a" | "b" | "c" | "d";
    correct: boolean;
    selfGrade: "got-it" | "partly" | "missed";
  }[];
};
```

No LLM calls are made while taking a quiz.

## Stack

| Need               | Choice                                   | Why                                     |
|--------------------|------------------------------------------|-----------------------------------------|
| Runtime            | Node.js 24 LTS (pinned)                  | Runs `.ts` directly via type stripping  |
| Type checking      | `typescript`, `tsc --noEmit`             | Checks types; no build step needed      |
| CLI arguments      | `node:util` `parseArgs`                  | Built in                                |
| Git                | `node:child_process` `execFile`          | No shell → no injection via filenames   |
| HTTP to the LLM    | built-in `fetch`                         | See the real request; no SDK            |
| Schema/validation  | `zod`                                    | Types + runtime validation + JSON Schema|
| Interactive prompt | `node:readline/promises`                 | Built in                                |
| Tests              | `node:test`                              | Built in                                |

Runtime dependencies: **zod** only. Dev: `typescript`, `@types/node`.
All versions pinned exactly.

## Layout

```
src/
  cli.ts            # the only file that reads env vars or prints
  core/
    scope.ts        # target → Scope; git queries
    context.ts      # Scope → numbered text, size budget
    llm.ts          # complete(messages, { apiKey, model, baseUrl }) → string
    quiz.ts         # zod schemas, prompt, parsing, citation validation
    render.ts       # shuffle; quiz → quiz.md / answers.md
    take.ts         # interactive flow; attempt file
test/
  *.test.ts
```

Rule: `src/core/` never touches `process.env` or `console` — it takes
arguments and returns values (or throws). This keeps it testable and reusable
by the future web server.

## Configuration

| Env var              | Required | Default                       | Purpose                       |
|----------------------|----------|-------------------------------|-------------------------------|
| `READBACK_API_KEY`   | yes      | —                             | LLM API key (never persisted) |
| `READBACK_BASE_URL`  | no       | `https://api.openai.com/v1`   | Any OpenAI-compatible endpoint|
| `READBACK_MODEL`     | yes      | —                             | Model ID                      |

## Out of scope for the MVP

- `mr:` / `pr:` targets, remote URLs, cloning, forge APIs
- Non-OpenAI-compatible providers
- Caching, history across quizzes, spaced repetition
- Web UI, server, container (see `AGENTS.md` delivery plan)

## Open questions (decide during implementation)

- Default `--questions` count and size budget (start: 8 questions, ~60k chars).
- Exact prompt wording — iterate on real repos.
- Whether `take` should allow skipping the free-text step.
