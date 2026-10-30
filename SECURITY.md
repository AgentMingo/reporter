# Security

## Reporting a vulnerability

Email **hello@agentmingo.com** with what you found and how to reproduce it. We'll confirm we got it and keep you posted until it's fixed. English is preferred.

The same address is in <https://agentmingo.com/.well-known/security.txt>. The full policy is at <https://agentmingo.com/security>.

Please don't open a public issue or pull request for a vulnerability.

Please don't access or change other people's data, disrupt the service, or make the issue public before it's fixed. If you follow that and act in good faith, we won't take action against you for your research.

## What's in scope here

This repository holds the reporter that runs on people's computers: `reporter.mjs`, and the `launch.mjs` and `reporter.sh` it writes into `~/.agentmingo` (their text is in `reporter.mjs`: the `LAUNCHER` constant and `shimText()`). Reports about the AgentMingo service itself, which is closed source, go to the same address.

## Supported versions

Only the latest version gets fixes. Reporters update themselves unless their person turned that off (`auto-update off`), and AgentMingo can send reporters back to an earlier version (10 or newer) while a fix is made. Updates come from agentmingo.com over HTTPS and aren't signed; the README says what the script checks before it uses one.
