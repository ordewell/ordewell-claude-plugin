---
description: Turn one goal into an Ordewell plan you review before anything runs. Its tasks can go to Claude Code, Codex, OpenCode or any other coding agent you have installed.
argument-hint: <goal>
disable-model-invocation: true
allowed-tools: Bash(command -v ordewell) Bash(git rev-parse --is-inside-work-tree) Bash(ordewell --version) Bash(ordewell planner) Bash(ordewell runners) Bash(ordewell status) Bash(ordewell status *) Bash(~/.local/bin/ordewell --version) Bash(~/.local/bin/ordewell planner) Bash(~/.local/bin/ordewell runners) Bash(~/.local/bin/ordewell status) Bash(~/.local/bin/ordewell status *)
---

# Plan a goal with Ordewell

The goal: $ARGUMENTS

You drive the `ordewell` command-line tool for the user. Ordewell's planner, a separate headless agent session and not you, reads the repository and writes the plan. Your job is to set it up, start it, and show its result faithfully. Don't write a plan yourself and don't start any task.

These hold for every step:

- Never pass `--yes` to an ordewell command. Never run `ordewell auto full`, `ordewell autorun on` or `ordewell automerge on`, and never set `ORDEWELL_AUTONOMOUS_MODE`, `ORDEWELL_APPROVAL_MODE` or `ORDEWELL_APPROVAL_ALLOW`. Each lets agents act without asking; a user who wants that turns it on in Ordewell themselves.
- Never run `ordewell run`, `ordewell merge` or anything else that starts or lands work. Running is `/ordewell:run`.
- Install or change nothing unless the user has agreed to it in this conversation.
- Run every command from the current working directory, the project's root, unless the user named another folder.
- Run each `ordewell` command as its own Bash call, written as below: not chained with `;` or `&&`, and not through a variable. Claude Code matches its permission rules against the command as written.
- If an `ordewell` command prints a notice that Ordewell checks for a new version once a day, show that notice to the user word for word, in the same message as that step's result. It goes to Ordewell's own output, which the user doesn't see unless you show it.

If the goal is empty, ask the user for one and stop.

## 1. Find ordewell

Run `command -v ordewell`. If it prints a path, write `ordewell` in every command below.

If it prints nothing, run `test -x ~/.local/bin/ordewell && echo found`. If that prints `found`, Ordewell is installed where its installer puts it, but that folder isn't on this session's PATH. Use `~/.local/bin/ordewell` in place of `ordewell` below, and tell the user to add `~/.local/bin` to their PATH and restart Claude Code, so that the commands this plugin lists as read-only stop asking for approval.

If neither finds it, Ordewell isn't installed. Say so, and offer to install the latest release the one way that fits this machine:

- macOS, or Linux with glibc: `curl -fsSL https://ordewell.ai/install.sh | sh`. It downloads a standalone binary from GitHub Releases into `~/.local/bin` and checks it against the release's SHA256SUMS, with no Node.js needed. Homebrew users can run `brew install ordewell/tap/ordewell` instead.
- Linux with musl, such as Alpine: `npm install -g ordewell`, which needs Node.js 20.3 or newer. The curl installer refuses this system. If it stops with "the standalone binary needs a glibc-based Linux", offer this line instead.
- Windows: `npm install -g ordewell`, which needs Node.js 20.3 or newer, and Git for Windows, which Ordewell requires there.

Ask whether to run it, then stop until the user answers. Run it only if they agree. When it has installed, stop there and tell the user to type `/ordewell:plan <their goal>` again, with the goal written out. The commands this skill lists as read-only run without asking only in the turn that invokes it, so planning in the turn after the install would ask for approval at every step.

Once Ordewell is found, run `ordewell --version`. This plugin needs 0.14.5 or later and was tested with 0.14.5. If the version is older, tell the user, suggest `ordewell update`, and stop.

## 2. Check the planner

Run `ordewell planner`. It lists everything that can plan, and `*` marks the current choice.

- If one is marked, keep it, and tell the user which one will plan.
- If none is marked, tell the user that Ordewell will plan with Claude Code on their own login, with no API key, and run `ordewell planner claude-code`. It saves that choice in Ordewell's settings.

## 3. Check the folder

Run `git rev-parse --is-inside-work-tree`. If it prints anything but `true`, this folder isn't a git repository: tell the user that the tasks will then edit the files in this folder directly, with no worktree and no run branch, so there will be nothing to merge or discard afterwards. Then go on.

## 4. Choose the agents that may run tasks

Run `ordewell runners`. It lists each coding agent Ordewell found, with its id and then its name. `✓` marks the ones enabled, which the planner may give tasks to. Sort each enabled agent by the line under it, if any:

- no line under it: signed in;
- a line that says it "is not signed in": not signed in. A task given to it can't start, and the run then waits on it;
- any other line, such as "Run codex login to sign in.": Ordewell couldn't tell. Count it as usable, and pass that line on to the user.

Choose the `--runner` flags:

- If the goal names agents, such as "use Codex for the tests" or "Claude Code only", pass each one with `--runner <id>`, for example `--runner claude-code --runner codex`. If one it names is not signed in, don't plan: give the user the sign-in line as `ordewell runners` printed it, tell them to sign in and then type `/ordewell:plan` with their goal again, and offer to plan now without that agent instead. Stop until they answer.
- If the goal names none, and every enabled agent is usable, pass no `--runner`: the planner may use every enabled agent.
- If the goal names none and an enabled agent is not signed in, pass `--runner <id>` for each enabled agent that is usable, so that no task goes to one that can't start.
- Use only ids that `ordewell runners` lists as enabled. If an agent is installed but disabled, mention `ordewell runners <id> on` and offer to run it. Run it only if the user agrees. Don't wait for this answer: plan with what is enabled.

Mixing agents in one plan is what Ordewell adds to Claude Code, so before you start the planner, always write this line, filled in from the list and your choice, with `none` where a part is empty:

> Agents this plan may use: <names>. Not signed in, left out: <names>. Installed but off: <names>.

## 5. Run the planner

Run this in the foreground with the Bash tool's longest timeout, 600000 milliseconds. Planning reads the repository and can take several minutes; a command that outlasts the timeout moves to the background on its own and keeps going, so wait for its completion notice instead of polling it with `sleep`. Running it in the foreground keeps the plan and step 6 in one turn, where this skill's read-only commands run without asking.

```
ordewell plan --goal="<the goal>" --no-chat [--runner <id> ...]
```

- Pass the goal as one shell argument joined to `--goal` by `=`, with no space, so that a goal which reads like a flag, such as `--yes`, is still taken as the goal. Escape any `"`, `$`, `\` or backtick in it. Copy the goal whole: don't shorten it, drop repeated text or reword it.
- You retype the goal into the command, and a long goal can lose words on the way. If it is longer than about 1,000 characters, or repeats a sentence several times, tell the user so before you run the command: you can't confirm that the copy is exact, and to be sure it arrives whole, they can write the goal in a file in the project and plan with a short goal that names it, such as `/ordewell:plan follow the instructions in goal.md`. Then go on.
- `--no-chat` asks for the plan in one pass: with no terminal, the planner can't ask the user questions, so it records its assumptions in the plan instead.
- When the planner asks to run a command, fetch a page or search the web, the CLI answers no and prints "Denied: no interactive terminal" or "Auto-denied". That is expected and the plan is still made. Don't run it again with `--yes`.

The output starts with `Generating plan for: "<goal>"...`: compare that goal with the goal at the top of this skill. If they differ in anything but the escaping, tell the user first, before the plan: quote what was lost or changed, say that the plan was made from the text Ordewell printed, and offer to plan again. Don't suggest running that plan until the user has answered. If you can't tell whether they match, say that plainly instead of saying they match.

If the command fails, show the error lines as printed and stop. Common causes:

- "No runner is enabled", or an unknown runner: show the output of `ordewell runners`.
- "Not logged in", or another sign-in error: the user signs in to that agent and runs `/ordewell:plan` again.
- The command can't write to `~/.ordewell`, can't reach `127.0.0.1`, or the Ordewell server doesn't start, while Claude Code's sandbox is on: Ordewell starts agent sessions of its own, which can't work inside the sandbox. Tell the user to add `"excludedCommands": ["ordewell *"]` to the `sandbox` block of their Claude Code settings, or to run the command in their own terminal. Don't retry it outside the sandbox yourself.

## 6. Show the plan

Show the plan as `ordewell plan` printed it, in a code block: from the `Plan:` line through the last task, the session id included. Don't summarise, reorder or reword it.

A task's line names its model in brackets, and sometimes the model's provider, but not the agent that runs it. To find the agent, run:

```
ordewell status --session-id <session id> --output .ordewell/plugin-plan.json
```

It writes the session as JSON to that file, inside Ordewell's own folder, which git ignores. Search it with the Grep tool if this session has one, and otherwise with this command, which only reads:

```
grep -E '^\s*"(order|title|assignedRunner)":' .ordewell/plugin-plan.json
```

Each task's `order` and `title` come before its `assignedRunner`, the id of the agent that runs it. The tasks may be listed twice, once for the plan and once for the run. A manual or script task has no `assignedRunner`.

After the plan, add briefly:

- for each task, its number, the agent that runs it, by the name `ordewell runners` gives that id, and its model from the line;
- any notes the CLI printed after the list.

Then tell the user they can change the plan before it runs, by asking you or by running:

- `ordewell task-runner <n> <runner>` to give a task to another agent, and `ordewell task-model <n> <model>` or `ordewell task-effort <n> <level>`;
- `ordewell task-deps <n> <a,b|none>` to change what a task waits for;
- `ordewell add-task --title "..."` and `ordewell remove-task <n>`;
- `ordewell tui`, the full terminal UI, or the Ordewell desktop app, to edit it on screen.

Run one of those only when the user asks for that change. End with: "When the plan looks right, run `/ordewell:run`."
