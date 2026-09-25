# Operating a component through the host agent

Once the component is installed, the code keeps changing on a laptop, but
the component runs on the host and the host agent is the only one who
touches it. This file covers that loop: a coding session on the laptop
(the **operator**) sends the host agent instructions over openmax and
reads its reports. It is what the amazon-ads-harness releases after v0.3
went through.

Scheduled reports still go to the owner's private channel (PLAYBOOK §5).
This is the other direction: instructions to the agent.

## The room

- **One group per component**: the owner, the host agent and the
  operator's agent identity. Instructions, replies and the owner's
  corrections all stay in one place, and the owner sees every instruction
  before or as it runs.
- **One agent identity per operator session.** Two coding sessions that
  share one openmax identity also share its login cache and its wake
  events, so one session can end up acting on the other's messages. Give
  each session its own identity: a shell function that starts `claude`
  with the openmax plugin's config and data-dir variables pointing at
  that identity's own files. Log it in once, interactively, then add it to
  the group.
- Keep that function and those files on the laptop. Never commit them
  or quote them in chat.

## Mention the host agent

The host agent acts on messages that **mention it**. Start every
instruction with `@<agent name>`. Sent through the plugin's `comm_send`,
the text mention becomes a real one. Check it in the message's `mentions`
field (`comm getMessages`): it must hold the agent's member id. The agent
reacts 👀 when it has picked the message up. A message without the mention
can sit unread.

## Sending from a session that isn't in the chat

The operator doesn't need an interactive chat session open. Use a headless
run of the identity's function:

```sh
<identity-fn> -p "Read /abs/path/msg.md and send its content verbatim as ONE message \
with comm_send to the group conversation <id>. Send nothing else. Print the message id." \
  --allowedTools "Read,mcp__plugin_openmax-channel_openmax__comm_send" \
  --output-format text
```

- **Allow only `Read` and `comm_send`**, so the run can't do anything else.
- Keep the prompt plain ASCII. Put the message itself, in any language,
  in a file.
- Find the conversation id the same way first: allow only the `comm` tool
  and ask it to `listConversations` and `conversationMembers` and print
  them. Two groups can share a name, so pick the one whose members include
  this identity.
- Read the replies the same way, with `getMessages` and a small `limit`.

## Writing an instruction the agent can follow

An instruction is a runbook the agent executes, reports on and stops at.
What worked:

1. **Numbered steps, each with its exact command** and what to report
   (exit code, a count, the last log line, a task id). The agent follows
   the text; don't let it guess commands.
2. **"Stop at the first failure and report."** Without it, the agent
   works around a failed step and the report hides it.
3. **A "never" list in the first lines**: e.g. never confirm on the
   owner's behalf, never pass the flag that writes to the outside system,
   never add the env key that enables writes. The agent prepares; the
   human confirms.
4. **Read-only checks before anything that changes state**, and the
   expected result spelled out ("the card shows 5, not 37") so a wrong
   number is caught in the reply.
5. **"Reply after each step."** Long runs (detached pulls) then show
   progress, and a stalled step is visible.
6. **Deferred steps get a time and a trigger**: "after tomorrow's 02:20
   UTC sync, …". Ask for the server's time zone once. Cron runs in it.

The agent may ask the owner before an install, upgrade or uninstall
(ZYLOS-CORE-FACTS.md) and reviews the hooks first. Say in the instruction
what changed since the last release: hooks, network calls, files read.
That review is then quick.

## Releasing when the host can't fetch the private repo

`zylos upgrade` fetches from GitHub. When the host has no lasting GitHub
access, reinstall from a local checkout of the tag instead:

1. Fetch the tag onto the host with a **read-only** token (not stored in
   the component).
2. `zylos uninstall <name>` **without** `--purge`, so the data dir stays.
3. `zylos add <local path of the tag>`, then post-install.
4. Put back anything the uninstall removed outside the data dir. In our
   case that was the console's Caddy block: keep it in `zylos/README.md`,
   make post-install **warn when it's missing**, and have the agent run
   `caddy validate` before reloading.
5. Run doctor, restart the pm2 service, and check that the scheduler tasks
   were re-registered. Their ids change, so ask for the new ones.
6. If the release changed the schema, the instruction names the one-off
   rebuild commands. Nothing else runs until the first scheduled sync.

## Checklist for one release

- [ ] Tag cut after the merge (PLAYBOOK §6), and mirrored if you mirror.
- [ ] Instruction file written: never-list, numbered steps with commands,
      what changed for the review, schema steps, read-only checks with
      expected values, deferred steps with their trigger time.
- [ ] Sent headless from the operator identity. `mentions` holds the
      agent's id, and the agent reacted.
- [ ] Each step's reply read. The first failure is handled before anything
      else is sent.
- [ ] Anything new you learned about the host goes into `zylos/README.md`
      "Verified" or this playbook.
