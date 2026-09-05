---
description: Set up AgentGuard end to end — install the binary if missing, run the first scan of ~/.claude, and offer the load-time gate.
argument-hint: "[config root, default ~/.claude]"
---

Get this machine from nothing to protected. Work through the steps in order and stop to
report at each decision point — do not install a hook or change any config without agreement.

1. **Binary.** `command -v aguard || ls ./bin/aguard`. If missing, use the `agentguard-audit`
   skill's `references/install.md` — including the checksum verification, which is not optional.
   If present, do NOT stop at "already installed": run `aguard version` — if it reports the
   plugin is newer than the binary, that is the answer — otherwise compare it with the
   latest release tag as described in install.md's "Upgrading" section, and when it is older,
   upgrade in place (same path, so an installed gate keeps working). Report the version you
   ended up with. A machine on an old binary gets none of the newer rules and none of the
   allowlist, and it looks exactly like an up-to-date one until someone checks.

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
   restart. Say how many of each the scan actually found — its `hooks` and `mcp_servers` counts
   include the ones plugins bundle — and never say "zero" about a surface the scan does not
   collect: Claude Desktop's own remote connectors are not read at all, so they are "not
   collected", not "none".

5. **Deep check — one question, no flow.** Say that an optional AI deep check exists (an
   isolated model, a hosted account of their own, redacted excerpts sent off the machine) and
   ask whether to set it up now or later. "Later" ends it here — the usage card says how to come
   back to it. "Now" → the `agentguard-audit` skill's `references/llm.md`. Do not pitch it: the
   static scan is complete without it, and this is already the run where they made two decisions.

Finish with what is now protected, what is not, and the one thing you would do next. Then
print the usage card from the `agentguard-audit` skill's `references/usage.md` and end on it — it is the last thing on screen, so it is the part they will keep.
