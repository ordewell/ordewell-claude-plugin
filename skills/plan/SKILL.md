---
description: Turn one goal into an Ordewell plan you review before anything runs. Its tasks can go to Claude Code, Codex, OpenCode or any other coding agent you have installed.
argument-hint: <goal>
disable-model-invocation: true
allowed-tools: Bash(command -v ordewell) Bash(ordewell --version) Bash(ordewell planner) Bash(ordewell runners) Bash(ordewell status) Bash(ordewell status *) Bash(~/.local/bin/ordewell --version) Bash(~/.local/bin/ordewell planner) Bash(~/.local/bin/ordewell runners) Bash(~/.local/bin/ordewell status) Bash(~/.local/bin/ordewell status *)
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

If the goal is empty, ask the user for one and stop.

## 1. Find ordewell

Run `command -v ordewell`. If it prints a path, write `ordewell` in every command below.

If it prints nothing, run `test -x ~/.local/bin/ordewell && echo found`. If that prints `found`, Ordewell is installed where its installer puts it, but that folder isn't on this session's PATH. Use `~/.local/bin/ordewell` in place of `ordewell` below, and tell the user to add `~/.local/bin` to their PATH and restart Claude Code, so that the commands this plugin lists as read-only stop asking for approval.

If neither finds it, Ordewell isn't installed. Say so, and offer to install version 0.14.5 the way that fits this machine:

- macOS or Linux: `curl -fsSL https://ordewell.ai/install.sh | sh -s -- --version 0.14.5`. It downloads a standalone binary from GitHub Releases into `~/.local/bin`, with no Node.js needed. Homebrew users can run `brew install ordewell/tap/ordewell` instead.
- Windows: `npm install -g ordewell@0.14.5`, which needs Node.js 20.3 or newer.

Ask whether to run it, then stop until the user answers. Run it only if they agree. When it has installed, stop there and tell the user to type `/ordewell:plan <their goal>` again, with the goal written out. The commands this skill lists as read-only run without asking only in the turn that invokes it, so planning in the turn after the install would ask for approval at every step.

Once Ordewell is found, run `ordewell --version`. This plugin was tested with 0.14.5. If a later command fails with an unknown option on an older version, suggest `ordewell update`.

## 2. Check the planner

Run `ordewell planner`. It lists everything that can plan, and `*` marks the current choice.

- If one is marked, keep it, and tell the user which one will plan.
- If none is marked, tell the user that Ordewell will plan with Claude Code on their own login, with no API key, and run `ordewell planner claude-code`. It saves that choice in Ordewell's settings.

## 3. Choose the agents that may run tasks

Run `ordewell runners`. It lists each coding agent Ordewell found. `✓` marks the ones the planner may give tasks to, and a line under an agent says how to sign in to it when it isn't signed in.

Mixing agents in one plan is what Ordewell adds to Claude Code. In the message where you start the planner, tell the user in a line or two which agents are enabled and signed in, and which ones this plan may use: the planner can give tasks to Codex, OpenCode or the others, not only to Claude Code.

- If the goal names agents, such as "use Codex for the tests" or "Claude Code only", pass each one with `--runner <id>`, for example `--runner claude-code --runner codex`. Use only ids that `ordewell runners` lists as enabled.
- Otherwise pass no `--runner`, and the planner may use every enabled agent.
- If an agent is installed but disabled, mention `ordewell runners <id> on` and offer to run it. Run it only if the user agrees. Don't wait for this answer: plan with what is enabled.

## 4. Run the planner

Run this as a background command, since planning reads the repository and can take several minutes:

```
ordewell plan --goal "<the goal>" --no-chat [--runner <id> ...]
```

- Pass the goal as one shell argument, with any `"`, `$`, `\` or backtick in it escaped.
- `--no-chat` asks for the plan in one pass: with no terminal, the planner can't ask the user questions, so it records its assumptions in the plan instead.
- When the planner asks to run a command, fetch a page or search the web, the CLI answers no and prints "Denied: no interactive terminal" or "Auto-denied". That is expected and the plan is still made. Don't run it again with `--yes`.

Wait for the command to finish instead of polling it with `sleep`. If it fails, show the error lines as printed and stop. Common causes:

- "No runner is enabled", or an unknown runner: show the output of `ordewell runners`.
- "Not logged in", or another sign-in error: the user signs in to that agent and runs `/ordewell:plan` again.
- The command can't write to `~/.ordewell`, can't reach `127.0.0.1`, or the Ordewell server doesn't start, while Claude Code's sandbox is on: Ordewell starts agent sessions of its own, which can't work inside the sandbox. Tell the user to add `"excludedCommands": ["ordewell *"]` to the `sandbox` block of their Claude Code settings, or to run the command in their own terminal. Don't retry it outside the sandbox yourself.

## 5. Show the plan

Show the plan as `ordewell plan` printed it, in a code block: from the `Plan:` line through the last task, the session id included. Don't summarise, reorder or reword it. After it, add briefly:

- the agent and model each task got, shown in brackets on its line;
- any notes the CLI printed after the list.

Then tell the user they can change the plan before it runs, by asking you or by running:

- `ordewell task-runner <n> <runner>` to give a task to another agent, and `ordewell task-model <n> <model>` or `ordewell task-effort <n> <level>`;
- `ordewell task-deps <n> <a,b|none>` to change what a task waits for;
- `ordewell add-task --title "..."` and `ordewell remove-task <n>`;
- `ordewell tui`, the full terminal UI, or the Ordewell desktop app, to edit it on screen.

Run one of those only when the user asks for that change. End with: "When the plan looks right, run `/ordewell:run`."
