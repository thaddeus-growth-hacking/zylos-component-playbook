# Patterns

Reusable designs, each with the problem it solves and the minimal shape.
The full working versions are in the reference repos (README.md).

## Core side (no Zylos in it)

### One data dir

**Problem:** a harness writes `.env`, its DB, backups and exports next to its
code. On Zylos the code dir is backed up and 3-way merged on every upgrade,
and state must live in `~/zylos/components/<name>`.

**Shape:** one setting, read from the shell only, defaulting to the repo
root:

```python
def resolve_data_dir() -> Path:
    return Path(os.environ.get("APP_DATA_DIR") or ROOT).expanduser().resolve()

@cache
def get_settings() -> Settings:
    return Settings(_env_file=resolve_data_dir() / ".env")   # pydantic-settings
```

Every other path (`db`, `backups/`, `exports/`, `imports/`) is derived from
it. Tests point it at `tmp_path`, which also guarantees they never read the
real `.env`.

### Run lock

**Problem:** a scheduler, a human and an agent can all start the same run.

**Shape:** `fcntl.flock(LOCK_EX | LOCK_NB)` on `<data dir>/<app>-run.lock`.
The OS releases it however the process ends, so it's never stale. Write
`{pid, started_at, run_id}` into the file for the curious; readers probe
with `LOCK_SH | LOCK_NB`. A second run prints `skipped: another run is
running (pid …)` and **exits 0**.

### Daily cap, enforced in code

**Problem:** a prompt that says "at most 4 a day" is advice; the proxy bill
isn't.

**Shape:** inside the lock, count today's started runs in a `runs` table
(`id, date, started_at, finished_at, status, error, args, pid,
exported_at, …`). At the cap, `skipped: daily cap reached: 4 of 4 …`, exit 0.
Count every started run, failed ones included: a broken dependency then
costs at most one day's cap, and each failure is reported. Record the
outcome in `finally`. Turn SIGTERM into `SystemExit` so a stopped run
records itself. A row still `running` with no lock holder is reported as
`killed`.

### `status --json` and an overridable NEXT

**Problem:** a tick (or a cheap model) needs to know exactly what to do next
without parsing prose.

**Shape:**

```json
{
  "preflight_ok": true,
  "preflight": [{"ok": true, "check": "PROXY is URL form", "fix": ""}],
  "progress": {…},
  "batches": {"cap": 4, "started_today": 1, "left_today": 3,
              "running": null, "last": {…}, "to_report": {"id": 7, "status": "ok", …}},
  "next": {"action": "batch", "args": ["run", "--limit", "300"],
           "command": "'/home/u/zylos/bin/my-cmd' run --limit 300", "text": "…"}
}
```

`action` is one of `fix | wait | batch | capped | caught_up`. `command` is
spelled with a setting (`APP_CMD`, default `uv run app`), which the host's
wrapper sets to its own absolute path. Batch size comes from settings too,
so the owner ramps up with configure, not code.

### Export one job, and record it

`export --batch <id>` writes that job's results and sets `exported_at` on
the row. `to_report` is the oldest finished row without it. That's how a
tick reports each job exactly once.

## Adapter side (`zylos/`)

### Paths, the way core derives them

```js
const SKILL_DIR = process.env.ZYLOS_SKILL_DIR || path.resolve(__dirname, '..', '..');
const ZYLOS_DIR = process.env.ZYLOS_DIR || path.join(os.homedir(), 'zylos');
const DATA_DIR  = process.env.ZYLOS_DATA_DIR || path.join(ZYLOS_DIR, 'components', NAME);
```

Node stdlib only: core runs hooks as plain `node <hook>`, with no
`npm install` when `lifecycle.npm: false`.

### configure's `.env`

- Merge only: update in place, append new keys, keep keys not given, and
  treat `""`/`null` as "leave as is". Write to a temp file, chmod 0600,
  rename, and clean up `.env.tmp-*` left by an interrupted run.
- Encode for the harness's reader. For python-dotenv (pydantic-settings):
  always single-quote, escaping `\` and `'`. That round-trips spaces, `#`,
  quotes and backslashes. **Refuse `${`**, which python-dotenv expands even
  in single quotes. Also refuse multi-line values and objects, and refuse
  the **whole** input so nothing half-writes.
- Test the round trip through the harness's real loader, not a JS parser.

### Runtime bootstrap (Python example)

- `uv` if missing: Astral's installer at a **pinned version, checked
  against its sha256 before it runs**, with `UV_NO_MODIFY_PATH=1`.
- The interpreter: `uv python find X.Y || uv python install X.Y`.
- Dependencies: `uv sync --frozen --no-dev --project <skill dir>` with
  `UV_PROJECT_ENVIRONMENT=<data dir>/.venv`, so the venv never lands in the
  skill dir. Run with `uv run --quiet --frozen --no-dev --project <skill
  dir> <app>`, plus `PYTHONPYCACHEPREFIX=<data dir>/.pycache`.
- Verify it: copy the skill dir, run once, and diff the file list. It must
  be unchanged.

### The linked command

`bin/<specific-name>.js` resolves its real path (it's invoked through a
symlink), builds the env (the data dir, the venv, `APP_CMD` = the link's
absolute quoted path, with a caller's own values winning), runs the
harness with `stdio: 'inherit'`, and exits with its status. Without the
link, post-install prints the long form (`APP_DATA_DIR=… uv run …`).

### Scheduler task reconcile

In post-install and post-upgrade:

1. If there's no scheduler skill, print a note and return.
2. `cli.js list --json`, filter by your task name.
3. Keep the task whose prompt already matches this release, else the
   oldest. `update` it when the prompt, cron, miss threshold or configured
   reply differs. Leave a reply that was set by hand alone when none is
   configured. `remove` the rest.
4. Judge success by the output lines, not exit codes.
5. With no reply channel, print the exact `cli.js update <id>
   --reply-channel … --reply-endpoint …` command on every run until one is
   set. The hint should name the **owner's private channel** (their DM or
   1:1 conversation id), never a group: one session serves every channel,
   and a group would see owner-only reports.

pre-uninstall removes every task with the name.

### Detached jobs

`detach.js <name> -- <cmd…>` does this:

- spawns `sh -c '"$@"; echo "APP-EXIT $?"'` with `detached: true`, cwd =
  the data dir, and output to `<data dir>/logs/<name>.log`;
- writes `<name>.pid` as `{pid, start, cmd}`, where `start` is the OS's
  process start time (`/proc/<pid>/stat` field 22 on Linux,
  `ps -o lstart=` on macOS), so a reused pid isn't mistaken for the job;
- serializes starts with an `O_EXCL` lock file (stale after 60 s);
- refuses while the job is alive.

pre-uninstall SIGTERMs the process group of live jobs.

### The tick

The prompt stays short and fixed. A script does the mechanics:

```
Run node '<skill dir>/zylos/bin/tick.js' (it returns in seconds). …
REPORT lines: deliver that job to the owner as the <name> skill's "Delivering" section says.
ALERT lines: send them to the owner. Neither: report nothing.
Never run or wait for a job yourself. Then call done.
```

`tick.js` works like this:

1. `status --json`.
2. If `to_report` is set: `export --batch <id>`, then print `REPORT` lines
   (status, times, error, file, counts, the log tail for a failed job).
3. If `next.action == "batch"`: `detach.js job -- <cmd> <next.args>`, then
   an `OK started …` line.
4. Otherwise an `OK` line saying why nothing started.
5. `ALERT` for `fix`, `caught_up` and its own failures, **once per
   condition**. Keep the sent alerts in `<data dir>/logs/tick-state.json`.
6. Mask credential values (and a URL password, raw and decoded) in every
   line.

Pick the miss threshold as about one period: a tick more than a period
late is skipped, since the next one is due.

Cron runs in the host's time zone (`TZ` in `~/zylos/.env`). For a
component many hosts install, avoid `:00` and `:30`, where everyone's
tasks pile up. Use an off-minute (`7,17,27,37,47,57 * * * *` rather than
`*/10`; `23 * * * *` rather than `0 * * * *`). Put anything daily at a
stated local time, and document which time zone it assumes.

### Delivery (openmax)

In `SKILL.md`, "Delivering a batch":

1. Upload the file to the ArtifactStore by sending `[MEDIA:file]<abs path>`
   into the owner's conversation through `c4-send`, with the message body
   **on stdin**:

   ```sh
   cat <<'EOF' | node ~/zylos/.claude/skills/comm-bridge/scripts/c4-send.js openmax '<conversation id>'
   [MEDIA:file]/home/u/zylos/components/<name>/exports/result.csv
   EOF
   ```

   Only the message body goes between the `<<'EOF'` line and the closing
   `EOF`, and the terminator must never end up in the message itself.
   Agents do sometimes paste it into the body. Never pass the body as an
   argument; see the host's `comm-bridge/SKILL.md`.
2. Comment naming the artifact, the job id and the summary, on the Task
   that tracks it. Whether one exists, and where, is the owner's intake
   decision: ask once, and never create an Issue or Project implicitly.
3. Ask the owner to accept. The work stays delivered, not accepted, until
   they do.

On other channels: the summary, plus the file if the channel takes files.
With 0 results, send the summary only. A rejection changes a setting or a
roadmap item, never the data.

### Boundary test

```python
MENTION = re.compile(r"zylos", re.I)
core = [p for d in ("src", "docs", "tests") for p in (ROOT / d).rglob("*")
        if p.is_file() and not p.name.startswith("test_zylos_")]
assert not [p for p in core if MENTION.search(p.read_text())]
```

Add frontmatter checks: `lifecycle` keys ⊆ `{npm, hooks, service}`; every
target exists under `zylos/`; `version` equals the package version; each
config key is a settings field or a hook-only key; declared `default` ==
the settings field's default; secrets are `sensitive`. Exclude files that
are history (a changelog), not dependencies, and say why in the test's
docstring.

### CI smoke against real core

The workflow checks out zylos-core **by commit** for each tag, runs
`npm ci --omit=dev --ignore-scripts`, asserts that
`package.json` version == the tag label, then runs
`node zylos/ci/smoke-core.mjs .zylos-core`. The smoke:

- copies the repo into a fake `$HOME/zylos/.claude/skills/<name>`;
- imports core's `parseSkillMd` and `linkBins`;
- installs core's real scheduler skill (`npm ci --omit=dev` in it);
- drives your hooks with test-only switches that skip the runtime download
  (enabled only when `<APP>_ZYLOS_HOOK_TEST=1`).
