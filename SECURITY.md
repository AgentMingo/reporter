# Security

## Reporting a vulnerability

Email **hello@agentmingo.com** with what you found and how to reproduce it. We'll confirm we got it and keep you posted until it's fixed. English is preferred.

The same address is in <https://agentmingo.com/.well-known/security.txt>. The full policy is at <https://agentmingo.com/security>.

Please don't open a public issue or pull request for a vulnerability.

Please don't access or change other people's data, disrupt the service, or make the issue public before it's fixed. If you follow that and act in good faith, we won't take action against you for your research.

## What's in scope here

This repository holds the reporter that runs on people's computers: `reporter.mjs`, and the `launch.mjs` and `reporter.sh` it writes into `~/.agentmingo` (their text is in `reporter.mjs`: the `LAUNCHER` constant and `shimText()`). Reports about the AgentMingo service itself, which is closed source, go to the same address.

From version 14 that includes answering permission prompts from AgentMingo (`approvals`, off unless a person turns it on): how it's turned on, what a prompt sends, and the checks the reporter makes before it applies an answer, such as an answer applied to a prompt it wasn't made for, Allow offered for a file that can run code or change settings, or a secret in a path or command that masking should have caught. It also includes the task folders `start` and the `start_task` tool make, and which `.env` files they copy. The page that shows a prompt and takes the answer is part of the service; report it to the same address.

## Supported versions

Only the latest version gets fixes. Reporters update themselves unless their person turned that off (`auto-update off`), and AgentMingo can send reporters back to an earlier version (10 or newer) while a fix is made. Updates come from agentmingo.com over HTTPS and aren't signed; the README says what the script checks before it uses one.
