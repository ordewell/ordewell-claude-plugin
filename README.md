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

The plugin drives the `ordewell` command-line tool, version 0.14.5 or later. If it isn't installed, `/ordewell:plan` offers to install the latest release and runs the installer only if you say yes:

- macOS and Linux: `curl -fsSL https://ordewell.ai/install.sh | sh`, a standalone binary in `~/.local/bin`, no Node.js needed. Or `brew install ordewell/tap/ordewell`.
- Alpine and other musl-based Linux: `npm install -g ordewell` (Node.js 20.3 or later). The standalone binary needs glibc.
- Windows: `npm install -g ordewell` (Node.js 20.3 or later). Git for Windows is required.

If your `ordewell` is older than 0.14.5, `/ordewell:plan` offers to upgrade it, and runs the upgrade only if you say yes. `ordewell update` upgrades it the way it was installed, from version 0.13.0. An older version has no `ordewell update`, and was published only on npm: upgrade it with `npm install -g ordewell`.

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

When the run ends, Claude summarises the diff from `ordewell merge review`. You land it with `ordewell merge`, or drop it with `ordewell merge discard`, in your own terminal: both ask you first, and Claude Code's `!` commands have no terminal to ask on, so there they change nothing. To land it from inside Claude Code after reading the diff, type `!` and the `ordewell merge --yes` line Claude gives you yourself, with the run's session and the `env -u` prefix described below written out: `--yes` is your confirmation, and Claude never passes it. `ordewell tui` also needs your own terminal.

If a task stops at a checkpoint with a question, tell Claude your answer and it passes it to the task. If a task waits because its agent isn't signed in, Claude offers to move it to another agent and start it again.

To use another agent for some tasks, say so in the goal ("use Codex for the tests"), or turn it on with `ordewell runners codex on`. `ordewell runners` lists the agents Ordewell found, and whether each one is signed in. Claude gives no task to an agent that `ordewell runners` reports as not signed in: it leaves that agent out, or, if your goal names it, asks you to sign in first.

## What it runs on your machine

The plugin is two skills: text instructions that Claude follows. It has no hooks, no MCP server, and no code of its own, and it sends nothing anywhere. On your machine, Claude runs the commands below. It writes every `ordewell` command it runs, in every turn, with the `env -u` prefix described below, and, once a plan or a run has named its session, with `--session-id` and that session, so that it acts on that plan and not on the last one made in the folder.

- `command -v ordewell`, `test -x ~/.local/bin/ordewell` and `ordewell --version`, to find the tool;
- with your consent only, the installer above, which downloads Ordewell from ordewell.ai and GitHub Releases, or from npm;
- `git rev-parse --is-inside-work-tree`, to tell whether the folder is a git repository;
- `ordewell planner` at the start of `/ordewell:plan`, and `ordewell status` at the start of `/ordewell:run`. Each is the skill's first command that talks to Ordewell's local server (port 3742 on 127.0.0.1). If no server is running, it starts one, which stays running. If the server running is an idle one of Ordewell 0.13.0 to 0.13.3, which the current version can't drive, it stops that server and starts one for the current version. One older than 0.13.0 can't say whether work is under way there, so it is left running and the command fails, asking for `ordewell stop --server`. Any other server is left as it is, one of an older version from 0.13.4 on included. The first lists the planners, the second the sessions in the folder;
- `ordewell runners`, which lists the agents. If no planner is chosen yet, `ordewell planner claude-code`, which saves Claude Code as the planner in Ordewell's settings;
- `ordewell plan --goal="..." --no-chat`, which starts a planner session of your own coding agent, which reads the repository and changes nothing in it;
- `ordewell status --session-id <id> --output .ordewell/plugin-plan-<id>.json`, which writes the plan as JSON to a file named for the plan's session in Ordewell's own folder, and `grep` on that file, so that Claude can read which agent runs each task;
- `ordewell run --unattended`, which starts the tasks: agent sessions of Claude Code, Codex or the other agents in the plan, on your own logins, each in a git worktree under `.ordewell/worktrees/`. If your working tree has uncommitted changes, and you agree, `ordewell run --unattended --stash`, which stashes your tracked changes first: `git stash pop` brings them back;
- `ordewell status`, without `--output`, and `ordewell merge review`, which only read;
- only when you ask for it or agree to it: an upgrade of an `ordewell` older than 0.14.5, and after it `ordewell stop --server`, which stops the old version's server so that the next `/ordewell:plan` starts one of the new version, and which Claude runs only if you say no Ordewell run is going, because it would cut that run off; `ordewell runners <id> on`, the plan edits (`ordewell task-runner`, `ordewell task-model`, `ordewell task-effort`, `ordewell task-deps`, `ordewell add-task`, `ordewell remove-task`), and the answers to a waiting or failed run (`ordewell checkpoint`, `ordewell cancel`, `ordewell retry`, `ordewell skip`, and the others the run names).

The skills tell Claude never to pass `--yes`, never to turn on Ordewell's Full mode, autorun or automerge, and never to run `ordewell merge` or `ordewell merge discard`: merging is yours. The skills pre-approve the commands that find the tool, `git rev-parse`, and, written with the `env -u` prefix, `ordewell --version`, `ordewell planner`, `ordewell runners`, `ordewell status` and `ordewell merge review`, and only in the turn in which you type `/ordewell:plan` or `/ordewell:run`. All of them only read, except that each of these `ordewell` commands but `--version` starts Ordewell's server when none is running, or replaces an idle one of Ordewell 0.13.0 to 0.13.3, and `ordewell status` with `--output` writes the file it names, replacing any file there, with no question asked. The skills name only the `.ordewell/plugin-plan-<id>.json` file, but the pre-approval covers any path. Your Claude Code permission settings decide every other command, and every command in a later turn, such as your reply to a question Claude asks. In Manual mode Claude Code asks you first, and in a later turn it asks for these commands too: type `/ordewell:plan` or `/ordewell:run` again to have them run without asking. In auto mode, which Claude Code v2.1.283 and later starts interactive sessions in, a classifier model decides instead of asking you, so the installer, `ordewell plan` and `ordewell run` can run without a prompt. There, the only check before the installer runs is the skill's rule that Claude asks you in the conversation and installs nothing without your yes. Press Shift+Tab to switch modes.

Ordewell keeps its state in `~/.ordewell` and in `.ordewell/` in your project, where the `.gitignore` it writes keeps that state out of git. Its server stays running after a run; `ordewell stop --server` stops it. When Ordewell prints its one-time notice about its daily version check, Claude shows it to you. What the `ordewell` tool itself sends, a daily version check to ordewell.ai and a public model catalog read from models.dev, is described under [What Ordewell sends](https://github.com/ordewell/ordewell#what-ordewell-sends), with the settings that turn each off. Your coding agents talk to their own providers as they always do.

Ordewell's server keeps the environment it was started with, and passes it on to the agent sessions it starts. Of Claude Code's variables, Ordewell 0.14.5 removes only `CLAUDECODE` before it starts an agent session, so a server started by an `ordewell` command run inside Claude Code would pass on that session's variables, such as `CLAUDE_CODE_SESSION_ID` and its messaging socket and token, and the Claude Code sessions Ordewell starts for planning and tasks, even for a later session, would record that first session as their parent. Every `ordewell` command that talks to the server starts one when none is running, whichever turn it runs in. That is why Claude runs every `ordewell` command, in every turn, with `env -u` and these fourteen names in front: `CLAUDECODE`, `CLAUDE_CODE_SESSION_ID`, `CLAUDE_CODE_MESSAGING_SOCKET`, `CLAUDE_CODE_MESSAGING_TOKEN`, `CLAUDE_CODE_CHILD_SESSION`, `CLAUDE_CODE_CHILD_NOTES_DIR`, `CLAUDE_CODE_ENTRYPOINT`, `CLAUDE_CODE_EXECPATH`, `CLAUDE_CODE_SESSION_ATTENDED`, `CLAUDE_EFFORT`, `CLAUDE_PID`, `CLAUDE_CODE_LAUNCHED_BY_SESSION_IDS`, `CLAUDE_CODE_BRIDGE_SESSION_ID` and `CLAUDE_CODE_STARTER_MAY_HOLD_CONNECTOR_FILE`. Limits remain:

- A server that is already running keeps the environment it had, and the prefix doesn't change it. One started from inside Claude Code without the prefix passes on that session's variables: for example, one started by an `ordewell` command you typed after `!` or asked Claude to run outside these skills, or by an earlier version of this plugin. When no run is going, stop it with `ordewell stop --server`. The next `/ordewell:plan` or `/ordewell:run` starts a clean one, or you start one from your own terminal, outside Claude Code, with `ordewell web --daemon`.
- A server of Ordewell 0.13.4 to 0.14.4 that is still running when you upgrade Ordewell yourself, outside `/ordewell:plan`, is driven by the new version as it is, old code and environment included. `ordewell update` tells you to restart Ordewell: `ordewell stop --server`, when no run is going, does that for the server.
- A variable that a later Claude Code version adds isn't on the list.

If Claude Code's sandbox is on, `ordewell` can't run inside it, because it starts agent sessions of its own. Add these two entries to the `sandbox` block of your Claude Code settings, or run the `ordewell` commands in your own terminal:

```json
"excludedCommands": ["ordewell *", "~/.local/bin/ordewell *"]
```

These entries also cover the commands the skills write with the `env -u` prefix in front: Claude Code's sandbox matches them past that prefix. The skills write `~/.local/bin/ordewell` when Ordewell is installed there but that folder isn't on Claude Code's PATH, and the second entry covers that form.

## Requirements

- Claude Code, signed in. It is also the planner unless you choose another in `ordewell planner`.
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
