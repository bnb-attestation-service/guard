---
description: Install, inspect, repair or remove AgentGuard's load-time gate, and manage its hash-keyed approvals.
argument-hint: "[install|status|uninstall|approvals|approve <path>|approvals forget <hash>]"
---

Use the `agentguard-gate` skill. Requested action: $ARGUMENTS (if empty, start with
`aguard hook status` and report what is and is not wired up).

`settings.json` belongs to the user: always show `aguard hook install --dry-run` and get
agreement before writing, install from a stable binary path, and say that the gate covers
skills only and takes effect only for sessions started after a restart.
