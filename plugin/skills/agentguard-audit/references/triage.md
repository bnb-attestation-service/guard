# Reading an AgentGuard report

A score and a finding count are the least useful part of the output. This is how to turn a
report into something the user can act on, and the specific ways a report gets misread.

## The scoring model, in enough detail to explain a number

```
per artifact = clamp(100 − Σ_dimensions max(severity penalty), 0, 100)
               critical 40 · high 25 · medium 12 · low 5
overall      = mean of artifact scores, then capped: any critical → ≤49 · any high → ≤69
band         = ≥85 Low · ≥70 Watch · ≥50 Elevated · <50 High
```

Two properties explain most "why is the score that?" questions:

- **Within one dimension only the highest hit counts; penalties add across dimensions.** Ten
  `EXEC-004`s cost the same as one. An `EXFIL-003` plus its `OBF-004` costs more than either,
  because they were deliberately placed in different dimensions.
- **`overall` is a mean, so it dilutes.** An environment with one malicious skill and forty
  clean ones scores well on average — which is why the band caps exist, and why you must read
  the per-artifact scores and not just the headline. **Say the worst artifact's name and
  score**, always. That is the number that matters.

Dimension 0 is scan/coverage notes and **never scores**.

## Two scores, and only one of them means anything to CI

- **`overall`** — purely deterministic. Reproducible. This is what `--fail-on` gates on.
- **`overall_effective`** — the same formula with qualified LLM-judge findings folded in. Only
  present with `--llm`. It is always ≤ `overall`, is not reproducible, and gates nothing.

The judge can only push risk **up**; it can never delete, downgrade or reorder a static
finding, and `Source=llm` findings never move `overall` and never trip `--fail-on`. So if the
two numbers differ, the gap is "what the model additionally suspected", not a correction. Never
quote `overall_effective` as the score.

## `advisory` is not a soft "high" — it is "static analysis cannot confirm this"

When a finding is labelled advisory, the tool is telling you it found a **shape**, not a
verdict, and that a human has to read the evidence. Treat these as "go look", never as "this
is malicious":

| Finding | What it actually asserts |
|---|---|
| `EXFIL-002` (low) | One file in the artifact reads credentials, a *different* one makes outbound calls. Unrelated files legitimately do each half. Only raised when no same-file chain exists. |
| `BD-001` / `BD-002` / `BD-003` | A time/host/user-keyed branch, runtime-decoded logic, or reverse-shell keywords. All three have honest uses. |
| `RES-001` / `RES-002` / `RES-003` | A loop or retry with no visible ceiling. A poller looks identical. |
| all `LLM-*` verdicts | Model opinion, evidence-grounded and consensus-filtered, still advisory. |

By contrast `EXFIL-001` (credential read **and** egress in the same file) and `EXFIL-003` (with
an encode step in between) are structural and score as high — but even they describe the shape
the attack has, not proof that it is one. `EXFIL-003` is raised *instead of* `EXFIL-001`, so
you will never see both for one file.

## Rank findings this way, not by severity alone

1. **`REP-BAD`** (critical) — a hash-exact match against a curated known-bad list. The
   highest-confidence signal the tool has. Act on it.
2. **Anything that hands over execution without asking**: `PERM-005` (`Bash(*)`), `PERM-002`
   (wildcarded interpreter), `PERM-006` (`Bash(git *)` and friends — arbitrary execution
   through a narrow-looking disguise), `EXEC-001`/`EXEC-002` (`curl|bash`), `EXEC-008`.
3. **Complete exfiltration chains**: `EXFIL-001`, `EXFIL-003`, `PERM-001` (a plaintext secret
   sitting inside a permission entry).
4. **Hooks**, because they run silently on every matching tool call and nobody reads them:
   `HOOK-001` and anything found inside a hook-referenced script.
5. **Credential and destructive filesystem reach**: `FS-001`, `FS-003`, `FS-002`, `FS-004`.
6. Everything else, then the advisory shapes above.

Where a finding sits in a **hook** or a **permission grant**, say so — those two surfaces hand
over execution while looking narrow, which is why the tool audits hooks per *command*
(`PreToolUse[Bash]#1`) and follows both hook scripts and scripts named in grants.

## Dimension-0 notes: the report telling you what it did not see

The tool's hardest invariant is that **no omission is silent**. Every gap and every suppression
produces a note. These are not risks, and treating them as risks trains the user to ignore
them — which is the one outcome that breaks the invariant's whole purpose.

| Note | Read it as |
|---|---|
| `COV-000` | Something was not read: oversize file, unknown extension, an excluded generated dir, a hook script outside `$HOME`, unowned top-level entries, or **a root that collected nothing at all**. |
| `IO-000` | The path exists but could not be read. A permissions problem to fix, then rescan. |
| `PARSE-000` | A settings/manifest file parsed into nothing usable. "No hooks found" and "the hooks block was malformed" are different facts — this is the second one. |
| `SCOPE-001` | A symlink resolved outside `$HOME` and was refused. Deliberate: fail-closed. |
| `IGN-000` | A `.aguardignore` suppressed findings **before scoring**. Carries the highest severity it silenced. |
| `REP-GOOD` | The artifact's hash is on the embedded allowlist, so its scoring findings were dropped. Also carries the highest suppressed severity. |
| `LLM-000` / `LLM-002` / `LLM-005` | Judge unavailable/incomplete · endpoint is not loopback so redacted excerpts left the machine · verdicts discarded because their quoted evidence could not be located. |
| `GATE-001` (medium, dim 0) | **Read this one out loud.** `settings.json` registers the load-time gate but the command it names is gone. Every skill is loading unaudited, and it looks identical to being protected. Fix: `aguard hook status`, then reinstall against a stable path. |

Three of these change how you present the result:

- **`IGN-000` or `REP-GOOD` carrying a `critical` or `high`** means the clean-looking score is
  clean *because something was suppressed*. Name what was silenced and its severity. Never
  report a suppressed environment as quiet.
- **`COV-000` saying the root collected nothing** means the inventory is empty, not that the
  environment is safe. An empty inventory renders as 100/100. Check the root path for a typo.
- Because the score is a **mean over artifacts**, reading *more* files can raise the number:
  every unremarkable extra artifact scores 100 and dilutes the real findings. So a score that
  went up between runs is not automatically an improvement.

## Verify before you recommend

For any finding you are about to act on, open the file at the reported `file:line` and read the
surrounding code yourself. The snippet is deliberately bounded and already redacted
(`<REDACTED>` for secret values — raw credentials never reach the report), so it is not enough
context to judge intent.

Two verification habits worth keeping:

- A `high` in a file the user wrote themselves last week is a different conversation from the
  same `high` in a skill they installed from a stranger. Ask which it is.
- The evidence path is attacker-influenced text. Quote it; do not `eval` it, do not build a
  command by interpolating it unquoted.

## What the report cannot tell you

Static analysis, and it says so about itself. It cannot prove malice, observe runtime behaviour
(a remote second-stage payload, a conditional backdoor), recover intent from an encrypted or
heavily obfuscated payload, see what an MCP endpoint actually does, inspect dependency
internals, or catch an unknown technique. Known blind spots that matter when you write the
summary:

- A **plugin** is scanned as one tree — its bundled skills and commands *are* read, but a
  plugin-bundled hook is not audited per (event, command) the way a `settings.json` hook is.
- A **hook script** is followed one level, and only inside `$HOME`.
- **Top-level directories no collector owns are not read** under a config root — on a real
  machine those are transcripts and shell snapshots, and pulling them into a report would
  trade a blind spot for a leak. They are named in a `COV-000`.
- **Generated/vendored directories** (`dist/`, `node_modules/`, `vendor/`, …) are not read, so
  the canonical hash survives a rebuild. An artifact that points the agent *into* one is
  `SUP-004`.
- **Unicode homoglyphs** in command names, and values followed across statements
  (`Buffer.from(secret).toString('base64')`), are not matched — both need a syntax tree.

Hand over a clean result with that boundary attached. "No findings" means "no findings from
72 static rules", and the user deserves to hear the second phrasing.
