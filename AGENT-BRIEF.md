# Brief for a coding agent

Fill in the `<…>` fields and hand this to the agent working in your repo.
Resolve every "Decisions" line before sending. An agent that has to guess
them will guess.

---

Make `<repo>` deployable as a Zylos component, following
https://github.com/thaddeus-growth-hacking/zylos-component-playbook
(README → PLAYBOOK → ZYLOS-CORE-FACTS → PATTERNS → CHECKLIST). Use the
adapter in `<reference repo>` (branch/tag `<ref>`) as the working example:
its root `SKILL.md`, `zylos/README.md`, `zylos/hooks/*.js`, `zylos/bin/*.js`,
`zylos/ci/smoke-core.mjs`, `.github/workflows/zylos-smoke.yml` and
`tests/test_zylos_*.py`.

Target zylos-core `<oldest tested tag>` and `<latest tag>`. Read
zylos-ai/zylos-core at those tags for the rules, and don't trust the
playbook over the source. If they differ, say so. Read
zylos-ai/zylos-openmax `SKILL.md` for how results are delivered
(ArtifactStore + comment + owner acceptance).

Decisions already made:

- Component name: `<GitHub repo name>`. Linked command: `<specific-name>`.
- Work starts: `<on request | every N minutes via a tick | as a pm2 service>`.
- Throughput grows through: `<knobs and defaults>`, not the schedule.
- Required config (sensitive): `<KEYS>`. Optional: `<KEYS>`.
- Results go to `<channel>`, the owner's private channel; the owner accepts. Open: `<anything undecided>`.
- Out of scope here (a separate check): `<e.g. browser login on the host>`.
  Keep that code path working.

Core changes (in `<src dir>`, each with tests, and nothing in the core may
mention Zylos):

1. One `<APP>_DATA_DIR` (default = repo root). Today state is tied to the
   repo root at `<file:line, …>`.
2. A run lock in the data dir: a second run prints "skipped: …" and exits 0.
3. `<a daily cap / budget>` as a setting, enforced in code.
4. The command that `status` prints after NEXT can be overridden.
5. `status --json`.

Adapter (`zylos/`):

- `SKILL.md` header with `version` `<X.Y.Z>`.
- Hooks: configure (merge into `<data dir>/.env`, no network, well under
  30 s); post-install and post-upgrade (runtime, data dir at 0700,
  preflight, one scheduler task); pre-uninstall (never deletes data).
- `bin/<command>.js` and `detach.js`.
- `<The tick prompt: what one tick does, and that it returns in seconds.>`
- Move the operating loop from `<agent file>` into the `SKILL.md` body. Keep
  it and the task prompt runtime-neutral (Claude Code or Codex).
- `zylos/README.md` starts with "What this touches" (endpoints, files,
  secrets, processes, tasks) for the host agent's security review.
- Add the smoke CI and the boundary test.

Run `<check command>` after every change. Open a PR when done. Don't tag
the release: that happens on the merge commit, after merge.
