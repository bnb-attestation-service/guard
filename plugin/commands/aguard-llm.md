---
description: Set up or check the optional AI deep check — a hosted model you provide, configured in one step.
argument-hint: "[status|test|setup]"
---

Use the `agentguard-audit` skill's `references/llm.md`. Requested action: $ARGUMENTS (if empty,
start with `aguard llm status` and report what is and is not configured).

The deep check sends redacted excerpts of the user's skills and config to a model endpoint the
USER chooses and pays for. Before anything is stored or sent: say which provider and endpoint
the content will go to, and get agreement. For the key itself, offer the terminal route first
(`aguard llm setup --provider <name>` prompts with hidden input) and the paste-it-here route
only with the transcript caveat stated. Never print or echo the API key back, never put it in
a file other than the one `aguard llm setup` writes, and never run a scan with `--llm` until
`aguard llm test` has answered OK.
