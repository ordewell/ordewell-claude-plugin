# Ordewell for Claude Code

Give Codex, OpenCode and Claude Code tasks from one plan, without leaving Claude Code.

Type `/ordewell:plan` and a goal. [Ordewell](https://ordewell.ai) reads your repository and writes a plan: a list of tasks with what each waits for, and the coding agent and model each one is given, which Claude names for each task. You read it and change it before anything runs. Then `/ordewell:run` runs the tasks side by side, each one its own agent session on its own git branch and worktree. A task passes only when its agent reports it done, or its script exits cleanly. In a git repository, your branch stays as it is until you merge the run yourself. In a folder that isn't one, the tasks edit its files directly.

Claude Code already runs subagents of its own. What Ordewell adds is the plan you can edit before it runs, and the choice of agent per task: Claude Code for one task, Codex or OpenCode for another, in the same plan.

## Install

In Claude Code (v2.1.275 or later):

```
/plugin install ordewell --marketplace ordewell/ordewell-claude-plugin
```

Or from your shell (v2.1.292 or later):

```
claude plugin install ordewell --marketplace ordewell/ordewell-claude-plugin
```

On an older Claude Code, add the marketplace first, then install. In Claude Code:

```
/plugin marketplace add ordewell/ordewell-claude-plugin
/plugin install ordewell@ordewell
```

Or from your shell:

```
claude plugin marketplace add ordewell/ordewell-claude-plugin
claude plugin install ordewell@ordewell
```

The plugin drives the `ordewell` command-line tool, version 0.14.9 or later, which you install yourself: follow [Ordewell's install instructions](https://ordewell.ai/docs#install). The plugin never installs or upgrades it. If `ordewell` isn't found, or is older than 0.14.9, `/ordewell:plan` and `/ordewell:run` say so, point you to those instructions, and stop.

After you upgrade Ordewell, the server of the old version keeps running, as described below: once no Ordewell run is going, stop it with `ordewell stop --server` in your own terminal, and the next `/ordewell:plan` starts one of the new version.

## Use

From your project's folder, type a goal after `/ordewell:plan`. For example:

```
/ordewell:plan add a multiply function to math.js, with a test
```

```
/ordewell:plan add a subtract function to math.js, and use Codex to write its test
```

```
/ordewell:plan add a parseDate helper in utils/date.js, then use it in report.js in place of the inline date parsing
```

The second gives one task to Codex, if Codex is installed and signed in. The third asks for two tasks, the second waiting for the first.

Claude shows the plan as Ordewell printed it, and names the agent and model each task got. Ask Claude to change it ("give task 2 to Codex", "task 3 doesn't need task 1"), or edit it yourself with `ordewell task-runner`, `ordewell task-deps`, `ordewell add-task` and `ordewell remove-task`, or in `ordewell tui` in your own terminal.

Claude retypes your goal into the `ordewell plan` command, and a long goal, especially one that repeats text, can lose words on the way: Ordewell's `Generating plan for:` line shows what it got, and Claude says so when it can't confirm an exact copy. For a long goal, write it in a file in your project and plan with a short goal that names it, such as `/ordewell:plan follow the instructions in goal.md`.

Then:

```
/ordewell:run
```

When the run ends, Claude summarises the diff from `ordewell merge review`. You land it with `ordewell merge`, or drop it with `ordewell merge discard`, in your own terminal: both ask you first, and Claude Code's `!` commands have no terminal to ask on, so there they change nothing. To land it from inside Claude Code after reading the diff, type `!` and the `ordewell merge --yes` line Claude gives you yourself, with the run's session written out: `--yes` is your confirmation, and Claude never passes it. `ordewell tui` also needs your own terminal.

A manual task is yours to do: no run starts it, and the tasks that wait for it start once you mark it complete, by asking Claude, in `ordewell tui` in your own terminal, or in the Ordewell desktop app. With a manual task open, `ordewell run --unattended` doesn't return on Ordewell 0.14.9 to 0.14.11, so Claude starts the run and stops waiting on that command after a minute: once the run has started what it can, about a minute in, Claude names each open manual task and how to mark it. The run goes on, and once you have marked them all, Claude follows it to the end.

If a task stops at a checkpoint with a question, tell Claude your answer and it passes it to the task. If a task waits because its agent isn't signed in, Claude offers to move it to another agent and start it again.

To use another agent for some tasks, say so in the goal ("use Codex for the tests"), or turn it on with `ordewell runners codex on`. `ordewell runners` lists the agents Ordewell found, and whether each one is signed in. Claude gives no task to an agent that `ordewell runners` reports as not signed in: it leaves that agent out, or, if your goal names it, asks you to sign in first.

## What it runs on your machine

The plugin is two skills: text instructions that Claude follows. It has no hooks, no MCP server, and no code of its own, and it sends nothing anywhere. On your machine, Claude runs the commands below. It writes every `ordewell` command it runs, in every turn, once a plan or a run has named its session, with `--session-id` and that session, so that it acts on that plan and not on the last one made in the folder.

- `command -v ordewell`, `test -x ~/.local/bin/ordewell` and `ordewell --version`, to find the tool and check its version;
- `git rev-parse --is-inside-work-tree`, to tell whether the folder is a git repository;
- `ordewell planner` at the start of `/ordewell:plan`, and `ordewell status` at the start of `/ordewell:run`. Each is the skill's first command that talks to Ordewell's local server (port 3742 on 127.0.0.1). If no server is running, it starts one, which stays running. If the server running is an idle one of Ordewell 0.13.0 to 0.13.3, which the current version can't drive, it stops that server and starts one for the current version. One older than 0.13.0 can't say whether work is under way there, so it is left running and the command fails, asking for `ordewell stop --server`. Any other server is left as it is, one of 0.13.4 to 0.14.8 left from before an upgrade included. The first lists the planners, the second the sessions in the folder;
- `ordewell runners`, which lists the agents. If no planner is chosen yet, `ordewell planner claude-code`, which saves Claude Code as the planner in Ordewell's settings;
- `ordewell plan --goal="..." --no-chat`, which starts a planner session of your own coding agent, which reads the repository and changes nothing in it;
- `ordewell status --session-id <id> --output .ordewell/plugin-plan-<id>.json`, which writes the plan as JSON to a file named for the plan's session in Ordewell's own folder, and `grep` on that file, so that Claude can read which agent runs each task, and, before `/ordewell:run` starts or follows a run, which tasks are manual tasks still open, and whether a run is going;
- `ordewell run --unattended`, which starts the tasks: agent sessions of Claude Code, Codex or the other agents in the plan, on your own logins, each in a git worktree under `.ordewell/worktrees/`. If your working tree has uncommitted changes, and you agree, `ordewell run --unattended --stash`, which stashes your tracked changes first: `git stash pop` brings them back;
- `ordewell status`, without `--output`, and `ordewell merge review`, which only read;
- only when you ask for it or agree to it: `ordewell runners <id> on`, the plan edits (`ordewell task-runner`, `ordewell task-model`, `ordewell task-effort`, `ordewell task-deps`, `ordewell add-task`, `ordewell remove-task`), and the answers to a waiting or failed run (`ordewell checkpoint`, `ordewell cancel`, `ordewell retry`, `ordewell skip`, and the others the run names). Claude reads a task's type before it marks it complete, and runs `ordewell complete` only on a manual task. On any other task, `ordewell complete` asks you first, which needs your own terminal, so Claude runs nothing and gives you that command to run there. While a run is starting, running or waiting to resume, Ordewell refuses it on such a task, in your terminal too, until the run has stopped: Claude tells you so, and gives you the command once the run has stopped.

The skills tell Claude never to pass `--yes`, never to turn on Ordewell's Full mode, autorun or automerge, never to install or upgrade Ordewell, and never to run `ordewell merge` or `ordewell merge discard`: merging is yours.

The plugin pre-approves no command. Claude Code's permission settings and mode decide each command above, in every turn. In Manual mode, Claude Code asks you before each one that your settings don't already allow, and before its first command each skill tells you which commands Claude Code will ask you to approve, and why each is needed. A goal with a line break in it is the exception: Claude Code can't check such an `ordewell plan` command against your settings. In Manual mode it asks you for it every time, even when you allow `ordewell plan`; where it can't ask, such as `claude -p` in its default mode, it refuses it; and in auto mode the classifier decides, as for any other command. To avoid that, write the goal in a file, as described under Use. If you refuse a command, Claude doesn't run it again or another way: it says what it couldn't do without it, gives you the command to run in your own terminal, and stops. In auto mode, which Claude Code v2.1.283 and later starts interactive sessions in, a classifier model decides instead of asking you, so `ordewell plan` and `ordewell run` can run without a prompt. Press Shift+Tab to switch modes.

Ordewell keeps its state in `~/.ordewell` and in `.ordewell/` in your project, where the `.gitignore` it writes keeps that state out of git. Its server stays running after a run; `ordewell stop --server` stops it. When Ordewell prints its one-time notice about its daily version check, Claude shows it to you. What the `ordewell` tool itself sends is described under [What Ordewell sends](https://github.com/ordewell/ordewell#what-ordewell-sends) in Ordewell's README, with the setting that turns each off: a daily version check to ordewell.ai, which [Ordewell's privacy notes](https://ordewell.ai/docs#privacy) also describe, and a read of the public model catalog from models.dev, which carries no data about you or your work. Your coding agents talk to their own providers as they always do.

Ordewell's server passes its environment on to the agent sessions it starts. From version 0.14.9, Ordewell starts its server without the variables a coding agent sets for the commands it runs: Claude Code's session id, its messaging socket and token and the others like them, and Codex's and OpenCode's. So a server started by an `ordewell` command run inside Claude Code doesn't pass that session on, and the Claude Code sessions Ordewell starts for planning and tasks don't record it as their parent. A server of Ordewell 0.13.4 to 0.14.8 that is still running when you upgrade is driven by the new version as it is, old code and environment included, and that environment may hold a Claude Code session's variables. When no run is going, stop it with `ordewell stop --server` in your own terminal, and the next `/ordewell:plan` or `/ordewell:run` starts one of the new version.

If Claude Code's sandbox is on, `ordewell` can't run inside it, because it starts agent sessions of its own. Add these two entries to the `sandbox` block of your Claude Code settings, or run the `ordewell` commands in your own terminal:

```json
"excludedCommands": ["ordewell *", "~/.local/bin/ordewell *"]
```

The skills write `~/.local/bin/ordewell` when Ordewell is installed there but that folder isn't on Claude Code's PATH, and the second entry covers that form.

## Requirements

- Claude Code, signed in. It is also the planner unless you choose another in `ordewell planner`.
- The `ordewell` command-line tool, version 0.14.9 or later, installed as [Ordewell's install instructions](https://ordewell.ai/docs#install) describe.
- macOS, Linux or Windows. On Windows, Node.js 20.3 or later and Git for Windows.
- git 2.38 or later is recommended: it gives each task its own worktree. Without git, tasks run in your folder.
- Optional: Codex, OpenCode or another agent Ordewell supports, installed and signed in, to give tasks to it.
- No Ordewell account and no API key.

## Links

- Ordewell: [ordewell.ai](https://ordewell.ai), [documentation](https://ordewell.ai/docs)
- Ordewell's source: [github.com/ordewell/ordewell](https://github.com/ordewell/ordewell)
- This plugin: [github.com/ordewell/ordewell-claude-plugin](https://github.com/ordewell/ordewell-claude-plugin), [issues](https://github.com/ordewell/ordewell-claude-plugin/issues)

## License

The plugin is licensed under the [Apache License 2.0](LICENSE). The name "Ordewell", the Ordewell wordmark and the Ordewell logos, including the icon assets/ordewell-icon.png, are not licensed under Apache-2.0: see [NOTICE](NOTICE).
