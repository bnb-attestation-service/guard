# Setting AgentGuard up, end to end

This is the first-run flow. `/aguard-setup` runs it; so does the audit skill when the user asks
to set up, install or get started with AgentGuard in plain words, or asks for a check on a machine
where the binary is not installed yet — a first scan without the gate offer and the usage card is
a first run done halfway. One flow, one file; edit it here.

Get this machine from nothing to protected. Work through the steps in order and stop to
report at each decision point — do not install a hook or change any config without agreement.

The flow speaks English end to end — roadmap, step headers, findings, questions, cards —
regardless of the conversation language. If the user asks something mid-flow, answer in their
language, then return to the flow in English.

Before step 1, print this roadmap so the user knows what is coming and how much of it is
theirs to decide:

> Setting up AgentGuard — five steps:
>
> 1. Install the `aguard` binary (checksum-verified)
> 2. First scan of your setup
> 3. Walk through what it found — **you choose** what happens to each finding
> 4. Offer the load-time gate — **your decision**
> 5. Offer the optional AI deep check — **your decision**
>
> Nothing is installed or changed without your agreement.

Then start each step's report with `[Step N/5] <name>` — e.g. `[Step 2/5] First scan` — so the
user always knows where they are and how many decisions remain.

1. **Binary.** `command -v aguard || ls ./bin/aguard`. If missing, use the `agentguard-audit`
   skill's `references/install.md` — including the checksum verification, which is not optional.
   Run its steps as separate single commands. If the app blocks a download or move step (auto
   mode does this at random), follow install.md's "If a step is blocked" section exactly: one
   sentence, the three Terminal lines with the platform filled in, wait for "done". Do not
   retry or improvise around the block.
   If present, do NOT stop at "already installed": run `aguard version` — if it reports the
   plugin is newer than the binary, that is the answer — otherwise compare it with the
   latest release tag as described in install.md's "Upgrading" section, and when it is older,
   upgrade in place (same path, so an installed gate keeps working). Report the version you
   ended up with. A machine on an old binary gets none of the newer rules and none of the
   allowlist, and it looks exactly like an up-to-date one until someone checks.

2. **First scan.** Use the `agentguard-audit` skill. Root: $ARGUMENTS (default `~/.claude`).
   Run `aguard scan --verbose --report` (the HTML report lands in the reports directory and
   its path is printed; the Downloads section lists agent-shaped items sitting in ~/Downloads,
   each with its own score, outside the environment score), then triage properly per that skill's `references/triage.md` —
   worst artifact by name and score, real findings separated from advisory shapes, and any
   dimension-0 note that changes how the result should be read (`IGN-000`/`REP-GOOD`
   suppressions, a `COV-000` saying nothing was collected, `GATE-001`).
   This is the first score the user has ever seen from this tool: print the score card from
   `references/score-card.md` right under it, so the number arrives with its reading
   instructions.
   **If the report carries the sandbox banner** (it ran in Claude Cloud / Cowork, root
   `/root/.claude`), lead with that, not the score: the number describes a throwaway cloud box,
   not the user's computer, and their real skills, hooks, permissions and connectors were not
   reachable from there. Tell them to run the scan in the desktop app's Code tab (`</>`) on
   their own machine for a result about their setup.

3. **Fix plan.** Do not walk the findings one by one yet — deferring them all is the default,
   and it costs nothing: nothing changes, and every future scan re-reports them. Say how many
   findings survived triage and ask ONE question: handle them now, or defer them all?
   Deferring ends this step in one line. Handling now enters the full plan — follow
   `references/remediate.md` ("Presenting the plan"): per-finding structured choices, at most
   four per round. Either way, do not edit their config unprompted.

4. **Gate.** Offer the load-time gate via the `agentguard-gate` skill. Open with what it is
   and what it buys, before any command output:

   > The gate is automatic protection at the moment that matters. It is a Claude Code hook
   > that checks every skill right before an agent loads it — clean skills pass silently,
   > anything carrying a finding asks you first. A scan tells you what was already in your
   > setup; the gate stands in front of what tries to load next. Approvals are remembered by
   > content hash, so an edited skill asks again by itself.

   Then `aguard hook status`, and `aguard hook install --dry-run` shown before any real
   install, from a *stable* binary path. State plainly that it covers skills only, that plugin hooks and MCP servers are live
   from turn one and are not gated, and that it takes effect only for sessions started after a
   restart. Say how many of each the scan actually found — its `hooks` and `mcp_servers` counts
   include the ones plugins bundle — and the `connectors` count is remote connectors
   seen in desktop sessions and checked for tool-description poisoning (they are not gated — live
   from turn one); a connector used only in the browser is not in the cache, so it is "seen in
   desktop sessions", never "all your connectors".

5. **Deep check — one question, no flow.** Say that an optional AI deep check exists (an
   isolated model, a hosted account of their own, redacted excerpts sent off the machine) and
   ask whether to set it up now or later. "Later" ends it here — the usage card says how to come
   back to it. "Now" → the `agentguard-audit` skill's `references/llm.md`. Do not pitch it: the
   static scan is complete without it, and this is already the run where they made two decisions.

Finish with what is now protected, what is not, and the one thing you would do next. Open the
HTML report in the browser (`open <path>` on macOS, `xdg-open <path>` on Linux) — this is the
one run where a visual report is worth a window — and say where the file is. Then
print the usage card from the `agentguard-audit` skill's `references/usage.md` and end on it — it is the last thing on screen, so it is the part they will keep.
