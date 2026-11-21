# Ordewell for Claude Code

Give Codex, OpenCode and Claude Code tasks from one plan, without leaving Claude Code.

Type `/ordewell:plan` and a goal. [Ordewell](https://ordewell.ai) reads your repository and writes a plan: a list of tasks with what each waits for, and the coding agent and model each one runs on. You read it and change it before anything runs. Then `/ordewell:run` runs the tasks side by side, each one its own agent session on its own git branch and worktree. A task passes only when its agent reports it done, or its script exits cleanly. Your branch stays as it is until you merge the run yourself.

Claude Code already runs subagents of its own. What Ordewell adds is the plan you can edit before it runs, and the choice of agent per task: Claude Code for one task, Codex or OpenCode for another, in the same plan.

## Install

In Claude Code (v2.1.275 or later):

```
/plugin install ordewell --marketplace ordewell/ordewell-claude-plugin
```

Or from your shell:

```
claude plugin install ordewell --marketplace ordewell/ordewell-claude-plugin
```

On an older Claude Code, add the marketplace first, then install:

```
/plugin marketplace add ordewell/ordewell-claude-plugin
/plugin install ordewell@ordewell
```

The plugin drives the `ordewell` command-line tool. If it isn't installed, `/ordewell:plan` offers to install version 0.14.5 and runs the installer only if you say yes:

- macOS and Linux: `curl -fsSL https://ordewell.ai/install.sh | sh -s -- --version 0.14.5`, a standalone binary in `~/.local/bin`, no Node.js needed. Or `brew install ordewell/tap/ordewell`.
- Windows: `npm install -g ordewell@0.14.5` (Node.js 20.3 or later).

## Use

From your project's folder:

```
/ordewell:plan add a multiply function to math.js, with a test
```

Claude shows the plan as Ordewell printed it. Ask Claude to change it ("give task 2 to Codex", "task 3 doesn't need task 1"), or edit it yourself with `ordewell task-runner`, `ordewell task-deps`, `ordewell add-task` and `ordewell remove-task`, or in `ordewell tui`. Then:

```
/ordewell:run
```

When the run ends, Claude summarises the diff from `ordewell merge review`. You land it with `ordewell merge`, or drop it with `ordewell merge discard`.

To use another agent for some tasks, say so in the goal ("use Codex for the tests"), or turn it on with `ordewell runners codex on`. `ordewell runners` lists the agents Ordewell found, and whether each one is signed in.

## What it runs on your machine

The plugin is two skills: text instructions that Claude follows. It has no hooks, no MCP server, and no code of its own, and it sends nothing anywhere. On your machine, Claude runs:

- `command -v ordewell` and `ordewell --version`, to find the tool;
- with your consent only, the installer above, which downloads Ordewell from ordewell.ai and GitHub Releases, or from npm on Windows;
- `ordewell planner` and `ordewell runners`, which list the planner and the agents. If no planner is chosen yet, `ordewell planner claude-code`, which saves Claude Code as the planner in Ordewell's settings;
- `ordewell plan --goal "..." --no-chat`, which starts Ordewell's local server (port 3742 on 127.0.0.1) if it isn't running, and a planner session of your own coding agent, which reads the repository and changes nothing in it;
- `ordewell run --unattended`, which starts the tasks: agent sessions of Claude Code, Codex or the other agents in the plan, on your own logins, each in a git worktree under `.ordewell/worktrees/`;
- `ordewell status` and `ordewell merge review`, which only read.

The skills tell Claude never to pass `--yes`, never to turn on Ordewell's Full mode, autorun or automerge, and never to run `ordewell merge` or `ordewell merge discard`: merging is yours. The skills pre-approve only the read-only commands above, and only in the turn you invoke them; your Claude Code permission settings decide every other command, and in the default mode Claude Code asks you first.

Ordewell keeps its state in `~/.ordewell` and in `.ordewell/` in your project, where the `.gitignore` it writes keeps that state out of git. Its server stays running after a run; `ordewell stop --server` stops it. What the `ordewell` tool itself sends, a daily version check to ordewell.ai and a public model catalog read from models.dev, is described under [What Ordewell sends](https://github.com/ordewell/ordewell#what-ordewell-sends), with the settings that turn each off. Your coding agents talk to their own providers as they always do.

If Claude Code's sandbox is on, `ordewell` can't run inside it, because it starts agent sessions of its own. Add `"excludedCommands": ["ordewell *"]` to the `sandbox` block of your Claude Code settings, or run the `ordewell` commands in your own terminal.

## Requirements

- Claude Code, signed in. It is also the planner unless you choose another in `ordewell planner`.
- macOS, Linux or Windows.
- git 2.38 or later is recommended: it gives each task its own worktree. Without git, tasks run in your folder.
- Optional: Codex, OpenCode or another agent Ordewell supports, installed and signed in, to give tasks to it.
- No Ordewell account and no API key.

## Links

- Ordewell: [ordewell.ai](https://ordewell.ai), [documentation](https://ordewell.ai/docs)
- Ordewell's source: [github.com/ordewell/ordewell](https://github.com/ordewell/ordewell)
- This plugin: [github.com/ordewell/ordewell-claude-plugin](https://github.com/ordewell/ordewell-claude-plugin), [issues](https://github.com/ordewell/ordewell-claude-plugin/issues)

## License

The plugin is licensed under the [Apache License 2.0](LICENSE). The name "Ordewell", the Ordewell wordmark and the Ordewell logos, including `assets/ordewell-icon.png`, are not licensed under Apache-2.0: see [NOTICE](NOTICE).
