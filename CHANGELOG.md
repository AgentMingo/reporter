# Changelog

Each version is the `VERSION` constant in `reporter.mjs`. Reporters update themselves to the version agentmingo.com serves, unless their person turned that off.

From version 11 on, each entry gives the SHA-256 of `reporter.mjs` as agentmingo.com served it, and the date it started serving it. This repository starts at version 12; each release is published here by hand for now. Versions 1 to 11 came before this repository and their files aren't published here. The entries for 1 to 10 are summaries of what changed, from the script's history in AgentMingo's own repository, and their dates are when each version was written, not when it was served.

## 14 (served from 2026-10-30)

SHA-256 of `reporter.mjs`: `a2cf1630604fdadf9a6b175b3b8ae8499d1eb523394b8331ecea1fe31dea552d`

- New `approvals on|off|status` command, to answer permission prompts from AgentMingo. It's off unless you turn it on, on each computer: `approvals on` runs only in a terminal of your own (it refuses inside a Claude Code thread's own commands, and without a terminal), says what it turns on and who could answer, and asks you to type yes. `--commands` also shows commands, masked, so you can deny them; `--wait` sets how long a prompt waits (5 minutes to 8 hours, an hour by default); `--unchecked` waits on Claude Code versions not checked with it. `approvals off` and `status` work from anywhere, and the command sends nothing to AgentMingo. No install flag turns it on; install keeps it as it was, and its new `--no-approvals` turns it off.
- With it on, in sessions started since: the `PermissionRequest` hook's command gets `--wait <seconds>` and its timeout becomes a minute longer than that. When a session stops on a permission prompt, the terminal prompt stays, and the hook also opens it on the thread's page and asks for an answer every 3 seconds for two minutes, then every 10; whichever is answered first counts. A file change inside the repository can be allowed (that one call only) or denied from there. A command or any other tool can only be denied. Allow is never offered for a file outside the repository, one that can run code or change settings by itself, a path with `.` or `..` or hidden characters, a hard-linked file, or a path masking changed.
- What it sends for such a prompt: the thread's id, the tool's name and the kind of prompt, a random id, an HMAC that ties an answer to the prompt (its key is made on the computer, kept in `config.json` and never sent), a hash of what it shows, how long it waits, whether Allow can be offered (and why not), and for a file change the path in the repository, masked, with how many lines it adds and removes, never the change itself. With `--commands`, the command, masked. When it stops waiting, it says how (answered on the computer, time up, the prompt never shown, an answer refused or too old, or the hook stopped). An answer is applied only when it carries that HMAC, is at most two minutes old, and, for Allow, matches what this computer sent, is still allowed here, and the prompt wasn't answered in the terminal meanwhile. After one is applied, the thread reports it was answered.
- For now it waits on no Claude Code version unless you also chose `--unchecked`: it waits only on versions checked with it, and this release lists none, so prompts are only reported, as before. It never waits for a subagent's prompt, a headless run, a thread that doesn't report, a question or a plan to approve, or, in a terminal, a prompt Claude Code hasn't said is on screen within 20 seconds.
- New `start_task` MCP tool, for "Work on WEB-12" in a thread. It makes the task's folder as `start` does, with git unable to ask for anything, and its answer tells the agent to ask you before it moves the thread there with Claude Code's EnterWorktree, or to give you a `claude-cli://` link that opens a new Claude Code session in that folder. It doesn't move the thread itself. The MCP server's instructions mention it. Install adds a permission rule for it: five rules in all, three MCP tools.
- `start` run from inside a thread still starts no second Claude Code, and now prints the folder and the same guidance as `start_task` (before, only the folder). `--no-launch`, which only makes the folder, is now in the usage.
- `start` copies only .env, .env.local, .env.development, .env.development.local, .env.test and .env.test.local into a task's folder, says which it left out, and warns that a copied .env.local can hold tokens. `--env-all` copies every one, and is refused inside a thread. Versions before 14 copied every .env file git ignores.
- A thread that moves into another repository or worktree mid-session reports it at once, and asks for the brief of the task its new branch names with its next message.
- Before each message, a thread that has a task, or whose branch names one, also asks for its workflow step, so a workflow started from its page begins with that message.
- Each report says how many files have changes not yet committed: a count, never their names or what changed.
- Masks more, on the computer: a user and password given to curl or wget (`-u`, `--user`), a password given with `-p` to sshpass or a registry login (docker, podman, helm and the like) and with `-k` or `-pass` to openssl, a secret-looking value after `--password`, `--passphrase`, `--passwd` or `--client-secret` with a space, cookies, X- headers carrying a token, key, secret, auth or signature, chat webhook addresses (Slack, Discord, Microsoft Teams, Power Automate) and a signed URL's `sig=`, and a value of any length given to a variable named for a password or secret (`PGPASSWORD=…`, `DB_PASS=…`). Masking stays best effort.
- New state files, removed when a thread ends: `state/<session id>.asked.json` (the branch last asked about for a brief) and `state/<session id>.prompt.json` (when Claude Code said a permission prompt was on screen). `status` and install say whether answering prompts is on.
- The header says all of the above: the uncommitted count, the new lookups, what answering prompts sends and its limits, what `approvals` changes in the settings, the start_task tool, the .env files copied, and five permission rules with three MCP tools.

## 13 (served from 2026-10-29)

SHA-256 of `reporter.mjs`: `197e13239f78a91f70b10653fd827d3de2614a8c788303dca5fb8f803199c2b9`

- Wording only: what `install` and `reviews on` print now says "review bots' and CI's results" instead of naming one review bot. Nothing it sends, reads or writes changed.

## 12 (served from 2026-10-29)

SHA-256 of `reporter.mjs`: `2e1d4d91f79701d0916775d09cc999420dd2fbd5dfc2dbd16854e60334b2cce4`

- The header lists everything the script sends, field by field: each moment with its time, id and the thread's id; the repository, branch, whether it's pushed and how many commits aren't; token counts by model; tool calls by kind with their output and time; git commands by subcommand; the title and a waiting thread's notice; its settings; what it passed on; the lookups it makes; and on a workflow, a step's result, text, attached files and the current-step lookup. It's the same list as AgentMingo's Privacy Policy. Version 11 sent all of it too; the one new request is the lookup below.
- With review results on, the hook that runs before each message asks AgentMingo, for a second at most, what's waiting for the thread (it sends the thread's id). Review results that became ready after the watcher stopped now arrive with that message, not the one after it.
- Each review result or notice is told once. Whichever of the message hook and the watcher claims it first tells it, with an empty file per result in `~/.agentmingo/state/<session id>.told/`. A thread's claims are removed when it ends; ones left by a thread that never ended are removed, when a later thread starts, once untouched for 7 days.
- A lookup before a message that gets no answer pauses those lookups for 60 seconds, so a slow AgentMingo holds up one message by a second, not every message.
- The notes added in one hook call are kept within 10,000 characters, as much as Claude Code takes from a hook. Review results that don't fit wait for the next message; a single one too long to fit is cut, and says so.
- The header says the watcher waits up to 18 minutes after each reply (as in 11; 20 minutes is the limit Claude Code puts on it), that it exits with code 2 when it wakes a thread, and that `--lean-git` changes `~/.claude/CLAUDE.md`.
- The header and comments no longer name files in AgentMingo's private server.

## 11 (served from 2026-10-29)

SHA-256 of `reporter.mjs`: `2409e08359ef9a0e01a1551bd3a0027d27306bebf8cadf1b2bc899ac45f8a99c`

- Ignores Cursor's own agent. When a hook's input comes from Cursor (it has `cursor_version` or `conversation_id`), the script exits at once and reads, sends and writes nothing. Claude Code inside Cursor reports as before.
- Repositories renamed or moved on GitHub keep reporting under their earlier names. For a folder whose repository shares its owner or its name with one an organization lists, the computer sends that name to the organization once a day, to ask whether it's the same repository. AgentMingo asks GitHub, and keeps the name only if it is.
- With sharing off, the script says so in each report, and AgentMingo names your sessions in no teammate's notes.
- The team's own rules reach every new session, whatever the git habits are set to.
- When a thread's transcript stops giving token or tool counts, the script says which is missing, so `status` and AgentMingo's admins can see it.
- The review watcher asks less often as it waits: every 30 seconds for 3 minutes, then every minute until 10, then every two.
- `status` says where to look when something doesn't work. `start` works only in a connected or added repository.

## 10 (written 2026-10-28)

- Reports only threads in a repository the organization connected, or one its person added with `repos add`. Threads in other folders send nothing about themselves. (Before 10, every thread on the computer reported.)
- Safer updates: a new version is used only after it's checked on the computer, the version before is kept as `reporter.prev.mjs`, and hooks run through `launch.mjs` and `reporter.sh`, which fall back to it. AgentMingo can hold reporters at a version or send them back to one.
- The setup command carries a one-time code that works for 15 minutes, instead of the token.
- Masks more in titles and in a waiting thread's notice: tokens and keys by their known shapes, private keys, and credentials in addresses, headers and command lines.
- The header now lists what a waiting thread's notice sends. Every version before sent it too, with titles on (see 9).
- Backs off when AgentMingo doesn't answer, honours limits, and waits an hour or more after a 403.
- `uninstall` disconnects the computer. Install keeps earlier choices. New commands: `repos`, `titles on|off`, `auto-update on|off`.

## 9 (written 2026-10-28)

- Masks emails, phone numbers, long numbers and keys in titles on the computer, and in what a waiting thread's notice says. Versions 1 to 8 sent the first 200 characters of that notice unmasked, with titles on.
- Says in each report whether review results are on.

## 8 (written 2026-10-26)

- One computer can report to several organizations. Each thread goes to the organization that lists its repository.

## 7 (written 2026-10-26)

- Updates itself from AgentMingo (`--no-auto-update` turns that off).
- Passes on the team's own rules with the git habits.

## 6 (written 2026-10-26)

- `start <task key>` makes the task's branch in a folder of its own next to your clone and starts the agent (Claude Code) there.
- Steps can carry a plan's decisions and attached files.

## 5 (written 2026-10-26)

- Threads follow a workflow one step at a time. The agent reports each step with `step` or the agentmingo MCP server, which install registers.

## 4 (written 2026-10-25)

- A branch named after a task (`WEB-12-login`) ties the thread to that task, and the thread gets the task's brief.

## 3 (written 2026-10-25)

- Git habits as a note for new threads, when the organization turns them on.

## 2 (written 2026-10-24)

- Tool calls counted by kind, and whether the branch is pushed.
- Review results: a review bot's (CodeRabbit, say) and CI's results on a thread's pull request come back to that thread (`--reviews`).
- With `--share-files`, the paths of files a thread edits, and a note about other threads in the same repository.
- `lean-git on|off|status`.

## 1 (written 2026-10-24)

- First version: a thread's state, repository and branch, title, and token counts, sent from Claude Code's hooks.
