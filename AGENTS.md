# Readback — agent guide

Readback is a small self-hosted web app that generates **comprehension quizzes about code
in targeted Git repositories**, using an LLM API key supplied by the user.
Purpose: help the owner actually understand code that was "vibe coded".

This is a **standalone project** (public GitHub repo: github.com/NikolaKatardjiev/Readback). It is deployed on the
owner's homelab, but the homelab repo (`home-server`, GitLab) only references
the published image — no app code lives there.

## Product decisions (agreed)
- **Scope quizzes narrowly**: a commit range or a path (file/folder) — never
  "the whole repo". Cheaper and sharper questions. MR/PR targets are deferred
  (user fetches the ref and passes a range).
- **Multiple choice, learning not exam**: one LLM call generates questions,
  options and answer key; taking a quiz needs no LLM. User writes their own
  answer before seeing options, then self-grades.
- Full CLI MVP design: `docs/design.md`.
- **Grounded answers**: every question cites `file:line-range` so the answer
  can be verified against source, not trusted blindly.
- Question mix: "what happens if X fails", "why is this needed",
  "predict the output", "spot the bug" — understanding over recall.
- Output is reviewable artifacts (e.g. Markdown quiz + answer key).

## Security decisions (agreed)
- **BYO AI key stays client-side**: held in the browser, sent per request,
  **never persisted** server-side or logged. If persistence is ever added,
  it must be encrypted at rest — treat as a design change.
- **Readback is not responsible for repo access**: it reads local clones the
  user can already read, using read-only local git commands only. The only
  secret it handles is the user's LLM key. (How the web phase gets repo
  content is undecided.)
- The app is **not public**: LAN/VPN only or behind auth at the reverse proxy.
- Never commit secrets; provide `.env.example` only.

## Delivery plan
1. **MVP first**: a TypeScript CLI — `readback generate <repo> <range|path:…>`
   → `quiz.json` (+ Markdown views) with `file:line` citations, and
   `readback take quiz.json` for interactive self-grading. Use it for a while.
2. Then wrap it in a small web UI (stack TBD — keep it simple).
3. Containerize; one image, configured via env vars, with a healthcheck.

## Release & deploy contract (with `home-server`)
- GitHub Actions builds on semver tag `vX.Y.Z` and pushes
  `ghcr.io/nikolakatardjiev/readback:X.Y.Z` (no reliance on `latest`).
- After publishing, the workflow calls a **GitLab pipeline trigger**
  (scoped trigger token stored as a GitHub secret) which opens an MR in
  `home-server` bumping the pinned tag. Git stays the source of truth —
  **no Watchtower-style auto-pulling**.
- Document the contract in `README.md` and keep it current:
  image name, tag format, listening port, env vars, healthcheck path.
- `home-server` will contain only `compose/readback/compose.yaml` (Traefik labels,
  external `proxy` network) and `docs/readback.md`.

## When editing
- Keep `README.md` (incl. the deploy contract) current with behavior.
- Pin dependency and base-image versions.
- Prefer small, readable code: the owner wants to learn from it, not just run it.
