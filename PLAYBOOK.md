# Playbook: from a laptop harness to a Zylos component

Each step says what to do, why (the core fact behind it, in
[ZYLOS-CORE-FACTS.md](ZYLOS-CORE-FACTS.md)), and how to know it's done.
Designs referred to by name are in [PATTERNS.md](PATTERNS.md).

## 0. Decide before you build

Answer these in writing (the PR description is a good place). Each answer
changes what you build.

1. **What is the component name?** It is the GitHub repo name, minus a
   leading `zylos-`, not your local folder's name. Check with
   `gh repo view --json name`. Everything else keys off it: the skill dir,
   the data dir, and `SKILL.md`'s `name`.
2. **How does work start?**
   - *On request only.* The agent runs a command when the owner asks. You
     need the linked command and `SKILL.md`, and no scheduler task.
   - *On a schedule.* A scheduler task sends a prompt every N minutes.
     Anything longer than a few minutes must run **detached** and be picked
     up by a later tick (the tick pattern), because a task stays "running"
     until the agent calls `done`.
   - *Always on.* A pm2 service (for example a web console). If it needs a
     login, declare no `http_routes` (they can't carry auth) and document a
     Caddy block instead.
3. **How does throughput grow?** Through knobs the harness enforces (a daily
   cap, a batch size), not by editing the schedule. The schedule stays
   fixed; the knobs are config keys.
4. **Which credentials are required, which are optional?** Required ones are
   what a terminal install prompts for. Mark every secret `sensitive: true`.
5. **How are results delivered, and who accepts them?** Name the channel
   (openmax, lark, …) and the acceptance rule. Reports go to the **owner's
   private channel** (their DM or 1:1 conversation), not a group. One agent
   session serves every channel, and a group would see owner-only
   data. On openmax: the file goes to
   the ArtifactStore, a comment names it, and the owner accepts. Which Issue
   or Task holds a result is the owner's decision. If it isn't made yet,
   write it down as an open item and make the agent ask. Don't guess it
   into the code.
6. **What needs something the component can't install?** For example a
   browser session logged in to a site. Keep that code path working, and
   make the host setup a separate, documented check.
7. **Which zylos-core versions?** Pin the oldest you support (`MIN_ZYLOS`)
   and the tags you test.

## 1. Make the core host-neutral

The harness must run the same on a laptop and on the host, and must never
mention Zylos. Each of these gets tests in the core's own suite.

1. **One data dir.** One setting (for example `<APP>_DATA_DIR`, default =
   the repo root so laptop use is unchanged) that holds every piece of
   state: `.env`, the database, backups, exports, imports, logs. Hunt down
   every path tied to the repo root or the cwd. It is read from the shell
   only, since it says where `.env` is.
2. **Defaults in the harness.** Every optional key's default is applied by
   the harness itself (core never applies `default`).
3. **A run lock**, if work can overlap. A second run prints
   `skipped: <why>` and exits 0 (normal for a scheduled caller). Use an OS
   lock (`flock`) that is released however the process dies.
4. **Hard limits in code.** A daily cap, or a budget, enforced by the
   command that spends money, not by the prompt. Past the limit: `skipped:`
   and exit 0.
5. **A machine-readable status.** `status --json` with at least: preflight
   (credentials present, never their values), progress, the running job,
   the last finished job not yet reported, and `next`
   (`action` + `args` + the shell `command`).
6. **An overridable command spelling.** Whatever the harness prints as "run
   this next" must use a setting (for example `<APP>_CMD`), so the host can
   print its command's absolute path.
7. **An export for one job**, recorded on the job (`exported_at`), so a
   tick knows what it has already reported.

Done when: the laptop workflow is unchanged, tests cover each item, and
`grep -ri zylos src/` finds nothing.

## 2. Write the adapter (`zylos/` + `SKILL.md`)

Copy the closest reference implementation's `zylos/` and adapt it. Don't
start from nothing: its edge cases took real hosts to find.

1. **`SKILL.md` frontmatter**: `name` (= the component name), `description`
   (when an agent should use it), `version` (semver, equal to your package
   version), `type: capability`, `next-steps` (only the extras a chat
   install needs), `bin`, `lifecycle` (`npm`, `hooks`, optional `service`),
   and `config` (`required` / `optional`, with `description`, `sensitive`
   and `default`). No `upgrade:` block, since core doesn't read it. Add
   `http_routes` only for routes that are safe without a login.
2. **`SKILL.md` body**: the operating loop for an agent (status → NEXT →
   repeat → report), the hard rules, and how results are delivered. If the
   repo had a Claude Code agent file with the loop, move the loop here and
   make that file a pointer. Keep it **runtime-neutral**: the host may run
   Claude Code or Codex. Name shell commands and files, not one runtime's
   tools (no "use the Task tool", no MCP tool names). The same goes for
   scheduler prompts.
3. **`hooks/lib.js`**: paths the way core derives them
   (`ZYLOS_DATA_DIR` → `$ZYLOS_DIR/components/<name>` →
   `$HOME/zylos/components/<name>`), the runtime bootstrap, the scheduler
   wrapper, the job-file helpers, and the version check.
4. **`hooks/configure.js`**: merge-only into `<data dir>/.env` (0600, data
   dir 0700). No network. It prints key names, never values. It refuses the
   whole input on a bad value. It encodes values so the harness's `.env`
   parser reads them back exactly (see PATTERNS.md, "configure's .env").
5. **`hooks/post-install.js`**, idempotent and capped at about 9 minutes:
   - hard steps: `zylos --version` ≥ `MIN_ZYLOS`, the data dir at 0700, the
     runtime (pinned and checksummed), dependencies, and the harness's own
     preflight;
   - soft checks (reachability, a status summary);
   - print the absolute command;
   - reconcile exactly one scheduler task, if you have one.
6. **`hooks/post-upgrade.js`**: post-install in light mode (runtime,
   dependencies from the new lockfile, preflight, task reconcile).
7. **`hooks/pre-uninstall.js`**: stop detached jobs (verified by pid and
   start time), remove your scheduler tasks, and never touch the data dir.
8. **`bin/<command>.js`**: runs the harness with the data dir, a cache and
   venv under it, and `<APP>_CMD` = its own absolute path. It writes
   nothing into the skill dir. Give it a specific name (`amazon-cn-sweep`,
   not `sweep`), since core overwrites same-named links.
9. **`bin/detach.js`** for anything longer than a few minutes, and
   **`bin/tick.js`** if scheduled. The tick does the mechanics
   deterministically and prints `REPORT` / `ALERT` / `OK` lines; the prompt
   only tells the agent what to do with them.
10. **`zylos/README.md`**: the ops guide. Cover where things live, who runs
    which hook, config, the command, the task, detached jobs, and what's
    verified vs not (including **which runtime**, Claude Code or Codex).
    Start it with a short **"What this touches"** section for the host
    agent's security review:
    - every network endpoint and when it's called;
    - the files read and written (data dir, skill dir, `~/.local/bin`,
      anything else);
    - the secrets used and where they're stored;
    - the processes started and stopped;
    - the scheduler tasks created.

    Keep it accurate. It's what makes the review quick, and a wrong entry
    fails it.

## 3. Tests and CI

1. **`tests/test_zylos_boundary.py`**:
   - nothing in the core mentions Zylos;
   - the frontmatter parses;
   - every hook and `bin` target exists under `zylos/`;
   - `version` matches the package version;
   - every config key is a harness setting or a key the hooks read;
   - each declared default equals the harness's own;
   - every secret is `sensitive`.
2. **`tests/test_zylos_hooks.py`**: run the real Node scripts against a
   temp `$HOME`/`$ZYLOS_DIR` with a fake runtime, a fake `zylos --version`
   and a fake scheduler CLI. Cover:
   - configure: merge, refusals, permissions, and a **round trip through
     the harness's real settings loader**;
   - task reconcile: add, update drift, remove duplicates, the reply note;
   - pre-uninstall stopping a detached job;
   - the command's env;
   - detach refusing a second start;
   - every tick branch;
   - secret masking.
3. **`zylos/ci/smoke-core.mjs`** + **`.github/workflows/zylos-smoke.yml`**,
   against a real zylos-core checkout, pinned by commit, one job per tested
   tag:
   - core's `parseSkillMd` reads your frontmatter;
   - core's `linkBins` links your command, and it runs by absolute path
     from a shell without `~/zylos/bin` on PATH;
   - core's real scheduler CLI, driven by your hooks: two installs plus an
     upgrade leave one task; a drifted task and a duplicate are healed;
     pre-uninstall removes it;
   - if scheduled, the tick the prompt names runs from that shell;
   - if you have a service, core's ecosystem loads it.
4. Run the whole thing on a laptop standing in for a host: a fake
   `ZYLOS_DIR`, the real runtime, placeholder credentials. Run configure,
   then post-install, then a few ticks. Check the skill dir is
   byte-identical afterwards.

## 4. Release

1. Merge the PR.
2. Tag **the merge commit** and push the tag:
   `git fetch origin && git tag vX.Y.Z origin/main && git push origin vX.Y.Z`.
   A tag made before the merge points at code without the adapter, and
   `zylos add` installs the latest tag.
3. Check the tag: `git ls-tree vX.Y.Z --name-only | grep SKILL.md`, and that
   `SKILL.md`'s `version` equals `X.Y.Z`.
4. **Registry (optional).** With a public repo, a PR to
   `zylos-ai/zylos-registry`'s `registry.json` lets owners run
   `zylos add <name>` and `zylos search`. A private repo shouldn't go in
   the public registry, since that would expose its name. Install it as
   `zylos add <org>/<repo>`, or map the name on the host in
   `~/zylos/.zylos/registry.json`.

## 5. First install on a real host

In chat, ask the host agent to install it, or run it at a terminal:

```
zylos add <org>/<repo>              # latest vX.Y.Z tag; private repos need GitHub access on the host
```

For a chat install, the agent:

1. **asks the owner to confirm** the install (every install, upgrade and
   uninstall needs an explicit yes, which is an async round trip in chat),
   and reviews the hooks' source first. Point it at `zylos/README.md`
   "What this touches";
2. collects the required keys, keeping secrets out of chat where possible
   (the owner can run configure at a terminal and paste JSON);
3. pipes them as one JSON object to `node zylos/hooks/configure.js`;
4. runs `node zylos/hooks/post-install.js` with a **10-minute** timeout;
5. relays both outputs to the owner.

For a **private repo**, the host's GitHub access must outlive the first
install: every `zylos upgrade` fetches from GitHub again. Use a token that
won't silently expire, or note when it must be renewed.

Then watch, and write down what you saw in `zylos/README.md` "Verified":

- post-install passes, and prints the absolute command;
- the task is registered **with a reply channel**, the owner's private one;
- the first tick does what it should and calls `done`;
- a report reaches the owner's channel;
- one `zylos upgrade` (the 3-way merge, then post-upgrade's output in the
  JSON result);
- which runtime the host ran (Claude Code or Codex).

## 6. Keep it working

- **New zylos-core release**: re-read the files listed in
  ZYLOS-CORE-FACTS.md, add the tag and commit to the smoke matrix, and
  handle a difference with a version branch inside the adapter
  (`checkZylosVersion`), not a second adapter.
- **New config key**: add it to `config`. The boundary test forces it to be
  real. After the upgrade, the agent pipes it to configure and reruns
  post-install.
- **Every release**: bump `version` in `SKILL.md` and the package together,
  merge, then tag.

## Mistakes we made, so you don't

- **Named the component after the local folder.** The repo was
  `amazon-seller-outreach`, the folder `amazon-cn-seller-outreach`. The
  hooks' data-dir fallback would have pointed at an empty directory. Check
  `gh repo view --json name` first.
- **Tagged before merging.** `v0.3.0` pointed at main without the adapter;
  it had to be deleted and re-tagged on the merge commit.
- **Assumed quoting protects `.env` values.** python-dotenv expands `${VAR}`
  even inside single quotes. Configure must refuse such values (or the
  harness's parser must not expand).
- **A tick that could block.** A scheduled task stays "running" until
  `done`. A tick that waits on a 60-minute job stops the schedule for an
  hour. Ticks start work detached and return.
- **Reporting every tick.** A tick that sees the same broken credential
  every 10 minutes must say so **once**. Remember sent alerts in the data
  dir.
- **Passing a message to `c4-send` as an argument.** Quotes, `$` and the
  `[MEDIA:file]` prefix get mangled. Pipe the body on stdin with a quoted
  heredoc, as the host's `comm-bridge/SKILL.md` shows.
- **Relaying logs into chat verbatim.** Error lines can carry a proxy URL
  with its password. Mask credential values in everything the tick prints.
