# Publishing / keeping this repo in sync

This is the **public distribution repo**. It holds no source — only the plugin (skills +
commands), the generated rule reference, the CI helper scripts, and, on each release, the
prebuilt binaries. The source of truth is the **private** repo
`bnb-attestation-service/agent-guard`. Everything here is copied out of it.

So there are two things to keep in sync, and they move on different clocks:

| What | When it changes | How to sync |
|---|---|---|
| `plugin/`, `docs/rules.md`, `hack/` | whenever those change in the source repo | copy them over, commit, push |
| release binaries | on each new version | build in the source repo, upload here as a release |

## 1. Sync the text (plugin, rules, hack)

From a checkout of the **private source repo** (on the tag or branch you are publishing):

```bash
SRC=path/to/agent-guard          # private source checkout
DIST=path/to/guard              # this repo's checkout

rm -rf "$DIST/plugin" "$DIST/docs" "$DIST/hack"
cp -r "$SRC/plugin"                       "$DIST/plugin"
cp "$SRC/.claude-plugin/marketplace.json" "$DIST/.claude-plugin/marketplace.json"
mkdir -p "$DIST/docs" "$DIST/hack"
cp "$SRC/docs/rules.md"                   "$DIST/docs/rules.md"
cp "$SRC/hack/pre-commit" "$SRC/hack/github-action.yml" "$DIST/hack/"
```

Sanity-check that the plugin still scans clean before committing — this is the same gate the
source repo's CI enforces, and it is the thing that would embarrass the tool if it broke:

```bash
aguard check "$DIST/plugin" --fail-on low     # must exit 0
```

Gate the **`plugin/` subtree**, not the whole repo: `aguard check "$DIST"` (the whole tree)
flags `docs/rules.md` as `INJ-002`, because the rule reference quotes the very attack phrases
the engine detects — the same reason the source tree self-reports. That is expected and
harmless: `docs/rules.md` never enters a user's `~/.claude` (only `plugin/` is installed via
the marketplace), so it is `plugin/` that must stay clean, and it does.

Then `git add -A && git commit && git push` in `$DIST`.

**Note:** the plugin's `install.md` and README point `curl` at THIS repo
(`bnb-attestation-service/guard`). If you rename this repo, grep the plugin for the old name
and fix it, or `/aguard-setup` will fetch from a 404.

Before tagging, bump the version in BOTH `plugin/.claude-plugin/plugin.json` and the plugin entry
of `.claude-plugin/marketplace.json` (the source repo's test keeps them equal). The desktop app's
update check reads the marketplace entry; the CLI reads plugin.json. And keep the marketplace
`name` equal to this repo's name (`guard`) — the desktop looks the marketplace up by repo name.

## 2. Cut a release (the binaries)

The binaries are built in the **source repo** (it owns the Makefile, the ldflags, the
checksums). The source repo's own `release.yml` already builds and publishes a release *there*
when a version tag is pushed. To get those same binaries onto THIS public repo:

**Option A — manual (simplest to start):**

```bash
# in the source repo:
make dist                        # produces dist/aguard-* and dist/SHA256SUMS.txt
```

Then in this repo on GitHub: **Releases → Draft a new release →** tag `vX.Y.Z` → drag the four
`aguard-*` binaries and `SHA256SUMS.txt` from `dist/` → **Publish**. Mark it as the latest
release so `releases/latest/download/…` (what `/aguard-setup` fetches) resolves.

With `gh` installed, the same thing without the browser:

```bash
gh release create vX.Y.Z -R bnb-attestation-service/guard \
  --title vX.Y.Z --generate-notes \
  dist/aguard-* dist/SHA256SUMS.txt
```

**Option B — automate it (once, when the manual step gets old):** give the source repo's
`release.yml` a step that, after `make dist`, pushes the same artifacts to this repo's release
using a deploy token with write access here. One-time token setup; after that a tag push in
the source repo publishes to both. Left as an exercise until the manual step is actually a
burden — see the source repo's ROADMAP.

## Version numbers

Keep the release tag here equal to the source repo's tag for the same build (`v0.2.0` ↔
`v0.2.0`). The binary stamps its own version from the source repo's `git describe`, so a
mismatch is visible: `aguard version` will not equal the release tag.
