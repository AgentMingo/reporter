# AgentMingo reporter

This is the script that connects the coding-agent threads on your computer to [AgentMingo](https://agentmingo.com).
It is the same file agentmingo.com serves at <https://agentmingo.com/reporter.mjs>, published here so you can read exactly what it does before you run it.

AgentMingo is a free open beta. The AgentMingo server is closed source. This script is open source, under the Apache-2.0 license.

## What it is

`reporter.mjs` is one file of plain JavaScript with no dependencies.
Your coding agent runs it from its hooks at a few moments in each thread.
It reports those moments to AgentMingo, so your team sees every thread in one place and knows which ones need a person.
It also hands your threads what they can't see on their own: a ticket's brief, review and CI results for their branch, teammates' live threads, your team's rules, and workflow steps one at a time.
It can start a thread on a task in a folder of its own, and, only if you turn it on on that computer, let you answer a thread's prompts (permissions for any tool, its questions, its plans), and reply to a thread that waits for you, from AgentMingo. From version 19, also only if you turn it on, a Mac can start a thread on a task when you press "Start on my computer" on the task's page (see [Starting threads from AgentMingo](#starting-threads-from-agentmingo-macos)).

AgentMingo isn't tied to one agent or model vendor. Today this script connects one agent, Claude Code, through Claude Code's hooks. Other agents (Cursor's own agent, Codex and the rest) aren't connected yet. Review results work the same way whichever review bot comments on the pull request: CodeRabbit is one example.

## Requirements

- macOS or Linux. Native Windows isn't supported yet.
- Node.js 18 or newer (`node --version`).
- Claude Code (`claude --version`), the agent this script connects today.

Cursor's own agent (its chat and Agent panel) is ignored. Claude Code inside Cursor works. See [Cursor](#cursor).

## Install

You get a setup command from the Sessions page in AgentMingo (or its setup guide). It looks like this (`acme` and the code are examples). Each of its first three boxes adds a flag: `--share-files` or `--no-share-files`, `--reviews` or `--no-reviews`, and `--approvals` or `--no-approvals`. From version 19, a fourth box, "Start threads on this computer from AgentMingo (Mac only)", is unticked by default and adds `--start-from-web` only when ticked (see [its box](#the-box-start-threads-on-this-computer-from-agentmingo-mac-only)). This is the command with the first three ticked, as they are by default:

```sh
curl -fsSL https://agentmingo.com/reporter.mjs -o ~/.agentmingo/reporter.mjs --create-dirs \
  && node ~/.agentmingo/reporter.mjs install --url https://agentmingo.com --org acme --code ama_… --share-files --reviews --approvals
```

The code works once, for 15 minutes. Install trades it for this computer's token, so the token never appears in a command line or a shell history. AgentMingo stores only the code's hash. The code starts with `ama_` when the command was made with the box "Answer prompts and reply to threads from AgentMingo" ticked, and with `amc_` when it wasn't (see [The box](#the-box-answer-prompts-and-reply-to-threads-from-agentmingo)).

Before it trades the code, install reads Claude Code's settings (`~/.claude/settings.json`, or where its hooks went before). When that file isn't valid JSON, install stops and changes nothing, and the code still works: fix the file and run the same command again within its 15 minutes. Versions before 16 traded the code first, so a broken settings file used it up.

To read the file before it runs, split the command in two:

1. Run the `curl` part only.
2. Read `~/.agentmingo/reporter.mjs`, and [check its sha256](#check-that-your-copy-is-a-published-one).
3. Run the `node … install …` part.

Running install again is safe. It replaces its own hooks instead of adding them twice, and it keeps your earlier choices (titles, sharing, review results, updating, answering prompts and replies from AgentMingo, starting threads from AgentMingo) unless a flag changes them.

### The box "Answer prompts and reply to threads from AgentMingo"

This box, on the Sessions page and in the setup guide, is ticked by default. Ticked, the setup command carries `--approvals`, its one-time code starts with `ama_`, and install turns on [answering prompts and replying from AgentMingo](#answering-prompts-and-replying-from-agentmingo) on this computer. Unticked, the code starts with `amc_` and the command carries `--no-approvals` instead, which turns answering prompts and replies off on that computer, or keeps them off.

From version 16, the setup command turns it on wherever it runs: in a terminal, in your editor's terminal, or in a coding agent's thread you paste the command into for the agent to run. You don't have to run anything else, and you don't have to run `approvals on`. The one-time code is your consent: it's made only for you, signed in to AgentMingo, it works once, and for 15 minutes.

Install turns it on only when both hold:

- the command carries `--approvals`;
- the command carries a one-time code (`--code`) made with the box ticked. Install learns that when it trades the code: AgentMingo answers whether the code was made with the box ticked, from the code itself, not from the flag.

Otherwise it stays off, and install still connects this computer. It prints a line saying it stayed off, why, and how to turn it on:

- a command made with the box unticked, with `--approvals` added by hand: the code still says unticked, so it stays off;
- the older form that carries `--token` instead of a code: it stays off.

`--approvals` never makes the install fail. A code whose prefix was changed by hand (`amc_` to `ama_`) is another matter: it is no longer the code AgentMingo made, since AgentMingo keeps only the hash of the whole code, so install stops as it does for an expired code, before anything changes. When answering prompts is on already, `--approvals` changes nothing, so running the command again from a provisioning script still works.

When it turns it on, install prints what it turned on, what is sent, and how to turn it off. The choice is the same as `approvals on` with no options (every prompt, an hour's wait, replies), without typing yes: the one-time code is the yes. To change the wait, run `approvals on --wait` afterwards. `--no-approvals` turns it off. Versions 15 and 16 turned on file edits only here, with commands not sent; from 17 there is no narrower choice (see [Prompts](#prompts)).

Its limits, plainly:

- **Whoever runs a command made with the box ticked, while its code is unused, turns it on.** That's you, or the coding agent you gave the command to, which runs everything else in the command too. Until the code is used or its 15 minutes are up, the command connects a computer as you, so give it only to a terminal or an agent you mean to run it.
- **A command pasted into a thread becomes part of that conversation**, which your agent sends to its model provider under your own account and plan. Once install has traded the code, the code is no use to anyone. The reporter masks AgentMingo's codes and tokens (`ama_`, `amc_` and `amk_`) in titles and in a waiting thread's notice.
- The setup command comes from AgentMingo, and it already downloads and runs the script, so the flag adds no trust of its own: what counts is that you run it, or give it to your agent to run. AgentMingo's website and servers can't turn it on on a computer by themselves, without a command run there (an update of the script can change anything it does, as it always could; see [Updates](#updates-and-the-kill-switch)).
- Which box a code was made with, and that it's made only for you and works once for 15 minutes, are enforced by AgentMingo's server, which this script can't show you. The script trusts the server's answer (`tradeSetupCode()`), and checks nothing about the terminal it runs in.

Version 15 turned it on only in a real terminal, outside Claude Code's Bash tool, and refused the whole install otherwise. Version 16 dropped those checks for the setup command, because the one-time code already stands for your consent. They remain for `approvals on`, which has no code.

### The box "Start threads on this computer from AgentMingo (Mac only)"

From version 19. This box is unticked by default. Ticked, the setup command carries `--start-from-web`, and on a Mac install turns on [starting threads from AgentMingo](#starting-threads-from-agentmingo-macos), only when the one-time code it trades was made with the box ticked. AgentMingo keeps that choice with the code and says it back when install trades it, as it does for the box above, so adding `--start-from-web` by hand to a command made unticked leaves it off.

Unticked, the command carries no flag for it, and the computer keeps what it had. It's one setting for the whole computer, so a command made unticked in another organization, or run again later, doesn't turn it off for all of them. `start-from-web off`, `uninstall`, or Turn off on your Account page in AgentMingo turn it off; install's `--no-start-from-web`, added by hand, does too.

It stays off, and install says why, when the code was made unticked, on Linux or Windows, with the older `--token` form, or when the copy running isn't the one the setup command installs in `~/.agentmingo`. Without the setup command, `start-from-web on` in a terminal of your own turns it on: it explains what it does, asks you to type yes, and refuses inside Claude Code's Bash tool and without a terminal.

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

They let the agent report a workflow step, and make a task's folder, without asking you. They allow only those two subcommands of this script, and the three tools of its own MCP server. Claude Code matches `start:*` only when a space follows `start`, so `start-from-web` and `run-start` aren't covered: the agent asks before it runs them. Making a task's folder doesn't move the thread there: the `start_task` tool's answer tells the agent to ask you first (see [Starting a task](#starting-a-task)).

**An MCP server** named `agentmingo`, registered for your user with `claude mcp add -s user` (in `~/.claude.json`). It runs this script with `mcp`, and has three tools:

- `current_step`: the thread's current workflow step. It sends the folder's repository and branch.
- `complete_step`: reports a step done or failed, with what the agent wrote and the files it attaches.
- `start_task`: makes a task's folder when you ask a thread to work on a task ("Work on WEB-12"). It sends the task's key, and a repository when you name one.

Install's `--no-mcp` skips it. When the `claude` command isn't found or doesn't answer, install prints the `claude mcp add` line to run yourself instead.

**The folder `~/.agentmingo`.** See [What it keeps on your computer](#what-it-keeps-on-your-computer).

**Only with `--lean-git`** (or `lean-git on`): six short git habits added to `~/.claude/CLAUDE.md`, between marker comments, with a copy of the file as it was kept beside it as `CLAUDE.md.agentmingo-backup`, when there was one. `lean-git off` takes them out.

**Only with starting threads from AgentMingo on, on a Mac** (version 19): a LaunchAgent, `~/Library/LaunchAgents/com.agentmingo.start-from-web.plist`, started with `launchctl bootstrap gui/<your uid>`. It runs `"/Users/you/.agentmingo/reporter.sh" start-from-web helper --home /Users/you/.agentmingo` while you're logged in, with only the system PATH (`/usr/bin:/bin:/usr/sbin:/sbin`) and no other environment variable, and launchd starts it again a minute after it fails (not after it stops because starting threads is off). What it prints to standard error goes to `~/.agentmingo/helper.log`. The file's first lines say it's AgentMingo's: the reporter never writes over or removes a LaunchAgent at that path without them. `start-from-web off` and `uninstall` stop it (`launchctl bootout`) and remove the file. Its full text comes from `startHelperPlist()` in `reporter.mjs`.

Nothing else on your computer changes.

## Which threads report

In the script's own words:

> Which threads report: only those in a repository your organization has connected in AgentMingo (also under an earlier name, when it was renamed or moved: GitHub keeps old names working, so an older clone may still use one), or one you added (`repos add` in its folder; the Sessions page lists them and removes them), named by the folder's origin remote. A thread anywhere else (another client's repository, a side project, a folder that isn't a git repository) sends nothing about itself: no events, no lookups, and AgentMingo's notes don't reach it. One thing is asked about such a folder: when its repository shares its owner or its name with one an organization lists (acme/web-old next to acme/web, say), this computer sends that name to the organization once a day, to ask whether it's the same repository under an earlier name. AgentMingo asks GitHub, and keeps the name only if it is. Until this computer has heard an organization's list (just installed or updated), it asks for it before sending anything about a thread, and drops what it queued for threads outside it.

`node ~/.agentmingo/reporter.mjs repos` lists the repositories that report from your computer. A prompt is opened on AgentMingo only for a thread that reports, and only such a thread's watcher listens for replies.

## What it sends

This is the list in the script's header, quoted verbatim. AgentMingo's [Privacy Policy](https://agentmingo.com/privacy#collect) lists the same things in its own words, and says how long AgentMingo keeps them.

> What it sends for those, every request but setting up and downloading an update (below) carrying this computer's connection token (`authorization: Bearer amk_…`) and the script's name and version (user-agent `agentmingo-reporter/` and a number, and from version 19 this computer's platform, like "(darwin)"), and like any request, your IP address:
>
> - What happens in each thread, and when: it started, you sent a message, it's working (at most every five minutes, and at once after a tool call that ran long), with how long the tool call behind it ran, it waits for you, it was answered, it replied (and whether it was a short turn to keep listening, below), it ended. Each with its time, a random id of its own (so it's recorded once), and Claude Code's id for the thread.
> - Where it works: its repository name and branch, whether the branch is pushed and how many of its commits aren't, how many files have changes not yet committed (a count, never their names or what changed), and the app it runs in.
> - Counts from the session's transcript, from when the reporter first saw the session: token counts by model (with the model's name), tool calls counted by kind (git, GitHub's command line, tests, builds, installs, other shell commands, reading files, edits, web, subagents, MCP tools, and any other tool) with how much output they produced and about how long they took, and git commands counted by subcommand (diff, log, status, push and so on) with how much output each produced. When a thread's transcript stops giving those counts (a Claude Code update can change how it writes transcripts), it says which of them is missing, so `status` and AgentMingo's admins can see it.
> - A short title (Claude Code's own session title, or the first line of your first message, with emails, phone numbers, long numbers, keys and tokens by their known shapes, private keys, credentials in addresses, headers and command lines, and secret-looking values given to a password, secret, token or key masked on this computer; turn titles off with --no-titles or `titles off`), and what a waiting session's notice asks you (masked the same way; with titles off, only whether it asks for a permission, and for which tool, or has a question).
> - Its settings: whether review results are on here, whether sessions know about each other here, whether the lean git rules are in a CLAUDE.md it reads or were given to it in a note, and while its organization measures them, which group of the test the session is in (with the rules, or without).
> - What it passed on: which review results, notices and workflow step it handed the session, so none is handed twice.
> - What it asks for: when a session starts on a branch whose name looks like a task's key, or comes onto one (it switched branches, or moved into a task's folder), that branch and the repository, for the task's brief. Before each message, the session's id, for its workflow step (when it follows one, or its branch names a task or it has one) and, with review results on, for results and notices waiting for it. After each reply, the watcher below.
>
> With --share-files (the setup command's "Let sessions know about each other", ticked by default) it also sends the paths of files a session edits, each with when it was edited, says sharing is on, and tells each session (in one short note) which other sessions are working on which files in the same repository (their person's first name, title and branch), so they stay out of each other's way. With it off (--no-share-files, or `share-files off`), it sends no paths, your sessions get no such note, and AgentMingo names your sessions in no teammate's notes: not in their note about other sessions, and not when someone else's thread works on the same task.
>
> Answering permission prompts and replies from AgentMingo is off unless you turn it on here: by running the setup command with its box "Answer prompts and reply to threads from AgentMingo" ticked (it is by default: install's --approvals, which turns it on with the command's one-time code from your signed-in Sessions page wherever the command runs, in a terminal, an editor or a coding agent's thread you paste it into, and says so), or with `approvals on` in your own terminal, which asks you to type yes. The one-time code is your consent: it's made only for you, signed in to AgentMingo (the Sessions page or the setup guide), it works once, and for 15 minutes. The code carries the box's choice, and AgentMingo says it back when install trades the code: adding --approvals to a command made with the box unticked doesn't turn it on (install connects this computer and says it stayed off), and the code can't be changed to say otherwise without breaking it. With the older --token form, --approvals leaves it off too: install still connects this computer and says how to turn it on. The limit: whoever runs a command made with the box ticked while its code is unused (you, or the coding agent you gave it to) turns it on, as they run everything else the command does. `approvals on` refuses inside Claude Code's Bash tool and without a terminal: those checks catch a command run by mistake or by your coding agent's Bash tool; a program on your computer set on faking a terminal could get past them. AgentMingo's website and servers can't turn it on by themselves: the setup command carries the flag, and you run it, or give it to your coding agent to run (it already downloads and runs this script); an update of this reporter can change anything it does, as it always could (turn updates off with `auto-update off`). Once it's on, when a session started since stops on a prompt (a permission for any tool: a file edit, a command, an MCP or other tool; a question; a plan to approve), Claude Code's prompt stays on screen as always, and the hook also opens the prompt on the thread's page in AgentMingo and waits up to the time you chose (an hour unless you say otherwise) for your answer there, with the same answers as the terminal; whichever is answered first counts. For that prompt it sends the tool's name, a random id, an HMAC that ties an answer to this prompt (made with a key that never leaves this computer), a hash of what it shows, how long it will wait, and what it wants, masked like a title: for a file change, the file's path (from the repository's top, or as given outside it, from ~ in your home folder) with how many lines it adds and removes, never the change itself; for a command, the command (20 lines and 1,000 characters at most) and its description; for an MCP or other tool, its MCP server and tool, and its input (every field it carries) in one line of at most 300 characters; for a question, its questions and options; for a plan, the plan (at most 20,000 characters); what kind of thing it is (outside the repository, Claude Code's settings, a file that can run code by itself, a command, characters that don't show); and what Claude Code's "don't ask again" would allow (offered only when every rule and folder it adds can be shown whole). While it waits it asks AgentMingo for an answer every 3 seconds for two minutes, then every 10, and says when it stops waiting. Anything can be allowed there, as in the terminal (a deny rule in your Claude Code settings still wins): Yes applies to that one call; "Yes, and don't ask again" applies Claude Code's own suggestions for this prompt, for the rest of this session only (never written to a settings file); No denies it, with your note quoted to Claude as your words. A question's answers and a plan's approval reach Claude as the tool's input. It waits only in a thread's main conversation (not a subagent's), not in a headless run, not on a Claude Code version known to misbehave (none so far), and in a terminal only once Claude Code says the prompt is on screen (within 20 seconds; else it leaves the prompt to the terminal at once): that notice is the guard. Claude Code doesn't document that a hook can wait while its prompt stays usable in a terminal, so a release of it could change that; in its VS Code extension and desktop app, which send no such notice, Claude Code shows its prompt and runs the hook at the same time, and takes whichever answers first. When Claude Code stops the hook, it tells AgentMingo it stopped waiting. Someone who gets into your AgentMingo account could allow what your agent asks to run on this computer, and someone who controls AgentMingo's servers or database could answer for you. `approvals off` turns it off.
>
> With it on, a thread that finishes its turn and waits for you can also be answered from its page in AgentMingo (Continue, Yes, No, or up to 1,000 characters of your own): the watcher below listens up to 8 hours after each reply, and when you send one, it takes it (once: it's claimed on this computer and on AgentMingo) and wakes the thread with it, introduced as your reply, with what your next message would have carried (the next workflow step, review results). For that, each report also says replies are on here, and the watcher says when it asks next. Only you can send one, signed in as yourself; someone who gets into your AgentMingo account could send your threads a reply, and allow what they then ask to run. A reply is for the turn it answers: typing in the thread first, the thread ending, or the thread going on without it (woken for something else) cancels it.
>
> Keep listening (from version 18): a thread that waits for you on work still open (its task isn't done or canceled, it has an open pull request, or it follows a workflow run that's going) stays reachable from its page while that work is open and this computer stays on and awake, a week at most, with replies on here and your setting in AgentMingo on (it is by default: "Keep threads listening while their work is open", on its Notifications page). As the watcher's time runs out (its 8 hours, or any shorter limit), it asks AgentMingo once more, so a reply sent in its last minutes is still picked up, saying (with replies on) it stops listening now. When it listened its whole 8 hours and AgentMingo's latest answer says to keep listening (your setting is on, the thread still waits for you on open work, and its wait began less than a week ago), it wakes the thread with a short note asking Claude to reply only "Still listening for AgentMingo" and do nothing else, so the thread takes one tiny turn and that turn's Stop starts a fresh watcher: about every 8 hours, up to 3 a day. AgentMingo's answer is only yes or no: the note's words are fixed in this script (KEEP_LISTENING_NOTE). At most 21 times in one wait (7 days of 8 hours; you typing in the thread, or a real turn ending, starts the count again), and never in a thread whose watch hook can't give it its whole 8 hours (one started before 8-hour listening keeps its 20 minutes: each such turn would buy it only 20 more), nor when you typed in the thread, it ended, a newer watcher took over, AgentMingo refused this computer, or you turned replies off here since. It stops, and nothing on the page reaches the thread until you type in it, if this computer is turned off or restarts, Claude Code is closed, or the short turn fails (a usage limit, say); and it may stop if this computer is asleep as a watch ends (as it wakes, it still asks and keeps listening, unless Claude Code stopped the hook while it slept: within the hook's last 10 minutes it never has). Each of those turns re-reads the whole conversation, so it uses tokens on your own Claude plan like any other turn, more the longer the conversation (8 hours on, it's no longer cached), counted in the thread's usage. Its Stop's report says it was such a turn (keepAlive), so AgentMingo doesn't start a new turn for it or tell you about it: that flag is the one new thing this sends. The count, and whether the next Stop is such a turn, stay on this computer (state/<session id>.keep.json, removed when you type in the thread, a real turn ends, or the thread starts again or ends; a compaction keeps it).
>
> On a workflow (below), it also sends whether a step Claude reports is done or failed, and what Claude writes when it reports a step (for the plan step, the plan itself, with the repositories it names and whether it needs an end-to-end test, and why; AgentMingo masks the same details when it arrives), the files Claude attaches to a step (reports, screenshots, videos, logs; up to 4 MB each, 20 at most), each with its name or its path in the session's folder, and when Claude asks for its current step, the folder's repository and branch. The commands send what they name: `start` and the start_task tool the task's key (and the repository, with --repo), `repos add` and `repos remove` a repository's name, and `repos` asks for the list. On a computer connected to several organizations, a step report and `start` go first to the organization of the folder's repository, and to the next only if that one doesn't know the step or the task. Install sends the setup command's one-time code, the organization's short name, the script's version and (from reporter 19 on) the computer's platform, and nothing else; when it replaces an earlier connection, and on uninstall, it asks AgentMingo to disconnect that one. Updating downloads the new script without its token and without anything about you or your threads. It doesn't send your code or other files (unless Claude attaches one to a step), the commands Claude runs (only their kind, and git's subcommand, are counted), tool input, or the rest of the conversation, except, with answering prompts from AgentMingo turned on here (above), what a waiting prompt wants, masked: a file's path and line counts, a command, a tool's input in one line, a question, a plan.

The header is written for the one agent the script connects today, so where it says "Claude", it means that agent. The exact fields of each report are built mainly in two functions you can read: `hookToEvent()` (the moment itself) and `enrich()` (what is added to it before it is sent). `runHook()` adds which notes it passed on. A waiting prompt's request is built in `openRequest()` from `permissionTarget()`: besides what the header lists, it carries the thread's id, the kind of prompt (a file edit, a file write, a new file, a notebook, a command, an MCP tool, another tool, a question or a plan), its `preview` (a file's path, the command, or another tool's input in one line), and `details`: its badges (the `BADGES` constant: outside the repository, Claude Code's or an agent's settings, a file that can run code by itself, characters that don't show, partly masked, a path that doesn't lead straight to its file, a command, shown cut, an MCP tool), a command's description, the MCP server and tool, the questions or the plan, and what "don't ask again" would allow. The hash of what it shows covers the details too. It still carries `canAllow` and `why` for AgentMingo's older pages: from 17, `canAllow` is always true and `why` empty. The `needs_you` event a prompt sends says which kind it is ("Permission to use" a tool by its name, "Has a question for you" or "Has a plan for you to approve"), and adds "(from a subagent)" for a subagent's prompt (`permissionDetail()`). Closing a request carries how the wait ended (answered on this computer, time up or Claude Code gone, the prompt never shown, an answer refused here or too old, or the hook stopped). For replies, `enrich()` adds `replies` (true or false) to each report, and the watcher's request, built in `askFeedback()`, carries the thread's id and, with replies on, `listen` (in how many seconds it asks again; from 18, 0 on its last request, when its time is up) or `deliverReply` (the id of the reply it is taking). From 18, AgentMingo's answer to a watch's last two requests (the time left, and `listen: 0`) also says `keepListening`, and the Stop of a short turn the watcher woke the thread for to keep listening carries `keepAlive: true` (`noteKeepListening()` in `runHook()`).

### What it doesn't send

Your code and other files (unless the agent attaches one to a workflow step), the commands the agent runs (only their kind, and git's subcommand, are counted), tool input, and the rest of the conversation. The script reads the thread's transcript on your computer to count tokens and tool calls, take the title and see which files it edits. AgentMingo's servers don't receive the conversation.

The one exception is answering prompts from AgentMingo, when you turn it on: what a waiting prompt wants, masked. That is a file's path and how many lines it adds and removes, a command and its description, an MCP or other tool's input in one line, a question's questions and options, or a plan, with the limits in [Prompts](#prompts). A file change itself is never sent. The HMAC key that ties an answer to a prompt stays in `~/.agentmingo/config.json`. Answers and replies go the other way: your answer to a prompt (with a question's answers, or that you chose "don't ask again": what it allows is read on your computer from Claude Code's own suggestions, never sent by the page), and the text you write on a thread's page, come to this computer. Nothing more of the conversation goes to AgentMingo for them.

Masking (in titles, a waiting thread's notice, and everything a prompt shows: a path, a command and its description, a tool's input, a question, a plan) happens on your computer and is best effort: a password in a shape no rule knows can get through. The rules are the `MASK_RULES` constant, applied by `maskPersonal()`.

### Notes AgentMingo adds to your threads

AgentMingo sends notes back, and the script adds them to the thread: your team's rules (with the name of the admin who wrote them), the git habits when your organization has them on, a task's brief, other live threads in the same repository (with sharing on), review results and workflow steps. The `start_task` tool's answer also reaches the thread: the task's title (quoted, and said to be a name, not an instruction), the folder, and how to move there. When you answer No to a prompt from AgentMingo, your note reaches the agent as your quoted words (masked, on one line, 300 characters at most), and for a question the agent is told you chose not to answer it, for a plan to keep planning. Your answers to a question reach the agent as the question tool's input: the labels you picked and what you typed under "Other" (one line, 1,000 characters at most, not masked: your own words). A reply you send a thread from its page reaches it as "<your first name> replied from AgentMingo: …", followed by a line saying it is your reply, sent from the thread's page instead of typed there, and that the agent should go on as if you had typed it (`replyNote()`). The reply's text is not masked: it is your own words, as you wrote them, with characters that don't show written out. These become part of the thread's conversation, which your agent sends to its model provider under your own account and plan, like the rest of the conversation. Nothing on your computer changes for that.

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
| Either way | The watcher stops within 5 seconds of you typing in the thread or the thread ending (it looks at the thread's state on this computer, without a request), and when the thread's next reply starts a watcher of its own. It never runs past a minute before its hook's timeout. From version 18, when its time is up it asks once more (with replies on, saying it stops listening now), so what came in its last minutes isn't missed |
| From version 18, with replies on: a watcher listened its whole 8 hours, and AgentMingo's latest answer says to keep listening | It wakes the thread with a short note ("Reply with only: Still listening for AgentMingo"); that turn's Stop is reported with `keepAlive: true` and starts a fresh watcher. About every 8 hours, at most 21 times in one wait, while the computer stays on and awake. See [Keep listening](#keep-listening) |
| With replies on, a reply is waiting | The watcher claims it on this computer, asks AgentMingo to hand it over (once), and wakes the thread with it, with the workflow step and (with review results on) the review results that were waiting, each marked as told |
| Only with answering prompts on: a thread stops on a prompt (a permission, a question or a plan) | The prompt's request (see [What it sends](#what-it-sends)), then a check for an answer every 3 seconds for two minutes, then every 10 seconds, until it is answered on your computer or on AgentMingo, its time is up, Claude Code stops the hook or is gone, or (in a terminal) Claude Code hasn't said within 20 seconds that the prompt is on screen. When it stops waiting without applying an answer from there, it tells AgentMingo so (best effort when Claude Code stops the hook; not when AgentMingo refused the check, no longer knows the request, or closed it already); after it applies one, an `answered` event |
| Only with starting threads from AgentMingo on, on a Mac (version 19), while you're logged in and it's awake | The helper's check, about every 10 seconds (AgentMingo can ask for 5 seconds to 5 minutes; a minute while it pauses starts), to each organization that still takes starts on this computer. It says only that the helper runs (or was turned on or off) and whether answering prompts from AgentMingo is on here. When AgentMingo doesn't answer, it waits 30 seconds, doubling to 5 minutes. For each start AgentMingo hands it: one request to take it, then one or two saying how it went (opened, then started or why not) |
| At most every 15 minutes | An empty report that asks for your organization's list of repositories. It carries nothing about any thread |
| Once a day per name | The name check described in [Which threads report](#which-threads-report) |
| When an answer says a different version should run | A download of `reporter.mjs` from the same AgentMingo, without the token (at most every 6 hours for the same version, or every 10 minutes when going back; none when going back to the version kept in `reporter.prev.mjs`; see [Updates](#updates-and-the-kill-switch)) |
| When you or the agent run a command | `install` (trades the code), `repos`, `repos add` or `repos remove` (the repository's name), `start` or the `start_task` tool (the task key, and the repository when you name one), `step` or the step MCP tools (the step's text and attached files; `current_step` sends the folder's repository and branch), `uninstall` (disconnects this computer). `approvals on`, `off` and `status` send nothing, and neither does install's `--approvals` beyond what install sends anyway. `start-from-web on` and `off` send the helper's check once to each organization, saying it was turned on or off; `start-from-web status` sends nothing |

When AgentMingo doesn't answer, the script holds back:

- A lookup (a task's brief, or what's waiting before a message) waits a second at most. When one gets no answer, lookups pause for 60 seconds, so a slow AgentMingo costs one message a second, not every message. What the lookup would have brought comes with a later answer instead.
- Sending stops for 30 seconds, doubling up to 10 minutes (or as long as a 429 asks), then what waited is sent.
- While lookups are paused, or sending is paused because AgentMingo didn't answer (not for a 429 or a reached limit), or the connection is refused (below), a prompt opens no request on AgentMingo: it stays in the terminal only.
- When AgentMingo says a limit on reports was reached, the events it skipped are dropped, and nothing more is sent until the limit's window ends.
- When it refuses the connection (403: the account or organization is suspended, or you were made a viewer), nothing is asked or sent there for an hour, doubling to a day, or until install runs again.

## What it keeps on your computer

Everything is in `~/.agentmingo`:

- `reporter.mjs`: this script. `reporter.prev.mjs`: the version before the last update, kept to go back to.
- `launch.mjs`: the launcher the hooks and the MCP server run. It runs `reporter.mjs`, or the kept version when the new one doesn't load. When the new one loads but fails, it puts the kept version back as `reporter.mjs` for the next run. Its full text is the `LAUNCHER` constant in `reporter.mjs`, and it doesn't change when the script updates.
- `reporter.sh` (macOS and Linux): a short shell script that runs `launch.mjs` with the Node.js install ran with, or the `node` on your PATH once that one is gone (an nvm or Homebrew upgrade). With neither, it does nothing. Its text comes from `shimText()` in `reporter.mjs`, with your Node.js path and home folder filled in.
- `config.json`: your choices, this computer's connection token and each organization's list of repositories, readable by you only (mode 600). With answering prompts and replies on, it also holds that choice (the wait, and `files` and `commands` flags that version 17 no longer reads: when 17 turns it on, it writes both as true, so a computer sent back to 16 shows commands to deny; a config written by an earlier version keeps the values it had, and may also hold an `unchecked` field, from versions 14 to 16, that no longer changes anything) and the key the HMAC is made with, which is made on your computer by `approvals on` or by install's `--approvals` and never sent; `approvals off` takes both out, and replies stop with them, since they go by the key. From version 15 it also notes the watch hook's timeout: that this version checked it (`watchTimeoutChecked`), and, when it raised it, when (`watchTimeoutRaisedAt`) and what it was before (`watchTimeoutWas`). When AgentMingo removes this computer's only connection, it is renamed `config.json.removed`.
- `outbox/`: events waiting to be sent.
- `flush.lock`: there while the background sender runs, so only one sends at a time.
- `state/`: each thread's state, what's waiting for it, when each organization's list of repositories was last heard, the names asked about once a day, team rules, other live threads in the same repository (with sharing on), when AgentMingo last didn't answer (`backoff.json`, and `lookups.json` for the 60-second lookup pause), the version AgentMingo last said to run and the last update tried, and lock files.
- `state/<session id>.asked.json`: the branch a thread last asked about for its task's brief, so a message asks again only when that changes. `state/<session id>.prompt.json`: when Claude Code last said a permission prompt was on screen in that thread, which a waiting hook needs in a terminal. Both are removed when the thread ends.
- `state/<session id>.hook.json`: the thread's state. From version 15 it also counts the thread's ends (with the messages you typed, which it counted already), so a watcher knows you typed or the thread ended, and notes the watch hook's timeout when the thread started.
- `state/<session id>.watch.lock`: names the thread's current watcher (its process, a random id and when it started). A newer reply's watcher writes its own, and the older one sees that and stops.
- `state/<session id>.keep.json` (version 18): how many times the thread's watcher woke it to keep listening in this wait, and whether its next Stop ends that short turn. Removed when you type in the thread, a real turn ends, or the thread starts again (a compaction, which a short turn can bring on, keeps it) or ends, whether or not its repository still reports.
- `state/<session id>.told/`: one empty file per review result or notice a thread was told, so the message hook and the `watch` hook never tell it the same thing twice. With replies on, also one per reply a watcher took (`reply-…`), so a reply is handed over once even when two watchers see it. A thread's folder is removed when it ends; one left by a thread that never ended is removed, when a later thread starts, once nothing was added to it for 7 days.
- `repos.json`: where each repository is cloned, so `start` and `start_task` work from any folder.
- Only with starting threads from AgentMingo on (version 19): `starts/` (mode 700), each start's files while it runs: `<id>.claimed` (so a start opens once), `<id>.json` (the start's id, its task's key, and which AgentMingo address and organization it came from, never the token) and `<id>.command` (what Terminal runs: `exec <reporter.sh> run-start <id> --home <folder>`, and nothing else). `run-start` takes the last two as it begins, and files a closed window left are removed after a day. `state/starts.json`: when the helper last heard each organization, when it opened the starts of the last hour, and the last result. `state/start-helper.pid`: the running helper's process id, so only one runs. `helper.log`: what launchd catches from the helper, emptied past 1 MB.
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

**What AgentMingo sends back is checked first.** From version 19, before anything reaches git or a path, `start`, `start_task` and `run-start` check AgentMingo's answer (`checkedStart()`). The task's key must be the one asked for. The branch must be the key, or the key and words of its title. The repository's names must be `owner/name`, and its main branch must pass `git check-ref-format --branch`, with no leading `-` and no `..`. Otherwise nothing is made. Task titles are printed without control characters (`terminalSafe()`).

## Starting threads from AgentMingo (macOS)

From version 19, macOS only for now, and off unless you turn it on (the setup command's [box](#the-box-start-threads-on-this-computer-from-agentmingo-mac-only), or `start-from-web on` in your own terminal). In the script's own words:

> Starting threads from AgentMingo (from version 19; macOS only for now; off unless you turn it on here): with it on, "Start on my computer" on a task's page in AgentMingo, from any device you're signed in on, starts a thread on this Mac. A small helper (a LaunchAgent, ~/Library/LaunchAgents/com.agentmingo.start-from-web.plist, that runs this script's `start-from-web helper` while you're logged in) asks AgentMingo about every 10 seconds, while the Mac is awake, whether you pressed it, sending only that it's on and whether answering prompts from AgentMingo is on here. When you did, AgentMingo sends a start's id and the task's key, nothing else. The helper checks both, takes the start (so it runs once), and opens Terminal (`open -g -a Terminal`) on a small file in ~/.agentmingo/starts that runs `run-start <id>` there: a fixed command built on this computer. In that window run-start makes the task's folder as `start` does (the same safe .env files), and runs Claude Code there with the prompt "Work on <task key>", as you'd start it yourself: Claude Code's own permission settings apply (no permission flag, and no settings or MCP config of AgentMingo's). AgentMingo hears how it went (opened, started, or why it didn't, in one line with your home folder shown as ~, and the repository's name), and the thread's first report says which start it came from. It opens at most 6 starts an hour, none within 15 seconds of the last, whatever AgentMingo sends. Only you can press it, signed in as yourself (not teammates or admins): someone who gets into your AgentMingo account could start a thread on a task here, and answer its prompts if answering them from AgentMingo is on. Turn it on with the setup command's box "Start threads on this computer from AgentMingo" (install's --start-from-web, with its one-time code), or with `start-from-web on` in your own terminal (it asks you to type yes). Turn it off with `start-from-web off` (it stops the helper and removes its LaunchAgent; so does uninstall), or on your Account page in AgentMingo, one organization at a time (at its next check the helper stops taking that organization's starts, and stops once none is left). A thread that starts this way may still ask, in its Terminal window, what only this Mac can answer (whether to trust a new folder, say). `start-from-web helper --once` runs one of the helper's rounds by hand and prints each step.

What the code does, step by step, with the functions to read:

1. **The helper** (`runStartHelper()`) runs while starting threads from AgentMingo is on in `config.json`, one at a time (its pid file). Each round, it asks each organization that still takes starts on this computer (`POST /api/v1/agent-sessions/starts/check`), and waits as long as AgentMingo says (5 seconds to 5 minutes). It stops when it's turned off on this computer, or every organization turned it off, and restarts as the new version after an update.
2. **A start** (`handleStart()`): the id must be AgentMingo's shape (`START_ID`) and the task's key a key (`TASK_KEY`). It takes the start (`starts/claim`), so it opens once even if AgentMingo hands it out twice. It writes the start's two files in `~/.agentmingo/starts` and opens the `.command` file with `open -g -a Terminal`, without a shell or AppleScript and without taking the focus. At most 6 an hour, and none within 15 seconds of the last (`START_OPENS_PER_HOUR`, `START_OPENS_APART_MS`).
3. **The Terminal window** (`runStart()`): `run-start` takes the start's file (so it runs once), makes the task's folder as `start` does, from that start's organization only, and runs `claude "Work on WEB-12"` there, plus `--add-dir` for the other repositories a task names, with `AGENTMINGO_START=<id>` in its environment. It never passes a permission flag, settings, or an MCP config. It refuses to run inside Claude Code's Bash tool, or without a terminal.
4. **AgentMingo hears how it went** (`sendStartResult()`): opened, started, or failed with one reason from a fixed list (`REPORTER_START_FAILURES`), one line of detail (300 characters at most, your home folder shown as `~`), and the repository's name. The thread's first report carries the start's id, so the task's page links to the thread.

**What it trusts.** AgentMingo can only ask your Mac to start Claude Code in a task's folder, with the fixed prompt "Work on <task key>", for a task your clone's repository has. It can't send a command, a path, a prompt or a flag. On AgentMingo's side, only you, signed in as yourself, can press Start for your computer: not a teammate or an admin, and not an API token. Every start shows in your notices, the task's history and the organization's audit log, and a start nobody takes expires after 2 minutes. Someone who gets into your AgentMingo account, or controls AgentMingo's servers, could start threads on your tasks on this Mac; with answering prompts from AgentMingo on, they could also answer those threads' prompts. Turn it off with `start-from-web off`, or on your Account page.

**What it can't answer for you.** A thread started this way may still ask in its Terminal window what only that Mac can answer, such as whether to trust a new folder. git runs without asking anything there, so a remote that needs a password or a passphrase fails the start, and says so.

## Answering prompts and replying from AgentMingo

One switch, on each computer, turns on two things: answering a thread's prompts from its page in AgentMingo (permissions for any tool, questions and plans), and replying from there to a thread that finished its turn and waits for you. It is off unless you turn it on, in one of two ways:

- **The setup command**, with its box "Answer prompts and reply to threads from AgentMingo" ticked, as it is by default. The command then carries `--approvals` and a code made with the box ticked, and turns it on wherever it runs: a terminal, your editor, or a coding agent you give it to. Nothing else to run. See [The box](#the-box-answer-prompts-and-reply-to-threads-from-agentmingo) for when it stays off.
- **`approvals on`**, in a terminal of your own:

```sh
node ~/.agentmingo/reporter.mjs approvals on                 # every prompt, an hour's wait
node ~/.agentmingo/reporter.mjs approvals on --wait 2h       # 5 minutes to 8 hours: 30m, 2h, 1h30m
node ~/.agentmingo/reporter.mjs approvals status
node ~/.agentmingo/reporter.mjs approvals off
```

`approvals on` prints what it turns on (replies included), its limits and who could answer, and turns it on only when you type `yes`. It refuses inside Claude Code's Bash tool (it checks Claude Code's environment variables, `CLAUDE_CODE_CHILD_SESSION`, or `CLAUDECODE` without a terminal) and when it has no terminal to ask you in; from version 16 its refusal also says that a new setup command with the box ticked turns it on wherever it runs. `--unchecked`, from version 14, and `--commands`, from versions 14 to 16, are still accepted and change nothing: from 17, every prompt is shared once it's on, and there is no setting to share less. `approvals on` keeps these checks because it has no one-time code to stand for your consent, as the setup command has. They guard against a mistake or your agent turning it on this way; they aren't a wall against other programs that run as you, which can edit `~/.agentmingo/config.json` and `~/.claude/settings.json` like any of your files. `approvals off` and `approvals status` work from anywhere. `status` also says whether it's on, and when it's off, how to turn it on (a new setup command with the box ticked, or `approvals on` in your own terminal). The command itself sends nothing to AgentMingo.

### Prompts

Once it's on, in threads started from then on (open threads keep the hook they started with), every prompt Claude Code hands its `PermissionRequest` hook in a thread's main conversation can be answered from the thread's page, with the answers the terminal offers. Versions 14 to 16 opened only permission prompts (not questions or plans), offered Allow only for a file inside the repository that couldn't run code or change settings by itself, and let everything else only be denied. From 17 nothing is deny-only.

Claude Code's prompt stays on screen as always. The hook also opens the prompt on the thread's page in AgentMingo and checks for your answer there. Whichever is answered first counts. When you answer in the terminal first, the hook sees the tool call's result in the thread's transcript and stops waiting.

What each kind of prompt shows on the page, masked on your computer (`permissionTarget()`):

| Prompt | What it shows | Limits |
| --- | --- | --- |
| **A file change** (Edit, MultiEdit, Write, NotebookEdit) | Its path, from the repository's top for a file inside it, or as given for one outside it (from `~` in your home folder), with how many lines it adds and removes. Never the change itself. | |
| **A command** (Bash) | The command and its description | 20 lines and 1,000 characters of the command; past that, the rest shows only in the terminal, and the card says it was cut. The description on one line, 300 characters at most: a longer one ends in "…" |
| **An MCP tool** | Its server and tool (`mcp__tracker__create_ticket` is tracker, create_ticket), and its input | The input on one line, 300 characters at most: the one field that says what it does (a URL, a query, a path) when it's the only one filled, or else every filled field as compact JSON, so nothing it carries hides behind one field |
| **Any other tool** | Its name and its input | As for an MCP tool |
| **A question** (AskUserQuestion) | Each question with its header, whether it takes several answers, and its options with their descriptions | 4 questions, 10 options each; 60 characters of a header, 500 of a question, 200 of an option's label, 500 of its description (longer text ends in "…"). More than 4 questions: the card says it was cut. Options past the tenth aren't shown, and the card doesn't say so: they can be picked only in the terminal |
| **A plan to approve** (ExitPlanMode) | The plan | 20,000 characters |

**Badges** say what a prompt is before you answer (the `BADGES` constant): a command; an MCP tool; outside the repository; Claude Code's or another agent's settings (`.claude/`, `.mcp.json`, `CLAUDE.md`, `AGENTS.md`, and in your home folder `~/.claude.json`); a file that can run code or change settings by itself (package.json, CI workflows, .env files, git, editor settings, task runners, test and build set-up, the folder git runs hooks from, and in your home folder a shell's startup files, `~/.gitconfig`, `~/.ssh/`, login agents, `~/.local/bin` and `~/bin`: `AGENT_FOLDERS`, `AGENT_FILES`, `PROTECTED_FOLDERS`, `PROTECTED_FILES`, `protectedKind()` and `homeKind()` have the full lists); characters that don't show as they are (written out); partly masked; a path that doesn't lead straight to its file (`.` or `..` in it, a hard link, or a link that leads nowhere yet or out of the repository); shown cut. Before 17, most of these meant Allow wasn't offered. Now they are warnings, and Allow is offered for all of them, as the terminal offers it.

**The answers:**

- **Yes** applies to that one call. A deny rule in your Claude Code settings still wins over it. Allowing a file change can still lead to running code later: tests and builds run what's in it.
- **Yes, and don't ask again** applies Claude Code's own suggestions for that prompt (the hook's `permission_suggestions`), for the rest of that session only: the hook hands them back as `updatedPermissions` with the destination `session`, whatever destination Claude Code suggested, so nothing is written to a settings file. Only allow rules, extra folders, and switching the session to one of four modes (`acceptEdits`, `default`, `plan` or `auto`; never `bypassPermissions` or `dontAsk`) are kept; anything else Claude Code suggests is left out. It is offered only when the page can list every rule and folder exactly as it's applied: not when there are more than 10 entries in all, a rule over 200 characters, or a rule or folder that masking changed or that has characters that don't show (`sessionSuggestions()`).
- **No** denies it. A note you add reaches the agent as your quoted words (masked, one line, 300 characters at most); you can also ask it to stop the turn.
- **A question** is answered by picking its options (one, or several where it takes several) or typing under "Other" (one line, 1,000 characters at most). Your answers reach the agent as the question tool's input (`updatedInput` with `answers`, `answeredInput()`): for each question, the labels you picked and what you typed, joined with ", ". Every question needs an answer. No tells the agent you chose not to answer it.
- **A plan** is approved with its own input, unchanged, and, with "don't ask again", with Claude Code's own suggestions for it (switching to accept edits, say), for this session, when it made any. No tells the agent to keep planning.
- The terminal says what happened, for example "Allowed once from AgentMingo by you at 14:05", "Allowed from AgentMingo by you at 14:05, and not to ask again this session for Bash(npm test:*)", "Answered from AgentMingo by you at 14:05", "Plan approved from AgentMingo by you at 14:05", or "Denied from AgentMingo by you at 14:05" (`decisionOutput()`).

Before it applies an answer, the script checks it on your computer (`applyAnswer()`): it must carry the HMAC made for that prompt with the key in your `config.json`, and be at most two minutes old when it arrives. For any Yes, the page must have shown exactly what this computer sent (the hash covers the path, command or input, the badges, the question or plan, and what "don't ask again" would allow), the prompt must still show the same when the script looks again (a file that turned into a link meanwhile changes it), and it must not have been answered in the terminal meanwhile. "Don't ask again" applies only the suggestions Claude Code made for this prompt, as this computer reads them from the hook's input, never a list sent by the page, and only when there are some to apply. A question's answers must fit its questions. Otherwise nothing is applied and the terminal prompt stays.

Its limits, plainly:

- **Anything can be allowed from AgentMingo.** Commands that run on your computer, MCP and other tools, file edits outside the repository and in Claude Code's own settings, questions and plans. Whoever is signed in to AgentMingo as you can allow what your agent asks to run here: you, someone who gets into your account, or an agent with a browser signed in as you. Turn it on only if that's acceptable to you, and off when you don't need it.
- **The guard is the on-screen check, not a list of tested versions.** Claude Code doesn't document that a hook can wait while its prompt stays usable in a terminal, so a release of it could change that. In a terminal, the hook waits only once Claude Code says the prompt is on screen; when it doesn't say so within 20 seconds, the hook stops waiting at once and leaves the prompt to the terminal alone. That applies to questions and plans too: in a terminal, they wait only when Claude Code sends that notice for them. Version 14, the first with answering prompts, waited only on Claude Code versions checked with it, and listed none (so prompts were only reported unless you chose `--unchecked`); from 15 the list is of versions known to misbehave (`KNOWN_BAD_TERMINAL_VERSIONS` and `KNOWN_BAD_APP_VERSIONS`), on which a prompt is only reported. It is empty: none is known so far, and a release adds one when one is found. If a terminal prompt ever stops taking your keys while it waits, run `approvals off` and tell us.
- In Claude Code's VS Code extension and desktop app, which send no on-screen notice, Claude Code shows its prompt and runs the hook at the same time, and takes whichever answers first. The on-screen check doesn't apply there.
- It doesn't wait for a subagent's prompt, which still shows only in the terminal: the thread's page shows the thread waiting, and the `needs_you` event says the prompt is from a subagent. It doesn't wait in a headless run (`claude -p`), in a thread whose repository doesn't report, or while AgentMingo isn't answering. Dialogs Claude Code doesn't hand to its `PermissionRequest` hook never reach the reporter at all; the Security page names sandbox network prompts, folder-trust and MCP-server dialogs.
- An answer arrives only while your computer is awake and online and Claude Code is running. The hook stops waiting when its time is up or Claude Code is gone.
- Masking is best effort. A command, a tool's input, a question or a plan can carry a secret in a shape no rule knows, and it would then show on the page.
- What the page shows is cut at the limits above. A command past 20 lines or 1,000 characters, a tool's input past 300 characters, or a plan past 20,000 shows in full only in the terminal; the card says it was cut, but Yes allows the whole call.
- Between the script checking the prompt once more and Claude Code running it, another program already running on your computer could change what it points at (swap a file for a link elsewhere). Allow trusts what already runs on your computer.
- Someone who controls AgentMingo's servers or database could answer for you, or show you a different file or command than the one that changes or runs. And an update of this script can change anything it does, as it always could (`auto-update off` stops updates).
- On AgentMingo's side, the server enforces who can answer and how, which this script can't show you: only you, the thread's own person, signed in as yourself (not teammates or admins), and only through the page, never with an API token. From 17 a signed-in session is enough; the earlier rule that allowing needed a sign-in in that browser within the last day is gone. Each answer is tied to one prompt and used once, and recorded in the organization's audit log (without what the prompt asked) and in the thread's history, shown to you. For 30 days after your computer last opened a prompt there, each new sign-in to your account is emailed to you. [AgentMingo's Security page](https://agentmingo.com/security#approvals) describes all of it.

### Replies

Replies go with answering prompts: a computer where you turned answering prompts on with version 14 takes replies too once it updates to 15, without asking again. `approvals off` turns both off.

With it on, when a thread finishes its turn and waits for you, its page in AgentMingo can send it a reply: Continue, Yes, No, or a text of your own (up to 1,000 characters). After each reply the thread makes, its `watch` hook listens up to 8 hours for one (see [The watcher and its timeout](#the-watcher-and-its-timeout)). When you send one:

1. The watcher sees it the next time it asks, at its pace: every 30 seconds at first, every 5 minutes after the first half hour.
2. It claims the reply on this computer (an empty file in `state/<session id>.told/`), then asks AgentMingo to hand it over. AgentMingo hands each reply over once, so two watchers can't both take it.
3. It wakes the thread with "<your first name> replied from AgentMingo: …" and a line saying this is your reply, sent from the thread's page instead of typed there, and that the agent should go on as if you had typed it. The workflow step and the review results that were waiting for your next message come with it, as they would have with a typed message.

A reply belongs to the turn it answers. Typing in the thread first, the thread ending, or the thread going on without it (woken for review results or a workflow step) cancels it: it isn't delivered later. The watcher stops within 5 seconds of you typing in the thread or the thread ending, and the thread's next reply starts a watcher of its own, which takes over.

What it sends for replies: each report says whether replies are on here, each watcher request says in how many seconds it asks next, so the page offers a reply only while something listens, and the request that takes a reply carries that reply's id. The reply's text comes from AgentMingo to this computer. Turning replies on adds nothing else to what is sent.

Its limits, plainly:

- **It reaches the thread only while the watcher listens.** That is up to 8 hours after the thread's last reply (from version 18, while its work is open and your computer stays on and awake, a week at most: [Keep listening](#keep-listening)), while your computer is awake and online and the thread is open, and never past its hook's timeout. A computer still on the 20-minute timeout from an earlier install listens 19 minutes at most, until the update raises it; a thread opened before that keeps its 20 minutes until you restart it. When nothing listens, the page says so, and you type in the thread instead.
- **A reply directs your agent as your own words.** The agent acts on it as if you had typed it. Someone who gets into your AgentMingo account could send your threads a reply, and so could an agent with a browser signed in to AgentMingo as you. A command the thread then wants to run still stops at a permission prompt (unless your Claude Code settings already allow it), but from 17 that prompt can be allowed from AgentMingo too, so whoever can reply can also allow what the thread then asks to run. Before 17, AgentMingo could only deny a command, and allowing one needed your terminal.
- **The reply's text isn't masked** on its way into the thread. It becomes part of the conversation, which your agent sends to its model provider under your own account and plan.
- Who can send a reply (only you, the thread's own person, signed in as yourself, from its page and not with an API token; the rule that also needed a sign-in in that browser within the last day was removed with 17), the 1,000-character limit, one reply waiting per thread, its expiry (2 hours), that it is taken once, rate limits, and the audit log entry are enforced by AgentMingo's server, which this script can't show you. [AgentMingo's Security page](https://agentmingo.com/security#approvals) describes them. On this computer, the script cuts a reply at 4,000 characters and writes out characters that don't show.
- Someone who controls AgentMingo's servers or database could send your threads a reply.
- A watcher already listening when you run `approvals off` read the setting when it started and doesn't look again: until it stops (you type in the thread, it ends, or its time is up), a reply sent from the page may still reach that thread. Watchers that start afterwards don't take replies.

### Turning it off

`approvals off` takes the key and the choice out of `config.json` and the `--wait` out of the hook, and so turns replies off too. Threads open at that moment keep the hook they started with until they end, but their next prompts open no request, since the key is gone. A prompt already waiting at that moment keeps waiting until it's answered or its time is up.

The code to read: `approvalsCommand()`, `tradeSetupCode()`, `approvalsFromSetup()`, `approvalsStayedOffSaid()`, `installConfig()`, `permissionTarget()`, `fileTarget()`, `protectedKind()`, `homeKind()`, `inputSummary()`, `questionsOf()`, `sessionSuggestions()`, `promptHost()`, `awaitDecision()`, `applyAnswer()`, `answeredInput()` and `decisionOutput()` for prompts; `runWatch()`, `watchOver()`, `replyOf()`, `takeReply()` and `replyNote()` for replies.

## Review results

With review results on (`--reviews`, or `reviews on`):

> when a review bot (CodeRabbit, say) or CI's check runs report on the pull request for a session's branch, AgentMingo decides what blocks and what to fix now, and the session gets that list with its next message (asked for just before the message reaches Claude, a second at most, so results that came in after the watcher stopped aren't a message late), or, if it's idle after pushing, is woken with it: a background hook waits up to 18 minutes after each reply for review results, asking every 30 seconds at first, then every minute, then every two; 3 automatic rounds per pull request by default. [...] A merge conflict on that pull request, and each step of the repository's pull request checklist, wake it too, outside those rounds; past them, results wait for you to say continue (your next message, or a reply from the thread's page). [...] `reviews off` stops all of that (a session that follows a workflow is still woken for its next step).

After the 3 automatic rounds, results are held: the watcher doesn't wake the thread with them. They come with your next message, or, with replies on, with a reply you send from the thread's page ("Continue" is enough). In versions before 15, a watcher stopped once the rounds were used up; from 15, one that listens up to 8 hours goes on listening, for a reply or a workflow step.

To wake a thread, the `watch` hook exits with code 2 and writes the note to stderr. That is how a background hook wakes a thread (`asyncRewake`). Every other hook exits 0. The woken turn runs on your own plan with your agent's provider, like any other turn.

## The watcher and its timeout

The `watch` hook runs in the background after each reply a thread makes, when there is something to wait for: review results on, a workflow, or replies on. In the script's own words:

> In a session that follows a workflow, and with replies on (above) in any session, it listens up to 8 hours instead (every five minutes after the first half hour, and once more as its time runs out), so a plan approved or a reply sent hours later still wakes it (and with keep listening, above, for as long as its work is open, a week at most). It stops within seconds of you typing in the thread or the thread ending (it looks at the thread's state on this computer every 5 seconds, without asking AgentMingo), and the thread's next reply starts a watcher of its own, which takes over from it. Claude Code stops the hook at its timeout: install gives it 8 hours and 10 minutes, and this version raises an earlier install's 20 minutes itself, once, changing only that number in your settings (`status` says whether it did; if it couldn't, run the setup command again). Sessions already open keep the timeout they started with until you restart them (Claude Code may say their hooks changed), and the watcher listens within it.

The exact rules:

- **How long it listens.** 18 minutes (`WATCH_FOR_MS`) for review results alone; 8 hours (`WATCH_LONG_MS`) in a thread that follows a workflow, or in any thread with replies on. A thread that starts following a workflow while the watcher waits switches it to 8 hours.
- **How often it asks.** Every 30 seconds for 3 minutes, every minute until 10, every 2 minutes until 30, then every 5 minutes (`watchEveryMs()`): about 17 requests in 18 minutes, about 110 in 8 hours. From version 18, once more when its time is up (with replies on, saying `listen: 0`: it stops listening now). While AgentMingo isn't answering, it asks nothing and waits.
- **When it stops early.** Within 5 seconds (`WATCH_CHECK_MS`) of you typing in the thread, the thread ending, a newer watcher taking the thread over, or the connection being refused, all looked at on this computer without a request (`watchOver()`). Also when AgentMingo says the computer is disconnected or refused, and, without replies or a workflow, when the branch has no pull request or the automatic rounds are used up.
- **The timeout.** Claude Code stops the hook at the timeout in your settings. Install gives it 29400 seconds (`WATCH_TIMEOUT_S`, 8 hours and 10 minutes). The watcher stops by itself a minute before the timeout its thread has: the one in your settings now, or the one the thread started with, whichever is lower (`sessionWatchTimeoutS()`).
- **The raise.** Installs before 15 gave the hook 1200 seconds. Once a computer runs version 15, its background sender, after it sends, raises that number to 29400 (`raiseWatchTimeout()`). It does so only from the copy install put in `~/.agentmingo`, only on the `watch` hook that copy added, only when its timeout is lower, and changes nothing else in `~/.claude/settings.json` (it writes the file with the same code install uses). Once it has checked, `config.json` notes it, and it doesn't look again, so a timeout you lower afterwards stays as you set it. When it can't write the settings (a link to a read-only file, say), it changes nothing, logs why in `reporter.log`, and tries again after a later send; `status` says to run the setup command again, which writes the new timeout.
- **Threads already open.** Claude Code may keep the hooks a thread started with, so a thread open during the raise keeps its 20 minutes until you restart it, and Claude Code may say its hooks changed. The script records the timeout each thread started with, and a thread that started before version 15 is taken to have the old one.

`status` and install say how long the watcher listens, from the timeout in your settings, and whether an update raised it.

### Keep listening

From version 18, with replies on, a thread that waits for you on work still open stays reachable from its page past one watcher's 8 hours. The header says it in full (quoted in [What it sends](#what-it-sends)). The exact rules:

- **Who decides.** AgentMingo, in its answers to a watch's last two requests (the time left, and `listen: 0` as its time is up): `keepListening` is true only while the thread waits for you, its computer takes replies, your setting "Keep threads listening while their work is open" is on (AgentMingo's Notifications page; on by default), the thread is tied to work still open (its task isn't done or canceled, an open or draft pull request on its branch, or a workflow run that's going), and its wait began less than a week ago; its operator can also turn it off for everyone. Each watch's end decides again, so once the work closes the next one says no. The watcher goes by the latest answer that says (so a last request lost as the computer wakes goes by the time left's, minutes old); without one (an older AgentMingo), it's no. The answer is only true or false: nothing AgentMingo sends goes into the note.
- **When it wakes the thread.** Only when a long watch with replies on listened its whole time (`runWatch()`), on a watch hook that gives it the whole 8 hours (`KEEP_LISTENING_MIN_CAP_MS`, so about every 8 hours and never more often; a hook whose timeout the script can't find doesn't count), with replies still on in `config.json` as it ends, and fewer than 21 times in this wait (`KEEP_LISTENING_REARMS`, counted in `state/<session id>.keep.json` by `rearmToKeepListening()`). Never when you typed in the thread, it ended, a newer watcher took over, AgentMingo refused this computer (401 or 403), or for Cursor's own agent. A compaction or `/clear` keeps the lower watch-hook timeout the thread already had (`startedWatchTimeoutS()`): Claude Code keeps the hooks it started with.
- **Sleep.** The watcher's pauses are awake time; once the clock says its time is up (the computer slept through it), it makes its last request at its next 5-second step, so a computer that wakes within the hook's last 10 minutes still keeps the thread listening. One that wakes later does too, unless Claude Code stopped the hook while it slept (whether its timer counts sleep isn't something the script can know); then nothing listens until you type in the thread. A restart, Claude Code closed, or a short turn that fails (a usage limit, say) always ends it that way.
- **What it wakes it with.** `KEEP_LISTENING_NOTE`: "AgentMingo: keeping this thread reachable from its page (your person may reply there). Nothing has changed and nothing is needed. Reply with only: Still listening for AgentMingo. Don't run any tools or continue earlier work. If your person has written to you since, ignore this note and do what they asked." Like the other notes, it becomes part of the conversation. Its last sentence is for a note that lands just as you type in the thread.
- **That turn's Stop.** `noteKeepListening()` marks it, and its report carries `keepAlive: true`, so AgentMingo starts no new turn for it (a reply you send before it, or during it, still answers the same wait: the page keeps its reply box open through it, and the fresh watcher picks the reply up) and tells you nothing. You typing in the thread, a real turn ending, or the thread starting again or ending starts the count again.
- **What it costs.** Each short turn re-reads the whole conversation on your own Claude plan, like any other turn, about every 8 hours (up to 3 a day). Eight hours on, the conversation is no longer cached, so most of it is read in again: the longer the conversation, the more each one uses. Its tokens are counted in the thread's usage, and the thread's page lists those turns with their tokens.

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
- An update keeps your choices, answering prompts from AgentMingo included: the `PermissionRequest` hook keeps its wait when the hooks are pointed at a new layout. An update doesn't turn answering prompts on where it was off. Where it was on, an update to 15 adds replies, which go with it, and an update to 17 shares every prompt from then on (commands, MCP and other tools' input, questions, plans, and files outside the repository), and lets each be allowed from AgentMingo, without asking again: version 17 doesn't read the earlier `--commands` choice. If you turned it on for file edits only and don't want that, run `approvals off`.
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
node ~/.agentmingo/reporter.mjs approvals off     # no prompts or replies on AgentMingo
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
