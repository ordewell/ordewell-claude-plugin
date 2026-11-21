---
description: Run the Ordewell plan you reviewed. Its tasks run side by side, each in its own agent session on its own git worktree, and nothing is merged into your branch.
argument-hint: [session-id]
disable-model-invocation: true
allowed-tools: Bash(command -v ordewell) Bash(ordewell --version) Bash(env -u CLAUDECODE -u CLAUDE_CODE_SESSION_ID -u CLAUDE_CODE_MESSAGING_SOCKET -u CLAUDE_CODE_MESSAGING_TOKEN -u CLAUDE_CODE_CHILD_SESSION -u CLAUDE_CODE_CHILD_NOTES_DIR -u CLAUDE_CODE_ENTRYPOINT -u CLAUDE_CODE_EXECPATH -u CLAUDE_CODE_SESSION_ATTENDED -u CLAUDE_EFFORT -u CLAUDE_PID ordewell status) Bash(ordewell status) Bash(ordewell status *) Bash(ordewell merge review) Bash(ordewell merge review *) Bash(~/.local/bin/ordewell --version) Bash(env -u CLAUDECODE -u CLAUDE_CODE_SESSION_ID -u CLAUDE_CODE_MESSAGING_SOCKET -u CLAUDE_CODE_MESSAGING_TOKEN -u CLAUDE_CODE_CHILD_SESSION -u CLAUDE_CODE_CHILD_NOTES_DIR -u CLAUDE_CODE_ENTRYPOINT -u CLAUDE_CODE_EXECPATH -u CLAUDE_CODE_SESSION_ATTENDED -u CLAUDE_EFFORT -u CLAUDE_PID ~/.local/bin/ordewell status) Bash(~/.local/bin/ordewell status) Bash(~/.local/bin/ordewell status *) Bash(~/.local/bin/ordewell merge review) Bash(~/.local/bin/ordewell merge review *)
---

# Run an Ordewell plan

Session: $ARGUMENTS

An empty session means the plan that `ordewell plan` made last in this folder. You start the run, follow it, and report what happened. The tasks run in agent sessions that Ordewell starts, not in this conversation.

These hold for every step:

- Never pass `--yes` to an ordewell command. Never run `ordewell auto full`, `ordewell autorun on` or `ordewell automerge on`, and never set `ORDEWELL_AUTONOMOUS_MODE`, `ORDEWELL_APPROVAL_MODE` or `ORDEWELL_APPROVAL_ALLOW`.
- Never run `ordewell merge` or `ordewell merge discard`. Landing the work is the user's decision, made with their own command.
- Never answer for the user: run `ordewell approve`, `ordewell checkpoint`, `ordewell complete`, `ordewell skip`, `ordewell retry`, `ordewell force-start`, `ordewell cancel`, `ordewell task-runner` or `ordewell task-model` only when the user asks for that action, or agrees to it when you offer it.
- Don't edit files under `.ordewell/worktrees/`: those are the tasks' own checkouts.
- Run every command from the current working directory, the project's root.
- Run each `ordewell` command as its own Bash call, written as below: not chained with `;` or `&&`, and not through a variable. Claude Code matches its permission rules against the command as written.
- `ordewell merge`, `ordewell merge discard` and `ordewell tui` need a terminal, which Claude Code's Bash tool and its `!` commands don't have: when you name one of them to the user, say to run it in their own terminal.
- If an `ordewell` command prints a notice that Ordewell checks for a new version once a day, show that notice to the user word for word, in the same message as that step's result.

## 1. Find ordewell

Run `command -v ordewell`. If it prints a path, write `ordewell` in every command below. If it prints nothing, run `test -x ~/.local/bin/ordewell && echo found`, and if that prints `found`, use `~/.local/bin/ordewell` in place of `ordewell` below. If neither finds it, tell the user to run `/ordewell:plan` first, and stop.

## 2. Start the run

First run this, exactly as written, with `~/.local/bin/ordewell` in place of `ordewell` if step 1 chose it:

```
env -u CLAUDECODE -u CLAUDE_CODE_SESSION_ID -u CLAUDE_CODE_MESSAGING_SOCKET -u CLAUDE_CODE_MESSAGING_TOKEN -u CLAUDE_CODE_CHILD_SESSION -u CLAUDE_CODE_CHILD_NOTES_DIR -u CLAUDE_CODE_ENTRYPOINT -u CLAUDE_CODE_EXECPATH -u CLAUDE_CODE_SESSION_ATTENDED -u CLAUDE_EFFORT -u CLAUDE_PID ordewell status
```

It is the first command here that talks to Ordewell's local server, and it only reads: it lists the sessions in this folder. If no server is running, it starts one. If the one running was left by an older Ordewell, as after an upgrade, and is idle, it stops it and starts one for this version, and says so in a line that starts "Stopped the idle server": pass that line on to the user. A server of this version is left as it is. The server passes its environment on to the task sessions it starts, and `env -u` keeps this Claude Code session's own variables, its session id and its messaging socket and token among them, out of it. Don't run it any other way, and don't print or show the environment. Run it alone, and wait for it to finish before any other `ordewell` command: any `ordewell` command that talks to the server can start or replace it too, and one run beside it would do that with this session's variables. If it fails, for example because the server left by an older Ordewell has work under way, show the error lines as printed and stop.

Then run this in the foreground with the Bash tool's longest timeout, 600000 milliseconds:

```
ordewell run --unattended [--session-id <id>]
```

Pass `--session-id` only when a session was given above. The command prints which session it runs, then a line for each task as it starts, passes or fails. It returns when the run ends, or when the run waits on a person, which `--unattended` reports instead of waiting. A run that outlasts the timeout moves to the background on its own and keeps going: wait for its completion notice instead of polling it with `sleep`. If the user asks for progress meanwhile, run `ordewell status`, with `--session-id <id>` for one session's tasks. Running it in the foreground keeps a short run and the review in step 4 in one turn, where this skill's read-only commands run without asking.

If it stops before starting because the working tree has uncommitted changes ("Nothing was started. Re-run with `--stash`..."), tell the user. Offer `ordewell run --unattended --stash`, which stashes the tracked changes first, and run it only if they agree. Don't use `--without-isolation` unless the user asks for it.

If it can't write to `~/.ordewell` or reach `127.0.0.1` while Claude Code's sandbox is on, tell the user to add `"excludedCommands": ["ordewell *", "~/.local/bin/ordewell *"]` to the `sandbox` block of their Claude Code settings, or to run the command in their own terminal. Claude Code matches these against the command as written, so the second is the one that covers `~/.local/bin/ordewell`, the form step 1 chose when that folder isn't on PATH. Don't retry it outside the sandbox yourself.

## 3. Report the outcome

Report from the output, not from what you expect it to say. The exit status means:

- `0`: every task completed or was skipped. A stopped run also exits 0.
- `2`: a task failed or is blocked.
- `3`: the run waits on a person, such as a checkpoint, an input, an approval or a task on hold. Ordewell keeps the run as it is: nothing was answered, stopped or skipped.
- `1`: the command itself failed: no server, a lost connection, or a refused start.

In every command below, `<n>` is the task's order number. Before you name one, run `ordewell status --session-id <id>`, with the session the run printed: each task's line starts with its number and a `.`, such as `2.` or, for a subtask, `2.1.`. Write that number into each command you give the user or run, as in `ordewell retry 2`: never leave `<n>` in it. The run's lines for a task's progress label it `#` and the last four characters of its id, such as `#be4c`, and `ordewell status` shows the last eight in brackets. These commands accept neither, so never write either into one. A command that the run prints on a line about a wait, such as `ordewell checkpoint 2 approve|reject [reason]`, already names the task in a form these commands accept.

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

Then follow the run on by running `ordewell run --unattended` again, the same way as in step 2: on a run that is still going it joins the run and reports from there, and on one that has ended since, it exits `1` with "Every task has already completed: there is nothing left to run". In that case, `ordewell status --session-id <id>` shows how each task ended.

For `1`, show the error as printed. "Every task has already completed" means nothing was left for `ordewell run` to start: manual tasks are never started by it, and `ordewell tui`, in the user's own terminal, works through them. When every task has completed, go on to step 4: the run's work may still wait for review.

### Answers in a later turn

This skill's read-only commands run without asking only in the turn in which the user typed `/ordewell:run`. When you run the user's answer in their reply, such as a checkpoint answer, a `--stash` yes, a retry or the sign-in steps, Claude Code's permission settings decide every command in that turn, `ordewell status` and `ordewell merge review` included. If one of them is refused, say so, and don't describe the run's diff from the task lines or handoffs in its place: tell the user to type `/ordewell:run`, which joins the run, or, once it has ended, goes on to the review, with these commands running without asking.

## 4. Review: the user lands the work

If the output has no `Run finished on <branch>` line because the folder isn't a git repository (the run says "Not a git repository: tasks run in the workspace root without worktree isolation"), the tasks have already edited the files in this folder. Say so, name the files the task lines report, and don't run `ordewell merge review`: there is no run branch, and nothing to merge or discard.

Otherwise Ordewell keeps the run's work on its own branch, the one the `Run finished on <branch>` line names, and the user's branch is unchanged. When the run has landed work, or every task had already completed, run `ordewell merge review`, which only reads: it prints the diff of that branch against the user's. Summarise which files changed and how, then tell the user how to land it themselves, in their own terminal:

- `ordewell merge` merges the run into their branch, and asks first;
- `ordewell merge discard` drops it, and asks first;
- `ordewell tui`, or the Ordewell desktop app, shows each task's transcript and diff.

From inside Claude Code, `! ordewell merge` can't ask its question and changes nothing. A user who has reviewed the diff and wants to land it from here types `! ordewell merge --yes` themselves: `--yes` is their own confirmation. If it then asks a further question, such as about checks that didn't pass, it stops, and the user runs `ordewell merge` in their own terminal.

Don't run `ordewell merge` or `ordewell merge discard` yourself, with or without `--yes`, even when asked to "finish up": tell the user the command, so they run it.
