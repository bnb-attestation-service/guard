# Turning findings into changes

Three outcomes exist for a finding, and the third is the one to argue against.

1. **Fix the artifact** — narrow the grant, replace the `curl|bash`, move the secret out.
2. **Remove it** — the artifact is not worth the risk, or the user does not remember installing
   it. Prefer quarantine over deletion so the decision is reversible.
3. **Baseline it** — record that it was reviewed and accepted. Requires a reason, in writing.

Never silently pick (3) because (1) looked like work.

## Fixes for the findings you will actually see

### Permission allowlists (`settings.json`)

| Finding | Fix |
|---|---|
| `PERM-005` — `Bash(*)` | Delete it. Command-layer protection is off while it is there. Replace with the handful of commands actually needed. |
| `PERM-002` — wildcarded interpreter (`Bash(sh *)`, `Bash(python *)`) | Same capability as `Bash(*)`. Replace with exact commands. |
| `PERM-006` — escapable binary (`Bash(git *)`, and `find` `awk` `tar` `docker` `ssh` `npm` `make` `xargs` `rsync`) | `git -c core.pager=…` reaches arbitrary execution, so an open-ended grant over `git` is not narrower than `Bash(*)`. **Fully specify it**: `Bash(git status)` and `Bash(git status:*)` both stay clean. The rule requires the grant to be open-ended *and* the escape handle to sit in the open segment — so pinning the subcommand is a real fix, not a trick to silence it. |
| `PERM-003` — `**` in a Read/Write/Edit grant | Narrow to specific sub-paths. |
| `PERM-001` — plaintext secret inside an allow entry | Remove the value, move it to a secret manager or an env var, **and rotate the credential** — it has been sitting in a config file, likely in git. Raise rotation explicitly; the scanner cannot. |
| `PERM-004` — allow with no deny | Add a deny fallback for `~/.ssh`, `~/.aws`, `.env`. Raised once per settings scope, not per entry. |

A grant that names one of the user's own scripts (`Bash(./scripts/deploy.sh *)`) is only as
narrow as what that script does — the scanner reads the script too, so findings attributed to
it are findings about that grant.

### Hooks

`HOOK-001` (shell chaining in a hook command) is unremarkable in a script and telling in a
hook — hooks run silently on every matching tool call. Fix: move the logic into a script file
and register a single command pointing at it. The script then gets read and scanned, which is
the point.

If a hook is one the user did not add, that is the finding. Hooks are the surface where
"I don't know where this came from" should stop the conversation and start an investigation.

### Execution and supply chain

- `EXEC-001` / `EXEC-002` (`curl|bash`, `wget|sh`) — download, verify a checksum, then run.
  Two steps instead of one.
- `EXEC-008` (`powershell -enc`) — treat as hostile until proven otherwise; decode it and read
  it before anything else.
- `SUP-001` / `SUP-002` — install from the official registry over HTTPS.
- `SUP-003` — pin the version.
- `SUP-004` — the artifact points the agent into a directory the scanner does not read
  (`dist/`, `node_modules/`, …). Either commit the real script somewhere readable, or accept
  that the referenced half was never audited. Do not baseline this one without reading the
  target by hand.

### Injection and obfuscation

- `INJ-001`…`INJ-003` — remove the directive. In a skill the user wrote, this is usually a
  clumsy prompt; in one they installed, it is the reason the scan exists.
- `INJ-004` (zero-width / directional control characters) — **look at the bytes**, not the
  rendered text. That is the whole technique. `grep -nP '[\x{200B}-\x{200F}\x{202A}-\x{202E}\x{2060}\x{FEFF}]'`
  on the file shows where they are.
- `OBF-001`…`OBF-004` — decode the blob and read it. Decode; never execute.

### Filesystem and exfiltration

- `FS-001` / `FS-002` (SSH key / AWS credential reads) and `FS-003` (`rm -rf` at root or home)
  in an installed skill: stop, do not narrow the code — ask why it is there at all.
- `EXFIL-001` / `EXFIL-003` — the two (or three) legs are named in the evidence. Read both
  before proposing anything; the honest version of this shape is a tool that reads a token to
  authenticate its own API call.

## Removing an artifact

Quarantine, don't delete:

```bash
aguard clean --zombie --apply --dry-run   # preview
aguard clean --zombie --apply             # move to <root>/.aguard-trash (recorded; --undo restores)
aguard clean --resolve D-… --keep <name>  # a duplicate pair: quarantine the side NOT named
aguard clean --undo last                  # put a batch back if the call was wrong
```

Those paths cover never-used skills and duplicate pairs. For any other artifact the user has
decided against, move it out of the config root by hand — `mv` to a directory **outside**
`~/.claude` (not into `.aguard-trash`, which is clean's managed store and expects its own
records), then rescan. Show the `mv` and get agreement first. Quarantined and hand-moved
content alike still scores until it is actually deleted — that is deliberate, so cleanup
cannot silently turn a failing score green.

## Baselining (`.aguardignore`)

Once findings have been reviewed, silence the accepted ones so repeat scans stay signal-only.
`scan` and `clean` read `<root>/.aguardignore`, or `--ignore <file>`:

```
# one rule per line — blank lines and # comments ignored
INJ-001                  # suppress this rule everywhere
EXEC-004  vendor/*       # suppress this rule only under matching evidence paths
*         dist/*         # suppress ANY rule under a path glob
```

Rules for writing one, in order of importance:

- **Every entry gets a comment saying who accepted it and why.** An uncommented baseline entry
  is indistinguishable from a bug being hidden, and six months later nobody can tell them
  apart. This is the single most useful thing you can add to the file.
- **Scope it with a path.** `EXEC-004 skills/deploy/*` accepts a decision about one skill;
  bare `EXEC-004` accepts it for every skill the user will ever install, including the next
  malicious one.
- **Never baseline a `critical` or an unreviewed `high`.** Suppression happens *before*
  scoring, so a baseline changes the number. The `IGN-000` note carries the highest severity
  it silenced precisely so this cannot be done quietly — but it is still the user who has to
  notice.
- `*  *` is not a configuration. If that is where the conversation is heading, the honest move
  is to say the tool is not being used.

`check` deliberately does **not** auto-discover a baseline — see `agentguard-vet` for why.

## Baseline vs. approve — different mechanisms, don't mix them

- `.aguardignore` suppresses **findings before scoring**. It changes the score and the
  `--fail-on` result. Keyed by rule ID and evidence path.
- `aguard approve <path>` tells the **load-time gate** to trust one exact set of bytes. It is
  keyed by the artifact's **canonical hash**, changes no score, and re-opens by itself the
  moment the artifact is edited.

"I've reviewed this skill and I'm keeping it" is an `approve`. "I've reviewed this rule firing
in this location and I accept it" is a baseline. See `agentguard-gate` for approvals.

## Finish by rescanning

```bash
aguard scan --root <root>
```

State the before/after: which artifact, which findings gone, which deliberately baselined and
on whose call. If the score moved because artifacts were *removed* rather than fixed, say that
too — a mean over fewer artifacts is a different number, not a better environment.
