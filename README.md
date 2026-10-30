# AgentMingo reporter

This is the script that connects the coding-agent threads on your computer to [AgentMingo](https://agentmingo.com).
It is the same file agentmingo.com serves at <https://agentmingo.com/reporter.mjs>, published here so you can read exactly what it does before you run it.

AgentMingo is a free open beta. The AgentMingo server is closed source. This script is open source, under the Apache-2.0 license.

## What it is

`reporter.mjs` is one file of plain JavaScript with no dependencies.
Your coding agent runs it from its hooks at a few moments in each thread.
It reports those moments to AgentMingo, so your team sees every thread in one place and knows which ones need a person.
It also hands your threads what they can't see on their own: a ticket's brief, review and CI results for their branch, teammates' live threads, your team's rules, and workflow steps one at a time.

AgentMingo isn't tied to one agent or model vendor. Today this script connects one agent, Claude Code, through Claude Code's hooks. Other agents (Cursor's own agent, Codex and the rest) aren't connected yet. Review results work the same way whichever review bot comments on the pull request: CodeRabbit is one example.

## Requirements

- macOS or Linux. Native Windows isn't supported yet.
- Node.js 18 or newer (`node --version`).
- Claude Code (`claude --version`), the agent this script connects today.

Cursor's own agent (its chat and Agent panel) is ignored. Claude Code inside Cursor works. See [Cursor](#cursor).

## Install

You get a setup command from the Sessions page in AgentMingo. It looks like this (`acme` and the code are examples; the boxes you tick add flags such as `--share-files` or `--no-share-files`, and `--reviews`):

```sh
curl -fsSL https://agentmingo.com/reporter.mjs -o ~/.agentmingo/reporter.mjs --create-dirs \
  && node ~/.agentmingo/reporter.mjs install --url https://agentmingo.com --org acme --code amc_…
```

The code works once, for 15 minutes. Install trades it for this computer's token, so the token never appears in a command line or a shell history.

To read the file before it runs, split the command in two:

1. Run the `curl` part only.
2. Read `~/.agentmingo/reporter.mjs`, and [check its sha256](#check-that-your-copy-is-a-published-one).
3. Run the `node … install …` part.

Running install again is safe. It replaces its own hooks instead of adding them twice, and it keeps your earlier choices (titles, sharing, review results, updating) unless a flag changes them.

## What install changes on your computer

**Hooks** in `~/.claude/settings.json`, with a copy of the file from before kept beside it as `settings.json.agentmingo-backup`. One hook on each of these events: `SessionStart`, `UserPromptSubmit`, `PostToolUse`, `PermissionRequest`, `Notification`, `Stop`, `StopFailure` and `SessionEnd`. `PostToolUse` runs in the background, so the agent never waits for it. A second `Stop` hook (`watch`) also runs in the background and can wake an idle thread when review results or its next workflow step arrive (see [Review results](#review-results)). On macOS and Linux, a hook looks like this, with your own home folder (`SessionEnd`'s timeout is 5; `PostToolUse` and `PermissionRequest` also have `"matcher": "*"`, and `PostToolUse` has `"async": true`):

```json
{
  "type": "command",
  "command": "\"/Users/you/.agentmingo/reporter.sh\" hook",
  "timeout": 10
}
```

The `watch` hook has `"async": true`, `"asyncRewake": true` and a timeout of 1200 seconds.

**Four permission rules**, added to `permissions.allow` in the same file:

```text
Bash("/Users/you/.agentmingo/reporter.sh" step:*)
Bash("/Users/you/.agentmingo/reporter.sh" start:*)
mcp__agentmingo__current_step
mcp__agentmingo__complete_step
```

They let the agent report a workflow step, and start a task's thread, without asking you. They allow only those two subcommands of this script, and the two tools of its own MCP server.

**An MCP server** named `agentmingo`, registered for your user with `claude mcp add -s user` (in `~/.claude.json`). It runs this script with `mcp`, and has two tools, `current_step` and `complete_step`. Install's `--no-mcp` skips it. When the `claude` command isn't found or doesn't answer, install prints the `claude mcp add` line to run yourself instead.

**The folder `~/.agentmingo`.** See [What it keeps on your computer](#what-it-keeps-on-your-computer).

**Only with `--lean-git`** (or `lean-git on`): six short git habits added to `~/.claude/CLAUDE.md`, between marker comments, with a copy of the file as it was kept beside it as `CLAUDE.md.agentmingo-backup`, when there was one. `lean-git off` takes them out.

Nothing else on your computer changes.

## Which threads report

In the script's own words:

> Which threads report: only those in a repository your organization has connected in AgentMingo (also under an earlier name, when it was renamed or moved: GitHub keeps old names working, so an older clone may still use one), or one you added (`repos add` in its folder; the Sessions page lists them and removes them), named by the folder's origin remote. A thread anywhere else (another client's repository, a side project, a folder that isn't a git repository) sends nothing about itself: no events, no lookups, and AgentMingo's notes don't reach it. One thing is asked about such a folder: when its repository shares its owner or its name with one an organization lists (acme/web-old next to acme/web, say), this computer sends that name to the organization once a day, to ask whether it's the same repository under an earlier name. AgentMingo asks GitHub, and keeps the name only if it is. Until this computer has heard an organization's list (just installed or updated), it asks for it before sending anything about a thread, and drops what it queued for threads outside it.

`node ~/.agentmingo/reporter.mjs repos` lists the repositories that report from your computer.

## What it sends

This is the list in the script's header, quoted verbatim. AgentMingo's [Privacy Policy](https://agentmingo.com/privacy#collect) lists the same things in its own words, and says how long AgentMingo keeps them.

> What it sends for those, every request but setting up and downloading an update (below) carrying this computer's connection token (`authorization: Bearer amk_…`) and the script's name and version (user-agent `agentmingo-reporter/` and a number), and like any request, your IP address:
>
> - What happens in each thread, and when: it started, you sent a message, it's working (at most every five minutes, and at once after a tool call that ran long), with how long the tool call behind it ran, it waits for you, it was answered, it replied, it ended. Each with its time, a random id of its own (so it's recorded once), and Claude Code's id for the thread.
> - Where it works: its repository name and branch, whether the branch is pushed and how many of its commits aren't, and the app it runs in.
> - Counts from the session's transcript, from when the reporter first saw the session: token counts by model (with the model's name), tool calls counted by kind (git, GitHub's command line, tests, builds, installs, other shell commands, reading files, edits, web, subagents, MCP tools, and any other tool) with how much output they produced and about how long they took, and git commands counted by subcommand (diff, log, status, push and so on) with how much output each produced. When a thread's transcript stops giving those counts (a Claude Code update can change how it writes transcripts), it says which of them is missing, so `status` and AgentMingo's admins can see it.
> - A short title (Claude Code's own session title, or the first line of your first message, with emails, phone numbers, long numbers, keys and tokens by their known shapes, private keys, credentials in addresses, headers and command lines, and secret-looking values given to a password, secret, token or key masked on this computer; turn titles off with --no-titles or `titles off`), and what a waiting session's notice asks you (masked the same way; with titles off, only whether it asks for a permission, and for which tool, or has a question).
> - Its settings: whether review results are on here, whether sessions know about each other here, whether the lean git rules are in a CLAUDE.md it reads or were given to it in a note, and while its organization measures them, which group of the test the session is in (with the rules, or without).
> - What it passed on: which review results, notices and workflow step it handed the session, so none is handed twice.
> - What it asks for: when a session starts on a branch whose name looks like a task's key, that branch and the repository, for the task's brief. Before each message, the session's id, for its workflow step (when it follows one) and, with review results on, for results and notices waiting for it. After each reply, the watcher below.
>
> With --share-files (the setup command's "Let sessions know about each other", ticked by default) it also sends the paths of files a session edits, each with when it was edited, says sharing is on, and tells each session (in one short note) which other sessions are working on which files in the same repository (their person's first name, title and branch), so they stay out of each other's way. With it off (--no-share-files, or `share-files off`), it sends no paths, your sessions get no such note, and AgentMingo names your sessions in no teammate's notes: not in their note about other sessions, and not when someone else's thread works on the same task.
>
> On a workflow (below), it also sends whether a step Claude reports is done or failed, and what Claude writes when it reports a step (for the plan step, the plan itself, with the repositories it names and whether it needs an end-to-end test, and why; AgentMingo masks the same details when it arrives), the files Claude attaches to a step (reports, screenshots, videos, logs; up to 4 MB each, 20 at most), each with its name or its path in the session's folder, and when Claude asks for its current step, the folder's repository and branch. The commands send what they name: `start` the task's key (and the repository, with --repo), `repos add` and `repos remove` a repository's name, and `repos` asks for the list. On a computer connected to several organizations, a step report and `start` go first to the organization of the folder's repository, and to the next only if that one doesn't know the step or the task. Install sends the setup command's one-time code, the organization's short name and the script's version, and nothing else; when it replaces an earlier connection, and on uninstall, it asks AgentMingo to disconnect that one. Updating downloads the new script without its token and without anything about you or your threads. It doesn't send your code or other files (unless Claude attaches one to a step), the commands Claude runs (only their kind, and git's subcommand, are counted), tool input, or the rest of the conversation.

The header is written for the one agent the script connects today, so where it says "Claude", it means that agent. The exact fields of each report are built mainly in two functions you can read: `hookToEvent()` (the moment itself) and `enrich()` (what is added to it before it is sent). `runHook()` adds which notes it passed on.

### What it doesn't send

Your code and other files (unless the agent attaches one to a workflow step), the commands the agent runs (only their kind, and git's subcommand, are counted), tool input, and the rest of the conversation. The script reads the thread's transcript on your computer to count tokens and tool calls, take the title and see which files it edits. AgentMingo's servers don't receive the conversation.

### Notes AgentMingo adds to your threads

AgentMingo sends notes back, and the script adds them to the thread: your team's rules (with the name of the admin who wrote them), the git habits when your organization has them on, a task's brief, other live threads in the same repository (with sharing on), review results and workflow steps. They become part of the thread's conversation, which your agent sends to its model provider under your own account and plan, like the rest of the conversation. Nothing on your computer changes for that.

## When it sends

Everything goes to the AgentMingo address in your setup command (`--url`, such as https://agentmingo.com). Events are sent by a separate background process, and wait in `~/.agentmingo/outbox` until they are delivered.

| When | What |
| --- | --- |
| A thread starts, you send a message, it asks for permission or has a question, it finishes replying, it ends | One event for that moment |
| While a thread works | A heartbeat at most every 5 minutes (at once after a tool call that ran a minute or more) |
| A permission prompt or question is still open after a minute | The same event again, so its person can be told |
| A thread starts on a branch that looks like a task key (`WEB-12-login`) | A lookup of that task's brief (the branch and repository), waiting a second at most |
| You send a message | With review results on: a lookup for the review results and notices waiting for the thread. In a thread that follows a workflow (also when it starts): a lookup of its current step. Each waits a second at most, and they run at the same time. Review results that became ready after the watcher stopped arrive with this message |
| After each reply, with review results on or on a workflow | The `watch` hook asks every 30 seconds for 3 minutes, then every minute until 10 minutes, then every 2 minutes, for up to 18 minutes (the agent stops it at 20, the timeout install gives it). It stops early when the branch has no pull request and the thread follows no workflow, and when you send a new message |
| At most every 15 minutes | An empty report that asks for your organization's list of repositories. It carries nothing about any thread |
| Once a day per name | The name check described in [Which threads report](#which-threads-report) |
| When an answer says a different version should run | A download of `reporter.mjs` from the same AgentMingo, without the token (at most every 6 hours for the same version, or every 10 minutes when going back; none when going back to the version kept in `reporter.prev.mjs`; see [Updates](#updates-and-the-kill-switch)) |
| When you or the agent run a command | `install` (trades the code), `repos`, `repos add` or `repos remove` (the repository's name), `start` (the task key), `step` or the MCP tools (the step's text and attached files; `current_step` sends the folder's repository and branch), `uninstall` (disconnects this computer) |

When AgentMingo doesn't answer, the script holds back:

- A lookup (a task's brief when a thread starts, or what's waiting before a message) waits a second at most. When one gets no answer, lookups pause for 60 seconds, so a slow AgentMingo costs one message a second, not every message. What the lookup would have brought comes with a later answer instead.
- Sending stops for 30 seconds, doubling up to 10 minutes (or as long as a 429 asks), then what waited is sent.
- When AgentMingo says a limit on reports was reached, the events it skipped are dropped, and nothing more is sent until the limit's window ends.
- When it refuses the connection (403: the account or organization is suspended, or you were made a viewer), nothing is asked or sent there for an hour, doubling to a day, or until install runs again.

## What it keeps on your computer

Everything is in `~/.agentmingo`:

- `reporter.mjs`: this script. `reporter.prev.mjs`: the version before the last update, kept to go back to.
- `launch.mjs`: the launcher the hooks and the MCP server run. It runs `reporter.mjs`, or the kept version when the new one doesn't load. When the new one loads but fails, it puts the kept version back as `reporter.mjs` for the next run. Its full text is the `LAUNCHER` constant in `reporter.mjs`, and it doesn't change when the script updates.
- `reporter.sh` (macOS and Linux): a short shell script that runs `launch.mjs` with the Node.js install ran with, or the `node` on your PATH once that one is gone (an nvm or Homebrew upgrade). With neither, it does nothing. Its text comes from `shimText()` in `reporter.mjs`, with your Node.js path and home folder filled in.
- `config.json`: your choices, this computer's connection token and each organization's list of repositories, readable by you only (mode 600). When AgentMingo removes this computer's only connection, it is renamed `config.json.removed`.
- `outbox/`: events waiting to be sent.
- `flush.lock`: there while the background sender runs, so only one sends at a time.
- `state/`: each thread's state, what's waiting for it, when each organization's list of repositories was last heard, the names asked about once a day, team rules, other live threads in the same repository (with sharing on), when AgentMingo last didn't answer (`backoff.json`, and `lookups.json` for the 60-second lookup pause), the version AgentMingo last said to run and the last update tried, and lock files.
- `state/<session id>.told/`: one empty file per review result or notice a thread was told, so the message hook and the `watch` hook never tell it the same thing twice. A thread's folder is removed when it ends; one left by a thread that never ended is removed, when a later thread starts, once nothing was added to it for 7 days.
- `repos.json`: where each repository is cloned, so `start` works from any folder.
- `reporter.log`: a log of problems and updates. `node ~/.agentmingo/reporter.mjs status` shows its last lines.

A downloaded update is written to `reporter.next.mjs` in `~/.agentmingo`, then checked in a scratch folder in your system's temporary folder, which is removed afterwards. If the check fails, `reporter.next.mjs` is deleted; if it passes, it becomes `reporter.mjs`.

## Review results

With review results on (`--reviews`, or `reviews on`):

> when a review bot (CodeRabbit, say) or CI's check runs report on the pull request for a session's branch, AgentMingo decides what blocks and what to fix now, and the session gets that list with its next message (asked for just before the message reaches Claude, a second at most, so results that came in after the watcher stopped aren't a message late), or, if it's idle after pushing, is woken with it: a background hook waits up to 18 minutes after each reply (Claude Code stops it at 20, the timeout install gives it), asking every 30 seconds at first, then every minute, then every two; at most 3 automatic rounds per pull request. A merge conflict on that pull request, and each step of the repository's pull request checklist, wake it too, outside those rounds. [...] `reviews off` stops all of that (a session that follows a workflow is still woken for its next step).

To wake a thread, the `watch` hook exits with code 2 and writes the note to stderr. That is how a background hook wakes a thread (`asyncRewake`). Every other hook exits 0. The woken turn runs on your own plan with your agent's provider, like any other turn.

## Cursor

> Cursor runs these hooks for its own agent too (it loads Claude Code's hooks by default). This reporter ignores Cursor's own agent for now: when the hook's input says it comes from Cursor, it exits at once and reads, sends and writes nothing. Claude Code inside Cursor (its extension, or Claude Code in Cursor's terminal) reports as usual.

The check is `fromCursor()`. It looks only at the hook's input (the fields `cursor_version` and `conversation_id`), not at environment variables. Versions 11 and newer have it.

## Updates and the kill switch

The script updates itself, so what runs later can differ from what you read today.

> It keeps itself current, so what runs later can differ from what you read here: when AgentMingo serves a newer version, it downloads it from there (the AgentMingo it reports to, such as agentmingo.com; the copy install put in ~/.agentmingo only), checks that it runs on this computer (in a scratch folder, connected to a stand-in for AgentMingo on this computer: Node loads it, `status` runs, a hook queues its event, and its sender sends it and goes back to this version when the stand-in says so), and uses it from the next hook, keeping the version before as reporter.prev.mjs. Hooks run it through launch.mjs, which runs that kept version instead when the new one doesn't load here (the kept version then puts itself back as reporter.mjs), and puts the kept version back itself when the new one loads but fails. AgentMingo can hold reporters at a version, or send them back to one (10 or newer).

Plainly:

- Updates come from agentmingo.com (the address you installed with) over HTTPS. The script refuses to update from a plain `http://` address other than this computer's own.
- Updates aren't signed. Before using a download, the script checks that it is the version asked for, starts as a Node.js script, is under 2 MB, and runs on your computer. It doesn't check who made it. It trusts the download because it came over HTTPS from the AgentMingo you connected to.
- The kill switch is AgentMingo telling reporters to go back to an earlier version (10 or newer), or to stay at one. It is the same check and swap, in the other direction.
- Each release is published to this repository by hand for now, with its sha256 in [CHANGELOG.md](CHANGELOG.md), so you can see what changed.

To turn updating off:

```sh
node ~/.agentmingo/reporter.mjs auto-update off
```

Or add `--no-auto-update` to the setup command. A newer version then comes only when you run a new setup command from the Sessions page. `node ~/.agentmingo/reporter.mjs status` shows the version you run.

## Check that your copy is a published one

[CHANGELOG.md](CHANGELOG.md) lists the sha256 of `reporter.mjs` for every version published here.

Hash the copy you run:

```sh
shasum -a 256 ~/.agentmingo/reporter.mjs     # macOS
sha256sum ~/.agentmingo/reporter.mjs         # Linux
```

Hash what agentmingo.com serves now:

```sh
curl -fsSL https://agentmingo.com/reporter.mjs | shasum -a 256
```

Your copy should match the sha256 CHANGELOG.md gives for the version `status` shows. When you run the latest version, the served file matches too, and so does `reporter.mjs` in this repository. If your copy matches no published sha256, tell us (see [Reporting a security issue](#reporting-a-security-issue)).

`launch.mjs` in `~/.agentmingo` should be exactly the `LAUNCHER` constant in your `reporter.mjs`. To compare them (importing the script this way doesn't run it):

```sh
REPORTER="$HOME/.agentmingo/reporter.mjs" node --input-type=module -e '
import { readFileSync } from "node:fs";
import { pathToFileURL } from "node:url";
const m = await import(pathToFileURL(process.env.REPORTER).href);
console.log(readFileSync(process.env.HOME + "/.agentmingo/launch.mjs", "utf8") === m.LAUNCHER ? "launch.mjs matches" : "launch.mjs differs");
'
```

`reporter.sh` differs per computer, because it names your Node.js and your home folder. Read it: it is ten lines.

## Turning parts off

```sh
node ~/.agentmingo/reporter.mjs titles off        # no titles; sessions show their repository
node ~/.agentmingo/reporter.mjs share-files off   # no edited file paths, no notes about others
node ~/.agentmingo/reporter.mjs reviews off       # no review results back, no waking for them
node ~/.agentmingo/reporter.mjs auto-update off   # stay at this version
```

## Uninstall

```sh
node ~/.agentmingo/reporter.mjs uninstall
```

> `uninstall` takes the hooks, the rules and the MCP server out and disconnects this computer; it leaves ~/.agentmingo and the copy of your settings for you to delete.

Uninstall deletes `config.json`, so the connection token is gone from your computer. It doesn't touch `~/.claude/CLAUDE.md`. If you added the lean git rules, take them out with `node ~/.agentmingo/reporter.mjs lean-git off` before you delete `~/.agentmingo`. Their backup, `~/.claude/CLAUDE.md.agentmingo-backup`, stays for you to delete too.

To remove what it leaves, delete the folder, and the settings copy once you no longer need it:

```sh
rm -rf ~/.agentmingo
rm ~/.claude/settings.json.agentmingo-backup
```

You can also disconnect a computer from the Sessions page in AgentMingo at any time. Its reports are refused from then on.

## Reporting a security issue

Email **hello@agentmingo.com** with what you found and how to reproduce it. We'll confirm we got it and keep you posted until it's fixed. The same address is in <https://agentmingo.com/.well-known/security.txt>, and our policy is at <https://agentmingo.com/security>.

Please don't open a public issue for a vulnerability. See [SECURITY.md](SECURITY.md).

## About this repository

The script is developed in AgentMingo's own repository, which is closed source, like the server. Each released version is published here by hand for now, after agentmingo.com starts serving it, with its sha256 in [CHANGELOG.md](CHANGELOG.md). This repository's history starts at version 12.

A pull request here can't be merged as is, since the next release would overwrite it. Open an issue instead, and we'll make the change there.

More help: <https://agentmingo.com/troubleshooting>.

Product names here (Claude Code, Cursor, Codex, CodeRabbit, GitHub) are trademarks of their owners. AgentMingo isn't affiliated with any of them.
