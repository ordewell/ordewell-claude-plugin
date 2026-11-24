---
description: Run the Ordewell plan you reviewed. Its tasks run side by side, each in its own agent session on its own git worktree, and nothing is merged into your branch.
argument-hint: [session-id]
disable-model-invocation: true
---

# Run an Ordewell plan

Session: $ARGUMENTS

An empty session means the plan that `ordewell plan` made last in this folder. You start the run, follow it, and report what happened. The tasks run in agent sessions that Ordewell starts, not in this conversation.

These hold for every step:

- Never pass `--yes` to an ordewell command. Never run `ordewell auto full`, `ordewell autorun on` or `ordewell automerge on`, and never set `ORDEWELL_AUTONOMOUS_MODE`, `ORDEWELL_APPROVAL_MODE` or `ORDEWELL_APPROVAL_ALLOW`.
- Never run `ordewell merge` or `ordewell merge discard`. Landing the work is the user's decision, made with their own command.
- Never answer for the user: run `ordewell approve`, `ordewell checkpoint`, `ordewell complete`, `ordewell skip`, `ordewell retry`, `ordewell force-start`, `ordewell cancel`, `ordewell task-runner` or `ordewell task-model` only when the user asks for that action, or agrees to it when you offer it.
- Don't edit files under `.ordewell/worktrees/`: those are the tasks' own checkouts.
- Never install or upgrade Ordewell, and never run an installer or a package manager for it.
- This skill pre-approves no command. Claude Code asks the user before each command you run, in this turn and in every later one, unless their own permission settings or permission mode allow it. Before your first command, tell the user in a few short lines what they will be asked to approve, and why each command is needed:
  - `command -v ordewell`, and if it finds nothing `test -x ~/.local/bin/ordewell && echo found`: to find Ordewell;
  - `ordewell --version`: to check that it is 0.14.9 or later;
  - `ordewell status`: to list the sessions in this folder. It starts Ordewell's local server if none is running. With `--session-id`, later, it shows each task and its number;
  - `ordewell status --session-id <id> --output .ordewell/plugin-plan-<id>.json`, and, if this session has no Grep tool, `grep` on that file, which only reads: to save the plan as JSON in Ordewell's own folder before the run starts, so that you can read each task's type and state, and whether a run is going. A manual task left open needs the user before the run can end;
  - `ordewell run --unattended`: to start the run. Its tasks are sessions of Claude Code, Codex or the other agents in the plan, on the user's own logins, each in its own git worktree under `.ordewell/worktrees/`, and the command follows them until the run ends or a task waits on the user's answer. While a manual task is open it doesn't return, so you then start it with a one-minute timeout, as step 2 says;
  - `ordewell merge review --session-id <id>`: to show the diff of the run's branch against the user's branch, once the run has landed work. It only reads.

  Say also that the user's answers to the run, such as a retry or a checkpoint answer, are further commands that ask too. Then run them one at a time, as the steps say.
- If a command is refused, by the user or by their settings, don't run it again in this turn, and don't get its result another way: not with the other form of `ordewell` that step 1 names, not chained into another command, not through a script and not through another tool. Say which command was refused and what you can't do without it, and give it written out, so that the user can approve it the next time they type `/ordewell:run`, or run it in their own terminal. Then stop, unless the step says how to go on without it. Don't ask again for a command the user refused in this conversation unless they ask you to run it.
- Run every command from the current working directory, the project's root.
- If step 1 chose `~/.local/bin/ordewell`, write it in place of `ordewell` in every command below. Don't print or show the environment.
- Once the session's full id is known, the one given above or the one the plan check in step 2 reads, write `--session-id <id>`, with that id, into every `ordewell` command after it, the ones you run and the ones you give the user: `status`, `merge review`, `retry`, `skip`, `checkpoint`, `cancel`, `task-runner`, `task-model`, `approve`, `complete`, `force-start`, `run --unattended`, `merge` and `merge discard`. Without it, each acts on the last plan made in this folder, which is another one once a plan has been made here since.
- Run each `ordewell` command as its own Bash call, as written below: not chained with `;` or `&&`, and not through a variable. Claude Code matches its permission rules against the command as written.
- `ordewell merge`, `ordewell merge discard` and `ordewell tui` need a terminal, which Claude Code's Bash tool and its `!` commands don't have: when you name one of them to the user, say to run it in their own terminal.
- When the user asks you to mark a task complete, first run the plan check in step 2, and read that task's `type` and the run's `advancesOnItsOwn` there. Run `ordewell complete <n> --session-id <id>` only for a manual task, whose `type` is `user`: Ordewell marks it without a question, while a run is going too. For a task of any other type, run nothing, because `ordewell complete` asks first there, which the Bash tool can't answer:
  - if the file shows `"advancesOnItsOwn": true`, the run is starting, running or waiting to resume, and Ordewell refuses to mark a task that isn't a manual task until the run has stopped, in the user's terminal too. Tell the user so. Don't write out or quote an `ordewell complete` command for that task in this reply, not even as one you will give later: it would be refused. Give it only once a plan check shows `"advancesOnItsOwn": false`. A user who wants the run stopped first runs `ordewell stop --session-id <id>` in their own terminal: don't run it yourself unless they ask, and don't offer `ordewell cancel` for this, which stops one task's agent, not the run;
  - otherwise, tell the user to run `ordewell complete <n> --session-id <id>` in their own terminal, without `--yes`, where it asks them first.

  If `ordewell complete` prints that the task "cannot be marked complete while the plan is starting, running or waiting to resume", whether to you or to the user in their terminal, pass that line on, and don't write out a command for that task until a plan check shows `"advancesOnItsOwn": false`.
- If an `ordewell` command prints a notice that Ordewell checks for a new version once a day, show that notice to the user word for word, in the same message as that step's result.

## 1. Find ordewell

Run `command -v ordewell`. If it prints a path, write `ordewell` in every command below. If it prints nothing, run `test -x ~/.local/bin/ordewell && echo found`, and if that prints `found`, use `~/.local/bin/ordewell` in place of `ordewell` below. If neither finds it, tell the user that this plugin needs the `ordewell` command-line tool, version 0.14.9 or later, and that Ordewell's own install instructions are at https://ordewell.ai/docs#install: don't install it, and don't write out an install command yourself. Once it is installed, they plan with `/ordewell:plan` first. Then stop.

Once Ordewell is found, run `ordewell --version`. This plugin needs 0.14.9 or later and was tested with 0.14.9. If the version is older, tell the user that this plugin needs 0.14.9 or later, and that they upgrade Ordewell themselves, as its install instructions at https://ordewell.ai/docs#install describe: don't upgrade it yourself, and don't write out an upgrade command. Tell them also that an upgrade leaves Ordewell's local server running as it was: a server of 0.13.4 to 0.14.8 is driven by the new version as it is, old code and the environment it was started with included, which may hold a Claude Code session's variables that it passes on to the agent sessions it starts. Once no Ordewell run is going anywhere on this machine, in any folder, in `ordewell tui` or in the Ordewell desktop app, they stop it with `ordewell stop --server` in their own terminal, so that the next `/ordewell:run` starts a clean one of the new version. `ordewell stop --server` cuts off a run that is going there. Then tell them to type `/ordewell:run` again, with the session given above if there was one, and stop.

## 2. Start the run

First run `ordewell status`, or `~/.local/bin/ordewell status` if step 1 chose that form.

It is the first command here that talks to Ordewell's local server, and it only reads: it lists the sessions in this folder. If no server is running, it starts one, and Ordewell starts it without this Claude Code session's variables, so that the agent sessions it starts aren't counted as this session's children. If the one running is an idle server of Ordewell 0.13.0 to 0.13.3, which this version can't drive, it stops it and starts one for this version, and says so in a line that starts "Stopped the idle server": pass that line on to the user. One older than 0.13.0 can't say whether work is under way there, so it is left running and the command fails, asking for `ordewell stop --server`. Any other server, one of 0.13.4 to 0.14.8 left from before an upgrade included, is left as it is, with its own code and the environment it was started with. If it fails, for example because the server left by an older Ordewell has work under way or is older than 0.13.0, show the error lines as printed and stop.

### The plan check

Before each `ordewell run --unattended`, this first one and every later one, check the plan for open manual tasks. Run:

```
ordewell status --session-id <id> --output .ordewell/plugin-plan-<id>.json
```

with the session's id in both places. Before the first run, that is the session given above. If none was given, use the eight characters after `Last active:` in the output of `ordewell status`, in both places: that line names the session that `ordewell run` runs when it is given none, and `--session-id` takes the last eight characters of a session's id. If that output has no `Last active:` line, skip the check: `ordewell run` then says that no session was found.

The command writes the session as JSON to that file, inside Ordewell's own folder, which git ignores. Search it with the Grep tool if this session has one, and otherwise with this command, which only reads:

```
grep -E '^\s*"(order|title|type|status|advancesOnItsOwn)":|"id": "session-' .ordewell/plugin-plan-<id>.json
```

- The first `"id": "session-…"` line holds the session's full id, `session-` included. If you used the eight characters, use that full id from then on, in every command, this skill's first `ordewell run` included.
- The tasks are listed twice, first for the plan and then for the run. Read the second list, the run's, where each task's `order`, `title`, `type` and `status` come one after the other.
- A task whose `type` is `user` is a manual task: a person does it and marks it complete, and no run starts it. It is open unless its `status` is `completed` or `skipped`.
- `"advancesOnItsOwn": true` means that a run is starting, running or waiting to resume.

Read only the file named for this session. If this command is refused or fails, follow the refusal rule above: without it you can't tell whether the run would wait on a manual task, so don't start the run.

### No manual task is open

Run this in the foreground with the Bash tool's longest timeout, 600000 milliseconds:

```
ordewell run --unattended --session-id <id>
```

with the id the plan check gave. Only if the check was skipped because no session was found, leave `--session-id` out: the command then says that no session was specified, and exits `1`. The command prints which session it runs, then a line for each task as it starts, passes or fails. It returns when the run ends, or when a task waits on a person at a checkpoint, an input, an approval or a hold, which `--unattended` reports instead of waiting. A run that outlasts the timeout moves to the background on its own and keeps going: wait for its completion notice instead of polling it with `sleep`. If the user asks for progress meanwhile, run `ordewell status --session-id <id>`. Running it in the foreground keeps a short run and the review in step 4 in one turn, so that the user gets the outcome and the diff in the same reply.

### A manual task is open

Don't run `ordewell run --unattended` with the long timeout. On Ordewell 0.14.9 to 0.14.11 it doesn't return while a manual task is open: it starts what can start, then waits for that task without naming it, at most with a line that says "nothing could start yet", even once nothing else can start, and running it again waits the same way. Instead:

1. Tell the user first, before you start the run, for each open manual task: its number and title; that it is theirs to do, and that the tasks that wait for it don't start until it is marked complete; and that they mark it, once it is done, by asking you, in `ordewell tui` in their own terminal, or in the Ordewell desktop app. The run does every task it can meanwhile.
2. Start the run in the foreground with a timeout of 60000 milliseconds, one minute: `ordewell run --unattended --session-id <id>`. If it returns within that minute, report it as step 3 says. If it is still going after it, Claude Code moves it to the background or stops it there; either way the run goes on in Ordewell's server. Then give the user, in this reply, the task lines it printed and the manual tasks that wait on them, and end the reply: don't wait for it, don't poll it, and don't run `ordewell run --unattended` again while a manual task is open.
3. When the user asks you to mark a manual task complete, run `ordewell complete <n> --session-id <id>`. Then run the plan check again. While a manual task is still open, name it, as in 1. Once none is open, run `ordewell run --unattended --session-id <id>` as under "No manual task is open": it joins the run that is going, and returns when it ends. Report the run once: if the completion notice of the command you started in 2 comes in after that, don't report it again. If that notice comes in first, because the user marked the task in `ordewell tui` or in the desktop app and the run has ended, report from it as step 3 says.

If it stops before starting because the working tree has uncommitted changes ("Nothing was started. Re-run with `--stash`..."), tell the user. Offer `ordewell run --unattended --stash --session-id <id>`, which stashes the tracked changes first, and run it only if they agree, with the timeout the plan check chose. If they would rather commit or stash the changes themselves, tell them to type `/ordewell:run` again afterwards, with the session given above if there was one, written out. Don't use `--without-isolation` unless the user asks for it.

If it can't write to `~/.ordewell` or reach `127.0.0.1` while Claude Code's sandbox is on, tell the user to add `"excludedCommands": ["ordewell *", "~/.local/bin/ordewell *"]` to the `sandbox` block of their Claude Code settings, or to run the command in their own terminal. The second entry covers `~/.local/bin/ordewell`, the form step 1 chose when that folder isn't on PATH. Don't retry it outside the sandbox yourself.

## 3. Report the outcome

Report from the output, not from what you expect it to say. The exit status means:

- `0`: every task completed or was skipped. A stopped run also exits 0.
- `2`: a task failed or is blocked.
- `3`: the run waits on a person, such as a checkpoint, an input, an approval or a task on hold. Ordewell keeps the run as it is: nothing was answered, stopped or skipped.
- `1`: the command itself failed: no server, a lost connection, or a refused start.

In every command below, `<n>` is the task's order number. Before you name one, run `ordewell status --session-id <id>`, with the session the run printed: each task's line starts with its number and a `.`, such as `2.` or, for a subtask, `2.1.`. Write that number into each command you give the user or run, as in `ordewell retry 2 --session-id <id>`: never leave `<n>` in it. The run's lines for a task's progress label it `#` and the last four characters of its id, such as `#be4c`, and `ordewell status` shows the last eight in brackets. These commands accept neither, so never write either into one. A command that the run prints on a line about a wait, such as `ordewell checkpoint 2 approve|reject [reason]`, already names the task in a form these commands accept, but not the session: add `--session-id <id>` to it.

For `0` or `2`, give the `Done.` line with its counts, each failed or blocked task with the reason as printed, and the `Run finished on <branch>` line.

For each failed or blocked task, give its reason and offer these next steps, with their commands written out, and run one only if the user asks for it or agrees:

- `ordewell retry <n>` starts the task again. If the reason points at its agent or model, `ordewell task-runner <n> <runner>` or `ordewell task-model <n> <model>` first changes them. Then `ordewell run --unattended`, as in step 2, runs it and follows it.
- `ordewell skip <n>` ends the task without landing its work, and lets the tasks that wait for it go ahead: follow it with `ordewell run --unattended` too.

Then go on to step 4 for the work that did land.

For `3`, give each line that says what waits ("Checkpoint", "Waiting for input", "Waiting for approval", "On hold", "Waits on a person") and the command the output names to answer it. Ask the user what to do, and run their answer only when they give it:

- A checkpoint: `ordewell checkpoint <n> approve` takes no text, and says only yes. To give the task an answer or an instruction, such as the user's reply to its question, run `ordewell checkpoint <n> reject "<the user's answer>"`: the task is told that text and goes on with it. The CLI writes this as `approve|reject [reason]`; the reason goes with `reject` only.
- A wait whose answer is `ordewell tui`, then `/open <n>`: that needs a terminal, so tell the user to run it in their own terminal.
- A wait whose reason is a failed sign-in, such as `401 Unauthorized`, "not signed in" or "not logged in": the task's agent can't start, and the wait doesn't end on its own. Offer to move the task to an agent that is signed in, with these commands, one at a time, and run them only if the user agrees:
  1. `ordewell cancel <n>`, which stops the task's runner. Its agent can't be changed while it runs.
  2. `ordewell task-runner <n> <runner>`, with the id, as `ordewell runners` lists it, of an agent the user chooses, such as `claude-code`. It re-picks the task's model, effort and mode for that agent: run `ordewell status --session-id <id>` and tell the user which model the task now has, and offer `ordewell task-model <n> <model>` if they want another.
  3. `ordewell retry <n>`.
  4. `ordewell run --unattended`, as in step 2.

  If the user would rather sign in to that agent, they do so in their own terminal, and the same steps without `task-runner` start the task again.

Then follow the run on by running `ordewell run --unattended` again, the same way as in step 2, after a plan check: on a run that is still going it joins the run and reports from there, and on one that has ended since, it exits `1` with a line that starts "Failed to start execution:" and goes on with "Every task has already completed", "Every task has completed or was skipped" or "Nothing is left for a run to start". In that case, `ordewell status --session-id <id>` shows how each task ended.

For `1`, show the error as printed. After "Failed to start execution:", each of these means nothing was left for `ordewell run` to start:

- "Every task has already completed" or "Every task has completed or was skipped": no task is left. Go on to step 4: the run's work may still wait for review.
- "Nothing is left for a run to start": the tasks left are manual tasks, which no run starts, and the line names them. A person does each one and marks it complete: with `ordewell complete <n> --session-id <id>`, which you run only when the user asks, or in `ordewell tui` in their own terminal, or in the Ordewell desktop app. If the line says that every AI and script task has completed, go on to step 4 too.

### Answers in a later turn

When you run the user's answer in their reply, such as a checkpoint answer, a `--stash` yes, a retry or the sign-in steps, each command asks for approval as before, `ordewell status` and `ordewell merge review` included. If `ordewell merge review` is refused, in any turn, say so, and don't describe the run's diff from the task lines or handoffs in its place: give the user `ordewell merge review --session-id <id>`, with the id written out, to approve the next time or to run in their own terminal.

## 4. Review: the user lands the work

If the output has no `Run finished on <branch>` line because the folder isn't a git repository (the run says "Not a git repository: tasks run in the workspace root without worktree isolation"), the tasks have already edited the files in this folder. Say so, name the files the task lines report, and don't run `ordewell merge review`: there is no run branch, and nothing to merge or discard.

Otherwise Ordewell keeps the run's work on its own branch, the one the `Run finished on <branch>` line names, and the user's branch is unchanged. When the run has landed work, or every task had already completed, run `ordewell merge review --session-id <id>`, which only reads: it prints the diff of that branch against the user's. Summarise which files changed and how, then tell the user how to land it themselves, in their own terminal, with the session's id written out in place of `<id>`, and with `~/.local/bin/ordewell` in place of `ordewell` if step 1 chose that form:

- `ordewell merge --session-id <id>` merges the run into their branch, and asks first;
- `ordewell merge discard --session-id <id>` drops it, and asks first;
- `ordewell tui`, or the Ordewell desktop app, shows each task's transcript and diff.

From inside Claude Code, `! ordewell merge` can't ask its question and changes nothing. A user who has reviewed the diff and wants to land it from here types `!` and `ordewell merge --yes --session-id <id>` themselves. Write that whole line out for them, in a code block, with the session id spelled out: don't only describe it. If step 1 chose `~/.local/bin/ordewell`, write the line as `! ~/.local/bin/ordewell merge --yes --session-id <id>`: a `!` command runs with this session's PATH, where `ordewell` alone isn't found. `--yes` is their own confirmation. If it then asks a further question, such as about checks that didn't pass, it stops, and the user runs `ordewell merge` in their own terminal.

Don't run `ordewell merge` or `ordewell merge discard` yourself, with or without `--yes`, even when asked to "finish up": tell the user the command, so they run it.
