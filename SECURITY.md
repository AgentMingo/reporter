# Security

## Reporting a vulnerability

Email **hello@agentmingo.com** with what you found and how to reproduce it. We'll confirm we got it and keep you posted until it's fixed. English is preferred.

The same address is in <https://agentmingo.com/.well-known/security.txt>. The full policy is at <https://agentmingo.com/security>.

Please don't open a public issue or pull request for a vulnerability.

Please don't access or change other people's data, disrupt the service, or make the issue public before it's fixed. If you follow that and act in good faith, we won't take action against you for your research.

## What's in scope here

This repository holds the reporter that runs on people's computers: `reporter.mjs`, and the `launch.mjs` and `reporter.sh` it writes into `~/.agentmingo` (their text is in `reporter.mjs`: the `LAUNCHER` constant and `shimText()`). Reports about the AgentMingo service itself, which is closed source, go to the same address.

From version 14 that includes answering permission prompts from AgentMingo (`approvals`, off unless a person turns it on): how it's turned on, what a prompt sends, and the checks the reporter makes before it applies an answer, such as an answer applied to a prompt it wasn't made for, or a secret in a path or command that masking should have caught. (In 14 to 16, Allow offered for a file that can run code or change settings was in scope too; from 17 that is allowed on purpose, with a badge: see below.) It also includes the task folders `start` and the `start_task` tool make, and which `.env` files they copy. The page that shows a prompt and takes the answer is part of the service; report it to the same address.

From version 15 it also includes:

- install's `--approvals` (the setup command's box "Answer prompts and reply to threads from AgentMingo"). In 15 it turned answering prompts and replies on only with the one-time code, in a real terminal, and not inside Claude Code's Bash tool; version 16 changed that (below).
- replies from a thread's page: how the watcher takes a reply and wakes the thread with it, such as a reply handed over twice, to the wrong thread, or after the turn it answered (you typed in the thread, it ended, or it went on without it).
- the watcher and the update's change to `~/.claude/settings.json`: such as the timeout raise changing anything but this reporter's own `watch` hook timeout, or a watcher that keeps listening after you typed in the thread or it ended.

The page that sends a reply, and who may send one, are part of the service too; report them to the same address.

From version 16, install's `--approvals` turns answering prompts and replies on wherever the setup command runs (a terminal, an editor, or a coding agent's Bash tool), when the command's one-time code was made with the box ticked (`ama_`; unticked, `amc_`). In scope:

- a way to turn it on without a code made with the box ticked: with `--approvals` added to a command made with the box unticked, with the older `--token` form, with a code whose prefix was changed, or with an answer from the trade that install misreads;
- install failing, or changing anything, because of `--approvals`, or spending the one-time code when Claude Code's settings aren't valid JSON;
- a setup code (`ama_` or `amc_`) that masking should have caught in a title, a waiting thread's notice, a prompt's path or a command;
- a way to get past `approvals on`'s checks (a typed yes, in a real terminal, outside Claude Code's Bash tool), such as a coding agent or a script without a terminal turning it on with it. A program on the computer that sets out to fake a terminal is a known limit, not a finding.

Known limits, not findings: whoever runs a command made with the box ticked while its code is unused turns it on, you or the coding agent you gave it to, as they run everything else in the command; and a command pasted into a thread stays in that conversation, with a code that works until it's used or its 15 minutes are up. How a code is made, which box it records, that it works once for 15 minutes, and that only its hash is kept are part of the service; report them to the same address.

From version 17, with answering prompts on, every prompt in a thread's main conversation that Claude Code hands its `PermissionRequest` hook opens on AgentMingo (file edits anywhere, commands, MCP and other tools, questions and plans), and can be answered there with the terminal's answers: Yes, "Yes, and don't ask again" for the session, or No. In scope:

- an answer the reporter applies when it shouldn't: one not made for that prompt (its HMAC), older than two minutes, for a prompt that shows something other than what the page showed, or after the prompt was answered in the terminal;
- "don't ask again" applying anything but Claude Code's own suggestions for that prompt, as the computer read them; applying a mode other than `acceptEdits`, `default`, `plan` or `auto`; applying a rule or folder the page didn't list exactly as applied; or reaching a destination other than the session (being written to a settings file);
- a question's answers reaching the agent other than as picked or typed, or a plan approved with input other than its own;
- a badge missing where it should be, such as a path outside the repository, a file in Claude Code's or another agent's settings, or one that can run code by itself not marked so, or a path that doesn't lead straight to its file not marked `indirect`;
- something a prompt carries that the page doesn't show and doesn't say was cut, such as a field of a tool's input left out of its one-line summary;
- a secret that masking should have caught in a command, its description, a tool's input, a question or a plan;
- `--commands`, or anything left in `config.json` by an earlier version, changing what 17 shares, or a computer turning it on without the setup command's code made with the box ticked or `approvals on`.

Known limits, not findings: anyone signed in to AgentMingo as you (you, someone who gets into your account, or an agent with a browser signed in as you) can allow what your agent asks to run on your computer, including commands and changes to Claude Code's settings, since a signed-in session is enough from 17; an update to 17 on a computer where answering prompts was on for file edits only shares every prompt without asking again; what the page shows is cut at its limits (a command at 20 lines or 1,000 characters, a tool's input at 300, a plan at 20,000) while Yes allows the whole call, with a badge that says it was cut; masking is best effort; a subagent's prompt, and dialogs Claude Code doesn't hand to its `PermissionRequest` hook, are answered in the terminal only; in a terminal, a prompt (a question or a plan included) waits only once Claude Code says it's on screen within 20 seconds. Who can answer, that answers go only through the page, the audit log entry and the thread's history, and emails about new sign-ins are part of the service; report them to the same address.

From version 19, on a Mac with starting threads from AgentMingo on (`start-from-web`, off unless a person turns it on), a LaunchAgent helper asks AgentMingo for starts and opens Terminal to run Claude Code on a task. In scope:

- a way to make the helper run anything but `run-start <id>` for one start, or make `run-start` run anything but Claude Code in a task's folder with the prompt "Work on <task key>" (plus `--add-dir` for the task's other clones): a command, a flag, a path or prompt text from AgentMingo reaching the `.command` file, git or Claude Code's command line;
- a value from AgentMingo reaching git or a path unchecked (`checkedStart()`), such as a branch read as an option, or a folder outside the clone's parent folder; a task title reaching the terminal with control characters (`terminalSafe()`);
- a start opened twice, from another organization's connection than the one it came from, past the limits on this computer (6 an hour, 15 seconds apart), or after starting threads was turned off on this computer;
- a way to turn it on without a setup code made with its box ticked or `start-from-web on`'s typed yes in a real terminal outside Claude Code's Bash tool, such as `--start-from-web` added to a command made unticked, or the agent running `start-from-web on`;
- the LaunchAgent: the reporter writing over or removing a file at its path that it didn't write, the helper getting an environment variable other than its PATH, or it staying loaded after `start-from-web off` or `uninstall`;
- something sent that the README doesn't list, such as the computer's name, a path outside the home folder's `~` form, or the token in a start's files.

Known limits, not findings: anyone signed in to AgentMingo as you, or anyone who controls AgentMingo's servers, can start a thread on your tasks on that Mac, and with answering prompts from AgentMingo on, answer its prompts; a thread started this way may wait in its Terminal window on what only the Mac can answer (trusting a new folder); the helper runs only while you're logged in and the Mac is awake. Who can press Start (only the computer's own person, signed in), the 2-minute expiry, the per-person rate limit, and the Account page's Turn off are part of the service; report them to the same address.

## Supported versions

Only the latest version gets fixes. Reporters update themselves unless their person turned that off (`auto-update off`), and AgentMingo can send reporters back to an earlier version (10 or newer) while a fix is made. Updates come from agentmingo.com over HTTPS and aren't signed; the README says what the script checks before it uses one.
