# quiz-app

Generate comprehension quizzes about code in your Git repositories — scoped
to a merge request, commit range, or path — using your own LLM API key.
Every question cites `file:line` so answers can be checked against the source.

> Status: planning. See `AGENTS.md` for agreed decisions.

## Deploy contract

Consumed by the `home-server` homelab repo. Fill in as the app takes shape.

| Item          | Value                              |
|---------------|------------------------------------|
| Image         | `ghcr.io/<owner>/quiz:X.Y.Z`       |
| Tag format    | semver, from git tag `vX.Y.Z`      |
| Port          | TBD                                |
| Healthcheck   | TBD                                |
| Env vars      | TBD (see `.env.example`)           |

Updates: a release triggers a GitLab pipeline that opens an MR in
`home-server` bumping the pinned tag.
