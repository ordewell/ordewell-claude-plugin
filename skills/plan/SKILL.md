---
description: Turn one goal into an Ordewell plan you review before anything runs. Its tasks can go to Claude Code, Codex, OpenCode or any other coding agent you have installed.
argument-hint: <goal>
disable-model-invocation: true
---

# Plan a goal with Ordewell

The goal: $ARGUMENTS

You drive the `ordewell` command-line tool for the user. Ordewell's planner, a separate headless agent session and not you, reads the repository and writes the plan. Your job is to set it up, start it, and show its result faithfully. Don't write a plan yourself and don't start any task.

These hold for every step:

- Never pass `--yes` to an ordewell command. Never run `ordewell auto full`, `ordewell autorun on` or `ordewell automerge on`, and never set `ORDEWELL_AUTONOMOUS_MODE`, `ORDEWELL_APPROVAL_MODE` or `ORDEWELL_APPROVAL_ALLOW`. Each lets agents act without asking; a user who wants that turns it on in Ordewell themselves.
- Never run `ordewell run`, `ordewell merge` or anything else that starts or lands work. Running is `/ordewell:run`.
- Never install or upgrade Ordewell, and never run an installer or a package manager for it. Change nothing else unless the user has agreed to it in this conversation.
- This skill pre-approves no command. Claude Code asks the user before each command you run, in this turn and in every later one, unless their own permission settings or permission mode allow it. Before your first command, tell the user in a few short lines what they will be asked to approve, and why each command is needed:
  - `command -v ordewell`, and if it finds nothing `test -x ~/.local/bin/ordewell && echo found`: to find Ordewell;
  - `ordewell --version`: to check that it is 0.14.9 or later;
  - `ordewell planner`: to list what can plan. It starts Ordewell's local server if none is running. If no planner is chosen yet, `ordewell planner claude-code` saves Claude Code as the planner in Ordewell's settings;
  - `git rev-parse --is-inside-work-tree`: to tell whether this folder is a git repository;
  - `ordewell runners`: to list the coding agents, and whether each is signed in;
  - `ordewell plan --goal="..." --no-chat`: to start Ordewell's planner, a session of a coding agent that reads the repository, changes nothing in it, and writes the plan. If the goal has a line break in it, say also that Claude Code asks for this command every time, even when the user's settings allow `ordewell plan`, and refuses it where it can't ask, such as in `claude -p`: it can't check a command with a line break in an argument against those settings. To plan without that, the user writes the goal in a file in the project and types `/ordewell:plan follow the instructions in <that file>`;
  - `ordewell status --session-id <id> --output .ordewell/plugin-plan-<id>.json`: to save the plan as JSON in Ordewell's own folder, so that you can read which agent runs each task, and, if this session has no Grep tool, `grep` on that file, which only reads.

  Then run them one at a time, as the steps say.
- If a command is refused, by the user or by their settings, don't run it again in this turn, and don't get its result another way: not with the other form of `ordewell` that step 1 names, not chained into another command, not through a script and not through another tool. Say which command was refused and what you can't do without it, and give it written out, so that the user can approve it the next time they type `/ordewell:plan`, or run it in their own terminal. Then stop, unless the step says how to go on without it. Don't ask again for a command the user refused in this conversation unless they ask you to run it.
- Run every command from the current working directory, the project's root, unless the user named another folder.
- If step 1 chose `~/.local/bin/ordewell`, write it in place of `ordewell` in every command below. Don't print or show the environment.
- Run each `ordewell` command as its own Bash call, as written below: not chained with `;` or `&&`, and not through a variable. Claude Code matches its permission rules against the command as written.
- If an `ordewell` command prints a notice that Ordewell checks for a new version once a day, show that notice to the user word for word, in the same message as that step's result. It goes to Ordewell's own output, which the user doesn't see unless you show it.

If the goal is empty, ask the user for one and stop.

## 1. Find ordewell

Run `command -v ordewell`. If it prints a path, write `ordewell` in every command below. If it is refused, stop as the rule above says.

If it prints nothing, run `test -x ~/.local/bin/ordewell && echo found`. If that prints `found`, Ordewell is installed where its installer puts it, but that folder isn't on this session's PATH. Use `~/.local/bin/ordewell` in place of `ordewell` below, and tell the user to add `~/.local/bin` to their PATH and restart Claude Code, so that `ordewell` runs by its name.

If neither finds it, Ordewell isn't installed, or isn't where this session can find it. Tell the user that this plugin needs the `ordewell` command-line tool, version 0.14.9 or later, and that Ordewell's own install instructions are at https://ordewell.ai/docs#install. Don't install it, and don't write out an install command yourself: point to that page. Tell them to type `/ordewell:plan <their goal>` again once it is installed, with the goal written out, and stop.

Once Ordewell is found, run `ordewell --version`. This plugin needs 0.14.9 or later and was tested with 0.14.9. If the version is older, tell the user that this plugin needs 0.14.9 or later, and that they upgrade Ordewell themselves, as its install instructions at https://ordewell.ai/docs#install describe. Don't upgrade it yourself, and don't write out an upgrade command. Tell them also that an upgrade leaves Ordewell's local server running as it was: a server of 0.13.4 to 0.14.8 is driven by the new version as it is, old code and the environment it was started with included, which may hold a Claude Code session's variables that it passes on to the agent sessions it starts. Once no Ordewell run is going anywhere on this machine, in any folder, in `ordewell tui` or in the Ordewell desktop app, they stop it with `ordewell stop --server` in their own terminal, so that the next `/ordewell:plan` starts a clean one of the new version. `ordewell stop --server` cuts off a run that is going there. Then tell them to type `/ordewell:plan <their goal>` again, with the goal written out, and stop.

## 2. Start Ordewell's server and check the planner

Run `ordewell planner`, or `~/.local/bin/ordewell planner` if step 1 chose that form.

It is the first command here that talks to Ordewell's local server. If no server is running, it starts one, and Ordewell starts it without this Claude Code session's variables, so that the agent sessions it starts aren't counted as this session's children. If the one running is an idle server of Ordewell 0.13.0 to 0.13.3, which this version can't drive, it stops it and starts one for this version, and says so in a line that starts "Stopped the idle server": pass that line on to the user. One older than 0.13.0 can't say whether work is under way there, so it is left running and the command fails, asking for `ordewell stop --server`. Any other server, one of 0.13.4 to 0.14.8 left from before an upgrade included, is left as it is, with its own code and the environment it was started with. If it fails, for example because the server left by an older Ordewell has work under way or is older than 0.13.0, show the error lines as printed and stop.

Its output lists everything that can plan, and `*` marks the current choice.

- If one is marked, keep it, and tell the user which one will plan.
- If none is marked, tell the user that Ordewell will plan with Claude Code on their own login, with no API key, and run `ordewell planner claude-code`. It saves that choice in Ordewell's settings.

## 3. Check the folder

Run `git rev-parse --is-inside-work-tree`. If it prints anything but `true`, this folder isn't a git repository. Remember it for step 6, which tells the user. Then go on. If it is refused, go on too, and in step 6 say that you couldn't tell whether this folder is a git repository.

## 4. Choose the agents that may run tasks

Run `ordewell runners`. It lists each coding agent Ordewell found, with its id and then its name. `✓` marks the ones enabled, which the planner may give tasks to. Sort each enabled agent by the line under it, if any:

- no line under it: signed in;
- a line that says it "is not signed in": not signed in. A task given to it can't start, and the run then waits on it;
- any other line, such as "Run codex login to sign in.": Ordewell couldn't tell. Count it as usable, and pass that line on to the user.

Choose the `--runner` flags:

- If the goal names agents, such as "use Codex for the tests" or "Claude Code only", pass each one with `--runner <id>`, for example `--runner claude-code --runner codex`. If one it names is not signed in, don't plan: give the user the sign-in line as `ordewell runners` printed it, and tell them to sign in and then type `/ordewell:plan` with their goal again. To plan without that agent instead, give them the exact line to type: `/ordewell:plan <their goal, with that agent's part taken out>`, written out, so that the goal this skill compares in step 5 is the new one. Stop there.
- If the goal names none, and every enabled agent is usable, pass no `--runner`: the planner may use every enabled agent.
- If the goal names none and an enabled agent is not signed in, pass `--runner <id>` for each enabled agent that is usable, so that no task goes to one that can't start.
- Use only ids that `ordewell runners` lists as enabled. If an agent is installed but disabled, mention `ordewell runners <id> on` and offer to run it. Run it only if the user agrees. Don't wait for this answer: plan with what is enabled.

Note, for step 6, this line, filled in from the list and your choice, with `none` where a part is empty:

> Agents this plan may use: <names>. Not signed in, left out: <names>. Installed but off: <names>.

"Not signed in, left out" names every enabled agent that `ordewell runners` reports as not signed in, even one the goal leaves out anyway, as "Claude Code only" leaves out Codex. Write `none` there only when no enabled agent is reported as not signed in.

You may also write it before you start the planner, but step 6 writes it in any case.

## 5. Run the planner

Run this in the foreground with the Bash tool's longest timeout, 600000 milliseconds. Planning reads the repository and can take several minutes; a command that outlasts the timeout moves to the background on its own and keeps going, so wait for its completion notice instead of polling it with `sleep`. Running it in the foreground keeps the plan and step 6 in one turn, so that the user sees the plan and its agents in the same reply.

```
ordewell plan --goal="<the goal>" --no-chat [--runner <id> ...]
```

- Pass the goal as one shell argument joined to `--goal` by `=`, with no space, so that a goal which reads like a flag, such as `--yes`, is still taken as the goal. Escape any `"`, `$`, `\` or backtick in it. Copy the goal whole: don't shorten it, drop repeated text or reword it.
- If the goal has line breaks, keep them inside the quotes: joining the lines changes the goal. Don't write it as `$'...'` either, which Claude Code also refuses to check. Claude Code asks for this command each time, as the notice before your first command said. If it is refused, follow the refusal rule above, and give the user the file route too: write the goal in a file in the project, then type `/ordewell:plan follow the instructions in <that file>`.
- You retype the goal into the command, and a long goal can lose words on the way. If it is longer than about 1,000 characters, or repeats a sentence several times, the comparison below says so.
- `--no-chat` asks for the plan in one pass: with no terminal, the planner can't ask the user questions, so it records its assumptions in the plan instead.
- When the planner asks to run a command, fetch a page or search the web, the CLI answers no and prints "Denied: no interactive terminal" or "Auto-denied". That is expected and the plan is still made. Don't run it again with `--yes`.

The output starts with `Generating plan for: "<goal>"...`: compare that goal with the goal at the top of this skill. If they differ in anything but the escaping, tell the user first, before the plan: quote what was lost or changed, say that the plan was made from the text Ordewell printed, and that to plan again they type `/ordewell:plan` with the goal again. Don't suggest running that plan until the user has answered. If you can't tell whether they match, say that plainly instead of saying they match.

If the goal is longer than about 1,000 characters, or repeats a sentence several times, write this with that comparison, before the plan, even when the two look the same: you can't confirm that your copy is exact, and to be sure a goal arrives whole, the user can write it in a file in the project and plan with a short goal that names it, such as `/ordewell:plan follow the instructions in goal.md`.

If the command fails, show the error lines as printed, then the agents line from step 4 and, if step 3 found no git repository, that note, and stop. Common causes:

- "No runner is enabled", or an unknown runner: show the output of `ordewell runners`.
- "Not logged in", or another sign-in error: the user signs in to that agent and runs `/ordewell:plan` again.
- The command can't write to `~/.ordewell`, can't reach `127.0.0.1`, or the Ordewell server doesn't start, while Claude Code's sandbox is on: Ordewell starts agent sessions of its own, which can't work inside the sandbox. Tell the user to add `"excludedCommands": ["ordewell *", "~/.local/bin/ordewell *"]` to the `sandbox` block of their Claude Code settings, or to run the command in their own terminal. The second entry covers `~/.local/bin/ordewell`, the form step 1 chose when that folder isn't on PATH. Don't retry it outside the sandbox yourself.

## 6. Show the plan

Show the plan as `ordewell plan` printed it, in a code block: from the `Plan:` line through the last task, the session id included. Don't summarise, reorder or reword it.

A task's line names its model in brackets, and sometimes the model's provider, but not the agent that runs it. To find the agent, run:

```
ordewell status --session-id <session id> --output .ordewell/plugin-plan-<session id>.json
```

with the session id the plan printed, in both places. It writes the session as JSON to that file, inside Ordewell's own folder, which git ignores. Search it with the Grep tool if this session has one, and otherwise with this command, which only reads:

```
grep -E '^\s*"(order|title|assignedRunner)":' .ordewell/plugin-plan-<session id>.json
```

Each task's `order` and `title` come before its `assignedRunner`, the id of the agent that runs it. The tasks may be listed twice, once for the plan and once for the run. A manual or script task has no `assignedRunner`. Read only the file named for this plan's session id. If the `ordewell status` command was refused or failed, don't read any other file: say that you couldn't read which agent runs each task, and that `ordewell status --session-id <session id>` shows it.

After the plan, add briefly, in the same message:

- the agents line from step 4;
- if step 3 found that this folder isn't a git repository: the tasks will edit the files in this folder directly, with no worktree and no run branch, so there will be nothing to merge or discard afterwards;
- for each task, its number, the agent that runs it, by the name `ordewell runners` gives that id, and its model from the line;
- any notes the CLI printed after the list.

Then tell the user they can change the plan before it runs, by asking you or by running these in their own terminal, with the session id the plan printed written out in place of `<session id>`:

- `ordewell task-runner <n> <runner> --session-id <session id>` to give a task to another agent, and `ordewell task-model <n> <model> --session-id <session id>` or `ordewell task-effort <n> <level> --session-id <session id>`;
- `ordewell task-deps <n> <a,b|none> --session-id <session id>` to change what a task waits for;
- `ordewell add-task --title "..." --session-id <session id>` and `ordewell remove-task <n> --session-id <session id>`;
- `ordewell tui`, the full terminal UI, or the Ordewell desktop app, to edit it on screen.

Run one of those only when the user asks for that change, with `--session-id`: without it, each acts on the last plan made in this folder, which is another one if a plan has been made here since. End with: "When the plan looks right, run `/ordewell:run`."
