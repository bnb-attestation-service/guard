# The AI deep check (LLM judge): setting it up and running it

The static scan is what `aguard` does by default and it needs nothing. The deep check is an
extra pass by an isolated model that reads redacted excerpts and looks for what rules cannot:
a description that says A while the code does B, an injection phrased to dodge keywords, an
encoded blob's real purpose. It is optional, it costs money on a hosted endpoint, and it can
only ADD findings — it never removes one and never lowers the deterministic score.

Three commands cover everything; the user never edits YAML or sets an environment variable.

## 1. Is it set up? — `aguard llm status`

Run this first, always. It prints the config file, endpoint, model and where the key comes
from (never the key). `enabled: false` or `key: none` means not set up → go to step 2. Anything
else → step 3 if the user asked for a check, or you are done if they asked about status.

## 2. Setting it up — `aguard llm setup`

Hosted models only. Two things to settle with the user, in their language, plainly:

1. **Which provider.** Show `aguard llm setup --list` (DeepSeek, OpenAI, Qwen, or any
   OpenAI-style endpoint with `--base-url`/`--model`). Do not recommend one; the choice is theirs
   and their account. Before they get a key, pass on one piece of advice: **create a key just for
   this tool, with a spending cap at the provider.** Then the worst any leak can do is one
   reset and one capped bill.
2. **The key — and which route it takes.** The key has to reach the machine once. Offer the
   safe route first and the convenient one second, and say why they differ:

   **Route A, recommended — the terminal.** Give the user this one line to paste into Terminal:

   ```bash
   aguard llm setup --provider deepseek
   ```

   It prompts for the key with hidden input: nothing on screen, nothing in shell history,
   nothing in this chat. Tell them to say "done" when it has finished, then continue at
   step 3 below with `aguard llm test`. (Add `--model <name>` to override the preset's model.)

   **Route B — paste it here.** Only if the user cannot or will not open a terminal. Say first,
   in one sentence: a key pasted into this chat is stored in this session's transcript on their
   disk and passes through the model that runs this conversation; the dedicated, capped key
   from point 1 is what makes that acceptable. With their go-ahead, feed it on stdin so it is
   never an argument:

   ```bash
   printf '%s\n' "<the key>" | aguard llm setup --provider deepseek --key-stdin
   ```

Either route ends with `aguard llm setup` printing where scanned content will go; read that
sentence back to the user. Then confirm the endpoint answers:

```bash
aguard llm test
```

`OK · <model> answered in …` means done. Anything else is the server's own error (bad key,
unknown model, no network), or a refusal of a plain-http remote endpoint — fix that with the
user, do not proceed to a scan.

## 3. Running it — `aguard scan --llm`

Only after `status` shows it configured. Everything in this skill's triage guidance applies,
plus the judge-specific parts of `references/triage.md`: the report now carries a second score
(`overall_effective`), which the judge may push DOWN but never up; LLM findings are labelled
and advisory; every LLM finding quotes what triggered it. `LLM-000` means the judge covered
less than everything and says why (key unavailable, calls over budget, endpoint failures) —
relay it, do not smooth it over. The stderr line with call count and tokens is the cost;
mention it.

If the user asks for the deep check and it is not set up, do not silently run the static scan
instead: say it needs a one-time setup, offer to do it now, and run the static scan meanwhile
if they want results immediately.

## What to say if asked why it is not on by default

Because it sends content off the machine and costs money, and because the tool's promise is
that the score is reproducible. The judge is the one part that is neither, so it is opt-in
per run and can only ever make a result look worse, never better.
