# Zylos component playbook

How to take a harness (a CLI, a pipeline, an agent's toolset) that already
works on a laptop and make it a **Zylos component**: installed with
`zylos add`, configured through Zylos, run by the Zylos agent on a schedule
or on request, upgraded with `zylos upgrade`, and removed without losing
data.

This playbook comes from two components that went through it:

| Reference | Shape | What it shows |
|---|---|---|
| `thaddeus-growth-hacking/amazon-ads-harness`, v0.3.2 | read-only reporting CLI + web console | a pm2 **service** behind the host's Caddy login, an **hourly** scheduler task, relaying confirmations from chat |
| `thaddeus-growth-hacking/amazon-seller-outreach`, v0.3.0 | long-running batch pipeline | a **10-minute tick** that starts detached batches under a daily cap, reports finished ones, and delivers results through openmax |

Both repos are private; ask the owner for access. Their `zylos/` folder,
root `SKILL.md`, `tests/test_zylos_*.py` and `.github/workflows/zylos-smoke.yml`
are the working versions of everything described here.

## Read in this order

1. [PLAYBOOK.md](PLAYBOOK.md): the steps, from "is this a fit?" to the
   first install on a real host.
2. [ZYLOS-CORE-FACTS.md](ZYLOS-CORE-FACTS.md): what zylos-core actually does
   at install, configure, upgrade and uninstall, checked in its source.
   Every rule in the playbook traces back to one of these.
3. [PATTERNS.md](PATTERNS.md): the reusable designs (data dir, run lock,
   daily cap, `status --json`, the tick, detached jobs, configure's `.env`
   encoding, the uv bootstrap, the boundary test, the CI smoke, delivery).
4. [CHECKLIST.md](CHECKLIST.md): what to verify before you open the PR and
   before you tag a release.
5. [AGENT-BRIEF.md](AGENT-BRIEF.md): a prompt template for handing the job
   to a coding agent.
6. [OPERATING.md](OPERATING.md): after the first install: how a coding
   session instructs the host agent over openmax (the group, one identity
   per session, mentions, headless sends, runbook-style instructions) and
   ships a release when the host can't fetch a private repo.

## The shape in one picture

```
your repo
├── src/…                 the harness ("core"). Never mentions Zylos.
├── SKILL.md              frontmatter = the Zylos manifest; body = how an agent operates the harness
├── zylos/                the adapter. Delete it and the core still runs anywhere.
│   ├── README.md         ops guide for Zylos only
│   ├── hooks/            lib.js, configure.js, post-install.js, post-upgrade.js, pre-uninstall.js
│   ├── bin/              the linked command, detach.js, tick.js (if scheduled)
│   └── ci/smoke-core.mjs smoke test against a real zylos-core checkout
├── tests/test_zylos_boundary.py   the core never mentions zylos; the manifest matches the code
├── tests/test_zylos_hooks.py      the adapter's behaviour, offline
└── .github/workflows/zylos-smoke.yml   the smoke, one job per tested zylos-core tag
```

## Tested against

zylos-core **v0.7.1** (`ec0b851c`) and **v0.8.1** (`a5ab5d12`). The files a
component depends on are identical between those two tags (see
ZYLOS-CORE-FACTS.md). When a new core ships, re-check those files and add
the tag to your smoke matrix.
