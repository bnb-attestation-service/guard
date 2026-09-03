# Getting the `aguard` binary

Shared by `agentguard-audit`, `agentguard-vet` and `agentguard-gate`.

## First, look before you fetch

```bash
command -v aguard                     # already on PATH?
ls ./bin/aguard 2>/dev/null           # built from source in this repo?
ls ~/.local/bin/aguard 2>/dev/null
```

If you find one, check it runs and report what you found:

```bash
aguard version    # prints version, commit, build date, and the reputation-list entry count
```

Use that path for the rest of the session. If the only copy is `./bin/aguard`, use
`./bin/aguard` explicitly rather than assuming `aguard` resolves — and mention that the
load-time gate needs a **stable** path, so a binary sitting in a build directory should be
moved before `aguard hook install` (a gate pointing at a deleted binary is the `GATE-001`
failure mode: every skill then loads unaudited, and the silence looks exactly like a clean
result).

## Prebuilt binary (macOS / Linux, no Go toolchain)

Detect the platform instead of asking:

```bash
case "$(uname -s)-$(uname -m)" in
  Darwin-arm64)  PLAT=darwin-arm64 ;;
  Darwin-x86_64) PLAT=darwin-amd64 ;;
  Linux-x86_64)  PLAT=linux-amd64 ;;
  Linux-aarch64) PLAT=linux-arm64 ;;
  *) echo "no prebuilt binary for this platform — build from source" ;;
esac
echo "$PLAT"
```

```bash
REPO=bnb-attestation-service/guard    # public distribution repo (binaries only)
curl -fsSLO "https://github.com/$REPO/releases/latest/download/aguard-$PLAT"
curl -fsSLO "https://github.com/$REPO/releases/latest/download/SHA256SUMS.txt"

# Verify BEFORE running it. Do not skip this step and do not offer to skip it.
shasum -a 256 --ignore-missing -c SHA256SUMS.txt     # Linux: sha256sum --ignore-missing -c
```

Only if that prints `OK`:

```bash
chmod +x "aguard-$PLAT" && sudo mv "aguard-$PLAT" /usr/local/bin/aguard
```

`sudo` will prompt interactively and you may not be able to answer it. Prefer telling the user
to run that one line themselves, or install without root:

```bash
mkdir -p ~/.local/bin && chmod +x "aguard-$PLAT" && mv "aguard-$PLAT" ~/.local/bin/aguard
# then make sure ~/.local/bin is on PATH
```

**If verification fails, stop.** Do not run the binary, do not retry with the check removed.
Report the mismatch — that is the one outcome where the correct action is to do nothing.

## From source (maintainers only — the source repo is private)

Public users do not need this: the `curl` install above pulls a prebuilt binary from the
public distribution repo. This path is for contributors who have access to the private
source repository.

```bash
git clone https://github.com/bnb-attestation-service/agent-guard && cd agent-guard
make build                                        # -> bin/aguard
# or, without make:
CGO_ENABLED=0 go build -o bin/aguard ./cmd/aguard
```

## Windows

Build from source only, on purpose. It cross-compiles, but no test suite has ever run there
and the symlink boundary the scanner relies on behaves differently. Say that plainly rather
than handing a Windows user a binary whose verdicts nothing has validated.

## Under WSL, containers, or a remote box

The scan reads a config root on the machine it runs on. Scanning a `~/.claude` that lives in
WSL means running `aguard` inside WSL — a Windows-side binary pointed at a `\\wsl$` path is
not the same check, and the symlink boundary is exactly what differs. Same for a devcontainer:
scan from inside it.
