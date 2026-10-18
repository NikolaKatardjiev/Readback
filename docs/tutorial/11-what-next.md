# 11 — What next

[← 10 The `take` command](10-take.md) · [README](README.md)

## Goal

You've built the whole CLI from [`../design.md`](../design.md): `generate`
turns a commit range or path into a cited multiple-choice quiz, `take` runs
it interactively and grades recall separately from recognition. This chapter
is about running it for real, then where to take it next. No new exercises,
tests, or solutions — just pointers.

## Running it against a real repo with a real key

Everything you tested in chapters 09–10 used `READBACK_API_KEY=dummy` and a
fake server. Switching to a real model means three environment variables and
one rule:

**Never pass the key as a `--flag`.** Flags leak into shell history
(`~/.bash_history`, `~/.zsh_history`) and are visible to anyone on the
machine via `ps aux` while the process runs. The design document is explicit
about this — `READBACK_API_KEY` is read from the environment and nowhere
else. You have two reasonable ways to set it:

- **Per shell session**, if you're just trying things out:
  ```
  $ export READBACK_API_KEY=sk-...
  $ export READBACK_MODEL=gpt-4o-mini
  $ npm start -- generate ~/code/some-project main..HEAD
  ```
- **In `.env`**, if you'll run this repeatedly (it's already in
  `.gitignore` from chapter 00 — double check before committing anything if
  you ever change that file):
  ```
  $ cat > .env <<'EOF'
  READBACK_API_KEY=sk-...
  READBACK_MODEL=gpt-4o-mini
  EOF
  $ node --env-file=.env src/cli.ts generate ~/code/some-project main..HEAD
  ```

`READBACK_BASE_URL` is optional and defaults to `https://api.openai.com/v1`.
Point it at any OpenAI-compatible chat-completions endpoint — a local model
server, Azure OpenAI, OpenRouter, Groq, whatever you have access to —
by setting `READBACK_BASE_URL` and `READBACK_MODEL` to match. That's the
entire point of `complete`'s `baseUrl` option in `llm.ts`: one request shape,
any compatible provider.

Other things worth trying on a real repo:

- **Tune `--questions`.** The default is 8. A one-file bugfix probably only
  supports 3–4 good questions before the model starts reaching; a big
  feature range can easily support more. If `generate` keeps warning about
  dropped questions and retrying, that's a signal the diff is too large or
  too thin for the question count you asked for — not a bug to fix in the
  code.
- **Read `answers.md` critically, not trustingly.** The whole reason every
  question carries a `file:line` citation (and why `validateCitations`
  exists at all) is that the model can still be *wrong* even when its JSON
  is well-formed and every citation points at a real line. A citation
  proves the model looked at the right place, not that its reasoning about
  what's there is correct. Before you trust an answer key, open the cited
  lines and check the explanation actually holds up. This is also just good
  practice for actually learning the code, which is the point of the whole
  tool.

## Ideas for extending it

These are intentionally left as exercises with **no solutions provided** —
by this point you have every piece of the codebase needed to build them
yourself, and sketching the design before touching code is good practice
on its own.

### `mr:`/`pr:` targets

Out of scope for the MVP, but `design.md` already says exactly how it would
fit:

> An `mr:123` / `pr:45` target would run `git fetch` (with the user's own
> git credentials) and resolve to a `range` scope. Only the parser changes.

Concretely: `parseTarget` in `scope.ts` would gain a branch recognizing
`mr:`/`pr:` prefixes, run `git fetch origin merge-requests/123/head:mr-123`
(or the GitHub equivalent) via your existing `execFile`-based `git` helper,
then build the same `{ kind: "range", ... }` shape every other target
produces. Nothing downstream of `parseTarget` — `context.ts`, `quiz.ts`,
`llm.ts`, `render.ts` — would need to change at all. That's the payoff of
having funneled everything through one `Scope` type from chapter 02 onward.

### `--no-free-text` for `take`

Some people will want to skip step 1 (typing a free-text answer) and go
straight to picking an option — faster, but loses the recall-vs-recognition
signal the design is built around. Try adding a `--no-free-text` flag to
`runTake`'s `parseArgs` call, threading a boolean through to `takeQuiz` (or
restructuring `takeQuiz` itself) to skip the `ownAnswer` prompt and self-grade
step when set. Think about what `Answer.ownAnswer` should contain in that
mode before you write any code.

### Nicer zod error output

`parseLlmQuiz` currently quotes zod's whole `result.error.message` — a big
formatted string, fine for a quick stderr line but not great for a 5-page
diff where you'd like to know exactly which field was wrong. Zod 4 also
exposes `result.error.issues`, a structured array of `{ path, message, ...
}`. Try formatting that into something like
`questions[2].citations[0].start: expected number, got string` instead.

### A summary across attempts

`saveAttempt` writes one `quiz.attempt-<timestamp>.json` per `take` run, but
nothing reads more than one back. If you retake the same quiz a few times
(or take several quizzes from the same project), a small script that globs
`quiz.attempt-*.json` files and reports something like "self-graded 'missed'
3 times on `why` questions about `src/auth/`" would turn individual attempts
into a trend — closer to actually tracking understanding over time.

### CI

`package.json` already has `typecheck` and `test` scripts; a GitHub Actions
workflow that runs both (`npm run typecheck && npm test`) on every push and
PR would catch the same class of mistakes chapters 00–10 asked you to catch
by hand, automatically, including for anyone who later contributes to this
repo.

## Where this project goes from here

This tutorial stops at the CLI, which is the whole of the current MVP scope.
Per [`../../AGENTS.md`](../../AGENTS.md)'s delivery plan, what comes after
(not covered here, and not yet built even in the reference this tutorial was
written from):

1. Use the CLI for a while, as-is.
2. Wrap it in a small web UI.
3. Containerize it: one image, configured via env vars, with a healthcheck.

If you've made it through all eleven chapters, you have a working, tested
command-line tool and (hopefully) a real handle on `async`/`await`, the
module system, `zod`, dependency injection for testability, and enough
`node:*` built-ins to be dangerous. That's the actual goal of this tutorial —
the CLI is just the vehicle.

[← 10 The `take` command](10-take.md) · [README](README.md)
