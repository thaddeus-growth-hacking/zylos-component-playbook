# Checklist

## Before opening the PR

**Decisions** (PLAYBOOK §0), written in the PR description:

- [ ] The component name equals `gh repo view --json name`, minus a leading
      `zylos-`, and `SKILL.md` `name` matches it.
- [ ] How work starts: on request, on a schedule (tick), or as a service.
- [ ] How throughput grows: which knobs, and their defaults.
- [ ] Required vs optional keys; every secret is `sensitive: true`.
- [ ] Delivery channel and acceptance rule. Anything undecided is listed as
      an open item, not coded.
- [ ] Setup the component can't do (browser logins, accounts), marked as
      separate checks.

**Core** (no Zylos in it):

- [ ] One data dir setting; laptop default unchanged; tests use a temp one.
- [ ] Every optional key's default is applied by the harness.
- [ ] Run lock and hard limits in code; `skipped: …` exits 0.
- [ ] `status --json` with `preflight`, `next.action/args/command`, the
      running job and `to_report`.
- [ ] The command spelling in NEXT comes from a setting.
- [ ] A per-job export that records itself on the job.
- [ ] The core suite passes and covers each of the above.

**Adapter:**

- [ ] Frontmatter: `name`, `description`, `version`, `type`, `next-steps`,
      `bin` (a specific name), `lifecycle` (`npm`, `hooks`, `service` only
      if needed), `config`. No `upgrade:` block, and no `http_routes` that
      would need a login.
- [ ] configure: merge-only, 0600 file in a 0700 dir, no network, prints key
      names only, refuses the whole input on a bad value, and round-trips
      through the harness's real `.env` loader.
- [ ] post-install:
  - [ ] version check, data dir, pinned and checksummed runtime,
        dependencies into the data dir, preflight;
  - [ ] capped under 10 minutes, idempotent;
  - [ ] prints the absolute command;
  - [ ] one scheduler task, reconciled by id, with a reply-channel note.
- [ ] post-upgrade: light mode, the same reconcile.
- [ ] pre-uninstall: stops live detached jobs, removes tasks, keeps data,
      exits 1 on a task it can't list or remove.
- [ ] Nothing is written into the skill dir (diff its file list after a
      run).
- [ ] The tick returns in seconds, reports each job once, alerts once per
      condition, and masks secrets.
- [ ] `SKILL.md` body: the loop, the hard rules, delivery. An old agent file
      now points to it.
- [ ] `zylos/README.md`: tested core versions, where things live, hooks,
      config, command, task, detached jobs, verified vs not.

**Tests and CI:**

- [ ] `tests/test_zylos_boundary.py` and `tests/test_zylos_hooks.py` pass.
- [ ] `node zylos/ci/smoke-core.mjs <core checkout>` passes for every tested
      tag locally.
- [ ] `.github/workflows/zylos-smoke.yml` pins each core tag by commit.
- [ ] A dry run on a laptop standing in for a host: configure, then
      post-install, then a few ticks.

## Before tagging

- [ ] The PR is **merged**; `git fetch origin`.
- [ ] `SKILL.md` `version` == the package version == the tag you're about to
      make.
- [ ] `git tag vX.Y.Z origin/main && git push origin vX.Y.Z`.
- [ ] `git ls-tree vX.Y.Z --name-only | grep SKILL.md` finds it.

## After the first real install

- [ ] post-install output relayed; it printed the absolute command.
- [ ] The task is registered with a reply channel.
- [ ] One tick ran, did the right thing, and called `done`.
- [ ] A report reached the owner.
- [ ] One `zylos upgrade` ran cleanly.
- [ ] `zylos/README.md` "Verified" is updated with what was seen and when.
