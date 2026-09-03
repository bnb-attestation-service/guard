---
description: Set up AgentGuard end to end — install the binary if missing, run the first scan of ~/.claude, and offer the load-time gate.
argument-hint: "[config root, default ~/.claude]"
---

Get this machine from nothing to protected. Work through the steps in order and stop to
report at each decision point — do not install a hook or change any config without agreement.

1. **Binary.** `command -v aguard || ls ./bin/aguard`. If missing, use the `agentguard-audit`
   skill's `references/install.md` — including the checksum verification, which is not optional.
   Report the version.

2. **First scan.** Use the `agentguard-audit` skill. Root: $ARGUMENTS (default `~/.claude`).
   Run `aguard scan --verbose`, then triage properly per that skill's `references/triage.md` —
   worst artifact by name and score, real findings separated from advisory shapes, and any
   dimension-0 note that changes how the result should be read (`IGN-000`/`REP-GOOD`
   suppressions, a `COV-000` saying nothing was collected, `GATE-001`).

3. **Fix plan.** For each surviving finding: fix, remove, or baseline with a written reason.
   Present it; let the user choose. Do not edit their config unprompted.

4. **Gate.** Offer the load-time gate via the `agentguard-gate` skill — `aguard hook status`,
   then `aguard hook install --dry-run` shown before any real install, from a *stable* binary
   path. State plainly that it covers skills only, that plugin hooks and MCP servers are live
   from turn one and are not gated, and that it takes effect only for sessions started after a
   restart.

Finish with what is now protected, what is not, and the one thing you would do next.
