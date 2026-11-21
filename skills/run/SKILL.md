---
description: Run the Ordewell plan you reviewed. Its tasks run side by side, each in its own agent session on its own git worktree, and nothing is merged into your branch.
argument-hint: [session-id]
disable-model-invocation: true
allowed-tools: Bash(command -v ordewell) Bash(ordewell --version) Bash(ordewell status) Bash(ordewell status *) Bash(ordewell merge review) Bash(ordewell merge review *) Bash(~/.local/bin/ordewell --version) Bash(~/.local/bin/ordewell status) Bash(~/.local/bin/ordewell status *) Bash(~/.local/bin/ordewell merge review) Bash(~/.local/bin/ordewell merge review *)
---

# Run an Ordewell plan

Session: $ARGUMENTS

An empty session means the plan that `ordewell plan` made last in this folder. You start the run, follow it, and report what happened. The tasks run in agent sessions that Ordewell starts, not in this conversation.

These hold for every step:

- Never pass `--yes` to an ordewell command. Never run `ordewell auto full`, `ordewell autorun on` or `ordewell automerge on`, and never set `ORDEWELL_AUTONOMOUS_MODE`, `ORDEWELL_APPROVAL_MODE` or `ORDEWELL_APPROVAL_ALLOW`.
- Never run `ordewell merge` or `ordewell merge discard`. Landing the work is the user's decision, made with their own command.
- Never answer for the user: run `ordewell approve`, `ordewell checkpoint`, `ordewell complete`, `ordewell skip`, `ordewell retry` or `ordewell force-start` only when the user asks for that exact action.
- Don't edit files under `.ordewell/worktrees/`: those are the tasks' own checkouts.
- Run every command from the current working directory, the project's root.
- Run each `ordewell` command as its own Bash call, written as below: not chained with `;` or `&&`, and not through a variable. Claude Code matches its permission rules against the command as written.

## 1. Find ordewell

Run `command -v ordewell`. If it prints a path, write `ordewell` in every command below. If it prints nothing, run `test -x ~/.local/bin/ordewell && echo found`, and if that prints `found`, use `~/.local/bin/ordewell` in place of `ordewell` below. If neither finds it, tell the user to run `/ordewell:plan` first, and stop.

## 2. Start the run

Run this in the foreground with the Bash tool's longest timeout, 600000 milliseconds:

```
ordewell run --unattended [--session-id <id>]
```

Pass `--session-id` only when a session was given above. The command prints which session it runs, then a line for each task as it starts, passes or fails. It returns when the run ends, or when the run waits on a person, which `--unattended` reports instead of waiting. A run that outlasts the timeout moves to the background on its own and keeps going: wait for its completion notice instead of polling it with `sleep`. If the user asks for progress meanwhile, run `ordewell status`, with `--session-id <id>` for one session's tasks. Running it in the foreground keeps a short run and the review in step 4 in one turn, where this skill's read-only commands run without asking.

If it stops before starting because the working tree has uncommitted changes ("Nothing was started. Re-run with `--stash`..."), tell the user. Offer `ordewell run --unattended --stash`, which stashes the tracked changes first, and run it only if they agree. Don't use `--without-isolation` unless the user asks for it.

If it can't write to `~/.ordewell` or reach `127.0.0.1` while Claude Code's sandbox is on, tell the user to add `"excludedCommands": ["ordewell *"]` to the `sandbox` block of their Claude Code settings, or to run the command in their own terminal. Don't retry it outside the sandbox yourself.

## 3. Report the outcome

Report from the output, not from what you expect it to say. The exit status means:

- `0`: every task completed or was skipped. A stopped run also exits 0.
- `2`: a task failed or is blocked.
- `3`: the run waits on a person, such as a checkpoint, an input, an approval or a task on hold. Ordewell keeps the run as it is: nothing was answered, stopped or skipped.
- `1`: the command itself failed: no server, a lost connection, or a refused start.

For `0` or `2`, give the `Done.` line with its counts, each failed or blocked task with the reason as printed, and the `Run finished on <branch>` line.

For `3`, give each line that says what waits ("Checkpoint", "Waiting for input", "Waiting for approval", "On hold", "Waits on a person") and the command the output names to answer it, such as `ordewell checkpoint <n> approve|reject [reason]`, or `ordewell tui` and then `/open <n>`. Ask the user what to do, and run their answer only when they give it. Then follow the run on by running `ordewell run --unattended` again, the same way as in step 2: on a run that is still going it joins the run and reports from there, and on one that has ended since, it exits `1` with "Every task has already completed: there is nothing left to run". In that case, `ordewell status --session-id <id>` shows how each task ended.

For `1`, show the error as printed. "Every task has already completed" means nothing was left for `ordewell run` to start: manual tasks are never started by it, and `ordewell tui` works through them.

## 4. Review: the user lands the work

Ordewell keeps the run's work on its own branch, the one the `Run finished on <branch>` line names, and the user's branch is unchanged. When the run has landed work, run `ordewell merge review`, which only reads: it prints the diff of that branch against the user's. Summarise which files changed and how, then tell the user how to land it themselves:

- `ordewell merge` merges the run into their branch, and asks first;
- `ordewell merge discard` drops it;
- `ordewell tui`, or the Ordewell desktop app, shows each task's transcript and diff.

Don't run `ordewell merge` or `ordewell merge discard` yourself, even when asked to "finish up": tell the user the command, so they run it.
