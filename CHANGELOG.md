# Changelog

Each version is the `VERSION` constant in `reporter.mjs`. Reporters update themselves to the version agentmingo.com serves, unless their person turned that off.

From version 11 on, each entry gives the SHA-256 of `reporter.mjs` as agentmingo.com served it, and the date it started serving it. This repository starts at version 12; each release is published here by hand for now. Versions 1 to 11 came before this repository and their files aren't published here. The entries for 1 to 10 are summaries of what changed, from the script's history in AgentMingo's own repository, and their dates are when each version was written, not when it was served.

## 19 (served from 2026-10-31)

SHA-256 of `reporter.mjs`: `1473c60220796e17e49f8bd32f19a2f908678d5ecd6f1eb1206455206a01eb55`

- Starting threads from AgentMingo (macOS only for now; off unless you turn it on). With it on, "Start on my computer" on a task's page, from any device you're signed in on, starts a Claude Code thread on that task on this Mac. Turn it on with the setup command's new box "Start threads on this computer from AgentMingo (Mac only)", unticked by default. Ticked, the command carries `--start-from-web`, which turns it on only with a one-time code made with the box ticked. Unticked, the command carries no flag for it, so the computer keeps what it had: one setting covers every organization on the computer, and a command made unticked in another one mustn't turn it off. Or turn it on with `start-from-web on` in your own terminal, which asks you to type yes and refuses inside Claude Code's Bash tool. Turn it off with `start-from-web off`, `uninstall`, or on your Account page in AgentMingo, one organization at a time.
- The helper is a LaunchAgent, `~/Library/LaunchAgents/com.agentmingo.start-from-web.plist`. It runs `reporter.sh start-from-web helper` while you're logged in. It gets only the system PATH and no other environment variable, and launchd restarts it a minute after a failure but not after it stops because starting threads is off. The file starts with a marker line: the reporter never writes over or removes a LaunchAgent without it.
- About every 10 seconds while the Mac is awake, the helper asks AgentMingo whether you pressed Start (`POST /api/v1/agent-sessions/starts/check`). It sends only that it's running and whether answering prompts from AgentMingo is on here. AgentMingo answers with at most 3 starts, each a start's id and a task key, nothing else.
- For each start, the helper checks the id (`START_ID`) and the key (`TASK_KEY`), and takes the start (`starts/claim`, a compare-and-set, so it runs once). It then writes `~/.agentmingo/starts/<id>.command` (mode 0700, in a 0700 folder) holding only `exec <reporter.sh> run-start <id> --home <folder>`, and opens it with `open -g -a Terminal` (no shell, no AppleScript). It opens at most 6 starts an hour, none within 15 seconds of the last, whatever AgentMingo sends.
- In that window, `run-start <id>` makes the task's folder the way `start` does, from that start's organization only, with the same safe `.env` files. It then runs `claude "Work on <task key>"` there (plus `--add-dir` for other repositories the task names), as you'd start it yourself: no permission flag, and no settings or MCP config of AgentMingo's. Claude Code's own permission settings apply. `AGENTMINGO_START=<id>` in Claude Code's environment lets the thread's first report (SessionStart's new `start` field) say which start it came from.
- AgentMingo hears how each start went (`starts/result`): opened, started, or failed with a reason from a fixed list. A failure carries one line of detail (300 characters at most, home folder shown as `~`) and the repository's name.
- Checks on AgentMingo's answers before they reach git, now also for `start` and the `start_task` MCP tool (`checkedStart()`). The task key must be the one asked for. The branch must be the key, or the key and words of its title. The repository's names must be `owner/name`, and its main branch must pass `git check-ref-format --branch`, with no leading `-` and no `..`. Before 19, the branch and main branch went into `git worktree add` as AgentMingo sent them.
- Task titles are printed without control characters, DEL or bidirectional overrides (`terminalSafe()`). Before 19, an escape sequence in a title reached the terminal.
- What it sends that's new: every request's user agent now says the platform (`agentmingo-reporter/19 (darwin)`), so AgentMingo offers the button only for a Mac. The `start` field on a SessionStart event started this way. The helper's check, claim and result requests, only while starting threads from AgentMingo is on.
- `reporter.sh`: when it finds no Node.js, the helper exits 1 (so launchd tries again a minute later), while hooks still exit 0. Turning the helper on, or an update, rewrites an older `reporter.sh`, and after a hook points it at a Node.js that's there, the reporter restarts a stopped helper (`launchctl kickstart`).
- New files: the LaunchAgent above, `~/.agentmingo/starts/` (a start's files, removed once it runs or after a day), `state/starts.json`, `state/start-helper.pid`, and `helper.log` (what launchd catches, emptied past 1 MB).
- New commands: `start-from-web on | off | status`, `start-from-web helper [--once]` (one round by hand, printing each step), and `run-start <id> [--no-launch]`. Install's `--start-from-web` / `--no-start-from-web`. Install keeps the choice from before when neither is given. `status` says whether it's on, whether the helper is loaded, and when it last checked.
- The permission rules install adds are unchanged (`Bash(<reporter> step:*)`, `Bash(<reporter> start:*)` and the MCP tools). Claude Code matches `start:*` only when a space follows `start`, so `start-from-web` and `run-start` still ask.
- On AgentMingo's side, not in this file: only the computer's own person can press Start, signed in as themselves (not teammates or admins). A start nobody takes expires after 2 minutes. Every start shows in that person's notices, the task's history and the organization's audit log. Start records are deleted after 30 days. See <https://agentmingo.com/security>.

## 18 (served from 2026-10-31)

SHA-256 of `reporter.mjs`: `c98c7a0c289bd6534a8a627215ce2c634ddd5efc8e589245d35be8edaaba925d`

- Keep listening. With replies on, a thread that waits for you on work still open (its task isn't done or canceled, it has an open or draft pull request, or it follows a workflow run that's going) stays reachable from its page past one watcher's 8 hours, while that work is open and the computer stays on and awake, a week at most. When a long watch has listened its whole time and AgentMingo's latest answer says `keepListening: true`, the `watch` hook wakes the thread with `KEEP_LISTENING_NOTE` ("Reply with only: Still listening for AgentMingo. Don't run any tools or continue earlier work. If your person has written to you since, ignore this note and do what they asked."), so the thread takes one short turn and that turn's Stop starts a fresh watcher: about every 8 hours, up to 3 a day. Before 18, nothing on the page reached a thread after its watcher's 8 hours until you typed in it.
- AgentMingo says `keepListening` in its answers to a watch's last two requests (the time left, and `listen: 0`): true only while your setting "Keep threads listening while their work is open" is on (AgentMingo's Notifications page, on by default), the thread waits for you, replies are on for its computer, its work is open, and its wait began less than a week ago. Once the work closes, the next watch's end says no. The watcher goes by the latest answer that says; without one (an AgentMingo before 18), it's no. The answer is only true or false: the note's words are in this file.
- Its own limits on the computer: at most 21 such turns in one wait (`KEEP_LISTENING_REARMS`, 7 days of 8-hour watches; you typing in the thread, a real turn ending, or the thread starting again or ending starts the count again), only on a watch hook that gives it its whole 8 hours (`KEEP_LISTENING_MIN_CAP_MS`, so never more often than every 8 hours; a thread started before 8-hour listening, still on its 20-minute hook, never re-arms, since each turn would buy it 20 minutes, and neither does one whose hook's timeout the script can't find), only with replies still on in `config.json` as the watch ends, and never after you typed, the thread ended, a newer watcher took over, a 401 or 403, or for Cursor's own agent.
- A compaction or `/clear` keeps the lower watch-hook timeout the thread had (`startedWatchTimeoutS()`), since Claude Code keeps the hooks it started with: before 18, a compaction after an update raised the timeout made a thread on the 20-minute hook look like an 8-hour one.
- One last ask. When a watcher's time is up (any limit: 18 minutes, 8 hours, or a minute before its hook's timeout), it asks AgentMingo once more, so a reply, review results or a step that came in its last minutes isn't missed. With replies on, that request says `listen: 0` (it stops listening now), and AgentMingo counts each earlier `listen` from 18, the time left before that last ask included, as listening until then. A watcher for review results alone now makes 18 requests in its 18 minutes, not 17.
- Sleep. The watcher's pauses stop once the clock says its time is up, so a computer that slept through the end makes its last ask within 5 seconds of waking, before Claude Code's timeout, rather than after a pause's worth of minutes.
- What it sends that's new: `keepAlive: true` on the Stop event of a short turn the watcher woke the thread for (`noteKeepListening()` in `runHook()`), so AgentMingo starts no new turn for it, sends no notice, and doesn't count it as work. Its usage is reported like any turn's. Nothing else.
- New state: `state/<session id>.keep.json` (how many times this wait re-armed, and whether the next Stop ends that short turn), removed as the thread ends even when its repository no longer reports.
- Each short turn re-reads the whole conversation on your own Claude plan, like any other turn, and 8 hours on it's no longer cached, so the longer the conversation, the more it uses. AgentMingo counts its tokens in the thread's usage and lists those turns on the thread's page.
- `status` and install say, with replies on, that a thread waiting on an open task, pull request or workflow run takes one short turn as its 8 hours end, to listen again, for up to a week while the computer stays on and awake, and that each uses tokens.
- The header says all of the above, and now says "3 automatic rounds per pull request by default" (each repository sets its own) instead of "at most 3".

## 17 (served from 2026-10-30)

SHA-256 of `reporter.mjs`: `2a1791f4057c01ba62de0417a69a869481fa0432ac931787e7704c4a6a527062`

- With answering prompts from AgentMingo on, every prompt in a thread's main conversation that Claude Code hands its `PermissionRequest` hook opens on the thread's page, not only permission prompts: file edits (inside the repository or outside it), commands, MCP and other tools, questions the agent asks you (AskUserQuestion), and plans to approve (ExitPlanMode). Before 17, questions and plans opened no request.
- Each can be answered there with the terminal's own answers, and nothing is deny-only any more. Yes allows that one call; a deny rule in your Claude Code settings still wins.
- "Yes, and don't ask again" applies Claude Code's own suggestions for that prompt (the hook's `permission_suggestions`), handed back as `updatedPermissions` with the destination `session` whatever Claude Code suggested, so they last for the rest of that session and are never written to a settings file. Only allow rules, extra folders, and the modes `acceptEdits`, `default`, `plan` and `auto` are kept (never `bypassPermissions` or `dontAsk`). It is offered only when every entry can be shown exactly as it's applied: at most 10 entries, no rule over 200 characters, and no rule or folder that masking changed or that has characters that don't show (`sessionSuggestions()`).
- No denies it, with an optional note quoted to the agent as your words, and optionally stops the turn.
- A question is answered by picking its options or typing under "Other" (one line, 1,000 characters at most). The answers reach the agent through `updatedInput` as the tool's `answers` (`answeredInput()`), checked on the computer against the questions. No tells the agent you chose not to answer. A plan is approved with its own input, and with "don't ask again" also with Claude Code's own suggestions for it (switching to accept edits, say) for the session, when it made any; No tells the agent to keep planning.
- Badges replace the deny-only rules. A file outside the repository, one that changes Claude Code's or another agent's settings, one that can run code by itself, a path with characters that don't show, a path masking changed, or one that doesn't lead straight to its file can now be allowed, with a badge on the card that says so (`BADGES`: `outside_repo`, `settings`, `runs_code`, `hidden_chars`, `masked`, `indirect`, `command`, `cut`, `mcp`). New `protectedKind()` tells settings from files that run code; new `homeKind()` marks files outside the repository in the home folder, such as `~/.claude/settings.json`, `~/.claude.json`, a shell's startup files, `~/.gitconfig` and `~/.ssh/`. `isProtectedPath()` stays.
- What a prompt sends grows, all masked on the computer, best effort: for a file outside the repository, its path as given (from `~` in the home folder; before, no path was sent for one); for a command, the command (20 lines and 1,000 characters at most) and its description (300); for an MCP or other tool, its server and tool, and its input on one line of at most 300 characters (the one field that says what it does when it's the only one filled, else every filled field, so nothing hides behind one field); for a question, its questions and options (4 questions, 10 options each); for a plan, the plan (20,000 characters); the badges; and what "don't ask again" would allow. The path, the command or a tool's input goes in the request's `preview`; the rest goes in a new `details` field, and the hash of what the page showed now covers it too. `canAllow` is now always true and `why` empty, kept for older pages.
- `--commands` is retired. Commands are always shared once answering prompts is on. `approvals on --commands` is still accepted and changes nothing, like `--unchecked`. Version 17 doesn't read the earlier choice in `config.json`, so on a computer where answering prompts was on for file edits only, the update to 17 shares every prompt, and lets each be allowed from AgentMingo, without asking again. `approvals off` turns it off. Turning it on now writes `files` and `commands` as true, so a computer sent back to 16 shows commands there to deny.
- Before applying a Yes, the computer checks, as before, the answer's HMAC, that it's at most two minutes old, that the page showed exactly what this computer sent and the prompt still shows the same, and that the terminal didn't answer first; now also that "don't ask again" has suggestions from Claude Code to apply (taken from the hook's input, never from the page), and that a question's answers fit its questions.
- The terminal says what happened in words for each kind: "Allowed from AgentMingo by you at 14:05, and not to ask again this session for Bash(npm test:*)", "Answered from AgentMingo …", "Plan approved from AgentMingo …", "Not answered …", "Plan not approved …".
- The `needs_you` event for a prompt now says "Has a question for you" or "Has a plan for you to approve" for those, and adds "(from a subagent)" when a subagent asks (`permissionDetail()`). A subagent's prompt is still answered in the terminal only.
- Unchanged: it waits only in a thread's main conversation, not in a headless run, and in a terminal only once Claude Code says the prompt is on screen, within 20 seconds; in a terminal, questions and plans wait only when Claude Code sends that notice for them too. The wait (an hour by default, 5 minutes to 8 hours), the polling pace, and replies work as in 16.
- On AgentMingo's side, not in this file: the rule that allowing a prompt or sending a reply needed a sign-in in that browser within the last day is removed; a signed-in session is enough. Only the thread's own person can answer, through the page, never with an API token, and each answer is recorded in the organization's audit log and the thread's history. See <https://agentmingo.com/security#approvals>.
- The header, `approvals on`, `approvals status`, `status` and install's messages now describe every prompt being shared, with the terminal's answers. The header, `approvals on` and install's message also say that whoever gets into your AgentMingo account could allow what your agent asks to run on your computer.

## 16 (served from 2026-10-30)

SHA-256 of `reporter.mjs`: `d6a22469bf6c41d835a902d180bd175fc9b1fef7b97cd9cdee191efadfaf5dcd`

- The setup command turns on answering permission prompts and replies from AgentMingo wherever it runs: in a terminal, in an editor, or in a coding agent's Bash tool you paste it into. You no longer have to run it in a terminal of your own, or run `approvals on` afterwards. Install's `--approvals` dropped version 15's checks for a real terminal and for Claude Code's Bash tool. The one-time code stands for your consent instead: it's made only for you, signed in, works once, and for 15 minutes.
- Setup codes carry the box "Answer prompts and reply to threads from AgentMingo": `ama_` when it was ticked, `amc_` when it wasn't. AgentMingo stores only the code's hash, so a code whose prefix was changed by hand isn't one AgentMingo knows, and install stops before anything changes. When install trades a code, AgentMingo's answer now also says whether it was made with the box ticked (`approvals`), and `tradeSetupCode()` returns that. `--approvals` turns it on only when that answer says ticked.
- `--approvals` never makes the install fail. With the older `--token` form, or with `--approvals` added to a command made with the box unticked, install connects the computer, leaves answering prompts and replies off, and prints a line saying so, why, and how to turn it on (`approvalsStayedOffSaid()`). In 15, install refused and changed nothing. Already on, `--approvals` still changes nothing. `approvalsFromSetup()` now takes the traded code's answer and returns `"on"`, `"no_code"`, `"unticked"` or `null`.
- `approvals on` is unchanged: it works only in your own terminal, outside Claude Code's Bash tool, and asks you to type yes, since it has no one-time code. Its refusal, and the line `status` and install print when it's off, now also say that a new setup command with the box ticked turns it on.
- Install reads Claude Code's settings before it trades the one-time code. Settings that aren't valid JSON stop install while the code still works, and nothing changes. Before 16, install traded the code first, so a broken settings file used it up.
- Masking on the computer covers `ama_` codes too, alongside `amc_` codes and `amk_` tokens. AgentMingo's own masking got the same rule.
- What it sends is unchanged. Install still sends the setup command's one-time code, the organization's short name and the script's version.
- The header says all of the above: where the setup command turns it on, that the code is your consent and carries the box, what happens with `--token` or an unticked code, and the limit (whoever runs a command made with the box ticked while its code is unused turns it on, you or the agent you gave it to).

## 15 (served from 2026-10-30)

SHA-256 of `reporter.mjs`: `587423d45826e291f1c02e905bc0f40d5c3c7caf3a8ea92f22630361dde83ac4`

- Install's new `--approvals` turns on answering permission prompts and replies from AgentMingo. The setup command carries it when its box "Answer prompts and reply to threads from AgentMingo" is ticked, as it is by default; unticked, the command carries `--no-approvals`, which turns it off. It turns it on only with the setup command's one-time code (`--code`; never with the older `--token`), in a real terminal, and not inside Claude Code's Bash tool. Otherwise install refuses, says why, and changes nothing, before the code is used. Already on, it changes nothing and refuses nothing. When it turns it on, install prints what it turned on: file edits, an hour's wait, commands not sent, replies. `approvals on` still works, and still asks you to type yes. Before 15, no install flag turned it on.
- Replies from the website. With answering prompts on, a thread that finished its turn and waits for you can be answered from its page (Continue, Yes, No, or up to 1,000 characters). The watcher picks the reply up, claims it on the computer and on AgentMingo so it's handed over once, and wakes the thread with "<first name> replied from AgentMingo: …" and a line saying it's your reply, with the workflow step and review results your next message would have carried. A reply belongs to the turn it answers: typing in the thread first, the thread ending, or the thread going on without it cancels it. Each report now says whether replies are on, and with replies on, each watcher request says in how many seconds it asks next. Replies go with the answering-prompts key, so a computer where it was turned on under 14 takes replies once it runs 15.
- The watcher listens up to 8 hours after each reply in a thread that follows a workflow, and in any thread with replies on: every 30 seconds for 3 minutes, every minute until 10, every 2 minutes until 30, then every 5 (about 110 requests in 8 hours). Alone for review results, it still waits 18 minutes. It now stops within 5 seconds of you typing in the thread or the thread ending, looked at on the computer without a request, and the thread's next reply starts a new watcher that takes over from the one before, which stops (before, the new reply's watcher didn't start while an earlier one still waited).
- The `watch` hook's timeout is 8 hours and 10 minutes (29400 seconds; it was 1200). Install writes it, and on a computer installed before 15, the reporter raises it itself, once, after it next sends: only from the copy in `~/.agentmingo`, only on its own `watch` hook, only when lower, changing only that number in `~/.claude/settings.json`. A timeout you lower afterwards stays. When it can't write the settings, it logs why, and `status` says to run the setup command again. Threads already open keep the timeout they started with until you restart them (Claude Code may say their hooks changed); the watcher never listens past a minute before its thread's timeout.
- After the 3 automatic review rounds on a pull request, results are held for you to say continue: your next message, or a reply from the thread's page. A watcher that listens 8 hours goes on listening past the rounds; before, it stopped.
- Waiting on permission prompts no longer depends on a list of checked Claude Code versions (14 listed none, so prompts were only reported unless you chose `--unchecked`). The list is now of versions known to misbehave (`KNOWN_BAD_TERMINAL_VERSIONS`, `KNOWN_BAD_APP_VERSIONS`), empty so far. In a terminal, the guard is Claude Code's notice that the prompt is on screen: without it within 20 seconds, the hook leaves the prompt to the terminal at once. `--unchecked` is still accepted and changes nothing. So with answering prompts on, a prompt now waits for an answer from AgentMingo on any Claude Code version not on that list.
- Wording: `status`, `approvals status` and install say "Answering prompts and replies from AgentMingo", and that commands can only be denied there. `status` and install add a line on how long the watcher listens, from the hook's timeout, and whether an update raised it. `approvals on` lists replies among what it turns on, and its refusal inside a thread now says "it doesn't turn on inside Claude Code's Bash tool".
- New state: `state/<session id>.told/reply-…` claim files (removed with the thread's folder), the thread's end count and starting watch timeout in `state/<session id>.hook.json`, a random id and start time in the watch lock, and `watchTimeoutChecked`, `watchTimeoutRaisedAt` and `watchTimeoutWas` in `config.json`.
- The header says all of the above: how answering prompts and replies turn on (the box, `--approvals` and its checks, what those checks don't stop), the known-bad version list and the on-screen guard, replies and what they send, the 8-hour watcher, the timeout raise and threads already open, held results past the rounds, and the update's one change to the settings.

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
