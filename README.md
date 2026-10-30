# AgentMingo reporter

This is the script that connects the coding-agent threads on your computer to [AgentMingo](https://agentmingo.com).
It is the same file agentmingo.com serves at <https://agentmingo.com/reporter.mjs>, published here so you can read exactly what it does before you run it.

AgentMingo is a free open beta. The AgentMingo server is closed source. This script is open source, under the Apache-2.0 license.

## What it is

`reporter.mjs` is one file of plain JavaScript with no dependencies.
Your coding agent runs it from its hooks at a few moments in each thread.
It reports those moments to AgentMingo, so your team sees every thread in one place and knows which ones need a person.
It also hands your threads what they can't see on their own: a ticket's brief, review and CI results for their branch, teammates' live threads, your team's rules, and workflow steps one at a time.
It can start a thread on a task in a folder of its own, and, only if you turn it on on that computer, let you answer a thread's permission prompts, and reply to a thread that waits for you, from AgentMingo.

AgentMingo isn't tied to one agent or model vendor. Today this script connects one agent, Claude Code, through Claude Code's hooks. Other agents (Cursor's own agent, Codex and the rest) aren't connected yet. Review results work the same way whichever review bot comments on the pull request: CodeRabbit is one example.

## Requirements

- macOS or Linux. Native Windows isn't supported yet.
- Node.js 18 or newer (`node --version`).
- Claude Code (`claude --version`), the agent this script connects today.

Cursor's own agent (its chat and Agent panel) is ignored. Claude Code inside Cursor works. See [Cursor](#cursor).

## Install

You get a setup command from the Sessions page in AgentMingo. It looks like this (`acme` and the code are examples; the boxes you tick add flags such as `--share-files` or `--no-share-files`, `--reviews`, and `--approvals`):

```sh
curl -fsSL https://agentmingo.com/reporter.mjs -o ~/.agentmingo/reporter.mjs --create-dirs \
  && node ~/.agentmingo/reporter.mjs install --url https://agentmingo.com --org acme --code amc_… --approvals
```

The code works once, for 15 minutes. Install trades it for this computer's token, so the token never appears in a command line or a shell history.

To read the file before it runs, split the command in two:

1. Run the `curl` part only.
2. Read `~/.agentmingo/reporter.mjs`, and [check its sha256](#check-that-your-copy-is-a-published-one).
3. Run the `node … install …` part.

Running install again is safe. It replaces its own hooks instead of adding them twice, and it keeps your earlier choices (titles, sharing, review results, updating, answering prompts and replies from AgentMingo) unless a flag changes them.

### The box "Answer prompts and reply to threads from AgentMingo"

This box on the Sessions page is ticked by default. Ticked, the setup command carries `--approvals`, and install turns on [answering permission prompts and replying from AgentMingo](#answering-prompts-and-replying-from-agentmingo) on this computer. Unticked, the command carries `--no-approvals` instead, which turns answering prompts and replies off on that computer, or keeps them off.

Install turns it on only when all three hold, and checks them before anything changes or the one-time code is used:

- the command carries the one-time code (`--code`). The older `--token` form never turns it on;
- it runs in a real terminal (both standard input and output are a terminal);
- it doesn't run inside Claude Code's Bash tool (`CLAUDE_CODE_CHILD_SESSION`, or `CLAUDECODE` without a terminal).

Otherwise install refuses, says why, and changes nothing: the whole install stops, not only this part, so a setup command with the box ticked doesn't run from a script without a terminal or from your coding agent. Paste it into a terminal of your own, or untick the box (the command then turns it off, as above). When answering prompts is on already, `--approvals` changes nothing and refuses nothing, so running the command again from a provisioning script still works.

When it turns it on, install prints what it turned on, what is sent, and how to turn it off. The choice is the same as `approvals on` with no options (file edits, an hour's wait, commands not sent, replies), without typing yes: running the command is the yes. To change the wait or show commands, run `approvals on` with its options afterwards. `--no-approvals` turns it off.

Those checks catch a command pasted by mistake or run by your coding agent's Bash tool. A program on your computer set on faking a terminal could get past them, as it could change your settings anyway. The setup command comes from AgentMingo, and it already downloads and runs the script, so the flag adds no trust of its own: what counts is that you run it. AgentMingo's website and servers can't turn it on on a computer by themselves, without you running a command there (an update of the script can change anything it does, as it always could; see [Updates](#updates-and-the-kill-switch)).

## What install changes on your computer

**Hooks** in `~/.claude/settings.json`, with a copy of the file from before kept beside it as `settings.json.agentmingo-backup`. One hook on each of these events: `SessionStart`, `UserPromptSubmit`, `PostToolUse`, `PermissionRequest`, `Notification`, `Stop`, `StopFailure` and `SessionEnd`. `PostToolUse` runs in the background, so the agent never waits for it. A second `Stop` hook (`watch`) also runs in the background and can wake an idle thread when review results, its next workflow step, or your reply from AgentMingo arrive (see [Review results](#review-results) and [The watcher](#the-watcher-and-its-timeout)). On macOS and Linux, a hook looks like this, with your own home folder (`SessionEnd`'s timeout is 5; `PostToolUse` and `PermissionRequest` also have `"matcher": "*"`, and `PostToolUse` has `"async": true`):

```json
{
  "type": "command",
  "command": "\"/Users/you/.agentmingo/reporter.sh\" hook",
  "timeout": 10
}
```

The `watch` hook has `"async": true`, `"asyncRewake": true` and a timeout of 29400 seconds (8 hours and 10 minutes). Versions before 15 gave it 1200 seconds (20 minutes). An update to 15 from an earlier version raises that number itself, once, and changes nothing else for it, as [The watcher and its timeout](#the-watcher-and-its-timeout) says.

**The `PermissionRequest` hook, only with answering prompts from AgentMingo on.** When it is on (install's `--approvals` turned it on, or `approvals on` did), this hook's command gets `--wait` and the number of seconds a prompt waits for an answer from AgentMingo, and its timeout becomes a minute longer than that, so the hook always ends on its own before Claude Code stops it. With the default wait of an hour:

```json
{
  "type": "command",
  "command": "\"/Users/you/.agentmingo/reporter.sh\" hook --wait 3600",
  "timeout": 3660
}
```

`approvals on` and `approvals off` change only this hook in the file, and `approvals off` puts it back as above. See [Answering prompts and replying from AgentMingo](#answering-prompts-and-replying-from-agentmingo). Replies change nothing more in the file: they use the `watch` hook above.

**Five permission rules**, added to `permissions.allow` in the same file:

```text
Bash("/Users/you/.agentmingo/reporter.sh" step:*)
Bash("/Users/you/.agentmingo/reporter.sh" start:*)
mcp__agentmingo__current_step
mcp__agentmingo__complete_step
mcp__agentmingo__start_task
```

They let the agent report a workflow step, and make a task's folder, without asking you. They allow only those two subcommands of this script, and the three tools of its own MCP server. Making a task's folder doesn't move the thread there: the `start_task` tool's answer tells the agent to ask you first (see [Starting a task](#starting-a-task)).

**An MCP server** named `agentmingo`, registered for your user with `claude mcp add -s user` (in `~/.claude.json`). It runs this script with `mcp`, and has three tools:

- `current_step`: the thread's current workflow step. It sends the folder's repository and branch.
- `complete_step`: reports a step done or failed, with what the agent wrote and the files it attaches.
- `start_task`: makes a task's folder when you ask a thread to work on a task ("Work on WEB-12"). It sends the task's key, and a repository when you name one.

Install's `--no-mcp` skips it. When the `claude` command isn't found or doesn't answer, install prints the `claude mcp add` line to run yourself instead.

**The folder `~/.agentmingo`.** See [What it keeps on your computer](#what-it-keeps-on-your-computer).

**Only with `--lean-git`** (or `lean-git on`): six short git habits added to `~/.claude/CLAUDE.md`, between marker comments, with a copy of the file as it was kept beside it as `CLAUDE.md.agentmingo-backup`, when there was one. `lean-git off` takes them out.

Nothing else on your computer changes.

## Which threads report

In the script's own words:

> Which threads report: only those in a repository your organization has connected in AgentMingo (also under an earlier name, when it was renamed or moved: GitHub keeps old names working, so an older clone may still use one), or one you added (`repos add` in its folder; the Sessions page lists them and removes them), named by the folder's origin remote. A thread anywhere else (another client's repository, a side project, a folder that isn't a git repository) sends nothing about itself: no events, no lookups, and AgentMingo's notes don't reach it. One thing is asked about such a folder: when its repository shares its owner or its name with one an organization lists (acme/web-old next to acme/web, say), this computer sends that name to the organization once a day, to ask whether it's the same repository under an earlier name. AgentMingo asks GitHub, and keeps the name only if it is. Until this computer has heard an organization's list (just installed or updated), it asks for it before sending anything about a thread, and drops what it queued for threads outside it.

`node ~/.agentmingo/reporter.mjs repos` lists the repositories that report from your computer. A permission prompt is opened on AgentMingo only for a thread that reports, and only such a thread's watcher listens for replies.

## What it sends

This is the list in the script's header, quoted verbatim. AgentMingo's [Privacy Policy](https://agentmingo.com/privacy#collect) lists the same things in its own words, and says how long AgentMingo keeps them.

> What it sends for those, every request but setting up and downloading an update (below) carrying this computer's connection token (`authorization: Bearer amk_…`) and the script's name and version (user-agent `agentmingo-reporter/` and a number), and like any request, your IP address:
>
> - What happens in each thread, and when: it started, you sent a message, it's working (at most every five minutes, and at once after a tool call that ran long), with how long the tool call behind it ran, it waits for you, it was answered, it replied, it ended. Each with its time, a random id of its own (so it's recorded once), and Claude Code's id for the thread.
> - Where it works: its repository name and branch, whether the branch is pushed and how many of its commits aren't, how many files have changes not yet committed (a count, never their names or what changed), and the app it runs in.
> - Counts from the session's transcript, from when the reporter first saw the session: token counts by model (with the model's name), tool calls counted by kind (git, GitHub's command line, tests, builds, installs, other shell commands, reading files, edits, web, subagents, MCP tools, and any other tool) with how much output they produced and about how long they took, and git commands counted by subcommand (diff, log, status, push and so on) with how much output each produced. When a thread's transcript stops giving those counts (a Claude Code update can change how it writes transcripts), it says which of them is missing, so `status` and AgentMingo's admins can see it.
> - A short title (Claude Code's own session title, or the first line of your first message, with emails, phone numbers, long numbers, keys and tokens by their known shapes, private keys, credentials in addresses, headers and command lines, and secret-looking values given to a password, secret, token or key masked on this computer; turn titles off with --no-titles or `titles off`), and what a waiting session's notice asks you (masked the same way; with titles off, only whether it asks for a permission, and for which tool, or has a question).
> - Its settings: whether review results are on here, whether sessions know about each other here, whether the lean git rules are in a CLAUDE.md it reads or were given to it in a note, and while its organization measures them, which group of the test the session is in (with the rules, or without).
> - What it passed on: which review results, notices and workflow step it handed the session, so none is handed twice.
> - What it asks for: when a session starts on a branch whose name looks like a task's key, or comes onto one (it switched branches, or moved into a task's folder), that branch and the repository, for the task's brief. Before each message, the session's id, for its workflow step (when it follows one, or its branch names a task or it has one) and, with review results on, for results and notices waiting for it. After each reply, the watcher below.
>
> With --share-files (the setup command's "Let sessions know about each other", ticked by default) it also sends the paths of files a session edits, each with when it was edited, says sharing is on, and tells each session (in one short note) which other sessions are working on which files in the same repository (their person's first name, title and branch), so they stay out of each other's way. With it off (--no-share-files, or `share-files off`), it sends no paths, your sessions get no such note, and AgentMingo names your sessions in no teammate's notes: not in their note about other sessions, and not when someone else's thread works on the same task.
>
> Answering permission prompts and replies from AgentMingo is off unless you turn it on here, in your own terminal: by running the setup command with its box "Answer prompts and reply to threads from AgentMingo" ticked (it is by default: install's --approvals, which turns it on only with the command's one-time code from your Sessions page, in a real terminal, not inside Claude Code's Bash tool, and says so; it refuses otherwise), or with `approvals on`, which asks you to type yes. Those checks catch a command run by mistake or by your coding agent's Bash tool; a program on your computer set on faking a terminal could get past them. AgentMingo's website and servers can't turn it on by themselves: the setup command carries the flag, and you run it (it already downloads and runs this script); an update of this reporter can change anything it does, as it always could (turn updates off with `auto-update off`). Once it's on, when a session started since stops on a permission prompt, Claude Code's prompt stays on screen as always, and the hook also opens the prompt on the thread's page in AgentMingo and waits up to the time you chose (an hour unless you say otherwise) for your answer there; whichever is answered first counts. For that prompt it sends the tool's name, a random id, an HMAC that ties an answer to this prompt (made with a key that never leaves this computer), a hash of what it shows, how long it will wait, and for a file change, the file's path in the repository (masked like a title) with how many lines it adds and removes, never the change itself; with --commands, a command, masked. While it waits it asks AgentMingo for an answer every 3 seconds for two minutes, then every 10, and says when it stops waiting. Allow applies to that one call (never an "always allow" rule), and only for a file inside the repository that can't run code or change settings by itself; a command or another tool can only be denied there. It waits only in a thread's main conversation (not a subagent's), not in a headless run, not on a Claude Code version known to misbehave (none so far), and in a terminal only once Claude Code says the prompt is on screen (within 20 seconds; else it leaves the prompt to the terminal at once): that notice is the guard. Claude Code doesn't document that a hook can wait while its prompt stays usable, so a release of it could change that; in its VS Code extension and desktop app, which send no such notice, Claude Code shows its prompt and runs the hook at the same time. When Claude Code stops the hook, it tells AgentMingo it stopped waiting. Someone who controls AgentMingo's servers or database could answer for you. `approvals off` turns it off.
>
> With it on, a thread that finishes its turn and waits for you can also be answered from its page in AgentMingo (Continue, Yes, No, or up to 1,000 characters of your own): the watcher below listens up to 8 hours after each reply, and when you send one, it takes it (once: it's claimed on this computer and on AgentMingo) and wakes the thread with it, introduced as your reply, with what your next message would have carried (the next workflow step, review results). For that, each report also says replies are on here, and the watcher says when it asks next. Only you can send one, signed in as yourself in that browser within the last day; someone who gets into your AgentMingo account could send your threads a reply, and a command a thread then wants to run still needs your terminal (AgentMingo can only deny commands). A reply is for the turn it answers: typing in the thread first, the thread ending, or the thread going on without it (woken for something else) cancels it.
>
> On a workflow (below), it also sends whether a step Claude reports is done or failed, and what Claude writes when it reports a step (for the plan step, the plan itself, with the repositories it names and whether it needs an end-to-end test, and why; AgentMingo masks the same details when it arrives), the files Claude attaches to a step (reports, screenshots, videos, logs; up to 4 MB each, 20 at most), each with its name or its path in the session's folder, and when Claude asks for its current step, the folder's repository and branch. The commands send what they name: `start` and the start_task tool the task's key (and the repository, with --repo), `repos add` and `repos remove` a repository's name, and `repos` asks for the list. On a computer connected to several organizations, a step report and `start` go first to the organization of the folder's repository, and to the next only if that one doesn't know the step or the task. Install sends the setup command's one-time code, the organization's short name and the script's version, and nothing else; when it replaces an earlier connection, and on uninstall, it asks AgentMingo to disconnect that one. Updating downloads the new script without its token and without anything about you or your threads. It doesn't send your code or other files (unless Claude attaches one to a step), the commands Claude runs (only their kind, and git's subcommand, are counted), tool input, or the rest of the conversation, except, with answering prompts from AgentMingo turned on here (above), a waiting prompt's file path and line counts and, with commands shown, its command, masked.

The header is written for the one agent the script connects today, so where it says "Claude", it means that agent. The exact fields of each report are built mainly in two functions you can read: `hookToEvent()` (the moment itself) and `enrich()` (what is added to it before it is sent). `runHook()` adds which notes it passed on. A waiting permission prompt's request is built in `openRequest()`: besides what the header lists, it carries the thread's id, the kind of prompt (a file edit, a file write, a new file, a notebook, a command or another tool), and whether Allow can be offered for it and, when not, why. Closing it carries how the wait ended (answered on this computer, time up or Claude Code gone, the prompt never shown, an answer refused here or too old, or the hook stopped). For replies, `enrich()` adds `replies` (true or false) to each report, and the watcher's request, built in `askFeedback()`, carries the thread's id and, with replies on, `listen` (in how many seconds it asks again) or `deliverReply` (the id of the reply it is taking).

### What it doesn't send

Your code and other files (unless the agent attaches one to a workflow step), the commands the agent runs (only their kind, and git's subcommand, are counted), tool input, and the rest of the conversation. The script reads the thread's transcript on your computer to count tokens and tool calls, take the title and see which files it edits. AgentMingo's servers don't receive the conversation.

The one exception is answering prompts from AgentMingo, when you turn it on: a waiting prompt's file path in the repository and how many lines it adds and removes, and, only with `--commands`, the command, masked. The file change itself is never sent. The HMAC key that ties an answer to a prompt stays in `~/.agentmingo/config.json`. Replies go the other way: the text you write on a thread's page comes to this computer, and nothing of the conversation goes to AgentMingo for it.

Masking (in titles, a waiting thread's notice, a prompt's file path and a command) happens on your computer and is best effort: a password in a shape no rule knows can get through. The rules are the `MASK_RULES` constant, applied by `maskPersonal()`.

### Notes AgentMingo adds to your threads

AgentMingo sends notes back, and the script adds them to the thread: your team's rules (with the name of the admin who wrote them), the git habits when your organization has them on, a task's brief, other live threads in the same repository (with sharing on), review results and workflow steps. The `start_task` tool's answer also reaches the thread: the task's title (quoted, and said to be a name, not an instruction), the folder, and how to move there. When you deny a permission prompt from AgentMingo, your note reaches the agent as your quoted words (masked, on one line, 300 characters at most). A reply you send a thread from its page reaches it as "<your first name> replied from AgentMingo: …", followed by a line saying it is your reply, sent from the thread's page instead of typed there, and that the agent should go on as if you had typed it (`replyNote()`). The reply's text is not masked: it is your own words, as you wrote them, with characters that don't show written out. These become part of the thread's conversation, which your agent sends to its model provider under your own account and plan, like the rest of the conversation. Nothing on your computer changes for that.

## When it sends

Everything goes to the AgentMingo address in your setup command (`--url`, such as https://agentmingo.com). Events are sent by a separate background process, and wait in `~/.agentmingo/outbox` until they are delivered.

| When | What |
| --- | --- |
| A thread starts, you send a message, it asks for permission or has a question, it finishes replying, it ends | One event for that moment |
| While a thread works | A heartbeat at most every 5 minutes (at once after a tool call that ran a minute or more) |
| A thread moves into another repository or worktree mid-session (Claude Code's EnterWorktree, say) | A heartbeat at once, from the new folder. Moving between folders of the same repository isn't reported early |
| A permission prompt or question is still open after a minute | The same event again, so its person can be told |
| A thread starts on a branch that looks like a task key (`WEB-12-login`) | A lookup of that task's brief (the branch and repository), waiting a second at most |
| You send a message | With review results on: a lookup for the review results and notices waiting for the thread. In a thread that follows a workflow, has a task, or is on a branch that names one (also when it starts): a lookup of its current step. When the thread is on a branch that looks like a task key and differs from the one it last asked about (it switched branches, or moved into a task's folder): a lookup of that task's brief. Each waits a second at most, and they run at the same time. Review results that became ready after the watcher stopped arrive with this message |
| After each reply, with review results on (replies off, no workflow) | The `watch` hook asks every 30 seconds for 3 minutes, then every minute until 10 minutes, then every 2 minutes, for up to 18 minutes (about 17 requests). It stops early when the branch has no pull request and the thread follows no workflow, or when the pull request's automatic review rounds are used up |
| After each reply, in a thread that follows a workflow, and with replies on in any thread that reports | The `watch` hook listens up to 8 hours instead: the same pace for the first half hour, then every 5 minutes (about 110 requests in 8 hours). With replies on, each request says in how many seconds it asks next, so the thread's page offers a reply only while it listens. See [The watcher and its timeout](#the-watcher-and-its-timeout) |
| Either way | The watcher stops within 5 seconds of you typing in the thread or the thread ending (it looks at the thread's state on this computer, without a request), and when the thread's next reply starts a watcher of its own. It never runs past a minute before its hook's timeout |
| With replies on, a reply is waiting | The watcher claims it on this computer, asks AgentMingo to hand it over (once), and wakes the thread with it, with the workflow step and (with review results on) the review results that were waiting, each marked as told |
| Only with answering prompts on: a thread stops on a permission prompt | The prompt's request (see [What it sends](#what-it-sends)), then a check for an answer every 3 seconds for two minutes, then every 10 seconds, until it is answered on your computer or on AgentMingo, its time is up, Claude Code stops the hook or is gone, or (in a terminal) Claude Code hasn't said within 20 seconds that the prompt is on screen. When it stops waiting without applying an answer from there, it tells AgentMingo so (best effort when Claude Code stops the hook; not when AgentMingo refused the check, no longer knows the request, or closed it already); after it applies one, an `answered` event |
| At most every 15 minutes | An empty report that asks for your organization's list of repositories. It carries nothing about any thread |
| Once a day per name | The name check described in [Which threads report](#which-threads-report) |
| When an answer says a different version should run | A download of `reporter.mjs` from the same AgentMingo, without the token (at most every 6 hours for the same version, or every 10 minutes when going back; none when going back to the version kept in `reporter.prev.mjs`; see [Updates](#updates-and-the-kill-switch)) |
| When you or the agent run a command | `install` (trades the code), `repos`, `repos add` or `repos remove` (the repository's name), `start` or the `start_task` tool (the task key, and the repository when you name one), `step` or the step MCP tools (the step's text and attached files; `current_step` sends the folder's repository and branch), `uninstall` (disconnects this computer). `approvals on`, `off` and `status` send nothing, and neither does install's `--approvals` beyond what install sends anyway |

When AgentMingo doesn't answer, the script holds back:

- A lookup (a task's brief, or what's waiting before a message) waits a second at most. When one gets no answer, lookups pause for 60 seconds, so a slow AgentMingo costs one message a second, not every message. What the lookup would have brought comes with a later answer instead.
- Sending stops for 30 seconds, doubling up to 10 minutes (or as long as a 429 asks), then what waited is sent.
- While lookups are paused, or sending is paused because AgentMingo didn't answer (not for a 429 or a reached limit), or the connection is refused (below), a permission prompt opens no request on AgentMingo: it stays in the terminal only.
- When AgentMingo says a limit on reports was reached, the events it skipped are dropped, and nothing more is sent until the limit's window ends.
- When it refuses the connection (403: the account or organization is suspended, or you were made a viewer), nothing is asked or sent there for an hour, doubling to a day, or until install runs again.

## What it keeps on your computer

Everything is in `~/.agentmingo`:

- `reporter.mjs`: this script. `reporter.prev.mjs`: the version before the last update, kept to go back to.
- `launch.mjs`: the launcher the hooks and the MCP server run. It runs `reporter.mjs`, or the kept version when the new one doesn't load. When the new one loads but fails, it puts the kept version back as `reporter.mjs` for the next run. Its full text is the `LAUNCHER` constant in `reporter.mjs`, and it doesn't change when the script updates.
- `reporter.sh` (macOS and Linux): a short shell script that runs `launch.mjs` with the Node.js install ran with, or the `node` on your PATH once that one is gone (an nvm or Homebrew upgrade). With neither, it does nothing. Its text comes from `shimText()` in `reporter.mjs`, with your Node.js path and home folder filled in.
- `config.json`: your choices, this computer's connection token and each organization's list of repositories, readable by you only (mode 600). With answering prompts and replies on, it also holds that choice (commands shown or not, the wait, and a `--unchecked` flag from version 14 that no longer changes anything) and the key the HMAC is made with, which is made on your computer by `approvals on` or by install's `--approvals` and never sent; `approvals off` takes both out, and replies stop with them, since they go by the key. From version 15 it also notes the watch hook's timeout: that this version checked it (`watchTimeoutChecked`), and, when it raised it, when (`watchTimeoutRaisedAt`) and what it was before (`watchTimeoutWas`). When AgentMingo removes this computer's only connection, it is renamed `config.json.removed`.
- `outbox/`: events waiting to be sent.
- `flush.lock`: there while the background sender runs, so only one sends at a time.
- `state/`: each thread's state, what's waiting for it, when each organization's list of repositories was last heard, the names asked about once a day, team rules, other live threads in the same repository (with sharing on), when AgentMingo last didn't answer (`backoff.json`, and `lookups.json` for the 60-second lookup pause), the version AgentMingo last said to run and the last update tried, and lock files.
- `state/<session id>.asked.json`: the branch a thread last asked about for its task's brief, so a message asks again only when that changes. `state/<session id>.prompt.json`: when Claude Code last said a permission prompt was on screen in that thread, which a waiting hook needs in a terminal. Both are removed when the thread ends.
- `state/<session id>.hook.json`: the thread's state. From version 15 it also counts the thread's ends (with the messages you typed, which it counted already), so a watcher knows you typed or the thread ended, and notes the watch hook's timeout when the thread started.
- `state/<session id>.watch.lock`: names the thread's current watcher (its process, a random id and when it started). A newer reply's watcher writes its own, and the older one sees that and stops.
- `state/<session id>.told/`: one empty file per review result or notice a thread was told, so the message hook and the `watch` hook never tell it the same thing twice. With replies on, also one per reply a watcher took (`reply-…`), so a reply is handed over once even when two watchers see it. A thread's folder is removed when it ends; one left by a thread that never ended is removed, when a later thread starts, once nothing was added to it for 7 days.
- `repos.json`: where each repository is cloned, so `start` and `start_task` work from any folder.
- `reporter.log`: a log of problems and updates. `node ~/.agentmingo/reporter.mjs status` shows its last lines.

A downloaded update is written to `reporter.next.mjs` in `~/.agentmingo`, then checked in a scratch folder in your system's temporary folder, which is removed afterwards. If the check fails, `reporter.next.mjs` is deleted; if it passes, it becomes `reporter.mjs`.

## Starting a task

A task's work goes in a folder of its own, a git worktree next to your clone (`<clone>-WEB-12`), on the task's branch. There are two ways to make one.

**In a thread**, say "Work on WEB-12". The agent calls the `start_task` tool, which:

1. asks AgentMingo for the task (its key, and a repository when you name one), which answers with the task's branch and the repository with its main branch;
2. finds your clone (the folder the thread is in, or the clone `repos.json` remembers for that repository), runs `git fetch origin` and `git worktree add` there, or uses the folder made for the task before. Git runs with nothing on screen and can't ask for anything (no password prompt, ssh in batch mode), and the fetch and the worktree add each give up after a minute;
3. copies your clone's local `.env` files into the folder (see below);
4. answers with the folder and what to do next. It doesn't move the thread. Its answer tells the agent to ask you first ("Switch this thread to WEB-12's folder?"), and only after you say yes to move the thread there with Claude Code's EnterWorktree tool. If you'd rather keep the thread where it is, or EnterWorktree isn't available, it gives you a link, `claude-cli://open?cwd=…&q=Work%20on%20WEB-12`, that opens a new terminal Claude Code session in that folder with the request filled in (you press Enter), and the same as a terminal command: `cd <folder> && claude "Work on WEB-12"`.

Asking you first is an instruction in the tool's answer and in the MCP server's instructions; the script can't enforce it. After the thread moves, its next hook comes from the new folder: the reporter reports the move at once, and the task's brief comes with your next message.

**In your own terminal**, from any folder:

```sh
node ~/.agentmingo/reporter.mjs start WEB-12
```

It makes the same folder, prints what it did, and starts Claude Code there on the task. `--repo <name>` picks the repository for a task that spans several, `--in <folder>` names your clone, and `--no-launch` only makes the folder. Run from inside a thread (the agent's own shell command), it starts no second Claude Code: it prints the folder and the same guidance as `start_task`. `start` works only in a connected or added repository.

**`.env` files.** Only these are copied, when git ignores them in your clone: `.env`, `.env.local`, `.env.development`, `.env.development.local`, `.env.test` and `.env.test.local`. Any other (`.env.production`, say) can hold production secrets that any thread working in that folder could read, so it is left out, and `start` says which ones it left out. A copied `.env.local` can still hold tokens of your own (`vercel env pull` writes one there), so `start` says so when it copies one. `--no-env` copies none. `--env-all` copies every one, and works only in your own terminal: it is refused from inside a thread. The list is the `SAFE_ENV_FILES` constant.

## Answering prompts and replying from AgentMingo

One switch, on each computer, turns on two things: answering a thread's permission prompts from its page in AgentMingo, and replying from there to a thread that finished its turn and waits for you. It is off unless you turn it on, in a terminal of your own, in one of two ways:

- **The setup command**, with its box "Answer prompts and reply to threads from AgentMingo" ticked, as it is by default. The command then carries `--approvals`; see [The box](#the-box-answer-prompts-and-reply-to-threads-from-agentmingo) for when install turns it on and when it refuses.
- **`approvals on`**:

```sh
node ~/.agentmingo/reporter.mjs approvals on                 # file edits, an hour's wait
node ~/.agentmingo/reporter.mjs approvals on --commands      # commands shown too, to deny them
node ~/.agentmingo/reporter.mjs approvals on --wait 2h       # 5 minutes to 8 hours: 30m, 2h, 1h30m
node ~/.agentmingo/reporter.mjs approvals status
node ~/.agentmingo/reporter.mjs approvals off
```

`approvals on` prints what it turns on (replies included), its limits and who could answer, and turns it on only when you type `yes`. It refuses inside Claude Code's Bash tool (it checks Claude Code's environment variables, `CLAUDE_CODE_CHILD_SESSION`, or `CLAUDECODE` without a terminal) and when it has no terminal to ask you in. `--unchecked`, from version 14, is still accepted and changes nothing. These checks guard against a mistake or your agent turning it on; they aren't a wall against other programs that run as you, which can edit `~/.agentmingo/config.json` and `~/.claude/settings.json` like any of your files. `approvals off` and `approvals status` work from anywhere. `status` also says whether it's on. The command itself sends nothing to AgentMingo.

### Permission prompts

Once it's on, in threads started from then on (open threads keep the hook they started with), when a thread stops on a permission prompt:

- Claude Code's prompt stays on screen as always. The hook also opens the prompt on the thread's page in AgentMingo and checks for your answer there. Whichever is answered first counts. When you answer in the terminal first, the hook sees the tool call's result in the thread's transcript and stops waiting.
- **A file change** (Edit, MultiEdit, Write, NotebookEdit) shows its path in the repository and how many lines it adds and removes, never the change itself. It can be allowed or denied from AgentMingo.
- **A command** shows, masked, only with `--commands`, and can only be denied from AgentMingo. **Any other tool** shows its name only, and can only be denied.
- **Allow** applies to that one call, never an "always allow" rule, and never with changed input. It is never offered for a file outside the repository, one that can run code or change settings by itself (package.json, CI workflows, .env files, git, editor and agent settings, task runners, test and build set-up, the folder git runs hooks from: `PROTECTED_FOLDERS`, `PROTECTED_FILES` and `isProtectedPath()` have the full list), a path with `.` or `..` in it, a file with another name elsewhere (a hard link), a path with characters that don't show as written, or a path that masking changed. Allowing a file change can still lead to running code later: tests and builds run what's in it.
- **Deny** always works. With a note, the note reaches the agent as your quoted words; you can also ask it to stop the turn.
- The terminal says what happened: "Allowed once from AgentMingo by you at 14:05", or "Denied from AgentMingo by you at 14:05".

Before it applies an answer, the script checks it on your computer (`applyAnswer()`): it must carry the HMAC made for that prompt with the key in your `config.json`, be at most two minutes old when it arrives, and, for Allow, the page must have shown what this computer sent, and the file must still be one Allow is offered for, and the prompt not answered in the terminal meanwhile. Otherwise nothing is applied and the terminal prompt stays.

Its limits, plainly:

- **The guard is the on-screen check, not a list of tested versions.** Claude Code doesn't document that a hook can wait while its prompt stays usable, so a release of it could change that. In a terminal, the hook waits only once Claude Code says the prompt is on screen; when it doesn't say so within 20 seconds, the hook stops waiting at once and leaves the prompt to the terminal alone. Version 14, the first with answering prompts, waited only on Claude Code versions checked with it, and listed none (so prompts were only reported unless you chose `--unchecked`); from 15 the list is of versions known to misbehave (`KNOWN_BAD_TERMINAL_VERSIONS` and `KNOWN_BAD_APP_VERSIONS`), on which a prompt is only reported. It is empty: none is known so far, and a release adds one when one is found. If a terminal prompt ever stops taking your keys while it waits, run `approvals off` and tell us.
- In Claude Code's VS Code extension and desktop app, which send no on-screen notice, Claude Code shows its prompt and runs the hook at the same time, and takes whichever answers first. The on-screen check doesn't apply there.
- It doesn't wait for a subagent's prompt, a headless run (`claude -p`), a thread whose repository doesn't report, a question the agent asks you, or a plan to approve. It doesn't wait while AgentMingo isn't answering.
- An answer arrives only while your computer is awake and online and Claude Code is running. The hook stops waiting when its time is up or Claude Code is gone.
- Masking of a path or command is best effort.
- Between the script checking the file once more and Claude Code writing it, another program already running on your computer could swap the file for a link elsewhere. Allow trusts what already runs on your computer.
- Someone who gets into your AgentMingo account could allow a file change on your computer, and so could an agent with a browser signed in to AgentMingo. Someone who controls AgentMingo's servers or database could answer for you, or show you a different file than the one that changes. And an update of this script can change anything it does, as it always could (`auto-update off` stops updates).
- Who can answer on AgentMingo's side (only you, signed in as yourself) is enforced by the server, which this script can't show you. [AgentMingo's Security page](https://agentmingo.com/security#approvals) describes it.

### Replies

Replies go with answering prompts: a computer where you turned answering prompts on with version 14 takes replies too once it updates to 15, without asking again. `approvals off` turns both off.

With it on, when a thread finishes its turn and waits for you, its page in AgentMingo can send it a reply: Continue, Yes, No, or a text of your own (up to 1,000 characters). After each reply the thread makes, its `watch` hook listens up to 8 hours for one (see [The watcher and its timeout](#the-watcher-and-its-timeout)). When you send one:

1. The watcher sees it the next time it asks, at its pace: every 30 seconds at first, every 5 minutes after the first half hour.
2. It claims the reply on this computer (an empty file in `state/<session id>.told/`), then asks AgentMingo to hand it over. AgentMingo hands each reply over once, so two watchers can't both take it.
3. It wakes the thread with "<your first name> replied from AgentMingo: …" and a line saying this is your reply, sent from the thread's page instead of typed there, and that the agent should go on as if you had typed it. The workflow step and the review results that were waiting for your next message come with it, as they would have with a typed message.

A reply belongs to the turn it answers. Typing in the thread first, the thread ending, or the thread going on without it (woken for review results or a workflow step) cancels it: it isn't delivered later. The watcher stops within 5 seconds of you typing in the thread or the thread ending, and the thread's next reply starts a watcher of its own, which takes over.

What it sends for replies: each report says whether replies are on here, each watcher request says in how many seconds it asks next, so the page offers a reply only while something listens, and the request that takes a reply carries that reply's id. The reply's text comes from AgentMingo to this computer. Turning replies on adds nothing else to what is sent.

Its limits, plainly:

- **It reaches the thread only while the watcher listens.** That is up to 8 hours after the thread's last reply, while your computer is awake and online and the thread is open, and never past its hook's timeout. A computer still on the 20-minute timeout from an earlier install listens 19 minutes at most, until the update raises it; a thread opened before that keeps its 20 minutes until you restart it. When nothing listens, the page says so, and you type in the thread instead.
- **A reply directs your agent as your own words.** The agent acts on it as if you had typed it. Someone who gets into your AgentMingo account could send your threads a reply, and so could an agent with a browser signed in to AgentMingo as you. A command the thread then wants to run still stops at a permission prompt, and AgentMingo can only deny a command, so allowing it still needs your terminal, unless your Claude Code settings already allow that command, in which case it runs as it would after any message.
- **The reply's text isn't masked** on its way into the thread. It becomes part of the conversation, which your agent sends to its model provider under your own account and plan.
- Who can send a reply (only you, the thread's own person, from its page and not with an API token, with a sign-in in that browser within the last day), the 1,000-character limit, one reply waiting per thread, its expiry (2 hours), that it is taken once, rate limits, and the audit log entry are enforced by AgentMingo's server, which this script can't show you. [AgentMingo's Security page](https://agentmingo.com/security#approvals) describes them. On this computer, the script cuts a reply at 4,000 characters and writes out characters that don't show.
- Someone who controls AgentMingo's servers or database could send your threads a reply.
- A watcher already listening when you run `approvals off` read the setting when it started and doesn't look again: until it stops (you type in the thread, it ends, or its time is up), a reply sent from the page may still reach that thread. Watchers that start afterwards don't take replies.

### Turning it off

`approvals off` takes the key and the choice out of `config.json` and the `--wait` out of the hook, and so turns replies off too. Threads open at that moment keep the hook they started with until they end, but their next prompts open no request, since the key is gone. A prompt already waiting at that moment keeps waiting until it's answered or its time is up.

The code to read: `approvalsCommand()`, `approvalsFromSetup()`, `installConfig()`, `permissionTarget()`, `fileTarget()`, `isProtectedPath()`, `promptHost()`, `awaitDecision()`, `applyAnswer()` and `decisionOutput()` for prompts; `runWatch()`, `watchOver()`, `replyOf()`, `takeReply()` and `replyNote()` for replies.

## Review results

With review results on (`--reviews`, or `reviews on`):

> when a review bot (CodeRabbit, say) or CI's check runs report on the pull request for a session's branch, AgentMingo decides what blocks and what to fix now, and the session gets that list with its next message (asked for just before the message reaches Claude, a second at most, so results that came in after the watcher stopped aren't a message late), or, if it's idle after pushing, is woken with it: a background hook waits up to 18 minutes after each reply for review results, asking every 30 seconds at first, then every minute, then every two; at most 3 automatic rounds per pull request. [...] A merge conflict on that pull request, and each step of the repository's pull request checklist, wake it too, outside those rounds; past them, results wait for you to say continue (your next message, or a reply from the thread's page). [...] `reviews off` stops all of that (a session that follows a workflow is still woken for its next step).

After the 3 automatic rounds, results are held: the watcher doesn't wake the thread with them. They come with your next message, or, with replies on, with a reply you send from the thread's page ("Continue" is enough). In versions before 15, a watcher stopped once the rounds were used up; from 15, one that listens up to 8 hours goes on listening, for a reply or a workflow step.

To wake a thread, the `watch` hook exits with code 2 and writes the note to stderr. That is how a background hook wakes a thread (`asyncRewake`). Every other hook exits 0. The woken turn runs on your own plan with your agent's provider, like any other turn.

## The watcher and its timeout

The `watch` hook runs in the background after each reply a thread makes, when there is something to wait for: review results on, a workflow, or replies on. In the script's own words:

> In a session that follows a workflow, and with replies on (above) in any session, it listens up to 8 hours instead (every five minutes after the first half hour), so a plan approved or a reply sent hours later still wakes it. It stops within seconds of you typing in the thread or the thread ending (it looks at the thread's state on this computer every 5 seconds, without asking AgentMingo), and the thread's next reply starts a watcher of its own, which takes over from it. Claude Code stops the hook at its timeout: install gives it 8 hours and 10 minutes, and this version raises an earlier install's 20 minutes itself, once, changing only that number in your settings (`status` says whether it did; if it couldn't, run the setup command again). Sessions already open keep the timeout they started with until you restart them (Claude Code may say their hooks changed), and the watcher listens within it.

The exact rules:

- **How long it listens.** 18 minutes (`WATCH_FOR_MS`) for review results alone; 8 hours (`WATCH_LONG_MS`) in a thread that follows a workflow, or in any thread with replies on. A thread that starts following a workflow while the watcher waits switches it to 8 hours.
- **How often it asks.** Every 30 seconds for 3 minutes, every minute until 10, every 2 minutes until 30, then every 5 minutes (`watchEveryMs()`): about 17 requests in 18 minutes, about 110 in 8 hours. While AgentMingo isn't answering, it asks nothing and waits.
- **When it stops early.** Within 5 seconds (`WATCH_CHECK_MS`) of you typing in the thread, the thread ending, a newer watcher taking the thread over, or the connection being refused, all looked at on this computer without a request (`watchOver()`). Also when AgentMingo says the computer is disconnected or refused, and, without replies or a workflow, when the branch has no pull request or the automatic rounds are used up.
- **The timeout.** Claude Code stops the hook at the timeout in your settings. Install gives it 29400 seconds (`WATCH_TIMEOUT_S`, 8 hours and 10 minutes). The watcher stops by itself a minute before the timeout its thread has: the one in your settings now, or the one the thread started with, whichever is lower (`sessionWatchTimeoutS()`).
- **The raise.** Installs before 15 gave the hook 1200 seconds. Once a computer runs version 15, its background sender, after it sends, raises that number to 29400 (`raiseWatchTimeout()`). It does so only from the copy install put in `~/.agentmingo`, only on the `watch` hook that copy added, only when its timeout is lower, and changes nothing else in `~/.claude/settings.json` (it writes the file with the same code install uses). Once it has checked, `config.json` notes it, and it doesn't look again, so a timeout you lower afterwards stays as you set it. When it can't write the settings (a link to a read-only file, say), it changes nothing, logs why in `reporter.log`, and tries again after a later send; `status` says to run the setup command again, which writes the new timeout.
- **Threads already open.** Claude Code may keep the hooks a thread started with, so a thread open during the raise keeps its 20 minutes until you restart it, and Claude Code may say its hooks changed. The script records the timeout each thread started with, and a thread that started before version 15 is taken to have the old one.

`status` and install say how long the watcher listens, from the timeout in your settings, and whether an update raised it.

## Cursor

> Cursor runs these hooks for its own agent too (it loads Claude Code's hooks by default). This reporter ignores Cursor's own agent for now: when the hook's input says it comes from Cursor, it exits at once and reads, sends and writes nothing. Claude Code inside Cursor (its extension, or Claude Code in Cursor's terminal) reports as usual.

The check is `fromCursor()`. It looks only at the hook's input (the fields `cursor_version` and `conversation_id`), not at environment variables. Versions 11 and newer have it. It covers the permission-prompt wait and the watcher too: a prompt from Cursor's own agent never opens a request, and the `watch` hook returns at once for it, so Cursor's own agent is never held for 8 hours, woken, or sent a reply.

## Updates and the kill switch

The script updates itself, so what runs later can differ from what you read today.

> It keeps itself current, so what runs later can differ from what you read here: when AgentMingo serves a newer version, it downloads it from there (the AgentMingo it reports to, such as agentmingo.com; the copy install put in ~/.agentmingo only), checks that it runs on this computer (in a scratch folder, connected to a stand-in for AgentMingo on this computer: Node loads it, `status` runs, a hook queues its event, and its sender sends it and goes back to this version when the stand-in says so), and uses it from the next hook, keeping the version before as reporter.prev.mjs. Hooks run it through launch.mjs, which runs that kept version instead when the new one doesn't load here (the kept version then puts itself back as reporter.mjs), and puts the kept version back itself when the new one loads but fails. AgentMingo can hold reporters at a version, or send them back to one (10 or newer).

Plainly:

- Updates come from agentmingo.com (the address you installed with) over HTTPS. The script refuses to update from a plain `http://` address other than this computer's own.
- Updates aren't signed. Before using a download, the script checks that it is the version asked for, starts as a Node.js script, is under 2 MB, and runs on your computer. It doesn't check who made it. It trusts the download because it came over HTTPS from the AgentMingo you connected to.
- The kill switch is AgentMingo telling reporters to go back to an earlier version (10 or newer), or to stay at one. It is the same check and swap, in the other direction.
- An update keeps your choices, answering prompts from AgentMingo included: the `PermissionRequest` hook keeps its wait when the hooks are pointed at a new layout. An update doesn't turn answering prompts on where it was off. Where it was on, an update to 15 adds replies, which go with it.
- The update to 15 changes one more thing in `~/.claude/settings.json`, once: the `watch` hook's timeout, from 20 minutes to 8 hours and 10 minutes. See [The watcher and its timeout](#the-watcher-and-its-timeout).
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
node ~/.agentmingo/reporter.mjs approvals off     # no permission prompts or replies on AgentMingo
node ~/.agentmingo/reporter.mjs auto-update off   # stay at this version
```

## Uninstall

```sh
node ~/.agentmingo/reporter.mjs uninstall
```

> `uninstall` takes the hooks, the rules and the MCP server out and disconnects this computer; it leaves ~/.agentmingo and the copy of your settings for you to delete.

That includes the `PermissionRequest` hook with its wait, and all five permission rules. Uninstall deletes `config.json`, so the connection token, and the key for answering prompts and replies if you turned that on, are gone from your computer. It doesn't touch `~/.claude/CLAUDE.md`. If you added the lean git rules, take them out with `node ~/.agentmingo/reporter.mjs lean-git off` before you delete `~/.agentmingo`. Their backup, `~/.claude/CLAUDE.md.agentmingo-backup`, stays for you to delete too. Task folders that `start` or `start_task` made are git worktrees of your clone, with the `.env` files copied into them: remove them with `git worktree remove` when you're done with them.

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

Product names here (Claude Code, Cursor, Codex, CodeRabbit, GitHub, Vercel) are trademarks of their owners. AgentMingo isn't affiliated with any of them.
