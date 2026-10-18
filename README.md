# Readback

An AI-based quiz app that tests your knowledge of git diffs to punish lazy reviewers.

Generate comprehension quizzes about code in your Git repositories — scoped
to a merge request, commit range, or path — using your own LLM API key.
Every question cites `file:line` so answers can be checked against the source.

> Status: planning. The CLI MVP is designed in [`docs/design.md`](docs/design.md);
> see `AGENTS.md` for agreed decisions. To build it yourself while learning
> TypeScript, follow the tutorial in [`docs/tutorial/`](docs/tutorial/README.md).

## Usage (planned CLI)

```sh
export READBACK_API_KEY=...        # your own LLM key; never stored by Readback
export READBACK_MODEL=...          # see .env.example for all settings

readback generate <repo-path> main...feature   # quiz a range of commits
readback generate <repo-path> path:src/auth/   # quiz files under a path
readback take ./readback-quiz-<timestamp>/quiz.json
```

Readback reads a local clone you already have access to. It never fetches
from remotes; to quiz a pull/merge request, fetch it first:

```sh
git fetch origin pull/45/head:pr-45                # GitHub
git fetch origin merge-requests/123/head:mr-123    # GitLab
readback generate . main...pr-45
```

`take` asks for your own answer first, then shows the multiple-choice
options, then the explanation with `file:line` citations, and lets you
self-grade. No LLM calls are made while taking a quiz.

## Deploy contract

Consumed by the `home-server` homelab repo. Fill in as the app takes shape.

| Item          | Value                                              |
|---------------|----------------------------------------------------|
| Image         | `ghcr.io/nikolakatardjiev/readback:X.Y.Z`          |
| Tag format    | semver, from git tag `vX.Y.Z`                      |
| Port          | TBD                                                |
| Healthcheck   | TBD                                                |
| Env vars      | TBD (see `.env.example`)                           |

Updates: a release triggers a GitLab pipeline that opens an MR in
`home-server` bumping the pinned tag.
