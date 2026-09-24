# What zylos-core does to a component

Checked in zylos-ai/zylos-core at **v0.7.1** (`ec0b851c22cbb2dd57e461c4cb7229908a12d887`)
and **v0.8.1** (`a5ab5d12e56a3a8feb426b7449ac4b5c7e4fdd28`). These files are
byte-identical between the two tags: `cli/commands/add.js`,
`cli/lib/upgrade.js`, `cli/lib/configure-hook.js`, `cli/lib/config.js`,
`cli/lib/bin.js`, `cli/lib/service.js`, `cli/lib/caddy.js`,
`cli/commands/component.js`, `templates/pm2/ecosystem.config.cjs` and
`skills/scheduler/`. Re-check them for every new core release; don't
assume.

Paths below use the defaults: `ZYLOS_DIR=~/zylos`.

## Names and places

| Fact | Where |
|---|---|
| The component name is the **GitHub repo name**, with a leading `zylos-` stripped. `org/zylos-foo` and `org/foo` both install as `foo`. A local clone's folder name doesn't matter. Make `SKILL.md`'s `name` equal it. | `cli/lib/components.js` (`repo.split('/')[1].replace(/^zylos-/, '')`) |
| Code goes to `~/zylos/.claude/skills/<name>` (the **skill dir**). | `cli/lib/config.js` `SKILLS_DIR` |
| State goes to `~/zylos/components/<name>` (the **data dir**). `add` creates it with the default mode (0755); tighten it yourself. | `cli/commands/add.js` |
| `bin` entries are symlinked into `~/zylos/bin/` and their targets chmod 755. | `cli/lib/bin.js` `linkBins` |
| `~/zylos/bin` is added to `~/.profile` / `~/.bashrc` only. **The agent's shell reads neither**, so always call a linked command by its absolute path. | `cli/commands/init.js` |

## Install (`zylos add`)

| Fact | Where |
|---|---|
| Installs from the latest **semver tag** `vX.Y.Z`; `--branch <b>` installs a branch. A private repo needs `GITHUB_TOKEN`, `GH_TOKEN` or `gh auth login` on the host. | `cli/commands/add.js`, `cli/lib/github.js` |
| **At a terminal**, core prompts only for `config.required` keys (`sensitive: true` ones hidden), pipes them as one JSON object to the `configure` hook, then runs `post-install`. | `cli/commands/add.js` |
| **In a chat install** (`zylos add … --json`), core runs **no hook**. It downloads, creates the data dir, links `bin`, and returns `config` and `next-steps` for the agent to act on. Your `next-steps` must say what the agent should do (typically: pipe keys to configure, run post-install with a 10-minute timeout, relay the output). | `cli/commands/add.js` |
| Core **never applies a declared `default`**. The harness must apply its own defaults, or configure must write them. | (no code reads `default`) |
| A failing post-install is reported as a warning, not an install failure. | `cli/commands/add.js` |

## The configure hook

| Fact | Where |
|---|---|
| Run as `node <hook>` with cwd = skill dir, the collected values as one JSON object on **stdin**, and env `ZYLOS_COMPONENT`, `ZYLOS_SKILL_DIR`, `ZYLOS_DATA_DIR`. | `cli/lib/configure-hook.js` |
| Killed after **30 s** (`CONFIGURE_HOOK_TIMEOUT_MS`). No network, no package installs in it. | same |
| No Zylos command re-runs configure later. To add or change keys, the agent pipes them to the hook again, so it must be merge-only. | (no reconfigure command) |
| Without a configure hook, core writes the values into the Zylos root `~/zylos/.env`, shared by everything. Declare the hook so secrets stay in your data dir. | `cli/commands/add.js` |

## post-install, post-upgrade, pre-uninstall

| Fact | Where |
|---|---|
| post-install and post-upgrade get **no `ZYLOS_*` env**; derive paths from `ZYLOS_DIR` / `HOME` the way core does. pre-uninstall and configure do get them. | `cli/commands/add.js`, `cli/lib/upgrade.js`, `cli/commands/component.js` |
| `zylos upgrade` backs the skill dir up, 3-way merges the new release in, then runs post-upgrade **synchronously while holding the component lock**. Its output lands in the upgrade's JSON result; a failure is non-fatal. Keep it light. | `cli/lib/upgrade.js` (step 7) |
| Upgrades compare semver tags with `SKILL.md`'s `version`. Every release bumps `version` and gets a matching tag **on the merged commit**. An `upgrade:` block in the frontmatter is not read. | `cli/lib/upgrade.js` |
| `zylos uninstall` runs pre-uninstall **before** stopping the service and removing files, and deletes the data dir **only** with `purge`. | `cli/commands/component.js` |
| The agent's foreground commands are capped at about 10 minutes, and a chat shell's default is shorter. A first install that downloads a runtime needs a 10-minute timeout; cap your own hooks below that. | host agent |

## Services and routes

| Fact | Where |
|---|---|
| `lifecycle.service` `{name, entry, type: pm2}`: core's own pm2 ecosystem loads `<skill dir>/ecosystem.config.cjs` for each installed component at boot. The file must sit at the skill root. A terminal `add` starts the service; a chat install doesn't (the agent runs `pm2 start <skill dir>/ecosystem.config.cjs && pm2 save`); `upgrade` restarts it; `uninstall` deletes it. Core refers to it as `zylos-<name>` unless `name` is set. | `templates/pm2/ecosystem.config.cjs`, `cli/lib/service.js` |
| `http_routes` supports only `{path, type: reverse_proxy, target, strip_prefix}`: **no auth, no header rules**. A declared route publishes the target with no login. For anything that needs a login, declare no route and document a Caddy block the host agent pastes (with `# BEGIN/END zylos-component:<name>` markers so uninstall removes it). | `cli/lib/caddy.js` |

## The scheduler (`~/zylos/.claude/skills/scheduler/scripts/cli.js`)

| Fact | Where |
|---|---|
| A task is a **prompt**, not a command. At each due time the daemon sends `[Scheduled Task: <id>] <prompt>` plus "run `cli.js done <id>`" to the agent. | `scripts/daemon.js` |
| A task stays `running` until the agent calls `done`, or until it is stale after **1 hour** (`TASK_TIMEOUT`). It isn't dispatched again meanwhile. So a tick must return quickly and always end with `done`. | `scripts/daemon.js`, `scripts/daemon-tasks.js` |
| `--miss-threshold <s>` (default 300): a task overdue by more than this while the agent runtime was offline is skipped to its next time. | `scripts/daemon.js` |
| `--reply-channel` / `--reply-endpoint` are the task's reply path. Without them, whatever the task "reports" reaches no one. | `scripts/cli.js` |
| Names are **not unique**. Reconcile by `list --json` and act by id. | `scripts/cli.js` |
| Exit codes don't say whether a change worked (a `remove` of an unknown id exits 0). Read the output: `Task created: <id>`, `Task updated: <id>`, `Removed task: <id>`. | `scripts/cli.js` |
| Other useful commands: `update <id> --prompt/--cron/--miss-threshold/--reply-*`, `pause`, `resume`, `list`. | `scripts/cli.js` |

## Delivering results (openmax)

From zylos-ai/zylos-openmax's `SKILL.md` (v2.20.0), when the owner works
through openmax:

- A result is a **file in the ArtifactStore** (`as.upload`, or sending
  `[MEDIA:file]/abs/path` through `c4-send` into the conversation, which
  uploads it).
- The executor writes a **comment naming the artifact** on its Task before
  moving the Task to done.
- Work counts as complete only when the **owner accepts** it. The agent asks
  and never accepts on the owner's behalf.
- Whether work is tracked in an Issue, and in which Project, is the
  **owner's decision** at intake. The agent never creates an Issue or
  Project without it.
