# The usage card

Print this when the user asks how to use AgentGuard or what it can do, when `/aguard-help` is
run, and as the last thing `/aguard-setup` shows. It is the one piece of onboarding a
non-technical user will actually see, so it lives in one place — edit it here, nowhere else.

Print it as written. Do not translate the examples into flags — the point of the card is that
nothing on it needs a flag.

---

AgentGuard is in place. From here on, just say what you want:

- **Check-up** — "Is my Claude setup safe? Scan it." → audits every installed skill, MCP server, hook and permission grant
- **Vet before installing** — "I downloaded this skill, is it safe to install?" → hand me the folder or the link
- **Understand a result** — "What does EXFIL-001 mean?" "Why is my score 61?"
- **Automatic protection** — "Turn on the gate" → every skill is checked before it loads; anything with a finding asks you first
- **Clean up** — "Clean up my unused skills" → finds duplicate, bloated and stale skills; moves them, never deletes

Forgot how? Type `/aguard-help`. Everything runs on your own machine; nothing is uploaded.
