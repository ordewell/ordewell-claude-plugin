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
  - `ordewell run --unattended`: to start the run. Its tasks are sessions of Claude Code, Codex or the other agents in the plan, on the user's own logins, each in its own git worktree under `.ordewell/worktrees/`, and the command follows them until the run ends or waits on the user;
  - `ordewell merge review --session-id <id>`: to show the diff of the run's branch against the user's branch, once the run has landed work. It only reads.

  Say also that the user's answers to the run, such as a retry or a checkpoint answer, are further commands that ask too. Then run them one at a time, as the steps say.
- If a command is refused, by the user or by their settings, don't run it again in this turn, and don't get its result another way: not with the other form of `ordewell` that step 1 names, not chained into another command, not through a script and not through another tool. Say which command was refused and what you can't do without it, and give it written out, so that the user can approve it the next time they type `/ordewell:run`, or run it in their own terminal. Then stop, unless the step says how to go on without it. Don't ask again for a command the user refused in this conversation unless they ask you to run it.
- Run every command from the current working directory, the project's root.
- If step 1 chose `~/.local/bin/ordewell`, write it in place of `ordewell` in every command below. Don't print or show the environment.
- Once `ordewell run` has named the session, the one given above or the one in its `Using last session: <id>` line, write `--session-id <id>`, with that id, into every `ordewell` command after it, the ones you run and the ones you give the user: `status`, `merge review`, `retry`, `skip`, `checkpoint`, `cancel`, `task-runner`, `task-model`, `approve`, `complete`, `force-start`, `run --unattended`, `merge` and `merge discard`. Without it, each acts on the last plan made in this folder, which is another one once a plan has been made here since.
- Run each `ordewell` command as its own Bash call, as written below: not chained with `;` or `&&`, and not through a variable. Claude Code matches its permission rules against the command as written.
- `ordewell merge`, `ordewell merge discard` and `ordewell tui` need a terminal, which Claude Code's Bash tool and its `!` commands don't have: when you name one of them to the user, say to run it in their own terminal. So does `ordewell complete` on a task that isn't a manual task: it asks first, and from the Bash tool it prints "Nothing was changed. Pass --yes to confirm without a prompt." and changes nothing. If it prints that, don't run it again: tell the user to run `ordewell complete <n> --session-id <id>` in their own terminal, where it asks them first.
- If an `ordewell` command prints a notice that Ordewell checks for a new version once a day, show that notice to the user word for word, in the same message as that step's result.

## 1. Find ordewell

Run `command -v ordewell`. If it prints a path, write `ordewell` in every command below. If it prints nothing, run `test -x ~/.local/bin/ordewell && echo found`, and if that prints `found`, use `~/.local/bin/ordewell` in place of `ordewell` below. If neither finds it, tell the user that this plugin needs the `ordewell` command-line tool, version 0.14.9 or later, and that Ordewell's own install instructions are at https://ordewell.ai/docs#install: don't install it, and don't write out an install command yourself. Once it is installed, they plan with `/ordewell:plan` first. Then stop.

Once Ordewell is found, run `ordewell --version`. This plugin needs 0.14.9 or later and was tested with 0.14.9. If the version is older, tell the user that this plugin needs 0.14.9 or later, and that they upgrade Ordewell themselves, as its install instructions at https://ordewell.ai/docs#install describe: don't upgrade it yourself, and don't write out an upgrade command. Tell them also that an upgrade leaves Ordewell's local server running as it was: a server of 0.13.4 to 0.14.8 is driven by the new version as it is, old code and the environment it was started with included, which may hold a Claude Code session's variables that it passes on to the agent sessions it starts. Once no Ordewell run is going anywhere on this machine, in any folder, in `ordewell tui` or in the Ordewell desktop app, they stop it with `ordewell stop --server` in their own terminal, so that the next `/ordewell:run` starts a clean one of the new version. `ordewell stop --server` cuts off a run that is going there. Then tell them to type `/ordewell:run` again, with the session given above if there was one, and stop.

## 2. Start the run

First run `ordewell status`, or `~/.local/bin/ordewell status` if step 1 chose that form.

It is the first command here that talks to Ordewell's local server, and it only reads: it lists the sessions in this folder. If no server is running, it starts one, and Ordewell starts it without this Claude Code session's variables, so that the agent sessions it starts aren't counted as this session's children. If the one running is an idle server of Ordewell 0.13.0 to 0.13.3, which this version can't drive, it stops it and starts one for this version, and says so in a line that starts "Stopped the idle server": pass that line on to the user. One older than 0.13.0 can't say whether work is under way there, so it is left running and the command fails, asking for `ordewell stop --server`. Any other server, one of 0.13.4 to 0.14.8 left from before an upgrade included, is left as it is, with its own code and the environment it was started with. If it fails, for example because the server left by an older Ordewell has work under way or is older than 0.13.0, show the error lines as printed and stop.

Then run this in the foreground with the Bash tool's longest timeout, 600000 milliseconds:

```
ordewell run --unattended [--session-id <id>]
```

In this first run, pass `--session-id` only when a session was given above; without it, the command prints `Using last session: <id>`, and from then on that is the session's id. The command prints which session it runs, then a line for each task as it starts, passes or fails. It returns when the run ends, or when the run waits on a person, which `--unattended` reports instead of waiting. A run that outlasts the timeout moves to the background on its own and keeps going: wait for its completion notice instead of polling it with `sleep`. If the user asks for progress meanwhile, run `ordewell status --session-id <id>`. Running it in the foreground keeps a short run and the review in step 4 in one turn, so that the user gets the outcome and the diff in the same reply.

If it stops before starting because the working tree has uncommitted changes ("Nothing was started. Re-run with `--stash`..."), tell the user. Offer `ordewell run --unattended --stash`, with `--session-id <id>` if a session was given, which stashes the tracked changes first, and run it only if they agree. If they would rather commit or stash the changes themselves, tell them to type `/ordewell:run` again afterwards, with the session given above if there was one, written out. Don't use `--without-isolation` unless the user asks for it.

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

Then follow the run on by running `ordewell run --unattended` again, the same way as in step 2: on a run that is still going it joins the run and reports from there, and on one that has ended since, it exits `1` with a line that starts "Failed to start execution:" and goes on with "Every task has already completed", "Every task has completed or was skipped" or "Nothing is left for a run to start". In that case, `ordewell status --session-id <id>` shows how each task ended.

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
