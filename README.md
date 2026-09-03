# AgentGuard (`aguard`)

> A local, read-only security scanner for AI agent environments — an "antivirus for your
> Claude Code setup." It audits everything an agent auto-loads (skills, MCP servers, hooks,
> permission grants, subagents, slash commands, `CLAUDE.md`) for prompt injection, credential
> exfiltration, arbitrary-execution grants and more. Fully offline; it never executes what it
> scans and never makes a network call.

This is the **distribution repo**: the plugin and prebuilt binaries. The source lives in a
separate repository.

## Install (Claude Code plugin)

Two commands, then a restart. In the **terminal** Claude Code they are slash commands; in the
**VS Code / JetBrains extension** run them from a shell as `claude plugin …` (the extension has
no `/plugin`).

```
/plugin marketplace add bnb-attestation-service/guard
/plugin install agentguard@agentguard
```

Restart Claude Code (the extension: reload the window), then run:

```
/aguard-setup
```

`/aguard-setup` brings up the `aguard` binary if it is missing (downloads the prebuilt one and
verifies its checksum), runs a first scan of your `~/.claude`, walks you through the findings,
and offers the load-time gate. From then on you just talk to it in plain language.

## Use it — no commands to memorize

Once set up, the three skills fire on their own when you ask. Say things like:

- *"Is my `~/.claude` safe? Scan it."*
- *"I downloaded this skill — check it before I install it."*
- *"What does `EXFIL-001` mean?"*
- *"My config is bloated, clean it up."*

Or reach for the slash commands directly: `/aguard-scan`, `/aguard-vet <path>`, `/aguard-gate`.

If you installed the load-time gate, every skill an agent tries to load is scanned first —
clean ones pass silently, risky ones are held for your decision, and editing an approved skill
re-opens the question by itself (approvals are keyed by content hash).

## What a finding means

Every rule ID the scanner can print — with its dimension, severity, and why it fires — is
listed in [`docs/rules.md`](docs/rules.md). Look one up rather than guessing from its name.

## What it is, and is not

It is a **static** scanner: a relative risk signal + cleanup, not a safety certificate. It
cannot prove malice, observe runtime behavior, decrypt an obfuscated payload, or see what an
MCP endpoint actually does. A clean report means "no findings from the static rules," which is
worth reading as exactly that.

The one command that writes is `clean`, and it only ever **moves** things into
`<root>/.aguard-trash` (reversible with `--undo`) — it never deletes. Everything else is
strictly read-only.

## Manual install (if `/aguard-setup` cannot fetch the binary)

Download the binary for your platform from the [latest release](../../releases/latest), plus
`SHA256SUMS.txt`, then:

```bash
cd ~/Downloads
shasum -a 256 --ignore-missing -c SHA256SUMS.txt          # verify — do not skip
chmod +x aguard-*                                          # e.g. aguard-darwin-arm64
xattr -d com.apple.quarantine aguard-* 2>/dev/null         # macOS: clear the download flag
mkdir -p ~/.local/bin && mv aguard-* ~/.local/bin/aguard
aguard version                                             # prints a version = installed
```

Make sure `~/.local/bin` is on your `PATH`. Platforms: `darwin-arm64` (Apple-silicon Mac),
`darwin-amd64` (Intel Mac), `linux-amd64`, `linux-arm64`.

## License

MIT.
